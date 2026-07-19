# The Fetch-Decode-Execute Cycle

During every read of a specific memory location, data that can be retrieved and later worked with or manipulated. 
This data can be either an operand (the data an instruction uses during its execution) or an opcode (the byte that identifies the instruction), but can the CPU distinguish the data used as an opcode or an operand? Spoiler Alert: **It can't**, the CPU is **"too dumb"** and **"helpless"** on its own. That is why the CPU has to have specific pins in order to operate as desired, with one of these pins being the previously mentioned `SYNC` pin. The CPU later identifies the instruction and begins with execution.


## 1) Fetching Initialisation

During the end of each instruction, the CPU sets the right conditions for the following instruction-fetching. The conditions are reading current address the `PC` points to, reseting the `TCU` to `0`, setting the `READ` signal and activating the `SYNC` pin.

Here is an example from the `IMMEDIATE` (`#`) addressing mode along with our `fetch()` function, `IMMEDIATE` is 2 clock cycles long:

```c
void m6510_fetch(m65xx_t* const m) {
  m6510_set_abus(m, m->pc);
  m6510_pin_on(m, M6510_SYNC);
}


void imme(m65xx_t* const m) {
  switch (m->tcu) {
    
    case 1: {
      m6510_set_abus(m, m->pc);

      break;
    }

    case 2: {
      m6502_opcode_table[m->ir].instr(m); 

      m->tcu = 0;
      m6510_fetch(m);
      break;
    }
    
  }
}
```

During `IMMEDIATE`'s execution, the 1st clock cycle reads the `PC` for the operand our instruction will use and the 2nd clock cycle executes our instruction and sets the right conditions for an `opcode fetch`. The CPU finished the instruction and continues with the next fetched opcode.


## 2) Fetching, Decoding, Executing


So, we have set the right conditions and finished the last instruction, what next? Well, we have to fetch the new instruction, but before that, the CPU checks for edge-cases and interrupt conditions, I/O requests before reaching the fetching cycle. The reason is, there are many high priority tasks and requests from other components and schedulers that need to be checked before even giving a chance for the CPU to do its own thing. That is how we allow chips such as the `VIC-II` to do their job as intended. 

After passing the checks for edge-cases and I/O requests, we deactivate the `SYNC` pin, after entering into the fetch-cycle phase. But before putting our opcode into the `IR` (this register stores and helps us identify the opcode), the CPU checks for interrupts in the first phase of the fetch cycle, because interrupt execution has priority over normal instruction execution. 


No interrupt has occurred and we can finally fetch our opcode? Great!

```c
void m6510_tick(m65xx_t* const m) {

  /*
    *Interrupt conditions, edge cases and I/O requests 
    *                    ...
  */

  if(m->m6510_pins & M6510_SYNC) {
    
    m6510_pin_off(m, M6510_SYNC);
  
    // Logic for executing interrupts here
    if(INTERRUPT_HAS_OCCURRED) {
        /* 
         * Check which interrupt and execute
         *             ...
        */
    } 
    else {
      m->ir = m6510_get_dbus(m);
      m->pc++; 
    }
  }

  m->tcu++;
  m->cpu_clock++;

  m6510_pin_on(m, M6510_RW);
  // Call instruction/addressing mode 
  m6502_opcode_table[m->ir].mode(m);
}
```

During a fetch cycle, the processor retrieves the opcode value from the data bus and puts it onto `IR` and increments the `PC`. Incrementing the `PC` is important because we are fetching an operand for the instruction for it to work with, which in most cases the operand will be used during the same instruction. Addressing modes such as `IMPLIED` and `ACCUMULATOR` are special cases. After the value of `IR` is updated, the CPU searches the opcode in a table according to the value retrieved from the data bus.

Let's say as an example `IR = 0xA9`, the CPU goes through its mapped table and finds out that `0xA9` is `LDA #`.

```c
m65xx_opcodes_t m6502_opcode_table[0x103] = {
  [0x00] =  { .mode = brk, .instr = impl },
  [0x01] =  { .mode = idxr, .instr = ora },
  [0x02] =  { .mode = jam, .instr = impl },
  [0x03] =  { .mode = idxm, .instr = slo },
 
 /*                ...                  */

  [0xA9] =  { .mode = imme, .instr = lda },

  /*                ...                  */

}
```

It decodes the opcode by checking the addressing mode and the instruction that will be executed. 
The addressing modes gets executed first, followed by the instruction:


```c 
/*         EXECUTE ADDRESSING MODE FIRST          */


void imme(m65xx_t* const m) {
  switch (m->tcu) {
    
    case 1: {
      m6510_set_abus(m, m->pc);

      break;
    }

    case 2: {
     
     // This is where we call the instruction before the new fetch cycle
     // when the instruction execution is done, we come back to this function again
     // and in the 2nd clock phase of the last clock cycle we set the conditions for fetch 
      m6502_opcode_table[m->ir].instr(m); 

      m->tcu = 0;
      m6510_fetch(m);
      break;
    }
    
  }
}

/*         NOW EXECUTE THE INSTRUCTION         */

void lda(m65xx_t* const m) {
  set_nz(m, m->a = m6510_get_dbus(m));
}
```

After executing the instruction in the 1st phase of the last clock cycle, we repeat! The CPU sets the conditions and for the next `fetch` cycle in the 2nd clock phase and those everything from scratch.


This wasn't too bad was it? In the next section we will be learning and going through different types of addressing modes and what happens during each clock cycle in each addressing mode.
