# postmarketOS-on-mido
postmarketOS on the Xiaomi Redmi Note 4/4X (mido): lk2nd + extlinux boot chain, custom lk 2nd build (UBL panel fix), SD/eMMC install, autoboot server setup.

<a id="top"></a>

# postmarketOS on the Xiaomi Redmi Note 4 / 4X (mido)

Getting postmarketOS running on **mido** (Snapdragon 625) with **lk2nd** and
**extlinux** — from a blank phone with an unofficial bootloader unlock (UBL) to
a headless mini server that powers itself on when power is applied.

**Language / Bahasa:** [English](#english) · [Bahasa Indonesia](#bahasa-indonesia)

| | |
|---|---|
| Device | Xiaomi Redmi Note 4 / 4X — `mido` (MSM8953 / Snapdragon 625, aarch64) |
| pmOS device package | `device-qcom-msm8953` (generic port; there is no `device-xiaomi-mido`) |
| Kernel | `linux-postmarketos-qcom-msm8953` |
| Boot chain | stock ABL/UBL → **lk2nd** (boot partition) → `extlinux.conf` → kernel |
| Known blocker | a generic UBL reports panel `qcom,mdss_dsi_r69006_1080p_cmd`, which lk2nd does not list for mido → custom lk2nd needed |
| Install paths | microSD (`install --disk`) or eMMC (`install` + `flash_rootfs`) |
| Access | `ssh <user>@172.16.42.1` over USB networking |
| Docs | English (primary) below, Bahasa Indonesia (secondary) after it |

Helper scripts used in this guide live in this folder:
`build-lk2nd-mido.sh`, `patch-lk2nd-mido.py`, `patch-lk2nd-autoboot.py`,
`cek-sd-pmos.sh`, `add-pmos-debug-shell.sh`.
Standalone versions of the two languages: `GUIDE-pmos-mido-EN.md`,
`PANDUAN-pmos-mido-ID.md`.

---

<a id="english"></a>

# English

> Primary language. Sections 1–14: device facts, fastboot modes, lk2nd build,
> installation (SD/eMMC), first boot, verification, power-on autoboot, server
> hardening, debugging, traps, scripts, condensed command sequence.

Written from an actual install on a Redmi Note 4X (Snapdragon 625) with an
**unofficial bootloader unlock (UBL)**, from a blank device to a headless mini
server that powers itself on when power is applied.

Path in one line:

    pmbootstrap init -> build a custom lk2nd -> install to SD/eMMC ->
    flash lk2nd from the STOCK fastboot -> boot -> verify -> server hardening

---

## 1. Device facts and boot chain

- Codename: **mido** (Redmi Note 4 / 4X), MSM8953 / Snapdragon 625, aarch64.
- Current pmaports has **no** `device-xiaomi-mido` package. mido is served by the
  generic **`device-qcom-msm8953`** (kernel `linux-postmarketos-qcom-msm8953`).
  In `pmbootstrap init`: vendor **qcom**, codename **msm8953**.
- This device uses **lk2nd** (a secondary bootloader) and **extlinux**, not a
  `boot.img`: `deviceinfo_generate_extlinux_config="true"`,
  `deviceinfo_boot_filesystem="ext2"`.
- Boot chain:

      PMIC (hardware)
        -> PBL/SBL1 (SoC firmware)
          -> ABL/aboot (the aboot partition; on this phone the unofficial UBL)
            -> lk2nd (the boot partition)
              -> pmOS kernel + initramfs (loaded by lk2nd via extlinux.conf)
                -> pmOS userspace (OpenRC/systemd)

- Partitions used by pmOS:
  - `pmOS_boot` = ext2, 512 MB: `/extlinux/extlinux.conf`, `/vmlinuz`,
    `/initramfs`, DTBs (`msm8953-xiaomi-mido.dtb`).
  - `pmOS_root` = ext4: the system.
  - For an eMMC install both live **inside** the `userdata` partition (the image
    carries its own partition table); lk2nd publishes the nested subpartitions
    and the initramfs handles them via `losetup -Pf`.

lk2nd limits when booting extlinux (matters for desktop installs):

| Item | Limit |
|---|---|
| initramfs | < 16 MiB |
| kernel | < 32 MiB |
| kernel + initramfs + dtb | < 50 MiB (128 MiB on msm8960) |

---

## 2. The two fastboot modes (do not mix them up)

| Mode | How to enter | Used for |
|---|---|---|
| **Stock fastboot** (ABL/UBL) | Power + Volume Down from off, press immediately | flashing **lk2nd**, recovery |
| **lk2nd fastboot** | Power on, wait for the vibration, then hold Volume Down | flashing the **pmOS system**, dumping the lk2nd log |

Hard rule: **never flash lk2nd from lk2nd's own fastboot.** lk2nd redirects
writes to the `boot` partition by +512 KiB so it cannot overwrite itself, so a
lk2nd image flashed that way lands at the wrong offset and bricks the
bootloader. pmbootstrap refuses with `You are currently running lk2nd`.

---

## 3. Host prerequisites

- `pmbootstrap` 3.x (`pmbootstrap --version`).
- `android-tools` (fastboot) and `make`.
- To build lk2nd (Fedora):

      sudo dnf install -y make arm-none-eabi-gcc-cs arm-none-eabi-binutils-cs \
                          dtc python3-libfdt android-tools

- Disk space: the install image is roughly 1.3 GB.

---

## 4. Initialise pmbootstrap

    pmbootstrap init
    # Vendor        : qcom
    # Device        : msm8953
    # Channel       : v26.06 (or the stable channel offered)
    # UI            : - 'console'  -> headless server (lightest)
    #                 - 'xfce4'    -> desktop
    # Service mgr   : openrc (default) or systemd
    # Username/password: write it down, you need it for SSH

Config is stored in `~/.config/pmbootstrap_v3.cfg`. Every command it ran, with
output, is in `~/.local/var/pmbootstrap/log.txt` — the best evidence source when
diagnosing.

---

## 5. Build a custom lk2nd (required for a generic UBL)

### When this is needed

lk2nd identifies the device **only by the panel name** reported by the previous
bootloader (`lk2nd,match-panel`). lk2nd lists five mido panels:

    qcom,mdss_dsi_ili9885_boe_fhd_video    -> xiaomi,boe-ili9885
    qcom,mdss_dsi_nt35532_fhd_video        -> xiaomi,nt35532
    qcom,mdss_dsi_nt35596_tianma_fhd_video -> xiaomi,tianma-nt35596
    qcom,mdss_dsi_otm1911_fhd_video        -> xiaomi,otm1911
    qcom,mdss_dsi_r63350_ebbg_fhd_video    -> xiaomi,ebbg-r63350

A generic UBL (cross-flashed from another device) reports the reference panel
**`qcom,mdss_dsi_r69006_1080p_cmd`**, which is not in that list. Result:

    Failed to find matching lk2nd device node: -1
    The dtb-files for this device is not set
    Failed to parse extlinux.conf
    Bootable file system not found. Reverting to android boot.
    ERROR: Invalid boot image header
    Could not do normal boot. Reverting to fastboot mode.

Symptom: the phone looks like it does nothing. Verify where the string comes
from:

    strings -a <UBL-file>  | grep -i mdss_dsi    # the panel name comes from here
    strings -a lk2nd.img   | grep -i xiaomi,mido # panels known to lk2nd

### How to build

    git clone --depth 1 --branch 23.1 \
        https://github.com/msm8916-mainline/lk2nd.git lk2nd-src

    # 1. add an alias node for the panel name the UBL reports
    python3 patch-lk2nd-mido.py xiaomi,tianma-nt35596 edt,edt-ft5406

    # 2. (optional) autoboot: menu continues into the OS after N seconds
    python3 patch-lk2nd-autoboot.py 5

    # 3. build
    make -C lk2nd-src TOOLCHAIN_PREFIX=arm-none-eabi- lk2nd-msm8953
    # output: lk2nd-src/build-lk2nd-msm8953/lk2nd.img

`build-lk2nd-mido.sh` does all three steps for five panel variants (one image
per panel), applies the autoboot patch, and writes the results to `lk2nd-build/`.

### Choosing the panel variant

`display_fastboot_menu()` starts with `if (!fb) return;`. Therefore:

- **Wrong** panel -> no framebuffer -> the menu is skipped -> the OS still boots,
  but the panel stays black in pmOS as well.
- **Correct** panel -> the lk2nd menu is visible -> that is the signal.

So flash variants one at a time and treat **"menu/countdown appears"** as the
proof of a correct panel.

### `touchscreen-compatible`

Each panel entry also maps the touchscreen driver. Default is `edt,edt-ft5406`
(Focaltech). For Goodix units build with:

    TOUCH=goodix,gt917d ./build-lk2nd-mido.sh

---

## 6. Installing the system

### Option A — microSD card (fastest to test)

    lsblk -o NAME,SIZE,FSTYPE,TRAN,MODEL      # confirm it is the card, not a disk
    sudo umount /dev/sdX1 2>/dev/null         # pmbootstrap refuses mounted disks
    pmbootstrap install --disk=/dev/sdX       # answer 'y' (ERASES the whole card)

Verify the card before removing it:

    sudo ./cek-sd-pmos.sh /dev/sdX

Expect: p1 `pmOS_boot` (ext2), p2 `pmOS_root` (ext4),
`extlinux/extlinux.conf` + `vmlinuz` + `initramfs` + `msm8953-xiaomi-mido.dtb`,
`pmos_boot_uuid`/`pmos_root_uuid` matching the real partitions, initramfs < 16 MiB.

### Option B — eMMC (recommended for permanent use)

    pmbootstrap install                        # no --disk -> produces the image
    # phone: enter lk2nd fastboot (power on, hold Volume Down after the vibration)
    pmbootstrap flasher flash_rootfs           # writes the image to userdata

lk2nd finds the subpartitions inside `userdata`; the initramfs mounts them via a
loop device. If a microSD is still inserted lk2nd may pick it first — remove the
card when booting from eMMC.

`pmbootstrap flasher flash_kernel` is **not usable** on this device (no `boot.img`
because it boots via extlinux). Ignore that line in pmbootstrap's summary.

---

## 7. Flash lk2nd and first boot

    # STOCK fastboot (Power + Volume Down from off)
    fastboot flash boot lk2nd-build/lk2nd-mido-tianma-nt35596.img
    fastboot reboot

Booting:

1. Power on (hold no keys).
2. If the lk2nd menu shows: Volume Down once, then Power.
   - On lk2nd 22.0 the initial selection is **Reboot** and there is **no
     timeout**, so pressing Power immediately reboots instead of booting the OS.
   - With the autoboot patch: a 5 second countdown, then it boots by itself.
3. If no menu appears (panel not matched yet): lk2nd continues booting directly.

---

## 8. Verifying the boot

On the host:

    lsusb                      # a 18d1:d001 gadget appears (pmOS default VID/PID)
    ip -br a                   # new interface, usually enp0s20f... / usb0
    ping -c3 172.16.42.1       # the pmOS device IP
    ssh ditzz@172.16.42.1      # password set during pmbootstrap install

If ping works but SSH says `connection refused`, the device is still in the
**initramfs** (the handover to the system has not happened). See Debugging.

---

## 9. Booting automatically when power is applied (mini server)

Three separate layers — do not conflate them:

| Symptom | Layer | Control |
|---|---|---|
| Phone does not turn on when power is applied | hardware/PMIC | no setting (hardware behaviour) |
| Powers on, but stops at a charging screen | ABL/UBL (before lk2nd) | `fastboot oem off-mode-charge 0` |
| Powers on, but sits in the lk2nd menu | lk2nd | autoboot patch (countdown) |

Charger mode only triggers when: cold boot **and** not a hard reset **and** the
power key was not pressed **and** USB power is actually present. Therefore:

- Battery inserted without a cable -> boots directly (the flag is irrelevant).
- Cable present during cold boot -> the flag matters.
- Software reboot (`reboot`, `panic=10`) -> never enters charger mode.

Set the flag (persistent, written to the `devinfo` partition):

    # STOCK fastboot
    fastboot oem off-mode-charge 0
    fastboot getvar charger-screen-enabled    # must read 0
    fastboot reboot

Note: this flag does **not** make the phone turn on; it only removes the charging
screen delay after power-on. Turning on when the battery is inserted is PMIC
behaviour.

---

## 10. Server hardening (OpenRC, console)

    # auto-reboot 10 seconds after a kernel panic
    sudo mkdir -p /etc/kernel-cmdline.d
    printf 'panic=10\n' | sudo tee /etc/kernel-cmdline.d/90-server.conf

    # optional: drop the splash and show boot messages on screen
    sudo tee /etc/kernel-cmdline.d/91-nosplash.conf >/dev/null <<'EOF'
    -quiet
    -splash
    loglevel=6
    EOF

    generate-kernel-cmdline        # preview the merged cmdline (no root needed)
    sudo mkinitfs                  # rebuild initramfs + /boot/extlinux/extlinux.conf

    rc-status                      # OpenRC services
    sudo rc-service sshd status
    ssh-copy-id ditzz@172.16.42.1  # key-based login

Cmdline layering: `/usr/lib/kernel-cmdline.d/` (distro & device) then
`/etc/kernel-cmdline.d/` (user overrides). A leading `-` removes a parameter.
Always run `sudo mkinitfs` after changing the cmdline.

---

## 11. Debugging

### initramfs debug shell

Add `pmos.debug-shell` to the cmdline (via `/etc/kernel-cmdline.d/` +
`mkinitfs`, or by editing `/boot/extlinux/extlinux.conf` directly when the system
cannot boot at all). The initramfs then stops at a shell reachable:

- on screen (with `buffyboard` for on-screen typing),
- as a getty on USB serial `/dev/ttyACM0`,
- via `telnet 172.16.42.1` (port 23).

Useful commands:

    cat /pmOS_init.log       # initramfs log
    cat /proc/cmdline
    blkid                    # partitions the kernel can see
    pmos_continue_boot       # continue booting
    pmos_logdump             # expose the log as a USB mass-storage device

### Reading boot messages

- `Loading` and `Starting` are the initramfs's last messages before
  `switch_root`. Stuck on **Starting** means kernel+initramfs+rootfs all worked —
  the problem is in the system after the handover.
- A "stuck splash" is **not** proof of a plymouth problem: the splash covers the
  console, hiding a panic or freeze behind it. Remove it (`-splash` +
  `loglevel=6`) to see the real last message.
- systemd installs: `journalctl -b -1` for the previous boot,
  `systemctl --failed`, `systemd-analyze blame`.
- OpenRC installs: `rc-status`, `rc-status --crashed`, `dmesg`.

### lk2nd log

    # in lk2nd fastboot
    fastboot oem log && fastboot get_staged lk2nd.log

This log separates "lk2nd failed to identify/boot" from "lk2nd handed off and the
OS is unhappy".

---

## 12. Common traps

1. **Appender/AnyKernel3 zips are not needed.** They append 5 magic bytes so a
   stock bootloader accepts an *Android boot image*. The pmOS kernel is never
   read by the bootloader (lk2nd loads it), so they are irrelevant.
2. **`Persist.zip` is not needed for pmOS.** It is the vendor `persist` partition
   image (Android calibration data). Never wipe it, but do not reflash it for pmOS.
3. **`flash_kernel` is useless** on this extlinux device.
4. **initramfs size**: desktop installs (plymouth/DE) come close to lk2nd's
   16 MiB limit. Watch `ls -l /boot/initramfs`.
5. **`pmbootstrap install --disk` erases the whole disk** and aborts if any
   partition is mounted (`/dev/sdX1 is mounted! Will not attempt to format this!`).
   Identify the disk by size/model first.
6. **Missing image**: `The rootfs has not been generated yet` means the image is
   gone (the native chroot was rebuilt). Run `pmbootstrap install` again.
7. **A one-off `losetup: ... failed to set up loop device`** during install is
   transient (pmbootstrap runs `modprobe loop` and retries immediately).
8. **microSD vs eMMC**: lk2nd boots the first extlinux it finds. With two installs
   (card + userdata), remove the card to select eMMC.

---

## 13. Helper scripts in this folder

| Script | Purpose |
|---|---|
| `cek-sd-pmos.sh /dev/sdX` | verify an `install --disk` card (extlinux, UUIDs, dtb, size limits) |
| `patch-lk2nd-mido.py [panel] [touch]` | add the UBL panel alias node to the mido entry |
| `patch-lk2nd-autoboot.py [seconds]` | make the lk2nd menu continue booting after N seconds |
| `build-lk2nd-mido.sh` | build five lk2nd variants + autoboot (`AUTOBOOT_SECS`, `TOUCH`) |
| `add-pmos-debug-shell.sh /dev/sdX [on\|off]` | toggle `pmos.debug-shell` on a card |

---

## 14. Condensed command sequence (from zero)

    # host
    sudo dnf install -y pmbootstrap android-tools make arm-none-eabi-gcc-cs \
                        arm-none-eabi-binutils-cs dtc python3-libfdt
    pmbootstrap init                 # qcom / msm8953 / ui / openrc
    ./build-lk2nd-mido.sh            # output: lk2nd-build/*.img

    pmbootstrap install --disk=/dev/sdX     # or 'pmbootstrap install' + flash_rootfs
    sudo ./cek-sd-pmos.sh /dev/sdX

    # phone: stock fastboot (Power+VolDown)
    fastboot flash boot lk2nd-build/lk2nd-mido-<panel>.img
    fastboot oem off-mode-charge 0
    fastboot reboot

    # phone: power on -> pick Continue (or wait for the countdown)
    ssh ditzz@172.16.42.1

    # inside the system
    sudo mkdir -p /etc/kernel-cmdline.d
    printf 'panic=10\n' | sudo tee /etc/kernel-cmdline.d/90-server.conf
    sudo mkinitfs

---

<a id="bahasa-indonesia"></a>

# Bahasa Indonesia

> Bahasa sekunder. Bagian 1–14: fakta perangkat, mode fastboot, build lk2nd,
> instalasi (SD/eMMC), boot pertama, verifikasi, autoboot saat diberi daya,
> hardening server, debugging, jebakan, skrip, urutan perintah ringkas.

Panduan ini disusun dari proses pemasangan nyata di Redmi Note 4X (Snapdragon 625)
dengan **bootloader hasil unlock tidak resmi (UBL)**, dari nol sampai sistem bisa
dipakai sebagai mini server yang menyala sendiri saat diberi daya.

Ringkasan jalur:

    pmbootstrap init -> build lk2nd khusus -> install ke SD/eMMC ->
    flash lk2nd dari STOCK fastboot -> boot -> verifikasi -> hardening server

---

## 1. Fakta perangkat dan rantai boot

- Codename: **mido** (Redmi Note 4 / 4X), SoC MSM8953 / Snapdragon 625, aarch64.
- Sejak pmaports terbaru **tidak ada** paket `device-xiaomi-mido`. mido dilayani
  paket generik **`device-qcom-msm8953`** (kernel `linux-postmarketos-qcom-msm8953`).
  Di `pmbootstrap init`: vendor **qcom**, codename **msm8953**.
- Device ini memakai **lk2nd** (secondary bootloader) dan **extlinux**, bukan
  `boot.img`. `deviceinfo_generate_extlinux_config="true"` dan
  `deviceinfo_boot_filesystem="ext2"`.
- Rantai boot:

      PMIC (hardware)
        -> PBL/SBL1 (firmware SoC)
          -> ABL/aboot (partisi aboot; pada HP ini = file UBL tidak resmi)
            -> lk2nd (partisi boot)
              -> kernel + initramfs pmOS (dibaca lk2nd lewat extlinux.conf)
                -> sistem pmOS (OpenRC/systemd)

- Partisi yang dipakai pmOS:
  - `pmOS_boot` = ext2 512 MB, isi: `/extlinux/extlinux.conf`, `/vmlinuz`,
    `/initramfs`, DTB (`msm8953-xiaomi-mido.dtb`).
  - `pmOS_root` = ext4, isi sistem.
  - Kalau di-install ke eMMC, keduanya berada **di dalam** partisi `userdata`
    (image membawa tabel partisinya sendiri); lk2nd mem-publish subpartisinya
    dan initramfs menanganinya lewat `losetup -Pf`.

Batas ukuran lk2nd saat boot extlinux (penting untuk instalasi desktop):

| Item | Batas |
|---|---|
| initramfs | < 16 MiB |
| kernel | < 32 MiB |
| kernel + initramfs + dtb | < 50 MiB (128 MiB di msm8960) |

---

## 2. Dua mode fastboot (jangan tertukar)

| Mode | Cara masuk | Untuk apa |
|---|---|---|
| **Stock fastboot** (ABL/UBL) | Power + Volume Down dari kondisi mati, tekan segera | flash **lk2nd**, flash recovery |
| **lk2nd fastboot** | Nyalakan, tunggu getar, baru tahan Volume Down | flash **sistem pmOS**, dump log lk2nd |

Aturan keras: **jangan pernah flash lk2nd dari lk2nd fastboot.** lk2nd menggeser
penulisan ke partisi `boot` sebesar 512 KiB supaya dirinya tidak tertimpa, jadi
lk2nd akan tertulis di offset yang salah dan bootloader-nya rusak. pmbootstrap
menolak dengan pesan `You are currently running lk2nd`.

---

## 3. Prasyarat di komputer

- `pmbootstrap` 3.x (`pmbootstrap --version`).
- `android-tools` (fastboot) dan `make`.
- Untuk membangun lk2nd (Fedora):

      sudo dnf install -y make arm-none-eabi-gcc-cs arm-none-eabi-binutils-cs \
                          dtc python3-libfdt android-tools

- Ruang disk: image instalasi sekitar 1,3 GB.

---

## 4. Inisialisasi pmbootstrap

    pmbootstrap init
    # Vendor        : qcom
    # Device        : msm8953
    # Channel       : v26.06 (atau versi stabil yang tersedia)
    # UI            : pilih sesuai kebutuhan
    #                 - 'console'  -> untuk server headless (paling ringan)
    #                 - 'xfce4'    -> untuk desktop
    # Service mgr   : openrc (default) atau systemd
    # Username/password: catat, dipakai untuk SSH

Catatan: konfigurasi tersimpan di `~/.config/pmbootstrap_v3.cfg`. Semua perintah
yang pernah dijalankan beserta outputnya ada di `~/.local/var/pmbootstrap/log.txt`
— sumber bukti terbaik saat mendiagnosis.

---

## 5. Build lk2nd khusus (wajib untuk UBL generik)

### Kapan langkah ini perlu

lk2nd mengidentifikasi perangkat **hanya dari nama panel** yang dilaporkan
bootloader sebelumnya (`lk2nd,match-panel`). Daftar panel mido di lk2nd ada 5:

    qcom,mdss_dsi_ili9885_boe_fhd_video    -> xiaomi,boe-ili9885
    qcom,mdss_dsi_nt35532_fhd_video        -> xiaomi,nt35532
    qcom,mdss_dsi_nt35596_tianma_fhd_video -> xiaomi,tianma-nt35596
    qcom,mdss_dsi_otm1911_fhd_video        -> xiaomi,otm1911
    qcom,mdss_dsi_r63350_ebbg_fhd_video    -> xiaomi,ebbg-r63350

Bootloader UBL generik (hasil cross-flash dari perangkat lain) melaporkan panel
referensi **`qcom,mdss_dsi_r69006_1080p_cmd`** yang tidak ada di daftar itu.
Akibatnya:

    Failed to find matching lk2nd device node: -1
    The dtb-files for this device is not set
    Failed to parse extlinux.conf
    Bootable file system not found. Reverting to android boot.
    ERROR: Invalid boot image header
    Could not do normal boot. Reverting to fastboot mode.

Gejalanya: HP seolah "tidak terjadi apa-apa". Cek string penyebabnya:

    strings -a <file-UBL> | grep -i mdss_dsi    # nama panel datang dari sini
    strings -a lk2nd.img  | grep -i xiaomi,mido # daftar panel di lk2nd

### Cara build

    git clone --depth 1 --branch 23.1 \
        https://github.com/msm8916-mainline/lk2nd.git lk2nd-src

    # 1. tambahkan node alias untuk nama panel yang dilaporkan UBL
    python3 patch-lk2nd-mido.py xiaomi,tianma-nt35596 edt,edt-ft5406

    # 2. (opsional) autoboot: menu otomatis lanjut boot OS setelah N detik
    python3 patch-lk2nd-autoboot.py 5

    # 3. build
    make -C lk2nd-src TOOLCHAIN_PREFIX=arm-none-eabi- lk2nd-msm8953
    # hasil: lk2nd-src/build-lk2nd-msm8953/lk2nd.img

`build-lk2nd-mido.sh` melakukan ketiga langkah di atas sekaligus untuk 5 varian
panel (satu image per panel), menerapkan patch autoboot, dan menyimpan hasilnya
di `lk2nd-build/`.

### Cara memilih varian panel

`display_fastboot_menu()` diawali `if (!fb) return;`. Artinya:

- Panel **salah** -> lk2nd tidak dapat framebuffer -> menu dilewati -> OS tetap
  boot, tetapi layar juga hitam di pmOS.
- Panel **benar** -> menu lk2nd terlihat -> itu penanda panel yang cocok.

Karena itu: flash varian satu per satu, dan **tampilnya menu/hitung mundur**
adalah sinyal panel benar.

### Token `touchscreen-compatible`

Setiap entri panel memetakan juga driver sentuh. Default `edt,edt-ft5406`
(Focaltech). Kalau unit Anda memakai sentuh Goodix, build dengan:

    TOUCH=goodix,gt917d ./build-lk2nd-mido.sh

---

## 6. Instalasi sistem

### Opsi A — ke kartu microSD (paling cepat untuk uji coba)

    lsblk -o NAME,SIZE,FSTYPE,TRAN,MODEL      # pastikan device kartu, bukan HDD
    sudo umount /dev/sdX1 2>/dev/null         # pmbootstrap menolak disk yang ter-mount
    pmbootstrap install --disk=/dev/sdX       # jawab 'y' (SEMUA ISI KARTU HAPUS)

Verifikasi kartu sebelum dicabut:

    sudo ./cek-sd-pmos.sh /dev/sdX

Yang harus terlihat: p1 `pmOS_boot` (ext2), p2 `pmOS_root` (ext4), ada
`extlinux/extlinux.conf` + `vmlinuz` + `initramfs` + `msm8953-xiaomi-mido.dtb`,
UUID `pmos_boot_uuid`/`pmos_root_uuid` cocok dengan partisi aslinya, dan
initramfs < 16 MiB.

### Opsi B — ke eMMC (disarankan untuk pemakaian permanen)

    pmbootstrap install                        # tanpa --disk -> menghasilkan image
    # HP: masuk lk2nd fastboot (nyalakan, tahan Volume Down setelah getar)
    pmbootstrap flasher flash_rootfs           # menulis image ke partisi userdata

lk2nd akan menemukan subpartisi di dalam `userdata`; initramfs me-mount-nya
lewat loop device. Kalau kartu SD masih terpasang, lk2nd bisa memilih kartu
lebih dulu — cabut kartu saat boot dari eMMC.

`pmbootstrap flasher flash_kernel` **tidak berlaku** untuk device ini (tidak ada
`boot.img` karena memakai extlinux). Abaikan baris itu di ringkasan pmbootstrap.

---

## 7. Flash lk2nd dan boot pertama

    # STOCK fastboot (Power + Volume Down dari kondisi mati)
    fastboot flash boot lk2nd-build/lk2nd-mido-tianma-nt35596.img
    fastboot reboot

Boot:

1. Nyalakan HP (tanpa menahan tombol).
2. Kalau menu lk2nd tampil: Volume Down 1x lalu Power 1x.
   - Di lk2nd 22.0, sorotan awal ada di **Reboot** dan **tidak ada timeout**,
     jadi menekan Power langsung = reboot, bukan boot OS.
   - Dengan patch autoboot: hitung mundur 5 detik lalu boot sendiri.
3. Kalau menu tidak tampil (panel belum cocok): lk2nd langsung melanjutkan boot.

---

## 8. Verifikasi setelah boot

Di komputer:

    lsusb                      # muncul gadget 18d1:d001 (VID/PID default pmOS)
    ip -br a                   # interface baru, biasanya enp0s20f... / usb0
    ping -c3 172.16.42.1       # IP perangkat pmOS
    ssh ditzz@172.16.42.1      # password dari pmbootstrap install

Kalau `ping` jalan tetapi SSH `connection refused`: sistem masih di **initramfs**
(belum serah terima ke sistem). Lihat bagian Debugging.

---

## 9. Boot otomatis saat diberi daya (untuk mini server)

Ada tiga lapisan terpisah — jangan dicampur:

| Gejala | Lapisan | Pengaturan |
|---|---|---|
| HP tidak menyala saat diberi daya | hardware/PMIC | tidak ada setelan (perilaku hardware) |
| Nyala, tapi berhenti di layar charging | ABL/UBL (sebelum lk2nd) | `fastboot oem off-mode-charge 0` |
| Nyala, tapi diam di menu lk2nd | lk2nd | patch autoboot (hitung mundur) |

Charger mode hanya aktif bila: cold boot **dan** bukan hard reset **dan** tombol
power tidak ditekan **dan** USB benar-benar ada daya. Jadi:

- Pasang baterai tanpa kabel -> langsung boot (flag tidak diperlukan).
- Ada kabel saat cold boot -> perlu flag 0.
- Reboot software (`reboot`, `panic=10`) -> tidak pernah masuk charger mode.

Set flag (persisten, ditulis ke partisi `devinfo`):

    # STOCK fastboot
    fastboot oem off-mode-charge 0
    fastboot getvar charger-screen-enabled    # harus 0
    fastboot reboot

Catatan: flag ini **tidak** membuat HP menyala; ia hanya menghapus jeda charger
screen setelah HP menyala. Menyala-saat-baterai-dipasang adalah perilaku PMIC.

---

## 10. Hardening server (OpenRC, console)

    # auto-reboot 10 detik setelah kernel panic
    sudo mkdir -p /etc/kernel-cmdline.d
    printf 'panic=10\n' | sudo tee /etc/kernel-cmdline.d/90-server.conf

    # opsional: matikan splash + tampilkan pesan boot di layar
    # (menghapus 'splash'/'quiet' dari config distro/device, menambah loglevel)
    sudo tee /etc/kernel-cmdline.d/91-nosplash.conf >/dev/null <<'EOF'
    -quiet
    -splash
    loglevel=6
    EOF

    generate-kernel-cmdline        # cek hasil gabungan (tanpa sudo)
    sudo mkinitfs                  # regenerasi initramfs + /boot/extlinux/extlinux.conf

    rc-status                      # service OpenRC
    sudo rc-service sshd status
    ssh-copy-id ditzz@172.16.42.1  # login pakai key

Konfigurasi cmdline bertingkat: `/usr/lib/kernel-cmdline.d/` (distro & device)
lalu `/etc/kernel-cmdline.d/` (override pengguna). Awalan `-` menghapus
parameter. Setelah mengubah cmdline, wajib `sudo mkinitfs`.

---

## 11. Debugging

### Debug shell initramfs

Tambahkan `pmos.debug-shell` ke cmdline (lewat `/etc/kernel-cmdline.d/` +
`mkinitfs`, atau edit `/boot/extlinux/extlinux.conf` langsung kalau sistem belum
bisa boot sama sekali). Saat boot, initramfs berhenti di shell:

- di layar (dengan `buffyboard` untuk mengetik di layar sentuh),
- getty di USB serial `/dev/ttyACM0`,
- `telnet 172.16.42.1` (port 23).

Perintah penting:

    cat /pmOS_init.log       # log initramfs
    cat /proc/cmdline
    blkid                    # partisi yang terlihat kernel
    pmos_continue_boot       # lanjut booting
    pmos_logdump             # ekspos log sebagai perangkat mass storage

### Membaca pesan boot

- `Loading` dan `Starting` adalah pesan terakhir initramfs sebelum
  `switch_root`. Kalau layar berhenti di **Starting**, kernel+initramfs+rootfs
  sudah berhasil — masalahnya di sistem setelah serah terima.
- Layar "stuck" di splash **bukan** bukti masalah plymouth: splash menutupi
  console, sehingga panic/freeze tersembunyi di belakangnya. Hapus splash
  (`-splash` + `loglevel=6`) untuk melihat pesan asli.
- Instalasi systemd: `journalctl -b -1` untuk boot sebelumnya,
  `systemctl --failed`, `systemd-analyze blame`.
- Instalasi OpenRC: `rc-status`, `rc-status --crashed`, `dmesg`.

### Log lk2nd

    # di lk2nd fastboot
    fastboot oem log && fastboot get_staged lk2nd.log

Log ini memisahkan dua dunia: "lk2nd gagal mengenali/mem-boot" vs "lk2nd sudah
menyerahkan boot ke OS".

---

## 12. Jebakan yang sering terjadi

1. **Appender/AnyKernel3 zip tidak diperlukan.** Zip itu menambahkan 5 byte
   penanda agar bootloader stok menerima *boot image Android*. Kernel pmOS tidak
   pernah dibaca bootloader (lk2nd yang membacanya), jadi tidak relevan.
2. **`Persist.zip` tidak diperlukan untuk pmOS.** Itu citra partisi `persist`
   (kalibrasi vendor untuk Android). Jangan pernah di-wipe, tapi juga tidak perlu
   di-flash ulang untuk pmOS.
3. **`flash_kernel` tidak ada gunanya** untuk device extlinux ini.
4. **Ukuran initramfs**: instalasi desktop (plymouth/DE) bisa mendekati batas
   16 MiB lk2nd. Pantau `ls -l /boot/initramfs`.
5. **`pmbootstrap install --disk` menghapus seluruh disk** dan gagal kalau ada
   partisi yang ter-mount (`/dev/sdX1 is mounted! Will not attempt to format
   this!`). Identifikasi disk dari ukuran/model dulu.
6. **Image hilang**: pesan `The rootfs has not been generated yet` berarti image
   sudah tidak ada (chroot native dibuat ulang). Jalankan `pmbootstrap install`.
7. **Halaman baru `losetup: ... failed to set up loop device`** saat install
   bersifat sementara (pmbootstrap langsung `modprobe loop` dan mencoba lagi).
8. **Kartu SD vs eMMC**: lk2nd mem-boot extlinux pertama yang ditemukan. Kalau
   ada dua instalasi (kartu + userdata), cabut kartu untuk memilih eMMC.

---

## 13. Skrip pendukung di folder ini

| Skrip | Fungsi |
|---|---|
| `cek-sd-pmos.sh /dev/sdX` | verifikasi kartu hasil `install --disk` (extlinux, UUID, dtb, batas ukuran) |
| `patch-lk2nd-mido.py [panel] [touch]` | tambah node alias panel UBL ke entri mido di lk2nd |
| `patch-lk2nd-autoboot.py [detik]` | menu lk2nd otomatis lanjut boot setelah N detik |
| `build-lk2nd-mido.sh` | build 5 varian lk2nd + autoboot (`AUTOBOOT_SECS`, `TOUCH`) |
| `add-pmos-debug-shell.sh /dev/sdX [on\|off]` | aktifkan/nonaktifkan `pmos.debug-shell` di kartu |

---

## 14. Urutan perintah ringkas (dari nol)

    # komputer
    sudo dnf install -y pmbootstrap android-tools make arm-none-eabi-gcc-cs \
                        arm-none-eabi-binutils-cs dtc python3-libfdt
    pmbootstrap init                 # qcom / msm8953 / ui / openrc
    ./build-lk2nd-mido.sh            # hasil: lk2nd-build/*.img

    pmbootstrap install --disk=/dev/sdX     # atau 'pmbootstrap install' + flash_rootfs
    sudo ./cek-sd-pmos.sh /dev/sdX

    # HP: stock fastboot (Power+VolDown)
    fastboot flash boot lk2nd-build/lk2nd-mido-<panel>.img
    fastboot oem off-mode-charge 0
    fastboot reboot

    # HP: nyalakan -> pilih Continue (atau tunggu hitung mundur)
    ssh ditzz@172.16.42.1

    # di dalam sistem
    sudo mkdir -p /etc/kernel-cmdline.d
    printf 'panic=10\n' | sudo tee /etc/kernel-cmdline.d/90-server.conf
    sudo mkinitfs

---

[⬆ Back to top / Kembali ke atas](#top)
