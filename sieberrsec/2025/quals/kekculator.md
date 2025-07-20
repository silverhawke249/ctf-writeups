# KEKculator

## The problem

A compiled Python file is given that implements a VM. The input bytecode is known, and we need to
read the `flag` file that resides in the same directory.

## The observations

`.pyc` files can easily be thrown into a decompiler, so I did that first. The resulting file is
still obfuscated, so I used VSCode to refactor these so the VM class is readable.

From the helpfully named method `start()` as well as the `__init__()` function, I note that the
bytecode is read and loaded into the VM's memory, starting from address 1000. There is no
separation between data and code in memory, so there is already the potential of writing our own
bytecode via some sort of buffer overflow.

Let's unravel `start()` first. Each address in the memory is supposed to hold a single byte, so
assuming this constraint holds, it will read four 32-bit values from the current instruction pointer,
indicated as EIP, as an unsigned integer. EIP will then advance by 16 bytes.

These four `uint32`s are opcode, `arg1`, `arg2`, and `arg3`, respectively. There are a handful of
opcodes available in this VM, which I list in the table below (note that I'm not super familiar with
opcode naming conventions, so I just named them as what I thought they did)

| opcode | operation  |
|--------|------------|
| 0x00   | exit       |
| 0x01   | add        |
| 0x02   | difference |
| 0x03   | multiply   |
| 0x04   | floordiv   |
| 0x05   | compare    |
| 0x06   | jump_eq    |
| 0x07   | jump_neq   |
| 0x08   | jump_gt    |
| 0x09   | jump_lt    |
| 0x0a   | jump_eq2   |
| 0x0b   | jump_neq2  |
| 0x0c   | jump       |
| 0x0d   | store      |
| 0x0e   | load       |
| 0x0f   | write      |
| 0xff   | nop        |
| 0xdd   | io         |

The `jump` opcodes, except for 0x0c, are to be used after using the `compare` opcode, since these work
off the EAX register.

The `store` and `load` opcodes are counterparts of each other -- these are supposed to handle 32-bit
integers on a stack that is maintained by the ESP register. The implementation of `store` actually allows
for storing integers larger than 32-bit, which will overflow to the higher memory addresses, but this is
not used, because the `write` opcode does pretty much the same thing, but it allows writing the value of
any register to anywhere in memory.

Most interestingly, the `io` opcode does three things, depending on the value of the ARG1 register:
* If ARG1 is 0, it will print a null-terminated string from a memory address, which is read from a
  register specified by ARG2.
* If ARG1 is 1, it will read 4 bytes from standard input, convert it into a 32-bit integer, and store
  it to a register specified by ARG2.
* If ARG1 is 2, it will read a file. The file name is a null-terminated string read from a memory address,
  read from a register specified by ARG2, and its contents will be stored to the memory address, read from
  a register specified by ARG3.

Next, let's see what the bytecode does. The following pseudocode shows what the bytecode does when
fed into the VM:

```
print "Welcome to KEKulator PRO!"
print "Your starting number: 1!"
print "This is a blackbox so I won't tell you what to do...teehee"
ECX <- 1
repeat
    EDX <- read 4 bytes input
    if EDX == "sub "
        EDX <- read 4 bytes input
        ECX <- difference(ECX, EDX)
    elseif EDX == "mul "
        EDX <- read 4 bytes input
        ECX <- multiply(ECX, EDX)
    elseif EDX == "div "
        EDX <- read 4 bytes input
        ECX <- floordiv(ECX, EDX)
    elseif EDX == "done"
        break
    else
        EDX <- read 4 bytes input
        ECX <- add(ECX, EDX)
write(ECX, 116)
ECX <- 116
print from ECX
```

If you read the bytecode as well, the else branch might look weird, since that branch should've
been executed when EDX contains `"add "`. However, the bytecode does not jump back to the input
instruction if all the tests fail. Hence, if all tests fail, the `add` subroutine will be executed
anyway, which means in higher level code, it will be put under the else branch. This does not
affect the solution, though.

