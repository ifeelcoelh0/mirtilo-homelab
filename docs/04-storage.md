# Storage Configuration

## Objective

Prepare the external HDD for reliable homelab use while preserving the existing personal data already stored on the disk.

The goal was to separate legacy personal storage from Linux-native homelab workloads and create a persistent ext4 area suitable for Docker, logs, projects and future services.

---

## Initial Disk State

The Raspberry Pi detected the external drive as:

```text
/dev/sda

The disk used an MBR/DOS partition table and initially contained three NTFS partitions:

/dev/sda1   549 MB   NTFS   System Reserved
/dev/sda2   ~930 GB  NTFS   Main data partition
/dev/sda3   553 MB   NTFS   Windows recovery partition

The main NTFS partition contained approximately:

Used: ~266 GB
Free: ~665 GB

The existing files were preserved during the storage redesign.

Filesystem Validation

Before resizing the NTFS partition, the drive reported that the filesystem had not been cleanly closed by Windows.

The disk was disconnected safely from the Raspberry Pi and checked on Windows using:

chkdsk X: /f

where X: represented the NTFS data volume.

This ensured the NTFS filesystem was repaired and in a clean state before resizing.

Partition Resize

The large NTFS partition was resized in Windows Disk Management.

The goal was to keep enough space for the existing data while freeing a large section for Linux-native homelab storage.

The final layout became approximately:

/dev/sda1   549 MB    NTFS
/dev/sda2   393 GB    NTFS
Unallocated ~537 GB
/dev/sda3   553 MB    NTFS

The unallocated space was intentionally left unformatted in Windows.

Creating the ext4 Partition

After reconnecting the disk to the Raspberry Pi, a new primary partition was created in the unallocated space.

The new partition became:

/dev/sda4

with a size of approximately:

537 GB

The partition was formatted as ext4 and labelled:

homelab

Example command:

sudo mkfs.ext4 -L homelab /dev/sda4
Why ext4

The homelab workload uses ext4 instead of NTFS because ext4 is better suited to Linux services and Docker workloads.

Relevant advantages include:

native Unix ownership
native Unix permissions
symbolic links
predictable chmod and chown behaviour
better compatibility with Linux applications
better suitability for Docker persistent data

The original NTFS partition remains available for legacy personal files.

Mount Point

The ext4 partition is mounted at:

/mnt/homelab

The mount point was created using:

sudo mkdir -p /mnt/homelab

The partition was initially mounted manually to validate the filesystem and mount point.

Persistent Mounting

The partition UUID was retrieved using:

sudo blkid /dev/sda4

The UUID was then added to:

/etc/fstab

using an entry similar to:

UUID=<EXT4_UUID> /mnt/homelab ext4 defaults,nofail 0 2

Using the filesystem UUID avoids relying on device names such as:

/dev/sda4

which may change between boots.

Validation

After modifying /etc/fstab, systemd was reloaded:

sudo systemctl daemon-reload

The partition was then tested without rebooting:

sudo umount /mnt/homelab
sudo mount -a

The mount was verified using:

findmnt /mnt/homelab

and:

df -hT /mnt/homelab

This confirmed that the ext4 partition could be mounted successfully using the persistent configuration.

Ownership and Permissions

The root of the homelab partition was assigned to the pi user:

sudo chown pi:pi /mnt/homelab

This allows normal homelab administration without requiring sudo for every file operation.

Permissions are managed more specifically at service level when required.

Homelab Directory Structure

The following directory structure was created:

/mnt/homelab/
├── docker/
│   ├── compose/
│   ├── volumes/
│   └── backups/
├── media/
│   └── movies/
├── projects/
└── logs/

The purpose of each area is:

docker/compose — Docker Compose definitions
docker/volumes — persistent application data
docker/backups — service backup data
media — media libraries used by services such as Jellyfin
projects — future homelab project data
logs — centralized or exported logs
Docker Storage Strategy

The Docker engine itself remains in the default location:

/var/lib/docker

on the Raspberry Pi system storage.

Persistent application data is stored under:

/mnt/homelab/docker/volumes

This approach keeps Docker's internal runtime data separate from service-specific persistent data.

It also makes future backups and migrations easier.

Media Permissions

Movie files stored under:

/mnt/homelab/media/movies

were configured with standard non-executable file permissions:

644

Example:

chmod 644 /mnt/homelab/media/movies/*.mkv

Media directories use permissions appropriate for traversal and read access.

This avoids unnecessarily permissive modes such as:

777
Final Storage Model

The resulting storage architecture is:

1 TB HDD
│
├── NTFS partition
│   └── existing personal files
│
└── ext4 partition
    └── /mnt/homelab
        ├── Docker persistent data
        ├── media
        ├── projects
        ├── logs
        └── backups

This creates a clear separation between legacy storage and Linux-native homelab workloads.

Lessons Learned

This setup reinforced several important concepts:

filesystems should be validated before resizing partitions
NTFS and ext4 serve different purposes in mixed Windows/Linux environments
partitions should be unmounted before physical removal
UUID-based mounts are more reliable than device-name-based mounts
/etc/fstab changes should be tested with mount -a before rebooting
Linux ownership and permissions matter for Docker and self-hosted services
persistent application data should be separated from disposable container runtime data
filesystem layout should be planned before deploying stateful services
Future Improvements

Possible future improvements include:

implementing automated backups
adding disk health monitoring with SMART
defining retention policies for logs and backups
adding filesystem usage alerts
testing restore procedures
introducing a second backup destination
documenting recovery steps for a failed SD card or HDD
