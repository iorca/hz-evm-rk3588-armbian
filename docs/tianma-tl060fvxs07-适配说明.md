# Tianma TL060FVXS07 MIPI 屏适配 —— 核验 & 落地说明

目标：HZ-EVM-RK3588（Armbian **主线** 6.18 / `BRANCH=current`）点亮天马 TL060FVXS07
（6.0"，1080x2160，4-lane MIPI DSI，RGB888，Samsung S6D6FT0 IC）。

最后更新：2026-09-27（并入板级 dts；触摸搁置）

---

## 0. 结论

| 项 | 结论 |
|---|---|
| 4 个引脚冲突 | **无**。背光 `GPIO1_A2`(`pwm0m2_pins`)、`GPIO0_B2`(LCD_RESX)、`GPIO1_B4`(TP_INT)、`GPIO1_A1/A0`(I2C2 = `i2c2m4_xfer`) 全是空的 |
| 参考案例 dts 能否直接抄 | **不能**。全是 Rockchip BSP（5.10/6.1）写法，主线不认 |
| 显示 | 已并入 `patch/kernel/rk35xx-current/dt/rk3588-hz-evm-rk3588.dts` |
| 触摸 | **搁置**。`sec,sec_ts` 主线不存在，节点保留为 `disabled` 只作接线记录 |
| LCD_RESX 语义 | 默认**高有效使能**；无需重编即可切换复位语义（见第 4 节） |

---

## 1. 引脚占用清单

核对方法：先在主线 v6.18 `rk3588-base-pinctrl.dtsi` / `rk3588-extra-pinctrl.dtsi`
里确认 GPIO 名对应的 mux 组，再逐条比对板级 dts 里**实际被引用**的组。
SoC dtsi 中所有引脚组都带 `/omit-if-no-ref/`，没被引用的组不会被应用，所以"被引用"才算占用。

### 1.1 新用到的 4 组：全部空闲

| 用途 | 引脚 | 主线 pinmux 组 | 现状 |
|---|---|---|---|
| 背光 PWM | GPIO1_A2 | `pwm0m2_pins`（func 11） | `&pwm0` 早就 `status="okay"`，只是没有 consumer |
| LCD_RESX | GPIO0_B2 | GPIO | 空闲 |
| TP_INT | GPIO1_B4 | GPIO | 空闲（触摸搁置，暂未 mux） |
| 触摸 I2C | GPIO1_A1(SCL)/A0(SDA) | `i2c2m4_xfer` | `&i2c2` 原本没启用（现 disabled 保留接线） |

`i2c2` 在 RK3588 上有 5 路复用（m0..m4），**这款是 m4**：

```
rk3588-base-pinctrl.dtsi:889
i2c2m4_xfer: i2c2m4-xfer {
	rockchip,pins =
		/* i2c2_scl_m4 */ <1 RK_PA1 9 &pcfg_pull_none_smt>,
		/* i2c2_sda_m4 */ <1 RK_PA0 9 &pcfg_pull_none_smt>;
};
```

**IO 电平域：已确认与触摸芯片上拉一致**（GPIO1_A0/A1）。

### 1.2 GPIO0 / GPIO1 既有占用（避让参考）

**GPIO0**：A5/A6/B3 + B1 = `spi2m2_*`（RK806 PMIC）；B5/B6 = `uart2m0`（**调试串口**）；
C4/C5 = `uart0m0`；C7/D0 = `i2c6m0`；D1/D2 = `i2c0m2`；D3 = `hp_det`；D4/D5 = `i2c1m2`；A7 = PMIC 中断。

**GPIO1**：B2 = `vcc3v3_mini_pcie` 使能；D0/D1 = `i2c7m0`；D5 = `hdmirx_det`；A2 = 本方案背光。

> 注：`&hdmi_receiver` 的 `hdmim1_rx_*` 组落在 **GPIO3**（PD1/PD2/PD3/PD4），不在 GPIO1。

---

## 2. 为什么参考案例不能抄

`mipi-hub-official-master/TL060FVXS07/` 下所有 RK3588 案例（NanoPC-T6、视美泰 AIoT-3588A、
Radxa ROCK 5C、KickPi K1）都依赖这几样**主线没有**的东西：

