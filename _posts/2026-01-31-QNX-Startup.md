---
layout: post
title:  "Go through QNX Startup Code for Raspberry Pi 4 (BCM2711)"
subtitle: "Based on SDP 8.0"
header-img: img/post-bg-coffee.jpeg
date:   2026-02-03 08:55:59
author: half cup coffee
catalog: true
tags:	
    - Arm64
    - QNX
---


## Introduction

This document provides an in-depth analysis of the QNX Neutrino RTOS startup code for the Raspberry Pi 4 board, based on the Broadcom BCM2711 SoC. The startup program is responsible for initializing the hardware, setting up the system page, and transferring control to the QNX kernel.

**Key Facts:**
- **Target Platform**: Raspberry Pi 4 Model B, Compute Module 4, Pi 400
- **SoC**: Broadcom BCM2711 (Quad-core Cortex-A72)
- **Architecture**: ARMv8-A (AArch64)
- **QNX Version**: SDP 8.0

### Information Sources

This document is based on:
1. **Direct Code Analysis**: Primary source - actual QNX startup code files
2. **Code Comments**: Inline documentation from BlackBerry/QNX engineers
3. **Raspberry Pi Architecture**: Public documentation about RPi boot process and VideoCore GPU
4. **BCM2711 Documentation**: Broadcom peripheral specifications
5. **ARM Architecture**: Cortex-A72 and GICv2 technical reference manuals

**Note on GPU/VideoCore Boot Sequence**: The detailed boot flow (GPU boots first, loads firmware, then releases ARM cores) is inferred from:
- Mailbox interface code (`mbox.c`) showing ARM-to-GPU communication
- Code comments referencing "VPU-inserted arguments" and "VideoCore memory"
- Memory management code explicitly handling GPU RAM allocation
- Minimal `_start.S` entry point (hardware already initialized by GPU)
- Public Raspberry Pi documentation about bootcode.bin → start.elf → kernel loading

---

## Boot Sequence Overview

**Source**: This sequence is derived from code analysis and Raspberry Pi architecture documentation.

The boot sequence follows this flow:

```
GPU Bootloader (VideoCore)
    ↓
ARM Stub / U-Boot (loads startup)
    ↓
_start.S (Entry Point)
    ↓
cstart (Common ARM64 startup)
    ↓
main() (Board-specific initialization)
    ↓
    ├→ init_board() - GPIO/Pinmux
    ├→ bcm2711_init_raminfo() - Memory Map
    ├→ init_smp() - Multi-core
    ├→ init_intrinfo() - Interrupts (GIC)
    ├→ init_hwinfo() - Hardware Database
    ├→ init_qtime() - System Timer
    ├→ init_cacheattr() - Cache Configuration
    └→ init_system_private() - Load IFS
    ↓
QNX Kernel (procnto-smp-instr)
```

---

## 1. Entry Point (_start.S)

**File**: `src/hardware/startup/boards/bcm2711/_start.S`

### Purpose
The assembly entry point that receives control from the bootloader and preserves critical boot parameters before jumping to the C runtime initialization.

### Code Analysis

```asm
    .text
    .align 2

    .global _start
_start:
    /* NOTE:
        Do NOT modify registers X0-X3 before returning on the bootstrap
        processor.
        These registers may contain information provided by the IPL and
        cstart will save them in the boot_regs variable for later perusal
        by other portions of startup.
    */

    b       cstart
```

### Key Points

1. **Register Preservation**: X0-X3 contain boot parameters:
   - **X0**: Device Tree Blob (DTB) address or machine type
   - **X1**: Reserved (typically 0)
   - **X2**: ATAGS or DTB address (legacy)
   - **X3**: Reserved

2. **Minimal Setup**: Unlike many embedded systems, this entry point is intentionally minimal because:
   - The GPU's bootloader has already initialized DRAM *(inferred from minimal DRAM setup code)*
   - ARM stub code has set up basic CPU state (EL1/EL2) *(Raspberry Pi boot architecture)*
   - No need for early MMU or cache setup at this stage *(evident from code structure)*

3. **Direct Jump**: Transfers control immediately to `cstart`, which is part of the common ARM64 startup library.

---

## 2. Main Initialization (main.c)

**File**: `src/hardware/startup/boards/bcm2711/main.c`

### Command Line Processing

The Raspberry Pi 4 uses a unique boot architecture where the VideoCore GPU is the primary bootloader. It can pass command-line arguments through the mailbox interface:

```c
void cpu_tweak_cmdline(struct bootargs_entry *bap, const char *name) {
    const static char *next_arg;

    if (!next_arg) { // only do this once
        char *str = mbox_get_cmdline();

        if (str == NULL) {
            return;
        }

        // skip vpu-inserted arguments up to dash or double-space
        do {
            next_arg = find_next_arg(str, &str, 0);
            if (next_arg[0] == '\0') break;
            if (next_arg[0] == '-') break;
            if ((next_arg[0] == ' ') && (next_arg[1] == ' ')) break;
        } while (1);

        for (;;) {
            if (next_arg[0] == '\0') break;
            if ((next_arg[0] == '-') && (next_arg[1] == '-')) break;
            bootstrap_arg_adjust(bap, NULL, next_arg);
            next_arg = find_next_arg(str, &str, 1);
        }
    }
}
```

**Analysis:**
- The GPU firmware can insert boot parameters in `config.txt` or `cmdline.txt` *(Raspberry Pi boot configuration)*
- Arguments before `--` are for the GPU, after `--` are for the OS *(from code comment: "skip vpu-inserted arguments")*
- This function filters GPU-specific args before passing to QNX *(evident from parsing logic)*

### Debug Device Configuration

QNX startup supports multiple debug serial ports:

```c
const struct debug_device debug_devices[] = {
    {   .name = "miniuart",
        .defaults[DEBUG_DEV_CONSOLE] = "fe215000^0.115200",  /* uart1 */
        .init = init_miniuart,
        .put = put_miniuart,
        .callouts[DEBUG_DISPLAY_CHAR] = &display_char_miniuart,
        .callouts[DEBUG_POLL_KEY] = &poll_key_miniuart,
        .callouts[DEBUG_BREAK_DETECT] = &break_detect_miniuart,
    },
    {   .name = "pl011-3",
        .defaults[DEBUG_DEV_CONSOLE] = "fe201600^0.115200.48000000", /* uart 3*/
        .init = init_pl011,
        .put = put_pl011,
        .callouts[DEBUG_DISPLAY_CHAR] = &display_char_pl011,
        .callouts[DEBUG_POLL_KEY] = &poll_key_pl011,
        .callouts[DEBUG_BREAK_DETECT] = &break_detect_pl011,
    },
};
```

**Two UART Options:**
1. **Mini UART (UART1)**: Simpler, tied to VPU core clock (variable frequency)
2. **PL011 (UART3)**: Full-featured ARM PrimeCell UART, fixed 48MHz clock

### Main Function Flow

