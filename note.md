# Porting Note — QSDK 11.5 -> QSDK 12.5

> Context-loss-safe log. Append-only. Update on every meaningful change.

## 已完成 (commits on `origin/master` and `QuickXand/qsdk@rax3000q-port`)

- fork `everything411/qsdk@12.5.2783.2994` -> `QuickXand/qsdk`
- branch `rax3000q-port` created at SHA `4d49909b`
- commit `9afbdd06` on `QuickXand/qsdk@rax3000q-port`:
  - new `qca/src/linux-5.4/arch/arm64/boot/dts/qcom/ipq5000-cmcc-rax3000q.dts`
    (board DTS, verbatim from `hzyitc/openwrt-redmi-ax3000@...11.5` `ipq5000-rax3000q.dts`)
  - new `qca/src/linux-5.4/arch/arm/boot/dts/ipq5000-rax3000q.dts`
    (ARM32 wrapper, 12.5 style: `#include <arm64/qcom/...>` + `#include "ipq5018.dtsi"`)
  - `arch/arm/boot/dts/Makefile`: add `ipq5000-rax3000q.dtb`
  - `arch/arm64/boot/dts/qcom/Makefile`: add `ipq5000-cmcc-rax3000q.dtb`
  - `target/linux/feeds/ipq50xx/image/generic.mk`:
    add `Device/UbiFit` macro + `Device/cmcc_rax3000q` (+ `TARGET_DEVICES += cmcc_rax3000q`)
- commit `c408992` on `QuickXand/Actions-OpenWrt-RAX3000Q@master`:
  - workflow `REPO_URL/REPO_BRANCH` -> `QuickXand/qsdk@rax3000q-port`
  - ccache `mixkey: qsdk12.5-rax3000q`
  - removed Dropbox kernel/backports tarball fetch step (12.5 kernel is in-tree)
  - removed `luci-app-turboacc` Dropbox zip step
  - kept golang / node / natmap / ddns-go / frp feed swaps
  - rewrote `.config` to minimal seed (~30 lines) targeting ipq50xx/ipq50xx_32/cmcc_rax3000q
  - added `AGENTS.md` documenting the mid-port state

## Run 29144005179 (workflow_dispatch, 2026-07-11 07:09Z-07:59Z)

- Status: **completed/failure** — runner lost communication
- Step trace (from `actions/jobs/86522295169`):
  - `Set up job` ✓, `Checkout` ✓
  - **`Initialization environment` in_progress** for **50 minutes**, then runner died
  - All later steps `pending` / never started
- Log artifact: **NOT available** — `actions/runs/.../logs` returns 24 bytes,
  `actions/jobs/.../logs` returns HTTP 404 (Azure BlobNotFound).
  Runner didn't manage to flush any step log before OOM/network drop.
- GitHub annotation: *"The hosted runner lost communication with the server.
  Anything in your workflow that terminates the runner process, starves it for
  CPU/Memory, or blocks its network access can cause this error."*

### Diagnosis

This is **not a build error** — we never reached `Clone source code`,
`make defconfig`, or `make V=s`. The failure is the `Initialization environment`
step itself (lines 45-83 of the workflow):

```
sudo apt-get remove -y '^dotnet-.*'
sudo apt-get remove -y '^llvm-.*'
sudo apt-get remove -y 'php.*'
sudo apt-get remove -y '^mongodb-.*'
sudo apt-get remove -y '^mysql-.*'
sudo apt-get remove -y azure-cli google-cloud-sdk google-chrome-stable firefox powershell mono-devel libgl1-mesa-dri
sudo apt-get autoremove -y
sudo apt-get clean
...
sudo rm -rf /usr/share/dotnet/ /usr/local/graalvm/ /usr/local/.ghcup/ \
           /usr/local/share/powershell /usr/local/share/chromium \
           /usr/local/lib/android /usr/local/lib/node_modules
sudo -E apt-get -qq update
sudo apt install -y ack antlr3 asciidoc ... (long list) ...
```

This boilerplate is P3TERX/Actions-OpenWrt era. On modern `ubuntu-22.04`
runners, `^llvm-.*` and `php.*` removal can take a long time due to dependency
reconciliation, and apt may stall against `unattended-upgrades` lock.
50-minute hang then OOM/kill is consistent with apt lock contention or
unattended-upgrades mid-flight.

### Next action

Slim `Initialization environment` to remove-only essential packs + rely on
the `ubuntu-22.04` preinstalled build-essential toolchain. Concretely:
1. Keep the `rm -rf /usr/local/{lib/android,lib/node_modules,share/*}`
   disk-release lines (those are cheap, ~30s, and reclaim many GB).
2. Drop the `^llvm-.*`, `php.*`, `^mongodb-.*`, `^mysql-.*`,
   `azure-cli google-cloud-sdk google-chrome-stable firefox powershell mono-devel`
   `apt-get remove` lines — ubuntu-22.04 image no longer has most of these,
   so the removes are no-ops that still cost dependency-graph time.
3. Keep `dotnet` removal (still installed, large) OR just `rm -rf /usr/share/dotnet`.
4. Run `sudo -E apt-get -qq update` only after disabling unattended-upgrades
   (`sudo systemctl disable --now unattended-upgrades` or wait loop).
5. `apt install` is still required for: device-tree-compiler, the Python deps,
   and QSDK-specific (`swig`, `python3-ply`, `python3-pyelftools` etc.).
   `build-essential` is already preinstalled.

This change is the highest priority — without a working Initialization
step we cannot reach `make defconfig` and therefore cannot harvest the
first useful `V=s` build error log that the whole port depends on.

## Known port gaps (answer the V=s log first before touching these)

- `ipq5000-cmcc-rax3000q.dts` includes `ipq5018.dtsi`, but in 12.5 that file
  is only the PMU node; real SoC nodes are in
  `arch/arm64/boot/dts/qcom/ipq5018.dtsi`. 11.5-era labels `&q6v5_wcss`,
  `&wifi0`, `&wifi1`, `&tlmm` may not exist with those exact names in 12.5.
  Expected first compile error: DTS label undefined in the rax3000q.dtb build.
- `ipq-wifi-cmcc_rax3000q` package + binary BDF not ported (lives in the 11.5
  fork `package/firmware/ipq-wifi`). `DEVICE_PACKAGES` in the image macro
  intentionally omits it to keep the first build moving.
- `target/linux/feeds/ipq50xx/base-files/{etc,lib}` for RAX3000Q (LED/network/
  upgrade scripts) not ported; 12.5 stock Qualcomm AP defaults remain.

## Useful remotes / SHAs

- `QuickXand/qsdk@rax3000q-port` HEAD: `9afbdd06`
- `QuickXand/Actions-OpenWrt-RAX3000Q@master` HEAD: `c408992`
- Last failed run: 29144005179 (logs unavailable — runner died)
- Old 11.5 .config backup (out of tree): `C:\Users\ROG\AppData\Local\Temp\opencode\qsdk-port\.config.qsdk11.5.bak`
