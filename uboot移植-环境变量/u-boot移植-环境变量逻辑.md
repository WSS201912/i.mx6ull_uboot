#u-boot移植-环境变量逻辑

u-boot的环境变量用来存储一些经常使用的参数变量，uboot希望将环境变量存储在静态存储器中（如nand nor eeprom mmc）。启动uboot，打印环境变量，如图：

![Snipaste_2020-02-28_14-23-51](K:\Linux\从零开发i.mx6ull\uboot的移植\i.mx6ull_uboot\uboot移植-环境变量\Snipaste_2020-02-28_14-23-51.jpg)

**Uboot环境变量设计逻辑**
Uboot环境变量的设计逻辑是在启动过程中将env从静态存储器中读出放到RAM中，之后在uboot下对env的操作（如printenv editenv setenv）都是对RAM中env的操作，只有在执行saveenv时才会将RAM中的env重新写入静态存储器中。
这种设计逻辑可以加快对env的读写速度。

**uboot中env的整个架构**
（1） 命令层，如saveenv，setenv editenv这些命令的实现，还有如启动时调用的env_relocat函数(ENV迁移函数)。
（2） 中间封装层，利用不同静态存储器特性封装出命令层需要使用的一些通用函数，如env_init,env_relocate_spec,saveenv这些函数。实现文件在common/env_xxx.c
（3） 驱动层，实现不同静态存储器的读写擦等操作，这些是uboot下不同子系统都必须的。

##一、env数据结构

在include/environment.h中定义了env_t，如下：

```c
typedef struct environment_s {
	uint32_t	crc;		/* CRC32 over data bytes	*/

	unsigned char	data[ENV_SIZE]; /* Environment data		*/
} env_t
  
#define ENV_SIZE (CONFIG_ENV_SIZE - ENV_HEADER_SIZE)
#define CONFIG_ENV_SIZE			SZ_8K//在include/configs/mx6ull_ex_emmc.h中
# define ENV_HEADER_SIZE	(sizeof(uint32_t))
```

CONFIG_ENV_SIZE是我们需要在配置文件中配置的环境变量的总长度，根据存储介质设置。
Env_t结构体头4个bytes是对data的crc校验码，没有定义CONFIG_SYS_REDUNDAND_ENVIRONMENT，所以后面紧跟data数组，数组大小是ENV_SIZE.ENV_SIZE是CONFIG_ENV_SIZE减掉ENV_HEADER_SIZE，也就是4bytes，所以env_t这个结构体就包含了整个我们规定的长度为CONFIG_ENV_SIZE的存储区域。头4bytes是crc校验码，后面剩余的空间全部用来存储环境变量。

<!--需要说明的一点，crc校验码是uboot中在saveenv时计算出来，然后写入存储介质，所以在第一次启动uboot时crc校验会出错（因为之前没有执行saveenv，介质没有crc），因为uboot从存储介质上读入的一个block数据是随机的，没有意义的，执行saveenv后重启uboot，crc校验就正确了.data 字段保存实际的环境变量。u-boot的env按name=value”\0”的方式存储，在所有env 的最后以”\0\0”表示整个env的结束。新的name=value 对总是被添加到env数据块的末尾，当删除一个 name=value 对时，后面的环境变量将前移，对一个已经存在的环境变量的修改实际上先删除再插入。-->

u-boot 把env_t 的数据指针还保存在了另外一个地方,这就是gd_t 结构（不同平台有不同的 gd_t 结构 ）,这里以ARM 为例仅列出和 env 相关的部分 

```c
typedef struct global_data 
{ 
     … 
     unsigned long env_off;        /* Relocation Offset */ 
     unsigned long env_addr;       /* Address of Environment struct  */ 
     unsigned long env_valid       /* Checksum of Environment valid */ 
     … 
} gd_t;        
```



## 二、 env初始化

按照执行流顺序，首先分析一下uboot启动的env初始化过程。
首先在board_init_f中调用init_sequence的env_init，这个函数是不同存储器实现的函数，env_mmc中的实现如下：

```c
int env_init(void)
{
	/* use default */
	gd->env_addr	= (ulong)&default_environment[0];
	gd->env_valid	= 1;

	return 0;
}
```

env_nand.c中如下

```c
/*
 * This is called before nand_init() so we can't read NAND to
 * validate env data.
 *
 * Mark it OK for now. env_relocate() in env_common.c will call our
 * relocate function which does the real validation.
 *
 * When using a NAND boot image (like sequoia_nand), the environment
 * can be embedded or attached to the U-Boot image in NAND flash.
 * This way the SPL loads not only the U-Boot image from NAND but
 * also the environment.
 */
int env_init(void)
{
    //去掉ifdef/* ENV_IS_EMBEDDED || CONFIG_NAND_ENV_DST */
	gd->env_addr	= (ulong)&default_environment[0];
	gd->env_valid	= 1;
	return 0;
}
```