The important bit of this code is the `write` instruction, which is at offset 0x5D0 in the provided
bytecode. This will write the contents of ECX into memory with no bounds checking, which means it's
possible to overflow into executable area, since it is placed after the 1000 bytes of memory provided
by the VM. Therefore, we just need to manipulate ECX via these operations to place a payload which
will be executed by the VM to read the flag and print it to standard output.

## The execution

First, let's craft the payload. My approach was as follows:
* Place "flag" at known address.
* Store said address in a register.
* Read from file "flag" and store the contents in a register.
* Write the contents of that register to some memory address.
* Print null-terminated string from that memory address.

To make my implementation straightforward, I write a word (32-bit integer) at a time, by performing
two multiplications with 0x10000 and adding the relevant value. This is due to the limitation that
we can only operate with 4 bytes at a time.

Since we know the memory will be written to at address 116 (0x74), I start by adding "flaf" (remember
that ECX already has 1 in it!). That takes up 4 bytes, so I need to write 1000 - 120 = 880 bytes to
exhaust the memory. I did this by repeated multiplications with 0x100.

Next, let's write the bytecode corresponding to the operations that we want to do:

```
# Store 116 (0x74) to ECX
00 00 00 01 00 65 63 78 00 00 00 74 00 00 00 00
# Read from file specified by ECX, store to EBX
# By the time we reach this code, EBX isn't modified anymore so this is OK
00 00 00 DD 00 00 00 02 00 65 63 78 00 65 62 78
# Write EBX to address 0x74
00 00 00 0F 00 65 62 78 00 00 00 74 00 00 00 00
# Print address specified by ECX (still 0x74)
00 00 00 DD 00 00 00 00 00 65 63 78 00 00 00 00
# Terminate program
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

This should work, but there is an apparent problem: at 0x5E0, there's only room for three instructions,
and then the VM runs out of memory. This is not an issue however, as we can just write this earlier
in memory and write an unconditional jump to the payload:

```
# We have 5 instructions, so write it to 0x590
# So this jump will be at 0x5E0, and we jump to 0x590
# Note that the jump address is offset by 1000! 0x590 + 1000 = 0x978
00 00 00 0C 00 00 09 78 00 00 00 00 00 00 00 00
```

I perform more multiplications to pad the result until we hit offset 0x590. Then, using multiplications
and additions, I write the payload and send `done`! The program prints the flag and exits gracefully.

## The hindsight

I realized some things after the CTF ended:
* Multiplying by 0xffffffff would blow ECX up faster -- this is necessary to hit the 700 operations
  challenge posed by the author.
* Technically, once I reach executable memory, I can do whatever I want -- placing "flag" in memory
  may not be necessary.

Reflecting on this, "flag" can be placed much later in memory, as long as we know its address.
The payload doesn't really need to be modified much, just the address to "flag" that needs to be
modified.

Let's try padding the memory until we reach just before 1000 + 0x590 via repeated multiplications
of 0xffffffff. Treating these bytes as base 256, that means we need no more than 1000 + 0x590 - 0x74 = 2308 digits.
In other words, solving for `floor(n log256(0xffffffff)) + 1 <= 2308`... and we see that 0xffffffff ^ 577
does take up 2308 bytes, of which the last four are 0xffffffff -- makes sense since that's basically
`(-1) ^ 577 mod 0x100000000`.

We can either subtract "flag"'s one's complement from this, or add "flag" + 1 ("flah"). Finally,
we work in groups of 7 bits -- so the byte code looks like this... (note that "flag" is now at
address 1000 + 0x590 - 4 = 2420 = 0x974)

```
0000000 1006563 7800000 9740000 0000000 000DD00
0000020 0656378 0065627 8000000 0F00656 2780000
0974000 0000000 0000DD0 0000000 0065637 8000000
0000000 0000000 0000000 0000000 0000000 000000C
0000097 8000000 000000
```

Each of these groups of 7, except for the last one, takes two operations to insert, so we need 53 operations --
a total of 578 + 53 = 631 operations!
