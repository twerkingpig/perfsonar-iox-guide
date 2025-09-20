# perfSONAR on Cisco IOx (CSR1000v / EVE‑NG)

A clean, repeatable build based on everything we verified together. This uses **LXC** (not Docker) with an **ext2 rootfs** and works on CSR1000v IOS‑XE that only accepts **schema 2.2** descriptors.

---

## 1) What you’ll get
- `perfsonar.tar` — installable IOx bundle
- `rootfs.img` — ext2 image containing the perfSONAR filesystem
- A minimal `package.yaml` (schema **2.2**, LXC with `kernel-version`)

---

## 2) Known‑good environment
- Build host: Linux x86_64
- Docker: **27.5.1**
- ioxclient: **1.17.0.0**
- CSR1000v in EVE‑NG with IOx enabled

> Newer Docker versions often break ioxclient’s Docker‑layer reads. We avoid this by building an **ext2 rootfs** via `ioxclient docker package -p ext2`, then repackaging for schema 2.2.

---

## 3) Build on your Linux host
Create a working directory:
```bash
mkdir -p ~/iox/perfsonar && cd ~/iox/perfsonar
```

### 3.1 Pull & sanity check image
```bash
docker pull perfsonar/testpoint:latest
# CSR1000v is x86_64 / amd64
docker image inspect perfsonar/testpoint:latest --format '{{.Architecture}}/{{.Os}}'
```

### 3.2 Build an LXC package with an ext2 rootfs (schema 2.12)
This step generates `rootfs.img` for us.
```bash
ioxclient docker package -p ext2 -r 512 \
  --skip-signing -n perfsonar \
  perfsonar/testpoint:latest .
# Output: perfsonar.tar (envelope with artifacts.tar.gz containing rootfs.img)
```

### 3.3 Extract `rootfs.img`
```bash
mkdir -p /tmp/ioxrepack && cd /tmp/ioxrepack
cp ~/iox/perfsonar/perfsonar.tar .
tar xf perfsonar.tar
# Pull out only the rootfs image
tar xf artifacts.tar.gz rootfs.img
mv rootfs.img ~/iox/perfsonar/
cd ~/iox/perfsonar
```

### 3.4 Add a wrapper entrypoint (spawns shell + starts perfSONAR)
```bash
sudo mkdir -p /mnt/ioxroot
sudo mount -o loop rootfs.img /mnt/ioxroot

# Wrapper runs perfSONAR if present, then keeps shell for console access
sudo tee /mnt/ioxroot/iox_entry.sh >/dev/null <<'EOF'
#!/bin/sh
[ -x /usr/lib/perfsonar/scripts/docker-start.sh ] \
  && /usr/lib/perfsonar/scripts/docker-start.sh || true
exec /bin/sh
EOF
sudo chmod +x /mnt/ioxroot/iox_entry.sh

sudo umount /mnt/ioxroot
rmdir /mnt/ioxroot
```

### 3.5 Create a schema‑2.2 LXC descriptor pointing at `rootfs.img`
> We include `kernel-version` because older schema requires it and IOS‑XE on CSR1000v often rejects newer schemas.
```yaml
# ~/iox/perfsonar/package.yaml

descriptor-schema-version: "2.2"

info:
  name: "perfsonar"
  description: "perfSONAR testpoint packaged for IOx"
  version: "1.0.0"

app:
  cpuarch: "x86_64"
  type: lxc
  kernel-version: "3.10"

  resources:
    profile: "custom"
    memory: 2048
    disk: 1500            # keep <= platform free space
    network:
      - interface-name: eth0

  startup:
    rootfs: "rootfs.img"
    target: "/iox_entry.sh"
```

### 3.6 Build the final bundle (schema 2.2)
```bash
cd ~/iox/perfsonar
ioxclient package -n perfsonar --skip-signing .
ls -lh perfsonar.tar
```

---

## 4) Copy to the router (SCP or SFTP)
Enable SCP on the router once:
```bash
conf t
 ip scp server enable
 username <YOU> privilege 15 secret <PW>
end
```
Copy from your Linux host (note the path syntax):
```bash
scp ~/iox/perfsonar/perfsonar.tar <YOU>@<ROUTER_IP>:bootflash:/perfsonar.tar
```
> Alternatives: `sftp` (uses SSH), or copy **from** the router: `copy scp: bootflash:`.

---

