# Homelab

Isolated cybersecurity lab built on Proxmox VE for offensive security testing and data engineering. The goal is a controlled environment where attack scenarios can be executed without risk to the production network.

---

## Architecture

### Network Topology

```
Internet
    │
 Home Router
    │
  vmbr0 (LAN/Admin — 192.168.1.0/24)
    │
 Proxmox Host ──── Tailscale VPN (remote access)
    │
  vmbr1 (Isolated Lab — 10.0.0.0/24)
    │
 OPNsense (Gateway + DHCP)
    │
  ┌──────────────┐
  │              │
Kali-01       Metasploitable
(dual-homed)  (isolated target)
```

### Nodes

| ID | Name | Type | Network | Role |
|---|---|---|---|---|
| Host | Proxmox | Hypervisor | vmbr0 + Tailscale | Hypervisor, VPN gateway |
| 101 | opnsense-gateway | VM (FreeBSD) | vmbr0 WAN / vmbr1 LAN | Firewall, DHCP for lab net |
| 200 | mgmt-node | LXC | vmbr0 | Ansible control node |
| 201 | web-01 | LXC | vmbr0 | Nginx target (offensive testing) |
| 202 | kali-01 | LXC | vmbr0 + vmbr1 | Dual-homed attack node |
| 300 | metasploitable | VM | vmbr1 only | Vulnerable target, fully isolated |

### Key Design Decisions

- **Two network segments** — admin (vmbr0) and isolated lab (vmbr1). Metasploitable has no path to the home network.
- **Kali is the only bridge** — dual-homed, intentionally. Acts as the pivot point and SSH jump host for OPNsense.
- **Tailscale for remote access** — no ports exposed to the internet. Web UI locked to Tailscale interface via iptables.
- **OPNsense as lab gateway** — handles DHCP for vmbr1, sits between the two segments.

---

## Security Hardening

| Control | Status | Detail |
|---|---|---|
| Internet exposure | ✅ None | No open ports to internet |
| Network segmentation | ✅ Done | vmbr0 / vmbr1, OPNsense between |
| Remote access | ✅ Tailscale | Subnet router on Proxmox host |
| Proxmox web UI | ✅ Locked | iptables DROP on LAN, ACCEPT on tailscale0 only |
| SSH authentication | ✅ Keys only | PasswordAuthentication no on all nodes |
| iptables persistence | ✅ Saved | /etc/iptables/rules.v4 via iptables-persistent |
| SSH user | ⚠️ Root | To be replaced with dedicated users |
| OPNsense rules | ⚠️ Undocumented | Inter-segment policy not yet defined |
| Logging/SIEM | ❌ Missing | No network visibility yet |

---

## Stack

| Tool | Role |
|---|---|
| Proxmox VE | Hypervisor |
| OPNsense 26.1 | Firewall / gateway |
| Kali Linux | Attack platform |
| Metasploitable | Vulnerable target |
| Nginx 1.22.1 | Web target |
| Ansible | Orchestration |
| Tailscale | VPN / remote access |

---

## Repository Structure

```
homelab/
├── README.md
├── docs/
│   └── log-2026.md       # Detailed session logs
├── ansible/
│   ├── deploy_ct.yml     # Spin up new LXC containers
│   ├── setup_web.yml     # Configure Nginx on web-01
│   └── hosts.ini         # Inventory
└── scripts/
    └── healthcheck.sh    # Verify lab is healthy
```

---

## Lessons Learned

- Never edit OPNsense `config.xml` directly — caused full reinstall
- OPNsense 26.1 detects live media by whether the ISO device is attached, not boot order — remove with `qm set 101 --ide2 none`
- `opnsense-installer` can be run from the live shell (option 8) — no need to catch the boot menu
- LXC containers on `local` (directory) storage don't support snapshots — use `vzdump` instead
- OPNsense 26.1 uses Kea DHCP — no lease file at `/var/db/dhcpd/dhcpd.leases`, use nmap scan instead
- When Tailscale is active but you're on the same LAN, traffic to the LAN IP bypasses `tailscale0` — always use the Tailscale IP for services locked to that interface
