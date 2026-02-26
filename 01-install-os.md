# Step 1: Install Raspberry Pi OS and Configure Network Access

## Hardware

- Raspberry Pi 4 Model B (8 GB RAM)
- microSD card (64 GB)
- USB microSD card reader
- Mac or PC for flashing

## Download Raspberry Pi Imager

Install **Raspberry Pi Imager** on your Mac/PC:

- **macOS:** `brew install --cask raspberry-pi-imager`
- **Linux:** `sudo apt install rpi-imager`
- **Any OS:** download from https://www.raspberrypi.com/software/

## Flash the SD Card

1. Insert the microSD card into your card reader and plug it into your computer.

2. Open **Raspberry Pi Imager**.

3. Click **Choose Device** → select **Raspberry Pi 4**.

4. Click **Choose OS** → **Raspberry Pi OS (other)** → **Raspberry Pi OS Lite (64-bit)**.
   This installs the Trixie (Debian 13) based headless image — no desktop, minimal footprint,
   ideal for a Kubernetes node.

5. Click **Choose Storage** → select your microSD card.

6. Click **Next**. The Imager will ask if you want to apply OS customisation — click **Edit Settings**.

### OS Customisation Settings

**General tab:**

| Setting            | Value                                    |
| ------------------ | ---------------------------------------- |
| Hostname           | `rara`                                   |
| Username           | `ra`                                     |
| Password           | *(choose a strong password)*             |
| WiFi SSID          | `hehe`                                   |
| WiFi Password      | *(your WiFi password)*                   |
| WiFi Country       | *(your country code, e.g. US, DE, GB)*   |
| Locale / Timezone  | *(your timezone, e.g. Europe/Berlin)*    |
| Keyboard layout    | `us`                                     |

**Services tab:**

| Setting        | Value                            |
| -------------- | -------------------------------- |
| Enable SSH     | **Yes**                          |
| Auth method    | **Password authentication**      |

> **Tip — SSH keys instead of password:**
> If you already have an SSH key pair (`~/.ssh/id_ed25519.pub`), select
> "Allow public-key authentication only" and paste your public key.
> This is more secure and lets you skip typing a password every time.
>
> Generate a key if you don't have one:
> ```bash
> ssh-keygen -t ed25519 -C "ra@rara"
> ```

7. Click **Save**, then **Yes** to apply settings, and **Yes** to confirm writing.

8. Wait for the write + verification to complete. Eject the card.

## First Boot

1. Insert the microSD card into the Pi.
2. Connect power (USB-C).
3. Wait **60–90 seconds** for the first boot to complete. The Pi will:
   - Resize the filesystem to fill the entire SD card
   - Configure the user account, hostname, and WiFi
   - Start the SSH server
   - Register itself on the local network via mDNS

## Connect via SSH

From your Mac/PC terminal:

```bash
ssh ra@rara.local
```

If it doesn't resolve immediately, wait another 30 seconds — the Avahi mDNS daemon
needs a moment to announce the hostname after first boot.

### How `rara.local` Works (mDNS)

You are **not** running a custom DNS server. The `.local` resolution uses **mDNS
(Multicast DNS)**, which works like this:

1. The Pi runs **Avahi** (installed by default on Raspberry Pi OS). On boot, Avahi
   broadcasts the hostname `rara` on the local network using multicast UDP on port 5353.

2. Your Mac (or Linux PC with `avahi-daemon` / `nss-mdns`) listens for these
   announcements and resolves `rara.local` to the Pi's current IP address.

3. No router configuration, no DNS server, no static IP required — it just works on
   any network as long as both devices are on the same subnet.

> **Windows note:** mDNS works on Windows 10+ with the Bonjour service (installed
> automatically with iTunes, or standalone from Apple). Most modern Windows machines
> resolve `.local` out of the box.

### Troubleshooting Connection Issues

**Can't resolve `rara.local`:**

```bash
# Find the Pi by scanning the network (replace with your subnet)
# macOS:
dns-sd -B _ssh._tcp local.
# or install nmap:
nmap -sn 192.168.86.0/24 | grep -B2 "Raspberry"
```

**Can SSH by IP but not by hostname:**

```bash
# Check Avahi is running on the Pi
systemctl status avahi-daemon

# If not installed (unlikely on RPi OS):
sudo apt install avahi-daemon
sudo systemctl enable --now avahi-daemon
```

**SSH key warning after re-flash:**

If you previously connected to `rara.local` with an older OS, your SSH client
will complain about a changed host key. Fix it:

```bash
ssh-keygen -R rara.local
```

## Verify the System

Once connected via SSH, run these quick checks:

```bash
# Confirm hostname
hostname

# Confirm architecture (should be aarch64)
uname -m

# Confirm OS version (should show trixie)
cat /etc/os-release | head -4

# Confirm memory
free -h

# Confirm disk
df -h /

# Confirm WiFi connection
nmcli dev status

# Confirm internet access
ping -c 3 google.com
```

Expected output summary:

| Check        | Expected                              |
| ------------ | ------------------------------------- |
| hostname     | `rara`                                |
| uname -m     | `aarch64`                             |
| OS           | Debian GNU/Linux 13 (trixie)          |
| RAM          | ~7.6 GiB total                        |
| Disk         | ~59 GB (ext4), ~5 GB used             |
| WiFi         | `wlan0` connected                     |
| Internet     | ping succeeds                         |

## Set Up SSH Key Authentication (if not done during flash)

If you used password auth during flash and want to switch to key-based auth:

```bash
# On your Mac/PC — copy your public key to the Pi
ssh-copy-id ra@rara.local

# Test key-based login
ssh ra@rara.local

# On the Pi — disable password auth (optional, improves security)
sudo sed -i 's/^#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl reload sshd
```

---

**Next:** [02-prepare-linux.md](./02-prepare-linux.md) — prepare the OS for Kubernetes.
