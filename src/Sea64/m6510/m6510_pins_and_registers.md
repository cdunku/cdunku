# Pins and Registers

In this section of the MOS 6510 documentation we will be covering the CPU's pin-out, the CPU registers and the CPU flags, as well as the implementation of the previously mentioned. I will consider that the reader is already familiar with the terms such as opcode or operand, reserving me the time to explain what these terms mean.


## 1) Pins

This specific CPU has in total 40 pins, with only some being relevant while writing an emulator with the functionality of most pins being applicable to the MOS 6502. The following pins will be used:


- `A0 - A15` -> 16-bit address bus length of the C64 bus stored in 64KB of memory.

- `D0 - D7` -> 8-bit data bus length of the C64 stored in 64KB of memory.

-  `RDY` -> Halts the CPU when it accesses other parts of a computer that have slower memory access in comparison to the 6502. 
While in the C64 it is not tied to slowery memory, but rather it is connected to the `VIC-II` during `DMA (Direct Memory Access)`.

- `IRQ` -> **The Interrupt ReQuest** pin is responsible for maskable interrupts (these interrupts are mainly run in the software level),
it disrupts the normal flow of a program and the system enters into an alternative routine (subroutine). 
It is a maskable interrupt, because it can be ignored in some cases. All components such as the `VIC-II`, `SID` and `CIA` chips can generate an IRQ signal.

- `NMI`  -> **Non Maskable Interrupt** does what its name refers, it creates a non-maskable interrupt at the hardware level which it cannot be ignored. 
This signal is generated when for an example on reboots their computer.

- `R/W` -> This is the Read/Write signal, when it is high it indicates a Read signal inside the memory, and low indicated Write, which writes data in a specific address.

- `SYNC` -> This pin signals that the previous instruction is finished and that the CPU is "fetching" or "reading" the program counter value (that holds a memory location) for the opcode.

- `RES`  -> The **RESET** pin is a specific time of interrupt that is signaled whenever the computer is booted up in order to initialise it.


- `AEC`  -> **Address Enable Control** enables a tri-state (high impedance) mode, disconnecting the MOS 6510 from the main bus entirely and handing control to other components (such as the VIC-II).

- `P0 - P5` -> The **PORT** pins that handle external peripheral I/O flow, datasette and memory banking.


That was a lot to uncover, but each of these pins play an important role not only in real hardware, but also in emulator implementations as well. There are various ways of implementing and storing the status of each CPU pin, but `floooh` handling the CPU pins seemed the most readable and appealing option.


## 2) CPU Pin-Out implementation

What essentially I have done is that I used a 64-bit pin-mask (such as `m6502_pins`) in order to store the current status of each pin. I used an `enum` in order to each pin and label them as `M6510_IRQ_PIN` or `M6510_RW_PIN`. In order not to overcomplicate things, I assumed that pins such as address-bus and data-bus pins are connected together, instead of seperating them like in the actual pin-out.

**NOTE:** All the emulated pins will be **active-high** instead of some being **active-low** in the original hardware. It makes it more convenient to write everything as **active-high** and more readable too! The Commodore 64 was engineered in a way that some pins were actually active when no electrical signal was supplied, due to a totally different and very important reason. 

```c
typedef enum M6510_PINOUT {

  M6510_ABUS_SHIFT = 0,
  M6510_DBUS_SHIFT = 16,
  M6510_PORT_SHIFT = 30,

  M6510_ABUS_MASK  = 0xFFFFULL << M6510_ABUS_SHIFT,
  M6510_DBUS_MASK  = 0xFFULL   << M6510_DBUS_SHIFT,
  M6510_PORT_MASK  = 0x3FULL   << M6510_PORT_SHIFT,

  M6510_RDY_PIN = 24,
  M6510_IRQ_PIN,
  M6510_RW_PIN,
  M6510_AEC_PIN,
  
  M6510_NMI_PIN,
  M6510_SYNC_PIN,
  M6510_RES_PIN,

  M6510_P0_PIN,
  M6510_P1_PIN,
  M6510_P2_PIN,
  M6510_P3_PIN,
  M6510_P4_PIN,
  M6510_P5_PIN,

} M6510_PINOUT;
```

