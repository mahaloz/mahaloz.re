---
title: "Reversing printf-as-a-VM service in Google Quals 2020"
classes: wide
layout: post
tags: [ctf]
description: "Solving a virtual machine implemented inside format strings found in the printf library in C with @kylebot."
---

Solving a virtual machine implemented inside format strings found in the printf
library in C.

<!--more--> 

## An overview
This writeup is based on the `sprint` challenge in google quals 2020. 
You can find all the challenge files, including our solve, [here](https://github.com/mahaloz/mahaloz.re/tree/master/writeup_code/google-quals-20).
In this challenge, Google introduced us to a new type of instruction set, which in turn allowed us to play a video game completely virtualized in the C language's `sprintf` format strings. 
Through some reverse engineering and IDA finagling, we created a disassembler for the sprint arch, which is the name we are declaring for this challenges [ISA](https://en.wikipedia.org/wiki/Instruction_set_architecture).

![Winning!]({{ site.baseurl}}/assets/images/solving_maze.gif)

We break this writeup into sections:
- [An overview](#an-overview)
- [Investigating the Program](#investigating-the-program)
  - [A brief review of format strings:](#a-brief-review-of-format-strings)
- [Instruction Lifting](#instruction-lifting)
  - [Syntax Parser](#syntax-parser)
  - [Semantic Parser](#semantic-parser)
- [IR Optimization: Making Readable Assembly](#ir-optimization-making-readable-assembly)
- [Decompilation, Execution, and Luck](#decompilation-execution-and-luck)
- [Finding a Game and Beating It](#finding-a-game-and-beating-it)
- [Thanks where thanks is due](#thanks-where-thanks-is-due)
- [Conclusion](#conclusion)

## Investigating the Program
Let's start by investigating the program in our disassembler of choice. We are lucky enough to have IDA Pro, but this should be doable in Ghidra. 

![investigate all the binaries](https://media.giphy.com/media/42wQXwITfQbDGKqUP7/giphy-downsized.gif)

The challenge gives us a very "simple" program that only does four things:
1. mmap a region and copy some data into it
2. read user input to the mmaped region
3. run `sprintf` in a loop with a weird exit condition
4. if part of the mmaped region satisfies some constraints, output that region as the flag

By some close examination, we found out that the "some data" in step 1 is actually a huge array of format strings.
At this point, we realized that this is a VM based on `sprintf`. And each format string is an "instruction".
To clarify, we believed that based on the normal operations of `sprintf` and format strings, one could achieve a [Turing Complete](https://en.wikipedia.org/wiki/Turing_completeness) machine that allowed you to perform all the normal operations in a computer like addition, storing data, and accessing data. 

Here is a sample of the aforementioned format strings that we found:
```
.rodata:0000000000002020                 public data
.rodata:0000000000002020 ; char data[61748]
.rodata:0000000000002020 data            db '%1$00038s%3$hn%1$65498s%1$28672s%9$hn',0
.rodata:0000000000002020                                         ; DATA XREF: main+46↑o
.rodata:0000000000002046 a100074s3Hn1654 db '%1$00074s%3$hn%1$65462s%1$*8$s%7$hn',0
.rodata:000000000000206A a100108s3Hn1654 db '%1$00108s%3$hn%1$65428s%1$1s%6$hn',0
.rodata:000000000000208C a100149s3Hn1653 db '%1$00149s%3$hn%1$65387s%1$*8$s%1$2s%7$hn',0
.rodata:00000000000020B5 a100183s3Hn1653 db '%1$00183s%3$hn%1$65353s%1$1s%6$hn',0
.rodata:00000000000020D7 a100218s3Hn1653 db '%1$00218s%3$hn%1$65318s%1$2s%11$hn',0
.rodata:00000000000020FA a100264s3Hn1652 db '%1$00264s%3$hn%1$65272s%1$*10$s%1$*10$s%17$hn',0
.rodata:0000000000002128 a100310s3Hn1652 db '%1$00310s%3$hn%1$65226s%1$28672s%1$*16$s%7$hn',0
```

The `sprintf` loop looks like this:
```
  while ( format != &ws[1].formats[6137] )      // A3 != 0xfffe
    sprintf(
      buffer,
      format,
      &nullptr,                                 // A1
      0LL,                                      // A2 == 0
      &format,                                  // A3
      buffer,                                   // A4
      (unsigned __int16)*A6,                    // A5
      A6,                                       // A6
      &A6,                                      // A7
      A8,                                       // A8
      &A8,                                      // A9
      A10,                                      // A10
      &A10,                                     // A11
      A12,                                      // A12
      &A12,                                     // A13
      A14,                                      // A14
      &A14,                                     // A15
      A16,                                      // A16
      &A16,                                     // A17
      A18,                                      // A18
      &A18,                                     // A19
      A20,                                      // A20
      &A20,                                     // A21
      A22,                                      // A22
      &A22);                                    // A23
```
When taking a look at the variables accessed and used by the initial sprint we realized that these variables are probably used as registers for this architecture. The comments on the right of the variables is the standard names we used to reference the variables, and eventually translate into higher level registers -- like the program counter.

Since the remainder of this writeup will be concerning format strings, it is useful to do a quick overview of the format specifiers relevant to this challenge. 

### A brief review of format strings:
The relevant specifiers for this challenge are `%n`, `$`, `%h`, `%c`, and `*`:
* `%c` specifies to print a character, or a single byte. This includes null bytes. 
* `%h` specifies the size of the print of the buffer to 2 bytes. This means everything with this specifier will print 2 bytes (a cunning way to always move 2 bytes). 
* `$` is a specifier to get the `n`th argument of the `printf`. For instance `3$` gets the third buffer relative to the printf.
* `%n` is a specifier that allows you to write to a buffer that is used in the `printf`. It will write to the buffer the number of charcters printed before the `%n`. An example can be found [here](https://www.geeksforgeeks.org/g-fact-31/).
* `*` is the trickest specifier for this challenge. The `*` is a dynamic specifier that determines the size of printing based on the next argument with an integer. For instance, if the next argument is `5`, it will print something with a size of `5` -- or 5 charcter slots. An example can be found [here](https://en.wikipedia.org/wiki/Printf_format_string#Width_field) also.

With this knowledge we can parse the first part of the first format string: `%1$00038s%3$hn`.
The `1$` accesses the first argument; the `00038s` prints a string that is of length 38, right formatted. The `%3$hn` then writes the value `38` formatted to 2 bytes to the third argument to `sprintf`. In conclusion, this instruction moved the value `0x0026` to register `A3`. 

## Instruction Lifting
Although we still had no idea how the VM worked at this point, we decided to write a lifter for the format string instructions and translate it into a human readable assembly. We needed to do some [lifting](https://reverseengineering.stackexchange.com/questions/12460/lifting-up-binaries-of-any-arch-into-an-intermediate-language-for-static-analysi)!

![lifffffffffft](https://media.giphy.com/media/3oriNZoNvn73MZaFYk/giphy.gif)



As noobs in reversing, we decided to do this in two overarching steps: a **syntax parser** and a **semantic parser** -- a similar path that a normal compiler takes.

### Syntax Parser
The syntax parser simply parses the format strings and gets information like what formatter it uses, which argument it takes, etc. For example, `%1$00038s` will be parsed as: `arg_num=1, con_len=38, formatter='s', len_arg_num=None` where `con_num` is the concrete length. `len_arg_num` is used for format string like `%1$*16$s` which takes the 16th argument as the length parameter. 

During writing the syntax parser, we realized that the program constantly writes to the third parameter (we named it `A3` as shown above). We guessed it was something like a program counter (PC). It turned out we were right.

### Semantic Parser
The semantic parser is built upon the syntax parser. It takes the output from the syntax parser and then translates it into an assembly style [Intermediate Representation](https://www.sciencedirect.com/topics/computer-science/intermediate-representation) (IR). As explained earlier, `%1$00038s%3$hn` writes 38 to `A3`. So, the semantic parser outputs `A3 = 38`.

During writing the semantic parser. We found a lot of interesting things:
1. each "instruction" may consist of 1 or 2 "micro-ops". For example, `%1$00038s%3$hn%1$65498s%1$28672s%9$hn` will be translated into `A3 = 0x26, A9 = 0x7000`
2. there is an amazing conditional jump instruction. Basically, it uses that fact that `%c` can output `\x00`. If the argument is equal to 0, it outputs `\x00` to the internal buffer, the internal buffer contains a shorter string. If the argument is not equal to 0, it outputs a valid character and the internal buffer contains a longer string. And then it writes the length of the internal string to `A3`. By carefully controlling the lengths of different strings, this mechanism can behave like `if-else` statement.
3. each general "register" has two handles: a read handle and a write handle. This has nothing to do with the VM architecture. It's just an easy way to perform read/write in `sprintf`. As shown above, `A11` actually is the write pointer to `A10`.
4. loop exists in the VM. What's worse, by reading the IR, we quickly identified the existence of nested loops. 
![oh no!](https://media.giphy.com/media/6yRVg0HWzgS88/source.gif)

A sample IR looks like this:
```
---- 0 ----
A3 = 0x26
A9 = 0x7000
-------------

---- 1 ----
A3 = 0x4a
A7 = max(A8, A1)
-------------

---- 2 ----
A3 = 0x6c
A6 = 0x1
-------------

---- 3 ----
A3 = 0x95
A7 = max(A8, A1) + 0x2
-------------

---- 4 ----
A3 = 0xb7
A6 = 0x1
-------------

---- 5 ----
A3 = 0xda
A11 = 0x2
-------------

---- 6 ----
A3 = 0x108
A17 = max(A10, A1) + max(A10, A1)
-------------

---- 7 ----
A3 = 0x136
A7 = 0x7000 + max(A16, A1)
-------------

---- 8 ----
A3 = 0x15b
A15 = max(A5, A1)
-------------

---- 9 ----
A3 = COND_JUMP_A14 + 0x1a3 + 1 + A4 + 0xffdb
-------------
```

## IR Optimization: Making Readable Assembly
Now that we have a relatively readable IR, what's next? Optimization of course.

This IR is far from optimized. For example, it shows `A3 = <num>`; however, what does it mean? In fact, it overwrites the last two bytes of the format string pointer so in the next `sprintf` loop, another format string will be processed. Basically, `A3` is the program counter! We realized it would be nice if we could show it like `PC = 5` for basic blocks instead of weird `A3 = 0xb7`.

There are many ways to achieve this. The most obvious way is to modify how we emit this IR. But personally, we hate this method, it just makes thing dirty.
we chose to do an LLVM-pass style optimization. It takes the raw IR and perform optimization pass-by-pass.

We implemented several passes. After the optimization, the optimized IR looks like this:
```
---- 0 ----
PC = 1                                  # A3 = 38
A8 = 0x7000
-------------

---- 1 ----
PC = 2                                  # A3 = 74
ptr = A8
-------------

---- 2 ----
PC = 3                                  # A3 = 108
*ptr = 0x1
-------------

---- 3 ----
PC = 4                                  # A3 = 149
ptr = A8 + 0x2
-------------

---- 4 ----
PC = 5                                  # A3 = 183
*ptr = 0x1
-------------

---- 5 ----
PC = 6                                  # A3 = 218
A10 = 0x2
-------------

---- 6 ----
PC = 7                                  # A3 = 264
A16 = A10 + A10
-------------

---- 7 ----
PC = 8                                  # A3 = 310
ptr = 0x7000 + A16
-------------

---- 8 ----
PC = 9                                  # A3 = 347
A14 = *ptr
-------------

---- 9 ----
PC = 10 if A14 == 0 else 21             # A3 = 384 if A14 == 0 else 804
-------------
```

## Decompilation, Execution, and Luck
At this stage, we had a highly readable IR. Hardcore reversers would be happy enough with it and figure out the logic pretty quickly. However, as reversing noobs, we chose another approach: translate it to x86_64 assembly, open it in IDA and get it's IDA decompilation.
The idea is based on the fact that control flow transition is done by setting `PC`. We can simply add a label for each basic block and translate `PC = 1; A8 = 0X7000` to `di = 0x7000; jmp label1`. The idea is nasty, but it worked pretty well. IDA is smart enough to understand the function even if there are many `jmp` instructions in it.

The moment we pressed F5 (decompile), our hearts stopped... It worked!
The outputted decompilation was pretty readable for the most part:

```c
  v0 = sys_mmap(0LL, 0x10000uLL, 3uLL, 0x32uLL, 0xFFFFFFFFFFFFFFFFLL, 0LL);
  v1 = sys_read(0, (char *)0xE000, 0x100uLL);
  HIWORD(v2) = 0;
  v3 = 28674LL;
  MEMORY[0x7000] = 1;
  MEMORY[0x7002] = 1;
  v4 = 2;
  do
  {
    LOWORD(v3) = 2 * v4 + 28672;
    if ( !*(_WORD *)v3 )
    {
      for ( i = 2 * v4; ; i += v4 )
      {
        LOWORD(v3) = -17;
        *(_WORD *)v3 = i;
        LOWORD(v3) = -16;
        if ( *(_WORD *)v3 )
          break;
        LOWORD(v3) = 2 * i + 28672;
        *(_WORD *)v3 = 1;
      }
    }
    ++v4;
  }
  [truncated]
```

Most amazingly, with some prologue, epilogue and most importantly some luck, this binary actually runs with `mmap_min_addr` set to 0(due to the fact that the VM uses address 0).

## Finding a Game and Beating It
![Video Game!](https://media.giphy.com/media/1lAJ9qQXW3NFecqT25/giphy.gif)

At this stage, the flag is not far from reach. We had extracted the program on the inside of this `sprintf` VM and we now had a way to execute, debug, and understand it. Quickly, we noticed that this was a game based on this code:

```c
if ( j != (__int16)0xFF02 )
    goto label143;
  LOWORD(v2) = 0;
  v7 = 0;
  LOWORD(v3) = -3840;
  v8 = *(_WORD *)v3;
  v9 = 1;
  while ( 1 )
  {
    LOWORD(v3) = v2 - 0x2000;
    v10 = *(_WORD *)v3;
    if ( !*(_WORD *)v3 )
      break;
    LOWORD(v2) = v2 + 1;
    switch ( v10 )
    {
      case 'u':
        v11 = -16;
        break;
      case 'r':
        v11 = 1;
        break;
      case 'd':
        v11 = 16;
        break;
      case 'l':
        v11 = -1;
        break;
      default:
        v9 = 0;
        v11 = 0;
        break;
    }
    [truncated]
```
We took special notice that it was interpreting up, down, left, right and changing our coordinates with each move. We realized the coordinates were in the form of a byte. So `0x12` translated to `(2,1)` on a normal (x,y) plane where the top right of the gird is `(0,0)`. You can verify this based on the subtraction of 16 when moving up, and the addition of 1 when moving right -- this is to access the high and low components of the byte. 

Continuing our examination, we realized that this game was a maze. Each byte of user input is interpreted as a move. We need to play the game and reach 9 checkpoints in a specified sequence within 254 steps. We came to this conclusion based on the final check in the game:

```c
if ( v9 && v7 == 9 )
```
Which does a check to see if you made it to 9 checkpoints. It then confirms this by making sure you hit each check point in the right order. The map of the maze is generated during runtime. To get it, hardcore reversers may read the nested loop logic and figure it out. We noobs with a binary in hand can attach to a gdb and dump the memory to figure it out. The map dumped to this:

```c
****************
* *     *       
* * ***** ******
*       * * *   
* ***** * * * **
* * *    
*** *** *******
*   *   *   * 
* ***** *** ****
*       * *   *
* * ***** * ***
* * * * *
* *** * * * ****
*         *
*** * ***** * **
*   * *     *   
```
The positions that we need to visit is actually embedded in the data section of the raw `sprint` binary at `0x11020`. After extracting that, we were left with a maze that had checkpoints and our starting location (marked with an 'X').

```c
****************
*X*     *      9
* * ***** ******
*       * * *  6
* ***** * * * **
*3*5*
*** *** *******
*   *8  *   *1
* ***** *** ****
*       * *   *
* * ***** * ***
* * * *4*
* *** * * * ****
*         *
*** * ***** * **
*7  * *     *  2
```

Putting everything together, we wrote a python script to play the game. Clever reversers may write a DFS algorithm to solve the maze automatically. However, we are noobs, noobs know nothing about automation. We wrote an interpreter for the game and solved it by hand.

Our winning movements looked something like this:

![Winning!]({{ site.baseurl}}/assets/images/solving_maze.gif)

## Thanks where thanks is due
This challenge was in no way a one man effort, as you can likely tell from the writing style. This challenge would not have been possible without [kylebot](http://kylebot.net/about/) on Shellphish. We struggled through the night, and he played a huge part in the chall (much more major than I). Also, shoutout to any other people who jumped in the Shellphish Discord and gave their two cents.

## Conclusion
This was a very very fun challenge to do. Now we know `even printf is turing complete`. The most impressive part to us is the `if-else` instruction. That was mind-blowing to us when we figured it out. I don't have any complaints about this challenge. The experience was flawless and the maze was quick enough to be solvable. If you too are reversing noob, know that even you can solve a hard challenge if you put in the time :). 
