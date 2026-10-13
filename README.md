<div align="center">

# Pi RAID 5 NAS

**A budget network-attached storage server built on a Raspberry Pi 4 with Linux software RAID 5**

![Raspberry Pi](https://img.shields.io/badge/Raspberry%20Pi%204-A22846?style=for-the-badge&logo=raspberrypi&logoColor=white)
![Debian](https://img.shields.io/badge/Raspberry%20Pi%20OS-64--bit-A81D33?style=for-the-badge&logo=debian&logoColor=white)
![RAID 5](https://img.shields.io/badge/RAID-5-2E86C1?style=for-the-badge)
![Samba](https://img.shields.io/badge/Samba-SMB-4CAF50?style=for-the-badge)
![NFS](https://img.shields.io/badge/NFS-Linux-FF9800?style=for-the-badge)

*227 GB of fault-tolerant storage for about $50*

[Features](#-features) •
[Hardware](#-hardware) •
[Installation](#-installation) •
[Usage](#-usage) •
[Troubleshooting](#-troubleshooting) •
[Roadmap](#-roadmap)

</div>

---

## About

Commercial NAS devices typically cost $200 or more. This project turns a Raspberry Pi 4 and three USB flash drives into a working NAS with RAID 5 redundancy, accessible from Windows, macOS, and Linux over the local network.

It was built as a project for **CNIT 17600: Information Technology Architectures** at **Purdue University** to get hands-on experience with storage architecture, Linux administration, and network services.

## Features

- **RAID 5 redundancy** – survives a single drive failure without data loss
- **Samba (SMB)** – native file sharing for Windows and macOS
- **NFS** – native file sharing for Linux clients
- **User authentication** – share access restricted to approved users
- **Auto-mount on boot** – array assembles and mounts automatically
- **Web dashboard** – system and storage monitoring through Cockpit
- **Low cost** – about $50 in parts beyond the Pi

## Architecture

```
        ┌───────────────────────────────────────┐
        │   Clients: Windows · macOS · Linux    │
        └───────────────────┬───────────────────┘
                            │  SMB / NFS over Ethernet
        ┌───────────────────▼───────────────────┐
        │        Raspberry Pi 4 (4 GB RAM)      │
        │  ┌─────────────────────────────────┐  │
        │  │   Samba  ·  NFS  ·  Cockpit     │  │
        │  ├─────────────────────────────────┤  │
        │  │   ext4 filesystem → /mnt/raid   │  │
        │  ├─────────────────────────────────┤  │
        │  │   mdadm RAID 5 → /dev/md0       │  │
        │  └─────────────────────────────────┘  │
        └───────────────────┬───────────────────┘
                            │  USB 3.0
          ┌─────────────────┼─────────────────┐
     ┌────▼────┐       ┌────▼────┐       ┌────▼────┐
     │  sda    │       │  sdb    │       │  sdc    │
     │ 128 GB  │       │ 128 GB  │       │ 128 GB  │
     └─────────┘       └─────────┘       └─────────┘
```

### How RAID 5 works here

Data is striped across all three drives, and a parity block (calculated with XOR) rotates between them:

| Stripe | Drive 1 | Drive 2 | Drive 3 |
|:------:|:-------:|:-------:|:-------:|
| 1 | Data A | Data B | **Parity (A⊕B)** |
| 2 | Data C | **Parity (C⊕D)** | Data D |
| 3 | **Parity (E⊕F)** | Data E | Data F |

If any one drive fails, its blocks can be rebuilt from the other two. For example, `B = A ⊕ Parity`.

## Hardware

| Component | Details |
|---|---|
| Single-board computer | Raspberry Pi 4 Model B, 4 GB RAM |
| Storage | 3 × 128 GB USB 3.0 flash drives |
| Connectivity | Unpowered USB 3.0 hub, Ethernet cable |
| Boot media | 32 GB+ microSD card |
| Power | Official 5V/3A USB-C power supply |

> **Note:** Flash drives are fine for learning, but have limited write endurance. For long-term use, consider SSDs or HDDs (HDDs will need a powered USB hub).

## Installation

### Prerequisites

Flash **Raspberry Pi OS Lite (64-bit)** with [Raspberry Pi Imager](https://www.raspberrypi.com/software/). In the customization settings, set your username and password, configure Wi-Fi if needed, and **enable SSH**.

### 1. Update and install mdadm

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install mdadm -y
```

### 2. Identify your drives

```bash
lsblk
```

You should see three drives of roughly equal size (for example `sda`, `sdb`, `sdc`).

### 3. Create the RAID 5 array

> **Warning:** This permanently erases all data on the selected drives.

```bash
sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sda /dev/sdb /dev/sdc
```

The initial sync runs in the background and can take several hours. The array is usable during this time.

```bash
cat /proc/mdstat
```

### 4. Format and mount

```bash
sudo mkfs.ext4 -F /dev/md0
sudo mkdir -p /mnt/raid
sudo mount /dev/md0 /mnt/raid
```

### 5. Make it persistent across reboots

```bash
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
echo '/dev/md0 /mnt/raid ext4 defaults,noatime 0 2' | sudo tee -a /etc/fstab
```

### 6. Set up Samba

```bash
sudo apt install samba samba-common-bin smbclient -y
sudo mkdir -p /mnt/raid/shared
sudo chown <username>:<username> /mnt/raid/shared
```

Add the share to the bottom of `/etc/samba/smb.conf`:

```ini
[NAS_Share]
   path = /mnt/raid/shared
   writeable = yes
   create mask = 0775
   directory mask = 0775
   public = no
   valid users = <username>
```

Set a Samba password and start the service:

```bash
sudo smbpasswd -a <username>
sudo systemctl restart smbd
sudo systemctl enable smbd
```

### 7. Set up NFS

```bash
sudo apt install nfs-kernel-server -y
```

Add an export to `/etc/exports`, restricted to your local subnet:

```
/mnt/raid/shared 192.168.1.0/24(rw,sync,no_subtree_check)
```

```bash
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
sudo systemctl enable nfs-kernel-server
```

### 8. Install the web dashboard (optional)

```bash
sudo apt install cockpit -y
sudo systemctl enable --now cockpit.socket
```

## Usage

Find the Pi's IP address:

```bash
hostname -I
```

| Platform | How to connect |
|---|---|
| **Windows** | File Explorer → `\\<pi-ip>\NAS_Share` |
| **macOS** | Finder → Go → Connect to Server → `smb://<pi-ip>/NAS_Share` |
| **Linux** | `sudo mount -t nfs <pi-ip>:/mnt/raid/shared /mnt/nas` |
| **Dashboard** | `http://<pi-ip>:9090` |

### Health checks

```bash
cat /proc/mdstat                 # quick array status
sudo mdadm --detail /dev/md0     # detailed status and drive states
df -h /mnt/raid                  # storage usage
```

### Testing fault tolerance

Simulate a drive failure and confirm the array keeps working:

```bash
sudo mdadm --manage /dev/md0 --fail /dev/sda     # mark a drive as failed
sudo mdadm --detail /dev/md0                     # shows "clean, degraded"
sudo mdadm --manage /dev/md0 --remove /dev/sda   # remove it
sudo mdadm --manage /dev/md0 --add /dev/sda      # add it back to rebuild
watch cat /proc/mdstat                           # watch the rebuild
```

> Let the initial sync finish before running this test.

## Troubleshooting

<details>
<summary><b>Mount fails with "wrong fs type, bad superblock"</b></summary>

The array has no filesystem yet. Run `sudo mkfs.ext4 -F /dev/md0` before mounting.
</details>

<details>
<summary><b>Boot loop or emergency mode after installing OpenMediaVault</b></summary>

On this build, OpenMediaVault caused service dependency failures at boot, including timeouts waiting for `/dev/md0`. Reflashing and configuring Samba and NFS manually was faster and more stable, and made it clearer how each service works.
</details>

<details>
<summary><b>"Network error" when connecting from Windows</b></summary>

1. Confirm Samba is serving the share: `smbclient -L localhost -U <username>`
2. Confirm the Pi is reachable: `ping <pi-ip>`
3. Double-check credentials. The Samba password is set separately with `smbpasswd`.
4. Some campus and dorm networks block device-to-device traffic. If so, try a different network or a personal router.
</details>

<details>
<summary><b>Wi-Fi won't connect (<code>raspi-config</code> or <code>wpa_cli</code> fails)</b></summary>

Recent Raspberry Pi OS releases use NetworkManager instead of `wpa_supplicant`/`dhcpcd`. Configure Wi-Fi in Raspberry Pi Imager before flashing, or use:

```bash
sudo nmcli dev wifi connect "<SSID>" password "<password>"
```
</details>

<details>
<summary><b>SSH says "Connection refused" or login fails</b></summary>

Newer images don't include a default `pi` user. Use the username you set in Raspberry Pi Imager, and make sure SSH was enabled before flashing.
</details>

<details>
<summary><b>"Unable to locate package webmin"</b></summary>

Webmin isn't in the default Debian repositories. Cockpit is a simpler alternative available through `apt`.
</details>

## Lessons Learned

- **Configure networking before flashing.** Setting Wi-Fi, SSH, and the username in Raspberry Pi Imager avoids hours of headaches.
- **Know your network.** Restricted networks can block SMB between devices regardless of how well the server is configured.
- **Manual beats GUI for learning.** Configuring Samba and NFS by hand was more stable and more educational.
- **RAID is not a backup.** It protects against drive failure, not accidental deletion, corruption, or losing the whole device.

## Security Notes

This is a lab setup. Before relying on it:

- Restrict NFS exports to specific hosts or subnets
- Avoid world-writable permissions such as `chmod 777`
- Never expose SMB (port 445) directly to the internet. Use a VPN such as WireGuard or Tailscale for remote access.

## Roadmap

- [ ] Secure remote access with Tailscale
- [ ] Email alerts on drive failure via `mdadm --monitor`
- [ ] Automated off-device backups with `rsync` or `restic`
- [ ] Replace flash drives with SSDs and benchmark the difference

## References

- [mdadm manual](https://linux.die.net/man/8/mdadm)
- [Raspberry Pi documentation](https://www.raspberrypi.com/documentation/)
- [Samba documentation](https://www.samba.org/samba/docs/)
- [Create a RAID volume on Raspberry Pi – stewright.me](https://www.stewright.me/create-raid-volume-raspberry-pi/)

---

<div align="center">

Built for **CNIT 17600** at **Purdue University**

</div>