```c
int main(const int argc, char ** const argv)
{
    int opt, options = 0;
    int wdt_timeout = BCM2711_WDT_USE_DEFAULT;
    uint32_t board_rev;

    // 1. Add callout array (reboot mechanism)
    const struct callout_slot callouts[] = {
        { CALLOUT_SLOT( reboot, _bcm2711 ) },
    };
    add_callout_array(callouts, sizeof callouts);

    // 2. Initialize board (GPIO/pinmux)
    init_board();

    // 3. Select debug device
    select_debug(debug_devices, sizeof(debug_devices));

    // 4. Parse command line options
    while ((opt = getopt(argc, argv, COMMON_OPTIONS_STRING "W:")) != -1) {
        switch (opt) {
            case 'W':
                options |= BCM2711_WDT_ENABLE;
                if (optarg) {
                    wdt_timeout = (int) strtol(optarg, &optarg, 0);
                }
                break;
            default:
                handle_common_option(opt);
                break;
        }
    }

    // 5. Enable watchdog if requested
    if (options & BCM2711_WDT_ENABLE) {
        bcm2711_wdt_enable(wdt_timeout);
    }

    // 6. Re-initialize debug device if "-D" option specified
    select_debug(debug_devices, sizeof(debug_devices));

    // 7. Get CPU frequency from VideoCore
    cpu_freq = mbox_get_clock_rate(MBOX_CLK_ARM);
    cycles_freq = cpu_freq;
    timer_freq = 0;

    // 8. Setup RAM
    bcm2711_init_raminfo();
    alloc_ram(shdr->ram_paddr, shdr->ram_size, 1);

    // 9. Enable Hypervisor if requested
    hypervisor_init(0);

    // 10. Initialize SMP
    init_smp();

    // 11. Debug info (if verbose)
    if (debug_flag > 3) {
        kprintf("%u CPUs @ %uMHz\n", lsp.syspage.p->num_cpu, cpu_freq / 1000000);
        kprintf("Temps %d\n", mbox_get_temperature(0));
        // ... more debug output
    }

    // 12. Initialize MMU if virtual mode
    if (shdr->flags1 & STARTUP_HDR_FLAGS1_VIRTUAL) {
        init_mmu();
    }

    // 13. Initialize interrupt system
    init_intrinfo();

    // 14. Initialize system timer
    init_qtime();

    // 15. Initialize cache attributes
    init_cacheattr();

    // 16. Initialize CPU info
    init_cpuinfo();

    // 17. Initialize hardware database
    init_hwinfo();

    // 18. Add board identification strings
    board_rev = mbox_get_board_revision();
    add_typed_string(_CS_MACHINE, get_board_name(board_rev));
    add_typed_string(_CS_HW_PROVIDER, get_mfr_name(board_rev));
    add_typed_string(_CS_ARCHITECTURE, get_soc_name(board_rev));
    add_typed_string(_CS_HW_SERIAL, get_board_serial());

    // 19. Load IFS and transfer control
    init_system_private();

    print_syspage();
    return 0;
}
```

### Command Line Options

QNX startup supports standard options plus BCM2711-specific ones:

- **Common Options** (`COMMON_OPTIONS_STRING`):
  - `-v`: Verbose mode
  - `-D`: Debug device specification
  - `-K`: Kernel options
  - `-P`: Processor options
  - And many more...

- **BCM2711-Specific**:
  - `-W[timeout]`: Enable watchdog with optional timeout in milliseconds

---

## 3. Board Configuration System

**Files**: 
- `src/hardware/startup/boards/bcm2711/bcm2711_init_board.c`
- `src/hardware/startup/boards/bcm2711/rpi4/rpi4_board_config.h`

### GPIO/Pinmux Architecture

The BCM2711 has 54 GPIO pins, each capable of 8 different functions:

```c
#define FUNC_IP     0  // Input
#define FUNC_OP     1  // Output
#define FUNC_A0     4  // Alternate Function 0
#define FUNC_A1     5  // Alternate Function 1
#define FUNC_A2     6  // Alternate Function 2
#define FUNC_A3     7  // Alternate Function 3
#define FUNC_A4     3  // Alternate Function 4
#define FUNC_A5     2  // Alternate Function 5
```

### Pin Configuration Structure

```c
typedef struct _bcm2711_pinmux {
    uint8_t pin_num;    // GPIO pin number (0-53)
    uint8_t fsel;       // Function select
    uint8_t pull;       // Pull-up/pull-down/none
    uint8_t gpio_lvl;   // Output level (if output)
    uint8_t is_last;    // Array terminator
} bcm2711_pinmux_t;
```

### Board Configuration Example

```c
const bcm2711_config_t rpi_board_config[] = {
    {   // UART Configuration
        .pinmux = (bcm2711_pinmux_t[])
        {
            { .pin_num = 14, .fsel = FUNC_A5, .pull = PULL_UP, .gpio_lvl = 0 }, /* TXD0 */
            { .pin_num = 15, .fsel = FUNC_A5, .pull = PULL_UP, .gpio_lvl = 0 }, /* RXD0 */
            { .pin_num =  4, .fsel = FUNC_A4, .pull = PULL_UP, .gpio_lvl = 0 }, /* TXD3 */
            { .pin_num =  5, .fsel = FUNC_A4, .pull = PULL_UP, .gpio_lvl = 0 }, /* RXD3 */
            { .is_last = 1 }
        },
    },
    {   // SPI Configuration
        .pinmux = (bcm2711_pinmux_t[])
        {
            { .pin_num =  7, .fsel = FUNC_A0, .pull = PULL_UP,   .gpio_lvl = 0 }, /* SPI0_CE1_N */
            { .pin_num =  8, .fsel = FUNC_A0, .pull = PULL_UP,   .gpio_lvl = 0 }, /* SPI0_CE0_N */
            { .pin_num =  9, .fsel = FUNC_A0, .pull = PULL_NONE, .gpio_lvl = 0 }, /* SPI0_MISO */
            { .pin_num = 10, .fsel = FUNC_A0, .pull = PULL_NONE, .gpio_lvl = 0 }, /* SPI0_MOSI */
            { .pin_num = 11, .fsel = FUNC_A0, .pull = PULL_NONE, .gpio_lvl = 0 }, /* SPI0_SCLK */
            { .is_last = 1 }
        },
    },
    {   // I2C Configuration
        .pinmux = (bcm2711_pinmux_t[])
        {
            { .pin_num = 0, .fsel = FUNC_A0, .pull = PULL_UP, .gpio_lvl = 0 }, /* SDA0 */
            { .pin_num = 1, .fsel = FUNC_A0, .pull = PULL_UP, .gpio_lvl = 0 }, /* SCL0 */
            { .pin_num = 2, .fsel = FUNC_A0, .pull = PULL_UP, .gpio_lvl = 0 }, /* SDA1 */
            { .pin_num = 3, .fsel = FUNC_A0, .pull = PULL_UP, .gpio_lvl = 0 }, /* SCL1 */
            { .is_last = 1 }
        },
    },
};
```

### GPIO Register Functions

