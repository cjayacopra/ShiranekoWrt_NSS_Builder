# Customizing the build

All knobs live in the **`env:` block of [`.github/workflows/build.yml`](../.github/workflows/build.yml)**,
the **[shared `.config`](../devices/common/config)** plus its per-device
companion, and the overlay directories. This page shows what each one does.

## Build parameters (`build.yml` → `env:`)

```yaml
env:
  UPSTREAM_REPO: JuliusBairaktaris/openwrt-nss-edma   # OpenWrt source tree
  UPSTREAM_REF: nss-edma-rework                       # branch, tag, or 40-char SHA
  NSS_REPO: JuliusBairaktaris/nss-packages            # NSS packages repo (blank to disable)
  NSS_REF: edma-nss
  VARIANT: edma-nss                                   # selects devices/<id>/files.<variant>
  RELEASE_PREFIX: edma-nss                            # tag = <prefix>-<ts>-<run id>
  KEEP: "2"                                           # newest releases to retain
  FEEDS: |                                            # newline-separated feed lines
    src-git nss https://github.com/JuliusBairaktaris/nss-packages.git;edma-nss
    src-link aw1000 $BUILDER_REPO/feed/aw1000         # vendored, edit in place
    src-git footstrap https://github.com/VizzleTF/luci-theme-footstrap.git;<pin>
    src-git footstrap_files https://github.com/VizzleTF/luci-app-footstrap-files.git;<pin>
    src-git luci_smstool https://github.com/4IceG/luci-app-sms-tool-js.git;<pin>
```

`prepare-build.sh` expands `$BUILDER_REPO` in feed lines before writing them
to `feeds.conf` (`scripts/feeds` does not expand env vars). The aw1000 feed
is the repo's own `feed/aw1000/` directory — the modem/WWAN packages live
here, so a package fix is an ordinary commit in this repo.

What gets built is the `build` job's matrix — one entry, the AW1000, whose
group config is `devices/aw1000/config`:

```yaml
    strategy:
      matrix:
        device: [aw1000]
```

The subtarget is part of the group's config (`CONFIG_TARGET_qualcommax_ipq807x=y`),
not of the shared one.

A group exists per device here — the AW1000 is the only board. Its config pins
the ath11k 1 GB memory profile and the NSS high memory profile, which are
compile-time and image-wide, so a second board with a different RAM size would
need its own group config anyway. Adding one is a new `devices/<id>/config`
with the target, the single device symbol and its packages, plus a matrix
entry.

Getting that right is on you. `kmod-ath11k` carries per-board profile selects,
but they key off the single-profile choice symbol
(`CONFIG_TARGET_qualcommax_ipq807x_DEVICE_<id>`), which the AW1000's build does
set — with `CONFIG_TARGET_MULTI_PROFILE` off, the one selected device's selects
apply. If you ever add a second device to a group without per-device rootfs,
the profile is whatever the group config says, and a board in the wrong one
gets an image built for the wrong amount of RAM.

`CONFIG_TARGET_PER_DEVICE_ROOTFS` is not used: with a single device per build
every image is that device's, so the pinning mechanism it exists for is
irrelevant.

When the build runs depends on the trigger:
- **schedule** → `check` skips the build when the upstream is unchanged since the last release
  (this is the "rebuild when upstream moves" path).
- **push** → always rebuilt — a push to this repo means the config, overlays, or scripts changed,
  so the image must be regenerated.
- **Run workflow** (`workflow_dispatch`) → always rebuilt.

## Device `.config`

The `.config` is assembled from two files: everything device-independent is in
[`devices/common/config`](../devices/common/config), and `devices/<id>/config`
adds the device list, the memory profiles and any board-specific package
pruning. `prepare-build.sh` concatenates them (the device file wins on a symbol
set in both) and runs `make defconfig`. To change something, edit in a real
OpenWrt checkout and diff back:

```sh
git clone --branch nss-edma-rework https://github.com/JuliusBairaktaris/openwrt-nss-edma openwrt
cat devices/common/config devices/aw1000/config > openwrt/.config
cd openwrt && make menuconfig
./scripts/diffconfig.sh > /tmp/full.config        # minimal .config (deltas only)
# then split the delta back over devices/common/config and devices/<id>/config
```

