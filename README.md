# SSV6051 WiFi Driver for X96 Mini on Armbian

Enable the onboard **South Silicon Valley SSV6051 "Cabrio"** WiFi chip on the X96 Mini TV box running modern Armbian (kernel 6.x). This chip has `SDIO_ID=3030:3030` and is not supported in the mainline kernel — this guide builds an out-of-tree module from Armbian's official patch.

## Requirements

- X96 Mini with S905W or S905X CPU
- Armbian installed (tested on `6.18.31-current-meson64`, Debian Trixie)
- Ethernet connection for the initial setup
- Confirm your WiFi chip is SSV6051 by running:

```bash
cat /sys/bus/mmc/devices/mmc2\:0001/uevent
```

Expected output:
```
MMC_TYPE=SDIO
SDIO_ID=3030:3030
SDIO_INFO1=SouthSiliconValleyInc.
SDIO_INFO2=Cabrio
```

---

## Step 1 — Install Kernel Headers and Build Tools

```bash
sudo apt update
sudo apt install -y linux-headers-current-meson64 build-essential git
```

Verify headers installed correctly:

```bash
ls /lib/modules/$(uname -r)/build
```

---

## Step 2 — Download and Extract the Driver

```bash
cd /usr/src
sudo mkdir ssv6051-driver && cd ssv6051-driver
sudo wget https://raw.githubusercontent.com/farhadbinfarooq/ssv6051-armbian-wifi/refs/heads/main/wifi-driver-ssv6051.patch
sudo patch -p5 --batch < wifi-driver-ssv6051.patch
```

The two "skipped" messages about `drivers/net/wireless/Kconfig` and `Makefile` are expected and harmless — those patches target the kernel source tree which we don't need.

---

## Step 3 — Create the Out-of-Tree Makefile

Replace the extracted Makefile with this out-of-tree version (mind the tabs):

```bash
sudo tee /usr/src/ssv6051-driver/Makefile << 'EOF'
KMODULE_NAME := ssv6051

KBUILD_TOP := $(shell pwd)
KVERSION := $(shell uname -r)
KDIR := /lib/modules/$(KVERSION)/build

ccflags-y += -DCONFIG_SSV6200_CORE
ccflags-y += -DCONFIG_SSV_CABRIO_E
ccflags-y += -DCONFIG_SSV_TX_LOWTHRESHOLD
ccflags-y += -DCONFIG_FW_ALIGNMENT_CHECK
ccflags-y += -DCONFIG_PLATFORM_SDIO_OUTPUT_TIMING=3
ccflags-y += -DCONFIG_PLATFORM_SDIO_BLOCK_SIZE=128
ccflags-y += -DSDIO_USE_SLOW_CLOCK
ccflags-y += -DCONFIG_SSV_RSSI
ccflags-y += -DCONFIG_SSV_VENDOR_EXT_SUPPORT
ccflags-y += -DRATE_CONTROL_REALTIME_UPDATA
ccflags-y += -DCONFIG_SSV6200_HAS_RX_WORKQUEUE
ccflags-y += -DUSE_THREAD_TX
ccflags-y += -DENABLE_AGGREGATE_IN_TIME
ccflags-y += -DENABLE_INCREMENTAL_AGGREGATION
ccflags-y += -DUSE_GENERIC_DECI_TBL
ccflags-y += -DFW_WSID_WATCH_LIST
ccflags-y += -DSSV6200_ECO
ccflags-y += -DHAS_CRYPTO_LOCK
ccflags-y += -DENABLE_TX_Q_FLOW_CONTROL
ccflags-y += -DUSE_MAC80211_DECRYPT_BROADCAST
ccflags-y += -I$(KBUILD_TOP) -I$(KBUILD_TOP)/include

obj-m += $(KMODULE_NAME).o

$(KMODULE_NAME)-objs := \
	ssv6051-generic-wlan.o \
	ssvdevice/ssvdevice.o \
	ssvdevice/ssv_cmd.o \
	hci/ssv_hci.o \
	smac/init.o \
	smac/dev.o \
	smac/ssv_rc.o \
	smac/ssv_ht_rc.o \
	smac/ap.o \
	smac/ampdu.o \
	smac/efuse.o \
	smac/ssv_pm.o \
	smac/sar.o \
	smac/ssv_cfgvendor.o \
	hwif/sdio/sdio.o

.PHONY: all clean

all:
	$(MAKE) -C $(KDIR) M=$(KBUILD_TOP) modules

clean:
	$(MAKE) -C $(KDIR) M=$(KBUILD_TOP) clean
EOF
```

---

## Step 4 — Build the Driver

```bash
cd /usr/src/ssv6051-driver
sudo make -j4
```

A successful build ends with:
```
LD [M]  ssv6051.ko
```

---

## Step 5 — Load the Driver

```bash
cd /usr/src/ssv6051-driver
sudo modprobe mac80211
sudo insmod ssv6051.ko
```

Verify `wlan0` appeared:

```bash
ip link show wlan0
```

Scan for networks:

```bash
sudo iw dev wlan0 scan | grep SSID:
```

---

## Step 6 — Connect to WiFi

```bash
sudo nmcli dev wifi connect "YourSSID" password "YourPassword"
```

Verify connection:

```bash
ip addr show wlan0
ping -c 3 -I wlan0 8.8.8.8
```

---

## Step 7 — Make It Permanent (Survive Reboots)

```bash
sudo cp /usr/src/ssv6051-driver/ssv6051.ko /lib/modules/$(uname -r)/kernel/drivers/net/wireless/
sudo depmod -a
echo "ssv6051" | sudo tee -a /etc/modules
```

WiFi will now come up automatically on every boot.

---

## Notes

- The `SDIO_USE_SLOW_CLOCK` flag in the Makefile is critical — without it the driver switches to 37.5 MHz after firmware upload, causing sporadic CRC errors (`EILSEQ -84`) on the X96 Mini's `meson-gx-mmc` controller.
- No external firmware binary is required — the driver initialises the chip without one.
- Both `eth0` and `wlan0` can be active simultaneously; your SSH session over ethernet is not affected when connecting WiFi.
- If you upgrade your kernel (`apt upgrade` can do this), you need to repeat steps 1 and 4–7. The source in `/usr/src/ssv6051-driver` can be reused — just run `sudo make -j4` again after installing the new headers, then copy the new `.ko` and run `depmod -a`.

---

## Tested On

| Field | Value |
|---|---|
| Device | X96 Mini |
| CPU | Amlogic S905W |
| WiFi Chip | SSV6051 (SDIO_ID 3030:3030, "Cabrio") |
| OS | Armbian 26.8.0-trunk.12 (Debian Trixie) |
| Kernel | 6.18.31-current-meson64 |
| Driver source | [armbian/build rockchip-6.19 patch](https://github.com/armbian/build/blob/main/patch/kernel/archive/rockchip-6.19/patches.armbian/wifi-driver-ssv6051.patch) |