```c
static void bcm2711_set_gpio_fsel(const uint32_t gpio, const uint32_t fsel)
{
    const uint32_t reg = (gpio / 10) * 4;  // GPFSEL0-5 (10 pins per register)
    const uint32_t sel = gpio % 10;
    uint32_t val;

    if ((gpio < GPIO_MIN) || (gpio > GPIO_MAX)) {
        return;
    }

    val = in32(BCM2711_GPIO_BASE + reg);
    val &= ~(0x7 << (3 * sel));  // Clear 3-bit field
    val |=  (fsel & 0x7) << (3 * sel);  // Set new function
    out32(BCM2711_GPIO_BASE + reg, val);
}
```

**Register Layout**: Each GPFSEL register controls 10 pins with 3 bits per pin:
- **GPFSEL0** (offset 0x00): GPIO 0-9
- **GPFSEL1** (offset 0x04): GPIO 10-19
- **GPFSEL2** (offset 0x08): GPIO 20-29
- ...and so on

### Pull-Up/Pull-Down Configuration (BCM2711-Specific)

The BCM2711 uses a different pull-up/down mechanism than earlier BCM283x chips:

```c
static void bcm2711_gpio_set_pull(const uint32_t gpio, const uint32_t type)
{
    const uint32_t reg = GPPUPPDN0 + (gpio >> 4) * 4;  // 16 pins per register
    const uint32_t shift = (gpio & 0xf) << 1;  // 2 bits per pin
    uint32_t val;

    if ((gpio < GPIO_MIN) || (gpio > GPIO_MAX)) {
        return;
    }

    if ((type < 0) || (type > 2)) {
        return;
    }

    val = in32(BCM2711_GPIO_BASE + reg);
    val &= ~(3 << shift);  // Clear 2-bit field
    val |= (type << shift);  // Set new pull type
    out32(BCM2711_GPIO_BASE + reg, val);
}
```

**Pull Types**:
- `0`: No pull
- `1`: Pull-up
- `2`: Pull-down

### Board Revision Decoding

```c
const char *get_board_name(const uint32_t board_rev)
{
    const char * const types[] = {
        [0]  = "RaspberryPi1A",
        [1]  = "RaspberryPi1B",
        [2]  = "RaspberryPi1A+",
        [3]  = "RaspberryPi1B+",
        [4]  = "RaspberryPi2B",
        [5]  = "RaspberryPiAlpha",
        [6]  = "RaspberryPiCM1",
        [8]  = "RaspberryPi3B",
        [9]  = "RaspberryPi0",
        [10] = "RaspberryPiCM3",
        [12] = "RaspberryPi0W",
        [13] = "RaspberryPi3B+",
        [14] = "RaspberryPi3A+",
        [16] = "RaspberryPiCM3+",
        [17] = "RaspberryPi4B",      // ← Our target
        [18] = "RaspberryPi02W",
        [19] = "RaspberryPi400",
        [20] = "RaspberryPiCM4",
    };

    const uint32_t idx = (board_rev >> BOARD_REV_TYPE_SHIFT) & BOARD_REV_TYPE_MASK;
    if (idx < NUM_ELTS(types)) {
        return types[idx];
    }
    return "Unknown";
}
```

**Board Revision Format** (New-style, 32-bit):
```
Bit 31-25: Overvoltage, OTP flags
Bit 24:    Unused
Bit 23:    New-style revision flag (always 1)
Bit 22-20: Memory size (0=256MB, 1=512MB, 2=1GB, 3=2GB, 4=4GB, 5=8GB)
Bit 19-16: Manufacturer (0=Sony UK, 1=Egoman, 2=Embest, 3=Sony Japan, etc.)
Bit 15-12: Processor (0=BCM2835, 1=BCM2836, 2=BCM2837, 3=BCM2711)
Bit 11-4:  Board type (17=RPi4B, 19=RPi400, 20=CM4)
Bit 3-0:   Revision number
```

---

## 4. Memory Subsystem

**File**: `src/hardware/startup/boards/bcm2711/bcm2711_init_raminfo.c`

### BCM2711 Memory Architecture

The BCM2711 has a complex memory map that differs from earlier Raspberry Pi models:

```
┌─────────────────────────────────┐ 0x4_0000_0000 (16GB)
│         Not Used                │
├─────────────────────────────────┤ 0x2_0000_0000 (8GB)
│     SDRAM (ARM) - High          │ ← For 8GB models only
│     (4GB region)                │
├─────────────────────────────────┤ 0x1_0000_0000 (4GB)
│   ARM Local Peripherals         │
│   (Local Timer, Mailboxes)      │
├─────────────────────────────────┤ 0x0_FF80_0000
│    Main Peripherals             │
│    (GPIO, UART, SPI, I2C, etc)  │
├─────────────────────────────────┤ 0x0_FC00_0000 (3840MB)
│     SDRAM (ARM) - Low           │
│     (before VideoCore split)    │
├─────────────────────────────────┤
│   VideoCore Memory (VC RAM)     │ ← GPU frame buffer, etc.
│   (split point varies)          │
├─────────────────────────────────┤
│     SDRAM (ARM) - Low           │ ← Main system RAM
│                                 │
├─────────────────────────────────┤ 0x0_0000_1000 (4KB)
│   Stub (Reserved, 4KB)          │ ← Startup stub
├─────────────────────────────────┤ 0x0_0000_0000
```

### Memory Initialization Code

```c
void bcm2711_init_raminfo(void) {
    const uint32_t vcram_size = mbox_get_vc_memory();
    const paddr_t vcram_start = GIG(1) - vcram_size;
    const uint32_t rev = mbox_get_board_revision();
    uint64_t ramsize = 0;

    /* Avoid touching the stub in startup */
    avoid_ram(BCM2711_DRAM_BASE, BCM2711_DRAM_STUB_SIZE);
    
    /* Avoid touching the VideoCore memory in startup */
    avoid_ram(vcram_start, vcram_size);

    ramsize = MEG(get_memory_size(rev));
    if (ramsize == 0) {
        ramsize = MEG(256);
        kprintf("Warning: Failed to detect RAM size. Defaulting to %u MB.\n",
            ramsize);
    }

    if (ramsize >= GIG(4)) {
        add_ram(BCM2711_DRAM_BASE, GIG(4) - MEG(64)); // periphs are mapped to 0xfc000000
        if (ramsize >= GIG(8)) {
            add_ram(BCM2711_DRAM_BASE_1, GIG(4));    // for rpi4 8GB target
        }
    } else {
        add_ram(BCM2711_DRAM_BASE, ramsize); // 256M, 512M, 1G, 2G
    }

    /* Reserve VideoCore memory */
    alloc_ram(vcram_start, vcram_size, 0);
    as_add_containing(vcram_start, vcram_start + vcram_size - 1, AS_ATTR_RAM, "vcram", "ram");

    /* Add a below1G tag for legacy peripherals */
    if (ramsize > GIG(1)) {
        as_add_containing(BCM2711_DRAM_BASE,  vcram_start - 1, AS_ATTR_RAM, "below1G", "ram");
    }

    if (debug_flag > 3) {
        kprintf("vcram start-size: %P-%P, memory size: %l\n", vcram_start, vcram_size, ramsize);
    }
}
```

### Key Memory Concepts

