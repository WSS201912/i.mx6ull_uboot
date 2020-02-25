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

找到实际使用的液晶屏的参数对应填入即可。
结构体 **fb_videomode** 里面的成员变量为 LCD 的参数，这些成员变量函数如下：
**name** ：LCD 名字，要和环境变量中的 panel 相等。
**xres 、yres** ：LCD X 轴和 Y 轴像素数量。
**pixclock**：像素时钟，每个像素时钟周期的长度，单位为皮秒。
**left_margin** ：HBP，水平同步后肩。
**right_margin** ：HFP，水平同步前肩。
**upper_margin**：VBP，垂直同步后肩。
**lower_margin**：VFP，垂直同步前肩。
**hsync_len** ：HSPW，行同步脉宽。
**vsync_len**：VSPW，垂直同步脉宽。
vmode ：大多数使用 FB_VMODE_NONINTERLACED，也就是不使用隔行扫描。四、修改 mx6ull_ex_emmc.h中的panel值。

打开 mx6ull_ex_emmc.h，把其中的"panel=TFT43AB"替换为"panel=TFT50AB",与结构体displays中的.name成员变量的值一致。

五、修改uboot的环境变量中的panel值
完成以上步骤，重新烧写uboot，发现LCD还是无法正常显示，是因为uboot的环境变量中的panel值还没有修改，它存在于emmc中，uboot优先从emmc中读取环境变量。如果emmc保存的panel不等于.name的值，也无法显示。在uboot的命令模式下输入如下命令：

```bash
setenv panel TFT50AB
saveenv
```



