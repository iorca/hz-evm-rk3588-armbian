# HZ-EVM-RK3588 + 天马 TL060FVXS07 MIPI 屏：主线排查结论与 BSP 接管交接

日期：2026-09-27
结论一句话：**主线（mainline）内核路径走不通，转用厂家 BSP 内核。**

---

## 一、屏与板的基本参数（已确认）

| 项 | 值 |
|---|---|
| 面板 | 天马 TL060FVXS07，1080×2160，MIPI DSI **4 lane**，RGB888 |
| 驱动 IC | Samsung S6D6FT0（社区推测） |
| SoC 侧 | RK3588 **DSI0**（`fde20000`），PHY = `mipidcphy0`（`feda0000`） |
| MIPI 座子引脚组 | `MIPI_DPHY0_TX`（已确认对应 DSI0） |
| LCD_RESX | **GPIO0_B2**（`RK_PB2` = line 10），低有效复位 |
| 背光 | PWM0（GPIO1_A2，`pwm0m2_pins`） |
| 触摸 | `sec,sec_ts`，I2C2（GPIO1_A0/A1），IRQ GPIO1_B4；**主线无驱动，搁置** |

---

## 二、主线路径的结论：走不通

### 2.1 铁证（逻辑分析仪实测，接屏状态）

DSLogic Plus，200 MHz / 阈值 0.6 V / Buffer 模式，接线 `L-0/L-1 = data0，L-2/L-3 = clk，L-4 = RESET`。

按块（每块 83.9 ms）统计跳变数：

```
块7  0.587s   data0=252    clk=0      ← 初始化 burst
块8  0.671s   data0=2778   clk=0      ← 序列继续发，时钟依然死
块9  0.755s   data0=16064  clk=15898  ← 到视频阶段时钟才起来
块10-18       data0≈42228  clk≈42228
```

**⇒ 命令模式下时钟通道完全不输出，时钟只在视频模式才起来。**

### 2.2 data0 上的 LPDT 波形是完美的

```
LP-00 → LP-10(bit=1) / LP-01(bit=0) → LP-00 → ...
位周期 = 12+9 = 21 采样 @200MHz ≈ 105 ns
```

DSView 解码逐字节对照参考，内容正确：

| 线上解出的 | 我们发的 | 参考原文（NanoPC-T6 dtsi） |
|---|---|---|
| `0x15 0x55 0x10` + ECC `0x33` | `15 00 02 55 10` | 第 94 行 `15 00 02 55 10` ✓ |
| `0x39` WC=`0x03 0x00` ECC=`0x09`，载荷 `F0 5A 5A`，CRC `5F 66` | `39 00 03 F0 5A 5A` | 第 95 行 `39 00 03 F0 5A 5A` ✓ |

（注意：解码器曾把 CRC 首字节 `5F` 划进载荷，看起来像 `F0 5A 5F`，是字段边界差一格，不是数据错。）

### 2.3 两种发送时机都失败

- **命令模式发**（`prepare`）：时钟通道零输出 → 屏解不出 → 读 `-110`
- **视频模式发**（`seq_at_enable=1`，此时时钟已活）：同样 `-110`

**⇒ 主线 dw-mipi-dsi2 在 RK3588 上根本无法跟 DSI 面板通信。**

### 2.4 源码佐证

`drivers/gpu/drm/bridge/synopsys/dw-mipi-dsi2.c`：

```c
/* dw_mipi_dsi2_mode_set()，由 atomic_pre_enable 调用 */
regmap_update_bits(dsi2->regmap, DSI2_PHY_CLK_CFG, CLK_TYPE_MASK,
    dsi2->mode_flags & MIPI_DSI_CLOCK_NON_CONTINUOUS ? NON_CONTINUOUS_CLK
                                                     : CONTINUOUS_CLK);
```

另外两个 ULPS 宏**定义了但从没用过**（说明相关功能在主线里缺失）：

- `phy-rockchip-samsung-dcphy.c:182` `T_ULPS_EXIT`
- `dw-mipi-dsi2.c:63-64` `TO_LPTXULPS`

Armbian 的 `patch/kernel/archive/rockchip64-6.18/` 里**也没有**任何 dsi2/dcphy 补丁。

---

## 三、硬件：全部排除

| 检查项 | 结果 |
|---|---|
| FPC 座屏电源 3.3 V | ✅ 正常 |
| FPC 座屏电源 1.8 V | ✅ 正常 |
| 时钟差分对（座子 ↔ SoC）通断 | ✅ 通 |
| 座子上 RESET 脉冲 | ✅ 实测 **124 ms**（驱动设计 120 ms），与 `msleep(120)` 一致 |
| **屏在别的板（BSP 环境）亮过** | ✅ **是** —— 这条最关键，证明硬件无问题 |

---

## 四、主线侧已试且无效（别再重复）

| 试过的 | 结果 |
|---|---|
| 完整序列 / 简略序列（仅 0x11+0x29） | 都无效 |
| 时序 154 MHz / 157 MHz / 172 MHz 三份 | 全部黑屏 |
| 命令在 `prepare` / 在 `enable`（视频模式后） | 都是 `-110` |
| `MIPI_DSI_CLOCK_NON_CONTINUOUS`（`noncont_clock=1`） | 时钟仍无波形 |
| 复位脉冲 | 实测正确 |
| LPM 标志链路（源码核对） | 正常 |
| DSI0 引脚组 / PHY 时钟通道使能 | 正常 |