1. **VideoCore Memory Split**:
   - The GPU (VideoCore VI) shares DRAM with the ARM cores
   - Split configured in `config.txt` (e.g., `gpu_mem=128`)
   - VideoCore memory is at the top of the first 1GB
   - Typical splits: 64MB, 128MB, 256MB

2. **4GB Peripherals Hole**:
   - BCM2711 peripherals are mapped at 0xFC000000-0xFFFFFFFF
   - This creates a 64MB hole unavailable for RAM
   - For ≥4GB models, RAM is split: [0, 3840MB) and [4GB, 8GB)

3. **8GB Support**:
   - 8GB models use both low (0-4GB) and high (4GB-8GB) regions
   - Requires kernel compiled with `PADDR_BITS=40` or `LPAE` support
   - High memory added separately with `add_ram(BCM2711_DRAM_BASE_1, GIG(4))`

4. **Address Space Tags**:
   - **"vcram"**: Reserved for GPU
   - **"below1G"**: RAM below 1GB barrier (for DMA-limited devices)
   - These tags allow drivers to allocate from specific regions

### Example Memory Configurations

**1GB Model:**
```
0x00000000 - 0x00000FFF: Stub (avoided)
0x00001000 - 0x3B3FFFFF: ARM RAM (~944MB)
0x3B400000 - 0x3FFFFFFF: VideoCore RAM (76MB, example)
0xFC000000 - 0xFFFFFFFF: Peripherals
```

**4GB Model:**
```
0x00000000 - 0x00000FFF: Stub (avoided)
0x00001000 - 0x3B3FFFFF: ARM RAM (~944MB)
0x3B400000 - 0x3FFFFFFF: VideoCore RAM (76MB)
0xFC000000 - 0xFFFFFFFF: Peripherals
```

**8GB Model:**
```
0x0_00000000 - 0x0_00000FFF: Stub (avoided)
0x0_00001000 - 0x0_3B3FFFFF: ARM RAM (~944MB)
0x0_3B400000 - 0x0_3FFFFFFF: VideoCore RAM (76MB)
0x0_FC000000 - 0x0_FFFFFFFF: Peripherals
0x1_00000000 - 0x1_FFFFFFFF: ARM RAM (4GB high memory)
```

---

## 5. Interrupt Controller Setup

**File**: `src/hardware/startup/boards/bcm2711/init_intrinfo.c`

### ARM Generic Interrupt Controller (GIC)

The BCM2711 includes an ARM GICv2 controller with the following configuration:

```c
void init_intrinfo(void)
{
    /*
     * Initialise GIC
     */
#ifndef BCM2711_GIC_V3
#define GICD_PADDR      0xff841000  // Distributor
#define GICC_PADDR      0xff842000  // CPU Interface
    gic_v2_init(GICD_PADDR, GICC_PADDR);
#else
#define GIC_GICD_BASE   0x4c0040000     // CPU distributor BASE address
#define GIC_GICR_BASE   0x4c0041000     // CPU redistributor BASE address
#define GIC_GICC_BASE   0x4c0042000     // CPU interface, CBAR register value
    gic_v3_init(GIC_GICD_BASE, GIC_GICR_BASE, GIC_GICC_BASE, 0, 1);
#endif

    // ... MSI interrupt setup follows
}
```

### GIC Architecture Overview

```
┌──────────────────────────────────────┐
│         Interrupt Sources            │
│  (Peripherals, Timers, etc.)         │
└──────────────┬───────────────────────┘
               │
               ↓
    ┌──────────────────────┐
    │   GIC Distributor    │  ← Routes interrupts to CPU interfaces
    │   (GICD @ 0xff841000)│     Configures priority, edge/level, etc.
    └──────────┬───────────┘
               │
        ┌──────┴──────┬──────┬──────┐
        ↓             ↓      ↓      ↓
    ┌────────┐   ┌────────┐  ...  ┌────────┐
    │ CPU IF │   │ CPU IF │        │ CPU IF │
    │ Core 0 │   │ Core 1 │        │ Core 3 │
    └────────┘   └────────┘        └────────┘
```

### GIC Interrupt Types

1. **SGI (Software Generated Interrupts)**: 0-15
   - Used for inter-processor communication (IPI)
   - Core-to-core signaling for SMP

2. **PPI (Private Peripheral Interrupts)**: 16-31
   - Per-core private interrupts
   - Examples: ARM generic timer, PMU

3. **SPI (Shared Peripheral Interrupts)**: 32-1019
   - Shared among all cores
   - BCM2711 peripherals (UART, GPIO, SPI, etc.)

### MSI (Message Signaled Interrupts) for PCIe

The BCM2711 includes a PCIe controller that supports MSI:

```c
static const paddr_t bcm2711_msi_base = BCM2711_PCIE_BASE + 0x4000;

void init_intrinfo(void)
{
    const struct startup_intrinfo interrupts[] =
    {
        /* MSI interrupts (32) starting at 0x200 */
        {   .vector_base = 0x200,           // MSI vector base (MSI 0x200 - 0x21F)
            .num_vectors = 32,              // 32 MSI vectors
            .cascade_vector = BCM2711_PCIE_MSI, // Parent SPI interrupt
            .cpu_intr_base = 0,             // CPU-relative base (0-31)
            .cpu_intr_stride = 0,
            .flags = INTR_FLAG_MSI,
            .id = { .genflags = INTR_GENFLAG_NOGLITCH, .size = 0, 
                    .rtn = &interrupt_id_bcm2711_msi },
            .eoi = { .genflags = INTR_GENFLAG_LOAD_INTRMASK, .size = 0, 
                     .rtn = &interrupt_eoi_bcm2711_msi },
            .mask = &interrupt_mask_bcm2711_msi,
            .unmask = &interrupt_unmask_bcm2711_msi,
            .config = 0,
            .patch_data = (paddr_t*)&bcm2711_msi_base,
        },
    };

    add_interrupt_array(interrupts, sizeof interrupts);
}
```

### MSI Interrupt Callout (Assembly)

**File**: `src/hardware/startup/boards/bcm2711/callout_interrupt_bcm2711_msi.S`

```asm
CALLOUT_START(interrupt_id_bcm2711_msi, 0, patch_intr)

    /* Get the MSI base address (patched) */
    mov     x7, #0x0000
    movk    x7, #0x0000, lsl #16
    movk    x7, #0x0000, lsl #32
    movk    x7, #0x0000, lsl #48

    mov     x19, #-1

    /*
     * Read Interrupt Mask and Status
     */
    ldr     w0, [x7, #BCM2711_MSI_MASK_STATUS]
    ldr     w1, [x7, #BCM2711_MSI_STATUS]
    
    /* clear any masked bits from the status register */
    bics    w1, w1, w0
    
    /* if no more bits set we're done */
    cbz     w1, 0f

    /* calculate the interrupt number to be returned */
    rbit    w2, w1      // Reverse bits
    clz     w1, w2      // Count leading zeros

    /* if ID not between 0 and 31, there's nothing to do */
    cmp     w1, #BCM2711_MAX_MSI
    bge     0f

    mov     w19, w1
    /* create the bit mask to clear the status register */
    mov     w2, #1
    lsl     w1, w2, w19
    str     w1, [x7, #BCM2711_MSI_MASK_SET]
    str     w1, [x7, #BCM2711_MSI_INTR_CLR]

0:

CALLOUT_END(interrupt_id_bcm2711_msi)
```

