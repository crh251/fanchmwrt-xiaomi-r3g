# fanchmwrt-xiaomi-r3g

用 GitHub Actions 编译 **fanchmwrt**(OpenWrt fork)固件,用于小米路由器 R3G
(Xiaomi Mi Router 3G,`ramips/mt7621`),并内置一块 RTL8188ETV(`0bda:0179`)
USB 无线网卡的驱动,用作无线中继(连上游 2.4G WiFi)。

本仓库只存 workflow + seed `.config` + vendored 的驱动包;真正的 fanchmwrt
源码在每次 CI 运行时现拉(见 `.github/workflows/build.yml`)。

## 编译 & 刷机

1. 推到自己的 GitHub(建议 public,Actions 免费)。
2. **Actions** → **Build R3G Firmware** → **Run workflow**(分支默认 `fanchmwrt-25.12.4`)。
3. 首次编译约 1~3 小时(现拉 + 编译整套工具链);之后有 `dl/` 缓存会快些。
4. 从该次运行的 **Artifacts** 下载 `...xiaomi_mi-router-3g-...sysupgrade.bin`。
5. 刷机:已装 Breed 的话较安全。用 sysupgrade 刷入,**不要勾"保留配置"**。
   万一引导失败,进 Breed 网页重刷。

## 文件说明

- `configs/r3g.config` — seed 配置(目标设备 + 8 个软件包),CI 里用 `make defconfig`
  展开。包含设备 `xiaomi_mi-router-3g` 与:
  `kmod-usb-core / kmod-usb2 / kmod-usb3 / kmod-usb-net / kmod-usb-net-cdc-ether /
  kmod-rtl8188eu / rtl8188eu-firmware / usbutils`(这套已在 ImmortalWrt 25.12.1
  同内核 6.12 上实测能驱动该网卡)。
- `packages/rtl8188eu/` — vendored 的 vendor 驱动包(源自 ImmortalWrt,
  aircrack-ng/rtl8188eus)。fanchmwrt 不自带这个能驱动 RTL8188ETV 的驱动;主线
  `kmod-rtl8xxxu` 在这颗芯片上会 EFuse 解析失败。workflow 会在编译前把它拷到
  `openwrt/package/kernel/rtl8188eu`。
- `rtl8188eu-firmware` 是 fanchmwrt 基础树自带的独立包
  (`package/firmware/linux-firmware/realtek.mk`),配置里直接启用即可。

## ⚠️ 刷机后:USB 网卡中继只能用命令行配(切勿用 LuCI 无线页面)

**内核 6.12 上,只要让 netifd/LuCI 去管这张 USB 卡(AP 或 STA 都算),
vendor 驱动就会卡死、整机假死。** 必须用 `wpa_supplicant` 手动驱动。以下流程
在 ImmortalWrt 25.12.1 上已验证成功(fanchmwrt 同内核,做法一致):

```sh
# 1) 掐掉 hotplug 对 USB 卡的自动探测(防止自动生成会崩的 AP),并删掉自动生成的 radio2
cat > /etc/hotplug.d/ieee80211/10-wifi-detect <<'EOF'
#!/bin/sh
[ "${ACTION}" = "add" ] && [ -f /etc/board.json ] && {
    case "$DEVPATH" in
        *1e1c0000.xhci*|*usb*) exit 0 ;;
    esac
    /sbin/wifi config
    ubus call network.wireless retry
}
EOF
uci -q delete wireless.radio2; uci -q delete wireless.default_radio2; uci commit wireless

# 2) 手动连上游(驱动会自动在 phy2 上建好 wlan0)
ip link set wlan0 up
cat > /etc/wpa_supplicant-wlan0.conf <<'EOF'
ctrl_interface=/var/run/wpa_supplicant-sta
network={
    ssid="上游SSID"
    psk="上游密码"
}
EOF
wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant-wlan0.conf
# 查状态:wpa_cli -p /var/run/wpa_supplicant-sta -i wlan0 status  → wpa_state=COMPLETED

# 3) 交给 netifd 拿 IP + 套 wan 防火墙/NAT(先确认 wan 区下标)
uci set network.wwan=interface
uci set network.wwan.proto='dhcp'
uci set network.wwan.device='wlan0'
uci commit network
uci add_list firewall.@zone[1].network='wwan'
uci commit firewall
/etc/init.d/network reload
```

开机自启:在 `/etc/rc.local` 的 `exit 0` 之前加:
```sh
sleep 10
ip link set wlan0 up
wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant-wlan0.conf
```

自带 2.4G/5G(radio0/radio1)走标准 mac80211,在 LuCI 无线页面正常配置即可
(此时 radio2 已不在配置中,网页操作安全)。

## 其他

- R3G v2(DSA 板):把 `configs/r3g.config` 的设备行换成
  `CONFIG_TARGET_ramips_mt7621_DEVICE_xiaomi_mi-router-3g-v2=y`。
- 要加第三方 feed(如 passwall):在 workflow 的 `feeds update` 前追加一行
  `src-git ...`,再在 `configs/r3g.config` 里加对应 `CONFIG_PACKAGE_*=y`。
