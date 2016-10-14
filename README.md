# Betelgeuse 9900

Yo dawg! We heard you like Assembly language...

## What's Shenzhen I/O?

[Shenzhen I/O](http://www.zachtronics.com/shenzhen-io/) is a Zach-like puzzle/engineering game by
[Zachtronics](http://www.zachtronics.com/), somewhat similar to their earlier
[TIS-100](http://www.zachtronics.com/tis-100/). Zach Barth strikes again! Zach. (I just thought that maybe there
weren't enough Zachs in this paragraph.)

Shenzhen I/O is currently (as of Oct 2016) in Early Access [on Steam](http://store.steampowered.com/app/504210/).

## B9900 Capabilities & Limitations

Betelgeuse 9900 is a universal, programmable, [von Neumann architecture](https://en.wikipedia.org/wiki/Von_Neumann_architecture)
microcomputer in [1970s style](https://en.wikipedia.org/wiki/Altair_8800)
([TEC-1 is perhaps a closer match](https://en.wikipedia.org/wiki/TEC-1)) implemented in sandbox mode of Shenzhen I/O.
It is very much a [RASP](https://en.wikipedia.org/wiki/Random-access_stored-program_machine), modulo certain ugly
real-world details. It has 42 almost-11-bit words of RAM, simple numeric display and a gamepad. Monitor/debugger is
implemented in "hardware" and uses the gamepad for input.

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

The virtual hardware in Shenzhen I/O is reasonably capable, but it's pretty different from traditional CPUs (using
[Harvard architecture](https://en.wikipedia.org/wiki/Harvard_architecture), in particular), and individual microcontrollers
have severe limitations, with the mighty MC6000 being [incapable](#user-content-fn1) of swapping the values of its two registers
without outside help, for example.

So my primary goal when starting this project was to implement something ostensibly general-purpose and at least approaching
capabilities of first microcomputers from the 1970s. I also wanted to create something that would have at least remotely
usable yet realistic looking UI. Using ROM chips for entering the code then dumping it into RAM on startup probably would
have been an easy way out, but I really wanted to fit numeric LCD and gamepad into this project.

Looking at the final results, I would say that B9900 successfully achieves both of these goals.

### Initial Plans

B9900 architecture and instruction set are largely something that I had to go with, as my original designs failed one after
another.

I dismissed the idea of using [OISC](https://en.wikipedia.org/wiki/One_instruction_set_computer) before I even started,
because I believed that it would undermine the design's practicality. OISC designs that I'm aware of typically require three
word instructions, and while those instructions do a lot, writing optimal OISC code seems to be pretty hard, while using
standard translations for simpler instructions likely wouldn't allow implementation even of something as trivial as the
factorial program due to memory constraints.

So my original design envisioned using three fairly specialized registers (`acc`, `addr` and `pc`), with the only addressing
mode being indirect using the `addr` register. This wasn't a proper load-store design due to small number of planned registers,
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

The controller reads data from the memory bus (`x2`). Negative number indicates that the sender wants to perform a short
read-only cycle, and will send address immediately afterwards. Non-negative numbers are interpreted as addresses, and
a full read-write cycle is initiated. Received address is sent to the address router through address bus (`x3`), and
the controller start waiting for data on data bus (`x1`). If a read-only cycle was requested, the data received is sent
both to the client on memory bus and back to the RAM chip controller on the data bus. Otherwise the data is sent on the
memory bus, and memory controller awaits for another bit of data to sent back on data bus for write.

### Register Machine?..

Next step was implementing register block. As it does not exist in the final design, I won't go into much detail, and
simply give a brief outline of the process and the outcome.

As soon as I started working on the register block I realized that implementing three specialized registers is probably
going to be much more involved than using another standard RAM chip to implement ten more general purpose registers.
Why ten? So that I can offload a large part of ALU duties onto the register block itself. The register block end up
being a fairly capable processing unit in its own right, interpreting a large, orthogonal set of opcodes where two
least significant digits encoded the numbers of source and destination registers. It supported instructions of the
general form `MOV rX + imm, rY` and `ADD rX + imm, rY` with several shortcuts for common special cases. At this point
I changed the planned instruction set to a proper load-store with indirect adressing, as all arithmetics could be done
insude the register block.

Unfortunately, while all of this was aesthetically pleasing etc., once I implemented the monitor/debugger and got down
to implementing the CPU, I realized that this just isn't going to work. Register block took up about as much space as
the RAM system with 28 words. The remaining space on the board was sufficient to place two MC6000s. With fairly rich
instruction set, simple experiments quickly demonstrated that there was no way to fit all the CPU logic into 28 MC6000
instructions.

I thrashed about a bit.

I ditched the idea of doing indirect addressing as the only addressing mode and switched to direct addresses. Hey, this
is a von Neumann arch, who cares! That helped a bit, but nowhere nearly enough. At this point I spent some time in a
pit of engineering despair, but upon considering everything I realized that there was only one choice--register block had
to go.

After a brief detour to discuss the hardware monitor/debugger, I'll finish up the long story of many permutations of the
B9900 arch and instruction set.

### Hardware Monitor/Debugger

This part of the overall design was absolutely useless as far as universality and "usefulness" were concerned, but it was
critical for achieving my second goal--making something that looked like an actual almost-consumer device that could be
used in the real world.

I won't go into much detail on implementation, code can be viewed [in the save file](./prototyping-area-1.txt).

This system consists of standard numeric LCD and gamepad components as well as two MC6000s, one responsible for talking
to memory controller and managing the edit address, and the other talking to gamepad and LCD and handling changes in the
data. Due to the way gamepad works, this part of design actually moves forward in time (the CPU and the rest can
operate in a single time unit). I eventually exploited this to my advantage: while the monitor tells the CPU to start
running by sending a bit through XBus, the CPU transfers the control back to the monitor (upon executing `HALT`) by
simply going to sleep.

I implemented monitor/debugger after finishing with the register block, and as with the memory system it largely stayed
the same through the subsequent permutations and architecture changes. I've run into a few problems down the road with
edit address controller polluting the memory bus despite CPU already running (the memory system design should readily
tell you that no two components should talk on the memory bus at the same time, unless they're coordinating between them
to perform a single read-write cycle), but that was esily if crudely fixable. Part of that involved moving the CPU
boot up from *Start* button key-down to key-up, the rest was reshuffling the way the monitor talked to the memory system
a bit.

As demonstrated in the videos above, this brilliant piece of UX is just about usable enough to input 40-word programs
through it and run them multiple times with different parameters.

### CPU

After the register block debacle I had the understanding that I needed something simpler, and thus the final B9900 design
and instruction set were born. After shuffling around the components on the board and adding another RAM chip (moving from
28 word RAM to 42 word RAM to compensate for mostly 3-word instructions in the new design vs. mostly 2-word instructions
in the old one) I had four spare MC6000s available to implement the CPU.

That turned out to be just about enough.

#### Program Counter (MC6000)

      slx x1 # wait on instruction bus
      mov x1 dat
      tlt dat 0
    + mov -1 x3 # read a word from memory and advance
    + mov acc x3
    + mov x3 dat
    + mov dat x1
    + add 1
    - mov dat acc # jump!

This little beauty simply manages the program counter. Despite the fact that it only has 9 instructions, it has to be an
MC6000, as it needs both registers.

PC listens on instruction bus (`x1`) which is connected the other three MCs constituting the CPU. Negative numbers are
interpreted as requests to read the word at PC and advance the counter. The word is received through memory bus (`x3`)
and sent back through instruction bus. Note that this reads a single word, not the entire instructions. It is the
client's responsibility to ensure instruction boundaries are respected.

Positive input on instruction bus is interpreted as a request for unconditional jump to that address. The PC's internal
state is set to whatever address was supplied. The counter does not advance automatically in this case.

Probably the hardest part of debugging the CPU was solving memory bus contention issues between the PC and the other CPU
components. Thankfully, information provided by Shenzhen I/O allowed to identify these issues easily, but rearranging
memory accessed to eliminate those problem was a whole 'nother story.

#### CPU Main+HALT (MC6000)

      slx x2 # wait for execution signal from the monitor
      mov x2 null
    lp: mov -1 x3
      mov x3 acc
      tgt acc 0 # opcode between 1 and 4?
    + tlt acc 5
    + mov -1 x3
    + mov acc x1
    + mov x1 null
    + jmp lp

The main task of this unit is handling the communications with the monitor, driving the PC (for the most part), and
forwarding opcodes trickier than `HALT` to other CPU sub-units. Note that this controller sends a request for the
second instruction word to the PC through instruction bus, despite the fact that it's the other two sub-units that
are going to be receiving the actual response. This was a necessary optimization to fit everything into the
available space.

`x2` is the bus connected to the monitor, `x3` is the instruction bus, while `x1` is used to communicate with CPU
JLEZ. The rest should be self-explanatory.

#### CPU JLEZ (MC6000)

    st: slx x3 # wait for opcode from CPU Main+HALT
      mov x3 dat
      teq 1 dat
    - mov dat x1 # not a JLEZ, forward to the last processor
    - mov x1 x3
    - jmp st
      mov x2 dat # read the word to be tested through memory bus
      mov -1 x0
      mov dat x0
      tlt x0 1 # is it <= 0?
      mov -1 x2 # read the last word of this instruction through the PC
      mov x2 dat
    + mov dat x2 # tell the PC to jump if the condition holds
      mov 0 x3

Handles the `JLEZ` instruction and forwards other instructions to the final CPU sub-unit.

`x3` is the XBus connected to CPU Main+HALT, `x2` is the instruction bus, `x0` is the memory bus, `x1` links
this unit to CPU ADD/MOV/MUL.

#### CPU ADD/MOV/MUL (MC6000)

  slx x3 # wait for opcode from CPU JLEZ
  tcp x3 3
  mov x2 acc # read the src address from the PC
  mov -1 x0 # read the word at src through memory bus
  mov acc x0
  mov x0 acc
  mov -1 x2 # read the dst address from the PC
  mov x2 dat
  mov dat x0 # initiate read-write for dst through memory bus
  mov x0 dat # receive current word at dst
+ mul dat
- add dat
  mov acc x0 # write the result to dst
  mov 0 x3

Handles the `ADD`, `MOV` and `MUL` instructions. The reason why the opcode for `MOV` is sandwiched between `ADD`'s
and `MUL`'s is that this allows the use of `tcp` here to efficiently decode the instruction.

`x3` is connected to CPU JLEZ, `x2` is the instruction bus and `x0` is the memory bus.

### Final Remarks On Design

The instruction set I ended up with kinda reminds me of [nand2tetris](http://www.nand2tetris.org/), but I'm sure
that's sheer coincidence.

This was a long, winding road, and I did not expect to find myself in this neighbourhood, but in hindsight I probably
should have started small to begin with. I would have preferred to have a more traditional machine on my hands, with
separate large but dumb RAM and a handful of capable, specialized registers, but hey. This is well within the
parameters I established for myself in this project.

## Turing Completeness

So. Is Betelgeuse 9900 Turing complete? By extension, is Shenzhen I/O Turing complete? The answer is, yes, and no, and
also sorta.

Frankly, this question was at best of tertiary interest to me when I started working on this project. Simply because it
is not a very interesting question, in my opinion.

The [proof](https://www.youtube.com/watch?v=sX_G8jtZceg) by Markus Persson that
[Infinifactory](http://www.zachtronics.com/infinifactory/) is Turing complete by construction of
[Rule 110](https://en.wikipedia.org/wiki/Rule_110) processor was amazing. But even then, it wasn't that the fact itself
was particularly surprising. Anyone who's played Infinifactory would quickly get the feeling that Things were Possible.
It's just that actually constructing a Rule 110 implementation in a friggin' game about blocks and stuff is an
astonishing feat of ingenuity, persistence and sheer nerdiness.

The fact that Shenzhen I/O is Turing complete is even less surprising, because it doesn't hide the notion of computation
behind the abstractions (or perhaps instantiations) of blocks and conveyors. It's very much a game that's all about
computation, pure and simple, and the feeling that you can do anything, given space and money, is much, much stronger.

Then again, of course neither Infinifactory nor Shenzhen I/O is *actually* Turing complete. We're not talking about
pure, divine combinatory logic here, mateys. Neither can operate on arbitrarily large data. The same applies to B9900,
of course. In practice, we usually stick to hand-wavy argument that go like, "Oh, but let's consider a slightly
extended impractical model of this computational device that operates on arbitrary precision integers." So we have this
notion of "sorta Turing completeness", which means that some simple generalization is Turing complete in precise sense,
so we assume that our computational device can placidly bask in reflected glory of its bigger brother.

The second aspect is that there's no actual proof that B9900 (real or ideal) is Turing complete, only some evidence in
favour of this. It is widely "known" that "some" arithmetics and reasonably powerful conditional branching are sufficient
for Turing completeness, but personally, I've never seen an actual proof of that. It is also widely "known" that
[SUBLEQ](https://en.wikipedia.org/wiki/One_instruction_set_computer#Subtract_and_branch_if_less_than_or_equal_to_zero)
(along with other OISCs) is Turing complete. Of course, `SUBLEQ a, b, c` is trivially expressible in B9900:

            MOV     a, TEMP
            MUL     MINUS1, TEMP
            ADD     TEMP, b
            JLEZ    b, c
            ...
    MINUS1: DATA    -1
    TEMP:   DATA     0

But again, I haven't been able to track down SUBLEQ's actual proof of Turing completeness. It's either a case of
infinite circular references of esolang to paper to esolang to paper etc., or a road that terminates in $200
paper-only monographs not available through ACM DL. Yeah, no.

So netheir argument consistutes an actual proof in my book.

How about implementing the same ole Rule 110? I believe that this is definitely possible, but it would be an extremely
futile effort. It certainly wouldn't fit in the 42 words of actual B9900, and therefore verifying it would be inordinately
hard.

No conceptual blocks, though, as far as I can tell. Here's the roadmap:

1. Implementing a single step is key, as the rest boils down to repeating that step until some condition holds.
    You could `HALT` every now and then, so that the operator would have to ask the program to continue.
    Alternately, just run until out of memory, it's just a proof of concept, right?
2. States could be represented by arrays of zeroes and ones, and arrays are certainly implementable using makeshift
    indirect addressing.
3. The three neighbouring cells can be easily interpreted as a binary number, which could then be dispatched on,
    or just used as an index in the static lookup table.
4. Infinite periodic background pattern might be the biggest problem conceptually, but I don't see any hard problems
    in writing a function that would yield the state of the pattern at any point, which could then be used whenever
    we need a value from outside the state we're actually keeping.

It's another long road, likely more winding than the road to B9900 was, and to me, far less interesting. Following it
is left as an exercise for a rigour-oriented reader.

### tl;dr

The summary is that, sure, B9900 can compute some stuff, like small factorials or GCDs. It's not lacking general
recursion, it's just lacking memory. Give me a larger board and more memory, and it will be able to compute even more
stuff. No reason why we couldn't stuff merge sort into a thousand words. If we consider an extended, ideal model using
arbitrary precision integers, and having an inexhaustible source of on-demand memory, why, it's blindingly obvious
that it would be Turing complete, even though the formal proof does not exist.

<a name="fn1">1</a> I'm gonna slap anyone who says, "Oh, but you can swap two integers..." here. If you don't understand
why you really can't, *slap*.

