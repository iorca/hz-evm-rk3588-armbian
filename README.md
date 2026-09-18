# HZ-EVM-RK3588 Armbian board port

Board support files to build **Armbian** for the **HZ-EVM-RK3588** (Rockchip RK3588,
vendor: 合众恒跃 / hezuo) using the upstream Armbian build framework.

This repo contains **only the board overlay** — `config/` and `patch/` — plus the
GitHub Actions workflow that clones `armbian/build`, injects these files, and builds
the image. It does **not** fork or vendor the whole Armbian tree.

## Layout

```
config/boards/hz-evm-rk3588.conf                        # board definition
config/bootenv/rk35xx.txt                              # family bootenv override
patch/kernel/rk35xx-current/dt/rk3588-hz-evm-rk3588.dts # kernel device tree
patch/u-boot/v2026.07/dt_upstream_rockchip/...         # mainline U-Boot dts
patch/u-boot/v2026.07/defconfig/hz-evm-rk3588_defconfig
patch/u-boot/v2026.07/dt_uboot/rk3588-hz-evm-rk3588-u-boot.dtsi
.github/workflows/build.yml                            # CI build
```

## Build locally (needs Docker)

```bash
git clone --depth 1 --branch=main https://github.com/armbian/build.git armbian-build
cp -r config patch armbian-build/
cd armbian-build
./compile.sh BOARD=hz-evm-rk3588 BRANCH=current RELEASE=noble \
             BUILD_DESKTOP=no BUILD_MINIMAL=yes KERNEL_CONFIGURE=no EXPERT=yes
```

## Build via GitHub Actions

Push to `main` (or run **Actions → Build Armbian → Run workflow**). The image is
uploaded as a `.img.xz` artifact (the raw `.img` exceeds GitHub's 2 GB artifact
limit, so `BUILD_MINIMAL` defaults to `yes` and the output is xz-compressed).

## Notes / status

- SoC is a **full RK3588** (used conservatively). `rk3588.dtsi` is included, **not**
  `rk3588j.dtsi` (the `-j` variant would only downclock the OPPs).
- `cpufreq` "unlisted initial frequency" warning fixed by adding `opp-1008000000`
  to `cluster1_opp_table` / `cluster2_opp_table` (big cores handed to the kernel at
  1008 MHz by mainline U-Boot).
- `avdd_0v75_s0` regulator "Bringing 750000uV into 837500uV" line is benign and left
  as-is (PMIC boot value, not dts-fixable).
- Peripherals disabled on this conservative bring-up: `gmac1` (2nd GbE, pending PHY
  addr from schematics), `sound` + `es8388` codec, `pcf8563` RTC, and `pcie3x4`
  (boot-hang workaround — PERST# polarity / slot wiring still to be confirmed from
  the baseboard schematic).
- Mainline `current` (6.18) has **no** NPU / MPP / ISP / RGA; switch `BRANCH=vendor`
  if those are required.