**How it Works:**
1. Read MSI status register (32 bits, one per MSI vector)
2. Mask out any already-masked interrupts
3. Find the first set bit (lowest priority-first)
4. Return the bit position as the interrupt ID
5. Mask and clear that MSI source

### BCM2711 Interrupt Map (Selected)

| Vector | Source | Description |
|--------|--------|-------------|
| 32 | TIMER0 | ARM Timer (system timer) |
| 33 | TIMER1 | ARM Timer |
| 64 | GPIO0 | GPIO bank 0 (pins 0-27) |
| 65 | GPIO1 | GPIO bank 1 (pins 28-45) |
| 66 | GPIO2 | GPIO bank 2 (pins 46-53) |
| 79 | I2C0 | I2C master 0 |
| 80 | SPI0 | SPI master 0 |
| 85 | I2C1 | I2C master 1 |
| 121 | UART0 | PL011 UART 0 |
| 125 | UART3 | PL011 UART 3 |
| 153 | ETHERNET | GENET Ethernet (queue 0) |
| 154 | ETHERNET | GENET Ethernet (queue 16) |
| 158 | PCIE_MSI | PCIe MSI (cascades to 32 MSI vectors) |

---

## 6. Hardware Information Database

**File**: `src/hardware/startup/boards/bcm2711/rpi4/init_hwinfo.c`

### Purpose

The Hardware Information (hwinfo) database is a QNX-specific feature that stores comprehensive hardware configuration in the system page. Device drivers query this database at runtime instead of using hardcoded addresses.

### Structure

```c
void init_hwinfo(void)
{
    const unsigned hwi_bus_internal = 0;
    
    /* Add ENET (Ethernet) */
    {
        unsigned i, hwi_off;
        uint8_t mac[6];
        hwiattr_enet_t attr = HWIATTR_ENET_T_INITIALIZER;
        hwiattr_common_t common_attr = HWIATTR_COMMON_INITIALIZER;
        HWIATTR_ENET_SET_NUM_IRQ(&attr, 1);

        uint32_t irqs[] = {
            BCM2711_ENET_IRQA,  // Queue 0 interrupt
            BCM2711_ENET_IRQB   // Queue 16 interrupt
        };

        HWIATTR_SET_NUM_IRQ(&common_attr, NUM_ELTS(irqs));

        /* Create GENET device */
        HWIATTR_ENET_SET_LOCATION(&attr, BCM2711_ENET_BASE,
                BCM2711_ENET_SIZE, 0, hwi_find_as(BCM2711_ENET_BASE, 1));
        hwi_off = hwidev_add_enet(BCM2711_HWI_GENET, &attr, hwi_bus_internal);
        hwitag_add_common(hwi_off, &attr);
        ASSERT(hwi_find_unit(hwi_off) == 0);

        /* Add IRQ numbers */
        for(i = 0; i < NUM_ELTS(irqs); i++) {
            hwitag_set_ivec(hwi_off, i, irqs[i]);
        }

        /* Get MAC address from VideoCore mailbox */
        mbox_get_board_mac_address(mac);
        hwitag_add_nicaddr(hwi_off, mac, sizeof(mac));
    }

    /* add the WATCHDOG device */
    {
        unsigned hwi_off;
        hwiattr_timer_t attr = HWIATTR_TIMER_T_INITIALIZER;
        const struct hwi_inputclk clksrc_kick = {.clk = 500, .div = 1};
        
        HWIATTR_TIMER_SET_NUM_CLK(&attr, 1);
        HWIATTR_TIMER_SET_LOCATION(&attr, BCM2711_PM_BASE, BCM2711_PM_SIZE, 0, 
                                   hwi_find_as(BCM2711_PM_BASE, 1));
        hwi_off = hwidev_add_timer("wdog", &attr, HWI_NULL_OFF);
        ASSERT(hwi_off != HWI_NULL_OFF);
        hwitag_set_inputclk(hwi_off, 0, (struct hwi_inputclk *)&clksrc_kick);

        hwi_off = hwidev_add("wdt,options", 0, HWI_NULL_OFF);
        hwitag_add_regname(hwi_off, "write_width", 32);
        hwitag_add_regname(hwi_off, "enable_width", 32);
    }
}
```

### HWInfo Database Layout

The hwinfo database is a tree structure:

```
System Bus (hwi_bus_internal)
├── Ethernet (GENET)
│   ├── Location: 0xFD580000, size 0x10000
│   ├── IRQ[0]: 153 (queue 0)
│   ├── IRQ[1]: 154 (queue 16)
│   └── MAC Address: xx:xx:xx:xx:xx:xx
├── Watchdog Timer
│   ├── Location: 0xFE100000, size 0x1000
│   ├── Clock: 500 Hz
│   └── Options: write_width=32, enable_width=32
└── [Other devices added by common code]
```

### Querying HWInfo at Runtime

Device drivers use the Hardware Abstraction Layer (HAL) to query hwinfo:

```c
// Example: Ethernet driver queries its configuration
#include <hw/hwinfo.h>

void enet_driver_init(void) {
    unsigned hwi_off = hwi_find_device("genet", 0);  // Find first GENET
    
    if (hwi_off != HWI_NULL_OFF) {
        uint64_t base;
        unsigned size;
        unsigned irq;
        uint8_t mac[6];
        
        // Get base address and size
        hwitag_find_ivaddr(hwi_off, NULL, &base, &size);
        
        // Get interrupt vector
        hwitag_find_ivec(hwi_off, 0, &irq);
        
        // Get MAC address
        hwitag_find_nicaddr(hwi_off, mac, sizeof(mac));
        
        kprintf("GENET at 0x%llx, IRQ %d, MAC %02x:%02x:%02x:%02x:%02x:%02x\n",
                base, irq, mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);
    }
}
```

---

## 7. Callout Mechanism

### What Are Callouts?

Callouts are small, position-independent assembly routines embedded in the system page that provide low-level hardware access. They're used by the kernel and drivers without requiring full context switches.

**Benefits:**
- Fast execution (no context switch)
- Small footprint
- Can run with minimal kernel state
- Position-independent (can be relocated)

### Types of Callouts

1. **Debug Callouts**: Character I/O for kernel debugging
2. **Reboot Callouts**: System restart
3. **Timer Callouts**: System timer access
4. **Interrupt Callouts**: Interrupt controller operations
5. **Cache Callouts**: Cache maintenance

### Debug Callouts - Mini UART

**File**: `src/hardware/startup/boards/bcm2711/callout_debug_miniuart.S`

