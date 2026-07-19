# Introduction


The **MOS 6510** is the derivative of the **MOS 6502** 8-bit processor used in machines such as the Apple-II, NES and Atari 2600. But this specific CPU variant was designed for the Commodore 64 home computer with a **built-in 6-pin I/O port (P0-P5 pins)** and **a tri-state address bus (AEC pin)**.


The pin-out view (* = **active low**):

```
                                     +-----------+
                             in   <- |           | ->  RES*
                             RDY  <- |           | ->  out 
                             IRQ* <- |           | ->  R/(W*)
                             NMI* <- |           | ->  D0
                             AEC  <- |           | ->  D1
                             VCC  <- |           | ->  D2
                             A0   <- |           | ->  D3
                             A1   <- |           | ->  D4
                             A2   <- |           | ->  D5
                             A3   <- |    MOS    | ->  D6
                             A4   <- |    6510   | ->  D7
                             A5   <- |           | ->  P0
                             A6   <- |           | ->  P1
                             A7   <- |           | ->  P2
                             A8   <- |           | ->  P3
                             A9   <- |           | ->  P4
                             A10  <- |           | ->  P5
                             A11  <- |           | ->  A15
                             A12  <- |           | ->  A14
                             A13  <- |           | ->  GND
                                     +-----------+
```

Even though there are some crucial changes in the pin-out, the instruction set and the addressing modes have not been changed and are directly compatible with the 6502 (unlike the CMOS-based 65c02). But first, let me introduce you to the key concepts of this CPU and what makes it special. Later, we will touch upon the specifics of these features.


## 1) Memory Banking

Since the 6502 and 6510 have the same address bus length (an address bus length of **16-bits**), they can only physically reference **64KB of total address space**, with each address holding a value of **1 byte** (so between *0-255*). If one populates and uses the whole memory space, there would not be any space for things like the **KERNAL ROM**, **the BASIC ROM** or **the I/O chips** (VIC-II for graphics and SID for sound). 

In order to solve this problem, the 6510 maps the I/O ports directly to addresses `0x0000` (**Data Direction Register** or **DDR** for short) and `0x0001` (**Data Port** or **DP** for short). The pins **P0-P2** are being used for this specific use case, they help us to determine with what ROM or I/O section operations will be performed at. The CPU controls the **Commodore's Programmable Logic Array (PLA)**, which instantly swaps RAM and ROM in and out of active memory.


## 2) The tri-state bus

Unlike the 6502, the 6510 includes an **AEC (Address Enable Control)** pin. This pin essentially enables the CPU to enter a tri-state (high-impedence) mode, disconnecting the CPU itself from the address bus and allowing the VIC-II video chip to take over the system memory (Direct Memory Access or DMA for short) and renders graphics during additional clock cycles (the VIC-II basically "steals" clock cycles from the CPU). This shared-bus actually prevents screen flickering and tearing during active processing.


## 3) Datasette and Peripherals


The remaining I/O port pins **(P3-P5)** were dedicated for the I/O operations for the **Datasette** (the Commodore way of saying Data Casette) and **other external peripherals**. This way the C64 eliminated the use of additional chips dedicated only for external I/O flow, which reduced the manufacturing cost and kept the motherboard very compact.


## 4) Why the Commodore 64?


My first remark of the C64 started with the 6502 itself, seeing that it was used in many systems like the NES and Apple-II made me think that creating a 6502 emulator might be a good starting point for cycle-accurate emulation with this CPU being an incredibly simple but amazing piece of engineering. Coming across the renowned Commodore 64 back in the day blew me away when I discovered the capabilities of this 8-bit computer. This computer was a remarkable piece of engineering back in the day, and hearing the nostalgia from its previous users made me curious about this masterpiece even more, hence the attempt of emulating it. 
