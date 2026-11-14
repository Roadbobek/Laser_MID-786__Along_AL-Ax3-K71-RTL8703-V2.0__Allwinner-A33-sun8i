# Kernel `.config`, Device Tree, & Driver Changes (`7.4.0-rc2`)

This file records every modification we made (and identified for permanent upstream/fork commits) on top of mainline Linux **`7.4.0-rc2` (`067c5c24cef2`)** to support the **Laser MID-786** (`Along AL-Ax3-K71-RTL8703-V2.0 / 20160128`, Allwinner A33).

When we fork the mainline Linux repository to commit and publish our board support, these are the exact changes to include.

---

## 1. Kernel `.config` Additions (On Top of `arch/arm/configs/sunxi_defconfig`)

### Problem
In mainline Linux `7.4.0-rc2`, `sunxi_defconfig` sets `CONFIG_DRM=y` and `CONFIG_DRM_SUN4I=y`, but leaves framebuffer emulation (`CONFIG_DRM_FBDEV_EMULATION`) and the framebuffer console (`CONFIG_FRAMEBUFFER_CONSOLE`) disabled (`# CONFIG_FB is not set`). Without those, Linux boots with `Console: colour dummy device 80x30` and never attaches `tty1` to the LCD screen.

### Config Symbols Enabled
```ini
CONFIG_DRM_SIMPLEDRM=y
CONFIG_DRM_FBDEV_EMULATION=y
CONFIG_FRAMEBUFFER_CONSOLE=y
```

### Commands to Reproduce on a Fresh `sunxi_defconfig`
```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- sunxi_defconfig
./scripts/config --enable DRM_SIMPLEDRM
./scripts/config --enable DRM_FBDEV_EMULATION
./scripts/config --enable FRAMEBUFFER_CONSOLE
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- olddefconfig
```
*(When maintaining our own kernel fork, we can either add these three lines directly to `arch/arm/configs/sunxi_defconfig` or create a dedicated `arch/arm/configs/laser_mid786_defconfig`.)*

---

## 2. Device Tree Patch (`arch/arm/boot/dts/allwinner/sun8i-a33-q8-tablet.dts`)

### Exact `git diff` Applied
```diff
diff --git a/arch/arm/boot/dts/allwinner/sun8i-a33-q8-tablet.dts b/arch/arm/boot/dts/allwinner/sun8i-a33-q8-tablet.dts
index 2db3be3bb689..8f49d561c92e 100644
--- a/arch/arm/boot/dts/allwinner/sun8i-a33-q8-tablet.dts
+++ b/arch/arm/boot/dts/allwinner/sun8i-a33-q8-tablet.dts
@@ -49,3 +49,35 @@
 / {
 	model = "Q8 A33 Tablet";
 	compatible = "allwinner,q8-a33", "allwinner,sun8i-a33";
 };
+
+&panel {
+	compatible = "panel-dpi";
+	width-mm = <154>;
+	height-mm = <86>;
+	bits-per-color = <6>;
+
+	panel-timing {
+		clock-frequency = <51000000>;
+		hactive = <1024>;
+		vactive = <600>;
+		hback-porch = <159>;
+		hfront-porch = <160>;
+		hsync-len = <1>;
+		vback-porch = <22>;
+		vfront-porch = <12>;
+		vsync-len = <1>;
+		hsync-active = <1>;
+		vsync-active = <1>;
+		de-active = <1>;
+		pixelclk-active = <0>;
+	};
+};
+
+&usb_otg {
+	dr_mode = "host";
+};
+
+&usbphy {
+	/delete-property/ usb0_vbus_power-supply;
+	/delete-property/ usb0_id_det-gpios;
+};
```

### Why Each Device Tree Change Is Required

