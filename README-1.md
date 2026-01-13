# Nokia 2.2 (TA-1183 / wasp) — Bootloader Unlock via MTKClient + Root with Magisk (Android 11, MT6761)

> **Tested device:** Nokia 2.2 TA-1183 (codename: `wasp`), MediaTek **MT6761**, Android 11, LK fastboot, A/B slots.  
> **Result:** Bootloader unlocked (`unlocked: yes`, `secure: no`) + Magisk root (patched `boot_b`).

---

## ⚠️ Disclaimer
This guide performs low-level operations on MediaTek partitions. You can **soft-brick**, bootloop, or lose data.  
Follow steps exactly. **Do not write** to dangerous partitions (preloader/lk/tee/md1img) unless you know what you're doing.

---

## ✅ What this guide does
1. Boot into **BROM** mode and connect using **MTKClient**
2. Dump GPT and **backup critical partitions**
3. Unlock bootloader by patching **seccfg** (`SecCfgV4`)
4. Root using **Magisk patched boot image** (`boot_b`)

---

## 0) Requirements

### Hardware
- Nokia 2.2 (TA-1183 / MT6761)
- USB data cable
- PC/Laptop (Linux recommended)

### Software
- Linux (Mint/Ubuntu Live USB recommended)
- `adb`, `fastboot`
- Python 3 + pip
- `mtkclient` by **bkerler**
- Official **Magisk** APK

---

## 1) Install tools (Linux)

```bash
sudo apt update
sudo apt install -y git python3 python3-pip android-tools-adb android-tools-fastboot libusb-1.0-0 usbutils
```

Clone and install MTKClient:

```bash
git clone https://github.com/bkerler/mtkclient
cd mtkclient
pip3 install -r requirements.txt
```

---

## 2) Enter BROM mode (MediaTek)
Typical Nokia 2.2 method:

1. Power off phone fully
2. Hold **VOL+ ** (if not detected, use VOL+ VOL- with combo)
3. Plug USB while holding key

Verify device is detected:

```bash
lsusb
```

You should see a MediaTek/Nokia device (VID usually `0e8d`).

---

## 3) Mount Windows partition for backups (recommended)

If you lack storage on Live USB, mount Windows `C:` (example partition `nvme0n1p3`):

```bash
lsblk
sudo mkdir -p /mnt/win
sudo mount /dev/nvme0n1p3 /mnt/win
sudo mkdir -p /mnt/win/mtk_backups
sudo chmod -R 777 /mnt/win/mtk_backups
df -h /mnt/win
```

✅ In Windows this folder will be:
- `C:\mtk_backups`

---

## 4) Dump GPT partition table

```bash
cd ~/mtkclient
sudo python3 mtk.py printgpt | tee /mnt/win/mtk_backups/gpt.txt
```

---

## 5) Backup critical partitions (MUST)

### Core backups
```bash
sudo python3 mtk.py r seccfg  /mnt/win/mtk_backups/seccfg.bin
sudo python3 mtk.py r proinfo /mnt/win/mtk_backups/proinfo.bin
sudo python3 mtk.py r nvram   /mnt/win/mtk_backups/nvram.bin
sudo python3 mtk.py r nvdata  /mnt/win/mtk_backups/nvdata.bin
sudo python3 mtk.py r nvcfg   /mnt/win/mtk_backups/nvcfg.bin
sudo python3 mtk.py r frp     /mnt/win/mtk_backups/frp.bin
```

### Recommended (needed for root / recovery)
```bash
sudo python3 mtk.py r boot_a   /mnt/win/mtk_backups/boot_a.img
sudo python3 mtk.py r boot_b   /mnt/win/mtk_backups/boot_b.img
sudo python3 mtk.py r vbmeta_a /mnt/win/mtk_backups/vbmeta_a.img
sudo python3 mtk.py r vbmeta_b /mnt/win/mtk_backups/vbmeta_b.img
```

Verify backups:
```bash
ls -lh /mnt/win/mtk_backups
sync
```

---

## 6) Unlock bootloader (seccfg unlock)

```bash
sudo python3 mtk.py da seccfg unlock
```

Expected:
- `SecCfgV4 - hwtype found: V4`
- `Successfully wrote seccfg.` ✅

Reset:
```bash
sudo python3 mtk.py reset
```

Disconnect USB when it says to.

---

## 7) Verify unlock state (fastboot)

```bash
adb reboot bootloader
fastboot getvar unlocked
fastboot getvar secure
fastboot getvar all
```

Expected:
- `unlocked: yes`
- `secure: no`
- Current slot likely: `current-slot: b`

Notes:
- `fastboot oem device-info` may fail on Nokia: **normal**
- `fastboot flashing get_unlock_ability` may still return 0: **ignore**

---

## 8) Install Magisk (official)

Download official APK from topjohnwu GitHub Releases:
- https://github.com/topjohnwu/Magisk/releases

If Package Manager crashes on phone, install via ADB from PC:

```bash
adb install -r Magisk-v30.6.apk
```

---

## 9) Patch boot image using Magisk

### Push boot image to phone (slot B example)
```bash
adb push /mnt/win/mtk_backups/boot_b.img /sdcard/Download/boot_b.img
```

On phone:
- Magisk → Install → **Select and Patch a File**
- Select `/sdcard/Download/boot_b.img`

Magisk creates:
- `/sdcard/Download/magisk_patched-30600_XXXX.img`

Verify:
```bash
adb shell ls -lh /sdcard/Download/magisk_patched-*.img
```

Pull patched image:
```bash
adb pull /sdcard/Download/magisk_patched-*.img .
```

---

## 10) Flash patched boot to active slot

Check slot:
```bash
adb reboot bootloader
fastboot getvar current-slot
```

If slot is `b`:
```bash
fastboot flash boot_b magisk_patched-30600_*.img
fastboot reboot
```

---

## 11) Verify root

Open Magisk app → should show **Installed**.

ADB test:
```bash
adb shell su -c id
```

Expected:
- `uid=0(root)`

---

## 🔧 Bootloop / Recovery (Safe restore)

If bootloop occurs after patched boot:
```bash
adb reboot bootloader
fastboot flash boot_b /mnt/win/mtk_backups/boot_b.img
fastboot reboot
```

---

## ❌ Do NOT flash these unless you fully understand
These can cause **hard brick** if wrong:
- `preloader`
- `lk`
- `tee`
- `md1img`
- anything outside boot/vbmeta/dtbo unless verified

---

## Credits
- MTKClient — B.Kerler: https://github.com/bkerler/mtkclient
- Magisk — topjohnwu: https://github.com/topjohnwu/Magisk
