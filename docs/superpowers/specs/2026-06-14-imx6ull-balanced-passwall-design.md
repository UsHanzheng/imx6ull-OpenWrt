# i.MX6ULL Balanced PassWall Firmware Design

## Goal

Build an OpenWrt 23.05 snapshot image for the Alientek i.MX6ULL board with a
balanced set of routing plugins and a 1024 MB root filesystem image.

## Target

- Target: `imx/cortexa7`
- Device: `imx6ull-atk-mmc`
- Boot media: 32 GB SD card or 8 GB eMMC
- Root filesystem partition: 1024 MB

## Included Features

- Existing vendor LuCI, USB, QMI/4G, serial, diagnostic, and Chinese packages
- PassWall with Xray as the primary proxy core
- SmartDNS
- DDNS
- UPnP
- SQM
- TTYD, tcpdump, and usbutils

## Excluded Features

- Docker
- OpenClash
- AdGuard Home
- Samba
- ZeroTier
- Sing-box and other redundant proxy cores

## Verification

- `make defconfig` retains the intended target and selected packages.
- A full build exits successfully.
- The combined image and sysupgrade image are generated.
- The combined image partition table contains a 1024 MB root partition.
- The generated package manifest includes the requested LuCI applications and
  PassWall/Xray packages.