## 5) Router configuration & install
Use **VirtualPortGroup0** for the app subnet `10.10.50.0/24` and **Gi1** to your home LAN `192.168.0.0/24`.

### 5.1 Interfaces (example)
```bash
conf t
 interface GigabitEthernet1
  description Conn to Home Network
  ip address 192.168.0.180 255.255.255.0
 !
 interface VirtualPortGroup0
  description IOx/perfSONAR
  ip address 10.10.50.1 255.255.255.0
  no shut
end
```

### 5.2 App resources & networking
```bash
conf t
 iox
 app-hosting appid perfsonar
  app-vnic gateway0 virtualportgroup 0 guest-interface 0
  app-resource profile custom
  app-resource cpu 1000
  app-resource memory 2048
  app-resource persist-disk 1500
  app-resource vcpu 1
  app-default-gateway 10.10.50.1 guest-interface 0
  app-vnic gateway0
   guest-interface 0
   guest-ipaddress 10.10.50.10 netmask 255.255.255.0
   exit
 exit
end
```

### 5.3 Install / activate / start
```bash
app-hosting install appid perfsonar package bootflash:perfsonar.tar
app-hosting activate appid perfsonar
app-hosting start appid perfsonar
```

After install/activate, delete the tar to keep flash healthy:
```bash
delete /force bootflash:perfsonar.tar
```

---

## 6) Reach the app from your LAN
**Option A — Routed (preferred)**  
- On CSR: default route to home gateway
  ```bash
  conf t
   ip route 0.0.0.0 0.0.0.0 192.168.0.1
  end
  ```
- On home router: static route back to IOx subnet
  ```
  10.10.50.0/24 via 192.168.0.180
  ```

**Option B — NAT (if you can’t edit the home router)**
```bash
conf t
 ip access-list extended IOX-NAT
  permit ip 10.10.50.0 0.0.0.255 192.168.0.0 0.0.0.255
 interface VirtualPortGroup0
  ip nat inside
 interface GigabitEthernet1
  ip nat outside
 ip nat inside source list IOX-NAT interface GigabitEthernet1 overload
 ip route 0.0.0.0 0.0.0.0 192.168.0.1
end

! Optional inbound forwards (examples)
ip nat inside source static tcp 10.10.50.10 443 192.168.0.180 8443
ip nat inside source static tcp 10.10.50.10 5201 192.168.0.180 5201
```

**Topology (simplified)**
```
192.168.0.0/24           10.10.50.0/24
+ Home GW 192.168.0.1    + CSR1000v
|                         |  Gi1: 192.168.0.180
|                         |  VPG0: 10.10.50.1
|                         +----- App: 10.10.50.10
```

---

## 7) Using the perfSONAR node
**Connect to console (LXC uses console/aux, not session):**
```bash
app-hosting connect appid perfsonar console
# Press Enter to get a prompt. Exit with Ctrl-C, Ctrl-C, Ctrl-C
```
Inside the shell:
```bash
# Services usually started by /iox_entry.sh, but safe to run once
[ -x /usr/lib/perfsonar/scripts/docker-start.sh ] && \
  /usr/lib/perfsonar/scripts/docker-start.sh || true

pscheduler status
ip addr ; ip route
ping -c 2 10.10.50.1
ping -c 2 192.168.0.1
```

**Run tests (examples)**
```bash
# Latency
pscheduler task latency --dest 192.168.0.1 --ip-versions 4

# Traceroute
pscheduler task trace --dest 1.1.1.1 --ip-versions 4

# Throughput (peer must run `iperf3 -s`)
pscheduler task --tool iperf3 throughput --dest 192.168.0.50 --duration PT10S
```

**Schedules**
```bash
pscheduler task --repeat PT5M --tool iperf3 throughput \
  --dest 192.168.0.50 --duration PT10S
pscheduler task --repeat PT1M latency --dest 192.168.0.1 --ip-versions 4
pscheduler tasks
pscheduler runs --after PT30M
```

---

## 8) Troubleshooting (field‑tested fixes)
**A. `show app-hosting list` hangs**  
- Check resources: `show platform resources` (RP CPU often 100%).  
- Restart IOx stack safely:
  ```bash
  conf t ; no iox ; end ; sleep 20 ; conf t ; iox ; end
  show iox ; show app-hosting list
  ```
- Give CSR more vCPU/RAM (≥2–4 vCPU, ≥6–8 GB RAM).