```asm
/*
 * Patch callouts
 * The patcher replaces the mov/movk sequence with the actual register address
 */
patch_debug:
    sub     sp, sp, #16
    stp     x19, x30, [sp]

    add     x19, x0, x2                 // address of callout routine

    // Map registers (convert physical to virtual)
    mov     x0, #0x100
    ldr     x1, [x4]                    // Get physical address from patch_data
    bl      callout_io_map              // Returns virtual address in x0

    // Patch the mov/movk instructions with virtual address
    CALLOUT_PATCH   x19, w6, w7

    ldp     x19, x30, [sp]
    add     sp, sp, #16
    ret

/*
 * Display a character on Mini UART
 */
CALLOUT_START(display_char_miniuart, 0, patch_debug)
    mov     x7, #0xabcd                 // These instructions are patched
    movk    x7, #0xabcd, lsl #16        // to contain the actual
    movk    x7, #0xabcd, lsl #32        // virtual address of
    movk    x7, #0xabcd, lsl #48        // the UART registers

    // Wait for transmitter ready
0:
    ldr     w2, [x7, #AUX_MU_LSR_REG]   // Read Line Status Register
    tbz     w2, #5, 0b                  // Test bit 5 (TX ready)

    // Send the character
    and     w2, w1, #0xff
    str     w2, [x7, #AUX_MU_IO_REG]

    // Wait for transmission complete
1:
    ldr     w2, [x7, #AUX_MU_LSR_REG]
    tbz     w2, #6, 1b                  // Test bit 6 (TX complete)

    ret
CALLOUT_END(display_char_miniuart)
```

### Callout Patching Process

The callout patching is a clever technique that allows the same callout code to work at different addresses:

1. **Startup Phase**:
   - Callout code is written with placeholder addresses (0xabcd...)
   - Physical hardware address is known

2. **Patching Phase**:
   - `callout_io_map()` creates a virtual mapping for the hardware
   - The patcher rewrites the `mov/movk` instructions with the actual virtual address

3. **Runtime Phase**:
   - Callout executes with the correct virtual address
   - No overhead - direct register access

### Reboot Callout

**File**: `src/hardware/startup/boards/bcm2711/callout_reboot_bcm2711.S`

```asm
#define RSTC    0x1c      // Reset Control register
#define WDOG    0x24      // Watchdog register
#define PASSWORD        0x5a000000  // Required password for PM registers
#define RESET_CMD       0x20        // Full reset command

CALLOUT_START(reboot_bcm2711, rw_size_reboot, patch_reboot)
    mov     w7, #0xabcd         // offset of rw data (patched)
    add     x7, x7, x0          // address of rwdata
    ldr     x5, [x7]            // register base (virtual address)

    /*
     * Set up the watchdog timer to something really small
     * and let it trigger the reboot
     */
    mov     w1, #PASSWORD

    mov     w0, #10             // 10 ticks (~150 microseconds)
    orr     w0, w0, w1
    str     w0, [x5, #WDOG]

    ldr     w0, [x5, #RSTC]
    bic     w0, w0, #0x30       // Clear reset type
    orr     w0, w0, w1          // Add password
    orr     w0, w0, #RESET_CMD  // Set full reset
    str     w0, [x5, #RSTC]

0:  b       0b                  // Wait for reboot (never returns)
CALLOUT_END(reboot_bcm2711)
```

**How BCM2711 Reboot Works:**
1. The PM (Power Management) watchdog is used for reboot
2. Set watchdog to very short timeout (10 ticks)
3. Configure RSTC for "full reset" mode
4. Watchdog times out → hardware reset triggered
5. GPU bootloader takes over from scratch

---

## 8. SMP (Multi-Core) Support

**File**: `src/hardware/startup/boards/bcm2711/board_smp.c`

### Raspberry Pi 4 Multi-Core Architecture

The BCM2711 has 4 Cortex-A72 cores:
- All cores start in a powered-off state except core 0
- Core 0 (bootstrap processor) initializes the system
- Secondary cores (1-3) are started by writing to spin table addresses

### SMP Initialization

```c
unsigned board_smp_num_cpu(void) {
    return 4;  // BCM2711 has 4 cores
}

void board_smp_init(struct smp_entry *smp, const unsigned num_cpus)
{
#ifndef BCM2711_GIC_V3
    smp->send_ipi = (void *)&sendipi_gic_v2;
#else
    smp->send_ipi = (void *)&sendipi_gic_v3_sr;
#endif

    /* Need to turn on the IPI interrupt for mailbox 0 on all cores */
    uint32_t cpu;
    for (cpu = 0; cpu < num_cpus; ++cpu) {
        const uint32_t mbic_addr = 0x40000050 + 0x4 * cpu;
        uint32_t mbic = in32(mbic_addr);
        mbic |= 0x1;  // Enable mailbox 0 interrupt
        out32(mbic_addr, mbic);
    }
}
```

### Starting Secondary Cores

```c
static void wake_aux_cpu(void)
{
    // Same assembly works in 32 and 64 bit
    __asm__ __volatile__(
        "dsb sy\n"      // Data Synchronization Barrier
        "sev\n"         // Send Event (wakes cores from WFE)
        : : : "memory"
    );
}

int board_smp_start(unsigned cpu, void (*start)(void))
{
    typedef void (*start_t)(void);
    volatile start_t *cpu_start = (start_t *)(0xd8L + cpu * 8);
    
    *cpu_start = start;  // Write start address to spin table
    wake_aux_cpu();      // Wake the core
    
    return 1;
}
```

### Spin Table Mechanism

The BCM2711 uses a spin table for SMP startup:

```
Address         Purpose
---------       -------
0x000000D8      CPU 1 start address
0x000000E0      CPU 2 start address
0x000000E8      CPU 3 start address
```

**Boot Sequence:**
1. GPU firmware places secondary cores in WFE (Wait For Event) loop
2. Each core polls its spin table address
3. Startup writes function pointer to spin table
4. Startup executes SEV (Send Event) instruction
5. Secondary core wakes up, reads function pointer, jumps to it

### Inter-Processor Interrupts (IPI)

IPIs are used for:
- TLB shootdown (invalidate TLB on other cores)
- Scheduler preemption
- System calls that affect all cores

```c
// IPI mechanism (simplified)
void sendipi_gic_v2(struct syspage_entry *syspage, unsigned cpu, int cmd, void *arg)
{
    // GIC uses SGI (Software Generated Interrupt) for IPI
    // Write to GICD_SGIR register:
    // - Target CPU in bits [16:23]
    // - SGI number (0-15) in bits [0:3]
    
    const unsigned sgi_num = 0;  // Use SGI 0 for IPI
    const uint32_t gicd_sgir = GICD_BASE + 0xF00;
    
    // Send SGI to specific CPU
    out32(gicd_sgir, (1 << (16 + cpu)) | sgi_num);
}
```

### Core-Local Mailboxes

The BCM2711 has ARM local peripherals including per-core mailboxes:

```
0x4000_0000     ARM Local Base
    +0x0050     Core 0 Mailbox 0 Interrupt Control
    +0x0054     Core 1 Mailbox 0 Interrupt Control
    +0x0058     Core 2 Mailbox 0 Interrupt Control
    +0x005C     Core 3 Mailbox 0 Interrupt Control
    +0x0060     Core 0 IRQ Source
    +0x0064     Core 1 IRQ Source
    ...
    +0x0080     Core 0 Mailbox 0 Write-Set
    +0x0084     Core 0 Mailbox 1 Write-Set
    ...
```

