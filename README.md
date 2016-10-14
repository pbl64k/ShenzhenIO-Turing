# Betelgeuse 9900

These notes are WIP, stay tuned.

## What's Shenzhen I/O?

Shenzhen I/O is a Zach-like puzzle/enginerring game by Zachtronics, somewhat similar to their earlier TIS-100. Zach Barth
strikes again! Zach. (I just thought that maybe there weren't enough Zachs in this paragraph.)

Shenzhen I/O is currently (as of Oct 2016) in Early Access on Steam.

## Spec

Betelgeuse 9900 is a universal, programmable, von Neumann architecture microcomputer in 1970s style. It is very much
a RASP, modulo certain ugly real-life details. It has 42 almost-11-bit words of RAM, simple numeric display and
a gamepad. Monitor/debugger is implemented in "hardware" and uses the gamepad for input.

## In Action!

- [Factorial](https://www.youtube.com/watch?v=8L4xBvnxPUQ)
- [GCD](https://www.youtube.com/watch?v=9reU8p3pkF4)

## In-Game Cost

| Part      | Number | Cost Per Unit | Total |                                                           |
| --------- | ------:| -------------:|------:| --------------------------------------------------------- |
| MC6000    | 6      | ¥5            | ¥30   | CPU1-3, PC, Mon/Dbg1-2                                    |
| MC4000X   | 5      | ¥3            | ¥15   | memory controller, address router, RAM chip controller x3 |
| 100P-14   | 3      | ¥2            | ¥6    | RAM chips                                                 |
| LX700     | 1      | ¥4            | ¥4    | Numeric LCD                                               |
| N4GP-1000 | 1      | ¥2            | ¥2    | Gamepad                                                   |
| Total     | 16     |               | ¥57   |                                                           |

## Instruction Set

| Opcode    | Mnemonic        | Description                                                            |
| ---------:| --------------- | ---------------------------------------------------------------------- |
| 1         | `JLEZ tst, tgt` | If the word at `tst` is less than or equal to zero, jumps to `tgt`     |
| 2         | `ADD src, dst`  | Sums the words at `src` and `dst` and writes the result to `dst`       |
| 3         | `MOV src, dst`  | Writes the word at `src` to `dst`                                      |
| 4         | `MUL src, dst`  | Multiplies the words at `src` and `dst` and writes the result to `dst` |
| any other | `HALT`          | Stops execution and transfers control to hardware monitor/debugger     |

`HALT` is a single-word instruction, all other instructions consist of three words--opcode followed by two addresses.

Words are decimal integers in -999..999 range, as dictated by the underlying "chips."

## Operation

B9900 starts in monitor/debugger mode, with edit address pointing at zeroth word. Use up and down keys on the gamepad
(`w` and `s` on actual keyboard) to move the edit address around. Change the value at edit address by using left and
right keys on the gamepad (`a` and `d` on keyboard). "A" button (`j`) increases the value at edit address by ten. "Start"
button (`Enter`) executes the program. If the program terminates by executing the `HALT` instruction, pressing "Start"
again will continue execution from the next address after `HALT`.

## Factorial

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

## GCD

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

## Design

...

## Turing Completeness

...
