# ShiranekoWrt NSS Builder

### OpenWrt firmware with Qualcomm NSS hardware offload for the Arcadyan AW1000, with the AW1000 modem/WWAN stack

[![Build](https://img.shields.io/github/actions/workflow/status/JuliusBairaktaris/Qualcommax_NSS_Builder/build.yml?branch=main&style=flat-square&logo=github&label=Build)](https://github.com/JuliusBairaktaris/Qualcommax_NSS_Builder/actions/workflows/build.yml)
[![Lint](https://img.shields.io/github/actions/workflow/status/JuliusBairaktaris/Qualcommax_NSS_Builder/lint.yml?branch=main&style=flat-square&logo=github&label=Lint)](https://github.com/JuliusBairaktaris/Qualcommax_NSS_Builder/actions/workflows/lint.yml)
[![License](https://img.shields.io/github/license/JuliusBairaktaris/Qualcommax_NSS_Builder?style=flat-square&label=License)](LICENSE)
[![Last Commit](https://img.shields.io/github/last-commit/JuliusBairaktaris/Qualcommax_NSS_Builder?style=flat-square&label=Last%20Commit)](https://github.com/JuliusBairaktaris/Qualcommax_NSS_Builder/commits/main)
[![Downloads](https://img.shields.io/endpoint?url=https%3A%2F%2Fraw.githubusercontent.com%2FJuliusBairaktaris%2FQualcommax_NSS_Builder%2Fstats%2Fdownloads.json&style=flat-square)](https://github.com/JuliusBairaktaris/Qualcommax_NSS_Builder/releases)

OpenWrt sysupgrade images with NSS hardware offload (NAT, PPPoE, SQM, bridge
and Wi-Fi) for the Arcadyan AW1000 (`qualcommax/ipq807x`, 1 GB RAM), plus the
modem/WWAN stack vendored in [`feed/aw1000/`](feed/aw1000):
`qmi_wwan_q` with QMAP, `quectel-cm`, the AW1000 modem LuCI pages and board
defaults. The offload runs on OpenWrt main's upstream
`qca_edma`/`qca_ppe` ethernet drivers instead of the vendor
`qca-nss-dp`/`qca-ssdk` stack. Images are built by GitHub Actions from
[openwrt-nss-edma](https://github.com/JuliusBairaktaris/openwrt-nss-edma)
and [nss-packages](https://github.com/JuliusBairaktaris/nss-packages).

Background and design are in the
[wiki](https://github.com/JuliusBairaktaris/openwrt-nss-edma/wiki), starting
with [NSS Offload Explained](https://github.com/JuliusBairaktaris/openwrt-nss-edma/wiki/NSS-Offload-Explained).

## Install

Download `...aw1000...-squashfs-sysupgrade.bin` from a
[release](https://github.com/JuliusBairaktaris/Qualcommax_NSS_Builder/releases)
and flash it from LuCI (System → Backup / Flash Firmware) or over ssh:

```sh
sysupgrade -n /tmp/openwrt-qualcommax-ipq807x-arcadyan_aw1000-squashfs-sysupgrade.bin
```

There is no factory image. The device must already run OpenWrt; install stock
OpenWrt first using its [device page](https://openwrt.org/toh/start).

| Release | Use it for |
|---|---|
| `edma-nss-*` | The default. NSS firmware 12.5 (`NSS.FW.12.5-210-HK.R`). |
| `ppe-offload-test` | One prerelease, replaced in place. Stock OpenWrt with PPE flowtable offload and no NSS, for testing only. |

Wi-Fi ships disabled with no key. Connect by cable, set an SSID and key under
Network → Wireless, and enable the radios.

<details>
<summary>Device: Arcadyan AW1000 (IPQ807x, 1 GB RAM)</summary>

The AW1000 is the only board this repo builds. Its config lives in
`devices/aw1000/config`; the ath11k 1 GB memory profile and the NSS high
memory profile are both image-wide build options, which is why there is one
group and no per-board variants.

The modem stack is vendored in `feed/aw1000/`: `kmod-qmi_wwan_q` (QMAP muxing
for Quectel modems), `quectel-cm` for data connection setup, `sms-tool` with
the AW1000 modem LuCI pages for SMS/USSD/signal, `aw1000-defaults` for the
board's modem and LED defaults, and `udp-broadcast-relay-redux` for
LAN multicast. Upstream `uqmi` and `kmod-usb-net-qmi-wwan` are pinned off so
they cannot race the vendored `qmi_wwan_q` driver for the modem's QMI
interface.

</details>

## Status and recovery

`nss-status` over ssh, or LuCI Status → NSS Offload, shows the offload state.
Boot messages are in `logread -e nss`. To run the plain host stack instead:

```sh
uci set nss.general.enabled='0'; uci commit nss; reboot
```

The setting survives sysupgrade.

## What ships

| Area | Packages |
|---|---|
| NSS core | `kmod-qca-nss-drv`, firmware memory profile matched to RAM |
| NAT offload | ECM (IPv4 NAT, IPv6 routing), PPPoE |
| Bridge, multicast | `kmod-qca-nss-drv-bridge-mgr`, `kmod-qca-nss-ecm` |
| SQM | NSS qdiscs with `sqm-scripts-nss` and `luci-app-sqm`, shipped disabled; set your line rates and enable it |
| QoS marking | `nssqos`, `luci-app-nssqos` (DSCP rules for offloaded flows) |
| Wi-Fi | ath11k NSS offload (wifili) |
| Modem/WWAN | `kmod-qmi_wwan_q` (QMAP), `quectel-cm`, `sms-tool`, `luci-proto-quectel`, `luci-app-aw1000-modem`, `aw1000-defaults`, `udp-broadcast-relay-redux` |
| Diagnostics | `nss-status`, `luci-app-nss`, `nssinfo` |
| Base | OpenSSH instead of Dropbear, OpenSSL, LuCI over HTTPS, BCP38, `htop`, `iperf3`, `curl` |
| Toolchain | GCC 15, Binutils 2.46, mold, LTO, PIE/SSP/FORTIFY_SOURCE=3/RELRO |

Routed multicast (IPTV), MAP-T, VXLAN, MACVLAN and GRE build but are off by
default. [docs/CUSTOMIZE.md](docs/CUSTOMIZE.md) shows how to enable them.

## Measured results

AW1000, NSS firmware 12.5:

| Metric | Host path | NSS offload |
|---|---|---|
| 311 Mbit/s PPPoE NAT | ~42 % of one core (softirq) | ~99.7 % CPU idle |
| SQM at 285 Mbit/s ingress | CPU-bound | 258 Mbit/s goodput, ~99 % idle |
| RTT under shaped load | bufferbloat | 16 ms avg vs 20 ms idle |
| Wi-Fi data path | mac80211/ath11k on the CPU | wifili on the NSS cores |

## Build it yourself

Fork this repo and edit the `env:` block of
[build.yml](.github/workflows/build.yml), or build locally:

```sh
git clone --branch nss-edma-rework https://github.com/JuliusBairaktaris/openwrt-nss-edma openwrt
cd openwrt
cp feeds.conf.default feeds.conf
echo "src-git nss https://github.com/JuliusBairaktaris/nss-packages.git;edma-nss" >> feeds.conf
echo "src-link aw1000 ../ShiranekoWrt_NSS_Builder/feed/aw1000" >> feeds.conf
./scripts/feeds update -a && ./scripts/feeds install -a
B=../ShiranekoWrt_NSS_Builder/devices
cat "$B/common/config" "$B/aw1000/config" > .config
make defconfig && make -j"$(nproc)"
```

> [!IMPORTANT]
> `nss-edma-rework` and the `edma-nss` feed are rebased regularly, so
> `git pull` and `./scripts/feeds update` do not update them. Use:
>
> ```sh
> git fetch origin && git reset --hard origin/nss-edma-rework
> rm -rf feeds/nss package/feeds/nss
> ./scripts/feeds update -a && ./scripts/feeds install -a
> make defconfig
> ```

[docs/CUSTOMIZE.md](docs/CUSTOMIZE.md) covers build options and overlays.
[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) describes the CI pipeline.

## Contributing

Issues and PRs are welcome, see [CONTRIBUTING.md](CONTRIBUTING.md).

## Acknowledgements

- [Christian Marangi (Ansuel)](https://github.com/Ansuel) for the upstream
  EDMA and PPE drivers.
- [Robert Marko (robimarko)](https://github.com/robimarko) for maintaining the
  OpenWrt qualcommax target.
- [qosmio](https://github.com/qosmio) for NSS development, the
  [openwrt-ipq](https://github.com/qosmio/openwrt-ipq) tree and the Wi-Fi
  offload patches
- [rodriguezst](https://github.com/rodriguezst) for the original
  [ipq807x-openwrt-builder](https://github.com/rodriguezst/ipq807x-openwrt-builder)
- The OpenWrt community in the
  [NSS build thread](https://forum.openwrt.org/t/qualcommax-nss-build/148529)

## Support

This is an unpaid single-maintainer project. Donations go toward OpenWrt test
hardware.

- [GitHub Sponsors](https://github.com/sponsors/JuliusBairaktaris)
- [PayPal](https://paypal.me/JuliusBairaktaris)

## License

[GPL-2.0](LICENSE), as OpenWrt.
