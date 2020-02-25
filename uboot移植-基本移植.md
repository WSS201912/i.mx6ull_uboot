#uboot移植-基本移植

##uboot 移植的一般流程：

①在 uboot 中找到参考的开发平台，一般是原厂的开发板，先编译一下，查看开发环境是否可用。然后，参考原厂开发板移植 uboot 到我们所使用的开发板上。
②修改u-boot

+ 硬件相关
  一般有DRAM、MMC、NAND、串口、网口、显示屏。
  首先要保证DRAM识别正常、SD、EMMC、Nand驱动正常，才能保证uboot启动正常。
  然后修改LCD和网络的驱动。
+ 参数之类
③编译
##具体步骤
1、**修改顶层Makefile中的ARCH和CORSS_COMPLE(247行后)**
2、**添加配置文件。进入configs文件夹，拷贝一个与实际目标板相近的的配置文件。**

```bash
cd configs
cp mx6ull_14x14_evk_emmc_defconfig  mx6ull_ex_emmc_defconfig
```
​      修改mx6ull_ex_emmc_defconfig文件 
修改前

```bash
1 CONFIG_SYS_EXTRA_OPTIONS="IMX_CONFIG=board/freescale/mx6ullevk/imximage.cfg,MX6ULL_EVK_EMMC_REWORK"
2 CONFIG_ARM=y
3 CONFIG_ARCH_MX6=y
4 CONFIG_TARGET_MX6ULL_14X14_EVK=y
5 CONFIG_CMD_GPIO=y
```
修改后
```bash
1 CONFIG_SYS_EXTRA_OPTIONS="IMX_CONFIG=board/freescale/mx6ull_ex_emmc/imximage.cfg,MX6ULL_EVK_EMMC_REWORK"
2 CONFIG_ARM=y
3 CONFIG_ARCH_MX6=y
4 CONFIG_TARGET_MX6ULL_EX_EMMC=y
5 CONFIG_CMD_GPIO=y
```

修改了1和4行。
3、**添加头文件**。
在 目 录 include/configs 下 添 加 对 应 的 头 文 件 ， 复 制
include/configs/mx6ullevk.h，并重命名为 mx6ull_ex_emmc.h，命令如下：

```bash
cp include/configs/mx6ullevk.h mx6ull_ex_emmc.h
```

修改头文件
①打头的条件编译宏定义
 \#ifndef __MX6ULL_EX_EMMC_CONFIG_H
\#define __MX6ULL_EX_EMMC_CONFIG_H
②根据实际需要修改各项的配置宏定义

4、**添加板级文件夹**

①uboot 中每个板子都有一个对应的文件夹来存放板级文件，复制 mx6ullevk，将其重命名为 mx6ull_ex_emmc，

```bash
cd board/freescale/
cp mx6ullevk/ -r mx6ull_ex_emmc
```

②进入 mx6ull_ex_emmc目录，将 其 中 的 mx6ullevk.c 文 件 重 命 名 为mx6ull_ex_emmc.c，命令如下：

```bash
cd mx6ull_alientek_emmc
mv mx6ullevk.c mx6ull_ex_emmc.c
```

  4.1修改mx6ull_ex_emmc目录下的Makefile文件

```bash
1 # (C) Copyright 2015 Freescale Semiconductor, Inc.
2 #
3 # SPDX-License-Identifier: GPL-2.0+
4 #
5
6 obj-y := mx6ull_ex_emmc.o #修改obj-y，与mx6ull_ex_emmc.c对应，这样才能编译mx6ull_ex_emmc.c
7
8 extra-$(CONFIG_USE_PLUGIN) := plugin.bin
9 $(obj)/plugin.bin: $(obj)/plugin.o
10 $(OBJCOPY) -O binary --gap-fill 0xff $< $@
```

  4.2修改mx6ull_ex_emmc目录下的imximage.cfg文件

将 imximage.cfg 中的下面一句：

```bash
PLUGIN board/freescale/mx6ullevk/plugin.bin 0x00907000
```

改为

```c
PLUGIN board/freescale/mx6ull_ex_emmc/plugin.bin 0x00907000
```

  4.3.修改 mx6ull_ex_emmc 目录下的 Kconfig 文件
修改Kconfig文件如下：

```bash
if TARGET_MX6ULL_EX_EMMC           #这里
 config SYS_BOARD
	default "mx6ull_ex_emmc"             #这里

 config SYS_VENDOR
 	default "freescale"

 config SYS_SOC
 	default "mx6"

 config SYS_CONFIG_NAME                 #这里
 	default "mx6ull_ex_emmc"

 endif
```

4.4修改mx6ull_ex_emmc 目录下的 MAINTAINERS 文件

```bash
MX6ULL_ALIENTEK_EMMC BOARD
M: Peng Fan <peng.fan@nxp.com>
S: Maintained
F: board/freescale/mx6ull_ex_emmc/
F: include/configs/mx6ull_ex_emmc
F:  configs/mx6ull_ex_emmc_defconfig
```

4.5修改U-Boot图形配置文件，位于arch/arm/cpu/armv7/mx6/Kconfig，把项目添加的uboot图形配置界面上。
在207行加入如下内容：

```bash
1 config TARGET_MX6ULL_EX_EMMC           #这里
2 	bool "Support mx6ull_ex_emmc"        #这里
3 	select MX6ULL
4 	select DM
5 	select DM_THERMA
```

在最后一行endif的前一行加入

```bash
1 source "board/freescale/mx6ull_ex_emmc/Kconfig"
```

5、编译

```bash
make mx6ull_ex_emmc_defconfig
#配置完毕后
make #编译
```

6、验证是否成功编译
在移植的过程中，可能从在修改不完全或者粗心（比如第一次修改，mx6ull_ex_emmc文件夹中的Kconfig的 SYS_CONFIG_NAME 没有修改，编译的时候有错）。

7、烧写
放入sd卡，然后从sd启动，连接串口，打印如下
![Snipaste_2020-02-25_15-48-54](K:\Linux\从零开发i.mx6ull\uboot的移植\Snipaste_2020-02-25_15-48-54.jpg)

此时的 Board 还是“MX6ULL 14x14 EVK”，因为我们参考的 NXP官方的 I.MX6ULL 开发板来添加自己的开发板。如果接了 LCD 屏幕的话会发现 LCD 屏幕并没有显示 NXP 的 logo，打印了unsupported panel TFT50AB，
野火的开发板用了NXP官方开发板一样的网络配置，所以打印了Net：FEC1 表示用了eNet1

