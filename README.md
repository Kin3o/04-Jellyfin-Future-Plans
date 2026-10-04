# Jellyfin Future Plans

This document records work that has not yet been completed. It intentionally separates the verified local deployment from planned storage, resilience, remote-access, and performance improvements.

Return to the [project overview](01-README.md), [build walkthrough](02-Jellyfin-Server-Build-Walkthrough.md), or [troubleshooting lab](03-Jellyfin-Troubleshooting-Lab.md).

## Status labels

| Label | Meaning |
|---|---|
| **Planned** | Explicitly intended for a future phase |
| **Recommended** | A sensible improvement that has not been implemented |
| **Not verified** | Insufficient evidence or testing exists |

## Current baseline

- **Verified:** Jellyfin 12.1 runs in Debian 13.7 VM `100`.
- **Verified:** Local access works at `http://192.168.8.4:8096`.
- **Verified:** *Big Buck Bunny* plays successfully.
- **Verified:** Hardware acceleration is set to `None`.
- **Verified:** No public TCP `8096` port forward exists.
- **Completed but not reverified:** Media directories were configured under `/srv/media`.
- **Temporary:** Media currently occupies the VM's 32 GB system disk.
- **Planned:** Permanent media storage will move to a NAS.

## Priority 1 - Required before adding a large media collection

### 1. Select or build the NAS

**Status: Planned**

Record the following before connecting it to Jellyfin:

- NAS hardware and operating system
- Storage-pool or RAID design
- Usable capacity
- Drive-health monitoring
- Backup destination
- Network address and interface speed
- Whether the NAS provides SMB, NFS, or both

RAID and storage pools improve availability but are not substitutes for backups.

### 2. Choose SMB or NFS

**Status: Planned**

| Option | Best fit | Considerations |
|---|---|---|
| NFS | Linux-to-Linux sharing with straightforward numeric user/group mapping | Requires careful export restrictions and UID/GID planning |
| SMB | Mixed Windows and Linux environments or easier Windows administration | Requires credential handling and CIFS mount options |