#### 2.1 `&panel` (`panel-dpi` `1024x600` RGB666 Timings)
- **Root Cause:** `arch/arm/boot/dts/allwinner/sun8i-q8-common.dtsi` declares the `panel: panel` node and wires its ports to the RGB666 LCD controller (`tcon0`), but intentionally omits a `compatible` string (`/* Tablet dts should provide panel compatible */`) because Q8 tablets come in both `800x480` and `1024x600` variants.
- **Symptom Without Fix:** `sun4i-drm` defers probing forever:
  ```text
  platform 1c0c000.lcd-controller: deferred probe pending: platform: wait for supplier /panel
  ```
- **How the Fix Works:** By specifying `compatible = "panel-dpi";`, `bits-per-color = <6>;` (18-bit RGB666), and the exact `panel-timing` values from U-Boot's `q8_a33_tablet_1024x600_defconfig` (`x:1024,y:600,depth:18,pclk_khz:51000,le:159,ri:160,up:22,lo:12,hs:1,vs:1,sync:3,vmode:0`), `panel-simple` binds to `/panel` and `sun4i-drm` registers `/dev/fb0` (`1024x600`).

#### 2.2 `&usb_otg` (`dr_mode = "host"`) and `&usbphy` (`usb0_vbus_power-supply` & `usb0_id_det-gpios` Removal)
- **Root Cause:**
  1. `sun8i-q8-common.dtsi` configures `usb_otg` with `dr_mode = "otg"`, `usb0_id_det-gpios = <&pio 7 8 ...>` (`PH8`), and `usb0_vbus_power-supply = <&usb_power_supply>` (the AXP223 VBUS sense block), plus `usb0_vbus-supply = <&reg_drivevbus>` (the AXP223 5V boost regulator).
  2. When U-Boot runs (or when `reg_drivevbus` is already outputting 5V on the Micro-USB port), the AXP223's internal VBUS sensor (`axp20x-usb-power-supply`) sees 5V on the shared Micro-USB pin and reports `POWER_SUPPLY_ONLINE=1`.
  3. During kernel boot, `sun4i_usb_phy_set_mode()` / `sun4i_usb_phy0_reroute()` in `drivers/phy/allwinner/phy-sun4i-usb.c` checks `data->vbus_power_supply`. Seeing `POWER_SUPPLY_ONLINE=1`, it assumes an external wall charger is supplying 5V:
     ```text
     sun4i-usb-phy 1c19400.phy: External vbus detected, not enabling our own vbus
     ```
     Consequently, `phy-sun4i-usb` **never calls `regulator_enable(phy->vbus)`** (`reg_drivevbus` / `usb0-vbus`), leaving its kernel regulator enable reference count at `0`.
  4. Approximately 27 seconds after boot (`late_initcall_sync` + 30s deferred regulator cleanup), the kernel disables all regulators with `use_count == 0`:
     ```text
     usb0-vbus: disabling
     musb-hdrc.2.auto: VBUS_ERROR in a_wait_bcon (80, <VBusValid), retry #3, port1 00000108
     ```
     Cutting `usb0-vbus` kills 5V power to the OTG port, causing the USB keyboard to go completely dead.
- **How the Fix Works:** Setting `dr_mode = "host"` on `&usb_otg` and deleting `usb0_vbus_power-supply` and `usb0_id_det-gpios` from `&usbphy` ensures `phy-sun4i-usb` unconditionally enables `reg_drivevbus` (`usb0-vbus`) and holds a reference to it for the entire uptime of the system.

---

## 3. USB OTG Cold-Boot Handover Fix (`usb stop` Workaround vs. Permanent Kernel Driver Patch)

