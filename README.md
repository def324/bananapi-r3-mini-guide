## Banana Pi R3 Mini — 5G Router Setup Guide

> **Objective:** Convert a Banana Pi R3 Mini fitted with a Quectel RM520N-GL into a dependable 5G router—from hardware setup and firmware flashing to Wi-Fi band-steering, secure user & SSH configuration, targeted package deployment, software-controlled fan management, everyday optimizations, optional GPS/VPN extras, and full recovery procedures.

**Warning:** The guide is a work in progress and may contain mistakes. The author takes no responsibility for any loss of data, hardware damage, or connectivity issues resulting from following these instructions. Proceed at your own risk and double-check any commands or configurations before applying them.


## Table of Contents

- [Banana Pi R3 Mini — 5G Router Setup Guide](#banana-pi-r3-mini--5g-router-setup-guide)
- [Table of Contents](#table-of-contents)
- [Motivation](#motivation)
- [Introduction](#introduction)
- [Hardware Overview](#hardware-overview)
  - [Parts List](#parts-list)
  - [Assembly Notes](#assembly-notes)
  - [Shortcomings](#shortcomings)
- [Firmware Installation](#firmware-installation)
  - [Choosing a Distribution](#choosing-a-distribution)
  - [Downloading Firmware Images](#downloading-firmware-images)
  - [Flashing eMMC](#flashing-emmc)
  - [Flashing NAND](#flashing-nand)
  - [Post-Flash Checklist](#post-flash-checklist)
- [Initial Configuration](#initial-configuration)
  - [User Management \& Remote Access](#user-management--remote-access)
    - [Change the Root Password](#change-the-root-password)
  - [Network Interfaces](#network-interfaces)
    - [Configure Wireless (Single SSID, Band Steering)](#configure-wireless-single-ssid-band-steering)
      - [Prerequisites](#prerequisites)
      - [LuCI Steps](#luci-steps)
      - [Verification](#verification)
      - [Tips](#tips)
  - [Package Installation](#package-installation)
    - [Overview](#overview)
    - [Package Groups](#package-groups)
    - [Install Command](#install-command)
- [Modem \& Connectivity](#modem--connectivity)
  - [Power Cycling via GPIO](#power-cycling-via-gpio)
    - [Why Timing Matters](#why-timing-matters)
    - [Init Script](#init-script)
  - [Setting the Modem to MBIM Mode](#setting-the-modem-to-mbim-mode)
    - [Why MBIM?](#why-mbim)
    - [Check Current Mode](#check-current-mode)
    - [Set MBIM Mode](#set-mbim-mode)
    - [Example Session](#example-session)
    - [Notes](#notes)
  - [Configuring the WWAN (5G/4G) Interface in LuCI](#configuring-the-wwan-5g4g-interface-in-luci)
    - [Steps](#steps)
    - [Verification](#verification-1)
    - [Tips](#tips-1)
  - [Watchdog \& Recovery Scripts](#watchdog--recovery-scripts)
- [GPS Integration](#gps-integration)
  - [Enabling GNSS](#enabling-gnss)
    - [Steps](#steps-1)
    - [Notes](#notes-1)
  - [Init Script](#init-script-1)
  - [`luci-app-gpoint`](#luci-app-gpoint)
- [Fan Control](#fan-control)
  - [`/usr/bin/fancontrol.sh` Script](#usrbinfancontrolsh-script)
- [Storage (NVMe SSD)](#storage-nvme-ssd)
- [Recovery \& Unbrick Guide](#recovery--unbrick-guide)
- [Useful Links](#useful-links)
- [TODO \& Future Improvements](#todo--future-improvements)


---

## Motivation

The Banana Pi R3 Mini caught my eye with its compact footprint, clean enclosure, USB-C power, and the ability to accept a 5G M.2 modem right out of the box. What began as a simple plan for reliable in-car Wi-Fi quickly snowballed into more ideas: pair a Quectel RM520N-GL with a Starlink Mini, bond both links through a VPS, and enjoy uninterrupted connectivity on any road. The modem's GNSS capability suggested yet another idea—privately logging every mile of a journey to a home-server map via wireguard.

The honeymoon ended fast. Flashing firmware, taming AT commands, smoothing kernel power-timing quirks, and persuading GPS to report coordinates were anything but plug-and-play. The stock fan kept temperatures under control but sounded like a miniature leaf-blower, begging for a redesign. Each obstacle, however, turned into a hands-on lesson in modem quirks and embedded Linux.

Now the R3 Mini feels like a pocket-sized lab. A high-capacity power bank powers it for hours of 5G connectivity, and a single USB-C cable to a laptop both feeds power to the board and opens a console for on-the-go troubleshooting.

Next up: a quieter case, smarter startup scripts, and real-world tests of bonded 5G + Starlink links. This guide captures the lessons so you can skip the headaches and dive straight into building something useful—and fun.

Let’s get serious 🧐

## Introduction

This guide walks you through building a compact 5G router powered based on the BananPi R3-Mini platform using ImmortalWrt, with proven reliability real-world automotive use. You’ll learn how to set up advanced features such as WireGuard VPN tunnels to your home network (optional), GPS logging, and continuous GPS location updates sent to a home server (also optional).

Step by step, the guide covers everything you need:

- Parts & Assembly
- Flashing firmware
- Custom fan control scripts
- Modem firmware upgrades
- Configuration via AT commands
- Startup scripts to prevent common glitches
- NVMe storage integration
- Unbrick and recovery procedures

Whether you want robust in-car connectivity or a flexible portable router, this guide equips you to achieve a stable, feature-rich 5G setup.

## Hardware Overview

### Parts List

- **Banana Pi R3 Mini Standard Kit**  
  - Includes: black ABS case, 30 mm fan, and three 2.4 / 5 GHz Wi-Fi antennas.

- **5G Modem: Quectel RM520N-GL-AA**  
  - M.2 Key-B WWAN module with four I-PEX MHF® 4L (20579-001E) antenna receptacles

- **5G/4G Wideband Monopole Antennas**  
  - Taoglas Apex TG.66.A113 hinged terminal-mount monopole covering 600 MHz–6 GHz, SMA-male connector

- **Coaxial Cables (MHF 4L → SMA Male)**  
  - ~1.9 inch custom micro-coax assembly with Ø 1.13 mm MHF® 4L plug to SMA-male pigtail (matches RM520N-GL receptacles)
    _Note: MHF1-to-SMA cables will **not** fit—the RM520N-GL requires the MHF 4L (IPEX 20579-001E) interface.

- **NVMe SSD: WD_BLACK 1 TB SN770M M.2 2230**  
  - PCIe Gen 4 ×4 NVMe SSD, up to 5150 MB/s sequential read, TLC 3D NAND, compact 2230 form factor. The specific model was picked for the strong performance to thermal ratio.

### Assembly Notes

- **Case & M.2 Module Installation**
  - Install the NVMe SSD first
  - Slide the WD_BLACK SN770M into the M.2 Key-M slot at a slight angle.  
  - Press down gently and secure with a screw.  
  - Install the Quectel RM520N-GL-AA modem 
  - Insert into the M.2 Key-B slot until it seats.  
  - Tighten the mounting screw to lock it in place.

- **Antenna Installation & Trade-Off**
  - Pre-route your cables
  - Connect each SMA pigtail to its MHF® 4L receptacle before closing the case.  
  - Lay cables along the board’s edges.  
  - Omit one 2.4 GHz Wi-Fi antenna
  - The case supports four external holes.  
  - Install all three 5G/4G antennas plus your preferred 5 GHz Wi-Fi antenna.  
  - Leave one 2.4 GHz hole empty.

- **Cable Routing Tips**
  - Tuck under the board  
  - Route each coax line beneath the board so there is no chance of it touching the fan blades.  
  - Minimize slack
  - Keep loops tight to the board perimeter—enough to close the lid without pinching. 

- **Avoid heatsink contact**
  - Hold cables clear of the main heatsink and any hot components.  

### Shortcomings

- **Stock Case** 
  - The stock case has fairly poor thermal performance, especially when using a 5G modem.
  - It lacks enough antenna ports to accommodate 4x5G antennas and 4xWiFi antennas which would yield optimal performance.

- **Alternatives** 
  - As of July 2025 there is active development in this area:
    - @JohnBottoms_1508911 developed a [great case](https://www.printables.com/model/1269483-banana-pi-r3-mini-case) that will comfortably fit a Noctou fan. It's footprint is larger than the stock case, but it should significantly improve thermals and noise. This case has not yet been tested as part of the guide development.
    - There is an active discussion on this [Banana Pi form thread](https://forum.banana-pi.org/t/banana-pi-r3-mini-issues/23673). The discussion revolves around modifying the stock case's lid to fit a Noctua fan without increasing the size of the case significantly. As part of the development of this guide we are actively looking into this option, but as of July 2025 we do not have a specific recommendation.

## Firmware Installation

### Choosing a Distribution

- This guide is tested with **ImmortalWrt 24.10.2**.
- **OpenWRT** (as of July 2025) does not offer reliable support for the Banana Pi R3 Mini and should be avoided.
- Other firmware options—such as **r00ter** and **bananwrt**—were evaluated, but caused failures during development and are not recommended.

### Downloading Firmware Images

- Download the recommended firmware files from:
  - https://downloads.immortalwrt.org/releases/24.10.2/targets/mediatek/filogic/
- Download **all files beginning with** `bananapi_bpi-r3-mini`.

### Flashing eMMC

- **Boot from NAND:**  
  
  - Flip the physical switch on the board to `NAND`.
  - Power on.  
  - The device should start with stock firmware (assuming it is still present).

- **Connect to Device:**  
  - The board creates an open Wi-Fi network.
  - Connect and access via SSH:  
    ```bash
    ssh root@192.168.1.1
    ```
  - No password is required.

- **Transfer Firmware Files:**  
  - Use `scp` to copy all necessary files to `/tmp` on the device.  
  - You may need to add the `-O` flag to force legacy SCP protocol:  
    ```bash
    scp -O immortalwrt-24.10.2-mediatek-filogic-bananapi_bpi-r3-mini* root@192.168.1.1:/tmp/
    ```

- **Write Images to eMMC:**  
  - Enter an interactive shell and run the following sequence:  
    ```bash
    cd /tmp
    dd if=immortalwrt-24.10.2-mediatek-filogic-bananapi_bpi-r3-mini-emmc-gpt.bin of=/dev/mmcblk0
    reboot
    # After reboot, reconnect via SSH.
    echo 0 > /sys/block/mmcblk0boot0/force_ro
    dd if=immortalwrt-24.10.2-mediatek-filogic-bananapi_bpi-r3-mini-emmc-preloader.bin of=/dev/mmcblk0boot0
    dd if=immortalwrt-24.10.2-mediatek-filogic-bananapi_bpi-r3-mini-emmc-bl31-uboot.fip of=/dev/mmcblk0p3
    dd if=immortalwrt-24.10.2-mediatek-filogic-bananapi_bpi-r3-mini-initramfs-recovery.itb of=/dev/mmcblk0p4
    dd if=immortalwrt-24.10.2-mediatek-filogic-bananapi_bpi-r3-mini-squashfs-sysupgrade.itb of=/dev/mmcblk0p5
    sync
    ```
  - Now cut the power, flip the switch to EMMC and you should boot into an the ImmortalWRT environment. There will again be an open wifi network that you can connect to.

### Flashing NAND

- **Preparation:** 
  - Ensure you are booted from **eMMC** before proceeding.
  - Copy all relevant ImmortalWrt files to `/tmp` on the device using `scp -O` (legacy mode).
    ```bash
    scp -O immortalwrt-24.10.2-mediatek-filogic-bananapi_bpi-r3-mini* root@192.168.1.1:/tmp/
    ```

- **Enable NAND Write Access:**  
  - Load the kernel module to allow NAND writing (use with caution):
    ```bash
    insmod mtd-rw.ko i_want_a_brick=1
    ```

- **Flashing Steps:**  
  - Run each command in sequence
    ```bash
    mtd write /tmp/immortalwrt-24.10.2-mediatek-filogic-bananapi_bpi-r3-mini-snand-preloader.bin /dev/mtd0
    ubidetach -m 1
    ubiformat /dev/mtd1
    ubiattach -m 1
    volsize=$(wc -c < /tmp/immortalwrt-24.10.2-mediatek-filogic-bananapi_bpi-r3-mini-snand-bl31-uboot.fip)
    ubimkvol /dev/ubi0 -N fip -n 0 -s $volsize -t static
    ubiupdatevol /dev/ubi0_0 /tmp/immortalwrt-24.10.2-mediatek-filogic-bananapi_bpi-r3-mini-snand-bl31-uboot.fip
    cd /lib/firmware/airoha
    cat EthMD32.dm.bin EthMD32.DSP.bin > /tmp/en8811h-fw.bin
    ubimkvol /dev/ubi0 -N en8811h-firmware -n 1 -s 147456 -t static
    ubiupdatevol /dev/ubi0_1 /tmp/en8811h-fw.bin
    ubimkvol /dev/ubi0 -n 2 -N ubootenv -s 126976
    ubimkvol /dev/ubi0 -n 3 -N ubootenv2 -s 126976
    volsize=$(wc -c < /tmp/immortalwrt-24.10.2-mediatek-filogic-bananapi_bpi-r3-mini-initramfs-recovery.itb)
    ubimkvol /dev/ubi0 -n 4 -N recovery -s $volsize
    ubiupdatevol /dev/ubi0_4 /tmp/immortalwrt-24.10.2-mediatek-filogic-bananapi_bpi-r3-mini-initramfs-recovery.itb
    volsize=$(wc -c < /tmp/immortalwrt-24.10.2-mediatek-filogic-bananapi_bpi-r3-mini-squashfs-sysupgrade.itb)
    ubimkvol /dev/ubi0 -n 5 -N fit -s $volsize
    ubiupdatevol /dev/ubi0_5 /tmp/immortalwrt-24.10.2-mediatek-filogic-bananapi_bpi-r3-mini-squashfs-sysupgrade.itb
    ```

- **Final Steps:**  
  - Power off the device.
  - Flip the boot switch back to `NAND`.
  - Power on. You should now boot into ImmortalWrt from NAND.

- **Notes:**  
  - NAND is primarily for recovery. Regular operation should use eMMC.
  - If eMMC becomes unusable, you can always boot from NAND and restore eMMC.

### Post-Flash Checklist

- Confirm the device boots from both **eMMC** and **NAND**:
  - Use the boot mode switch to test each storage option.
  - The system should load ImmortalWrt in both cases.

- Verify network access:
  - Connect via Wi-Fi (default open network) and ensure you can reach the web interface or SSH.
  - Connect via Ethernet (plugged into the LAN port) and ensure you have connectivity.

- Check LuCI (Web UI):
  - Access `http://192.168.1.1` from your browser.
  - Confirm the LuCI interface loads correctly.

- Confirm all intended firmware files are present on the system.

- Check device status:
  - Run `dmesg` and review for hardware or storage errors.
  - Confirm that the modem and other peripherals (if installed) are recognized.

- Change default passwords:
  - Set a strong root password to secure the device.

- For further steps, ensure you are booted from **eMMC**.

## Initial Configuration

### User Management & Remote Access

#### Change the Root Password

1. Log in to LuCI (`http://192.168.1.1` by default).  
2. Go to **System › Administration**.  
3. In **Router Password**, enter your new password twice.  
4. Click **Save & Apply**.

### Network Interfaces

#### Configure Wireless (Single SSID, Band Steering)

**Objective:** Provide one SSID that automatically steers clients between 2.4 GHz and 5 GHz based on capability and signal quality.

##### Prerequisites

- Both radios (`radio0` – 2.4 GHz, `radio1` – 5 GHz) enabled.  
- Unique hex *Mobility Domain* value ready (e.g., `4f57`).  
- Desired SSID and passphrase decided.

##### LuCI Steps

1. Log in to LuCI (`http://192.168.1.1` by default).  
2. Navigate to **Network › Wireless**.

**Configure the 5 GHz AP**

- Click **Edit** on the `radio1` entry.  
- **General Setup**  
  - *Mode*: **Access Point**  
  - *ESSID*: **<Your-SSID>**  
  - *Network*: **lan**  
- **Wireless Security**  
  - *Encryption*: **WPA3-SAE/WPA2-PSK (mixed mode)**  
  - *Key*: **<Your-passphrase>**  
- **WLAN Roaming**  
  - Enable **802.11k** and **802.11v (BSS Transition)**.  
  - Enable **802.11r Fast Transition**.  
  - *Mobility Domain*: **<same 4-digit hex>**  
  - *FT over DS*: **Disabled** (leave default unless roaming issues appear).  
- Click **Save**.

**Configure the 2.4 GHz AP**

- Click **Edit** (or **Add**) on the `radio0` entry.  
- Repeat the **General Setup**, **Wireless Security**, and **WLAN Roaming** sections above, using:  
  - *Channel*: **1** (or any non-overlapping 2.4 GHz channel).  
  - *ESSID*, *Encryption*, *Key*, and *Mobility Domain*: **exactly match the 5 GHz settings**.  
- Click **Save**.

3. Back in **Network › Wireless**, click **Save & Apply**.  
4. Wait for the wireless restart to complete (status turns green).
5. You will need to reconnect to the new wirless network.

##### Verification

- Under **Status › Overview › Wireless**, confirm both radios broadcast the same SSID.  
- Move a client device farther from the router; it should roam to 2.4 GHz automatically.  
- Return close and verify it reconnects to 5 GHz.

##### Tips

- Keep the same **DTIM Interval** on both bands (default `2` – `3` works well).  
- Avoid using *Auto* channel on 5 GHz if DFS channels are unstable in your region.

### Package Installation
#### Overview

Install these packages to enable the web components, cellular modem control, VPN, GPS mapping, and basic monitoring. Dependencies resolve automatically—no need to list them.

#### Package Groups

- **LuCI Apps**
  - `luci-app-3ginfo-lite` – Lightweight cellular-status panel.
  - `luci-app-gpoint` – Live GPS map overlay.
  - `luci-app-statistics` – RRD graphs via *collectd*. (Optional) 
- **Cellular / WWAN**
  - `luci-proto-mbim` – LuCI support for MBIM modems.
  - `umbim` – User-space MBIM control tool.
  - `mbim-utils` – Extra MBIM helpers (signal, firmware info).
  - `kmod-usb-serial-option` – Kernel driver for USB LTE/5G modems.
- **GPS**
  - Installation of the recommended GPS app will be covered in the GPS section.
- **Diagnostics & Utilities**
  - `usbutils` – `lsusb` and friends for USB troubleshooting.
  - `picocom` – Minimal serial console (handy for AT commands).
- **VPN** (Optional)
  - `luci-proto-wireguard` – LuCI integration for WireGuard.
  - `wireguard-tools` – CLI utilities (`wg`, `wg-quick`).
- **Monitoring** (Optional)
  - `collectd-mod-thermal` – Temperature sensor plugin for *collectd*.

#### Install Command

~~~bash
opkg update
opkg install \
  luci luci-app-3ginfo-lite luci-app-gpoint \
  luci-proto-mbim umbim mbim-utils kmod-usb-serial-option \
  gpsd gpsd-clients \
  usbutils picocom
# Optional WireGuard VPN
opkg install luci-proto-wireguard wireguard-tools
# Optional stistics
opkg install luci-app-statistics collectd-mod-thermal
~~~


## Modem & Connectivity

### Power Cycling via GPIO

When the R3 Mini starts up, it doesn't automatically power up the modem just by supplying the standard 3.3V to the M.2 slot. The modem also needs a special "power enable" signal, controlled by a GPIO pin on the board. If this pin isn't set correctly during boot, the modem might not turn on or show up at all.
So, it’s not enough to just plug in the modem—the software must flip a specific "switch" (the WWAN enable pin) at the right time so the modem powers on properly.
With current **OpenWrt / ImmortalWrt** device trees the kernel leaves that GPIO floating, so the modem may power up in an undefined state.  
Rebuilding ImmortalWrt with a patched DTS fixes this, but the goal here is **zero custom images**, so we rely on an init script.

#### Why Timing Matters

- The modem’s internal PMIC expects the enable pin **low** for a few seconds after main power appears, then **high**.  
- If raised too early (or not driven at all) the module may remain invisible on USB.  
- A cold-boot timing sequence of **low → 5 s → high** proved ~ 99 % reliable in testing.

#### Init Script

Create `/etc/init.d/modem-power`:

```bash
#!/bin/sh /etc/rc.common

START=99

USE_PROCD=1

start_service() {
    logger -t modem_gps "Waiting for Quectel modem..."
    found=0
    for i in $(seq 1 60); do
        if lsusb | grep -qi quectel; then
            found=1
            logger -t modem_gps "Quectel modem found."
            break
        fi
        sleep 1
    done

    if [ "$found" -eq 0 ]; then
        logger -t modem_gps "Modem not found after 60s, rebooting."
        reboot
        exit 1
    fi

    logger -t modem_gps "Waiting 10 seconds before enabling modem."
    sleep 10
    logger -t modem_gps "Enabling GPS on modem via /dev/ttyUSB2."
    echo -e "AT+QGPS=1\r" > /dev/ttyUSB2
    logger -t modem_gps "GPS enable command sent."
}
```

Enable and start it:
```bash
chmod +x /etc/init.d/modem-power
/etc/init.d/modem-power enable
reboot
# After reboot check that the modem came up correctly:
lsusb
```

The early **START=01** slot makes sure GPIO control runs before USB enumeration.

**Troubleshooting Notes**
* **Occasional no-show:** Even with correct timing the modem can fail to enumerate after an unclean shutdown. The recovery strategy is covered in the GPS section. (*Note:* A dedicated watchdog script could be a potential improvement here)
* **Fine-tuning:** If you swap modems or use a custom enclosure, adjust the sleep duration—some variants respond better to 3–4 s instead of 5 s.
### Setting the Modem to MBIM Mode

The Quectel RM520N-GL supports multiple USB operating modes. For use with OpenWrt or ImmortalWrt, set the modem to **MBIM** mode (`usbnet=2`).

#### Why MBIM?

- MBIM mode offers stable, high-speed operation with broad Linux support.
- Integrates cleanly with LuCI and network scripts.
- Among all the available modes has been tested to be be most reliable.

#### Check Current Mode

Connect to the modem’s serial port (e.g., `/dev/ttyUSB2`) using `picocom` or a similar tool:

```bash
picocom -b 115200 /dev/ttyUSB2
```

At the terminal prompt, send:

```
AT+QCFG="usbnet"
```

- Typical response:
  - `2` = MBIM (Recommended)
  - `0` = ECM
  - `1` = QMI
  - `3` = RNDIS

#### Set MBIM Mode

To set MBIM mode, enter:

```
AT+QCFG="usbnet",2
```

- You should see an `OK` response.
- At this point it’s best to reboot the device.

#### Example Session

```bash
picocom -b 115200 /dev/ttyUSB2
# Then type:
AT+QCFG="usbnet"      # Check current mode
AT+QCFG="usbnet",2    # Switch to MBIM mode
```

#### Notes

- Setting persists across reboots.
- After switching to MBIM, the modem should show up as an MBIM network device (e.g., `wwan0`) in `ifconfig` or `ip a`.

### Configuring the WWAN (5G/4G) Interface in LuCI

To bring up your Quectel RM520N-GL modem for cellular data, configure the WWAN interface using the LuCI web interface. This ensures a persistent, easy-to-manage connection.

#### Steps

1. **Open LuCI:**
   - In your browser, go to `http://192.168.1.1` and log in.

2. **Navigate to Network Interfaces:**
   - Go to **Network › Interfaces**.

3. **Add a New Interface:**
   - Click **Add new interface...**
   - Name: `wwan`
   - Protocol: **MBIM (Mobile Broadband Interface Model)**

4. **Device Selection:**
   - For **Device**, choose your MBIM modem device (usually `wwan0`).

5. **APN Configuration:**
   - Enter your cellular provider’s APN in the **APN** field.
     - Example: `internet`, `broadband`, or your SIM’s documented APN.
   - Leave username/password empty unless required by your provider.

6. **Authentication (if needed):**
   - Set the **PAP/CHAP** method if your provider uses authentication (most don’t for data-only SIMs).

7. **PIN (if required):**
   - If your SIM card is PIN-locked, enter the PIN under **PIN**.
   - Leave blank if PIN is not set or has already been disabled.

8. **Advanced Options:**
   - You may leave advanced options at defaults for most use cases.
   - If dual-SIM or custom MTU is needed, adjust under **Advanced Settings**.

9. **Firewall Settings:**
   - Assign the interface to the **wan** firewall zone (default is correct for internet access).

10. **Save & Apply:**
    - Click **Save** then **Save & Apply**.
    - Wait for the configuration to reload.

#### Verification

- Return to **Network › Interfaces**.
- The `wwan` interface should show a valid IPv4 address if the SIM and APN are correct.
- If not, check:
  - SIM card orientation and data plan.
  - APN spelling.
  - Modem mode (should be MBIM).

#### Tips

- You can view live modem status under **Modem > Information about 3G/4G/5G connection** section in LUCI.
- For SIM PIN issues, try a different SIM or use a phone to disable the PIN.

### Watchdog & Recovery Scripts
*TODO* (Currently handled by GPS integration)

## GPS Integration

### Enabling GNSS

The Quectel RM520N-GL includes built-in GNSS (GPS/GLONASS/BeiDou/Galileo) functionality. To enable GPS features for mapping and location-based services, activate GNSS output over USB using AT commands. Especially in MBIM mode getting GNSS to work requires some tweaks.

#### Steps

1. **Connect to the Modem’s AT Command Port**

   - Use a serial terminal such as `picocom` to open the correct port (typically `/dev/ttyUSB2`).

   ```bash
   picocom -b 115200 /dev/ttyUSB2
   ```

2. **Enable GPS**

   - Issue the following AT command to turn on the GNSS receiver:

   ```
   AT+QGPS=1
   ```

   - This command is needed after every reboot to start the GNSS engine. (See below for the GPS startup script)

3. **Configure NMEA Output Over USB**

   - The next commands are usually persistent, but can be re-sent to ensure correct setup:

   ```
   AT+QGPSCFG="outport","usbnmea"
   AT+QGPSCFG="gpsnmeatype",31
   ```

   - This sends NMEA GPS data over the modem’s USB port, enabling tools like `gpsd` to read real-time positioning data.

4. **Check GPS Status**

   - To check for a current GPS fix or view raw data (**Note:** It can take 5-10 minuntes to establish an initial GPS fix):

   ```
   AT+QGPSLOC=0
   AT+QGPSGNMEA="RMC"
   ```

   - `AT+QGPSLOC=0` returns the current position if available.
   - `AT+QGPSGNMEA="RMC"` outputs the NMEA Recommended Minimum data sentence.

#### Notes

- The modem must have a clear view of the sky (or antenna properly routed) for GNSS lock.
- If no fix is reported, check antenna placement and wait several minutes for initial lock (cold start).
- Most settings persist across reboots, except for the actual GNSS engine activation (`AT+QGPS=1`), which must be sent every time the system starts.

### Init Script

The following `/etc/init.d/modem-gps` script will make sure that GPS is enabled after the modem is fully detected. At the moment the script will also reboot the device in case the modem does not come up. This is not really related to GPS, but rather to the GPIO timing issues explained in the Modem & Connectivity section. It would be cleaner to have a dedicated watchdog script for this.
**Important:** This script could get your device stuck in a reboot loop simply disable/delete it if that happens. You need to make first sure your modem reliably gets detected on every boot as described in the Modem & Connectivity section.

```bash
#!/bin/sh /etc/rc.common

START=99

USE_PROCD=1

start_service() {
    logger -t modem_gps "Waiting for Quectel modem..."
    found=0
    for i in $(seq 1 60); do
        if lsusb | grep -qi quectel; then
            found=1
            logger -t modem_gps "Quectel modem found."
            break
        fi
        sleep 1
    done

    if [ "$found" -eq 0 ]; then
        logger -t modem_gps "Modem not found after 60s, rebooting."
        reboot
        exit 1
    fi

    logger -t modem_gps "Waiting 10 seconds before enabling modem."
    sleep 10
    logger -t modem_gps "Enabling GPS on modem via /dev/ttyUSB2."
    echo -e "AT+QGPS=1\r" > /dev/ttyUSB2
    logger -t modem_gps "GPS enable command sent."
}
```

```bash
/etc/init.d/modem-gps enable
/etc/init.d/modem-gps start
# In case you get stuck in a reboot loop:
/etc/init.d/modem-gps disable
```

### `luci-app-gpoint`
This is a great LUCI app to monitor GPS status and to even send status updates to a remote server such as [traccar](https://www.traccar.org/) via a wireguard (or similar) VPN connection. This is an ongoing area of improvement for this guide, however here are a few helpful links:
* [luci-app-gpoint](https://github.com/Kodo-kakaku/luci-app-gpoint) - Needs to be downloaded from github and installed manually
  * Configuration:
    * Parser mode: gpoint
    * Timezone: (Your timezone)
    * Modem: (Should be auto selected)
    * Modem port: `/dev/ttyUSB1`
    * Enable Server: (Needed for traccar integration, optional)
    * Geohash Filter (opinionated basic settings):
      * Enabled
      * Jump: 3
      * Area: 7
      * Speed: 2
    * Kalman Filter:
      * Enabled
      * Noise: 1.0
* [Setup traccar in docker](https://www.traccar.org/docker/)

## Fan Control

Your Banana Pi R3 Mini has no on-board fan controller, so thermal management relies on software.  
The combination of a shell script (`fancontrol.sh`) and a procd init service keeps the SoC within safe limits while minimising noise.

### `/usr/bin/fancontrol.sh` Script

- **Sensor Source** – Reads the current SoC temperature from `/sys/class/thermal/thermal_zone0/temp` (raw millidegrees).  
- **Four Fan Levels** – OFF, Low, Medium, High—implemented by writing to kernel *trip points* (`trip_point_2–4`).  
- **Thresholds** – Defaults are **40 °C**, **47 °C**, **54 °C**, with an emergency cut-off at **80 °C**.  
  - Tweak `LOW_TEMP`, `MEDIUM_TEMP`, `HIGH_TEMP`, and `VERY_HIGH_TEMP` to for a tradeoff between noise and device temperature
  - A `HYSTERESIS` value (4 °C) prevents rapid toggling near the boundaries.  
- **Logic Flow**
  1. Determine the current “level” based on thresholds plus hysteresis.  
  2. If the level changes, update the trip points so the kernel switches fan GPIO speeds accordingly.  
  3. Loop every two seconds.

Because the script manipulates hardware trip points instead of driving PWM directly, it plays nicely with the kernel’s thermal framework and avoids busy-looping on GPIO writes.

The script has been adapted from user cwxiaos on the [Banana Pi forum](https://forum.banana-pi.org/t/banana-pi-bpi-r3-mini-5g-module-heatsink-and-fan/17738/20)

```bash
#!/bin/ash

# Temperature sensor path
TEMP_PATH="/sys/class/thermal/thermal_zone0/temp"

# Trip Points for Fan Speed Control
TRIP_0="/sys/class/thermal/thermal_zone0/trip_point_0_temp"
TRIP_1="/sys/class/thermal/thermal_zone0/trip_point_1_temp"
TRIP_2="/sys/class/thermal/thermal_zone0/trip_point_2_temp"
TRIP_3="/sys/class/thermal/thermal_zone0/trip_point_3_temp"
TRIP_4="/sys/class/thermal/thermal_zone0/trip_point_4_temp"

# Temperature thresholds (in millidegrees Celsius)
LOW_TEMP=40000
MEDIUM_TEMP=47000
HIGH_TEMP=54000
VERY_HIGH_TEMP=70000
CRITICAL_TEMP=80000

HYSTERESIS=4000

# Previous fan level (-1 means uninitialized)
PREV_LEVEL=-1

set_trip_points() {
    level=$1

    case "$level" in
        0)  # Fan OFF
            echo "$VERY_HIGH_TEMP" > "$TRIP_4"
            echo "$VERY_HIGH_TEMP" > "$TRIP_3"
            echo "$VERY_HIGH_TEMP" > "$TRIP_2"
            ;;
        1)  # Low speed
            echo 0 > "$TRIP_4"
            echo "$VERY_HIGH_TEMP" > "$TRIP_3"
            echo "$VERY_HIGH_TEMP" > "$TRIP_2"
            ;;
        2)  # Medium speed
            echo 0 > "$TRIP_4"
            echo 0 > "$TRIP_3"
            echo "$VERY_HIGH_TEMP" > "$TRIP_2"
            ;;
        3)  # High speed
            echo 0 > "$TRIP_4"
            echo 0 > "$TRIP_3"
            echo 0 > "$TRIP_2"
            ;;
    esac

    PREV_LEVEL=$level
}

# Set emergency shutdown temperature
echo "$VERY_HIGH_TEMP" > "$TRIP_1"
echo "$CRITICAL_TEMP" > "$TRIP_0"

while true; do
    # Read current temperature
    if [ -f "$TEMP_PATH" ]; then
        TEMP=$(cat "$TEMP_PATH")
    else
        echo "Error: Unable to read temperature from $TEMP_PATH"
        exit 1
    fi

    # Determine fan speed level based on hysteresis control
    case "$PREV_LEVEL" in
        0)
            [ "$TEMP" -ge "$LOW_TEMP" ] && LEVEL=1 || LEVEL=0
            ;;
        1)
            if [ "$TEMP" -ge "$MEDIUM_TEMP" ]; then
                LEVEL=2
            elif [ "$TEMP" -le "$((LOW_TEMP - HYSTERESIS))" ]; then
                LEVEL=0
            else
                LEVEL=1
            fi
            ;;
        2)
            if [ "$TEMP" -ge "$HIGH_TEMP" ]; then
                LEVEL=3
            elif [ "$TEMP" -le "$((MEDIUM_TEMP - HYSTERESIS))" ]; then
                LEVEL=1
            else
                LEVEL=2
            fi
            ;;
        3)
            [ "$TEMP" -le "$((HIGH_TEMP - HYSTERESIS))" ] && LEVEL=2 || LEVEL=3
            ;;
        *)
            LEVEL=0  # Default to fan OFF
            ;;
    esac

    # Update fan speed only if the level changes
    [ "$LEVEL" -ne "$PREV_LEVEL" ] && set_trip_points "$LEVEL"

    sleep 2
done

### Init Service

The companion init script (`/etc/init.d/fancontrol`) wraps `fancontrol.sh` in a supervised procd instance.

```bash
#!/bin/sh /etc/rc.common

START=99
STOP=10

USE_PROCD=1

start_service() {
    procd_open_instance
    procd_set_param command /usr/bin/fancontrol.sh
    procd_set_param respawn
    procd_close_instance
}
```

- **Startup Order** – `START=99` ensures the service runs late, after the kernel exposes thermal zones.  
- **Respawn** – procd restarts the script automatically if it crashes.  
- **Enable at Boot**
  ```bash
  chmod +x /usr/bin/fancontrol.sh
  chmod +x /etc/init.d/fancontrol
  /etc/init.d/fancontrol enable
  /etc/init.d/fancontrol start
  ```


## Storage (NVMe SSD)
**TODO**
## Recovery & Unbrick Guide
**TODO**
## Useful Links
* [ImmortalWRT Downloads](https://downloads.immortalwrt.org/)
* [BanaPi R3/Mini Forum](https://forum.banana-pi.org/c/banana-router/bpi-r3/64)
## TODO & Future Improvements
* Dedicated watchdog script to try to recover from failed modem detection
* GPS/Wireguard/traccar setup
* Storage (NVMe SSD)
* sysupgrade.conf documentation for config to survive upgrades
* More info on case mods