# Embedded-Systems-Final-Project

## About Me

I am a junior in IE and self-taught Embedded Systems Engineer.

Currently an Embedded Systems Engineer at Continental AG working in Industrial IoT, where I primarily work with:
- Bare-metal / Embedded Linux drivers
- Closed-loop control systems
- Low-level networking
- PCBs and schematics

## Purpose

Why I selected this project:
- Understand ARM acrhitecture and the MCU startup procedure
- Stop relying on proprietary vendor SDKs
- Improve my skills in working with the wide array of open-source Embedded Software Tools 
- Learn to work with new MCUs or ones I design myself
- Gain respect for the people that work with these board bring up process before any tools are created

## Progress

### 0. Board Selection and Toolchain Setup

#### Selected Board
| Board | MCU Datasheet | Board Data Brief |
| ----- | ------------- | --------------- |
| [32F411EDISCOVERY](https://www.st.com/en/evaluation-tools/32f411ediscovery.html) | [STM32F411xC STM32F411xE](https://www.digikey.ch/htmldatasheets/production/1776125/0/0/1/stm32f411xc-stm32f411xe.html) | [Discovery kit with STM32F411VE MCU](https://www.st.com/resource/en/data_brief/32f411ediscovery.pdf) |

#### Overview

##### Tools
| Tool | Purpose |
| ---- | ------- |
| Ccache | accelerated recompilation |
| GNU Arm Embedded | compilation and linking |
| GNU Make | build automation |
| STLINK Tools | debug and flashing |
| ARM Assembly | memory and thread profiling |

##### Programming Languages
- ARM Assembly
- C
- C++

##### Test
| Tool | Purpose |
| ---- | ------- |
| Digital Multimeter | continuity and coupling |
| Logic Probe | reading pin values |
| Oscilloscope | in-circuit validation |

### 1. Project Architecture

The project primarily consists of:
- Drivers
- Embedded webserver
- Power Supply

![image](https://user-images.githubusercontent.com/65039828/234351505-14eef787-66e9-48a5-9525-9acca3ce7cdf.png)


### 2. Datasheet Review

The datasheet contains information necessary for much of the project. The information gathered will be used for everything from controlling the clock frequency, to creating the IVT and its corresponding interrupt handlers.

#### Memory Map

The following information can be found in Section 5 of the MCU datasheet. There are other sections of memory, but these are the ones necessary for this project.

| Section | Start Address | Stop Address | Size |
| ------- | ------------- | ------------ | ---- |
| Code | 0x0000 0000 | 0x1FFF FFFF | 512 Mb |
| SRAM (aliased by bit-banding) | 0x2000 0000 | 0x2002 0000 | 128 Kb | 
| Peripherals | 0x4000 0000 | 0x5FFF FFFF | 512 Mb |
| Flash | 0x0800 0000 | 0x0807 FFFF | 512 Kb |

![image](https://user-images.githubusercontent.com/65039828/234352358-77d14d8d-fff4-468e-934a-d594a478757a.png)


#### Peripherals

The following peripherals will be used in this project, and can be found in Section 5 of the MCU datasheet.

| Peripheral | Bus | Start Address |
| ------- | --- | ------------- |
| USART1 | APB2 | 0x4001 1000 |
| USART6 | APB2 | 0x4001 1400 |
| GPIOA | AHB1 | 0x4002 0000 |
| GPIOB | AHB1 | 0x4002 0400 |

![image](https://user-images.githubusercontent.com/65039828/234352876-be59c644-599e-4c1d-8aed-e0c7dc799e27.png)
![image](https://user-images.githubusercontent.com/65039828/234352912-b0416102-61a5-4808-b50c-3b63b30cb664.png)

### Linker Script

From `link.ld`:
```
/* set entry point to beginning of the firmware */
ENTRY(_reset);

/* define the memory regions */
MEMORY {
	flash(rx) : ORIGIN = 0x08000000, LENGTH = 512k		
	sram(rwx) : ORIGIN = 0x20000000, LENGTH = 128k
}

/* set stack pointer to end of SRAM */
_estack = ORIGIN(sram) + LENGTH(sram);

SECTIONS {
	/* store vector table in the following order: */
	.vectors  : { KEEP(*(.vectors)) }   > flash		/* 1. flash memory */
	.text     : { *(.text*) }           > flash		/* 2. firmware section */
	.rodata   : { *(.rodata*) }         > flash		/* 3. read-only data section */

	.data : {
		_sdata = .;   								/* section start */
		*(.first_data)
		/* sort sections in ascending order in output file */
			*(.data SORT(.data.*))
			_edata = .;  								/* section end */
} > sram AT > flash
_sidata = LOADADDR(.data);

.bss : {
	_sbss = .;              					/* section start */
	/* sort sections in ascending order in output file */
	*(.bss SORT(.bss.*) COMMON)
	_ebss = .;              					/* section end */
} > sram

/* insert padding bytes to align on an 8-byte boundary */
. = ALIGN(8);

_end = .;
}

```

## License

FultronRTOS is released under the GNU GENERAL PUBLIC LICENSE. See LICENSE for more information.