These mailboxes provide an alternative IPI mechanism, though QNX primarily uses GIC SGIs.

---

## 9. VideoCore Mailbox Interface

**Files**: 
- `src/hardware/startup/boards/bcm2711/mbox.c`
- `src/hardware/startup/boards/bcm2711/mbox.h`

### VideoCore Architecture

**Source**: This boot sequence is documented in Raspberry Pi architecture guides and confirmed by evidence in the code.

The Raspberry Pi 4 has a unique architecture where the GPU (VideoCore VI) boots first:

```
Power On
   ↓
VideoCore GPU boots from bootcode.bin on SD card
   ↓
GPU initializes DRAM
   ↓
GPU loads start.elf (GPU firmware)
   ↓
GPU parses config.txt
   ↓
GPU loads kernel (startup-bcm2711-rpi4)
   ↓
GPU releases ARM cores from reset
   ↓
ARM cores start executing at kernel entry point
```

**Code Evidence for GPU-first Boot:**
- `mbox.c`: Extensive mailbox communication functions (ARM requests info from GPU)
- `mbox_get_vc_memory()`: GPU owns and allocates VideoCore RAM
- `mbox_get_cmdline()`: Boot cmdline comes from GPU firmware
- `mbox_get_clock_rate()`: CPU frequency set by GPU
- Minimal `_start.S`: No DRAM initialization (GPU already did it)
- Memory reservation for "VideoCore memory" in `bcm2711_init_raminfo.c`

### Mailbox Protocol

**Source**: Protocol details from code analysis (`mbox.c` implementation).

The mailbox is a bidirectional communication channel:

```c
#define MBOX_REG_BASE               0xfe00b880
#define MBOX_BUFFER_ADDR            0x100      // Fixed buffer at 0x100
#define MBOX_BUFFER_SIZE            128
#define MBOX_SEND_CHANNEL           8

// Mailbox registers
#define MBOX0_READ                  0x00       // Read from VC
#define MBOX0_STATUS                0x18       // Status flags
#define MBOX1_WRITE                 0x20       // Write to VC
#define MBOX1_STATUS                0x38       // Status flags

#define MBOX_STATUS_FULL            0x80000000
#define MBOX_STATUS_EMPTY           0x40000000
```

### Message Structure

```c
struct mbox_msg_header {
    uint32_t size;      // Total message size (including header)
    uint32_t code;      // Request: 0, Response: 0x80000000
};

struct mbox_msg_tag {
    uint32_t tag;       // Property tag ID
    uint32_t size;      // Buffer size
    uint32_t code;      // Request: 0, Response: 0x80000000 | value_length
};

// Example message format
struct mbox_msg_get_clock_rate {
    struct mbox_msg_header header;
    struct mbox_msg_tag tag;
    struct {
        uint32_t clock_id;  // Input: which clock
        uint32_t rate;      // Output: clock rate in Hz
    } body;
    uint32_t end;           // End tag: 0x00000000
};
```

### Sending a Mailbox Message

```c
static uint32_t mbox_send_message(void) {
    uint32_t rdata;

    // Wait for mailbox ready (not full)
    for (;;) {
        rdata = *((volatile uint32_t *) (MBOX_REG_BASE + MBOX0_STATUS));
        if (!(rdata & MBOX_STATUS_FULL)) {
            break;
        }
    }

    // Send message address | channel number
    rdata = (MBOX_BUFFER_ADDR & ~0xf) | MBOX_SEND_CHANNEL;
    *((volatile uint32_t *) (MBOX_REG_BASE + MBOX1_WRITE)) = rdata;

    // Wait for response
    for (;;) {
        // Wait for mailbox not empty
        rdata = *((volatile uint32_t *) (MBOX_REG_BASE + MBOX0_STATUS));
        if (rdata & MBOX_STATUS_EMPTY) {
            continue;
        }

        // Read response
        rdata = *((volatile uint32_t *) (MBOX_REG_BASE + MBOX0_READ));
        
        // Make sure it's for our channel
        if ((rdata & 0xf) == MBOX_SEND_CHANNEL) {
            return rdata & ~0xf;
        }
    }

    return 0;
}
```

### Common Mailbox Functions

#### Get Board Revision

```c
uint32_t mbox_get_board_revision(void) {
    volatile struct mbox_msg_get_board_revision {
        struct mbox_msg_header header;
        struct mbox_msg_tag tag;
        struct {
            uint32_t revision;
        } body;
        uint32_t end;
    } *msg = (volatile struct mbox_msg_get_board_revision *) MBOX_BUFFER_ADDR;

    MSG_INIT(msg, MBOX_TAG_GET_BOARDREVISION);
    mbox_send_message();
    return msg->body.revision;
}
```

**Returns**: 32-bit revision code, e.g., `0xa03111` for RPi 4B 1GB:
- `0xa` = New-style revision
- `3` = BCM2711 SoC
- `1` = 512MB RAM
- `11` = RPi 4B type
- `1` = Revision 1.1

*(Format documented in `init_board_type.c` comments and Raspberry Pi documentation)*

#### Get MAC Address

```c
void mbox_get_board_mac_address(uint8_t* const address) {
    volatile struct mbox_msg_get_board_mac_address {
        struct mbox_msg_header header;
        struct mbox_msg_tag tag;
        struct {
            uint8_t address[6];
            uint8_t _pad[2];
        } body;
        uint32_t end;
    } *msg = (volatile struct mbox_msg_get_board_mac_address *) MBOX_BUFFER_ADDR;

    MSG_INIT(msg, MBOX_TAG_GET_MACADDRESS);
    mbox_send_message();
    memcpy(address, (uint8_t*) msg->body.address, 6);
}
```

**Returns**: Ethernet MAC address (burned into BCM2711 OTP)

#### Get Clock Rate

```c
uint32_t mbox_get_clock_rate(const uint32_t id) {
    volatile struct mbox_msg_get_clock_rate {
        struct mbox_msg_header header;
        struct mbox_msg_tag tag;
        struct {
            uint32_t clock_id;
            uint32_t rate;
        } body;
        uint32_t end;
    } *msg = (volatile struct mbox_msg_get_clock_rate *) MBOX_BUFFER_ADDR;

    MSG_INIT(msg, MBOX_TAG_GET_CLOCKRATE);
    msg->body.clock_id = id;
    mbox_send_message();
    return msg->body.rate;
}
```

**Clock IDs:**
- `1`: EMMC1
- `2`: UART
- `3`: ARM (CPU frequency)
- `4`: CORE (VPU frequency)
- `5`: V3D
- `6`: H264
- `7`: ISP
- `8`: SDRAM
- `9`: PIXEL
- `10`: PWM
- `12`: EMMC2

#### Get Temperature

```c
int32_t mbox_get_temperature(const uint32_t id) {
    volatile struct {
        struct mbox_msg_header header;
        struct mbox_msg_tag tag;
        struct {
            uint32_t temperature_id;
            int32_t millidegrees;
        } body;
        uint32_t end;
    } *msg = (volatile void *) MBOX_BUFFER_ADDR;

    MSG_INIT(msg, MBOX_TAG_GET_TEMPERATURE);
    msg->body.temperature_id = id;
    mbox_send_message();
    return msg->body.millidegrees;  // Returns temperature in millidegrees C
}
```

