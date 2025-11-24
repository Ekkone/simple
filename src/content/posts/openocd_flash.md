---
title: openocd新增flash driver
published: 2024-07-14
description: ''
updated: ''
tags:
  - openocd
draft: false
pin: 0
toc: true
lang: ''
abbrlink: ''
---

# 文件修改
```shell
src/flash/nor/Makefile.am  // %D%/rk3568.c
src/flash/nor/driver.h     // extern const struct flash_driver rk3568_flash;
src/flash/nor/driver.c     // &rk3568_flash,
src/flash/nor/rk3568.c     //新增驱动文件
```
# 驱动文件实现内容
主要需要实现如下结构体：
```c
/**
 * @brief Provides the implementation-independent structure that defines
 * all of the callbacks required by OpenOCD flash drivers.
 *
 * Driver authors must implement the routines defined here, providing an
 * instance with the fields filled out.  After that, the instance must
 * be registered in flash.c, so it can be used by the driver lookup system.
 *
 * Specifically, the user can issue the command: @par
 * @code
 * flash bank DRIVERNAME ...parameters...
 * @endcode
 *
 * OpenOCD will search for the driver with a @c flash_driver_s::name
 * that matches @c DRIVERNAME.
 *
 * The flash subsystem calls some of the other drivers routines a using
 * corresponding static <code>flash_driver_<i>callback</i>()</code>
 * routine in flash.c.
 */
struct flash_driver {
	/**
	 * Gives a human-readable name of this flash driver,
	 * This field is used to select and initialize the driver.
	 */
	const char *name;

	/**
	 * Gives a human-readable description of arguments.
	 */
	const char *usage;

	/**
	 * An array of driver-specific commands to register.  When called
	 * during the "flash bank" command, the driver can register addition
	 * commands to support new flash chip functions.
	 */
	const struct command_registration *commands;

	/**
	 * Finish the "flash bank" command for @a bank.  The
	 * @a bank parameter will have been filled in by the core flash
	 * layer when this routine is called, and the driver can store
	 * additional information in its struct flash_bank::driver_priv field.
	 *
	 * The CMD_ARGV are: @par
	 * @code
	 * CMD_ARGV[0] = bank
	 * CMD_ARGV[1] = drivername {name above}
	 * CMD_ARGV[2] = baseaddress
	 * CMD_ARGV[3] = lengthbytes
	 * CMD_ARGV[4] = chip_width_in bytes
	 * CMD_ARGV[5] = bus_width_in_bytes
	 * CMD_ARGV[6] = driver-specific parameters
	 * @endcode
	 *
	 * For example, CMD_ARGV[4] = 2 (for 16 bit flash),
	 *	CMD_ARGV[5] = 4 (for 32 bit bus).
	 *
	 * If extra arguments are provided (@a CMD_ARGC > 6), they will
	 * start in @a CMD_ARGV[6].  These can be used to implement
	 * driver-specific extensions.
	 *
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
	__FLASH_BANK_COMMAND((*flash_bank_command));

	/**
	 * Bank/sector erase routine (target-specific).  When
	 * called, the flash driver should erase the specified sectors
	 * using whatever means are at its disposal.
	 *
	 * @param bank The bank of flash to be erased.
	 * @param first The number of the first sector to erase, typically 0.
	 * @param last The number of the last sector to erase, typically N-1.
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
	int (*erase)(struct flash_bank *bank, unsigned int first,
		unsigned int last);

	/**
	 * Bank/sector protection routine (target-specific).
	 *
	 * If protection is not implemented, set method to NULL
	 *
	 * When called, the driver should enable/disable protection
	 * for MINIMUM the range covered by first..last sectors
	 * inclusive. Some chips have alignment requirements will
	 * cause the actual range to be protected / unprotected to
	 * be larger than the first..last range.
	 *
	 * @param bank The bank to protect or unprotect.
	 * @param set If non-zero, enable protection; if 0, disable it.
	 * @param first The first sector to (un)protect, typically 0.
	 * @param last The last sector to (un)project, typically N-1.
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
	int (*protect)(struct flash_bank *bank, int set, unsigned int first,
		unsigned int last);

	/**
	 * Program data into the flash.  Note CPU address will be
	 * "bank->base + offset", while the physical address is
	 * dependent upon current target MMU mappings.
	 *
	 * @param bank The bank to program
	 * @param buffer The data bytes to write.
	 * @param offset The offset into the chip to program.
	 * @param count The number of bytes to write.
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
	int (*write)(struct flash_bank *bank,
			const uint8_t *buffer, uint32_t offset, uint32_t count);

	/**
	 * Read data from the flash. Note CPU address will be
	 * "bank->base + offset", while the physical address is
	 * dependent upon current target MMU mappings.
	 *
	 * @param bank The bank to read.
	 * @param buffer The data bytes read.
	 * @param offset The offset into the chip to read.
	 * @param count The number of bytes to read.
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
	 int (*read)(struct flash_bank *bank,
			uint8_t *buffer, uint32_t offset, uint32_t count);

	/**
	 * Verify data in flash.  Note CPU address will be
	 * "bank->base + offset", while the physical address is
	 * dependent upon current target MMU mappings.
	 *
	 * @param bank The bank to verify
	 * @param buffer The data bytes to verify against.
	 * @param offset The offset into the chip to verify.
	 * @param count The number of bytes to verify.
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
	int (*verify)(struct flash_bank *bank,
			const uint8_t *buffer, uint32_t offset, uint32_t count);

	/**
	 * Probe to determine what kind of flash is present.
	 * This is invoked by the "probe" script command.
	 *
	 * @param bank The bank to probe
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
	int (*probe)(struct flash_bank *bank);

	/**
	 * Check the erasure status of a flash bank.
	 * When called, the driver routine must perform the required
	 * checks and then set the @c flash_sector_s::is_erased field
	 * for each of the flash banks's sectors.
	 *
	 * @param bank The bank to check
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
	int (*erase_check)(struct flash_bank *bank);

	/**
	 * Determine if the specific bank is "protected" or not.
	 * When called, the driver routine must must perform the
	 * required protection check(s) and then set the @c
	 * flash_sector_s::is_protected field for each of the flash
	 * bank's sectors.
	 *
	 * If protection is not implemented, set method to NULL
	 *
	 * @param bank - the bank to check
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
	int (*protect_check)(struct flash_bank *bank);

	/**
	 * Display human-readable information about the flash
	 * bank.
	 *
	 * @param bank - the bank to get info about
	 * @param cmd - command invocation instance for which to generate
	 *              the textual output
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
	int (*info)(struct flash_bank *bank, struct command_invocation *cmd);

	/**
	 * A more gentle flavor of flash_driver_s::probe, performing
	 * setup with less noise.  Generally, driver routines should test
	 * to see if the bank has already been probed; if it has, the
	 * driver probably should not perform its probe a second time.
	 *
	 * This callback is often called from the inside of other
	 * routines (e.g. GDB flash downloads) to autoprobe the flash as
	 * it is programming the flash.
	 *
	 * @param bank - the bank to probe
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
	int (*auto_probe)(struct flash_bank *bank);

	/**
	 * Deallocates private driver structures.
	 * Use default_flash_free_driver_priv() to simply free(bank->driver_priv)
	 *
	 * @param bank - the bank being destroyed
	 */
	void (*free_driver_priv)(struct flash_bank *bank);
};
```
## commands
```c
/*
 * Commands should be registered by filling in one or more of these
 * structures and passing them to [un]register_commands().
 *
 * A conventional format should be used for help strings, to provide both
 * usage and basic information:
 * @code
 * "@<options@> ... - some explanation text"
 * @endcode
 *
 * @param name The name of the command to register, which must not have
 * been registered previously in the intended context.
 * @param handler The callback function that will be called.  If NULL,
 * then the command serves as a placeholder for its children or a script.
 * @param mode The command mode(s) in which this command may be run.
 * @param help The help text that will be displayed to the user.
 */
struct command_registration {
	const char *name;
	command_handler_t handler;
	Jim_CmdProc *jim_handler;
	enum command_mode mode;
	const char *help;
	/** a string listing the options and arguments, required or optional */
	const char *usage;

	/**
	 * If non-NULL, the commands in @c chain will be registered in
	 * the same context and scope of this registration record.
	 * This allows modules to inherit lists commands from other
	 * modules.
	 */
	const struct command_registration *chain;
};
```
### 实现内容

