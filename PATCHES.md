# 对厂商 SDK 的改动

本包内的 `csl/` 代码版权归 华芯微特（Synwit）所有，来自官方各系列
`02.SDK/00.标准外设库及参考示例/*_Lib` 包。为了让它在 PlatformIO 下可用，
只做了下面两类改动，**都由 `platform-synwit/tools/import_csl.py` 自动施加**，
重新导入后依然生效。所有被改动的行都带 `>>> PIO adaptation` 注释，方便 diff。

## 1. system_<芯片>.c — 时钟宏加 #ifndef 保护

把下列宏（各系列有差异，缺失的跳过）包进 `#ifndef ... #endif`，默认值不变：

```
SYS_CLK  __HSI  __LSI  __HSE  __LSE
SYS_PLL_SRC  PLL_IN_DIV  PLL_FB_DIV  PLL_OUT_DIV
```

原因：官方 SDK 要换时钟必须改源文件，而 PlatformIO 的配置入口是
`platformio.ini`。加了保护之后可以直接写：

```ini
board_build.synwit_clock = sys_clk_24mhz        ; 见 mcu.json 的 clock_presets
; 或任意组合
board_build.synwit_clock = SYS_CLK=SYS_CLK_XTAL, __HSE=8000000UL
```

涉及文件：`csl/*/CMSIS/DeviceSupport/system_SWM{166,181,190,201,211,221,241,260,320,341}.c`

## 2. swm221/SWM221_StdPeriph_Driver/SWM221_flash.c — 补 GCC 版 Cache_Clear()

厂商只给了 `__CC_ARM`（AC5/AC6）和 `__ICCARM__` 两个分支的 `Cache_Clear()`，
没有 `__GNUC__` 分支，导致 GCC 链接 `SystemInit()` 时报
`undefined reference to 'Cache_Clear'`。按 IAR 分支的等价实现补了一个：

```c
#elif defined ( __GNUC__ )
void Cache_Clear(void)
{
	FMC->CACHE |= FMC_CACHE_CCLR_Msk;	// Cache Clear
	__NOP(); __NOP(); __NOP(); __NOP();
}
#endif
```

## 已知的厂商 SDK 缺陷（未改，用 src_filter 规避）

- **SWM341**：`CMSIS/DSP_Src/SupportFunctions/` 里的排序函数需要 `arm_sorting.h`，
  但 `CMSIS/DSP_Inc` 里并没有（DSP_Inc 与 DSP_Src 版本不匹配）。
  `mcu.json` 中 dsp 的 `src_filter` 排除了这些文件，其余 CMSIS-DSP 正常。
- **SWM181/190/201/211/221/241/260**：SDK 里没有 CMSIS-DSP，所以也没有 dsp 选项；
  只有 SWM320（`CMSIS/DSP_Lib`）和 SWM341（`CMSIS/DSP_Src`）有。
- **SWM166/221/241**：官方没有提供 GCC 链接脚本，这三个系列的 `.ld` 是按
  `SWM181/swm181.ld` 的段布局、配合 pdsc 里的实际 Flash/RAM 容量生成的
  （文件头有说明）。SWM181/190/201/211/260/320/341 用的是官方原版脚本，
  只改了 MEMORY 的长度，让每个器件的容量都对得上。
