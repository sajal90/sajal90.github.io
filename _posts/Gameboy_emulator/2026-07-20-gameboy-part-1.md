---
layout: post
title: "Building a Game Boy Emulator in Rust: Part 1"
date: 2026-07-20
---

Recently, I built a `chip8_emulator` in rust. So, naturally, I wanted to continue this passion of building emulators. I looked into some online discussions, and people recommend building either a NES emulator or GameBoy emulator as a natural next step after `chip8_emulator`.

I decided to go with GameBoy, mainly because I have a lot of childhood memories attached to it. Searching online, I came across this [blog post by Jeremy Banks](https://jeremybanks.github.io/0dmg/2018/05/23/getting-started.html), while I didn't read their blogs fully. I found some other awesome resources through their blogs, like this [ultimate gameboy talk](https://youtu.be/HyzD8pNlpwI), which was really inspiring and exciting, and this [unofficial manual](http://marc.rawer.de/Gameboy/Docs/GBCPUman.pdf), which was really helpful for completing the instruction set.

I picked Rust for this because my `chip8_emulator` was also written in Rust, and the main reason I even started that project was to learn Rust. It felt natural to just keep using it here too instead of switching to something else.

## 1. CPU and Registers

The Game Boy is a handheld game console developed and marketed by Nintendo. It was released on April 21, 1989. It uses a Sharp LR35902 processor, a hybrid of the Intel 8080 and Zilog Z80 processors. The CPU is a 8-bit CPU with 8 different 8-bit registers, labelled `{'a', 'b', 'c', 'd', 'e', 'f', 'h', 'l'}`. But there are instructions that allow to read and write 16-bit data, these are achieved through combining the 8-bit registers as `'af'`, `'bc'`, `'de'`, and `'hl'`.

The eight registers are simple, just normal `u8` fields on the `Cpu` struct, except the `'f'` (flags) register. The lower four bits of the register are always 0s and the CPU automatically writes to the upper four bits when certain things happen. Bit 7 is `Z` (zero), bit 6 is `N` (subtract), bit 5 is `H` (half carry), bit 4 is `C` (carry). I made a small `FlagsRegister` type that turns these 4 flags into one byte and back again. So in the rest of the code I can just write `cpu.registers.f.zero` instead of doing bit math by hand every time.

The 16-bit pairs, `AF`, `BC`, `DE`, `HL`, as I said before, are two registers put together in a fixed order. For `AF`, `A` is the high byte and `F` is the low byte. Same for the rest. I wrote small `get` and `set` functions for each pair, so it can be used later when needed.

## 2. Instructions

For implementing instructions, I was following [this guide](https://rylev.github.io/DMG-01/public/book/cpu/register_data_instructions.html). I liked it because It has examples of some of the instructions, but majority of the work was left for the reader. For the rest of the instructions I used the [unofficial manual](http://marc.rawer.de/Gameboy/Docs/GBCPUman.pdf), google searches, and my brain :)
The GameBoy has  a base instruction set of 8 distinct 8-bit opcodes that are expanded into 244 unique base instructions, and 257 extended CB-prefixed instruction set (mostly for bitwise operations), the total native CPU instructions come to 501 possible variations.

Mapping and implementing this large of instruction set was a pain, I had to do a lot of copy pasting. The instructions are mapped to their respective opcodes in the `instructions.rs` file, which is imported by the `cpu.rs` file, where a huge `match` statement in the `execute()` function is used to match the instruction with its functionality.

The most fun part of this was writing bit manipulation logic for these instructions. Not that I learnt any new bit manipulation trick (maybe I did, and I already forgot in 2 weeks), but it was good revision of it. Most of the hardships came from the gameboy quirks, for example GameBoy is [endian little](https://en.wikipedia.org/wiki/Endianness). So for a 16bit value, its LSB is stored at the lower address.

While implementing the instructions, I also wanted to test them on the way. I found these [blargg test roms](https://github.com/retrio/gb-test-roms) on some wiki, which I can't found at this moment (I tried looking into my history but I mostly do my research on incognito :(
Now, I have the test roms, but things that are important to load the rom, such as memory mapping, load instructions, stack etc. wasn't done yet. So, I decided to use Claude to implement a minimal working model that can load the rom. I, honestly, didn't expect it to work, but if it did work it could save me a lot of time. And it did work. But I didn't pushed the Claude's written code on github, mainly because I didn't look much into it.
So, I ran the tests, most were failing, as expected. But on test `04-op\ r,imm.gb` gave a thread panic due to overflow. The culprit was this line:

```rust
let (new_val, overflow) = self.registers.a.overflowing_add(val + carry);

```

The bug being if `val` was `0xFF` (max size of `u8`) and `carry` was `1`, it would overflow. So, I cast both, `val` and `carry`, into `u16`, and later the result into `u8`:

```rust
let (new_val, overflow) = self.registers.a.overflowing_add(((val as u16) + (carry as u16)) as u8);

```

and re-ran the tests, and now the error was `CE Failed`. I looked online, and it seemed that the error means wrong implementation of carry or half-carry flag. I didn't know if the error was in the `add_carry` method or any other method. And I was tired, So, I just added a `'BUG:'` comment above the function and left it for some other day.

6 days later, I finally realized what the problem with my above approach was. If `val == 0xFF` and `carry == 1`, the result will be `256` in `u16`, but when we cast it to `u8`. It will wrap around to `0` and the `overflow` bool will be false.

Fixed that by using another variable:

```rust
let sum = (val as u16) + (carry as u16);
let (new_val, overflow) = self.registers.a.overflowing_add(sum as u8);

```

and for overflow flag, checking both if `val+carry == 256` or `val+carry+reg_a > 255`:

```rust
self.registers.f.carry = overflow || sum > 0xFF;

```

## 3. Stack, Function Calls, and Interrupts

I love stacks, it's such a simple data structure, but the ways in which it can be used always surprises me. GameBoy has a 16-bit hardware stack in its work ram, which can be used to `PUSH` and `POP` 16-bit registers. `Push` moves `SP` (stack pointer) down by 1, writes the high byte, moves `SP` down again, writes the low byte. `Pop` does the same but backward, low byte first, then high byte, moving `SP` up after each read. This allows us to call functions and return from them.

`CALL` and `RET` (return) instructions are used to, well, call a function and return from a function. In `CALL`, we `PUSH` the next value of the program counter to the stack, this is the return address of the function. Then, we jump to the actual address of the function, which is the last 2 bytes of the 3 bytes `CALL` instruction. For `RET`, we `POP` that saved return address off the top of the stack and jump back to it to resume our program.

Next is Interrupts, Interrupts are hardware signals that the main CPU code execution to handle other events, such as keyboard event, or timer interrupt. GameBoy handles this using a master enable flag (`IME`) inside the CPU, and two memory-mapped registers: `IE` (Interrupt Enable) at `0xFFFF` and `IF` (Interrupt Flag) at `0xFF0F`. When a bit in both `IE` and `IF` is set to 1, and `IME` is true, an interrupt is requested. The `EI` and `DI` instructions are used to set `IME` to true and false respectively. Although `DI` runs immediately, `EI` on real hardware take a 1 instruction delay, though I haven't implemented that yet as everything runs fine for now.

## 4. Memory

Now that the CPU could execute instructions, it needed somewhere to actually read and write data. The Game Boy's memory isn't one flat block, it's a bunch of separate regions stitched together, and the CPU has no idea that `0x8000` is video RAM while `0xC000` is work RAM, that's the memory bus's job. I wrote a `MemoryBus` struct holding each region as its own array (`vram`, `eram`, `wram`, `oam`, `io`, `hram`), with `read_byte`/`write_byte` routing through one big `match` on the address, not that different from the `execute()` match from the CPU section, just matching addresses instead of opcodes.

The fun part was the serial port hack for the blargg test ROMs. On real hardware, writing `0x81` to `0xFF02` (`SC`) kicks off a byte-by-byte transfer over a link cable, using whatever's staged in `0xFF01` (`SB`). I don't have a link cable, but the test ROMs use this exact mechanism to print pass/fail results as plain text, so I just intercept writes to `SC`, and if the value is `0x81`, grab the byte in `SB` and print it straight to stdout.

Two more things worth talking about: `interrupt_flag` starts at `0xE1`, not `0`, because the top 3 bits are unused and hardwired high on real hardware. And the `0x0000..=0x7FFF` write branch currently does nothing, which is fine for simple 32KB ROMs but means no Memory Bank Controller support yet, so anything that needs bank switching is broken until I implement MBC1/MBC3.

## 5. Where things stand

Once the core CPU was complete, I tried to implement the hardware timers to pass Blargg's CPU test ROMs. I quickly ran into issues with cycle accuracy. The timer hardware expects a precise falling-edge logic implementation, and my initial cycle-counting approach was getting out of sync with the CPU execution.

I decided to do a `git reset` to clear out the broken timer code and leave the repository in a stable state.