```c
static const struct command_registration at91sam3_command_handlers[] = {
	{
		.name = "at91sam3",
		.mode = COMMAND_ANY,
		.help = "at91sam3 flash command group",
		.usage = "",
		.chain = at91sam3_exec_command_handlers,
	},
	COMMAND_REGISTRATION_DONE
};
```
```c
static const struct command_registration stm32l4_command_handlers[] = {
	{
		.name = "stm32l4x",
		.mode = COMMAND_ANY,
		.help = "stm32l4x flash command group",
		.usage = "",
		.chain = stm32l4_exec_command_handlers,
	},
	COMMAND_REGISTRATION_DONE
};
```
## flash_bank_command
```c
/**
	 * Finish the "flash bank" command for @a bank.  The
	 * @a bank parameter will have been filled in by the core flash
	 * layer when this routine is called, and the driver can store
	 * additional information in its struct flash_bank::driver_priv field.
	 *
	 * The CMD_ARGV are: @par
	 * @code
	 * CMD_ARGV[0] = bank
	 * CMD_ARGV[1] = drivername {name above}
	 * CMD_ARGV[2] = baseaddress
	 * CMD_ARGV[3] = lengthbytes
	 * CMD_ARGV[4] = chip_width_in bytes
	 * CMD_ARGV[5] = bus_width_in_bytes
	 * CMD_ARGV[6] = driver-specific parameters
	 * @endcode
	 *
	 * For example, CMD_ARGV[4] = 2 (for 16 bit flash),
	 *	CMD_ARGV[5] = 4 (for 32 bit bus).
	 *
	 * If extra arguments are provided (@a CMD_ARGC > 6), they will
	 * start in @a CMD_ARGV[6].  These can be used to implement
	 * driver-specific extensions.
	 *
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
/**
	 * 完成@a库的“闪存库”命令。 这
	 * @a组参数将由核心闪存填充
	 * 层，并且驱动程序可以存储
	 * 其结构 flash_bank：:d river_priv 字段中的其他信息。
	 *
	 * CMD_ARGV是：@par
	 * @code
	 * CMD_ARGV[0] = bank
	 * CMD_ARGV[1] = drivername {上面的名字}
	 * CMD_ARGV[2] = 基址
	 * CMD_ARGV[3] = 长度字节
	 * CMD_ARGV[4] = chip_width_in 字节
	 * CMD_ARGV[5] = bus_width_in_bytes
	 * CMD_ARGV[6] = 特定于驱动程序的参数
	 * @endcode
	 *
	 * 例如，CMD_ARGV[4] = 2（对于 16 位闪存），
	 * CMD_ARGV[5] = 4（对于 32 位总线）。
	 *
	 * 如果提供额外的参数（@a CMD_ARGC > 6），它们将
	 * 从 @a CMD_ARGV[6] 开始。 这些可用于实现
	 * 特定于驱动程序的扩展。
	 *
	 * 如果成功，则@returns ERROR_OK;否则为错误代码。
	 */
	__FLASH_BANK_COMMAND((*flash_bank_command));
```
### 传入参数
```c
	#define __FLASH_BANK_COMMAND(name) \

        COMMAND_HELPER(name, struct flash_bank *bank)

/**
 * Provides details of a flash bank, available either on-chip or through
 * a major interface.
 *
 * This structure will be passed as a parameter to the callbacks in the
 * flash_driver_s structure, some of which may modify the contents of
 * this structure of the area of flash that it defines.  Driver writers
 * may use the @c driver_priv member to store additional data on a
 * per-bank basis, if required.
 */
struct flash_bank {
	char *name;

	struct target *target; /**< Target to which this bank belongs. */

	const struct flash_driver *driver; /**< Driver for this bank. */
	void *driver_priv; /**< Private driver storage pointer */

	unsigned int bank_number; /**< The 'bank' (or chip number) of this instance. */
	target_addr_t base; /**< The base address of this bank */
	uint32_t size; /**< The size of this chip bank, in bytes */

	unsigned int chip_width; /**< Width of the chip in bytes (1,2,4 bytes) */
	unsigned int bus_width; /**< Maximum bus width, in bytes (1,2,4 bytes) */

	/** Erased value. Defaults to 0xFF. */
	uint8_t erased_value;

	/** Default padded value used, normally this matches the  flash
	 * erased value. Defaults to 0xFF. */
	uint8_t default_padded_value;

	/** Required alignment of flash write start address.
	 * Default 0, no alignment. Can be any power of two or FLASH_WRITE_ALIGN_SECTOR */
	uint32_t write_start_alignment;
	/** Required alignment of flash write end address.
	 * Default 0, no alignment. Can be any power of two or FLASH_WRITE_ALIGN_SECTOR */
	uint32_t write_end_alignment;
	/** Minimal gap between sections to discontinue flash write
	 * Default FLASH_WRITE_GAP_SECTOR splits the write if one or more untouched
	 * sectors in between.
     * Can be size in bytes or FLASH_WRITE_CONTINUOUS */
	uint32_t minimal_write_gap;

	/**
	 * The number of sectors on this chip.  This value will
	 * be set initially to 0, and the flash driver must set this to
	 * some non-zero value during "probe()" or "auto_probe()".
	 */
	unsigned int num_sectors;
	/** Array of sectors, allocated and initialized by the flash driver */
	struct flash_sector *sectors;

	/**
	 * The number of protection blocks in this bank. This value
	 * is set initially to 0 and sectors are used as protection blocks.
	 * Driver probe can set protection blocks array to work with
	 * protection granularity different than sector size.
	 */
	unsigned int num_prot_blocks;
	/** Array of protection blocks, allocated and initialized by the flash driver */
	struct flash_sector *prot_blocks;

	struct flash_bank *next; /**< The next flash bank on this chip */
};
```
### 实现内容
主要包含内部结构内存申请，变量初始化赋值，参数列表处理三部分内容，如下：
```c
FLASH_BANK_COMMAND_HANDLER(stm32x_flash_bank_command)
{
	struct stm32x_flash_bank *stm32x_info;

	if (CMD_ARGC < 6)
		return ERROR_COMMAND_SYNTAX_ERROR;

	stm32x_info = malloc(sizeof(struct stm32x_flash_bank));

	bank->driver_priv = stm32x_info;
	stm32x_info->probed = false;
	stm32x_info->has_dual_banks = false;
	stm32x_info->can_load_options = false;
	stm32x_info->register_base = FLASH_REG_BASE_B0;
	stm32x_info->user_bank_size = bank->size;

	/* The flash write must be aligned to a halfword boundary */
	bank->write_start_alignment = bank->write_end_alignment = 2;

	return ERROR_OK;
}
```
```c
FLASH_BANK_COMMAND_HANDLER(rp2040_flash_bank_command)
{
	struct rp2040_flash_bank *priv;
	priv = malloc(sizeof(struct rp2040_flash_bank));
	priv->probed = false;

	/* Set up driver_priv */
	bank->driver_priv = priv;

	return ERROR_OK;
}
```
```c
/* flash_bank LPC288x 0 0 0 0 <target#> <cclk> */
FLASH_BANK_COMMAND_HANDLER(lpc288x_flash_bank_command)
{
	struct lpc288x_flash_bank *lpc288x_info;

	if (CMD_ARGC < 6)
		return ERROR_COMMAND_SYNTAX_ERROR;

	lpc288x_info = malloc(sizeof(struct lpc288x_flash_bank));
	bank->driver_priv = lpc288x_info;

	/* part wasn't probed for info yet */
	lpc288x_info->cidr = 0;
	COMMAND_PARSE_NUMBER(u32, CMD_ARGV[6], lpc288x_info->cclk);

	return ERROR_OK;
}
```
```c
FLASH_BANK_COMMAND_HANDLER(swm050_flash_bank_command)
{
	free(bank->sectors);
	bank->write_start_alignment = 4;
	bank->write_end_alignment = 4;
	bank->size = SWM050_FLASH_PAGE_SIZE * SWM050_FLASH_PAGES;

	bank->num_sectors = SWM050_FLASH_PAGES;
	bank->sectors = alloc_block_array(0, SWM050_FLASH_PAGE_SIZE, SWM050_FLASH_PAGES);
	if (!bank->sectors)
		return ERROR_FAIL;

	for (unsigned int i = 0; i < bank->num_sectors; i++)
		bank->sectors[i].is_protected = 0;

	return ERROR_OK;
}
```
## erase
```c
/**
	 * Bank/sector erase routine (target-specific).  When
	 * called, the flash driver should erase the specified sectors
	 * using whatever means are at its disposal.
	 *
	 * @param bank The bank of flash to be erased.
	 * @param first The number of the first sector to erase, typically 0.
	 * @param last The number of the last sector to erase, typically N-1.
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
	 /**
	 * 块/扇区擦除例程（针对特定目标）。  调用时
	 * 闪存驱动程序应使用其所能使用的任何方法擦除指定扇区。
	 *
	 * 参数 bank 要擦除的闪存块。
	 * @param first 要擦除的第一个扇区的编号，通常为 0。
	 * @param last 要擦除的最后一个扇区的编号，通常为 N-1。
	 * 如果成功，则返回 ERROR_OK；否则返回错误代码。
	 */
	int (*erase)(struct flash_bank *bank, unsigned int first,
		unsigned int last);
```
## protect
```c
/**
	 * Bank/sector protection routine (target-specific).
	 *
	 * If protection is not implemented, set method to NULL
	 *
	 * When called, the driver should enable/disable protection
	 * for MINIMUM the range covered by first..last sectors
	 * inclusive. Some chips have alignment requirements will
	 * cause the actual range to be protected / unprotected to
	 * be larger than the first..last range.
	 *
	 * @param bank The bank to protect or unprotect.
	 * @param set If non-zero, enable protection; if 0, disable it.
	 * @param first The first sector to (un)protect, typically 0.
	 * @param last The last sector to (un)project, typically N-1.
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
/**
	 * 块/扇区保护例行程序（针对特定目标）。
	 *
	 * 如果未实施保护，则将方法设为 NULL
	 *
	 * 调用时，驱动程序应启用/禁用保护功能
	 * 最小保护范围为第一个......最后一个扇区
	 * 包含在内。某些芯片的对齐要求会
	 * 导致受保护/不受保护的实际范围
	 * 大于第一个...最后一个扇区的范围。
	 *
	 * @param bank 要保护或取消保护的存储体。
	 * 如果非零，则启用保护；如果为 0，则禁用。
	 * @param first 要（取消）保护的第一个扇区，通常为 0。
	 * @param last 要（解除）保护的最后一个扇区，通常为 N-1。
	 * 如果成功，返回 ERROR_OK；否则，返回错误代码。
	 */
	 
	int (*protect)(struct flash_bank *bank, int set, unsigned int first,
		unsigned int last);
```
## write
```c
/**
	 * Program data into the flash.  Note CPU address will be
	 * "bank->base + offset", while the physical address is
	 * dependent upon current target MMU mappings.
	 *
	 * @param bank The bank to program
	 * @param buffer The data bytes to write.
	 * @param offset The offset into the chip to program.
	 * @param count The number of bytes to write.
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
	 /**
	 * 将数据写入闪存。  注意 CPU 地址将是
	 * "bank->base+offset"，而物理地址取决于当前目标 MMU 映射。
	 * 取决于当前目标 MMU 映射。
	 *
	 * 参数 bank 要编程的存储体
	 * 要写入的数据字节。
	 * @param offset 要编程的芯片偏移量。
	 * 要写入的字节数。
	 * 如果成功，则返回 ERROR_OK；否则，返回错误代码。
	 */
	int (*write)(struct flash_bank *bank,
			const uint8_t *buffer, uint32_t offset, uint32_t count);
```
## read
```c
/**
	 * Read data from the flash. Note CPU address will be
	 * "bank->base + offset", while the physical address is
	 * dependent upon current target MMU mappings.
	 *
	 * @param bank The bank to read.
	 * @param buffer The data bytes read.
	 * @param offset The offset into the chip to read.
	 * @param count The number of bytes to read.
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
	 /**
	 * 从闪存读取数据。注意 CPU 地址将是
	 * "bank->base+offset"，而物理地址取决于当前目标 MMU 映射。
	 * 取决于当前目标 MMU 映射。
	 *
	 * @param bank 要读取的存储体。
	 * @param buffer 读取的数据字节。
	 * @param offset 要读取的芯片偏移量。
	 * 要读取的字节数。
	 * 如果成功，则返回 ERROR_OK；否则，返回错误代码。
	 */
	 int (*read)(struct flash_bank *bank,
			uint8_t *buffer, uint32_t offset, uint32_t count);
```
## verify
```c
/**
	 * Verify data in flash.  Note CPU address will be
	 * "bank->base + offset", while the physical address is
	 * dependent upon current target MMU mappings.
	 *
	 * @param bank The bank to verify
	 * @param buffer The data bytes to verify against.
	 * @param offset The offset into the chip to verify.
	 * @param count The number of bytes to verify.
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
	 /**
	 * 校验闪存中的数据。  注意 CPU 地址将是
	 * "bank->base+offset"，而物理地址取决于当前目标 MMU 映射。
	 * 取决于当前目标 MMU 映射。
	 *
	 * 参数 bank 要验证的 bank
	 * 要验证的数据字节。
	 * @param offset 要验证的芯片偏移量。
	 * @param count 要验证的字节数。
	 * 如果成功，则返回 ERROR_OK；否则，返回错误代码。
	 */
	int (*verify)(struct flash_bank *bank,
			const uint8_t *buffer, uint32_t offset, uint32_t count);
```
## probe
```c
/**
	 * Probe to determine what kind of flash is present.
	 * This is invoked by the "probe" script command.
	 *
	 * @param bank The bank to probe
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
	 /**
	 * 探测以确定存在何种闪存。
	 * 由 "probe "脚本命令调用。
	 *
	 * 参数 bank 要探测的存储体
	 * 如果成功，则返回 ERROR_OK；否则，返回错误代码。
	 */
	int (*probe)(struct flash_bank *bank);
```
## erase_check
```c
/**
	 * Check the erasure status of a flash bank.
	 * When called, the driver routine must perform the required
	 * checks and then set the @c flash_sector_s::is_erased field
	 * for each of the flash banks's sectors.
	 *
	 * @param bank The bank to check
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
	 /**
	 * 检查闪存组的擦除状态。
	 * 调用时，驱动程序例程必须执行所需的检查，然后设置 @c flash_sector_s::is_erased 字段。
	 * 检查，然后设置每个闪存组扇区的 @c flash_sector_s::is_erased 字段。
	 * 为每个闪存组的扇区设置 @c flash_sector_s::is_erased 字段。
	 *
	 * 要检查的闪存库
	 * 如果成功，则返回 ERROR_OK；否则，返回错误代码。
	 */
	int (*erase_check)(struct flash_bank *bank);
```
## protect_check
```c
/**
	 * Determine if the specific bank is "protected" or not.
	 * When called, the driver routine must must perform the
	 * required protection check(s) and then set the @c
	 * flash_sector_s::is_protected field for each of the flash
	 * bank's sectors.
	 *
	 * If protection is not implemented, set method to NULL
	 *
	 * @param bank - the bank to check
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
	 /**
	 * 确定特定银行是否受 "保护"。
	 * 调用时，驱动程序例程必须执行
	 * 必要的保护检查，然后为每个闪存设置 @c
	 * flash_sector_s::is_protected 字段。
	 * 银行扇区。
	 *
	 * 如果未实施保护，则将方法设为 NULL
	 *
	 * @param bank - 要检查的存储体
	 * 如果成功，则返回 ERROR_OK；否则，返回错误代码。
	 */
	int (*protect_check)(struct flash_bank *bank);
```
## info
```c
/**
	 * Display human-readable information about the flash
	 * bank.
	 *
	 * @param bank - the bank to get info about
	 * @param cmd - command invocation instance for which to generate
	 *              the textual output
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
	 /**
	 * 显示闪存块的可读信息
	 *
	 * @param bank - 要获取信息的闪存块
	 * @param cmd - 命令调用实例，用于生成
	 * 文本输出
	 * 如果成功，返回 ERROR_OK；否则，返回错误代码。
	 */
	int (*info)(struct flash_bank *bank, struct command_invocation *cmd);
```
## auto_probe
```c
/**
	 * A more gentle flavor of flash_driver_s::probe, performing
	 * setup with less noise.  Generally, driver routines should test
	 * to see if the bank has already been probed; if it has, the
	 * driver probably should not perform its probe a second time.
	 *
	 * This callback is often called from the inside of other
	 * routines (e.g. GDB flash downloads) to autoprobe the flash as
	 * it is programming the flash.
	 *
	 * @param bank - the bank to probe
	 * @returns ERROR_OK if successful; otherwise, an error code.
	 */
	 /**
	 * 更温和的 flash_driver_s::probe，执行
	 * 设置时噪音较小。  一般来说，驱动程序例程应测试
	 * 银行是否已被探测过。
	 * 驱动程序可能不应再次执行探测。
	 *
	 * 该回调通常在其他
	 * 例程（如 GDB 闪存下载）内部调用该回调，以便在对闪存进行编程时自动探测闪存。
	 * 对闪存进行编程。
	 *
	 * 参数 bank - 要探测的存储体
	 * 如果成功，返回 ERROR_OK；否则，返回错误代码。
	 */
	int (*auto_probe)(struct flash_bank *bank);
```
## free_driver_priv
```c
/**
	 * Deallocates private driver structures.
	 * Use default_flash_free_driver_priv() to simply free(bank->driver_priv)
	 *
	 * @param bank - the bank being destroyed
	 */
	 /**
	 * 卸载私有驱动程序结构。
	 * 使用 default_flash_free_driver_priv() 简单地 free(bank->driver_priv)
	 *
	 * @param bank - 被销毁的存储块
	 */
	void (*free_driver_priv)(struct flash_bank *bank);
```
# 驱动文件依赖内容
## 读写内存
```c
int target_read_u64(struct target *target, target_addr_t address, uint64_t *value);
int target_read_u32(struct target *target, target_addr_t address, uint32_t *value);
int target_read_u16(struct target *target, target_addr_t address, uint16_t *value);
int target_read_u8(struct target *target, target_addr_t address, uint8_t *value);
int target_write_u64(struct target *target, target_addr_t address, uint64_t value);
int target_write_u32(struct target *target, target_addr_t address, uint32_t value);
int target_write_u16(struct target *target, target_addr_t address, uint16_t value);
int target_write_u8(struct target *target, target_addr_t address, uint8_t value);

int target_write_phys_u64(struct target *target, target_addr_t address, uint64_t value);
int target_write_phys_u32(struct target *target, target_addr_t address, uint32_t value);
int target_write_phys_u16(struct target *target, target_addr_t address, uint16_t value);
int target_write_phys_u8(struct target *target, target_addr_t address, uint8_t value);
```
比如`sh_qspi`的初始化，完全由读写内存构成，原理就是通过读写内存的方式操作spi寄存器。
```c
static int sh_qspi_init(struct flash_bank *bank)
{
	struct target *target = bank->target;
	struct sh_qspi_flash_bank *info = bank->driver_priv;
	uint8_t val;
	int ret;

	/* QSPI initialize */
	/* Set master mode only */
	ret = target_write_u8(target, info->io_base + SH_QSPI_SPCR, SPCR_MSTR);
	if (ret != ERROR_OK)
		return ret;

	/* Set SSL signal level */
	ret = target_write_u8(target, info->io_base + SH_QSPI_SSLP, 0x00);
	if (ret != ERROR_OK)
		return ret;

	/* Set MOSI signal value when transfer is in idle state */
	ret = target_write_u8(target, info->io_base + SH_QSPI_SPPCR,
			      SPPCR_IO3FV | SPPCR_IO2FV);
	if (ret != ERROR_OK)
		return ret;

	/* Set bit rate. See 58.3.8 Quad Serial Peripheral Interface */
	ret = target_write_u8(target, info->io_base + SH_QSPI_SPBR, 0x01);
	if (ret != ERROR_OK)
		return ret;

	/* Disable Dummy Data Transmission */
	ret = target_write_u8(target, info->io_base + SH_QSPI_SPDCR, 0x00);
	if (ret != ERROR_OK)
		return ret;

	/* Set clock delay value */
	ret = target_write_u8(target, info->io_base + SH_QSPI_SPCKD, 0x00);
	if (ret != ERROR_OK)
		return ret;

	/* Set SSL negation delay value */
	ret = target_write_u8(target, info->io_base + SH_QSPI_SSLND, 0x00);
	if (ret != ERROR_OK)
		return ret;

	/* Set next-access delay value */
	ret = target_write_u8(target, info->io_base + SH_QSPI_SPND, 0x00);
	if (ret != ERROR_OK)
		return ret;

	/* Set equence command */
	ret = target_write_u16(target, info->io_base + SH_QSPI_SPCMD0,
			       SPCMD_INIT2);
	if (ret != ERROR_OK)
		return ret;

	/* Reset transfer and receive Buffer */
	ret = target_read_u8(target, info->io_base + SH_QSPI_SPBFCR, &val);
	if (ret != ERROR_OK)
		return ret;

	val |= SPBFCR_TXRST | SPBFCR_RXRST;

	ret = target_write_u8(target, info->io_base + SH_QSPI_SPBFCR, val);
	if (ret != ERROR_OK)
		return ret;

	/* Clear transfer and receive Buffer control bit */
	ret = target_read_u8(target, info->io_base + SH_QSPI_SPBFCR, &val);
	if (ret != ERROR_OK)
		return ret;

	val &= ~(SPBFCR_TXRST | SPBFCR_RXRST);

	ret = target_write_u8(target, info->io_base + SH_QSPI_SPBFCR, val);
	if (ret != ERROR_OK)
		return ret;

	/* Set equence control method. Use equence0 only */
	ret = target_write_u8(target, info->io_base + SH_QSPI_SPSCR, 0x00);
	if (ret != ERROR_OK)
		return ret;

	/* Enable SPI function */
	ret = target_read_u8(target, info->io_base + SH_QSPI_SPCR, &val);
	if (ret != ERROR_OK)
		return ret;

	val |= SPCR_SPE;

	return target_write_u8(target, info->io_base + SH_QSPI_SPCR, val);
}
```
## SPI相关
内置了很多型号的spi flash的操作指令码及参数可以根据型号直接使用：
```c
// SPDX-License-Identifier: GPL-2.0-or-later

/***************************************************************************
 *   Copyright (C) 2018 by Andreas Bolsch                                  *
 *   andreas.bolsch@mni.thm.de                                             *
 *                                                                         *
 *   Copyright (C) 2012 by George Harris                                   *
 *   george@luminairecoffee.com                                            *
 *                                                                         *
 *   Copyright (C) 2010 by Antonio Borneo                                  *
 *   borneo.antonio@gmail.com                                              *
 ***************************************************************************/

#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "imp.h"
#include "spi.h"
#include <jtag/jtag.h>

 /* Shared table of known SPI flash devices for SPI-based flash drivers. Taken
  * from device datasheets and Linux SPI flash drivers. */
const struct flash_device flash_devices[] = {
	/* Note: device_id is usually 3 bytes long, however the unused highest byte counts
	 * continuation codes for manufacturer id as per JEP106xx.
	 *
	 * All sizes (page, sector/block and flash) are in bytes.
	 *
	 * Guide to select a proper erase command (if both sector and block erase cmds are available):
	 * Use 4kbit sector erase cmd and set erase size to the size of sector for small devices
	 * (4Mbit and less, size <= 0x80000) to prevent too raw erase granularity.
	 * Use 64kbit block erase cmd and set erase size to the size of block for bigger devices
	 * (8Mbit and more, size >= 0x100000) to keep erase speed reasonable.
	 * If the device implements also 32kbit block erase, use it for 8Mbit, size == 0x100000.
	 */
	/*        name                  read qread  page  erase chip  device_id   page   erase   flash
	 *                              _cmd _cmd   _prog _cmd* _erase            size   size*   size
	 *                                          _cmd        _cmd
	 */
	FLASH_ID("st m25pe10",          0x03, 0x00, 0x02, 0xd8, 0x00, 0x00118020, 0x100, 0x10000, 0x20000),
	FLASH_ID("st m25pe20",          0x03, 0x00, 0x02, 0xd8, 0x00, 0x00128020, 0x100, 0x10000, 0x40000),
	FLASH_ID("st m25pe40",          0x03, 0x00, 0x02, 0xd8, 0x00, 0x00138020, 0x100, 0x10000, 0x80000),
	FLASH_ID("st m25pe80",          0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00148020, 0x100, 0x10000, 0x100000),
	FLASH_ID("st m25pe16",          0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00158020, 0x100, 0x10000, 0x200000),
	FLASH_ID("st m25p05",           0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00102020, 0x80,  0x8000,  0x10000),
	FLASH_ID("st m25p10",           0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00112020, 0x80,  0x8000,  0x20000),
	FLASH_ID("st m25p20",           0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00122020, 0x100, 0x10000, 0x40000),
	FLASH_ID("st m25p40",           0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00132020, 0x100, 0x10000, 0x80000),
	FLASH_ID("st m25p80",           0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00142020, 0x100, 0x10000, 0x100000),
	FLASH_ID("st m25p16",           0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00152020, 0x100, 0x10000, 0x200000),
	FLASH_ID("st m25p32",           0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00162020, 0x100, 0x10000, 0x400000),
	FLASH_ID("st m25p64",           0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00172020, 0x100, 0x10000, 0x800000),
	FLASH_ID("st m25p128",          0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00182020, 0x100, 0x40000, 0x1000000),
	FLASH_ID("st m45pe10",          0x03, 0x00, 0x02, 0xd8, 0xd8, 0x00114020, 0x100, 0x10000, 0x20000),
	FLASH_ID("st m45pe20",          0x03, 0x00, 0x02, 0xd8, 0xd8, 0x00124020, 0x100, 0x10000, 0x40000),
	FLASH_ID("st m45pe40",          0x03, 0x00, 0x02, 0xd8, 0xd8, 0x00134020, 0x100, 0x10000, 0x80000),
	FLASH_ID("st m45pe80",          0x03, 0x00, 0x02, 0xd8, 0xd8, 0x00144020, 0x100, 0x10000, 0x100000),
	FLASH_ID("sp s25fl004",         0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00120201, 0x100, 0x10000, 0x80000),
	FLASH_ID("sp s25fl008",         0x03, 0x08, 0x02, 0xd8, 0xc7, 0x00130201, 0x100, 0x10000, 0x100000),
	FLASH_ID("sp s25fl016",         0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00140201, 0x100, 0x10000, 0x200000),
	FLASH_ID("sp s25fl116k",        0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00154001, 0x100, 0x10000, 0x200000),
	FLASH_ID("sp s25fl032",         0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00150201, 0x100, 0x10000, 0x400000),
	FLASH_ID("sp s25fl132k",        0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00164001, 0x100, 0x10000, 0x400000),
	FLASH_ID("sp s25fl064",         0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00160201, 0x100, 0x10000, 0x800000),
	FLASH_ID("sp s25fl164k",        0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00174001, 0x100, 0x10000, 0x800000),
	FLASH_ID("sp s25fl128s",        0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00182001, 0x100, 0x10000, 0x1000000),
	FLASH_ID("sp s25fl256s",        0x13, 0x00, 0x12, 0xdc, 0xc7, 0x00190201, 0x100, 0x10000, 0x2000000),
	FLASH_ID("sp s25fl512s",        0x13, 0x00, 0x12, 0xdc, 0xc7, 0x00200201, 0x200, 0x40000, 0x4000000),
	FLASH_ID("cyp s25fl064l",       0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00176001, 0x100, 0x10000, 0x800000),
	FLASH_ID("cyp s25fl128l",       0x03, 0x00, 0x02, 0xd8, 0xc7, 0x00186001, 0x100, 0x10000, 0x1000000),
	FLASH_ID("cyp s25fl256l",       0x13, 0x00, 0x12, 0xdc, 0xc7, 0x00196001, 0x100, 0x10000, 0x2000000),
	FLASH_ID("cyp s28hl256t",       0x13, 0xec, 0x12, 0xdc, 0xc7, 0x00195a34, 0x100, 0x40000, 0x2000000), /* page! */
	FLASH_ID("cyp s28hs256t",       0x13, 0xec, 0x12, 0xdc, 0xc7, 0x00195b34, 0x100, 0x40000, 0x2000000), /* page! */
	FLASH_ID("cyp s28hl512t",       0x13, 0xec, 0x12, 0xdc, 0xc7, 0x001a5a34, 0x100, 0x40000, 0x4000000), /* page! */
	FLASH_ID("cyp s28hs512t",       0x13, 0xec, 0x12, 0xdc, 0xc7, 0x001a5b34, 0x100, 0x40000, 0x4000000), /* page! */
	FLASH_ID("cyp s28hl01gt",       0x13, 0xec, 0x12, 0xdc, 0xc7, 0x001b5a34, 0x100, 0x40000, 0x8000000), /* page! */
	FLASH_ID("cyp s28hs01gt",       0x13, 0xec, 0x12, 0xdc, 0xc7, 0x001b5b34, 0x100, 0x40000, 0x8000000), /* page! */
	FLASH_ID("atmel 25f512",        0x03, 0x00, 0x02, 0x52, 0xc7, 0x0065001f, 0x80,  0x8000,  0x10000),
	FLASH_ID("atmel 25f1024",       0x03, 0x00, 0x02, 0x52, 0x62, 0x0060001f, 0x100, 0x8000,  0x20000),
	FLASH_ID("atmel 25f2048",       0x03, 0x00, 0x02, 0x52, 0x62, 0x0063001f, 0x100, 0x10000, 0x40000),
	FLASH_ID("atmel 25f4096",       0x03, 0x00, 0x02, 0x52, 0x62, 0x0064001f, 0x100, 0x10000, 0x80000),
	FLASH_ID("atmel 25fs040",       0x03, 0x00, 0x02, 0xd7, 0xc7, 0x0004661f, 0x100, 0x10000, 0x80000),
	FLASH_ID("adesto 25sf041b",     0x03, 0x00, 0x02, 0xd8, 0xc7, 0x0001841f, 0x100, 0x10000, 0x80000),
	FLASH_ID("adesto 25df081a",     0x03, 0x00, 0x02, 0xd8, 0xc7, 0x0001451f, 0x100, 0x10000, 0x100000),
	FLASH_ID("adesto 25sf081b",     0x03, 0x00, 0x02, 0xd8, 0xc7, 0x0001851f, 0x100, 0x10000, 0x100000),
	FLASH_ID("adesto 25sf161b",     0x03, 0x00, 0x02, 0xd8, 0xc7, 0x0001861f, 0x100, 0x10000, 0x200000),
	FLASH_ID("adesto 25df321b",     0x03, 0x00, 0x02, 0xd8, 0xc7, 0x0001471f, 0x100, 0x10000, 0x400000),
	FLASH_ID("adesto 25sf321b",     0x03, 0x00, 0x02, 0xd8, 0xc7, 0x0001871f, 0x100, 0x10000, 0x400000),
	FLASH_ID("adesto 25xf641b",     0x03, 0x00, 0x02, 0xd8, 0xc7, 0x0001881f, 0x100, 0x10000, 0x800000), /* sf/qf */
	FLASH_ID("adesto 25xf128a",     0x03, 0x00, 0x02, 0xd8, 0xc7, 0x0001891f, 0x100, 0x10000, 0x1000000), /* sf/qf */
	FLASH_ID("adesto xp032",        0x03, 0x0b, 0x02, 0xd8, 0xc7, 0x0700a743, 0x100, 0x10000, 0x400000), /* 4-byte */
	FLASH_ID("adesto xp064b",       0x03, 0x0b, 0x02, 0xd8, 0xc7, 0x0001a81f, 0x100, 0x10000, 0x800000), /* 4-byte */
	FLASH_ID("adesto xp128",        0x03, 0x0b, 0x02, 0xd8, 0xc7, 0x0000a91f, 0x100, 0x10000, 0x1000000), /* 4-byte */
	FLASH_ID("mac 25l512",          0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001020c2, 0x010, 0x10000, 0x10000),
	FLASH_ID("mac 25l1005",         0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001120c2, 0x010, 0x10000, 0x20000),
	FLASH_ID("mac 25l2005",         0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001220c2, 0x010, 0x10000, 0x40000),
	FLASH_ID("mac 25l4005",         0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001320c2, 0x010, 0x10000, 0x80000),
	FLASH_ID("mac 25l8005",         0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001420c2, 0x010, 0x10000, 0x100000),
	FLASH_ID("mac 25l1605",         0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001520c2, 0x100, 0x10000, 0x200000),
	FLASH_ID("mac 25l3205",         0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001620c2, 0x100, 0x10000, 0x400000),
	FLASH_ID("mac 25l6405",         0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001720c2, 0x100, 0x10000, 0x800000),
	FLASH_ID("mac 25l12845",        0x03, 0xeb, 0x02, 0xd8, 0xc7, 0x001820c2, 0x100, 0x10000, 0x1000000),
	FLASH_ID("mac 25l25645",        0x13, 0xec, 0x12, 0xdc, 0xc7, 0x001920c2, 0x100, 0x10000, 0x2000000),
	FLASH_ID("mac 25l51245",        0x13, 0xec, 0x12, 0xdc, 0xc7, 0x001a20c2, 0x100, 0x10000, 0x4000000),
	FLASH_ID("mac 25lm51245",       0x13, 0xec, 0x12, 0xdc, 0xc7, 0x003a85c2, 0x100, 0x10000, 0x4000000),
	FLASH_ID("mac 25r512f",         0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001028c2, 0x100, 0x10000, 0x10000),
	FLASH_ID("mac 25r1035f",        0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001128c2, 0x100, 0x10000, 0x20000),
	FLASH_ID("mac 25r2035f",        0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001228c2, 0x100, 0x10000, 0x40000),
	FLASH_ID("mac 25r4035f",        0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001328c2, 0x100, 0x10000, 0x80000),
	FLASH_ID("mac 25r8035f",        0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001428c2, 0x100, 0x10000, 0x100000),
	FLASH_ID("mac 25r1635f",        0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001528c2, 0x100, 0x10000, 0x200000),
	FLASH_ID("mac 25r3235f",        0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001628c2, 0x100, 0x10000, 0x400000),
	FLASH_ID("mac 25r6435f",        0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001728c2, 0x100, 0x10000, 0x800000),
	FLASH_ID("mac 25u1635e",        0x03, 0xeb, 0x02, 0x20, 0xc7, 0x003525c2, 0x100, 0x1000,  0x100000),
	FLASH_ID("mac 66l1g45g",        0x13, 0xec, 0x12, 0xdc, 0xc7, 0x001b20c2, 0x100, 0x10000, 0x8000000),
	FLASH_ID("mac 66um1g45g",       0x13, 0xec, 0x12, 0xdc, 0xc7, 0x003b80c2, 0x100, 0x10000, 0x8000000),
	FLASH_ID("mac 66lm1g45g",       0x13, 0xec, 0x12, 0xdc, 0xc7, 0x003b85c2, 0x100, 0x10000, 0x8000000),
	FLASH_ID("micron n25q032",      0x03, 0xeb, 0x02, 0xd8, 0xc7, 0x0016ba20, 0x100, 0x10000, 0x400000),
	FLASH_ID("micron n25q064",      0x03, 0xeb, 0x02, 0xd8, 0xc7, 0x0017ba20, 0x100, 0x10000, 0x800000),
	FLASH_ID("micron n25q128",      0x03, 0xeb, 0x02, 0xd8, 0xc7, 0x0018ba20, 0x100, 0x10000, 0x1000000),
	FLASH_ID("micron n25q256 3v",   0x13, 0xec, 0x12, 0xdc, 0xc7, 0x0019ba20, 0x100, 0x10000, 0x2000000),
	FLASH_ID("micron n25q256 1.8v", 0x13, 0xec, 0x12, 0xdc, 0xc7, 0x0019bb20, 0x100, 0x10000, 0x2000000),
	FLASH_ID("micron mt25ql512",    0x13, 0xec, 0x12, 0xdc, 0xc7, 0x0020ba20, 0x100, 0x10000, 0x4000000),
	FLASH_ID("micron mt25ql01",     0x13, 0xec, 0x12, 0xdc, 0xc7, 0x0021ba20, 0x100, 0x10000, 0x8000000),
	FLASH_ID("micron mt25qu01",     0x13, 0xec, 0x12, 0xdc, 0xc7, 0x0021bb20, 0x100, 0x10000, 0x8000000),
	FLASH_ID("micron mt25ql02",     0x13, 0xec, 0x12, 0xdc, 0xc7, 0x0022ba20, 0x100, 0x10000, 0x10000000),
	FLASH_ID("win w25q80bv",        0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001440ef, 0x100, 0x10000, 0x100000),
	FLASH_ID("win w25q16jv",        0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001540ef, 0x100, 0x10000, 0x200000),
	FLASH_ID("win w25q16jv",        0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001570ef, 0x100, 0x10000, 0x200000), /* QPI / DTR */
	FLASH_ID("win w25q32fv/jv",     0x03, 0xeb, 0x02, 0xd8, 0xc7, 0x001640ef, 0x100, 0x10000, 0x400000),
	FLASH_ID("win w25q32fv",        0x03, 0xeb, 0x02, 0xd8, 0xc7, 0x001660ef, 0x100, 0x10000, 0x400000), /* QPI mode */
	FLASH_ID("win w25q32jv",        0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001670ef, 0x100, 0x10000, 0x400000),
	FLASH_ID("win w25q64fv/jv",     0x03, 0xeb, 0x02, 0xd8, 0xc7, 0x001740ef, 0x100, 0x10000, 0x800000),
	FLASH_ID("win w25q64fv",        0x03, 0xeb, 0x02, 0xd8, 0xc7, 0x001760ef, 0x100, 0x10000, 0x800000), /* QPI mode */
	FLASH_ID("win w25q64jv",        0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001770ef, 0x100, 0x10000, 0x800000),
	FLASH_ID("win w25q128fv/jv",    0x03, 0xeb, 0x02, 0xd8, 0xc7, 0x001840ef, 0x100, 0x10000, 0x1000000),
	FLASH_ID("win w25q128fv",       0x03, 0xeb, 0x02, 0xd8, 0xc7, 0x001860ef, 0x100, 0x10000, 0x1000000), /* QPI mode */
	FLASH_ID("win w25q128jv",       0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001870ef, 0x100, 0x10000, 0x1000000),
	FLASH_ID("win w25q256fv/jv",    0x03, 0xeb, 0x02, 0xd8, 0xc7, 0x001940ef, 0x100, 0x10000, 0x2000000),
	FLASH_ID("win w25q256fv",       0x03, 0xeb, 0x02, 0xd8, 0xc7, 0x001960ef, 0x100, 0x10000, 0x2000000), /* QPI mode */
	FLASH_ID("win w25q256jv",       0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001970ef, 0x100, 0x10000, 0x2000000),
	FLASH_ID("win w25q512jv",       0x03, 0x00, 0x02, 0xd8, 0xc7, 0x002040ef, 0x100, 0x10000, 0x4000000),
	FLASH_ID("win w25q01jv",        0x13, 0x00, 0x12, 0xdc, 0xc7, 0x002140ef, 0x100, 0x10000, 0x8000000),
	FLASH_ID("win w25q01jv-dtr",    0x03, 0xeb, 0x02, 0xd8, 0xc7, 0x002170ef, 0x100, 0x10000, 0x8000000),
	FLASH_ID("win w25q02jv-dtr",    0x03, 0xeb, 0x02, 0xd8, 0xc7, 0x002270ef, 0x100, 0x10000, 0x10000000),
	FLASH_ID("gd gd25q512",         0x03, 0x00, 0x02, 0x20, 0xc7, 0x001040c8, 0x100, 0x1000,  0x10000),
	FLASH_ID("gd gd25q10",          0x03, 0x00, 0x02, 0x20, 0xc7, 0x001140c8, 0x100, 0x1000,  0x20000),
	FLASH_ID("gd gd25q20",          0x03, 0x00, 0x02, 0x20, 0xc7, 0x001240c8, 0x100, 0x1000,  0x40000),
	FLASH_ID("gd gd25q40",          0x03, 0x00, 0x02, 0x20, 0xc7, 0x001340c8, 0x100, 0x1000,  0x80000),
	FLASH_ID("gd gd25q16c",         0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001540c8, 0x100, 0x10000, 0x200000),
	FLASH_ID("gd gd25q32c",         0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001640c8, 0x100, 0x10000, 0x400000),
	FLASH_ID("gd gd25q64c",         0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001740c8, 0x100, 0x10000, 0x800000),
	FLASH_ID("gd gd25q128c",        0x03, 0x0b, 0x02, 0xd8, 0xc7, 0x001840c8, 0x100, 0x10000, 0x1000000),
	FLASH_ID("gd gd25q256c",        0x13, 0x00, 0x12, 0xdc, 0xc7, 0x001940c8, 0x100, 0x10000, 0x2000000),
	FLASH_ID("gd gd25q512mc",       0x13, 0x00, 0x12, 0xdc, 0xc7, 0x002040c8, 0x100, 0x10000, 0x4000000),
	FLASH_ID("gd gd25lx256e",       0x13, 0x0c, 0x12, 0xdc, 0xc7, 0x001968c8, 0x100, 0x10000, 0x2000000),
	FLASH_ID("zbit zb25vq16",       0x03, 0x00, 0x02, 0xd8, 0xc7, 0x0015605e, 0x100, 0x10000, 0x200000),
	FLASH_ID("zbit zb25vq32",       0x03, 0x0b, 0x02, 0xd8, 0xc7, 0x0016405e, 0x100, 0x10000, 0x400000),
	FLASH_ID("zbit zb25vq32",       0x03, 0x0b, 0x02, 0xd8, 0xc7, 0x0016605e, 0x100, 0x10000, 0x400000), /* QPI mode */
	FLASH_ID("zbit zb25vq64",       0x03, 0x0b, 0x02, 0xd8, 0xc7, 0x0017405e, 0x100, 0x10000, 0x800000),
	FLASH_ID("zbit zb25vq64",       0x03, 0x0b, 0x02, 0xd8, 0xc7, 0x0017605e, 0x100, 0x10000, 0x800000), /* QPI mode */
	FLASH_ID("zbit zb25vq128",      0x03, 0x0b, 0x02, 0xd8, 0xc7, 0x0018405e, 0x100, 0x10000, 0x1000000),
	FLASH_ID("zbit zb25vq128",      0x03, 0x0b, 0x02, 0xd8, 0xc7, 0x0018605e, 0x100, 0x10000, 0x1000000), /* QPI mode */
	FLASH_ID("issi is25lq040b",     0x03, 0xeb, 0x02, 0x20, 0xc7, 0x0013409d, 0x100, 0x1000,  0x80000),
	FLASH_ID("issi is25lp032",      0x03, 0x00, 0x02, 0xd8, 0xc7, 0x0016609d, 0x100, 0x10000, 0x400000),
	FLASH_ID("issi is25lp064",      0x03, 0x00, 0x02, 0xd8, 0xc7, 0x0017609d, 0x100, 0x10000, 0x800000),
	FLASH_ID("issi is25lp128d",     0x03, 0xeb, 0x02, 0xd8, 0xc7, 0x0018609d, 0x100, 0x10000, 0x1000000),
	FLASH_ID("issi is25wp128d",     0x03, 0xeb, 0x02, 0xd8, 0xc7, 0x0018709d, 0x100, 0x10000, 0x1000000),
	FLASH_ID("issi is25lp256d",     0x13, 0xec, 0x12, 0xdc, 0xc7, 0x0019609d, 0x100, 0x10000, 0x2000000),
	FLASH_ID("issi is25wp256d",     0x13, 0xec, 0x12, 0xdc, 0xc7, 0x0019709d, 0x100, 0x10000, 0x2000000),
	FLASH_ID("issi is25lp512m",     0x13, 0xec, 0x12, 0xdc, 0xc7, 0x001a609d, 0x100, 0x10000, 0x4000000),
	FLASH_ID("issi is25wp512m",     0x13, 0xec, 0x12, 0xdc, 0xc7, 0x001a709d, 0x100, 0x10000, 0x4000000),
	FLASH_ID("xtx xt25f02e",        0x03, 0x00, 0x02, 0xd8, 0xc7, 0x0012400b, 0x100, 0x10000, 0x40000),
	FLASH_ID("xtx xt25f04d",        0x03, 0x00, 0x02, 0xd8, 0xc7, 0x0013400b, 0x100, 0x10000, 0x80000),
	FLASH_ID("xtx xt25f08b",        0x03, 0x00, 0x02, 0xd8, 0xc7, 0x0014400b, 0x100, 0x10000, 0x100000),
	FLASH_ID("xtx xt25f16b",        0x03, 0x00, 0x02, 0xd8, 0xc7, 0x0015400b, 0x100, 0x10000, 0x200000),
	FLASH_ID("xtx xt25f32b",        0x03, 0x0b, 0x02, 0xd8, 0xc7, 0x0016400b, 0x100, 0x10000, 0x200000),
	FLASH_ID("xtx xt25f64b",        0x03, 0x0b, 0x02, 0xd8, 0xc7, 0x0017400b, 0x100, 0x10000, 0x400000),
	FLASH_ID("xtx xt25f128b",       0x03, 0x0b, 0x02, 0xd8, 0xc7, 0x0018400b, 0x100, 0x10000, 0x800000),
	FLASH_ID("xtx xt25q08d",        0x03, 0x0b, 0x02, 0xd8, 0xc7, 0x0014600b, 0x100, 0x10000, 0x100000),
	FLASH_ID("xtx xt25q16b",        0x03, 0x0b, 0x02, 0xd8, 0xc7, 0x0015600b, 0x100, 0x10000, 0x200000),
	FLASH_ID("xtx xt25q32b",        0x03, 0x0b, 0x02, 0xd8, 0xc7, 0x0016600b, 0x100, 0x10000, 0x400000), /* exists ? */
	FLASH_ID("xtx xt25q64b",        0x03, 0x0b, 0x02, 0xd8, 0xc7, 0x0017600b, 0x100, 0x10000, 0x800000),
	FLASH_ID("xtx xt25q128b",       0x03, 0x0b, 0x02, 0xd8, 0xc7, 0x0018600b, 0x100, 0x10000, 0x1000000),
	FLASH_ID("zetta zd25q16",       0x03, 0x00, 0x02, 0xd8, 0xc7, 0x001560ba, 0x100, 0x10000, 0x200000),

	/* FRAM, no erase commands, no write page or sectors */

	/*        name                  read qread  page  device_id   total
	 *                              _cmd _cmd   _prog             size
	 *                                          _cmd
	 */
	FRAM_ID("fu mb85rs16n",         0x03, 0,    0x02, 0x00010104, 0x800),
	FRAM_ID("fu mb85rs32v",         0x03, 0,    0x02, 0x00010204, 0x1000), /* exists ? */
	FRAM_ID("fu mb85rs64v",         0x03, 0,    0x02, 0x00020304, 0x2000),
	FRAM_ID("fu mb85rs128b",        0x03, 0,    0x02, 0x00090404, 0x4000),
	FRAM_ID("fu mb85rs256b",        0x03, 0,    0x02, 0x00090504, 0x8000),
	FRAM_ID("fu mb85rs512t",        0x03, 0,    0x02, 0x00032604, 0x10000),
	FRAM_ID("fu mb85rs1mt",         0x03, 0,    0x02, 0x00032704, 0x20000),
	FRAM_ID("fu mb85rs2mta",        0x03, 0,    0x02, 0x00034804, 0x40000),
	FRAM_ID("cyp fm25v01a",         0x03, 0,    0x02, 0x060821c2, 0x4000),
	FRAM_ID("cyp fm25v02",          0x03, 0,    0x02, 0x060022c2, 0x8000),
	FRAM_ID("cyp fm25v02a",         0x03, 0,    0x02, 0x060822c2, 0x8000),
	FRAM_ID("cyp fm25v05",          0x03, 0,    0x02, 0x060023c2, 0x10000),
	FRAM_ID("cyp fm25v10",          0x03, 0,    0x02, 0x060024c2, 0x20000),
	FRAM_ID("cyp fm25v20a",         0x03, 0,    0x02, 0x060825c2, 0x40000),
	FRAM_ID("cyp fm25v40",          0x03, 0,    0x02, 0x064026c2, 0x80000),
	FRAM_ID("cyp cy15b102qn",       0x03, 0,    0x02, 0x06002ac2, 0x40000),
	FRAM_ID("cyp cy15v102qn",       0x03, 0,    0x02, 0x06042ac2, 0x40000),
	FRAM_ID("cyp cy15b104qi",       0x03, 0,    0x02, 0x06012dc2, 0x80000),
	FRAM_ID("cyp cy15b104qi",       0x03, 0,    0x02, 0x06a12dc2, 0x80000),
	FRAM_ID("cyp cy15v104qi",       0x03, 0,    0x02, 0x06052dc2, 0x80000),
	FRAM_ID("cyp cy15v104qi",       0x03, 0,    0x02, 0x06a52dc2, 0x80000),
	FRAM_ID("cyp cy15b104qn",       0x03, 0,    0x02, 0x06402cc2, 0x80000),
	FRAM_ID("cyp cy15b108qi",       0x03, 0,    0x02, 0x06012fc2, 0x100000),
	FRAM_ID("cyp cy15b108qi",       0x03, 0,    0x02, 0x06a12fc2, 0x100000),
	FRAM_ID("cyp cy15v108qi",       0x03, 0,    0x02, 0x06052fc2, 0x100000),
	FRAM_ID("cyp cy15v108qi",       0x03, 0,    0x02, 0x06a52fc2, 0x100000),
	FRAM_ID("cyp cy15b108qn",       0x03, 0,    0x02, 0x06012ec2, 0x100000),
	FRAM_ID("cyp cy15b108qn",       0x03, 0,    0x02, 0x06032ec2, 0x100000),
	FRAM_ID("cyp cy15b108qn",       0x03, 0,    0x02, 0x06a12ec2, 0x100000),
	FRAM_ID("cyp cy15v108qn",       0x03, 0,    0x02, 0x06052ec2, 0x100000),
	FRAM_ID("cyp cy15v108qn",       0x03, 0,    0x02, 0x06072ec2, 0x100000),
	FRAM_ID("cyp cy15v108qn",       0x03, 0,    0x02, 0x06a52ec2, 0x100000),

	FLASH_ID(NULL,                  0,    0,    0,    0,    0,    0,          0,     0,       0)
};

```
