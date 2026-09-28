# Raspberry Pi Home Server & Cybersecurity Homelab

A Raspberry Pi-based home server and cybersecurity homelab built to develop practical skills in **Linux administration, networking, Docker, system hardening, monitoring and security operations**.

The project combines a useful self-hosted environment with a realistic infrastructure that can gradually evolve into a small **SOC / security lab**.

> This repository focuses on architecture, configuration, security decisions, troubleshooting and lessons learned. Secrets, personal data and sensitive content are intentionally excluded.

---

## Project Goals

- Build a reliable self-hosted home server
- Apply secure-by-default configurations
- Improve Linux and networking administration skills
- Learn Docker and Docker Compose
- Practice SSH hardening, firewalling and secure remote access
- Understand service exposure and container networking
- Build a foundation for future SIEM, IDS and detection engineering labs
- Document technical decisions and troubleshooting

---

## Architecture

The current environment is built around a **Raspberry Pi running Raspberry Pi OS**.

### Core Stack

- Raspberry Pi OS
- Docker
- Docker Compose
- Tailscale
- OpenSSH
- UFW
- 1 TB external HDD
- Jellyfin

### Storage Layout

The external HDD contains:

- **NTFS partition** for existing personal data
- **ext4 partition** for Linux, Docker and homelab workloads

The homelab partition is mounted at:

```text
/mnt/homelab
```

```text
/mnt/homelab/
├── docker/
│   ├── compose/
│   ├── volumes/
│   └── backups/
├── media/
│   └── movies/
├── projects/
└── logs/
```

Persistent mounts use filesystem **UUIDs through `/etc/fstab`** rather than relying on device names such as `/dev/sda4`.

---

## Security Baseline

### SSH Hardening

SSH uses public-key authentication only:

```text
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
```

A dedicated **Ed25519 key** is used for homelab access.

Password authentication and root SSH login are disabled.

### Private Remote Access

**Tailscale** provides the private management network and runs directly on the Raspberry Pi host.

Keeping Tailscale outside Docker means remote administration remains available even if the Docker environment is stopped or misconfigured.

SSH authentication continues to be handled by standard OpenSSH.

### Firewall

UFW follows a default-deny inbound policy:

```text
Incoming: deny
Outgoing: allow
Routed: deny
```

SSH is only allowed through the:

```text
tailscale0
```

interface, preventing direct SSH exposure through the normal LAN interface.

---

## Docker Strategy

Docker is used to run self-hosted services.

The Docker engine remains in its standard location:

```text
/var/lib/docker
```

Persistent application data is stored separately on the ext4 homelab partition:

```text
/mnt/homelab/docker/
├── compose/
├── volumes/
└── backups/
```

Each service is designed to have its own Docker Compose configuration.

---

## Jellyfin

Jellyfin is the first service deployed in the homelab.

It runs as a Docker container managed with Docker Compose, with persistent configuration stored on the ext4 partition.

Media is stored under:

```text
/mnt/homelab/media/movies
```

The media directory is mounted **read-only** inside the container:

```text
Host:      /mnt/homelab/media
Container: /media
```

This allows Jellyfin to access media while preventing the container from modifying or deleting the original files.

### Network Exposure

Jellyfin was initially exposed using:

```text
0.0.0.0:8096
```

This made it reachable through both the LAN and Tailscale interfaces.

During testing, this highlighted an important security consideration: **Docker manages its own networking and forwarding rules**, meaning a host firewall such as UFW cannot always be treated as the only layer controlling published container ports.

The service was therefore changed to bind port `8096` specifically to the Raspberry Pi's **Tailscale IP**.

Expected access model:

```text
LAN IP:8096        -> unavailable
Tailscale IP:8096  -> available
```

---

## Key Lessons Learned

### SSH Configuration Precedence

Multiple files under:

```text
/etc/ssh/sshd_config.d/
```

can affect the final SSH configuration.

The effective configuration was verified with:

```bash
sshd -T
```

This reinforced the importance of validating the **effective configuration** rather than assuming a configuration file is being applied as expected.

### SSH Key Selection

A custom private-key filename initially caused the SSH client to fall back to password authentication.

An SSH client configuration entry was created to explicitly define the identity file, allowing simplified access such as:

```bash
ssh mirtilo
```

### Docker and UFW

Publishing a Docker port made Jellyfin reachable from the LAN despite UFW's default-deny inbound policy.

The issue was mitigated by binding the service directly to the Tailscale IP instead of all network interfaces.

### Filesystem Preparation

The external NTFS filesystem initially reported an unclean state.

It was checked and repaired with Windows `chkdsk` before resizing and creating the ext4 homelab partition.

### Persistent Storage

UUID-based mounts were configured in `/etc/fstab` and validated before reboot using:

```bash
mount -a
```

---

## Repository Structure

```text
homelab/
├── README.md
├── architecture/
├── docker/
│   └── jellyfin/
│       └── compose.yml
├── docs/
│   ├── 01-tailscale.md
│   ├── 02-ssh-hardening.md
│   ├── 03-firewall.md
│   ├── 04-storage.md
│   └── 05-jellyfin.md
└── screenshots/
```

The `docs/` directory contains more detailed implementation notes, including design decisions, configuration, validation, troubleshooting and lessons learned.

---

## Roadmap

Planned improvements include:

- Host and service monitoring
- Personal cloud and file storage
- Photo management
- Backup and recovery procedures
- Centralized logging
- Wazuh-based mini SOC
- Host and Docker log ingestion
- Custom detection rules
- MITRE ATT&CK mapping
- Network IDS monitoring
- Controlled attack simulations
- Incident investigation scenarios
- Additional Raspberry Pi nodes
- Managed switching and VLAN segmentation
- Separation between production-like services and attack-lab systems

The long-term goal is to build security exercises following a realistic workflow:

```text
activity
   ↓
telemetry
   ↓
detection
   ↓
investigation
   ↓
conclusion
   ↓
remediation
```

---

## Skills Practiced

`Linux` · `SSH` · `Public-Key Authentication` · `UFW` · `Networking` · `Tailscale` · `Docker` · `Docker Compose` · `Linux Filesystems` · `ext4` · `NTFS` · `Storage Management` · `Unix Permissions` · `System Hardening` · `Troubleshooting` · `Security Monitoring`

---

## Security & Privacy

The following are intentionally excluded from version control:

- Private SSH keys
- Passwords and authentication tokens
- API keys
- `.env` files containing secrets
- Personal and media files
- Docker runtime data
- Databases and backups containing personal data
- Sensitive screenshots

Configuration files are reviewed before being committed to reduce the risk of accidentally publishing sensitive information.

---

## Disclaimer

This repository documents a personal home server and cybersecurity lab.

All testing and security exercises are performed only on systems and services owned or explicitly controlled by the lab owner.
