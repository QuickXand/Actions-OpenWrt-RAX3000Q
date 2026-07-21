# Build OpenWrt (QSDK 12.5) for CMCC RAX3000Q

[![Build](https://github.com/QuickXand/Actions-OpenWrt-RAX3000Q/actions/workflows/build-openwrt.yml/badge.svg)](https://github.com/QuickXand/Actions-OpenWrt-RAX3000Q/actions/workflows/build-openwrt.yml)

QSDK 12.5.2783.2994 based firmware for the CMCC RAX3000Q / QY router.

- **Target:** `ipq50xx_32` (ARM Cortex-A7, 32-bit)
- **Kernel:** in-tree `qca/src/linux-5.4`
- **Source:** [QuickXand/qsdk](https://github.com/QuickXand/qsdk/tree/rax3000q-port) `rax3000q-port`

## Status: Mid-port from QSDK 11.5 → 12.5

Initial scaffolded port passes compilation. Applied fixes:

| Fix | Commit (qsdk `rax3000q-port`) |
|-----|-------------------------------|
| `CONFIG_SKB_EXTENSIONS` Kconfig select | `983fc0a9` |
| u-boot TARGETCC quoting | `3261d339` |
| u-boot KCPPFLAGS GCC version guard | `1f46e501` |
| ARM32 DTS wrapper include path + PMU compatible | `6451c71c` |

## Usage

Trigger **Build OpenWrt** from [Actions](https://github.com/QuickXand/Actions-OpenWrt-RAX3000Q/actions) tab (manual `workflow_dispatch` or weekly cron).

## U-Boot

Set computer IP to `192.168.1.8`, use LAN1 port.  
[Reference](https://github.com/hzyitc/openwrt-redmi-ax3000/issues/73#issuecomment-2259591683)

## Acknowledgments

- [P3TERX/Actions-OpenWrt](https://github.com/P3TERX/Actions-OpenWrt)
- [everything411/qsdk](https://github.com/everything411/qsdk)
- [kkstone/immortalwrt-ipq50xx](https://github.com/kkstone/immortalwrt-ipq50xx)
