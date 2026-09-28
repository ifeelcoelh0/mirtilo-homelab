# SSH Hardening

## Objective

Harden remote administration of the Raspberry Pi by removing password-based SSH access and requiring public-key authentication.

The goal was to reduce the risk of credential-based attacks while keeping remote administration simple and reliable.

---

## Initial State

The Raspberry Pi initially allowed both password and public-key authentication.

The effective SSH configuration showed:

```text
permitrootlogin without-password
pubkeyauthentication yes
passwordauthentication yes

Although public-key authentication was supported, the actual login method being used was still password authentication.

SSH Key Generation

A dedicated Ed25519 SSH key was created on the Kali WSL client.

Example:

ssh-keygen -t ed25519 -C "homelab-kali"

The private key was stored on the client under:

~/.ssh/mirtilo_ed25519

The public key was copied to the Raspberry Pi.

Example:

ssh-copy-id -i ~/.ssh/mirtilo_ed25519.pub pi@TAILSCALE_IP

The private key remains only on the client system.

SSH Client Configuration

Because the SSH key used a custom filename, the SSH client did not automatically select it.

A client configuration entry was created in:

~/.ssh/config

Example:

Host mirtilo
    HostName TAILSCALE_IP
    User pi
    IdentityFile ~/.ssh/mirtilo_ed25519
    IdentitiesOnly yes

This allows simplified access using:

ssh mirtilo
Server Hardening

The target SSH configuration was:

PubkeyAuthentication yes
PasswordAuthentication no
PermitRootLogin no

A dedicated hardening file was created under:

/etc/ssh/sshd_config.d/
Configuration Precedence Issue

The initial hardening file was created with a high-numbered filename:

99-mirtilo-hardening.conf

However, another configuration file already contained:

PasswordAuthentication yes

The conflicting file was:

/etc/ssh/sshd_config.d/50-cloud-init.conf

Because SSH uses the first applicable value for many configuration options, the earlier file was taking precedence.

The hardening file was renamed to load earlier:

01-mirtilo-hardening.conf

This resulted in the intended effective configuration.

Validation

SSH syntax was validated before reloading the service:

sudo sshd -t

The effective configuration was verified with:

sudo sshd -T | grep -E 'passwordauthentication|pubkeyauthentication|permitrootlogin'

Expected result:

passwordauthentication no
pubkeyauthentication yes
permitrootlogin no

The SSH service was then reloaded.

Authentication Testing

Public-key authentication was confirmed using:

ssh -v mirtilo

The successful authentication method showed:

Authenticated to ... using "publickey"

Password authentication was tested by explicitly disabling public-key authentication on the client:

ssh -o PubkeyAuthentication=no pi@TAILSCALE_IP

The expected result was:

Permission denied (publickey)

This confirmed that password authentication was no longer available.

Final Access Model

The final SSH access path is:

Client
  │
  │ Tailscale
  ▼
Raspberry Pi
  │
  └── OpenSSH
       │
       └── Ed25519 public-key authentication

A successful connection requires:

Network access to the Raspberry Pi through Tailscale
A valid SSH private key corresponding to an authorized public key
Security Improvements

The final SSH configuration provides:

no password-based SSH authentication
no direct root login
dedicated SSH key authentication
simplified client configuration
reduced exposure to password guessing attacks
Lessons Learned

This setup reinforced several important concepts:

SSH server support for public keys does not mean the client is actually using them
custom private-key filenames may require explicit SSH client configuration
SSH configuration files can override each other depending on load order
effective configuration should be validated using sshd -T
authentication changes should always be tested in a second session before closing the original connection
Future Improvements

Possible future improvements include:

using separate SSH keys per client device
adding AllowUsers restrictions
implementing key rotation procedures
reviewing SSH logging and failed-login events
integrating SSH logs into a SIEM
