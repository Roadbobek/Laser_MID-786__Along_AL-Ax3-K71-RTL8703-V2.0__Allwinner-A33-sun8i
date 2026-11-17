# Mainline Linux on the Laser MID-786 Tablet — Complete Build & Boot Guide

This guide documents the exact end-to-end steps we took to bring up **Mainline U-Boot (`2026.10`)**, **Mainline Linux (`7.4.0-rc2`)**, and a **Debian 12 (`bookworm`) `armhf` RootFS** on the **Laser MID-786** tablet, booting from an external MicroSD card to an interactive `tty1` LCD console with a working USB OTG keyboard.

For a dedicated reference of the kernel `.config` additions, Device Tree (`sun8i-a33-q8-tablet.dts`) patch, and future C driver fixes for our upcoming kernel fork, see [KERNEL_AND_DTB_CHANGES.md](file:///C:/Users/Roadb/Documents/Linux%20on%20Laser%20MID-786/KERNEL_AND_DTB_CHANGES.md).

---

## 1. Hardware Specifications & Boot Architecture

### 1.1 Hardware Identification (from PCB Inspection & Stock Android ADB Dumps)
| Component | Specification / Identifier | Notes |
| :--- | :--- | :--- |
| **Retail Model** | Laser MID-786 | Stock OS: Android 4.4.2 (Linux `3.4.39` SDK kernel) |
| **Mainboard Silkscreen** | `AL-Ax3-K71-RTL8703-V2.0 / 20160128` | Manufactured by Along; closest upstream Q8 reference board: `sun8i-a33-q8-tablet` (`AL-AX3-Q8`) |
| **SoC** | Allwinner A33 (`sun8i`, `soc=00001667`) | Quad-Core ARM Cortex-A7 (`armv7l` / `armhf`), Mali-400 MP2 GPU |
| **PMIC** | X-Powers AXP223 (`sunxi-rsb-3a3`) | Connected via Allwinner Reduced Serial Bus (`RSB` @ `0x01f03400`); controls rails, battery charger, and `usb0-vbus` (`drivevbus`) |
| **RAM** | 1 GiB DDR3 (`0x40000000`–`0x7dffffff`) | Two Nanya `NT5CC256M16DP-DI` (4 Gb / 512 MB each), clocked at 456 MHz (`CONFIG_DRAM_ZQ=15291`) |
| **Internal Storage** | SanDisk `SDTNRGAMA-008G` (8 GB) | **Raw Parallel MLC NAND Flash** (TSOP-48), *not* eMMC |
| **Display Panel** | 7-inch `1024x600` TN LCD (`depth:18`) | 6 bits-per-color (RGB666), 51 MHz pixel clock; `PH7` LCD power enable, `PH6` backlight enable, `PH0` PWM |
| **Touchscreen** | Silead `gslX680` (`gsl1680`) | 5-point capacitive touch over I2C (`0x40`) |
| **Wi-Fi / Bluetooth** | Realtek `RTL8703BS` | SDIO 2.0 on `mmc1` (`sunxi-mmc.1`) |
| **Accelerometer** | `mc32x0` | 3-axis accelerometer over I2C |
| **USB Port** | 1x Micro-USB 2.0 OTG (`usb@1c19000`) | Mentor Graphics Inventra MUSB controller + `sun4i-usb-phy` (`phy@1c19400`); used for both FEL/ADB and OTG Host (via Micro-USB OTG adapter) |
| **Serial UART** | Unpopulated | `CONFIG_CONS_INDEX=5` (`UART4` multiplexed on `PB` pins), no dedicated `TX`/`RX`/`GND` debug pads exposed on PCB |

### 1.2 Why We Boot from MicroSD Instead of Internal NAND
The internal SanDisk `SDTNRGAMA-008G` chip is **raw parallel MLC NAND**, not managed eMMC. Allwinner's stock Android `3.4.39` kernel uses a proprietary, closed-source NAND translation layer (`libnand` + Boot0/Boot1) with custom hardware ECC/scrambling. Mainline U-Boot and Mainline Linux do not support booting directly off this proprietary Allwinner MLC NAND layout reliably without risking wear/corruption of the stock Android partitions (`boot`, `system`, `bootloader`).

Conveniently, the Allwinner A33 mask ROM (**BROM**, baked permanently into the silicon at `0xffff0000`) checks **MicroSD (`MMC0` / `sdc0`)** *before* internal NAND (`NAND0`) on every power-on:
1. If a MicroSD card with a valid Allwinner `eGON.BT0` SPL signature at **byte offset `8192` (`8 KiB`, sector 16)** is inserted, the A33 boots completely from the MicroSD card and never touches internal NAND.
2. If you power off, remove the MicroSD card, and power on, the tablet boots straight back into stock Android 4.4.2 untouched.

### 1.3 The 5-Stage ARM Linux Boot Sequence
1. **BROM (Allwinner Mask ROM):** Hardwired in the A33 SoC. Checks `MMC0` (MicroSD) sector 16 (`8 KiB`). Finds our SPL header (`eGON.BT0`), loads the SPL into internal SRAM (`32 KiB`), and jumps to it.
2. **SPL (Secondary Program Loader):** Initializes the AXP223 PMIC voltages over RSB and trains the 1 GiB DDR3 RAM controller (`456 MHz`). Once DRAM is live, SPL loads full U-Boot from the MicroSD card (immediately following SPL) into DRAM and jumps to it.
3. **Das U-Boot (Main Bootloader):** Turns on the `1024x600` LCD (`PH7`/`PH6`/`PH0`), initializes the MicroSD controller (`mmc 0`) and USB OTG controller (`usb@1c19000`), scans Partition 1 (`mmc 0:1`, FAT32), and executes `boot.scr`.
4. **Linux Kernel (`zImage`) + Device Tree Blob (`sun8i-a33-q8-tablet.dtb`) + Modules (`/lib/modules/<release>`):**
   - `zImage`: Self-extracting compressed ARM32 kernel image loaded into RAM at `${kernel_addr_r}` (`0x42000000`).
   - `.dtb`: Compiled binary hardware map loaded at `${fdt_addr_r}` (`0x43000000`) telling the kernel what hardware exists on this specific board and at which memory addresses/GPIOs.
5. **RootFS + `init` (PID 1):** The kernel mounts Partition 2 (`/dev/mmcblk0p2`, `ext4`) as `/` and executes `/sbin/init` (`systemd`), which mounts filesystems from `/etc/fstab`, loads kernel modules from `/lib/modules/...`, and spawns the `getty` login prompt on `tty1`.

---

## 2. Host Environment & Dependencies

### What Needs to Be Done (General)
You need an `x86_64` (or `aarch64`) Linux build host with:
- A 32-bit ARM hard-float cross-compiler (`arm-linux-gnueabihf-gcc`)
- U-Boot & Linux kernel build tools (`make`, `bison`, `flex`, `swig`, `python3-dev`, `libssl-dev`, `device-tree-compiler`, `u-boot-tools`)
- Allwinner USB FEL utility (`sunxi-tools`)
- Root filesystem & formatting tools (`debootstrap`, `qemu-user-static` / `qemu-user` with `binfmt_misc`, `dosfstools`, `e2fsprogs`)

> [!IMPORTANT]
> **No Spaces in Build Path:** Neither U-Boot's nor the Linux kernel's `Makefile` system supports spaces in directory paths. Always place your build workspace in a path without spaces.

### What We Did Specifically (Kali Linux `x86_64` Live USB with Persistence)
```bash
sudo apt update
sudo apt install -y \
  build-essential git gcc-arm-linux-gnueabihf \
  swig python3-dev device-tree-compiler libssl-dev \
  bison flex sunxi-tools debootstrap qemu-user-static \
  u-boot-tools dosfstools e2fsprogs

mkdir -p ~/Laser_MID-786
cd ~/Laser_MID-786
```

---

## 3. Building & Testing Mainline U-Boot

### 3.1 Configure & Compile U-Boot
The Laser MID-786 (`AL-Ax3-K71-RTL8703-V2.0`) matches the mainline U-Boot **`q8_a33_tablet_1024x600_defconfig`** target:
- `CONFIG_DEFAULT_DEVICE_TREE="sun8i-a33-q8-tablet"`
- `CONFIG_DRAM_CLK=456` / `CONFIG_DRAM_ZQ=15291`
- `CONFIG_VIDEO_LCD_MODE="x:1024,y:600,depth:18,pclk_khz:51000,le:159,ri:160,up:22,lo:12,hs:1,vs:1,sync:3,vmode:0"`
- `CONFIG_VIDEO_LCD_DCLK_PHASE=0`
- `CONFIG_VIDEO_LCD_POWER="PH7"` / `CONFIG_VIDEO_LCD_BL_EN="PH6"` / `CONFIG_VIDEO_LCD_BL_PWM="PH0"`
- `CONFIG_REGULATOR_AXP_DRIVEVBUS=y` / `CONFIG_REGULATOR_AXP_USB_POWER=y` / `CONFIG_USB_MUSB_HOST=y`

```bash
cd ~/Laser_MID-786
git clone https://github.com/u-boot/u-boot.git
cd u-boot

# Load the Q8 A33 1024x600 tablet configuration
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- q8_a33_tablet_1024x600_defconfig

# Compile U-Boot (produces u-boot-sunxi-with-spl.bin)
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j$(nproc)
```
This produced `u-boot-sunxi-with-spl.bin` (`551,904` bytes / ~`539 KiB`, version `U-Boot 2026.10-00908-g903c9578eeb3`), which combines both the **SPL** and **U-Boot proper** into a single image ready to be flashed at the `8 KiB` offset. (We also saved a backup copy as [default-q8_a33_defconfig-u-boot-sunxi-with-spl.bin](file:///C:/Users/Roadb/Documents/Linux%20on%20Laser%20MID-786/default-q8_a33_defconfig-u-boot-sunxi-with-spl.bin).)

### 3.2 Optional: Testing U-Boot in RAM via Allwinner FEL Mode
Before touching a MicroSD card, you can test U-Boot directly in RAM over a Micro-USB cable using the A33's built-in **FEL mode**:
1. Power the tablet completely off and unplug the USB cable.
2. Press and **hold Volume Down**.
3. Press and **hold Power for ~2 seconds**, then **rapidly click Power 3–5 times**, and release all buttons. (The screen stays completely black in FEL mode.)
4. Plug the tablet into the Kali PC via Micro-USB and verify FEL detection:
   ```bash
   lsusb | grep -i "1f3a:efe8"
   # Bus 001 Device 005: ID 1f3a:efe8 Allwinner Technology sunxi SoC OTG connector in FEL/flashing mode

   sudo sunxi-fel version
   # AWUSBFEX soc=00001667(A33) 00000001 ver=0001 44 08 scratchpad=00007e00 00000000 00000000
   ```
5. Upload and execute U-Boot in RAM:
   ```bash
   sudo sunxi-fel -v uboot u-boot-sunxi-with-spl.bin
   ```
   The `1024x600` LCD lights up with the U-Boot console and penguin logo!
   *(Note: When booted over FEL, the USB controller remains in FEL/peripheral state, so a USB OTG keyboard will not enumerate until U-Boot is booted standalone from a MicroSD card.)*

---

## 4. MicroSD Card Geometry, Partitioning, & Flashing U-Boot

### 4.1 Why the Exact Disk Byte Layout Matters
Because the Allwinner BROM loads `u-boot-sunxi-with-spl.bin` from raw unpartitioned space starting at **byte `8192` (`8 KiB`)**, the MicroSD card **must** use a classic **MBR (`dos`)** partition table and start Partition 1 at **sector `2048` (`1 MiB`)**:

| Offset Range | Sector Range (512B/sector) | Size | Contents |
| :--- | :--- | :--- | :--- |
| `0 B` – `511 B` | Sector `0` | `512 B` | **MBR (`dos`) Partition Table** (Must **not** be GPT! GPT occupies sectors 2–33 / `1–17 KiB`, which overwrites the SPL at `8 KiB`.) |
| `8 KiB` – `~547 KiB` | Sector `16` – `~1094` | `~539 KiB` | **`u-boot-sunxi-with-spl.bin`** (Written raw with `dd seek=8`) |
| `1 MiB` – `129 MiB` | Sector `2048` – `264191` | `128 MiB` | **Partition 1 (`sdc1` / `mmcblk0p1`)** — `FAT32` (`BOOT`): `boot.scr`, `zImage`, `sun8i-a33-q8-tablet.dtb` |
| `129 MiB` – End | Sector `264192` – End | `~29.6 GiB` | **Partition 2 (`sdc2` / `mmcblk0p2`)** — `ext4` (`rootfs`): Debian 12 `armhf` RootFS + `/lib/modules/...` |

### 4.2 Partitioning the MicroSD Card with `fdisk`
- **General:** Identify your MicroSD card block device with `lsblk` (e.g., `/dev/sdX` or `/dev/mmcblkX`). Unmount any auto-mounted partitions first (`sudo umount /dev/sdX*`).
- **What we did specifically:** On our Kali machine, the 32 GB (`29.7G`) MicroSD card in a USB reader was **`/dev/sdc`**.

```bash
sudo umount /dev/sdc* 2>/dev/null
sudo fdisk /dev/sdc
```
Inside `fdisk`, enter these exact commands in order:
1. `o` — Create a new empty **MBR (`dos`)** partition table.
2. `n` -> `p` -> `1` -> First sector: `2048` -> Last sector: `+128M` — Create `128 MiB` Partition 1 starting safely at `1 MiB` (`2048`).
3. `t` -> `c` — Set Partition 1 type to `W95 FAT32 (LBA)` (`0x0c`).
4. `n` -> `p` -> `2` -> First sector: `<Enter>` (default `264192`) -> Last sector: `<Enter>` (default end of disk) — Create Partition 2 using the remaining ~`29.6 GiB` (defaults to type `83` `Linux`).
5. `w` — Write the partition table and exit.

### 4.3 Flashing U-Boot to the `8 KiB` Offset
Write `u-boot-sunxi-with-spl.bin` to `/dev/sdc` (the whole disk device, **not** `/dev/sdc1`) starting at `8 KiB` (`bs=1k seek=8`):
```bash
cd ~/Laser_MID-786/u-boot
sudo dd if=u-boot-sunxi-with-spl.bin of=/dev/sdc bs=1k seek=8 conv=fsync,notrunc
```

### 4.4 Formatting the Boot (`FAT32`) & Root (`ext4`) Partitions
```bash
sudo mkfs.vfat -F 32 -n BOOT /dev/sdc1
sudo mkfs.ext4 -F -L rootfs /dev/sdc2
```
Verify with `lsblk -f /dev/sdc` that `sdc1` shows `FSTYPE=vfat LABEL=BOOT` and `sdc2` shows `FSTYPE=ext4 LABEL=rootfs`.

---

## 5. Building the Mainline Linux Kernel (`7.4.0-rc2`) & Device Tree

All details on *why* each kernel config and Device Tree modification was required are also cataloged in [KERNEL_AND_DTB_CHANGES.md](file:///C:/Users/Roadb/Documents/Linux%20on%20Laser%20MID-786/KERNEL_AND_DTB_CHANGES.md).

### 5.1 Clone Mainline Linux & Load Base `sunxi_defconfig`
```bash
cd ~/Laser_MID-786
git clone --depth 1 https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
cd linux

make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- sunxi_defconfig
```

### 5.2 Enable Required Framebuffer / Console `.config` Options
In mainline Linux `7.4.0-rc2`, `sunxi_defconfig` leaves `CONFIG_DRM_SIMPLEDRM`, `CONFIG_DRM_FBDEV_EMULATION`, and `CONFIG_FRAMEBUFFER_CONSOLE` disabled (`# CONFIG_FB is not set`), which results in `Console: colour dummy device 80x30` and no text on `tty1`. Enable them using the kernel's `./scripts/config` helper:
```bash
./scripts/config --enable DRM_SIMPLEDRM
./scripts/config --enable DRM_FBDEV_EMULATION
./scripts/config --enable FRAMEBUFFER_CONSOLE

# Resolve dependencies automatically
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- olddefconfig
```
Verify with:
```bash
grep -E "DRM_SIMPLEDRM|DRM_FBDEV_EMULATION|FRAMEBUFFER_CONSOLE" .config
```
Expected output:
```text
CONFIG_DRM_FBDEV_EMULATION=y
CONFIG_DRM_SIMPLEDRM=y
CONFIG_FRAMEBUFFER_CONSOLE=y
```

### 5.3 Patch the Device Tree (`arch/arm/boot/dts/allwinner/sun8i-a33-q8-tablet.dts`)
Out of the box, `sun8i-a33-q8-tablet.dts` only includes `sun8i-a33.dtsi` and `sun8i-q8-common.dtsi` without defining the LCD panel timings or overriding the OTG VBUS/ID detection for our tablet's single-port OTG host wiring.

We added three overrides to the bottom of `arch/arm/boot/dts/allwinner/sun8i-a33-q8-tablet.dts`:
1. **`&panel` (`panel-dpi` with `1024x600` timings):** Satisfies the `/panel` dependency (`1c0c000.lcd-controller: wait for supplier /panel`) using the exact timing parameters from U-Boot's `q8_a33_tablet_1024x600_defconfig`, allowing `sun4i-drm` to initialize `/dev/fb0` (`1024x600`).
2. **`&usb_otg { dr_mode = "host"; };`:** Forces the MUSB controller (`usb@1c19000`) into dedicated USB Host mode instead of waiting on an unconnected/floating ID pin.
3. **`&usbphy` (`/delete-property/ usb0_vbus_power-supply;` & `/delete-property/ usb0_id_det-gpios;`):**
   - When U-Boot turns on the AXP223 `usb0-vbus` (`drivevbus`) 5V boost regulator, the PMIC's `usb_power_supply` senses 5V on the VBUS line.
   - If `usb0_vbus_power-supply` is left present in `&usbphy`, `phy-sun4i-usb` mistakes its own 5V output for an external charger (`External vbus detected, not enabling our own vbus`) and skips calling `regulator_enable(phy->vbus)`.
   - Then, ~28 seconds into boot, Linux's unused-regulator cleanup runs `usb0-vbus: disabling`, cutting 5V power to the USB keyboard (`musb-hdrc.2.auto: VBUS_ERROR in a_wait_bcon`).
   - Deleting `usb0_vbus_power-supply` and `usb0_id_det-gpios` makes `phy-sun4i-usb` properly claim and hold `usb0-vbus` (`5000mV`) enabled indefinitely.

Append these nodes to `arch/arm/boot/dts/allwinner/sun8i-a33-q8-tablet.dts`:
```dts
&panel {
	compatible = "panel-dpi";
	width-mm = <154>;
	height-mm = <86>;
	bits-per-color = <6>;

	panel-timing {
		clock-frequency = <51000000>;
		hactive = <1024>;
		vactive = <600>;
		hback-porch = <159>;
		hfront-porch = <160>;
		hsync-len = <1>;
		vback-porch = <22>;
		vfront-porch = <12>;
		vsync-len = <1>;
		hsync-active = <1>;
		vsync-active = <1>;
		de-active = <1>;
		pixelclk-active = <0>;
	};
};

&usb_otg {
	dr_mode = "host";
};

&usbphy {
	/delete-property/ usb0_vbus_power-supply;
	/delete-property/ usb0_id_det-gpios;
};
```

### 5.4 Compile `zImage`, Kernel Modules, and Device Tree Blobs
```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j$(nproc) zImage modules dtbs
```
Outputs:
- Kernel image: `arch/arm/boot/zImage`
- Device Tree Blob: `arch/arm/boot/dts/allwinner/sun8i-a33-q8-tablet.dtb`
- Kernel version string: `7.4.0-rc2-g067c5c24cef2-dirty`

---

## 6. Creating the Debian 12 (`bookworm`) `armhf` RootFS

### 6.1 Mount the MicroSD Card Partitions
> [!CAUTION]
> Always run `lsblk /dev/sdc` after mounting to confirm `/dev/sdc1` and `/dev/sdc2` are genuinely mounted on `/mnt/sd_boot` and `/mnt/sd_card_root`. If you write to an unmounted `/mnt/...` directory, the files simply go into your Kali host RAM/overlay disk instead of the MicroSD card!

```bash
sudo mkdir -p /mnt/sd_boot /mnt/sd_card_root
sudo mount /dev/sdc1 /mnt/sd_boot
sudo mount /dev/sdc2 /mnt/sd_card_root
lsblk -f /dev/sdc
```

### 6.2 Stage 1 & Stage 2 `debootstrap` (`armhf`)
Because the host PC is `x86_64` and the tablet is 32-bit ARM (`armhf`), `debootstrap` runs in two stages:
1. **Stage 1 (`--foreign`):** Downloads and unpacks the `.deb` archives using the host CPU.
2. **Stage 2 (`chroot`):** Configures the packages inside the target rootfs using QEMU user-mode emulation (`qemu-arm`).

- **Environment Note (`qemu-arm-static` vs modern `binfmt_misc`):**
  - Older guides instruct you to run `sudo cp /usr/bin/qemu-arm-static /mnt/sd_card_root/usr/bin/`.
  - On modern Debian/Kali (`qemu-user-binfmt`), `/usr/bin/qemu-arm-static` no longer exists. Instead, `/proc/sys/fs/binfmt_misc/qemu-arm` is registered with `flags: POF` (`F` = *fix-binary*, meaning the host kernel opens `/usr/bin/qemu-arm` once in host namespace and passes the open file descriptor into any `chroot` automatically). You do **not** need to copy any binary into the chroot.

```bash
# Stage 1: Download & extract Debian 12 (bookworm) armhf packages
sudo debootstrap --arch=armhf --foreign bookworm /mnt/sd_card_root http://deb.debian.org/debian

# Stage 2: Complete package setup inside ARM chroot
sudo chroot /mnt/sd_card_root /debootstrap/debootstrap --second-stage
```

### 6.3 Configure System Files Inside the RootFS
Enter the `chroot`:
```bash
sudo chroot /mnt/sd_card_root /bin/bash
```
Inside the `chroot`, set the `root` password, hostname, and `/etc/fstab`:
```bash
# 1. Set root password (we set it to: root)
passwd

# 2. Set hostname
echo "mid786" > /etc/hostname

# 3. Create /etc/fstab (Note: On the tablet, the MicroSD slot is mmcblk0!)
cat << 'EOF' > /etc/fstab
/dev/mmcblk0p2  /      ext4  defaults,noatime  0  1
/dev/mmcblk0p1  /boot  vfat  defaults,noatime  0  2
EOF

# 4. Enable persistent systemd journal logs on the SD card (critical for inspecting boot logs offline)
mkdir -p /var/log/journal

exit
```

### 6.4 Install Kernel, DTB, and Kernel Modules to the MicroSD Card
From `~/Laser_MID-786/linux`:
```bash
# Copy kernel and DTB to Partition 1 (FAT32 BOOT)
sudo cp arch/arm/boot/zImage /mnt/sd_boot/
sudo cp arch/arm/boot/dts/allwinner/sun8i-a33-q8-tablet.dtb /mnt/sd_boot/

# Install loadable kernel modules to Partition 2 (/lib/modules/7.4.0-rc2-...)
sudo make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=/mnt/sd_card_root modules_install
```

---

## 7. Creating the U-Boot Boot Script (`boot.cmd` -> `boot.scr`)

### 7.1 Why `usb stop` Is First in `boot.cmd`
When a Micro-USB OTG adapter and USB keyboard are plugged in when you press Power, **U-Boot** initializes the Allwinner MUSB controller (`usb@1c19000`) in Host mode and enumerates the keyboard so you can type in the U-Boot shell.

However, when U-Boot jumps to Linux via `bootz` without stopping USB first:
- Mainline Linux's `sunxi_musb_init()` (`drivers/usb/musb/sunxi.c`) and `sun4i_usb_phy_init()` (`drivers/phy/allwinner/phy-sun4i-usb.c`) only call `reset_control_deassert()`—which is a no-op if U-Boot already deasserted the reset line!
- Because the MUSB controller is never reset during the handoff, its registers remain polluted (`DevCtl: 0x99` = `BDEVICE | VBUS | SESSION`, `Power: 0xf0` = `HSMODE` stuck set, `Frame: 0x00ea` frozen), leaving Linux MUSB stuck in `a_wait_bcon` and ignoring the plugged-in keyboard.
- Running **`usb stop`** at the top of `boot.cmd` cleanly shuts down U-Boot's USB host stack before `bootz` runs, allowing Linux to bring up MUSB cleanly (`DevCtl: 0x3d` = `LSDEV | VBUS | HM | SESSION`, `Power: 0xe0`) whether the keyboard was plugged in before power-on or hot-plugged later.

When using the modified Linux kernel and or U-Boot you do not need to include `usb stop`.

### 7.2 What Each Line of `boot.cmd` Does
- `usb stop`: Tears down U-Boot's active MUSB host session before handing hardware control to Linux.
- `setenv bootargs "..."`: Passes kernel command-line arguments to Linux:
  - `console=ttyS0,115200`: Sends kernel messages to serial UART0 (standard fallback).
  - `console=tty1`: Makes **`tty1` (the `1024x600` LCD screen)** the primary console so boot logs and the `mid786 login:` prompt appear on screen.
  - `root=/dev/mmcblk0p2`: Tells the kernel that the root filesystem is on MicroSD (`mmcblk0`) Partition 2 (`p2`).
  - `rootwait`: Waits asynchronously for the MicroSD controller driver (`sunxi-mmc`) to finish probing `/dev/mmcblk0p2` before attempting to mount `/`.
  - `rw`: Mounts the root filesystem read-write.
  - `loglevel=7`: Prints verbose kernel messages (`KERN_DEBUG` and above) to the LCD during boot.
- `fatload mmc 0:1 ${kernel_addr_r} zImage`: Reads `zImage` from MicroSD device `0`, partition `1` into RAM at `0x42000000`.
- `fatload mmc 0:1 ${fdt_addr_r} sun8i-a33-q8-tablet.dtb`: Reads the compiled Device Tree Blob into RAM at `0x43000000`.
- `bootz ${kernel_addr_r} - ${fdt_addr_r}`: Boots the `zImage` at `${kernel_addr_r}` (`-` = no separate `initramfs`), passing the DTB address `${fdt_addr_r}`.

### 7.3 Write `boot.cmd` and Compile `boot.scr`
```bash
cd ~/Laser_MID-786

printf '%s\n' \
  'usb stop' \
  'setenv bootargs "console=ttyS0,115200 console=tty1 root=/dev/mmcblk0p2 rootwait rw loglevel=7"' \
  'fatload mmc 0:1 ${kernel_addr_r} zImage' \
  'fatload mmc 0:1 ${fdt_addr_r} sun8i-a33-q8-tablet.dtb' \
  'bootz ${kernel_addr_r} - ${fdt_addr_r}' > boot.cmd

# Wrap boot.cmd with a 64-byte U-Boot mkimage header
mkimage -C none -A arm -T script -d boot.cmd boot.scr

# Copy boot.scr to Partition 1 and cleanly unmount the MicroSD card
sudo cp boot.scr /mnt/sd_boot/
sync
sudo umount /mnt/sd_boot /mnt/sd_card_root
```

---

## 8. Booting the Tablet

1. Insert the MicroSD card into the Laser MID-786.
2. Plug your USB keyboard into the Micro-USB OTG adapter and connect it to the tablet's Micro-USB port.
3. Hold the **Power** button for ~2–3 seconds:
   - **U-Boot** initializes DRAM, lights up the `1024x600` LCD, detects the USB keyboard, runs `boot.scr` (`usb stop` -> loads `zImage` and `sun8i-a33-q8-tablet.dtb` -> `Starting kernel ...`).
   - **Mainline Linux `7.4.0-rc2`** initializes `sun4i-drm` (`fb0: sun4i-drmdrmfb`), turns on `usb0-vbus` (`5000mV`), enumerates the USB keyboard (`input: ... Keyboard as /devices/.../usb1/1-1/...`), mounts `/dev/mmcblk0p2`, and starts Debian 12 `systemd`.
   - You arrive at `Debian GNU/Linux 12 mid786 tty1` -> `mid786 login:`. Log in with username `root` and password `root`.

---

## 9. Lessons Learned, Gotchas, & Maintenance Reference

### 9.1 Windows-to-Linux Clipboard `CRLF` (`\r`) Corruption
When copying multi-line blocks from Windows into a Kali terminal, Windows line endings (`\r\n` / `^M`) or terminal line-wrapping can sneak into `boot.cmd`:
- A trailing `\r` on `fatload mmc 0:1 ${kernel_addr_r} zImage\r` causes U-Boot to look for a file literally named `"zImage\r"`, failing with `** Unable to read file zImage **` and `bootz: Bad magic!`.
- Always use `printf '%s\n' ... > boot.cmd` or strip carriage returns with `sed -i 's/\r$//' boot.cmd`, and verify with `cat -A boot.cmd` (every line should end purely with `$`, never `^M$`).

### 9.2 Battery Drain While Using USB OTG Host Mode
Because the tablet has a single Micro-USB port and no separate DC barrel jack:
- When a USB OTG adapter + keyboard is plugged in, the AXP223 PMIC boosts the internal 3.7V Li-ion battery up to `5.0V` (`usb0-vbus`) to power both the tablet (~`1.0A` draw with the LCD backlight at 100%) and the attached USB peripheral.
- It **cannot charge** while the OTG adapter is plugged in. Check battery percentage anytime in Linux via:
  ```bash
  cat /sys/class/power_supply/axp20x-battery/capacity
  cat /sys/class/power_supply/axp20x-battery/voltage_now
  ```
- Always run `poweroff` cleanly before the battery hits `0%` (`~3.3V`), and charge the tablet with the MicroSD card removed (or while powered off) using a standard Micro-USB charger.

### 9.3 Recovering from an Unclean Shutdown (`ext4` Journal / Orphan Inodes)
If the battery ever dies mid-session and systemd logs fail to open (`system.journal: Structure needs cleaning`) or files act read-only:
1. Insert the MicroSD card into the Kali PC.
2. Unmount **all** mountpoints of `/dev/sdc1` and `/dev/sdc2` (check `lsblk /dev/sdc`—watch out for desktop auto-mounters mounting under `/media/kali/...` while you also mounted `/mnt/sd_card_root`):
   ```bash
   sudo umount /dev/sdc1 /dev/sdc2 2>/dev/null
   ```
3. Run filesystem checks on both partitions:
   ```bash
   sudo fsck.vfat -a /dev/sdc1
   sudo fsck.ext4 -f -y /dev/sdc2
   ```

### 9.4 Reading Offline Boot Logs from the MicroSD Card (When Debugging Without Serial UART)
Because our PCB has no soldered UART pads, we created `/var/log/journal` on Partition 2. Whenever you want to inspect the tablet's full kernel `dmesg` and `systemd` logs on your Kali PC:
```bash
sudo mount -o ro /dev/sdc2 /mnt/sd_card_root
sudo journalctl -D /mnt/sd_card_root/var/log/journal -b -0 -k --no-pager
sudo umount /mnt/sd_card_root
```

### 9.5 Example Early Hardware Debug Script & `systemd` Oneshot Service
Because we could not type on the tablet before USB OTG worked (and had no serial UART), we created a temporary `systemd` oneshot service to capture live `sysfs`/`debugfs` hardware state (`power_supply`, USB devices, MUSB `mode`/`vbus`, `extcon`, `/sys/kernel/debug/usb/musb*/regdump`, and `dmesg`) directly into `/boot/usb_debug.txt` on the FAT32 partition so we could inspect it offline from Kali.

#### 1. Create the Debug Snapshot Script (`/usr/local/sbin/usb-debug.sh`) on Partition 2 (`sdc2`)
```sh
#!/bin/sh
exec > /boot/usb_debug.txt 2>&1
echo ===EARLY===
grep "" /sys/class/power_supply/*/uevent
ls /sys/bus/usb/devices
grep "" /sys/bus/platform/devices/musb*/mode
grep "" /sys/bus/platform/devices/musb*/vbus
grep "" /sys/class/extcon/*/state
ls /sys/kernel/debug /sys/kernel/debug/usb
cat /sys/kernel/debug/usb/musb*/regdump
dmesg | grep -iE "usb|musb|phy|axp|vbus|hid"
sync
echo EARLY_SAVED > /dev/tty1
sleep 20
echo ===20S===
ls /sys/bus/usb/devices
grep "" /sys/bus/platform/devices/musb*/mode
grep "" /sys/bus/platform/devices/musb*/vbus
cat /sys/kernel/debug/usb/musb*/regdump
dmesg | tail -n 20
sync
echo 20S_SAVED > /dev/tty1
```
Make it executable (and strip any stray Windows `\r` line endings if pasted):
```bash
sudo sed -i 's/\r$//' /mnt/sd_card_root/usr/local/sbin/usb-debug.sh
sudo chmod +x /mnt/sd_card_root/usr/local/sbin/usb-debug.sh
```

#### 2. Create & Enable the `systemd` Oneshot Unit (`/etc/systemd/system/usb-debug.service`)
```ini
[Unit]
After=local-fs.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/usb-debug.sh

[Install]
WantedBy=multi-user.target
```
Because we were editing an unbooted rootfs offline from Kali rather than running `systemctl enable` on the live tablet, we created the `multi-user.target.wants` symlink manually:
```bash
sudo sed -i 's/\r$//' /mnt/sd_card_root/etc/systemd/system/usb-debug.service
sudo mkdir -p /mnt/sd_card_root/etc/systemd/system/multi-user.target.wants
sudo ln -sf /etc/systemd/system/usb-debug.service /mnt/sd_card_root/etc/systemd/system/multi-user.target.wants/usb-debug.service
sync && sudo umount /mnt/sd_card_root
```
After booting the tablet and waiting for `20S_SAVED` on `tty1`, we mounted `/dev/sdc1` read-only on Kali (`sudo mount -o ro /dev/sdc1 /mnt/sd_boot`) and read `/mnt/sd_boot/usb_debug.txt`.

> [!NOTE]
> Reading `/sys/kernel/debug/usb/musb*/regdump` probes standard MUSB register offsets that Allwinner's stripped-down MUSB core does not implement, causing harmless `musb-sunxi: Error unknown readw/readb offset ...` lines in `dmesg` / `tty1`.

#### 3. Cleaning Up the Temporary Debug Service Once Fixed
Once keyboard login works, run these on the tablet as `root` to remove the debug service and silence those `regdump` probe messages:
```bash
systemctl disable usb-debug.service
rm /etc/systemd/system/usb-debug.service
rm /usr/local/sbin/usb-debug.sh
rm /boot/usb_debug.txt
systemctl daemon-reload
```
