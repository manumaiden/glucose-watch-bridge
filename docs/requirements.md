# Requirements

## Hardware

| What | Notes |
|---|---|
| **FreeStyle Libre 2 sensor** | Tested with the EU version. Other Libre sensors that report to LibreLinkUp should work too |
| **Patient phone** | Runs the official **LibreLink** app with LibreLinkUp sharing enabled |
| **Follower phone (Android)** | Receives the notifications. It can be the same phone as the patient's |
| **Smartwatch** | Any watch whose companion app **mirrors Android notifications** (tested: Huawei Watch Fit 5 Pro via Huawei Health) |
| **Always-on host** | Runs Docker: a NAS, a Raspberry Pi 4, a mini PC or a VM. About 1 GB of RAM is enough for Nightscout + MongoDB + ntfy |

> **MongoDB and CPUs:** MongoDB 5.0+ needs **AVX** on x86_64 (many Intel Celeron/Atom NAS CPUs don't have it). MongoDB 4.4.19+ needs **ARMv8.2-A** on ARM64 (a Raspberry Pi 4 is ARMv8.0). The compose file explains which tag to use.

## Software

| Component | Version tested | Role |
|---|---|---|
| Docker + Docker Compose | 24+ | Runs Nightscout, MongoDB and ntfy |
| [Nightscout](https://github.com/nightscout/cgm-remote-monitor) | 15.x | Stores readings, dashboard |
| MongoDB | 4.4 | Nightscout database |
| [ntfy](https://ntfy.sh) server | 2.x | Self-hosted push notifications |
| [n8n](https://n8n.io) | 2.x (self-hosted) | Workflow engine: polling, transformation, alerts |
| ntfy Android app | latest | Receives the push on the phone |
| Watch companion app | — | Mirrors the phone notifications to the watch |
| [xDrip+](https://github.com/NightscoutFoundation/xDrip) *(optional)* | 2026.x | Charts on the phone in *Nightscout Follower* mode |

n8n can run on the same host as Nightscout/ntfy or elsewhere. It only needs HTTPS access to LibreLinkUp and HTTP access to Nightscout and ntfy.

## Accounts

| Account | Why |
|---|---|
| **LibreLinkUp follower account** | The poller logs in as a *follower*: from the patient's LibreLink app, invite an email address, accept the invitation in the LibreLinkUp app and use those credentials. The patient account itself does **not** return data from `/llu/connections` |
| ntfy user (local) | Created on your own ntfy server, see [setup](setup.md#2-ntfy) |

No other cloud account is required.

## Network

- **At home only:** the phone reaches ntfy on the LAN. No exposure is needed, but you only get alerts when you are connected to the home Wi-Fi.
- **Everywhere (recommended):** expose **ntfy** (and optionally Nightscout) over **HTTPS**, using a reverse proxy with a TLS certificate, a tunnel (e.g. Cloudflare Tunnel) or a VPN (WireGuard, Tailscale). Keep `auth-default-access: deny-all` on ntfy and `AUTH_DEFAULT_ROLES=denied` on Nightscout. Credentials travel with every request, so never expose plain `http://` endpoints to the internet.
