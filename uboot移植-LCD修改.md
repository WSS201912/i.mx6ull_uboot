# uboot移植-LCD修改

一般 uboot 中修改驱动基本都是在 xxx.h 和 xxx.c 这两个文件中，比如本例中的
include/configs/mx6ull_ex_emmc.h和board/freescale/mx6ull_ex_emmc/mx6ull_ex_emmc.c。

LCD驱动修改的重点为：
①、LCD 所使用的 GPIO，查看 uboot 中 LCD 的 IO 配置是否正确。
②、LCD 背光引脚 GPIO 的配置。
③、LCD 配置参数是否正确。

一、LCD 的 GPIO
mx6ull_ex_emmc.c下查找数组lcd_pads[]，其为iomux_v3_cfg_t型数组，查看定义为

```c
typedef u64 iomux_v3_cfg_t;
```

以CLK引脚为例：

```c
static iomux_v3_cfg_t const lcd_pads[] = {
	MX6_PAD_LCD_CLK__LCDIF_CLK | MUX_PAD_CTRL(LCD_PAD_CTRL),
	MX6_PAD_LCD_ENABLE__LCDIF_ENABLE | MUX_PAD_CTRL(LCD_PAD_CTRL),
    ...
    /* Use GPIO for Brightness adjustment, duty cycle = period. */
	MX6_PAD_GPIO1_IO08__GPIO1_IO08 | MUX_PAD_CTRL(NO_PAD_CTRL),}
```

```c
//mx6ull_pins.h 511行
MX6_PAD_LCD_CLK__LCDIF_CLK  = IOMUX_PAD(0x0390, 0x0104, 0, 0x0000, 0, 0),
//iomux-v3.h 67行
#define MUX_PAD_CTRL(x)		((iomux_v3_cfg_t)(x) << MUX_PAD_CTRL_SHIFT)
//mx6ull_ex_emmc.c 63行
#define LCD_PAD_CTRL    (PAD_CTL_HYS | PAD_CTL_PUS_100K_UP | PAD_CTL_PUE | \
					  PAD_CTL_PKE | PAD_CTL_SPEED_MED | PAD_CTL_DSE_40ohm)
```
lcd_pads定义的前半部分为gpio引脚，后半部分为LCD的设置（不用修改）。所以对照实际板子的引脚做修改。

二、LCD 背光引脚 GPIO 的配置

背光引脚为lcd_pads的最后一个MX6_PAD_GPIO1_IO08__GPIO1_IO08 | MUX_PAD_CTRL(NO_PAD_CTRL)

三、LCD 配置参数

```c
struct display_info_t const displays[] = {{
	.bus = MX6UL_LCDIF1_BASE_ADDR,
	.addr = 0,
	.pixfmt = 24,
	.detect = NULL,
	.enable	= do_enable_parallel_lcd,
	.mode	= {
		.name			= "TFT50AB",
		.xres           = 800,
		.yres           = 480,
		.pixclock       = 108695,
		.left_margin    = 46,
		.right_margin   = 22,
		.upper_margin   = 23,
		.lower_margin   = 22,
		.hsync_len      = 1,
		.vsync_len      = 1,
		.sync           = 0,
		.vmode          = FB_VMODE_NONINTERLACED
} } };
```

