# NFQUEUE on Xiaomi AX3600

This fork enables NFQUEUE support in the Xiaomi AX3600 image while preserving
the upstream Qualcomm NSS/EDMA build.

## Packages added

The change is in:

```text
devices/xiaomi_ax3600/config
```

and consists of:

```text
CONFIG_PACKAGE_kmod-nfnetlink-queue=y
CONFIG_PACKAGE_kmod-nft-queue=y
```

### kmod-nfnetlink-queue

Provides the kernel Netfilter queue interface used to hand selected packets to
a userspace process.

### kmod-nft-queue

Provides the nftables `queue` expression used to enqueue matching packets.

Together they allow:

```text
nftables rule
    ↓
NFQUEUE
    ↓
userspace processor
    ↓
verdict / modified packet
```

NFQUEUE is generic functionality. Zapret2 is one consumer, but it is not the
only possible use.

## Why this is built into the firmware

OpenWrt snapshot kernel modules must match the running kernel build exactly.
Installing a kmod from a newer snapshot after the image has aged can fail
because the repository kernel version/hash has moved on.

Building the NFQUEUE modules into the same firmware image avoids that mismatch:
the kernel and both queue modules are produced by the same build.

## Build

Run the repository's **Build** workflow.

For this fork's NFQUEUE change, the relevant job is:

```text
Build xiaomi_ax3600 (default)
```

The desired release family is:

```text
edma-nss-*
```

not:

```text
edma-nss-mesh-*
```

unless 802.11s mesh support is specifically required.

The AX3600 sysupgrade image is:

```text
openwrt-qualcommax-ipq807x-xiaomi_ax3600-squashfs-sysupgrade.bin
```

## Verification

After boot:

```sh
apk list --installed | grep -E 'kmod-(nfnetlink|nft)-queue'
```

Both packages should be present.

Then:

```sh
find /lib/modules/$(uname -r) -type f \
  | grep -E 'nfnetlink_queue|nft_queue'
```

Expected modules:

```text
nfnetlink_queue.ko
nft_queue.ko
```

For the NSS data path:

```sh
nss-status
```

A healthy NSS build should report:

```text
NSS offload status: active
...
verdict:     OFFLOAD ACTIVE
```

## Zapret2

Zapret2 is not bundled. Install and configure it separately.

On OpenWrt with nftables, a functional setup should show both a running
`nfqws2` process and nftables rules that queue traffic to its NFQUEUE number:

```sh
/etc/init.d/zapret2 status
pgrep -af nfqws2
/etc/init.d/zapret2 list_table
```

The default Zapret2 strategy may not work for every ISP. Strategy selection,
host lists and protocol/port coverage are separate from the kernel NFQUEUE
support provided by this fork.

### NSS/ECM interaction

Qualcomm NSS/ECM acceleration is a separate data path from the generic OpenWrt
flow-offloading switch. When combining packet manipulation with NSS, verify the
real router state instead of assuming one setting implies the other:

```sh
nss-status
```

If packet-processing software works and `nss-status` still reports
`OFFLOAD ACTIVE`, the two are operating together as intended.

## Updating the fork

Upstream can continue to change kernel, NSS and package revisions. When pulling
or merging upstream updates, keep these two symbols in
`devices/xiaomi_ax3600/config` and let the normal build validation confirm
they remain available.

Do **not** solve snapshot kernel drift by installing an arbitrary newer kmod
onto an older running kernel. Rebuild the firmware so the kernel and kmods are
generated together.

## Scope

At present this fork intentionally changes only the Xiaomi AX3600 build.
If NFQUEUE support is later desired for every device group, the package symbols
can instead be moved to `devices/common/config`, but that is a broader change
than the current fork intends.