---

## 五、可直接复用到 BSP 的资产

### 5.1 初始化序列（已线上验证正确，可放心照搬）

来源：`C:\Users\orca\Desktop\RK3588\mipi-hub-official-master\TL060FVXS07\友善\NanoPC-T6\设备树\rk3588-nanopi6-mipi-lcd-TL060FVXS07.dtsi` 第 92–101 行：

```
39 00 03 9F A5 A5        // 解锁
05 78 01 11              // exit sleep，延时 120ms
15 00 02 55 10
39 00 03 F0 5A 5A        // 进 vendor page
15 00 02 73 94
39 00 3D EA 00 73 11 1B 23 2A 40 59 70 6E A1 86 92 9F AB B8 4D 59 65 7F (×3 组，共 60 字节)
39 00 3D EB 00 73 ... (同上 60 字节)
39 00 03 F0 A5 A5        // 出 vendor page
05 09 01 29              // display on，延时 9ms
39 00 03 9F 5A 5A        // 重新 lock
```

三份参考（NanoPC-T6 / Radxa ROCK 5C / 视美泰 AIoT-3588A）**内容一致**。
Radxa 那份（`瑞莎/rock5c/.../rock-5a-tianma-display-6fhd.dtsi`）另有简略版：只 `05 78 01 11` + `05 0A 01 29`。

### 5.2 显示时序（三选一，BSP 下用哪份待定）

| 来源 | pixel clock | htotal | vtotal |
|---|---|---|---|
| NanoPC-T6 | 154 MHz | 1180 | 2176 |
| Radxa ROCK 5C | 157 MHz | 1211 | 2182 |
| qcom 原始 | 172 MHz | 1317 | 2176 |

### 5.3 复位设计

低有效，assert 120 ms 后释放，释放后再等 120 ms 才发第一条命令。pinctrl 用下拉
（上电到驱动 probe 之间把屏按在复位里）。

---

## 六、BSP 路径的计划

1. **目标**：先编一个 **BSP 点屏固件**（单一验证：BSP 下这块板 + 这块屏能不能亮）
2. 成功后，再决定要不要集成进 Armbian 的 `vendor` 分支
3. 主线那份 dts **不能直接用** —— BSP 的节点/绑定不同，需按 BSP 格式重写：
   `&dsi0` + `panel@0 { compatible = "simple-panel-dsi"; panel-init-sequence = [...] }`
   + `dsi,format` / `dsi,lanes` + `reset-gpios` + 背光

### 需要从厂家 SDK 确认的东西

- [ ] 内核版本与分支（例如 5.10 / 6.1 rkr）
- [ ] 板级 dts 文件名与路径（本板在 SDK 里叫什么）
- [ ] DSI 相关 config 项名（`CONFIG_ROCKCHIP_DW_MIPI_DSI` 之类，BSP 与主线的名字不同）
- [ ] SDK 里是否已有 `simple-panel-dsi` / `panel-init-sequence` 支持
- [ ] 是否有本板的 MIPI 屏参考 dts（优先复用）
- [ ] 编译方式（`build.sh` 还是 `make.sh`）与产物

---

## 七、相关文件

| 文件 | 说明 |
|---|---|
| `drivers/panel-tianma-tl060fvxs07.c` | 主线版面板驱动（含 `gpio_is_reset` / `short_init_seq` / `read_id` / `seq_at_enable` / `timing` / `noncont_clock` 六个运行时开关） |
| `patch/kernel/rk35xx-current/0002-drm-panel-add-tianma-*.patch` | 驱动的内核补丁 |
| `patch/kernel/rk35xx-current/dt/rk3588-hz-evm-rk3588.dts` | 主线版板级 dts（含 DSI0 / mipidcphy0 / PWM0 背光 / 触摸节点 disabled） |
| `dt-overlays/tianma-resetfix.dts` | 复位线修正 overlay（把 `enable-gpios` ACTIVE_HIGH 换成 `reset-gpios` ACTIVE_LOW） |
| `docs/tianma-tl060fvxs07-适配说明.md` | 适配说明与排障记录 |
| `.workbuddy/memory/2026-09-27.md` | 排查过程完整记录（含踩过的坑） |

---

## 八、给主线报 issue 的要点（可选）

标题方向：`dw-mipi-dsi2 on RK3588 does not drive the clock lane in command mode`

证据：
1. 接屏实测：初始化 burst 期间 data lane 有 LPDT（105 ns/bit，内容正确），**clock lane 零输出**
2. 时钟只在视频阶段才起来（按块统计）
3. `MIPI_DSI_CLOCK_NON_CONTINUOUS` 不改变上述行为
4. `T_ULPS_EXIT` / `TO_LPTXULPS` 两个 ULPS 宏定义了从未使用

影响：主线内核下 RK3588 无法点亮任何需要初始化命令的 DSI 面板。
