# Firewall Configuration

## Objective

Configure a host firewall on the Raspberry Pi using UFW with a default-deny inbound policy.

The goal was to reduce unnecessary network exposure while keeping required administrative access available through Tailscale.

---

## Firewall Choice

UFW was selected because it provides a simple interface for managing Linux firewall rules while still allowing practical exposure to concepts such as:

- default policies
- inbound versus outbound traffic
- interface-specific rules
- port filtering
- least privilege
- service exposure

The objective was to understand the policy model without immediately working directly with lower-level `nftables` syntax.

---

## Initial Policy

The firewall was configured with:

```text
Incoming: deny
Outgoing: allow
Routed: deny

This means:

unsolicited inbound traffic is blocked by default
outbound traffic initiated by the Raspberry Pi is allowed
forwarded traffic is denied by default

The configuration was applied using UFW.

SSH Rule

SSH should only be reachable through the Tailscale interface.

The rule used was:

sudo ufw allow in on tailscale0 to any port 22 proto tcp

This permits TCP traffic destined for port 22 only when it arrives through:

tailscale0

No equivalent SSH rule was created for the normal LAN interface.

Resulting Access Model

The intended SSH access model is:

Internet
   │
   X
   │
Local LAN
   │
   X
   │
Tailscale
   │
   ▼
TCP/22
   │
   ▼
OpenSSH

This means SSH access through the Raspberry Pi LAN IP is blocked, while SSH access through the Tailscale IP remains available.

Validation

After enabling UFW, the active policy was verified with:

sudo ufw status verbose

The resulting configuration showed:

Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)

The SSH rule showed:

22/tcp on tailscale0       ALLOW IN    Anywhere
22/tcp (v6) on tailscale0  ALLOW IN    Anywhere (v6)
Connectivity Testing

Two SSH tests were performed.

Tailscale IP
ssh mirtilo

This remained functional.

LAN IP
ssh pi@LAN_IP

This failed as expected.

This confirmed that SSH was restricted to the Tailscale interface.

Docker Networking Discovery

After deploying Jellyfin with Docker, the container was initially published using:

ports:
  - "8096:8096/tcp"

Docker reported the port as:

0.0.0.0:8096->8096/tcp

This meant port 8096 was published on all host IPv4 interfaces.

Testing showed that Jellyfin was accessible through both:

Tailscale IP:8096
LAN IP:8096

even though UFW was configured with a default-deny inbound policy.

Why This Happened

Docker manages networking and forwarding rules when container ports are published.

Traffic destined for published container ports can be processed through Docker-managed forwarding and NAT rules rather than only through the normal host input path controlled by simple UFW policies.

This means a host firewall configuration should not automatically be assumed to fully control Docker-published ports.

Mitigation

Instead of publishing Jellyfin on all interfaces, the Docker Compose configuration was changed to bind only to the Raspberry Pi Tailscale IP.

The configuration was changed from:

ports:
  - "8096:8096/tcp"

to:

ports:
  - "TAILSCALE_IP:8096:8096/tcp"

After recreating the container, Docker reported the service as bound specifically to the Tailscale address.

Final Jellyfin Access Model

The resulting access model is:

LAN IP:8096
   │
   X

TAILSCALE_IP:8096
   │
   ▼
Jellyfin container

The service is therefore not published on the normal LAN interface.

Defense in Depth

The final design uses multiple independent controls:

Tailscale
   ↓
Private network access
   ↓
Interface-specific service exposure
   ↓
UFW host firewall
   ↓
Application authentication

Rather than exposing a service broadly and relying only on the firewall to block access, unnecessary exposure is avoided directly at the Docker port-binding level.

Lessons Learned

This setup reinforced several important concepts:

default-deny firewall policies are useful but must be validated
interface-specific rules provide more precise access control
destination port and source port are different concepts
Docker can create networking and forwarding rules independently of UFW
published container ports should be tested from multiple network paths
0.0.0.0 means a service is bound to all IPv4 interfaces
security controls should be validated through actual connectivity testing
reducing exposure at the application or container layer can complement firewall controls
Future Improvements

Possible future improvements include:

reviewing Docker-specific firewall filtering
evaluating the DOCKER-USER chain for centralized container filtering
introducing network segmentation
creating separate trusted and lab networks
adding VLANs using a managed switch
forwarding firewall logs to a SIEM
monitoring denied connections
documenting per-service exposure requirements