| 参考案例 | 主线 6.18 实情 |
|---|---|
| `compatible = "simple-panel-dsi"` | **没有**。binding `panel-simple-dsi.yaml` 的 compatible 是具体面板枚举，没有 catch-all；`drivers/gpu/drm/panel/` 下 120 个驱动也没人注册这个串 |
| `panel-init-sequence = [...]` | **没人解析**。v6.18 的 `struct panel_desc` 已无 `init_seq` 字段 |
| `&route_dsi0 { connect = <&vp2_out_dsi0>; }` / `&dsi0_in_vp2` / `&dsi0_pwm` | **没有**。BSP 专属节点。主线走 **VOP2 endpoint 图** |
| `&pwm_backlight`（BSP 预定义） | 主线要自己写 `pwm-backlight` 节点 |

**主线这套 DSI 通路本身是完整的**（已核对）：

- `&dsi0` = `rockchip,rk3588-mipi-dsi2`（`rk3588-base.dtsi:1520`），`&mipidcphy0` = `rockchip,rk3588-mipi-dcphy`（`:3157`）
- 桥接层 `drivers/gpu/drm/bridge/synopsys/dw-mipi-dsi2.c`：`.transfer = dw_mipi_dsi2_host_transfer`（650-692 行，支持长写+读）；978 行 `mipi_dsi_host_register`
- 面板发现：531 行 `devm_drm_of_get_bridge(dev, dev->of_node, 1, 0)`
  → **panel endpoint 必须挂在 DSI host 的 port@1，也就是 `&dsi0_out`**

另外：**主线没有给 dts 用的 `dt-bindings/display/mipi-dsi.h`**，所以
`dsi,lanes` / `dsi,format` / `dsi,flags` 那套宏在 DT 侧根本用不了 ——
本方案把这三项写死在驱动的 probe 里。

---

## 3. 做了什么

### 3.1 新增 panel 驱动

`patch/kernel/rk35xx-current/0002-drm-panel-add-tianma-tl060fvxs07-DSI-panel-driver.patch`

- 新文件 `drivers/gpu/drm/panel/panel-tianma-tl060fvxs07.c`
- Kconfig + Makefile 各一行
- compatible `tianma,tl060fvxs07`
- 结构照抄主线 `panel-elida-kd35t133.c`（v6.18 原文件），所有 API 同版本都在
- init 序列按参考案例 1:1 翻译（0x9F/0xF0 解锁 + 两段 60 字节 gamma 表）
- 顶部 `TL060FVXS07_SHORT_INIT_SEQ` 改成 1 → 参考案例也验证过能跑的**简略版**
  （只有 exit_sleep + display_on），排障第一步就试它
- 供电 optional（`devm_regulator_get_optional`），没原理图也能先跑
- 时序：`.clock=154000` kHz、hback4/hsync4/hfront92、vback6/vsync2/vfront8
  → 1180 × 2176 @ 154 MHz = **59.98 Hz**（与 Nanopi T6 那份一致）

源文件副本在 `drivers/panel-tianma-tl060fvxs07.c`，改完记得同步进 patch。

### 3.2 内核配置补三个开关

`config/boards/hz-evm-rk3588.conf` 新增 `custom_kernel_config__enable_mipi_dsi_display()`：

| 配置 | 现状 | 需要 |
|---|---|---|
| `CONFIG_ROCKCHIP_DW_MIPI_DSI2` | 未设置（老的 `ROCKCHIP_DW_MIPI_DSI` 是 rk3288/rk3399 用的，别混） | `=y` |
| `CONFIG_PHY_ROCKCHIP_SAMSUNG_DCPHY` | 未设置 | `=y` |
| `CONFIG_DRM_PANEL_TIANMA_TL060FVXS07` | 新增驱动 | `=y` |

面板驱动**编进内核而不是模块**：DSI 子设备的 of: modalias 自动加载不好赌。

> 若你的 `armbian/build` 版本 helper 名字变了，先
> `grep -rn "function kernel_config_set" armbian-build/lib/`。

### 3.3 板级 dts

`patch/kernel/rk35xx-current/dt/rk3588-hz-evm-rk3588.dts` 新增：

- 根节点 `tianma_bl: backlight-tianma`（`pwm-backlight`，`pwms = <&pwm0 0 25000 0>`）
- `&mipidcphy0` / `&dsi0` + `panel@0`
- `&dsi0_out` ↔ panel endpoint；`&dsi0_in` ↔ `&vp2` endpoint
- `&pinctrl` 里 `tianma-lcd`（GPIO0_B2）
- `&i2c2` + `touchscreen@48`：**disabled**，只作接线记录

