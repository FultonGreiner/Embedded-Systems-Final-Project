# Embedded-Systems-Final-Project

## Purpose

Why I selected this project:
- Understand ARM architecture and the MCU startup procedure
- Stop relying on proprietary vendor SDKs
- Improve my skills in working with the wide array of open-source Embedded Software Tools 
- Learn to work with new MCUs or ones I design myself
- Gain respect for the people that work with these board bring up process before any tools are created

## Progress

### 0. Board Selection

#### Selected Board
| Board | MCU Datasheet | Board Data Brief |
| ----- | ------------- | --------------- |
| [32F411EDISCOVERY](https://www.st.com/en/evaluation-tools/32f411ediscovery.html) | [STM32F411xC STM32F411xE](https://www.digikey.ch/htmldatasheets/production/1776125/0/0/1/stm32f411xc-stm32f411xe.html) | [Discovery kit with STM32F411VE MCU](https://www.st.com/resource/en/data_brief/32f411ediscovery.pdf) |

### 1. Toolchain Setup

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

### 2. Project Architecture

The project primarily consists of:
- Startup code
- Drivers
- Embedded webserver

![image](https://user-images.githubusercontent.com/65039828/234351505-14eef787-66e9-48a5-9525-9acca3ce7cdf.png)


### 3. Datasheet Review

The datasheet contains information necessary for much of the project. The information gathered will be used for everything from controlling the clock frequency, to creating the IVT and its corresponding interrupt handlers.

### 4. Memory Map

The following information can be found in Section 5 of the MCU datasheet. There are other sections of memory, but these are the ones necessary for this project.

| Section | Start Address | Stop Address | Size |
| ------- | ------------- | ------------ | ---- |
| Code | 0x0000 0000 | 0x1FFF FFFF | 512 Mb |
| SRAM (aliased by bit-banding) | 0x2000 0000 | 0x2002 0000 | 128 Kb | 
| Peripherals | 0x4000 0000 | 0x5FFF FFFF | 512 Mb |
| Flash | 0x0800 0000 | 0x0807 FFFF | 512 Kb |

![image](https://user-images.githubusercontent.com/65039828/234352358-77d14d8d-fff4-468e-934a-d594a478757a.png)

From `link.ld`:
```
/* define the memory regions */
MEMORY {
	flash(rx) : ORIGIN = 0x08000000, LENGTH = 512k		
	sram(rwx) : ORIGIN = 0x20000000, LENGTH = 128k
}
```

### 5. GPIOs

The following peripherals will be used in this project, and can be found in Section 5 of the MCU datasheet.

| Peripheral | Bus | Start Address |
| ------- | --- | ------------- |
| GPIOA | AHB1 | 0x4002 0000 |
| GPIOB | AHB1 | 0x4002 0400 |

![image](https://user-images.githubusercontent.com/65039828/234352876-be59c644-599e-4c1d-8aed-e0c7dc799e27.png)

The datasheet also displays the GPIO registers and their configuration bits. For example, the port mode register:

![image](https://user-images.githubusercontent.com/65039828/234358449-29cfd785-8b35-4434-aedc-ef749c37c4b1.png)

From `main.h`:
```
struct gpio {
	volatile uint32_t MODER,
			 OTYPER,
			 OSPEEDR,
			 PUPDR,
			 IDR,
			 ODR,
			 BSRR,
			 LCKR,
			 AFRL,
			 AFRH;
};
#define GPIO(bank) ((struct gpio *) (0x40020000 + 0x400 * (bank)))
```

From `main.c`:
```
enum {
	GPIO_MODE_INPUT,
	GPIO_MODE_OUTPUT,
	GPIO_MODE_AF,
	GPIO_MODE_ANALOG
};

static inline void gpio_set_mode(uint16_t pin, uint8_t mode) {
	struct gpio *gpio = GPIO(PINBANK(pin));
	int n = PINNO(pin);
	// clear current setting
	gpio->MODER &= ~(3U << (n * 2));
	// set new mode
	gpio->MODER |= (mode & 3U) << (n * 2);
}
```

### 6. Interrupt Vector Table (IVT)

The selected MCU includes 16 ARM interrupts, as well as 62 MCU specific interrupts.

![image](https://user-images.githubusercontent.com/65039828/234355733-7bfff272-d56b-490c-b76c-4be047ed9565.png)

From `main.c`:
```
__attribute__((section(".vectors"))) void (*tab[16 + 62])(void) = {_estack, _reset};
```

From `link.ld`:
```
/* set entry point to beginning of the firmware */
ENTRY(_reset);
...
/* set stack pointer to end of SRAM */
_estack = ORIGIN(sram) + LENGTH(sram);
```

### 7. Reset and Clock Control (RCC)

The RCC provides an interface for enabling/disabling GPIOS.

The boundary address can be found in Section 5:

![image](https://user-images.githubusercontent.com/65039828/234360726-96e9848a-cb9e-4d36-bec6-56c5cde4383c.png)

And the register map in Section 6:

![image](https://user-images.githubusercontent.com/65039828/234360846-335b8159-e1df-4c03-b00f-bf99f92efb86.png)
![image](https://user-images.githubusercontent.com/65039828/234360872-94e9d13b-f23a-4632-8615-93cfdb68c0a0.png)
![image](https://user-images.githubusercontent.com/65039828/234360901-a8204a0d-0b08-4d96-b850-564f2c6dc94f.png)


From `main.h`:
```
struct rcc {
	volatile uint32_t CR,
			 PLLCFGR,
			 CFGR,
			 CIR,
			 AHB1RSTR,
			 AHB2RSTR,
			 RESERVED0,
			 RESERVED1,
			 APB1RSTR,
			 APB2RSTR,
			 RESERVED3,
			 RESERVED4,
			 AHB1ENR,
			 AHB2ENR,
			 RESERVED5,
			 RESERVED6,
			 APB1ENR,
			 APB2ENR,
			 RESERVED7,
			 RESERVED8,
			 AHB1LPENR,
			 AHB2LPENR,
			 RESERVED9,
			 RESERVED10,
			 APB1LPENR,
			 APB2LPENR,
			 RESERVED11,
			 RESERVED12,
			 BDCR,
			 CSR,
			 RESERVED13,
			 RESERVED14,
			 SSCGR,
			 PLLI2SCFGR,
			 DCKCFGR;
};
#define RCC ((struct rcc *) 0x40023800)
```

To enable a peripheral:
From `main.c':
```
RCC->AHB1ENR |= peripheral_bit;
```

### 8. UART

| Peripheral | Bus | Start Address |
| ------- | --- | ------------- |
| USART1 | APB2 | 0x4001 1000 |
| USART6 | APB2 | 0x4001 1400 |

![image](https://user-images.githubusercontent.com/65039828/234365783-176be1f6-9f07-445b-8083-b41a8a70590b.png)

Redirecting `printf()` to UART for USB debug output:

From `sys.c':
```
int _write(int fd, char *p, int len) {
  (void) fd, (void) p, (void) len;
  if (fd == 1) uart_write(UART1, p, (size_t) len);
  return -1;
}
```

This will redirect the buffer to UART1 if `fd == 1`.

### 9. Webserver

I Mongoose Web Server, an open-source embedded webserver platform, to locally host a customizable device dashboard. I was able to get this working a few times until unexpected problems occured.

## Problems
- Segger's STLinkReflash Utility
  - After converting the onboard STLINK to JLINK, the tool failed in reverting it back
  - I was unable to read the debug output which I believe was due to the default baud rate changing
  - The changed baud rate prevented me from recieving the IP address of my webserver
- Lead times
  - When I tried to purchase a replacement board of the same model, the lead time was 92 weeks
- Changing development boards
  - The data sheet for the new board was much more confusing and disorganized than the previous
  - There was very little publically availble information online forums or elsewhere
  - Many changes were required including resizing the IVT and completely redoing the Memory Mapping, GPIOs, UART, and RCC. This required me to redo nearly the entire project again.
- MacOS
  - I found that MacOS is not the friendliest environment for embedded development or electronics
  - A Windows VM was required for some purposes, such as using a USB oscilloscope
- 

## What I Learned
- What specifications to look for in a development board before buying
- To review a board's datasheet and the resources avaiable for it BEFORE purchasing
- How to write a GNU linker script
- How the IVT works and triggers the respective interrupt handlers
- The basic code required for startup on an ARM MCU
- How to properly review a datasheet and what sections are important for an Embedded Software Engineer
- How to use many of the largest open source tools available to Embedded Developers
