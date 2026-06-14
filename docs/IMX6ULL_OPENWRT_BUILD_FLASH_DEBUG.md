# i.MX6ULL OpenWrt 编译、烧录与 PassWall 调试手册

本文记录本仓库在 WSL Ubuntu 22.04 中编译适用于正点原子 i.MX6ULL 开发板的 OpenWrt 固件，并烧录到 SD 卡或 eMMC、完成网络配置及 PassWall 调试的完整流程。

当前验证环境：

- OpenWrt：23.05-SNAPSHOT
- 目标：`imx/cortexa7`
- 设备：`imx6ull-atk-mmc`
- 内核：5.15
- 根分区：1024 MB
- WSL 源码目录：`/home/hank/imx6ull_openwrt-imx6ull_openwrt`
- 固件文件：`openwrt-23.05-snapshot-unknown-imx-cortexa7-imx6ull-atk-mmc-squashfs-combined.bin`
- PassWall：26.6.2
- Xray：24.12.31

## 1. 编译环境

推荐使用 Ubuntu 22.04。安装依赖：

```sh
sudo apt update
sudo apt install -y build-essential clang flex bison g++ gawk gcc-multilib \
  gettext git libncurses5-dev libssl-dev python3-distutils rsync unzip zlib1g-dev \
  file wget curl subversion swig time xsltproc zstd
```

进入源码目录并使用纯 Linux PATH，避免 Windows 程序污染 OpenWrt 构建：

```sh
cd /home/hank/imx6ull_openwrt-imx6ull_openwrt
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

## 2. 更新 feeds 并应用 PassWall 兼容补丁

`feeds.conf.default` 已包含 OpenWrt 23.05 feeds 和 PassWall feeds：

```text
src-git passwall_packages https://github.com/Openwrt-Passwall/openwrt-passwall-packages.git;main
src-git passwall_luci https://github.com/Openwrt-Passwall/openwrt-passwall.git;main
```

更新并安装 feeds：

```sh
./scripts/feeds update -a
./scripts/feeds install -a
```

PassWall 26.6.2 默认生成的新式 Xray 出站格式不能由 OpenWrt 23.05 自带的 Xray 24.12.31 正确使用。本仓库提供兼容补丁：

```sh
git -C feeds/passwall_luci apply ../../patches/passwall-xray24-compat.patch
```

补丁完成三项处理：

1. 将 VMess/VLESS 出站恢复为传统 `vnext[].users[]` 格式。
2. 将 Socks、HTTP、Shadowsocks、Trojan 出站恢复为传统 `servers[]` 格式。
3. 允许为使用自签名证书的节点生成 `allowInsecure=true`。

若补丁提示已应用，可检查：

```sh
grep -n 'alterId\|allowInsecure' feeds/passwall_luci/luci-app-passwall/luasrc/passwall/util_xray.lua
```

## 3. 固件配置

仓库根目录中的 `.config` 是已经验证的编译配置，包含 PassWall、Xray、SmartDNS、DDNS、UPnP、SQM、TTYD、USB/QMI 和 RTL8188EU 等插件。

直接整理配置：

```sh
make defconfig
```

需要重新配置时：

```sh
make menuconfig
```

关键配置：

```text
Target System: NXP i.MX
Subtarget: Cortex-A7
Target Profile: Freescale ATK IMX6ULL Board
CONFIG_TARGET_ROOTFS_PARTSIZE=1024
```

1024 MB 根分区同时适用于 8 GB eMMC 和 32 GB SD 卡。剩余空间不会自动加入根分区，可以后续创建数据分区。

## 4. 编译固件

首次下载源码包：

```sh
make download -j"$(nproc)"
```

开始编译：

```sh
make -j"$(nproc)"
```

并行编译失败时，用单线程获取明确错误：

```sh
make -j1 V=s
```

只重新编译 PassWall：

```sh
make package/feeds/passwall_luci/luci-app-passwall/compile V=s -j"$(nproc)"
```

固件输出目录：

```text
bin/targets/imx/cortexa7/
```

主要文件：

```text
openwrt-23.05-snapshot-unknown-imx-cortexa7-imx6ull-atk-mmc-squashfs-combined.bin
openwrt-23.05-snapshot-unknown-imx-cortexa7-imx6ull-atk-mmc-squashfs-sysupgrade.bin
openwrt-23.05-snapshot-unknown-imx-cortexa7-imx6ull-atk-mmc.manifest
sha256sums
```

校验文件：

```sh
cd bin/targets/imx/cortexa7
sha256sum -c sha256sums
```

## 5. 烧录 SD 卡

推荐先使用 SD 卡验证固件，再写入 eMMC。

### Windows 图形界面

1. 使用读卡器连接 SD 卡。
2. 打开 balenaEtcher。
3. 选择 `*-squashfs-combined.bin`。
4. 选择正确的 SD 卡，确认不要选中系统硬盘。
5. 开始烧录并等待校验完成。
6. 将开发板拨码切换到 SD 启动并上电。

### Linux 命令行

先用 `lsblk` 确认 SD 卡设备，再执行：

```sh
sudo dd if=openwrt-23.05-snapshot-unknown-imx-cortexa7-imx6ull-atk-mmc-squashfs-combined.bin \
  of=/dev/sdX bs=4M status=progress conv=fsync
sync
```

`/dev/sdX` 必须替换为整张 SD 卡设备，写错会破坏其他磁盘数据。

## 6. 从 SD 卡写入 eMMC

本开发板中通常：

- `mmcblk0`：启动用 SD 卡
- `mmcblk1`：板载 8 GB eMMC

必须以实际 `lsblk` 输出为准：

```sh
lsblk
```

把 combined 固件上传到开发板 `/tmp`，然后再次确认 eMMC 设备没有被挂载。写入 eMMC 会清除其中全部数据：

```sh
umount /dev/mmcblk1p1 2>/dev/null
umount /dev/mmcblk1p2 2>/dev/null

