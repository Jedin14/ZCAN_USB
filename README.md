# zcan_usb — Linux SocketCAN Driver for NXP USB CANFD DEBUG

A fully reverse-engineered Linux kernel driver for the **NXP USB CANFD DEBUG**
adapter (USB ID `04d8:0053`), also sold as:
- Zhiyuan Electronics USBCANFD-200U
- Waveshare USB-CAN-FD-B

The vendor provides a proprietary Linux library (`libcontrolcanfd.so`) but no
kernel driver. This driver exposes the device as standard SocketCAN interfaces
(`can0`, `can1`), compatible with all Linux CAN tools (`candump`, `cansend`,
`python-can`, etc.).

> **All protocol information was reverse-engineered from USB traffic captures
> and static analysis of `libcontrolcanfd.a`. See [PROTOCOL.md](PROTOCOL.md)
> for full details.**

---

## Features

- Two independent CAN channels (`can0` / `can1`)
- Classic CAN (up to 1 Mbit/s)
- Standard SocketCAN interface — works with `can-utils`, `python-can`, etc.
- Automatic DKMS support for kernel updates

---

## Requirements

- Linux kernel ≥ 5.10
- Kernel headers for your running kernel
- `can-utils` (optional, for testing)

```bash
# Arch Linux
sudo pacman -S linux-headers can-utils

# Ubuntu / Debian
sudo apt install linux-headers-$(uname -r) can-utils
```

---

## Quick Install

On a new PC, simply clone this repository and run the provided install script. This will compile the driver and register it with DKMS so that it automatically rebuilds when you update your kernel.

```bash
git clone https://github.com/Jedin14/ZCAN_USB.git
cd ZCAN_USB
sudo ./install_driver.sh
```

That's it! The driver will be loaded, and `can0` / `can1` will be available whenever you plug in the adapter. You can then bring the interfaces up with standard `ip link` commands (e.g., `sudo ip link set can0 type can bitrate 1000000 up`).

## Build (Manual) & Install

### Manual

```bash
git clone https://github.com/youruser/zcan_usb
cd zcan_usb
make
sudo insmod zcan_usb.ko
```

### DKMS (survives kernel updates)

```bash
sudo cp -r . /usr/src/zcan_usb-0.5.1
sudo dkms add -m zcan_usb -v 0.5.1
sudo dkms build -m zcan_usb -v 0.5.1
sudo dkms install -m zcan_usb -v 0.5.1
```

### Load at boot (after DKMS install)

```bash
echo "zcan_usb" | sudo tee /etc/modules-load.d/zcan_usb.conf
```

---

## Usage

```bash
# Bring up CAN interface at 500 kbit/s
sudo ip link set can0 type can bitrate 500000
sudo ip link set can0 up

# Receive frames
candump can0

# Send a frame
cansend can0 7DF#0201050000000000

# Bring down
sudo ip link set can0 down
```

Verify the driver loaded correctly:
```bash
dmesg | grep zcan
# usb 1-1: device handshake OK
# usb 1-1: fw=0x0200 hw=0x0212 serial=FD212606182356USBCAN
# usb 1-1: registered can0 (channel 0)
# usb 1-1: registered can1 (channel 1)
# usb 1-1: ZCAN USB CANFD connected (2 channels) [v0.5.1]
```

---

## Supported Bitrates

| Bitrate    | Command               |
|------------|-----------------------|
| 125 kbit/s | `bitrate 125000`      |
| 250 kbit/s | `bitrate 250000`      |
| 500 kbit/s | `bitrate 500000`      |
| 1 Mbit/s   | `bitrate 1000000`     |

---

## How It Works

The device uses a custom USB bulk transfer protocol with BEEF/DEAD framing and
AES-128 ECB authentication on open. See [PROTOCOL.md](PROTOCOL.md) for the
complete reverse-engineered protocol specification including:

- Packet framing and checksum algorithm
- AES-128 challenge-response handshake
- All command payloads (INIT_CAN, SET_BAUD, START_CAN, TRANSMIT, etc.)
- TX and RX frame formats
- Endpoint routing
- Initialization sequence quirks

---

## Reverse Engineering Notes

The driver was reverse-engineered using:

1. **USB traffic captures** — Wireshark with `usbmon` on the original
   vendor library (`libcontrolcanfd.so`) running on Linux
2. **Static analysis** — `objdump`, `nm`, and GDB on the static library
   `libcontrolcanfd.a` to recover function names, identify the AES key,
   decode the checksum algorithm, and understand command payloads

Key findings that were non-obvious:
- The device sends 2-byte polling packets on EP1 IN every ~16ms; these must
  be discarded when waiting for command responses
- Both channels must be initialized (INIT+BAUD+START) even when only one is
  used, or RX stops after TX
- TX frames use `0xF1` marker (classic CAN), not `0xF2` (CANFD), with a
  26-byte payload — not the 86-byte CANFD payload
- `transmit_type = 0x00` (auto-retry) is required; `0x01` causes ~50% packet
  loss
- The `type` byte in INIT_CAN must be `0x01` even for classic CAN

---

## License

GPL v2 — see [SPDX header](zcan_usb.c).