A symbol whose dependencies are unmet is dropped silently by `make defconfig`,
which is how an image ships without an option someone asked for. Every
requested `=y` symbol is therefore re-checked against the resolved `.config`,
and a missing one fails the build.

Anything a board does not have belongs in its own config, never in the shared
one: `DEVICE_PACKAGES` pulls in each board's ath10k firmware, USB and 10G PHY
support, and disabling those globally ships a broken image for every board that
does have the hardware.

## Enabling offload extras

The image ships the full default offload stack (NAT/PPPoE/VLAN via ECM, wired
bridge offload, same-subnet multicast, NSS SQM, Wi-Fi). A few more offloads are
wired in the source tree but left **off by default** because the reference
network does not use them. Each is enabled by adding its package to
`devices/common/config` and rebuilding:

| Feature | Add | How it offloads |
|---|---|---|
| Routed L3 multicast (IPTV WAN→LAN) | `CONFIG_PACKAGE_igmpproxy=y` (or `smcroute`) | The daemon installs kernel multicast-routing (MFC) entries; ECM (`ECM_MULTICAST_ENABLE`, already on) turns each into a PPE hardware rule. Needs a real upstream multicast source and the receiver subnet on its own netdev (one bridge is a single ipmr VIF). |
| MAP-T / 464XLAT | `CONFIG_PACKAGE_kmod-nat46=y` | nat46 headers and QCA MAP-T exports are staged in the openwrt tree for ECM. |
| VXLAN | `CONFIG_PACKAGE_kmod-vxlan=y` | fdb/age-update offload via kernel patch `0972`. |
| MACVLAN | `CONFIG_PACKAGE_kmod-macvlan=y` | ECM MACVLAN support via kernel patch `0962`. |
| GRE | `CONFIG_PACKAGE_kmod-gre=y` | ECM GRE support. |

Each package brings its own init/userland; `nss-up` does not start them. Not
available on this platform/firmware: IPsec (ESP) offload, TLS/DTLS, CoDel ECN
marking (see the wiki Limitations page). Wi-Fi mesh offload is not built: it
needs the 11.4 firmware line, and this repo ships 12.5 only.

> Routed multicast note: `nss-up` carries a **commented-out** start stage for a
> multicast daemon — it is not run in the default image (no IPTV source on the
> reference network). To enable routed multicast in a fork: add the package
> above, ship your own `/etc/config/igmpproxy` for your topology, disable the
> daemon's boot autostart (so it starts only after the WAN is up), and uncomment
> the stage in `nss-up`.

## Custom feeds

Each `src-git <name> <url>` line in `FEEDS` is appended to `feeds.conf` and updated/installed
individually. The corresponding `CONFIG_FEED_<name>` is then set to `n` so you don't bundle every
package from the feed — only what you explicitly enable in `.config` ships.

## Rootfs overlay

Four overlay layers, applied in order (later wins), each optional:

1. `devices/common/files/` — every image
2. `devices/common/files.edma-nss/` — every image, variant-specific
3. `devices/<id>/files/` — one device
4. `devices/<id>/files.edma-nss/` — one device, variant-specific

Anything under these is copied to the image root, preserving paths. Special handling:
- `etc/ssh/sshd_config` is `chmod 0600`'d automatically
- `etc/uci-defaults/<name>` files run once on first boot, then are deleted
- `etc/rc.local` runs on every boot

## Cron schedule

GitHub Actions requires `cron:` values to be static. Edit [`.github/workflows/build.yml`](../.github/workflows/build.yml):

```yaml
on:
  schedule:
    - cron: "0 */2 * * *"            # every 2 hours
```

Use [crontab.guru](https://crontab.guru) if you're unsure.

## Disabling caching

This build does not use `actions/cache` and explicitly sets `# CONFIG_CCACHE is not set` in the
config. ccache without persistent storage is a no-op on fresh runners, and `actions/cache` for
OpenWrt's multi-GB build dir is a footgun (easily corrupts mid-build).

## Disabling the schedule

If you only want manual builds, remove the `schedule:` block from `build.yml` (keep
`workflow_dispatch` and `push`).