Next, one can store the bit positions of each pin in a seperate variable by shifting the value of `1` into their respective positions:

```c
static const uint64_t M6510_RDY  = PINMASK(M6510_RDY_PIN);
static const uint64_t M6510_IRQ  = PINMASK(M6510_IRQ_PIN);
static const uint64_t M6510_NMI  = PINMASK(M6510_NMI_PIN);
static const uint64_t M6510_SYNC = PINMASK(M6510_SYNC_PIN);
static const uint64_t M6510_RES  = PINMASK(M6510_RES_PIN);
static const uint64_t M6510_RW   = PINMASK(M6510_RW_PIN);

static const uint64_t M6510_AEC  = PINMASK(M6510_AEC_PIN);
static const uint64_t M6510_P0   = PINMASK(M6510_P0_PIN);
static const uint64_t M6510_P1   = PINMASK(M6510_P1_PIN);
static const uint64_t M6510_P2   = PINMASK(M6510_P2_PIN);
static const uint64_t M6510_P3   = PINMASK(M6510_P3_PIN);
static const uint64_t M6510_P4   = PINMASK(M6510_P4_PIN);
static const uint64_t M6510_P5   = PINMASK(M6510_P5_PIN);
```

But how does one actually update the live status of a CPU pin? It is simple really, we can create a seperate function for enabling a pin that looks like this:

```c
void pin_on(uint64_t *pins, uint64_t bit)  { *pins |= bit; }
```

This updates the value of one single pin! Same can be done when setting the CPU pin to inactive by doing:

```c
void pin_off(uint64_t *pins, uint64_t bit)  { *pins &= ~bit; }
```


But what about setting/getting the address-bus and data-bus values? Well in principle it is simple really, we mask the desired pins inside our `m6510_pins` variable, shift them so that the value returned is 16-bit or 8-bit.


The return function for the address bus:

```c
uint16_t get_abus(uint64_t pins, uint64_t mask, unsigned shift) {
  return (uint16_t)((pins & mask) >> shift);
}
```

The set function for the address bus would be a bit more complicating but relatively straight forward:

```c
void set_abus(uint64_t *pins, uint64_t mask, unsigned shift, uint16_t addr) {
  *pins = (*pins & ~mask) | (((uint64_t)addr << shift) & mask);
}
```


Essentially, we clear out the previous data inside the bit-range with `*pins & ~mask` and set the new value by shifting the address to the desired position, and mask it, in order to prevent accidental override with other pins (in case the shift value is changed). 


## 3) Registers

Unlike the Intel 8080, the MOS 6510 does not have as many registers. But each register can be categorised into different groups, heavily depending on their function:


- `A`  -> **The Accumulator** is the workhorse of the CPU, it is the only register that is directly wired to the **ALU (Arithmethic Logic Unit)**. 
Which means that it is responsible for **all arithmethic** (ADC, SBC), **bit-shifting** (ASL, ROL) and **bit-masking** (EOR, AND) operations.

- `X` -> Used primarily for structural addressing techniques, such as `Indexed Indirect Zeropage` or `$(ZP, X)`, it acts as a counter loop variable because it is incremented/decremented (INX, DEX) **directly inside** the CPU.
Which saves precious time from memory accesses, where **INX and DEX** are only single-byte instructions.

- `Y` -> The opposite from **X**, dedicated for handling large-scale data arrays through addressing modes such as `($ZP), Y`. 
**Y** cannot be used in certain unique arithmethic operations and addressing modes using **Y** use more clock cycles in comparison to **X**.


- `PC` -> **The Program Counter** is the only 16-bit CPU register, it helps the developer to track execution of the program (fetch the operands for each instruction) and fetches the next instruction (opcode). 
It holds the address the instruction is currently using, from which the CPU reads from in order to fetch data previously mentioned.
The PC is split internally into two 8-bit registers: `PCH` (High byte) and `PCL` (Low byte).