从注释就基本可以看出这个函数的作用，因为env_init要早于静态存储器的初始化，所以无法进行env的读写，这里将gd中的env相关变量进行配置，默认设置env为valid。方便后面env_relocate函数进行真正的env从nand到ram的relocate(迁移)。
继续执行，在commom/board_r.c中的board_init_r函数中执行initr_env函数，如下：

```c
static int initr_env(void)
{
	/* initialize environment */
	if (should_load_env())
		env_relocate();
	else
		set_default_env(NULL);

	/* Initialize from environment */
	load_addr = getenv_ulong("loadaddr", 16, load_addr);
	return 0;
}
```

这是在所有存储器初始化完成后执行的（init_sequence_r[]中initr_nand或initr_mmc在initr_env之前）。
首先调用commom/board_r.c中的should_load_env，如下：

```c
/*
 * Tell if it's OK to load the environment early in boot.
 *
 * If CONFIG_OF_CONTROL is defined, we'll check with the FDT to see
 * if this is OK (defaulting to saying it's OK).
 *
 * NOTE: Loading the environment early can be a bad idea if security is
 *       important, since no verification is done on the environment.
 *
 * @return 0 if environment should not be loaded, !=0 if it is ok to load
 */
static int should_load_env(void)
{
#ifdef CONFIG_OF_CONTROL
	return fdtdec_get_config_int(gd->fdt_blob, "load-environment", 1);
#elif defined CONFIG_DELAY_ENVIRONMENT
	return 0;
#else
	return 1;
#endif
}
```

从注释可以看出，CONFIG_OF_CONTROL没有定义，鉴于考虑安全性问题，如果我们想要推迟env的load，可以定义CONFIG_DELAY_ENVIRONMENT,如果定义，返回0。我们可以在之后的某个地方在调用env_relocate来load env。这里我们选择在这里直接load env。所以没有定义CONFIG_DELAY_ENVIRONMENT（u-boot中默认没有定义CONFIG_DELAY_ENVIRONMENT）,返回1。调用env_relocate。

在common/env_common.c中：

```c
void env_relocate(void)
{
    //...
	if (gd->env_valid == 0) {
      //...
		bootstage_error(BOOTSTAGE_ID_NET_CHECKSUM);
		set_default_env("!bad CRC");
	} else {
		env_relocate_spec();
	}
}
```

gd->env_valid在之前的env_init中设置为1，所以这里调用env_relocate_spec,
这个函数也是不同存储器的中间封装层提供的函数，在commom/env_nand.c中

```c

void env_relocate_spec(void)
{
   int ret;
    ALLOC_CACHE_ALIGN_BUFFER(char, buf, CONFIG_ENV_SIZE);
    ret = readenv(CONFIG_ENV_OFFSET, (u_char *)buf);
    if (ret) {
        set_default_env("!readenv() failed");
        return;
    }
    env_import(buf, 1);
}
```

在commom/env_mmc.c中

```c
void env_relocate_spec(void)
{
#if !defined(ENV_IS_EMBEDDED)
	ALLOC_CACHE_ALIGN_BUFFER(char, buf, CONFIG_ENV_SIZE);//定义长度为CONFIG_ENV_SIZE的buf
	struct mmc *mmc;
	u32 offset;
	int ret;
	int dev = mmc_get_env_dev();//找到当前用的mmc设备 sd卡启动的话 就是sd卡 emmc启动就是emmc 也可                                   以设置从那个设备取
	const char *errmsg;

	mmc = find_mmc_device(dev);

	errmsg = init_mmc_for_env(mmc);//初始化mmc设备 确保可以读到env
	if (errmsg) {
		ret = 1;
		goto err;
	}

	if (mmc_get_env_addr(mmc, 0, &offset)) {         //得到env在mmc中的偏移地址
		ret = 1;
		goto fini;
	}

	if (read_env(mmc, CONFIG_ENV_SIZE, offset, buf)) {//从mmc读取env
		errmsg = "!read failed";
		ret = 1;
		goto fini;
	}

	env_import(buf, 1);
	ret = 0;

fini:
	fini_mmc_for_env(mmc);
err:
	if (ret)
		set_default_env(errmsg);
#endif
}
```

两种情况：
1 存储介质中没有env（比如第一次烧写启动），执行set_default_env。
2 存储介质有env，buf读到后，调用env_import，如下：

