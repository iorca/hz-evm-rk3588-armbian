# HZ-EVM-RK3588 Armbian board port

Board support files to build **Armbian** for the **HZ-EVM-RK3588** (Rockchip RK3588 /
RK3588J industrial, vendor: 合众恒跃 / hezuo) using the upstream Armbian build framework.

This repo contains **only the board overlay** — `config/` and `patch/` — plus the
GitHub Actions workflow that clones `armbian/build`, injects these files, and builds
the image. It does **not** fork or vendor the whole Armbian tree.

## Layout

```
config/boards/hz-evm-rk3588.conf                              # board definition
config/bootenv/rk35xx.txt                                     # family bootenv override

patch/kernel/rk35xx-current/
  0000.patching_config.yaml                                   # REQUIRED: tells the framework
                                                              #   to copy dt/*.dts into
                                                              #   arch/arm64/boot/dts/rockchip/
                                                              #   and auto-patch the Makefile
  dt/rk3588-hz-evm-rk3588.dts                                 # full Linux device tree

patch/u-boot/v2026.07/
  0000.patching_config.yaml                                   # REQUIRED: maps defconfig/,
                                                              #   dt_uboot/ and
                                                              #   dt_upstream_rockchip/
                                                              #   onto the u-boot source tree
  0001-add-hz-evm-rk3588-board-target.patch                   # adds TARGET_HZ_EVM_RK3588:
                                                              #   mach-rockchip/rk3588/Kconfig,
                                                              #   board/hezuo/hz-evm-rk3588/*,
                                                              #   include/configs/hz-evm-rk3588.h
  defconfig/hz-evm-rk3588_defconfig                           # -> u-boot configs/
  dt_uboot/rk3588-hz-evm-rk3588-u-boot.dtsi                   # -> u-boot arch/arm/dts/
  dt_upstream_rockchip/rk3588-hz-evm-rk3588.dts               # -> u-boot dts/upstream/src/arm64/rockchip/

.github/workflows/build.yml                                   # CI build -> GitHub Release
```

### Why the `0000.patching_config.yaml` files are mandatory

Armbian's current patch engine reads a per-directory manifest. Without it, files
dropped into `defconfig/`, `dt/`, `dt_uboot/` or `dt_upstream_rockchip/` are
**silently ignored** — the build then fails much later (or ships an image without
your board DTS). The kernel-side manifest uses `dts-directories` +
`auto-patch-dt-makefile`; the u-boot-side one uses `overlay-directories`.

### Why U-Boot needs a patch (not just a defconfig)

Mainline U-Boot has no `hz-evm-rk3588` target, so `CONFIG_TARGET_HZ_EVM_RK3588`
would be undefined and the build would abort. `0001-…-board-target.patch` adds the
Kconfig symbol, the board directory and `include/configs/hz-evm-rk3588.h`
(following the `rock5b-rk3588` layout). If you bump `BOOTBRANCH` to a newer U-Boot
tag, this patch may need refreshing.

## Build via GitHub Actions

Push to `main`, or run **Actions → Build Armbian → Run workflow**. The image is
published as a **GitHub Release** (tag `armbian-<run>.<attempt>`) with the
`.img.xz` (xz-compressed) and `.img.txt` as release assets.

`compile.sh` runs under `sudo`: it registers qemu binfmt handlers, sets up loop
devices, mounts and chroots into the target rootfs. Running it as the unprivileged
`runner` user aborts early.

The raw `.img` is ~3 GB and exceeds GitHub's 2 GB per-file limit, so the workflow
always xz-compresses it. `BUILD_MINIMAL` defaults to `no` (full image). Releases are
permanent, unlike artifacts which expire.

## Build locally

```bash
git clone --depth 1 --branch=main https://github.com/armbian/build.git armbian-build
cp -r config patch armbian-build/
cd armbian-build
sudo ./compile.sh BOARD=hz-evm-rk3588 BRANCH=current RELEASE=noble \
                 BUILD_DESKTOP=no BUILD_MINIMAL=no KERNEL_CONFIGURE=no EXPERT=yes
```

## Notes / status

- SoC is treated as a **full RK3588** (conservative). `rk3588.dtsi` is included, **not**
  `rk3588j.dtsi` (the `-j` variant would only downclock the OPPs).
- Boot chain: `BOOT_SCENARIO="binman"` — DDR/BL31 come from `rkbin` blobs, U-Boot
  produces `u-boot-rockchip.bin` via binman, deployed with `dd … bs=32k seek=1`.
- `cpufreq` "unlisted initial frequency" warning fixed by adding `opp-1008000000`
  to `cluster1_opp_table` / `cluster2_opp_table` (big cores handed to the kernel at
  1008 MHz by mainline U-Boot).
- `avdd_0v75_s0` regulator "Bringing 750000uV into 837500uV" line is benign and left
  as-is (PMIC boot value, not dts-fixable).
- Peripherals disabled on this conservative bring-up: `gmac1` (2nd GbE, pending PHY
  addr from schematics), `sound` + `es8388` codec, `pcf8563` RTC, and `pcie3x4`
  (boot-hang workaround — PERST# polarity / slot wiring still to be confirmed from
  the baseboard schematic).
- The **U-Boot DTS is deliberately minimal** and separate from the kernel DTS.
  Mainline U-Boot ships a trimmed rk3588 dtsi that lacks most kernel-only nodes
  (vop/gpu/tsadc/hdmi_receiver/…), so anything referenced there must exist in the
  U-Boot tree. GMAC is left out of U-Boot for now.
- Mainline `current` (6.18) has **no** NPU / MPP / ISP / RGA; switch `BRANCH=vendor`
  if those are required.