**B. Install failed: `Unable to extract docker rootfs … Mandatory layer blobs is missing!`**  
- You packaged as Docker but provided a flat rootfs. Use **LXC + ext2** method above.

**C. Install failed: `Given schema version 2.12 is not supported`**  
- Repackage with **schema 2.2** and include `kernel-version: "3.10"`.

**D. Activation failed: `Invalid value for cpu 0`**  
- Set CPU units on router: `app-resource cpu 1000` under the app’s custom profile.

**E. Activation failed: `Required: 4000 … available 2309` (storage)**  
- Lower app disk/persist‑disk to fit (e.g., **1500**).  
- Free `bootflash:` (delete old TARs/ISOs), or rebuild with smaller `disk:` in `package.yaml`.

**F. Bootflash low space alarm**  
- Delete upload tar after install: `delete /force bootflash:perfsonar.tar`.

**G. `perfsonar is not docker type` when connecting**  
- Use `console` or `aux`, not `session` (Docker‑only).

**H. No shell prompt after connect**  
- Ensure wrapper `target: /iox_entry.sh` exists and `exec /bin/sh`. Rebuild if needed.

**I. SCP keeps disconnecting**  
- Enable `ip scp server enable`, ensure `username … privilege 15`, and use path `bootflash:/filename`.  
- Or use `sftp`, or pull from the router with `copy scp:`.

**J. Memory during install**  
- IOx extracts into `/tmp` (RAM). If installs fail, temporarily `no iox`→`iox`, give more RAM, or rebuild with smaller `-r` headroom (e.g., `-r 256`).

---

## 9) End‑to‑end cheat sheet (copy‑paste)
```bash
# Build host
mkdir -p ~/iox/perfsonar && cd ~/iox/perfsonar
docker pull perfsonar/testpoint:latest
ioxclient docker package -p ext2 -r 512 --skip-signing -n perfsonar perfsonar/testpoint:latest .
mkdir -p /tmp/ioxrepack && cd /tmp/ioxrepack && cp ~/iox/perfsonar/perfsonar.tar .
tar xf perfsonar.tar && tar xf artifacts.tar.gz rootfs.img && mv rootfs.img ~/iox/perfsonar/ && cd ~/iox/perfsonar
sudo mkdir -p /mnt/ioxroot && sudo mount -o loop rootfs.img /mnt/ioxroot
sudo tee /mnt/ioxroot/iox_entry.sh >/dev/null <<'EOF'
#!/bin/sh
[ -x /usr/lib/perfsonar/scripts/docker-start.sh ] && /usr/lib/perfsonar/scripts/docker-start.sh || true
exec /bin/sh
EOF
sudo chmod +x /mnt/ioxroot/iox_entry.sh
sudo umount /mnt/ioxroot && rmdir /mnt/ioxroot
cat > package.yaml <<'YAML'
descriptor-schema-version: "2.2"
info: { name: "perfsonar", description: "perfSONAR testpoint packaged for IOx", version: "1.0.0" }
app:
  cpuarch: "x86_64"
  type: lxc
  kernel-version: "3.10"
  resources: { profile: "custom", memory: 2048, disk: 1500, network: [ { interface-name: eth0 } ] }
  startup: { rootfs: "rootfs.img", target: "/iox_entry.sh" }
YAML
ioxclient package -n perfsonar --skip-signing .
scp perfsonar.tar <YOU>@<ROUTER_IP>:bootflash:/perfsonar.tar

# Router
conf t
 interface GigabitEthernet1
  ip address 192.168.0.180 255.255.255.0
 interface VirtualPortGroup0
  ip address 10.10.50.1 255.255.255.0
  no shut
 app-hosting appid perfsonar
  app-vnic gateway0 virtualportgroup 0 guest-interface 0
  app-resource profile custom
  app-resource cpu 1000
  app-resource memory 2048
  app-resource persist-disk 1500
  app-resource vcpu 1
  app-default-gateway 10.10.50.1 guest-interface 0
  app-vnic gateway0
   guest-interface 0
   guest-ipaddress 10.10.50.10 netmask 255.255.255.0
   exit
 end
app-hosting install appid perfsonar package bootflash:perfsonar.tar
app-hosting activate appid perfsonar
app-hosting start appid perfsonar
app-hosting connect appid perfsonar console
pscheduler status
```
