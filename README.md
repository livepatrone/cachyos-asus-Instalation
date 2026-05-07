# linux-ASUS G14/16-guide
My Asus G14 Cachyos installation guide and issues
# CachyOS on the ASUS ROG Zephyrus G14 (GA403WW) — Full Setup Guide

> **Hardware:** ASUS ROG Zephyrus G14 GA403WW · AMD Ryzen AI 9 HX · RTX 5080 · Dual-boot Windows 11  
> **OS:** CachyOS (Arch-based) · KDE Plasma · Wayland  
> **Author's note:** This guide documents a real install — including all the mistakes, dead ends, and fixes. If something tripped us up, it's in here.

---

## Table of Contents

1. [Why CachyOS?](#why-cachyos)
2. [Pre-Install Checklist](#pre-install-checklist)
3. [Partition Layout](#partition-layout)
4. [Installation — Bootloader Choice](#installation--bootloader-choice)
5. [Post-Install: Secure Boot with sbctl](#post-install-secure-boot-with-sbctl)
6. [Post-Install: ASUS Stack (asusctl / ROG Control Center)](#post-install-asus-stack)
7. [Post-Install: NVIDIA Drivers](#post-install-nvidia-drivers)
8. [Post-Install: WiFi (MT7925)](#post-install-wifi-mt7925)
9. [Post-Install: GPU MUX](#post-install-gpu-mux)
10. [Post-Install: Gaming Stack](#post-install-gaming-stack)
11. [Post-Install: Audio — Optional (EasyEffects + PipeWire)](#post-install-audio-optional)
12. [Post-Install: Razer Cooling Pad — Optional](#post-install-razer-cooling-pad-optional)
13. [Issues & Fixes Log](#issues--fixes-log)
14. [Final Checklist](#final-checklist)

---

## Why CachyOS?

CachyOS is the top recommendation from [asus-linux.org](https://asus-linux.org) for G14 hardware. Key reasons:

- The G14 kernel patchset is baked directly into the `linux-cachyos` kernel — no separate `linux-g14` needed
- As of March 2025, CachyOS includes the `asus-armory-driver` in their main kernel
- The `linux-cachyos` kernel (7.0.3+) includes the CachyOS scheduler and performance tuning on top of G14 patches — installing `linux-g14` (7.0.2) would actually be a downgrade
- The `g14` extra repo provides pre-built, optimized packages for asusctl and related tooling

---

## Pre-Install Checklist

Before you boot the installer USB:

- [ ] Back up any configs, scripts, or dotfiles you want to keep from your current install
- [ ] If dual-booting Windows, note your Windows partition UUID (`blkid`)
- [ ] Set Windows GPU mode to **Hybrid** in G-Helper before rebooting into the installer
- [ ] Have a USB with the CachyOS ISO ready (verify the checksum)
- [ ] **Before booting the USB — go into BIOS, clear your Secure Boot keys, and make sure Secure Boot is still enabled.** Clearing the keys puts it in Setup Mode — Secure Boot is on but not enforcing any keys, so the unsigned CachyOS USB can boot freely. Do not disable Secure Boot entirely; leave it enabled in Setup Mode. You'll enroll your own keys with sbctl after the install. On ASUS: `Advanced → Secure Boot → Key Management → Clear Secure Boot Keys`, then confirm Secure Boot is still toggled **On**

---

## Partition Layout

This guide assumes a dual-boot with Windows 11 already installed. Before running the CachyOS installer, you need to have free unallocated space on your drive. Use a tool like GParted or the Windows Disk Management utility to shrink your Windows partition and create the following:

> ⚠️ **Your partition names will be different.** The table below uses the partition names from one specific machine as a reference. Your drive may be `/dev/nvme0n1`, `/dev/sda`, or something else entirely. Use `lsblk` or GParted to identify your actual partitions before assigning mount points.

| Purpose | Size | Filesystem | Mount Point | Format? |
|---------|------|------------|-------------|---------|
| Windows EFI (existing) | ~260MB | FAT32 | *(leave alone, do not mount)* | ❌ NO |
| Windows OS (existing) | varies | NTFS | *(leave alone)* | ❌ NO |
| Linux `/boot` | 2–4GB | FAT32 | `/boot` | ✅ Yes |
| Linux `/` (root) | remainder | btrfs | `/` | ✅ Yes |

> ⚠️ **Critical:** Do NOT assign the Windows EFI partition a mount point. Limine boots entirely from the dedicated `/boot` partition — mounting the Windows EFI as `/boot/efi` is unnecessary and was the root cause of multiple failed install attempts on this hardware.

---

## Installation — Bootloader Choice

### Use Limine

When the CachyOS installer asks you to choose a bootloader, select **Limine**. You do not need to run any commands — the installer handles it automatically.

Limine is recommended over GRUB for a few reasons:

- Cleaner, faster boot interface
- Automatically detects and adds your Windows partition to the boot menu — no extra configuration needed
- More straightforward to sign for Secure Boot with sbctl

> ⚠️ **Partition size note:** Your dedicated `/boot` partition should be at least **2–4GB**. Verify the exact requirement against the [Limine documentation](https://limine-bootloader.org) before partitioning.

---

## Post-Install: Secure Boot with sbctl

At this point your BIOS has Secure Boot enabled with no keys enrolled (Setup Mode). `sbctl` is the tool that lets you create your own Secure Boot keys, enroll them into your firmware, and sign your bootloader and kernel so the system trusts them. Once your keys are enrolled and everything is signed, Secure Boot will only allow your signed bootloader and kernel to run — anything unsigned gets blocked.

Do this immediately after the bootloader is working — before installing anything else. Sign the bootloader and kernel while they're in a known good state.

```bash
sudo pacman -S sbctl
sudo sbctl create-keys
sudo sbctl enroll-keys          # Note: do NOT use --firmware-builtin on ASUS hardware
sudo sbctl sign -s /boot/EFI/LIMINE/LIMINE.EFI
sudo sbctl sign -s /boot/vmlinuz-linux-cachyos
sbctl status
```

After signing, reboot to confirm Secure Boot is holding with your new keys.

> ⚠️ **ASUS-specific:** Do not use `--firmware-builtin` when enrolling keys. ASUS firmware handles Microsoft certs separately — using that flag can break Secure Boot enrollment entirely.

> 💡 Any time the kernel updates, `sbctl` will automatically re-sign it if you used `-s` (the save flag) during initial signing.

---

## Post-Install: ASUS Stack

### Add the G14 Repo

```bash
sudo nano /etc/pacman.conf
```

Add at the bottom:

```ini
[g14]
Server = https://arch.asus-linux.org
```

Then update and install:

```bash
sudo pacman -Syu
sudo pacman -S asusctl rog-control-center power-profiles-daemon
```

### Enable the asusd Service

```bash
sudo systemctl enable --now asusd
sudo systemctl enable --now power-profiles-daemon
```

> ⚠️ **Known Issue:** On a fresh install, `asusd` will fail to start with a "directory not found" error.  
> **Fix:**
> ```bash
> sudo mkdir -p /etc/asusd
> sudo systemctl start asusd
> ```

### Battery Charge Limit

```bash
asusctl battery set-limit 80
```

80% is recommended for longevity if you're often plugged in.

### Fan Curves

Enable fan curves on all three profiles via ROG Control Center GUI, or via CLI:

```bash
asusctl fan-curve --mod-profile Quiet
asusctl fan-curve --mod-profile Balanced
asusctl fan-curve --mod-profile Performance
```

### Profile Switching

```bash
asusctl profile set Balanced   # Quiet | Balanced | Performance
asusctl profile get            # check current
```

Fn+F5 cycles through profiles once asusctl is running.

---

## Post-Install: NVIDIA Drivers

### Install

```bash
sudo pacman -S nvidia-open nvidia-laptop-power-cfg
```

### Kernel Parameters

Create `/etc/modprobe.d/nvidia.conf`:

```
options nvidia NVreg_EnableS0ixPowerManagement=1
options nvidia NVreg_DynamicPowerManagement=0x02
```

### Enable NVIDIA Power Services

```bash
sudo systemctl enable nvidia-suspend
sudo systemctl enable nvidia-hibernate
sudo systemctl enable nvidia-resume
sudo systemctl enable nvidia-powerd
```

### S0ix / Sleep

> 💡 S0ix (suspend-to-idle deep sleep) was a known issue on the RTX 5080 with `nvidia-open` in earlier driver versions. It may be resolved in newer drivers — verify on your hardware. To confirm true suspend is working rather than just the display turning off, run `sudo systemctl suspend` and check that the system fully suspends (fans off, minimal power draw) and resumes cleanly.

---

## Post-Install: WiFi (MT7925)

The MT7925 chip has aggressive power saving that causes connectivity issues. Disable it:

**`/etc/NetworkManager/conf.d/wifi-powersave.conf`:**
```ini
[connection]
wifi.powersave = 2
```

**`/etc/modprobe.d/mt7925.conf`:**
```
options mt7925e power_save=0
```

Apply:
```bash
sudo systemctl restart NetworkManager
```

---

### MUX Switch Commands
 
```bash
# List all available armoury attributes including gpu_mux_mode
asusctl armoury get list
 
# Check current MUX mode
asusctl armoury get gpu_mux_mode
# 0 = dedicated/dGPU mode
# 1 = hybrid mode (recommended)
 
# Switch modes (requires reboot)
sudo asusctl armoury set gpu_mux_mode 1   # hybrid
sudo asusctl armoury set gpu_mux_mode 0   # dedicated (gaming, external monitor)
```

### GPU MUX Guard Script *(Optional — if you keep getting black screens)*

If you frequently forget to switch back to Hybrid before rebooting from Windows, this script runs at boot, detects dGPU mode, and automatically corrects it to Hybrid before the display manager loads.

**`/usr/local/bin/gpu-mux-guard.sh`:**
```bash
#!/bin/bash
MUX_PATH="/sys/devices/platform/asus-nb-wmi/gpu_mux_mode"

if [[ ! -f "$MUX_PATH" ]]; then
    logger -t gpu-mux-guard "MUX sysfs path not found, skipping"
    exit 0
fi

MUX_MODE=$(cat "$MUX_PATH")

# Filter false positives (Unknown-1 state)
if [[ "$MUX_MODE" == "0" ]]; then
    logger -t gpu-mux-guard "dGPU mode detected — switching to hybrid and rebooting"
    asusctl armoury set gpu_mux_mode 1
    sleep 2
    systemctl reboot
else
    logger -t gpu-mux-guard "Hybrid mode confirmed (mode=$MUX_MODE), continuing boot"
fi
```

**`/etc/systemd/system/gpu-mux-guard.service`:**
```ini
[Unit]
Description=GPU MUX Mode Guard — auto-correct dGPU to hybrid on boot
DefaultDependencies=no
After=sysinit.target asusd.service
Before=display-manager.service graphical.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/gpu-mux-guard.sh
RemainAfterExit=yes

[Install]
WantedBy=graphical.target
```

```bash
sudo chmod +x /usr/local/bin/gpu-mux-guard.sh
sudo systemctl enable gpu-mux-guard.service
```

Check logs with: `journalctl -t gpu-mux-guard -b`

---

## Post-Install: Gaming Stack

### Install Packages

Open **CachyOS Hello → Apps/Tweaks → Install Gaming packages**. This installs the full gaming stack including Steam, MangoHud, Heroic, Lutris, Proton, Wine, and all required `lib32` compatibility libraries in one shot.

### Windows Partition Mount *(Optional — if you want Steam on Linux to access your Windows Steam library)*

If you have games installed on Windows and want Steam on CachyOS to point at the same library without reinstalling everything, mount your Windows partition first.

Add to `/etc/fstab`:

```
UUID=<your-windows-partition-uuid>  /mnt/windows  ntfs-3g  defaults,uid=1000,gid=1000,dmask=022,fmask=133  0  0
```

Find your Windows partition UUID with `sudo blkid` and look for the NTFS partition. Then:

```bash
sudo mkdir -p /mnt/windows
sudo mount -a
```

Once mounted, add the Windows Steam library path in Steam → Settings → Storage.

> 💡 **Tip:** If a game has a native Linux version, it's worth downloading the Linux version separately into a dedicated Linux Steam library rather than running the Windows version through Proton. Native Linux builds generally perform better and are more stable. You can have both libraries active at the same time — Windows library for your existing installs, Linux library for new ones going forward.

---

## Post-Install: Audio *(Optional)*

### PipeWire Sample Rate Fix (Scarlett 2i2)

After a fresh install, PipeWire may default to a different sample rate than your interface, causing crackling in Discord and other apps.

```bash
mkdir -p ~/.config/pipewire/pipewire.conf.d
nano ~/.config/pipewire/pipewire.conf.d/10-clock.conf
```

```
context.properties = {
    default.clock.rate = 48000
    default.clock.allowed-rates = [ 44100 48000 96000 ]
    default.clock.quantum = 1024
    default.clock.min-quantum = 32
    default.clock.max-quantum = 8192
}
```

```bash
systemctl --user restart pipewire pipewire-pulse wireplumber
```

### EasyEffects Mic Chain

Install EasyEffects and rebuild the chain for the MV7X via Scarlett 2i2:

1. Deep Noise Remover (attenuation 80, output +10dB)
2. Equalizer: HP 120Hz ×4, Bell 220Hz -2dB, Bell 3.5kHz +2dB, High Shelf 10kHz +2dB
3. Gate: 5ms attack / 350ms release, -55dB reduction, -22dB threshold
4. Compressor: 3:1 ratio, knee -6dB, -18dB threshold, makeup +3dB

Enable "Copy input audio buffer" in EasyEffects settings.

In Discord: set input to **EasyEffects Source** and also enable Krisp.

---

## Post-Install: Razer Cooling Pad *(Optional)*

Uses the [razer-coolingpad-linux](https://github.com/Nox1on/razer-coolingpad-linux) project to control fan speed on Razer cooling pads from Linux.

---

**Step 1 — Clone the project**

Pick a permanent location for this — it needs to stay there since the systemd service will point to it.

```bash
git clone https://github.com/Nox1on/razer-coolingpad-linux
cd razer-coolingpad-linux
```

---

**Step 2 — Create and enter the Python virtual environment**

```bash
python3 -m venv venv
source venv/bin/activate
```

Your terminal prompt will now show `(venv)` at the start — this means you're inside the isolated Python environment.

```bash
pip install -r requirements.txt
```

---

**Step 3 — Exit the venv**

> ⚠️ Do this before running any other commands. Staying inside the venv and running unrelated commands can install packages into the wrong environment and break things.

```bash
deactivate
```

The `(venv)` prefix will disappear from your prompt confirming you're out.

---

**Step 4 — Find your cooling pad's USB ID**

Plug in your cooling pad, then run:

```bash
lsusb
```

Look for your Razer device in the list. Note the two values after `ID` — they'll look like `1532:XXXX`. The first part (`1532`) is the vendor ID (Razer), the second is the product ID specific to your model.

---

**Step 5 — Create the udev rule**

This gives your user permission to talk to the device without needing root every time.

```bash
sudo nano /etc/udev/rules.d/99-razer-coolingpad.rules
```

Add this line, replacing `XXXX` with your product ID from Step 4:

```
SUBSYSTEM=="usb", ATTRS{idVendor}=="1532", ATTRS{idProduct}=="XXXX", MODE="0666"
```

Reload udev rules:

```bash
sudo udevadm control --reload-rules && sudo udevadm trigger
```

---

**Step 6 — Create the systemd service**

This runs the cooling pad controller automatically on boot. Replace `<your-username>` with your actual username and update the path to wherever you cloned the project in Step 1.

```bash
sudo nano /etc/systemd/system/razer-coolingpad.service
```

```ini
[Unit]
Description=Razer Cooling Pad Controller
After=multi-user.target

[Service]
User=<your-username>
ExecStart=/home/<your-username>/razer-coolingpad-linux/venv/bin/python3 /home/<your-username>/razer-coolingpad-linux/main.py --config balanced.json
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now razer-coolingpad.service
```

Verify it's running:

```bash
systemctl status razer-coolingpad.service
```

---

**Profiles**

Three fan profiles are available:

| Profile | Description |
|---------|-------------|
| `silent.json` | Low speed, quiet |
| `balanced.json` | Default everyday use |
| `performance.json` | Max airflow — recommended for gaming |

To switch profiles, open the service file:

```bash
sudo nano /etc/systemd/system/razer-coolingpad.service
```

Find the `ExecStart` line and change `balanced.json` to whichever profile you want, for example:

```
ExecStart=...main.py --config performance.json
```

Save the file, then reload and restart the service for the change to take effect:

```bash
sudo systemctl daemon-reload
sudo systemctl restart razer-coolingpad.service
```

---

## Issues & Fixes Log

A running list of every problem encountered and how it was resolved.

---

### ❌ Multiple Bootloader Failures During Install

**Problem:** Multiple failed install attempts with EFI partition overflow errors, systemd-boot size failures, and GRUB install errors.

**Root cause:** Assuming Limine needed to be set up like GRUB — mounting the Windows EFI partition as `/boot/efi`. Limine doesn't need it at all and the 260MB Windows EFI partition is too small to share with a Linux bootloader anyway.

**Fix:** The correct setup uses only two mount points:
- Your Linux `/boot` partition → `/boot` → format it
- Your Linux root partition → `/` → format it

Do not assign your Windows EFI partition any mount point. Limine boots entirely from your dedicated `/boot` partition and never needs to touch it.

---

### ❌ sbctl `--firmware-builtin` Breaks Secure Boot on ASUS

**Problem:** Using `sbctl enroll-keys --firmware-builtin` on ASUS hardware causes Secure Boot enrollment to fail or results in a broken boot environment.

**Fix:** Enroll keys without `--firmware-builtin`. ASUS firmware handles Microsoft certs separately.
```bash
sudo sbctl enroll-keys
```

---

### ❌ asusd Fails to Start on Fresh Install

**Problem:** `systemctl start asusd` fails immediately after install with a config directory error.

**Fix:**
```bash
sudo mkdir -p /etc/asusd
sudo systemctl start asusd
```

---

### ❌ Black Screen When Booting CachyOS from Windows

**Problem:** Switching from Windows with G-Helper set to dGPU/Performance mode causes a black screen in CachyOS — no login, no TTY.

**Root cause:** MUX switch mismatch. The MUX is left routing the display through the dGPU, but CachyOS can't drive the internal screen through it before the NVIDIA driver fully initializes.

**Fix:** Always set G-Helper to **Hybrid mode** before rebooting into CachyOS. If this keeps happening, set up the optional GPU MUX Guard script to auto-correct it on boot.

---

### ❌ Internal Laptop Display Not Initializing Without External Monitor

**Problem:** On first boot after install, the internal screen stays black unless an external monitor is connected.

**Root cause:** The MUX was left in dGPU mode from Windows. This is the same black screen issue described in the MUX section — not a KDE or Wayland problem.

**Fix:** Before booting into CachyOS, make sure Windows is set to Hybrid mode in G-Helper. Once you boot into CachyOS in Hybrid mode the internal display works normally.

---

### ❌ Discord Audio Crackling After Fresh Reinstall

**Problem:** Everyone in Discord sounds crackling/distorted. PipeWire config is not carried over on a fresh install.

**Fix:** Set PipeWire sample rate to match your audio interface (see [Audio section above](#post-install-audio-optional)).

---

### ⚠️ S0ix Sleep (Status Uncertain)

**Problem:** Suspend-to-idle (S0ix) was broken on the RTX 5080 with `nvidia-open` in earlier driver versions.

**Current status:** May be resolved in newer drivers — unconfirmed. To verify, run `sudo systemctl suspend` and confirm the system fully suspends and resumes rather than just blanking the display.

---

### 🔍 Double Lock Screen on Boot (Unresolved)

**Problem:** After booting into CachyOS, two lock screens appear stacked on top of each other. Only one has an actual password field — unlocking that one gets you into the desktop. The other is just sitting behind it with no input.

**Status:** No fix found yet. Suspected to be related to KDE Plasma and SDDM both triggering a lock on session start, but root cause is unconfirmed. If you find a fix, please open an issue or PR.

---

## Final Checklist

Use this after every fresh install to make sure nothing is missed:

- [ ] Partitions set correctly — Linux `/boot` partition mounted at `/boot`, Linux root partition mounted at `/`, Windows EFI partition left unmounted
- [ ] Limine selected in the CachyOS installer
- [ ] Secure Boot: `sbctl` keys created, enrolled (no `--firmware-builtin`), bootloader + kernel signed
- [ ] G14 repo added to `pacman.conf`
- [ ] `asusctl` + `rog-control-center` + `power-profiles-daemon` installed
- [ ] `/etc/asusd` directory created — `asusd` service started and enabled
- [ ] Battery charge limit set to 80%
- [ ] Fan curves enabled on all 3 profiles
- [ ] `nvidia-open` + `nvidia-laptop-power-cfg` installed
- [ ] `/etc/modprobe.d/nvidia.conf` created with Runtime D3 config
- [ ] NVIDIA power services enabled (suspend, hibernate, resume, powerd)
- [ ] WiFi powersave disabled (`wifi-powersave.conf` + `mt7925.conf`)
- [ ] *(Optional)* GPU MUX guard script installed and service enabled if black screen is a recurring issue
- [ ] *(Optional)* Windows partition mounted in `/etc/fstab` if pointing Steam at Windows library
- [ ] Gaming packages installed (`cachyos-gaming-meta` + `cachyos-gaming-applications`)
- [ ] PipeWire sample rate config applied
- [ ] *(Optional)* EasyEffects mic chain rebuilt
- [ ] *(Optional)* Razer cooling pad service set up

---

*Built from real session notes — ASUS ROG Zephyrus G14 GA403WW on CachyOS, May 2026.*
