# Jellyfin Deployment

## Objective

Deploy Jellyfin as the first real self-hosted service in the homelab.

The goal was to practice:

- Docker Compose
- bind mounts
- persistent container storage
- Linux permissions
- service exposure
- container networking
- validation of firewall assumptions

Jellyfin was chosen because it provides a useful everyday service while also generating realistic application and network activity for future monitoring and security labs.

---

## Directory Structure

Jellyfin uses the following directories:

```text
/mnt/homelab/
├── docker/
│   ├── compose/
│   │   └── jellyfin/
│   │       └── compose.yml
│   └── volumes/
│       └── jellyfin/
│           ├── config/
│           └── cache/
└── media/
    └── movies/

The directories were created before deploying the container.

Docker Compose Configuration

Jellyfin is managed with Docker Compose.

The Compose file is stored at:

/mnt/homelab/docker/compose/jellyfin/compose.yml

The service uses persistent bind mounts for configuration and cache data.

The media directory is mounted read-only.

Example structure:

services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin

    user: "1000:1000"

    ports:
      - "TAILSCALE_IP:8096:8096/tcp"

    volumes:
      - /mnt/homelab/docker/volumes/jellyfin/config:/config
      - /mnt/homelab/docker/volumes/jellyfin/cache:/cache
      - type: bind
        source: /mnt/homelab/media
        target: /media
        read_only: true

    restart: unless-stopped

The real Tailscale IP is intentionally replaced with a placeholder in the public documentation.

Running as a Non-Root User

The Jellyfin container is configured to run using the Raspberry Pi user's UID and GID:

1000:1000

This avoids running the application as root unnecessarily.

The host UID and GID were verified before deployment.

Example:

id pi
Persistent Data

Jellyfin configuration data is stored in:

/mnt/homelab/docker/volumes/jellyfin/config

Cache data is stored in:

/mnt/homelab/docker/volumes/jellyfin/cache

These directories exist outside the container.

This means the container can be recreated without losing the Jellyfin configuration.

The container itself is treated as disposable, while persistent data is stored separately.

Media Storage

Movie files are stored under:

/mnt/homelab/media/movies

Inside the container, the media directory appears as:

/media

The Jellyfin movie library therefore uses:

/media/movies
Read-Only Media Mount

The media bind mount is configured as read-only:

read_only: true

This means Jellyfin can read and stream the files but cannot modify or delete the original media through that mount.

The access model is:

Host media
    │
    │ read-only bind mount
    ▼
Jellyfin container

This follows the principle of giving the service only the permissions it requires.

Media Permissions

Movie files were initially transferred with overly permissive permissions.

They appeared as:

-rwxrwxrwx

This corresponds to:

777

Since media files do not need execute permissions or write access for every user, permissions were changed to:

644

Example:

chmod 644 /mnt/homelab/media/movies/*.mkv

The resulting permission model is:

Owner: read + write
Group: read
Others: read

Directories remain traversable so Jellyfin can access the files.

Media Naming

Movie filenames were cleaned before indexing.

Release-specific information such as:

720p
Dual Audio
source names
release site names

was removed.

Files were renamed into a simple format such as:

Parasite (2019).mkv
The Lord of the Rings - The Fellowship of the Ring (2001).mkv

This improves Jellyfin metadata matching.

Deployment

The Compose configuration was validated before starting the container:

docker compose config

The service was then started using:

docker compose up -d

The container status was verified with:

docker ps

The Jellyfin container reported:

healthy

indicating that the service started successfully.

Initial Configuration

The Jellyfin web interface was accessed through the Raspberry Pi Tailscale address.

The initial setup included:

creating the administrator account
configuring language preferences
creating a Movies library
selecting /media/movies as the library path

After scanning the library, the media files were detected successfully.

Setup Wizard Issue

During the initial setup, navigating backwards in the setup wizard caused the web interface to display an error.

Container logs showed repeated messages similar to:

Token is required. URL GET /socket

The container itself remained healthy.

The issue was resolved by reopening the Jellyfin interface and completing authentication normally.

This highlighted the value of checking application logs before assuming the container or deployment had failed.

Useful command:

docker logs --tail 100 jellyfin
Initial Network Exposure

The first Docker Compose configuration used:

ports:
  - "8096:8096/tcp"

Docker displayed the port as:

0.0.0.0:8096->8096/tcp

This means the service was published on all IPv4 interfaces of the Raspberry Pi.

Testing confirmed that Jellyfin was reachable through both:

LAN_IP:8096
TAILSCALE_IP:8096
Firewall Assumption

At this point, UFW was already configured with:

default deny incoming

However, Jellyfin remained accessible through the LAN.

This demonstrated that Docker-published ports cannot be assumed to behave exactly like normal host services under simple UFW rules.

Docker creates its own NAT and forwarding rules for published container ports.

Network Exposure Mitigation

The Jellyfin port binding was changed from:

ports:
  - "8096:8096/tcp"

to:

ports:
  - "TAILSCALE_IP:8096:8096/tcp"

The container was then recreated:

docker compose down
docker compose up -d

After the change, Docker no longer published port 8096 on all interfaces.

Validation

Two access tests were performed.

Tailscale Address
http://TAILSCALE_IP:8096

Result:

Accessible
LAN Address
http://LAN_IP:8096

Result:

Unavailable

This confirmed that Jellyfin was only exposed through the private Tailscale address.

Final Access Model

The resulting network path is:

Trusted Client
     │
     │ Tailscale
     ▼
Raspberry Pi Tailscale IP
     │
     │ TCP/8096
     ▼
Docker
     │
     ▼
Jellyfin

The service is not intentionally exposed through:

Public Internet
Local LAN interface
Security Decisions

The final Jellyfin deployment includes:

containerized deployment
persistent configuration outside the container
persistent cache outside the container
non-root container user
read-only media mount
restrictive media file permissions
private access through Tailscale
port binding limited to the Tailscale address
no router port forwarding
Lessons Learned

This deployment reinforced several concepts:

containers should be treated as disposable
persistent application data should live outside the container
bind mounts can be restricted to read-only access
filesystem permissions should follow least privilege
clean filenames improve media metadata detection
application logs are essential for troubleshooting
0.0.0.0 means a service is exposed on all available IPv4 interfaces
Docker networking can behave differently from normal host services
firewall assumptions should be validated through real connectivity testing
network exposure should be restricted at more than one layer
Future Improvements

Possible future improvements include:

hardware-accelerated transcoding
automated media backups
monitoring Jellyfin availability
collecting Jellyfin logs centrally
ingesting Jellyfin logs into the future SIEM
adding alerts for authentication failures
monitoring unusual access patterns
introducing HTTPS or a reverse proxy if future access requirements change
documenting restore procedures for Jellyfin configuration