The [Debian Reference](https://www.debian.org/doc/manuals/debian-reference/ch10) describes SMB/CIFS and NFS as common network-sharing options. The [Debian Administrator's Handbook](https://www.debian.org/doc/manuals/debian-handbook/sect.nfs-file-server.en.html) includes NFS client and `/etc/fstab` examples.

Do not select a protocol solely because it is familiar. Test permissions, reconnect behavior, filename handling, and performance with the actual NAS.

### 3. Create a dedicated media share

**Status: Planned**

Create one NAS share with clear subdirectories:

```text
media/
├── movies/
├── tv/
└── music/
```

Create a dedicated service identity or tightly scoped NAS account for Jellyfin. Grant only the permissions Jellyfin needs. Read-only media access is safer if Jellyfin should never rename or delete the source files; write access may be needed if lyrics or artwork are intentionally stored beside media.

### 4. Test a temporary mount first

**Status: Recommended**

> **Warning:** Do not add an untested network mount to `/etc/fstab`. A bad entry can delay startup or leave the server waiting for unavailable storage.

Before creating a persistent mount:

1. Install the appropriate Debian client package (`nfs-common` for NFS or `cifs-utils` for SMB).
2. Create a temporary mount point.
3. Mount the NAS share manually.
4. Confirm Jellyfin can read the test media.
5. Restart Jellyfin and repeat playback.
6. Unmount the share cleanly.

Exact mount commands cannot be written until the NAS platform, protocol, share name, address, and credentials are known.

### 5. Create a resilient persistent mount

**Status: Planned**

After the temporary test succeeds, create a persistent mount using `/etc/fstab` or a systemd mount unit. Consider network-aware options such as `_netdev`, `nofail`, and systemd automount behavior where appropriate.

Validate before rebooting:

```bash
findmnt --verify
mount -a
findmnt
```

The exact entry must be generated from the selected NAS protocol and tested environment. Do not paste an invented example into a production `/etc/fstab` file.

### 6. Test NAS failure behavior

**Status: Planned**

Test these cases before relying on the share:

1. Boot the Jellyfin VM while the NAS is online.
2. Boot while the NAS is offline.
3. Start playback, then interrupt NAS connectivity.
4. Restore connectivity and confirm the mount recovers.
5. Confirm library scans do not remove the entire catalog during a temporary outage.
6. Confirm Jellyfin starts even if media storage is unavailable.

### 7. Migrate the temporary library

**Status: Planned**

> **Warning:** Do not delete or move the only copy of media. Copy first, validate the destination, and remove the source only after backup and playback tests succeed.

The intended outcome is for the NAS share to provide the media directories while Debian and Jellyfin remain on the VM disk. Keep these categories separate:

| Data | Planned location |
|---|---|
| Debian operating system | VM system disk |
| Jellyfin program packages | VM system disk |
| Jellyfin database and configuration | VM system disk plus backups |
| Movies, TV shows, and music | NAS |
| Backups | Separate destination, not only the source NAS pool |

## Priority 2 - Security, backups, and reliability

### 1. Create Proxmox VM backups

**Status: Planned**

Configure a Proxmox backup target with enough capacity for VM `100`. Create an initial backup, record its duration and size, and perform a restore test to a non-conflicting VM ID before considering the backup strategy proven.

Avoid storing the only backup on the same 128 GB drive that hosts the running VM.

### 2. Back up Jellyfin separately

**Status: Planned**

The [official Jellyfin backup guide](https://jellyfin.org/docs/general/administration/backup-and-restore/) identifies these default Debian package locations:

```text
/var/lib/jellyfin
/etc/jellyfin
```

Jellyfin 10.11 and later also provides a built-in backup feature. A built-in backup is not sufficient if the resulting archive remains only on the same VM disk. Copy tested backups to independent storage.

Recommended validation:

1. Create a Jellyfin backup during low activity.
2. Copy it to the backup destination.
3. Record the Jellyfin version.
4. Restore in an isolated test environment.
5. Confirm users, libraries, metadata, and playback settings return.

### 3. Monitor storage capacity

**Status: Recommended**

Track at least:

```bash
df -h /
df -h /tmp
du -sh /var/lib/jellyfin
du -sh /srv/media
```

Set warning thresholds before the VM or NAS is close to full. Jellyfin metadata, logs, backups, and transcoding cache can grow even when the media itself is stored elsewhere.

### 4. Create non-administrator Jellyfin users

**Status: Planned**

Create separate accounts for everyday playback. Grant access only to appropriate libraries. Keep the Jellyfin administrator account for configuration work, and do not permit ordinary users to delete media unless that behavior is intentional.

### 5. Establish an update routine

**Status: Planned**

Before updates:

1. Read release notes.
2. Confirm backups are current.
3. Record the current Debian and Jellyfin versions.
4. Apply Debian security updates.
5. Update Jellyfin from its configured official repository.
6. Confirm the service starts.
7. Test library browsing and playback.
8. Review logs for new errors.

Avoid unattended major-version changes until backup restoration has been tested.

## Priority 3 - Clients, performance, and remote access

### 1. Install and test Jellyfin clients

**Status: Planned**

Install appropriate clients on televisions, phones, tablets, or computers. Test:

- Sign-in and Quick Connect behavior
- Direct play of common media formats
- Subtitle playback
- Resume position
- Audio-track selection
- Multiple simultaneous streams
- LAN performance over wired and wireless connections

### 2. Record a direct-play and transcoding baseline

**Status: Planned**

Before enabling GPU acceleration, record:

- Media codec and resolution
- Client type
- Direct Play, Remux, or Transcode status
- CPU usage
- Memory usage
- Startup delay
- Buffering behavior
- Number of simultaneous streams

This baseline shows whether hardware acceleration is actually needed and provides evidence that a later change improved performance.

### 3. Identify the processor and GPU

**Status: Not verified**

Run on the Proxmox host:

```bash
lscpu | grep 'Model name'
lspci -nn | grep -Ei 'vga|display|3d'
```

Record the exact Intel i5 model and graphics device. Do not infer codec support from the OptiPlex model alone.

### 4. Plan Intel Quick Sync access

**Status: Planned; compatibility not verified**

Jellyfin documents Intel Quick Sync as a supported Linux acceleration method in its [hardware-acceleration guide](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/). Enabling it in this VM may require host firmware settings, IOMMU configuration, and GPU passthrough or another deliberately selected design.

> **Warning:** GPU passthrough can remove the device from the Proxmox host or make the VM difficult to recover if configured incorrectly. Preserve console access and take a tested backup first.

Only enable hardware acceleration after:

1. Identifying the exact CPU and iGPU.
2. Confirming codec support.
3. Choosing an appropriate Proxmox GPU-access method.
4. Confirming the device appears inside Debian.
5. Confirming Jellyfin can access the render device.
6. Testing actual playback and transcoding.

Until then, keep **Dashboard > Playback > Transcoding > Hardware acceleration** set to `None`.

### 5. Add Tailscale remote access

**Status: Planned**

Do not expose TCP `8096` directly to the public Internet as the default remote-access design. Tailscale can provide authenticated access between the Jellyfin VM and approved client devices without a public router port forward.

The [official Tailscale media-server guide](https://tailscale.com/docs/use-cases/personal-or-at-home-use/access-nas-media-file-servers?tab=linux) documents accessing Jellyfin through a Tailscale address or hostname on port `8096`.

Planned validation:

1. Install Tailscale in the Jellyfin VM from official packages or instructions.
2. Authenticate it to the intended tailnet.
3. Install Tailscale on one remote client.
4. Test Jellyfin through the VM's Tailscale hostname or address.
5. Verify TCP `8096` remains closed on the public router.
6. Review Tailscale access controls before adding more users or devices.

## Recommended implementation order

1. Identify the NAS platform and address.
2. Choose SMB or NFS.
3. Create and permission the NAS media share.
4. Test a temporary mount from Debian.
5. Test NAS outage behavior.
6. Create and validate persistent mounting.
7. Copy and validate media on the NAS.
8. Configure and restore-test Proxmox backups.
9. Configure and restore-test Jellyfin backups.
10. Create normal Jellyfin playback accounts.
11. Test clients and record a performance baseline.
12. Identify the exact CPU and iGPU.
13. Evaluate hardware acceleration only if the baseline justifies it.
14. Add and validate Tailscale remote access.
15. Establish recurring update and capacity reviews.

Do not mark an item complete merely because it was configured. Mark it complete only after its expected behavior and recovery path have been tested.
