# Tailscale Setup

## Objective

Provide secure remote access to the Raspberry Pi without exposing SSH directly to the public Internet.

The goal was to create a private management network that would allow access to the homelab from trusted devices while keeping administrative services off the public network.

---

## Design Decision

Tailscale was installed directly on Raspberry Pi OS instead of running inside Docker.

This was chosen for two main reasons:

1. Remote administration should remain available even if Docker is stopped or misconfigured.
2. Management infrastructure should not depend on the application layer it is supposed to manage.

The resulting architecture is:

```text
Client Device
     │
     │ Tailscale
     ▼
Raspberry Pi
├── Tailscale
├── OpenSSH
└── Docker

Tailscale is used only as the private network layer.

SSH authentication continues to be handled by standard OpenSSH.

Why Not Tailscale SSH

Tailscale SSH was deliberately not used.

The goal of the homelab is to practice standard SSH administration concepts, including:

SSH key generation
authorized_keys
SSH client configuration
sshd_config
public-key authentication
root login restrictions
password authentication control

Using standard OpenSSH over the Tailscale network keeps the networking and authentication layers separate.

Installation

Tailscale was installed directly on the Raspberry Pi host.

After installation, the Raspberry Pi was authenticated into the Tailscale network.

The device received a private Tailscale IP in the 100.x.x.x range.

The Windows host was also added to the same Tailscale network.

Validation

Connectivity was first verified by checking the Tailscale status on the Raspberry Pi.

Example commands:

tailscale status
tailscale ip -4

Remote SSH access was then tested using the Raspberry Pi Tailscale IP:

ssh pi@TAILSCALE_IP

The connection was also tested from outside the local home network using a mobile hotspot.

This confirmed that the Raspberry Pi could be administered remotely without relying on LAN addressing or exposing port 22 through the router.

Access Model

The intended management path is:

Trusted Client
      │
      │ Tailscale
      ▼
Raspberry Pi
      │
      └── OpenSSH

A device must first be connected to the Tailscale network before it can reach the Raspberry Pi through its Tailscale IP.

SSH authentication is then handled separately using SSH keys.

This creates two independent access requirements:

1. Device must have network access through Tailscale
2. Client must have an authorized SSH key
Security Benefits

Using Tailscale avoids exposing SSH directly to the Internet.

No router port forwarding is required for SSH access.

The management interface therefore remains reachable only through the private Tailscale network.

This reduces unnecessary attack surface while still allowing remote administration.

Lessons Learned

This setup helped reinforce the distinction between:

network access
service exposure
authentication
authorization

Tailscale determines whether a device can reach the Raspberry Pi over the private network.

OpenSSH independently determines whether the user is allowed to authenticate.

This separation became important later when implementing SSH hardening and firewall rules.

Future Improvements

Possible future improvements include:

introducing more restrictive Tailscale access policies
adding additional trusted homelab devices
documenting device-specific access rules
evaluating Tailscale ACLs or Grants for larger lab environments
testing subnet routing when multiple Raspberry Pi nodes are introduced