dd if=/tmp/openwrt-23.05-snapshot-unknown-imx-cortexa7-imx6ull-atk-mmc-squashfs-combined.bin \
  of=/dev/mmcblk1 bs=4M conv=fsync
sync
```

关机，将拨码切换到 eMMC 启动后重新上电：

```sh
poweroff
```

## 7. 首次启动与网络

典型网络：

- 上级路由器网关：`192.168.2.1`
- OpenWrt WAN `eth0`：从上级路由器获取 `192.168.2.x`
- OpenWrt LAN `br-lan`：`192.168.1.1`

查看地址：

```sh
ifconfig
ip route
```

如果开发板 WAN 地址是 `192.168.2.129`，上级网络中的电脑可打开：

```text
http://192.168.2.129
```

LAN/Wi-Fi 客户端应把网关和 DNS 设置为：

```text
192.168.1.1
```

## 8. 设置 Wi-Fi 密码

网页进入“网络 -> 无线 -> 修改 -> 无线安全”，选择 WPA2-PSK/CCMP 并设置至少 8 位密码。

命令行设置：

```sh
WIFI="$(uci show wireless | sed -n "s/^\(wireless\.[^=]*\)=wifi-iface$/\1/p" | head -1)"
uci set "$WIFI.encryption=psk2"
uci set "$WIFI.key=请替换为你的WiFi密码"
uci commit wireless
wifi reload
```

## 9. PassWall 初始配置

在 PassWall 中添加订阅并选择 TCP/UDP 节点。建议启用本机代理和 SOCKS，便于测试。

当前验证配置：

```sh
uci set passwall.@global[0].enabled='1'
uci set passwall.@global[0].localhost_proxy='1'
uci set passwall.@global[0].dns_mode='udp'
uci set passwall.@global[0].direct_dns='192.168.2.1:53'
uci set passwall.@global[0].direct_dns_mode='udp'
uci set passwall.@global[0].remote_dns='1.1.1.1:53'
uci set passwall.@global[0].dns_redirect='0'
uci commit passwall
```

某些机场 VMess 节点使用自签名 TLS 证书，需要对当前节点允许不安全证书：

```sh
NODE="$(uci get passwall.@global[0].tcp_node)"
uci set passwall.$NODE.tls_allowInsecure='1'
uci commit passwall
/etc/init.d/passwall restart
```

检查生成配置：

```sh
CFG=/tmp/etc/passwall/acl/default/TCP_UDP_SOCKS.json
jsonfilter -i "$CFG" -e '@.outbounds[0].settings.vnext[0].users[0].alterId'
jsonfilter -i "$CFG" -e '@.outbounds[0].streamSettings.tlsSettings.allowInsecure'
```

预期分别输出 `0` 和 `true`。

## 10. PassWall 验证

检查进程和端口：

```sh
ps | grep '[x]ray'
netstat -lntp | grep -E '1041|1070|15353'
```

检查 DNS：

```sh
nslookup www.google.com 127.0.0.1
nslookup api.ipify.org 127.0.0.1
```

检查透明代理和 SOCKS：

```sh
curl -I https://www.google.com
curl -I --socks5-hostname 127.0.0.1:1070 https://www.google.com
curl -4 https://checkip.amazonaws.com
```

本次验证中，原始公网 IP 为 `121.35.3.103`，代理出口 IP 为 `85.234.83.145`。出口地址会随节点变化。

## 11. 常见故障

### Xray 报 `0 VMess receiver configured`

原因：PassWall 26.6.2 输出的新式配置格式与 Xray 24.12.31 不兼容。

处理：应用 `patches/passwall-xray24-compat.patch` 后重新编译并安装 `luci-app-passwall`。

### SOCKS 存在但 TLS 握手 EOF

如果节点端口可达、UUID 正确，但请求返回：

```text
ssl_handshake returned - mbedTLS: (-0x7280)
```

检查当前节点是否使用自签名证书，并启用：

```sh
NODE="$(uci get passwall.@global[0].tcp_node)"
uci set passwall.$NODE.tls_allowInsecure='1'
uci commit passwall
/etc/init.d/passwall restart
```

### PassWall 启动后 DNS 无响应

若 ChinaDNS 日志显示可信 DNS 为 `tcp://1.1.1.1#53` 且查询失败，改用 UDP：

```sh
uci set passwall.@global[0].dns_mode='udp'
uci commit passwall
/etc/init.d/passwall restart
```

确认日志：

```sh
grep 'ChinaDNS-NG' /tmp/log/passwall.log | tail -2
```

### nftables 报 `psw_black` 或 `psw_white` 空元素错误

可能看到：

```text
Error: Could not process rule: No such file or directory
add element inet passwall psw_black {
```

这是 PassWall 向 nftables 集合添加空元素产生的警告。本次验证中未阻止 Xray、DNS、透明代理和 SOCKS 工作，可通过实际连通性判断是否影响使用。

### LuCI PassWall 页面 XML 错误

切换到 Bootstrap 主题并清理缓存：

```sh
uci set luci.main.mediaurlbase='/luci-static/bootstrap'
uci commit luci
rm -rf /tmp/luci-*
/etc/init.d/uhttpd restart
```

## 12. 配置备份与恢复

生成配置备份：

```sh
sysupgrade -b /tmp/openwrt-passwall-working.tar.gz
```

恢复备份：

```sh
sysupgrade -r /tmp/openwrt-passwall-working.tar.gz
reboot
```

备份中可能包含订阅地址、节点 UUID、Wi-Fi 密码等敏感信息，不要提交到公开 Git 仓库。
