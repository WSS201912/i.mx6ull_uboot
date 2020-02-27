uboot环境变量 saveenv

uboot.从什么位置启动就保存在哪里。

每种存储介质都有对应的env_xxx.c，且里边有对env的操作函数，uboot移植时设计好。本次试验用的sd卡，所以saveenv用的env_mmc.c下函数操作。

```c
common/env_mmc.c
#ifdef CONFIG_CMD_SAVEENV
static inline int write_env(struct mmc *mmc, unsigned long size,
			    unsigned long offset, const void *buffer)
{
	uint blk_start, blk_cnt, n;

	blk_start	= ALIGN(offset, mmc->write_bl_len) / mmc->write_bl_len;
	blk_cnt		= ALIGN(size, mmc->write_bl_len) / mmc->write_bl_len;

	n = mmc->block_dev.block_write(&mmc->block_dev, blk_start,
					blk_cnt, (u_char *)buffer);

	return (n == blk_cnt) ? 0 : -1;
}


int saveenv(void)
{
	ALLOC_CACHE_ALIGN_BUFFER(env_t, env_new, 1);
	int dev = mmc_get_env_dev();//得到env的设备
	struct mmc *mmc = find_mmc_device(dev);
	u32	offset;
	int	ret, copy = 0;
	const char *errmsg;

	errmsg = init_mmc_for_env(mmc);
	if (errmsg) {
		printf("%s\n", errmsg);
		return 1;
	}

	ret = env_export(env_new);
	if (ret)
		goto fini;

#ifdef CONFIG_ENV_OFFSET_REDUND
	env_new->flags	= ++env_flags; /* increase the serial */

	if (gd->env_valid == 1)
		copy = 1;
#endif

	if (mmc_get_env_addr(mmc, copy, &offset)) {
		ret = 1;
		goto fini;
	}

	printf("Writing to %sMMC(%d)... ", copy ? "redundant " : "", dev);
	if (write_env(mmc, CONFIG_ENV_SIZE, offset, (u_char *)env_new)) {
		puts("failed\n");
		ret = 1;
		goto fini;
	}

	puts("done\n");
	ret = 0;

#ifdef CONFIG_ENV_OFFSET_REDUND
	gd->env_valid = gd->env_valid == 2 ? 1 : 2;
#endif

fini:
	fini_mmc_for_env(mmc);
	return ret;
}
#endif /* CONFIG_CMD_SAVEENV */
```

第一步，使用mmc_get_env_dev函数得到env的设备，env_mmc.c下的mmc_get_env_dev标记了weak，soc.c中有此函数，如下。得到启动的设备。不再展开了。

```c
//arch/arm/cpu/armv7/mx6/soc.c
int mmc_get_env_dev(void)
{
	int devno = mmc_get_boot_dev();

	/* If not boot from sd/mmc, use default value */
	if (devno < 0)
		return CONFIG_SYS_MMprintC_ENV_DEV;

	return board_mmc_get_env_dev(devno);
}
```

这个函数自uboot启动自动运行
![Snipaste_2020-02-27_17-07-13](K:\Linux\从零开发i.mx6ull\uboot的移植\i.mx6ull_uboot\uboot移植-环境变量\Snipaste_2020-02-27_17-07-13.jpg)

可以看到MMC:FSL_SDHC:  0,FSL_SDHC: 1的下一行
In mx6/soc.c-mmc_get_env_part devno = 0  这段是自行添加的。如图

![Snipaste_2020-02-27_17-10-43](K:\Linux\从零开发i.mx6ull\uboot的移植\i.mx6ull_uboot\uboot移植-环境变量\Snipaste_2020-02-27_17-10-43.jpg)

每次执行saveenv命令都调用这个函数，打印出OFFSET

![Snipaste_2020-02-27_17-12-57](K:\Linux\从零开发i.mx6ull\uboot的移植\i.mx6ull_uboot\uboot移植-环境变量\Snipaste_2020-02-27_17-12-57.jpg)