- `S` -> **The Stack Pointer** acts as an active pointer to the hardware's stack in memory and it is only 8-bits wide unlike the Intel 8080 (with a 16-bit SP). 
The address is physically hardcoded between the address range of `0x100 - 0x1FF`. The register decrements during pushing onto the stack (e.g. instruction PHA) and increments when pulled/popped (instruction PLA).


- `IR` -> **Instruction Register** holds the value of the current and following opcode. Value retrieved when `SYNC` is active.

- `TCU` -> **Timing Control Unit** tracks the current amount of cycles executed during the execution of an instruction.


- `AL` -> **The Address Latch** is a seperate 16-bit register, which is used for temporary address storing during the execution of addressing modes. 
It helps with the index-based calculation in addressing mode.

- `DL` -> **The Data Latch** very similar functionality to the `AL`, but instead of it storing an address, it stores a byte of data.


- `P` -> **The Processor Status Register** stores the single-bit conditional flags into this register. 
The status of each bit is affected and updated due to the recent outcomes of most ALU calculations or system interrupts.

With each bit condition having a very unique and important purpose:

```
                                  N V 1 B D I Z C
                                  ^ ^ ^ ^ ^ ^ ^ ^
                                  7 6 5 4 3 2 1 0
                                  | | | | | | | |
                                 +---------------+
                                 |  Register P   |
                                 +---------------+
```

- `N` -> **Negative flag** is set when the target result is signed (In signed integers, when the last bit is set to `1`, it is a negative integer).

- `V` -> **Overflow flag** is set if an invalid signed arithmethic wrap-around (basically overflow) occurs.

- `1` -> **Unused bit** set to `1` by default.

- `B` -> **Break flag** checks and distinguishes if a stack push was caused by a `BRK` software command.

- `D` -> **Decimal flag** switches the ALU into Binary Coded Decimal (BCD) mode.

- `I` -> **Interrupt Disable flag** blocks level-triggered (IRQ) interrupts when set. Checked in the second-to-last cycle of every instruction (or polling-phase).

- `Z` -> **Zero flag** is set when the value checked is equal to `0`.

- `C` -> **Carry flag** is set when an unsigned operation overflows `8` bits or underflows `0`. 


The amount of information might be overwhelming, but all of these serve a very important purpose in the CPU execution as we are going to see in the following chapters.


## 4) 6502 struct implementation


Since we have covered the key facts about the 6510's registers, let us actually implement them in C!


First, let's declare a struct that will store our variables:

```c
typedef struct m65xx_t {
 
 /* ... */

  uint64_t m6510_pins;
  
  uint8_t a, x, y, s, p, tcu;
  
  uint16_t ir;
  
  union { struct { uint8_t pcl; uint8_t pch; }; uint16_t pc; };
  
  union { struct { uint8_t adl; uint8_t adh; }; uint16_t ad; };

  /* ... */

} m65xx_t;
```

As previously mentioned, we use the `m6510_pins` variable to store the live state of the CPU pins and the all the `8-bit` registers are all declared at once, nice!
But what about `IR` (which in hardware only stores values up to 255), you may ask. 
Since the all other interrupts are essentially very similar to `BRK` under the hood, they can be written in the same `BRK` function with a bunch of conditional checks in order to determine the `interrupt vectors`. 
I avoided that implementation due to it being incredibly unreadable and not understandable, hence I have declared the interrupts and mapped them to the values `0x100`, `0x101` and `0x102`. 

You can sometimes break the rules in order to avoid redundancy :)! 

What about this weird `union { struct { ... } ... };` declaraction? It just basically maps out the low and high bytes of `PC` and `AD`inside memory, so that data stored inside either variable will be directly reflected to the main `16 bit` variable (only possible in the C standards after `C99`). Additionally, since the MOS 6502 is natively `little-endian`, the lower bytes have to be declared first and right after the high bytes follow.


This is the end of the section regarding the pins and registers of the `MOS 6510`! Next, we will be covering `fetch cycle` and how it works under the hood.