### 3.4 保留的 overlay 支持

`patch/kernel/rk35xx-current/0001-arm64-dts-rockchip-enable-DT-symbols-*.patch`
给 `arch/arm64/boot/dts/rockchip/Makefile` 加：

```makefile
DTC_FLAGS_rk3588-hz-evm-rk3588 := -@
```

主线 `scripts/Makefile.dtbs:114-115` 只对组合型 `-dtbs` 目标加 `-@`，
普通板级 dtb 没有 `/__symbols__`，U-Boot 的 `fdt apply` 会失败。
**本方案已不走 overlay，这个 patch 是给以后留的**（`boot-rk35xx.cmd` 的
`overlays=` / `user_overlays=` 两条路都能用）。

---

## 4. LCD_RESX：高有效，且切换语义不用重编

按你的要求设为**高有效**，并且因为不确定它是"使能"还是"复位"，驱动里两种语义都支持：

```dts
enable-gpios = <&gpio0 RK_PB2 GPIO_ACTIVE_HIGH>;
```

- 默认（`enable-gpios`）：驱动把它**拉高并保持**（使能语义）
- 切复位语义，**不用重新编译**，在 `armbianEnv.txt` 的 `extraargs` 里加：

```
panel-tianma-tl060fvxs07.gpio_is_reset=1
```

（编进内核的模块参数走 cmdline，就是这个写法。加完 reboot。）

- 或者改 dts：把 `enable-gpios` 换成 `reset-gpios`（极性不变），驱动自动走
  assert → hold → release 的脉冲。

pinctrl 用的是 `&pcfg_pull_down`（上电到驱动 probe 之间保持确定的关态）。
若底板外面已有上拉、或需要开机即亮，改成 `&pcfg_pull_up`。

---

## 5. 触摸：搁置

`sec,sec_ts`（Samsung "6FT0"）**主线不存在**，只存在于带外驱动包
（`TL060FVXS07/泰山派/my_sec_ts/`），且针对 5.10/6.1 写的，直接打到 6.18 编译不过。

处理方式：`&i2c2` 和 `touchscreen@48` 都保留在 dts 里但 `status = "disabled"`，
只当接线记录，不会去抢 GPIO1_A0/A1/B4。

要接手时：移植 sec_ts，建议砍掉 fw 升级/自检那套，只留 probe + input 上报；
然后把两处 `status` 改成 `okay`。

---

## 6. 文件清单

```
config/boards/hz-evm-rk3588.conf                                     # 改：kernel config hook
patch/kernel/rk35xx-current/
  0001-arm64-dts-rockchip-enable-DT-symbols-for-hz-evm-rk3588.patch   # 新：给以后用 overlay 留门
  0002-drm-panel-add-tianma-tl060fvxs07-DSI-panel-driver.patch        # 新：panel 驱动
  dt/rk3588-hz-evm-rk3588.dts                                         # 改：并入 DSI0 + 背光 + 触摸(禁用)
drivers/panel-tianma-tl060fvxs07.c                                    # 新：驱动源文件副本
docs/tianma-tl060fvxs07-适配说明.md                                    # 本文
.github/workflows/build-mipi.yml                                      # 新：本分支专用 CI
```

`0002-*.patch` 已用 `git apply --check` + `git apply` 对着 v6.18 真实的
`drivers/gpu/drm/panel/{Kconfig,Makefile}` 验证通过；`0001-*.patch` 对着
`arch/arm64/boot/dts/rockchip/Makefile` 验证通过。

---

## 7. 上机验收

```bash
zcat /proc/config.gz | grep -E 'DW_MIPI_DSI2|SAMSUNG_DCPHY|PANEL_TIANMA'
dmesg | grep -iE 'dsi|dcphy|tianma|backlight|drm'
cat /sys/class/backlight/*/brightness          # 能改 = 背光 PWM 通了
for d in /sys/class/drm/card0-*; do echo "$d $(cat $d/status) $(cat $d/enabled) $(head -1 $d/modes)"; done
```

排障顺序：

### 0. "背光亮但无画面" —— 先看 console 落在哪个口

