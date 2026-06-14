# i.MX6ULL Balanced PassWall Firmware Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Produce a verified i.MX6ULL OpenWrt image containing PassWall and the approved balanced routing plugin set.

**Architecture:** Extend the OpenWrt 23.05 feeds with the official PassWall source and package repositories. Start from the vendor defconfig, correct the renamed target device, set a 1024 MB root partition, then select only Xray and the approved LuCI applications.

**Tech Stack:** OpenWrt buildroot, GNU Make, OpenWrt feeds, WSL Ubuntu 22.04

---

### Task 1: Configure Feeds

**Files:**
- Modify: `feeds.conf.default`

- [ ] Add the official PassWall source and package feeds.
- [ ] Run `./scripts/feeds update -a`.
- [ ] Run `./scripts/feeds install -a`.
- [ ] Verify PassWall, Xray, SmartDNS, DDNS, UPnP, and SQM package definitions exist.

### Task 2: Create Firmware Configuration

**Files:**
- Create: `.config`

- [ ] Copy `atk_imx6ull_defconfig` to `.config`.
- [ ] Replace the stale `imx6ull-atk-emmc` target with `imx6ull-atk-mmc`.
- [ ] Set `CONFIG_TARGET_ROOTFS_PARTSIZE=1024`.
- [ ] Select PassWall, Xray, SmartDNS, DDNS, UPnP, SQM, and Chinese LuCI translations.
- [ ] Run `make defconfig`.
- [ ] Verify all requested package selections remain enabled.

### Task 3: Build Firmware

**Files:**
- Output: `bin/targets/imx/cortexa7/`

- [ ] Run `make download -j20`.
- [ ] Run `make -j20`.
- [ ] If the parallel build fails, rerun `make -j1 V=s` and fix the reported issue.

### Task 4: Verify Artifacts

- [ ] Confirm the build command exits with status 0.
- [ ] Confirm combined and sysupgrade images exist.
- [ ] Inspect the combined image partition table for the 1024 MB root partition.
- [ ] Inspect the generated manifest for all approved packages.
- [ ] Report image paths, checksums, sizes, and safe SD/eMMC flashing guidance.