### Root Cause of the U-Boot -> Linux MUSB Handover Bug
When a USB OTG keyboard is plugged in **at cold power-on**:
1. U-Boot initializes `usb@1c19000` (`Allwinner mUSB OTG (Host)`) and enumerates the low-speed USB keyboard (`1 USB Device(s) found`).
2. U-Boot's `bootz` command does **not** invoke `.remove()` on the MUSB driver unless `CONFIG_DM_DEVICE_REMOVE` + `DM_FLAG_ACTIVE_DMA` / `DM_FLAG_OS_PREPARE` is set on the `musb-new` U-Boot driver (or `usb stop` is run explicitly).
3. When Linux boots and probes `drivers/phy/allwinner/phy-sun4i-usb.c` (`sun4i_usb_phy_init`) and `drivers/usb/musb/sunxi.c` (`sunxi_musb_init`), both drivers only call:
   ```c
   reset_control_deassert(...);
   ```
   Because U-Boot *already* deasserted `RST_USB_PHY0` and `RST_BUS_OTG`, calling `reset_control_deassert()` in Linux does **not** pulse the hardware reset lines!
4. As a result, the Allwinner MUSB IP block retains stale register state from U-Boot right through `sunxi_musb_enable()`:
   - **Broken Cold-Boot Register State (without reset / `usb stop`):**
     - `Power  : f0` (`MUSB_POWER_ISOUPDATE | MUSB_POWER_SOFTCONN | MUSB_POWER_HSENAB | MUSB_POWER_HSMODE` — stuck in High-Speed mode `0x10` even though a Low-Speed keyboard is attached!)
     - `DevCtl : 99` (`MUSB_DEVCTL_BDEVICE | MUSB_DEVCTL_VBUS | MUSB_DEVCTL_SESSION` — `BDEVICE` bit `0x80` set, `HM` Host Mode bit `0x04` cleared, `LSDEV` bit `0x20` cleared!)
     - `Frame  : 00ea` (frozen frame counter from U-Boot)
   - **Clean Register State (with `usb stop` or hardware reset pulse):**
     - `Power  : e0` (`MUSB_POWER_ISOUPDATE | MUSB_POWER_SOFTCONN | MUSB_POWER_HSENAB`)
     - `DevCtl : 3d` (`MUSB_DEVCTL_LSDEV | MUSB_DEVCTL_VBUS | MUSB_DEVCTL_HM | MUSB_DEVCTL_SESSION` — `HM` Host Mode active, Low-Speed device detected, `Frame` incrementing)

### Current Bootloader Fix (Applied in `boot.cmd` -> `boot.scr`)
Adding `usb stop` as the first line of `boot.cmd` cleanly stops U-Boot's MUSB host controller before `bootz` executes.

### Permanent C Driver Patch for Our Future Linux Kernel Fork
When we fork Linux so the kernel is completely self-healing regardless of bootloader state (even if someone boots without `usb stop` in `boot.scr`), we can pulse the hardware reset control in `drivers/usb/musb/sunxi.c` (`sunxi_musb_init`) and `drivers/phy/allwinner/phy-sun4i-usb.c` (`sun4i_usb_phy_init`):

```diff
--- a/drivers/usb/musb/sunxi.c
+++ b/drivers/usb/musb/sunxi.c
@@ -625,6 +625,8 @@ static int sunxi_musb_init(struct musb *musb)
 		return ret;
 
 	if (test_bit(SUNXI_MUSB_FL_HAS_RESET, &glue->flags)) {
+		reset_control_assert(glue->rst);
+		usleep_range(10, 20);
 		ret = reset_control_deassert(glue->rst);
 		if (ret)
 			goto error_clk_disable;
```
And similarly in `drivers/phy/allwinner/phy-sun4i-usb.c` inside `sun4i_usb_phy_init()`:
```diff
--- a/drivers/phy/allwinner/phy-sun4i-usb.c
+++ b/drivers/phy/allwinner/phy-sun4i-usb.c
@@ -286,6 +286,8 @@ static int sun4i_usb_phy_init(struct phy *_phy)
 		return ret;
 	}
 
+	reset_control_assert(phy->reset);
+	usleep_range(10, 20);
 	ret = reset_control_deassert(phy->reset);
 	if (ret) {
 		clk_disable_unprepare(phy->clk);
```
Pulsing `reset_control_assert()` followed by `reset_control_deassert()` forces the Allwinner CCU to hardware-reset the MUSB controller and USB PHY registers back to their power-on defaults before Linux configures them.