**不要**把"背光亮"当成 panel 起来了：pwm-backlight 在 probe 时就把
`default-brightness-level` 写出去了，panel 有没有 enable 跟背光无关。

```
$ for d in /sys/class/drm/card0-*; do echo "$d $(cat $d/status) $(cat $d/enabled) $(head -1 $d/modes)"; done
/sys/class/drm/card0-DSI-1      connected enabled 1080x2160   ← 屏本身是好的
/sys/class/drm/card0-HDMI-A-1   connected enabled 1024x768
/sys/class/drm/card0-HDMI-A-2   connected enabled 1024x768
```

只要 `card0-DSI-1` 是 `connected enabled` 且 mode 是 1080x2160，**硬件和驱动就已经通了**，
剩下的只是文本控制台落到了先枚举到的 HDMI-A-1 上（`Console: switching to colour
frame buffer device 128x48` —— 128x48×8x16 = 1024x768，不是我们的屏）。

修法（**不用重编**），在 `/boot/armbianEnv.txt` 的 `extraargs` 里禁用 HDMI 输出：

```
extraargs=... video=HDMI-A-1:d video=HDMI-A-2:d
```

`video=<connector>:d` 是 `drm_fb_helper` 的标准语法（`d` = disable）。
fb helper 只剩 DSI-1 可用，console 自然就过去了。想恢复 HDMI 就把这两个参数摘掉重启。

要更彻底就在 dts 里把 `&hdmi0` / `&hdmi1` 改成 `status = "disabled"`。

### 1. 真正的故障排查

1. panel 压根没 probe（dmesg 无 `tianma`）→ 检查三个 CONFIG
2. probe 成功但黑屏 → ①加 `panel-tianma-tl060fvxs07.gpio_is_reset=1`（第 4 节）
   ②把 `TL060FVXS07_SHORT_INIT_SEQ` 改成 1 重编驱动
3. 还黑 → 换参考案例里 radxa 的备选时序
   （`clock-frequency = <157000000>` + hfront115/hback13/hsync3/vfront10/vback10）；
   或用示波器量 DPHY 的 lane0 有没有 HS 差分信号
4. 背光不亮 → 确认 `&pwm0` 有没有跑起来、量 GPIO1_A2；必要时把 pinctrl 改成 `pull_up`
5. 画面方向反 → 打开 dts 里的 `rotation = <90>`

---

## 8. 开机日志里几条无害的噪音（别去查）

### `ERROR:   Error initializing runtime service opteed_fast`

**TF-A（BL31）打的，不是内核也不是 U-Boot。** 同一段前面那行 WARNING 就是答案：

```
WARNING: No OPTEE provided by BL2 boot loader, Booting device without OPTEE initialization. SMC`s destined for OPTEE will return SMC_UNK
ERROR:   Error initializing runtime service opteed_fast
INFO:    BL31: Preparing for EL3 exit to normal world     ← 照样继续往下跑
```

- 出处：`common/runtime_svc.c:415-421` —— `rc = service->init(); if (rc != 0) { ERROR(...); continue; }`。
  **是 `continue`，不是 panic**，所以启动照常。
- 成因：rkbin 的 BL31 二进制（`BL31: v2.3 ... fwver: v1.48`）是按 `SPD=opteed` 编的，
  但 Armbian 的 `BOOT_SCENARIO=binman` 只塞了 BL31 + DDR(TPL)，**没有 BL32 / OP-TEE 镜像**。
  于是 OP-TEE 初始化失败，服务名 `opteed_fast`（OP-TEE 的 fast SMC 那一路）报这个错。
- 后果：**本机没有 OP-TEE**。没有 `/dev/tee0`、没有 tee-supplicant，
  OP-TEE 提供的高级功能（安全存储、密钥、Widevine DRM、fTPM）都用不了。
  对 CLI/服务器用途**零影响**。
- 要不要修：不用。真要 OP-TEE 得单独编 BL32 并让 binman 把它塞进去，收益对这台机器没有。

### `Loading Boot0000 'mmc 0' failed` / `Boot failed (err=-14)`

U-Boot 标准启动流程：先试 EFI boot manager，失败后回落 `distro_bootcmd`
扫 `mmc@fe2c0000.bootdev.part_1` 的 `/boot.scr`。日志里紧跟着就是
`** Booting bootflow ... with script` + `Boot script loaded from mmc 0:1`，**正常**。
