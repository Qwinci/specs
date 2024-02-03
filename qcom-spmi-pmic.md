# Unofficial specification for Qualcomm SPMI PMIC Arbiter

## Glossary
- PPID
	- 12-bit peripheral identifier
	
			[0-7] MSB of 16-bit address
			[8-11] SID
- APID
	- Internally used 8-bit arbiter peripheral identifier
	- Software maps PPID->APID
- EE
	- Execution environment
- ARB_EE
	- Current EE
	- DT property `qcom,ee`
- MAX_PERIPHS
	- Maximum number of peripherals, 512 for < V7 and 1024 for V7
- ARB_CHANNEL
	- DT property `qcom,channel`
- Max transfer size is 8 bytes
- Opcodes:
	```
	ARB_OP_EXT_WRITEL = 0,
	ARB_OP_EXT_READL = 1,
	ARB_OP_EXT_WRITE = 2,
	ARB_OP_RESET = 3,
	ARB_OP_SLEEP = 4,
	ARB_OP_SHUTDOWN = 5,
	ARB_OP_WAKEUP = 6,
	ARB_OP_AUTHENTICATE = 7,
	ARB_OP_MSTR_READ = 8,
	ARB_OP_MSTR_WRITE = 9,
	ARB_OP_EXT_READ = 13,
	ARB_OP_WRITE = 14,
	ARB_OP_READ = 15,
	ARB_OP_ZERO_WRITE = 16
	```
- [XX-XX] Bits XX-XX (inclusive)
- [0xXX-0xXX] Bytes XX-XX (exclusive)

## Registers
- Core (DT register "core")
	- [0x0-0x4] ARB_VERSION
		
		Arbiter version
			
			V1 <  0x20010000
			V2 <  0x30000000
			V3 <  0x50000000
			V5 <  0x70000000
			V7 >= 0x70000000
	
	- [0x4-0x8] ARB_FEATURES

		Arbiter features
			
			if V7 and DT has optional property "qcom,bus-id" with value 1
				[0-10] BASE_APID
			else if V5
				[0-10] NUM_PERIPHS
					- Number of peripherals

	- [0x8-0xC] ARB_FEATURES1

		Arbiter features 1

			if V7 and DT has optional property "qcom,bus-id" with value 1
				[0-10] NUM_PERIPHS
					- Number of peripherals

	- ARB_REG_CHNLn (>= V5)

			BASE_APID = 0 for < V7
			n = BASE_APID..BASE_APID + NUM_PERIPHS
			V1-V3: 0x800  + 0x4 * i
			V5:    0x900  + 0x4 * i
			V7:    0x2000 + 0x4 * i

		Software address translation table (full zero entries should be ignored)

			[8-19] PPID for APID i
			[24] EE irq owner
				- Highest APID which has the irq owner bit set will receive the irqs from PPID

- Write channels (On V1 same as Core, else DT register "chnls")

	Data area for channel n starts at the following offsets, note that V1 only supports one channel that is the arbiter channel:

		V1: 0x800 + 0x80 * ARB_CHANNEL
		V2-V3: 0x1000 * ARB_EE + 0x8000 * APID
		V5: 0x10000 * APID
		V7: 0x1000 * APID

	- [0x0-0x4] CHN_CMD

		Channel command

			V1:
				[0-2] Byte count - 1
				[4-19] Address
				[20-23] SID
				[27-31] Opcode
			V2-V7:
				[0-2] Byte count - 1
				[4-11] LSB of address
				[27-31] Opcode

	- [0x4-0x8] CHN_CONFIG

		Channel config

	- [0x8-0xC] CHN_STATUS

		Channel status

			[0] Done
			[1] Transaction failure
			[2] Transaction denied
			[3] Transaction dropped

	- [0x10-0x14] CHN_WDATA0

		Channel write fifo data 0

	- [0x14-0x18] CHN_WDATA1

		Channel write fifo data 1 (used if writing more than 4 bytes)

	- [0x18-0x1C] CHN_RDATA0

		Channel read fifo data 0
	
	- [0x1C-0x1F] CHR_RDATA1

		Channel read fifo data 1 (used if reading more than 4 bytes)

	- ARB_IRQ_STATUSn (>= V5)

			V5: 0x104 + 0x10000 * i
			V7: 0x104 + 0x1000 * i

		Interrupt status for peripheral i

			[0-7] Each bit indicates an interrupt
	
	- ARB_IRQ_CLEARn (>= V5)

			V5: 0x108 + 0x10000 * i
			V7: 0x108 + 0x1000 * i

		Interrupt clear for peripheral i

			[0-7] Writing one to a bit clears the corresponding interrupt

- Read channels (On V1 same as Core, else DT register "obsrvr")
	- Same starting offsets and registers as `Write channels`
- Interrupters (DT register "intr")

	- ARB_IRQ_STATUSn (<= V3)

			V1: 0x600 + 0x4 * i
			V2-V3: 0x4 + 0x1000 * i

		Interrupt status for peripheral i

			[0-7] Each bit indicates one interrupt
	
	- ARB_IRQ_CLEARn (<= V3)

			V1: 0xA00 + 0x4 * i
			V2-V3: 0x8 + 0x1000 * i

		Interrupt clear for peripheral i

			[0-7] Writing one to a bit clears the corresponding interrupt
	
	- ARB_OWNER_ACC_STATUS

			V1: 0x20 * OWNER_EE + 0x4 * i
			V2: 0x100000 + 0x1000 * OWNER_EE + 0x4 * i
			V3: 0x200000 + 0x1000 * OWNER_EE + 0x4 * i
			V5: 0x10000 * OWNER_EE + 0x4 * i
			V7: 0x1000 * OWNER_EE + 0x4 * i

		Owner accumulated interrupt status for peripherals `32 * (i + 1)`

			[0-31] Accumulated interrupt status, each bit indicates one peripheral

	- ARB_ACC_ENABLE

			V1: 0x200 + 0x4 * i
			V2-V3: 0x1000 * i
			V5: 0x100 + 0x10000 * i
			V7: 0x100 + 0x1000 * i

		Accumulated interrupt enable for APID i

			[0] Accumulated interrupt enable

- Channel Config (DT register "cnfg")

	- SPMI_PERIPHx_OWNER

			V1-V5: 0x700 + 0x4 * i
			V7:	   0x4 * (i - BASE_APID)
		
		Write owner EE for APID i

			[0-2] Write owner EE
	
	- SPMI_MAPPING_TABLEx

			V1: 0xB00 + 0x4 * i

		Mapping table used to convert PPID to APID (only needed on V1)

			[18-21] Bit index
			[17] Bit is 0 flag
			[9-16] Bit is 0 result
			[8] Bit is 1 flag
			[0-7] Bit is 1 result

		Below is pseudocode for the algorithm used to calculate the APID:

		```
		i = 0
		// The mapping tree contains at max 16 levels
		For depth in 0..15:
			data = read(cnfg, 0xB00 + 0x4 * i)
			
			if is_bit_set(data, data.bit_index):
				if data.bit_is_1_flag:
					i = data.bit_is_1_result
				else:
					apid = data.bit_is_1_result
			else:
				if data.bit_is_0_flag:
					i = data.bit_is_0_result
				else:
					apid = data.bit_is_0_result
		```