#### Get GPIO State (Expansion GPIO)

The BCM2711 has "expansion GPIOs" managed by the GPU firmware:

```c
uint32_t mbox_get_expgpio(const uint32_t gpio) {
    volatile struct {
        struct mbox_msg_header header;
        struct mbox_msg_tag tag;
        struct {
            uint32_t gpio;
            uint32_t state;
        } body;
        uint32_t end;
    } *msg = (volatile void *) MBOX_BUFFER_ADDR;

    MSG_INIT(msg, MBOX_TAG_GET_GPIO_STATE);
    msg->body.gpio = gpio;
    msg->body.state = 0;
    mbox_send_message();
    return msg->body.state;
}
```

**Expansion GPIOs:**
- `128`: BT_ON (Bluetooth power)
- `129`: WL_ON (WiFi power)
- `130`: PWR_LED (Power LED control)
- `131`: ACT_LED (Activity LED)
- `132`: SD_VDDIO (SD card voltage)
- `133`: CAM_SHUTDOWN (Camera shutdown)

### Mailbox in Main Startup

The main initialization uses mailbox extensively:

```c
// Get CPU frequency
cpu_freq = mbox_get_clock_rate(MBOX_CLK_ARM);

// Get memory configuration
const uint32_t vcram_size = mbox_get_vc_memory();

// Get board info
board_rev = mbox_get_board_revision();
uint64_t serial = mbox_get_board_serial();

// Debug output
if (debug_flag > 3) {
    kprintf("Temps %d\n", mbox_get_temperature(0));
    kprintf("Volts");
    for (unsigned i = 1; i <= 4; i++) {
        kprintf(" %u=%u", i, mbox_get_voltage(i));
    }
    kprintf("\n");
}
```

---

## 10. Watchdog Timer

**File**: `src/hardware/startup/boards/bcm2711/init_wdt.c`

### BCM2711 PM Watchdog

The Power Management block includes a watchdog timer that can reset the system:

```c
#define PM_PASSWORD                 0x5a000000  // Required for all PM writes
#define PM_WDOG_TIME_SET            0x000fffff  // Max watchdog value
#define PM_RSTC_WRCFG_CLR           0xffffffcf  // Clear reset config
#define PM_RSTS_HADWRH_SET          0x00000040  // Hardware reset flag
#define PM_RSTC_WRCFG_SET           0x00000030  // Full reset config
#define PM_RSTC_WRCFG_FULL_RESET    0x00000020  // Full reset mode
#define PM_RSTC_STOP                0x00000102  // Stop watchdog
#define PM_WDOG_MS_TO_TICKS         67UL        // Conversion factor
#define WDT_MAX_TICKS               0x000fffff  // Maximum ticks
#define WDT_DEFAULT_TICKS           0x00010000  // Default timeout
```

### Watchdog Enable

```c
void bcm2711_wdt_enable(const int timeout_value)
{
    const uintptr_t base = (uintptr_t) BCM2711_PM_BASE;
    uint32_t val;
    uint64_t ticks;

    /* Disable watch dog first */
    out32(base + BCM2711_PM_RSTC, PM_PASSWORD | PM_RSTC_STOP);

    // Calculate timeout in ticks
    if (timeout_value <= 1000) {
        val = WDT_DEFAULT_TICKS;  // ~4.2 seconds
    } else {
        ticks = timeout_value * PM_WDOG_MS_TO_TICKS;
        if (ticks > WDT_MAX_TICKS) {
            ticks = WDT_MAX_TICKS;  // Cap at maximum (~262 seconds)
        }
        val = (uint32_t) ticks;
    }
    
    /* Reload the timer value */
    out32(base + BCM2711_PM_WDOG, PM_PASSWORD | val);

    /* Enable the watchdog timer in full reset mode */
    val = in32(base + BCM2711_PM_RSTC) & PM_RSTC_WRCFG_CLR;
    val |= (PM_PASSWORD | PM_RSTC_WRCFG_FULL_RESET);
    out32(base + BCM2711_PM_RSTC, val);
}
```

### Watchdog Timing

- **Tick Rate**: ~16 kHz (one tick ≈ 62.5 microseconds)
- **Conversion**: 1 millisecond ≈ 67 ticks (actually 67.108...)
- **Default**: 0x10000 ticks ≈ 4.2 seconds
- **Maximum**: 0xFFFFF ticks ≈ 262 seconds

### Usage

Enable watchdog with custom timeout:
```bash
startup-bcm2711-rpi4 -W5000  # 5 second watchdog
```

The watchdog must be "kicked" periodically by a userspace utility (`wdtkick`) to prevent system reset:

```c
// Typical wdtkick implementation
while (1) {
    out32(BCM2711_PM_BASE + BCM2711_PM_WDOG, PM_PASSWORD | current_value);
    sleep(timeout / 2);  // Kick before timeout expires
}
```

---

## Conclusion

The QNX startup code for Raspberry Pi 4 is a sophisticated piece of low-level systems programming that bridges the gap between the VideoCore GPU bootloader and the QNX kernel. Key takeaways:

### Architecture Highlights

1. **Dual-Processor Boot**: GPU boots first, initializes hardware, then releases ARM cores
2. **Clean Abstraction**: Board-specific code separated from common ARM64 logic
3. **Mailbox Communication**: Extensive use of GPU mailbox for hardware queries
4. **Modern ARM**: GICv2 interrupts, ARMv8-A AArch64 mode, 4-core SMP

### Memory Management

- Flexible RAM configuration (1GB to 8GB)
- VideoCore memory sharing (split configurable)
- High memory support for 8GB models
- Address space tagging for DMA constraints

### Hardware Database

- Runtime hardware discovery
- No hardcoded device addresses in drivers
- Supports board variants transparently
- MAC address, serial number from OTP

### Performance Optimizations

- Callouts avoid context switches
- Direct hardware access in critical paths
- SMP support for all 4 cores
- MSI interrupts for PCIe

### Production Features

- Watchdog timer support
- System reboot mechanism
- Comprehensive debug output
- Multiple UART options

This startup code demonstrates production-quality embedded systems design, balancing flexibility, performance, and maintainability. It serves as an excellent reference for anyone developing low-level code for modern ARM SoCs.

---

## Additional Resources

- **QNX Documentation**: http://www.qnx.com/developers/docs/
- **BCM2711 Documentation**: Broadcom BCM2711 ARM Peripherals
- **Raspberry Pi Documentation**: https://www.raspberrypi.org/documentation/
- **ARM Cortex-A72 TRM**: ARM Technical Reference Manual
- **GICv2 Architecture**: ARM Generic Interrupt Controller Architecture Specification

---

**Author Note**: This document is based on analysis of QNX SDP 8.0 BSP for Raspberry Pi 4 (BCM2711). Code examples are simplified for clarity. Always refer to the original source code and official documentation for production use.

**License**: This analysis document follows the same Apache License 2.0 as the analyzed code.

**Date**: February 7, 2026
