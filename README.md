# perfSONAR on Cisco IOx (CSR1000v / EVE‑NG)

This repo contains a **working, repeatable guide** for packaging and running a **perfSONAR Testpoint** on **Cisco IOx** (CSR1000v in EVE‑NG), using **LXC** with an **ext2 rootfs** and **descriptor schema 2.2**.

> We validated this flow end-to-end, including common installation/activation errors and how to fix them.

## Quick start
Read the guide:
- [`docs/iox-perfsonar-fresh-start-guide.md`](docs/iox-perfsonar-fresh-start-guide.md)

## Known-good tool versions
- `ioxclient`: **1.17.0.0**
- Docker: **27.5.1**
- Platform: CSR1000v with IOx enabled (EVE‑NG)

## Repo layout
```
.
├── README.md
├── LICENSE
└── docs/
    └── iox-perfsonar-fresh-start-guide.md
```

## License
MIT — see [`LICENSE`](LICENSE).