```c
/*
 * Check if CRC is valid and (if yes) import the environment.
 * Note that "buf" may or may not be aligned.
 */
int env_import(const char *buf, int check)
{
    env_t *ep = (env_t *)buf;
    if (check) {
        uint32_t crc;
 
        memcpy(&crc, &ep->crc, sizeof(crc));
 
        if (crc32(0, ep->data, ENV_SIZE) != crc) {
            set_default_env("!bad CRC");
            return 0;
        }
    }
    if (himport_r(&env_htab, (char *)ep->data, ENV_SIZE, '\0', 0,
            0, NULL)) {
        gd->flags |= GD_FLG_ENV_READY;
        return 1;
    }
    error("Cannot import environment: errno = %d\n", errno);
    set_default_env("!import failed");
    return 0;
}
```

首先将buf强制转换为env_t类型，然后对data进行crc校验，跟buf中原有的crc对比，不一致则使用默认env。
最后调用himport_r，该函数将给出的data按照‘\0’分割填入env_htab的哈希表中。
之后对于env的操作，如printenv setenv editenv，都是对该哈希表的操作。
Env_relocate执行完成，env的初始化就完成了。

##三、env的操作实现

Uboot对env的操作命令实现在common/cmd_nvedit.c中。

对于setenv printenv editenv这3个命令，看其实现代码，都是对relocate到RAM中的env_htab的操作，这里就不再详细分析了，重点来看一下savenv实现。

```c
static int do_env_save(cmd_tbl_t *cmdtp, int flag, int argc,
		       char * const argv[])
{
	printf("Saving Environment to %s...\n", env_name_spec);
	return saveenv() ? 1 : 0;
}

U_BOOT_CMD(
	saveenv, 1, 0,	do_env_save,
	"save environment variables to persistent storage",
	""
);
```

在do_env_save调用saveenv,这个函数是不同存储器实现的封装层函数。每种存储介质都有对应的env_xxx.c，且里边有对env的操作函数。以sd卡（mmc）为例，所以saveenv用的env_mmc.c下函数操作。

```c
int saveenv(void)
{
	ALLOC_CACHE_ALIGN_BUFFER(env_t, env_new, 1);
	int dev = mmc_get_env_dev();           //得到对env操作的设备（默认为启动的设备，可以设置）
	struct mmc *mmc = find_mmc_device(dev);
	u32	offset;
	int	ret, copy = 0;
	const char *errmsg;

	errmsg = init_mmc_for_env(mmc);       //初始化env的设备
	if (errmsg) {
		printf("%s\n", errmsg);
		return 1;
	}

	ret = env_export(env_new);//Emport the environment and generate CRC for it. 
	if (ret)
		goto fini;


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
```

第一步，使用mmc_get_env_dev函数得到env的设备，env_mmc.c下的mmc_get_env_dev标记了weak，soc.c中有此函数，如下。得到启动的设备。不再展开了。
第二步，env_export函数从哈希表（在哈希表中做的修改）中得到环境变量，并校验。
最后，得到偏移量，写入新的环境变量（全部）。

在savenv readenv函数以及printenv setenv的实现函数中涉及到的函数himport_r hexport_r hdelete_r hmatch_r都是对env_htab哈希表的一些基本操作函数。

这些函数都封装在uboot的lib/hashtable.c中，这里就不仔细分析这些函数了。

##四、MMC启动，环境变量保存位置

通过以上分析，初始化和保存环境变量都调用了mmc_get_env_dev函数，如下：

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

mmc_get_boot_dev函数一顿底层寻址操作，确定是从MMC0还是MMC1启动，然后返回。所以u-boot模式使在启动的mmc设备上操作env。

这个函数自uboot启动初始化env时就会调用，
![Snipaste_2020-02-27_17-07-13](K:\Linux\从零开发i.mx6ull\uboot的移植\i.mx6ull_uboot\uboot移植-环境变量\Snipaste_2020-02-27_17-07-13.jpg)

可以看到MMC:FSL_SDHC:  0,FSL_SDHC: 1的下一行
In mx6/soc.c-mmc_get_env_part devno = 0  这段是自行添加的。如图

![Snipaste_2020-02-27_17-10-43](K:\Linux\从零开发i.mx6ull\uboot的移植\i.mx6ull_uboot\uboot移植-环境变量\Snipaste_2020-02-27_17-10-43.jpg)

每次执行saveenv命令都调用这个函数，打印出OFFSET

![Snipaste_2020-02-27_17-12-57](K:\Linux\从零开发i.mx6ull\uboot的移植\i.mx6ull_uboot\uboot移植-环境变量\Snipaste_2020-02-27_17-12-57.jpg)