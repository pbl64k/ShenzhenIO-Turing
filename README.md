# Betelgeuse 9900

## What's Shenzhen I/O?

[Shenzhen I/O](http://www.zachtronics.com/shenzhen-io/) is a Zach-like puzzle/engineering game by
[Zachtronics](http://www.zachtronics.com/), somewhat similar to their earlier
[TIS-100](http://www.zachtronics.com/tis-100/). Zach Barth strikes again! Zach. (I just thought that maybe there
weren't enough Zachs in this paragraph.)

Shenzhen I/O is currently (as of Oct 2016) in Early Access [on Steam](http://store.steampowered.com/app/504210/).

## B9900 Capabilities & Limitations

Betelgeuse 9900 is a universal, programmable, [von Neumann architecture](https://en.wikipedia.org/wiki/Von_Neumann_architecture)
microcomputer in [1970s style](https://en.wikipedia.org/wiki/Altair_8800)
([TEC-1 is perhaps a closer match](https://en.wikipedia.org/wiki/TEC-1)). It is very much a
[RASP](https://en.wikipedia.org/wiki/Random-access_stored-program_machine), modulo certain ugly real-world details. It
has 42 almost-11-bit words of RAM, simple numeric display and a gamepad. Monitor/debugger is implemented in "hardware"
and uses the gamepad for input.

The architecture is capable of addressing 1000 words of RAM, and the memory controller design is such that more RAM
chips could be easily added to it up to that limit. Unfortunately, there simply isn't enough space on the virtual
circuit board to fit more that one address router assembly.

The CPU has no access to I/O devices, so the only way the operator can interact with the program is through using the
hardware monitor/debugger. Once again, ignoring the board space limitations, I/O could be implemented without changing
the architecture or instruction set by injecting a DMA controller into the memory bus.

The CPU counts in decimal, with value range from -999 to 999, because that's what virtual microcontrollers it's implemented
on do.

## In Action!

- [Factorial program](https://www.youtube.com/watch?v=8L4xBvnxPUQ)
- [GCD program](https://www.youtube.com/watch?v=9reU8p3pkF4)

## In-Game Cost

| Part      | Units | Cost Per Unit | Total |                                                           |
|:--------- | -----:| -------------:|------:| --------------------------------------------------------- |
| MC6000    | 6     | ¥5            | ¥30   | CPU1-3, PC, Mon/Dbg1-2                                    |
| MC4000X   | 5     | ¥3            | ¥15   | memory controller, address router, RAM chip controller x3 |
| 100P-14   | 3     | ¥2            | ¥6    | RAM chips                                                 |
| LX700     | 1     | ¥4            | ¥4    | Numeric LCD                                               |
| N4GP-1000 | 1     | ¥2            | ¥2    | Gamepad                                                   |
| Total     | 16    |               | ¥57   |                                                           |

## Instruction Set

B9900's instruction set is even more limited than that of the underlying MC series virtual microcontrollers, but it
compensates for that by being a stored-program computer, thus being capable of running self-modifying code.

| Opcode    | Mnemonic        | Description                                                            |
| ---------:| --------------- | ---------------------------------------------------------------------- |
| 1         | `JLEZ tst, tgt` | If the word at `tst` is less than or equal to zero, jumps to `tgt`     |
| 2         | `ADD src, dst`  | Sums the words at `src` and `dst` and writes the result to `dst`       |
| 3         | `MOV src, dst`  | Writes the word at `src` to `dst`                                      |
| 4         | `MUL src, dst`  | Multiplies the words at `src` and `dst` and writes the result to `dst` |
| any other | `HALT`          | Stops execution and transfers control to hardware monitor/debugger     |

`HALT` is a single-word instruction, all other instructions consist of three words--opcode followed by two addresses.

Words are decimal integers in the -999 to 999 range, as dictated by the underlying "chips."

## Indirect Addressing

While all instructions use direct addressing mode, indirect addressing can be easily implemented by using
self-modifying code. For example, `MOV [src], [dst]` could be implemented as:

            MOV     src, ADDR + 1
            MOV     dst, ADDR + 2
    ADDR:   MOV     0, 0

Obviously, this requires knowledge of `ADDR`, and both `ADDR + 1` and `ADDR + 2` are constants that should be computed
when translating from assembly language notation to opcodes.

Capability for indirect addressing means it's possible to implement arrays, stacks and other data structures. In
particular, `PUSH`, `POP`, `CALL` and `RET` can be implemented to create a call stack. Unfortunately, the actual
B9900 implementation with its meagre 42 words of memory (which can fit no more than 14 non-`HALT` instructions) can't
possibly run any useful programs using the call stack implemented this way.

## Operation

B9900 starts in monitor/debugger mode, with edit address pointing at zeroth word. Use up and down keys on the gamepad
(`w` and `s` on actual keyboard) to move the edit address around. Only the word at the edit address is displayed on the
LCD, not the address itself, as adding a second LCD would be prohibitively expensive in terms of circuit board space.
Change the value at edit address by using left and right keys on the gamepad (`a` and `d` on keyboard). The *A* button (`j`)
increases the value at edit address by ten. *Start* button (`Enter`) executes the program. If the program terminates by
executing a `HALT` instruction, pressing *Start* again will continue execution from the next address after `HALT`.

## Example Programs

### Factorial

Given a number *n* placed at address `N` (20), computes *n!* and puts the result at address `FACT` (19). Duh. Sets `N` to
zero in the process. Will produce 1 as the result for negative inputs.

    START:  MOV     ONE, FACT
    LOOP:   JLEZ    N, END
            MUL     N, FACT
            ADD     MINUS1, N
            JLEZ    ZERO, LOOP
    END:    HALT
            JLEZ    ZERO, START
    ZERO:   DATA    0
    ONE:    DATA    1
    MINUS1: DATA    -1
    FACT:   DATA    0
    N:      DATA    n

Translated:

     0 3    ; START
     1 3
     2 19
     3 1    ; LOOP/ONE
     4 20
     5 15
     6 4
     7 20
     8 19
     9 2
    10 15
    11 20
    12 1
    13 18
    14 3
    15 -1   ; END/MINUS1
    16 1
    17 18
    18 0    ; ZERO
    19 0    ; FACT
    20 n    ; N

### GCD

Given two positive numbers *a* and *b* placed at addresses `A` (39) and `B` (40), computes their greatest common
divisor and places the result in both `A` and `B`. Supplying non-positive numbers as inputs could be inadvisable.
Uses the original Euclidean algorithm.

    START:  MOV     B, ASUBB
            MUL     MINUS1, ASUBB
            ADD     A, ASUBB        ; ASUBB = A - B
            MOV     ASUBB, BSUBA
            MUL     MINUS1, BSUBA   ; BSUBA = B - A
            JLEZ    ASUBB, ALEB
            MOV     ASUBB, A
            JLEZ    ZERO, START
    ALEB:   JLEZ    BSUBA, END
            MOV     BSUBA, B
            JLEZ    ZERO, START
    END:    HALT
            JLEZ    ZERO, START
    ZERO:   DATA    0
    MINUS1: DATA    -1
    ASUBB:  DATA    0
    BSUBA:  DATA    0
    A:      DATA    a
    B:      DATA    b
    
Translated:

     0 3        ; START
     1 40
     2 37
     3 4
     4 33
     5 37
     6 2
     7 39
     8 37
     9 3
    10 37
    11 38
    12 4
    13 33
    14 38
    15 1
    16 37
    17 24
    18 3
    19 37
    20 39
    21 1
    22 23
    23 0        ; ZERO
    24 1        ; ALEB
    25 38
    26 33
    27 3
    28 38
    29 40
    30 1
    31 23
    32 0
    33 -1       ; END/MINUS1
    34 1
    35 23
    36 0
    37 0        ; ASUBB
    38 0        ; BSUBA
    39 a        ; A
    40 b        ; B
    41 :)       ; Put anything you want here, this is the one free memory cell.

**NOTE:** As with the factorial program, this uses the trick of using opcodes as constants instead of allocating separate
memory cells for them. Unlike with factorial, in this case this is necessary, or the program wouldn't fit in the available
memory.

## Design

### Goals

The virtual hardware in Shenzhen I/O is reasonably capable, but it's pretty different from traditional CPUs (using Harvard
architecture, in particular), and individual microcontrollers have severe limitations, with the mighty MC6000 being incapable
of swapping the values of its two registers without outside help, for example.

So my primary goal when starting this project was to implement something ostensibly general-purpose and at least approaching
capabilities of first microcomputers from the 1970s. I also wanted to create something that would have at least remotely
usable yet realistic looking UI. Using ROM chips for entering the code then dumping it into RAM on startup probably would
have been an easy way out, but I really wanted to fit numeric LCD and gamepad into this project.

Looking at the final results, I would say that B9900 successfully achieves these goals.

### Initial Plans

B9900 architecture and instruction set are largely something that I had to go with, as my original designs failed one after
another.

I dismissed the idea of using OISC before I even started, because I believed that it would undermine the design's
practicality. OISC designs that I'm aware of typically require three word instructions, and while those instructions do a
lot, writing optimal OISC code seems to be pretty hard, while using standard translations for simpler instructions likely
wouldn't allow implementation even of something as trivial as the factorial program due to memory constraints.

So my original design envisioned using three fairly specialized registers (`acc`, `addr` and `pc`), with the only addressing
mode being indirect using `addr` register. This wasn't a proper load-store design due to small number of planned registers,
so there was an `ADD [addr]` instruction, for example. Having outlined all of this in a text file, I started working on the
first part of the machine.

### RAM system

Surprisingly, this survived all the iterations of my design with only minor changes.

Shenzhen I/O provides a 14-word RAM chip with two separate auto-incrementing address pointers. While it's a fairly capable
piece of virtual hardware, I felt I needed some controller infra to make it easier to use by the planned CPU. I started with...

#### RAM chip controller (MC4000X)

      slx x3 # address bus
      mov x3 acc
      sub 14 # only for Type 2 chips
      mov acc x1
      mov x0 x2 # read cycle
      mov acc x1
      mov x2 x0 # write cycle

The pin layout differs slightly in the three copies for optimization reasons, and one of the copies automatically subtracts
14 from the address received to take some load off the address router, but all the are the same design in essence.

RAM chip controller sleeps until it receives an address on address bus (`x3`), reads the word at that address from the RAM,
sends it on data bus (`x2`) shared by all RAM chip controllers, then awaits for the word to write at the same address on
the data bus. RAM chip controller always performs both a read and a write when given an address. One essential but perhaps
non-obvious function this chip performs is that it allows the memory controller to await for data on the data bus shared by
all RAM chip controllers. This wouldn't have been possible with standard RAM chips directly, as they send the data
immediately whenever someone waits on their data bus.

#### Address Router (MC4000X)

    s:slx x3 # address bus IN
      mov x3 acc
      tlt acc 14
    + mov acc x0 # first RAM chip controller
    + jmp s
      tlt acc 28
    + mov acc x1 # second RAM chip controller
    - sub 28
    - mov acc x2 # next router in line/terminator/third RAM chip controller

Address router takes addresses from the memory controller (`x3`) and redirects them as appropriate to one of its connected
devices. Addresses less than 28 are redirected to RAM chip controllers 1 (`x0`) and 2 (`x1`), while addresses higher than
that are shifted by 28 and sent down the line to `x2`. The device connected on `x2` could be another address router, which,
given sufficient space, allows for utilizing as many RAM controller chips as needed for covering the entire 1000 address
space. In this instance, `x2` is simply connected to another RAM controller chip for a total of 42 words of RAM.

Note that the RAM chip controller on `x1` must be Type 2 (with `sub 14` instruction) as there's no more instruction space
available to do that directly on the address router. Conversely, RAM chip controller on `x0` must be Type 1 (w/o `sub 14`),
and `x2` must be connected either to another address router, or to a Type 1. Early designs included a terminator chip,
intended to handle addresses outside the physically available range, which returned 0 on all reads and ignored all writes,
but I eventually had to ditch that to save space.

#### Memory controller (MC4000X)

      slx x2 # awaits on memory bus
      mov x2 acc
      tlt acc 0
    + mov x2 acc # negative packet head on memory bus, read another address
      mov acc x3
      mov x1 acc
      mov acc x2
    + mov acc x1 # negative packet head indicates that the client doesn't want to write
    - mov x2 x1 # full read-write cycle

The primary purpose of the memory controller is to mux/demux a single memory bus into separate address and data buses
handled down the line by address router(s) and individual RAM chip controllers. Support for optional read-only cycle
was added late in the final stages of design due to the need to optimize CPU implementation.

The controller reads data from the memory bus (`x2`). Negative number indicates that the sender want to perform a short
read-only cycle, and will send address immediately afterwards. Non-negative numbers are interpreted as addresses, and
a full read-write cycle is initiated. Received address is send to the address router through address bus (`x3`), and
the controllers start waiting for data on data bus `x1`. If a read-only cycle was requested, the data received is sent
both to the client on memory bus and back to the RAM chip controller on the data bus. Otherwise the data is sent on the
memory bus, and memory controller awaits for another bit of data to sent back on data bus for write.

### Register Machine?..

...

## Turing Completeness

...

