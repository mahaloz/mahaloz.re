---
title: "Reversing a game implemented in a rom-to-eng translator in HackTM 2020"
classes: wide
layout: post
tags: [ctf]
description: "Solving a video game wrapped inside an obfuscated translation engine."
---

Solving a video game wrapped inside an obfuscated translation engine. 

<!--more-->


## Challenge Description:
> "The binary is still in development. The final version would have let you play
> boggle"

All the files for this chall, including my solve, can be found [here](https://github.com/mahaloz/mahaloz.re/tree/master/writeup_code/hacktm-20).

## Introduction
After the first 24 hours of HackTM-20 playing with pwndevils, I was finished working 
on all the "Bear" reversing challenges with @fish. I decided it was time to confront 
my fears and finally reverse a Rust binary.

## Overall Understanding 
Within the first few seconds of getting this binary we do a simple file on the
two files provided:
```bash
$ file hackdex hacktm.hdex
hackdex: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/l, for GNU/Linux 3.2.0, BuildID[sha1]=f97d38c4cea7bde09bc0e6367820b33a2ed8f7b6, not stripped

hacktm.hdex: UTF-8 Unicode text, with very long lines
```

Luckily it's an ELF. Next we do a simple `strace` on the binary to understand
how this binary is interacting with the provided "plaintext" file.

```bash
$ strace ./hackdex
[truncated]
read(3, "fitoare\nyeoman<br>razes; soldat "..., 8192) = 6861
read(3, "", 8192)                       = 0
close(3)                                = 0
write(1, "Welcome to HackDex. Your EN-RO d"..., 112Welcome to HackDex. Your EN-RO dictionary for HackTM CTF!


    Options:
    1. Translation console
    *. Exit
) = 112
write(1, "hackdex> ", 9hackdex> )                = 9
brk(0x5573d4ea1000)                     = 0x5573d4ea1000
```

We get overwhelmed with output since it seems that the binary is reading in the
ascii file line by line (and it has a lot of lines):

```bash
$ cat hacktm.hdex | wc -l
52305
```

Playing with the binary we found out we have a translation program that converts
English to Romanian by doing a table lookup in the ascii file. It's easy to see
this is true since they put a delimiter `<BR>` between each English and Romanian
translation -- also, the entire file was loaded into memory earlier. Time to
jump into this binaries assembly code! 

## Discovering important functions 

After about 0.01 seconds of looking at this binary in any decompiler, IDA Pro in
my case, we see that this binary was compiled from the language Rust. That is
unfortunate, seeing that rust includes all it's library's function code in the
binary -- making our binary bloated as hell.

```bash
$ readelf --syms hackdex | wc -l
2599
```

Yeah we have around 2599 functions in this binary, so we want to figure out
where to start static analysis by seeing where this binary does the important
things in dynamic analysis. This is also possible by following calls to `main`.

```bash
$ gdb ./hackdex
[truncated] 
Reading symbols from ./hackdex...(no debugging symbols found)...done.
gef➤  r
Starting program: /home/mahaloz/hacktm-20/hackdex/hackdex
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Welcome to HackDex. Your EN-RO dictionary for HackTM CTF!


    Options:
    1. Translation console
    *. Exit
hackdex> ^Z

[truncated] 
──────────────────────────────────────────────────────────────── trace
[#0] 0x7ffff7af4081 → __GI___libc_read(fd=0x0, buf=0x555557055e20, nbytes=0x2000)
[#1] 0x5555555c9e34 → <std::io::buffered::BufReader<R> as std::io::BufRead>::fill_buf()
[#2] 0x5555555cb086 → std::io::stdio::Stdin::read_line()
[#3] 0x555555560268 → dex::get_input()
[#4] 0x555555560518 → dex::translate()
[#5] 0x555555562ea0 → dex::main()
[#6] 0x555555563a43 → std::rt::lang_start::{{closure}}()
[#7] 0x5555555d13a3 → std::panicking::try::do_call()
[#8] 0x5555555d36a7 → __rust_maybe_catch_panic()
[#9] 0x5555555d1d80 → std::rt::lang_start_internal()
```

Causing a keyboard interrupt in the middle of translation allows us to see some
basic information about how the binary is getting it's translations. We can see
from the backtrace that the main package/class in this binary is the `dex`
class. We can see the important functions `dex::translate()`, `dex::main()`, and
`dex::get_input()`. Let's take a look into `dex::main` function. 

Taking a look at offset `0xEDD5`:

```nasm
.text:000000000000EDD5                 mov     rax, qword ptr [rsp+768h+var_748+8]
.text:000000000000EDDA                 cmp     rax, 1
.text:000000000000EDDE                 jz      loc_EE93
.text:000000000000EDE4                 cmp     rax, 9
.text:000000000000EDE8                 jnz     loc_EF56
.text:000000000000EDEE                 lea     rdi, [rsp+768h+var_720]
.text:000000000000EDF3                 call    _ZN3dex5extra17hc0d1779f7950f2c7E ; dex::extra::hc0d1779f7950f2c7
```

We see a data dereference loaded into `rax` and then compared against 9. If it
is 9 then we enter the elusive `extra` function. Without 100% knowing where that
data references, we just try a simple test:

```bash
$ ./hackdex
Welcome to HackDex. Your EN-RO dictionary for HackTM CTF!


    Options:
    1. Translation console
    *. Exit
hackdex> 9
Only PRO users!
```

Yup, we found the secret mode of the binary. Recall the hint said we are dealing
with a program that was not finished being developed, so we can assume that this
area was not meant to be accessed yet. Our new target is the `hex::extra`
function. Taking a look at it's internals we see offset `0xC6C4`:

```nasm
.text:000000000000C6C4                 cmp     cs:_ZN3dex11PRO_VERSION17h62387400d9c485dbE, 1337h ; dex::PRO_VERSION::h62387400d9c485db
.text:000000000000C6CF                 jz      loc_C88C
```

This causes us to leave the function. Assuming that we will never be able to
satisfy this with normal user input, we can just hex patch this section out by
modifying the `jz` to a `jnz`. Now we can get to the pro mode:

```
$ ./hackdex
Welcome to HackDex. Your EN-RO dictionary for HackTM CTF!


    Options:
    1. Translation console
    *. Exit
hackdex> 9
hackdex(9)>
```

## How do we get the flag? 

Once we enter the PRO Mode, we can vaguely see that this is the right path to a
flag since we see `"HackDex!\n\n===> GG!"` at offset `0xE453` -- of course since
this is rust, strings are not null terminated. From here on, it may be helpful to 
see the decompilation, so I will assume you have IDA Pro. If you don't, Ghidra 
will have similar results for these sections. We continue!

Executing the binary in the pro-mode, we see a rejection for input:

```bash
$ ./hackdex.bak
Welcome to HackDex. Your EN-RO dictionary for HackTM CTF!


    Options:
    1. Translation console
    *. Exit
hackdex> 9
hackdex(9)> Password
Incorrect!
```

At offset `0xC813` we see the use of `dex::get_input()`:
```c
  dex::get_valid_words::hd122c9be5dc18ee7((__int64)&valid_words_1, 4LL, v10);
  dex::get_input::h4a48495279fb4e49(...) 
```

Along with the `get_input()`, we see that a `get_valid_words` is called with
each instance of the `get_inputs` (for almost all of them). If this is the
validation, we can assume we need to get valid words to get the flag.

We see the output of the valid words used in the `contains_key` map function for
a series of condition checks:

```c
if ( (unsigned
__int8)hashbrown::map::HashMap$LT$K$C$V$C$S$GT$::contains_key::hfb0cbca595abaea2(...
```

Using a similar method to patching the pro-edition check we can make this check
pass each time. After patching we try the test again:

```bash
$ ./hackdex
Welcome to HackDex. Your EN-RO dictionary for HackTM CTF!


    Options:
    1. Translation console
    *. Exit
hackdex> 9
hackdex(9)> pwndevils
Correct!
hackdex(9)> pwndevils
Correct!
hackdex(9)> pwndevils
Correct!
hackdex(9)> pwndevils
Correct!
hackdex(9)> pwndevils
Correct!
hackdex(9)> pwndevils
Correct!
===> GG!
```

As you can see, the check is run 6 times. We can confirm this by looking at the
cross references to the function. So we need to input 6 "correct" words to get
the flag. Now we need to understand what makes a word "valid."

## Valid Words

At this point, we need to explore the `dex::extra` function and find things that
look interesting in mangling our input. We find two such functions at `0xE312`
and `0xE46A`:

```c
_$LT$sha2..sha256..Sha256$u20$as$u20$digest..FixedOutput$GT$::fixed_result(...)

...

c2_chacha::rustcrypto_impl::init_chacha::hc4f97d8d62bc9341(...)
```

Yup, a `sha256` sum, and a `chacha` cipher. Let's work from the latter first.
With some google searching and some back tracking in the code, we discover that
something we input is used as the key to the cipher. You can guess what the
information is in the cipher text (the flag). 

But notice we only make it to the `chacha` if we make it passed a check:

```c
if ( _mm_movemask_epi8(
       _mm_and_si128(
         _mm_cmpeq_epi8(
           _mm_loadu_si128((const __m128i *)&sha256_output_buff),
           (__m128i)xmmword_9A610),
         _mm_cmpeq_epi8(_mm_loadu_si128((const __m128i *)&v171), (__m128i)xmmword_9A600))) == 0xFFFF )
{
```
I've annotated some of the variables in this, but it still looks terrible. Like
actual trash. Before fully understand what this `if` statement is doing, let's
go back to the former mentioned `sha256` digest. 

Looking at the call to the `sha256 Digest` we can see an input buffer filled
with "dynamic" looking strings before digesting:

```c
sha2::sha256::Engine256::input::hfd2fbea0c9756b12(
     &sha256_output_buff,
     input_strings,
     some_length);
```

From looking at this line, I realised that out input was SHA256ed then saved to
the output buffer, then used in the former `if` statement:

```c
if ( _mm_movemask_epi8(
       _mm_and_si128(
         _mm_cmpeq_epi8(
           _mm_loadu_si128((const __m128i *)&sha256_output_buff),
           (__m128i)xmmword_9A610),
         _mm_cmpeq_epi8(_mm_loadu_si128((const __m128i *)&v171), (__m128i)xmmword_9A600))) == 0xFFFF )
{
```

This `if` statement references the output of the `SHA256` sum. If you're picking
up what I'm putting down, you are guessing the input to the SHA256 is our
inputs, and you would be right! From the gruesome IDA code above, we can see the
output of the `sha` compared to some hex string. Following the above psuedo
operations by IDA, we use the below strings to get the expected sum:

```nasm
[...]
.rodata:000000000009A600 xmmword_9A600   xmmword 87A296E16B40FC1059EB2A66660AEF13h
.rodata:000000000009A600                                         ; DATA XREF: dex::extra::hc0d1779f7950f2c7+1C7A↑r
.rodata:000000000009A610 xmmword_9A610   xmmword 0BFD7A92606149E66179C8D06A8BA50F5h
.rodata:000000000009A610                                         ; DATA XREF: dex::extra::hc0d1779f7950f2c7+1C82↑r
.rodata:000000000009A620 xmmword_9A620   xmmword 8C6A0F1FA2C44D42CD205662BBA3DC86h
.rodata:000000000009A620                                         ; DATA XREF: dex::extra::hc0d1779f7950f2c7+1CBC↑r
.rodata:000000000009A630 xmmword_9A630   xmmword 5890B9434E79F522DD1E5B5D08129995h
.rodata:000000000009A630                                         ; DATA XREF: dex::extra::hc0d1779f7950f2c7+1CC6↑r
.rodata:000000000009A640 xmmword_9A640   xmmword 247ECE8C303B4D2189EB6670A480547Eh
.rodata:000000000009A640                                         ; DATA XREF: dex::extra::hc0d1779f7950f2c7+1CD1↑r
.rodata:000000000009A650 xmmword_9A650   xmmword 42BBFB04117574ED3441492CB8B45AE3h
[...]
```

Following the operations, we get the expected sum as:
`0xF550BAA8068D9C17669E140626A9D7BF13EF0A66662AEB5910FC406BE196A287`. 

To recap, we have six strings that are inputted from us, then are sha256 hashed.
If the output of the hash is:
`0xF550BAA8068D9C17669E140626A9D7BF13EF0A66662AEB5910FC406BE196A287`
then we have inputted the correct strings and we decrypt the stored flag using
the `chacha` cipher. 

## Playing some Boggle

After figuring all of that out, I was stumped for little since the input space
of 6 strings was a little too large to bruteforce for an expected hash. Out of
nowhere @fish, the reversing god himself, bestowed upon me that the call of
every `get_input` looked like it was setting up an array in memory. That's when
he put together that these tables must be the boggle tables reference in the
hint. 

So I took a look at the initialization of the arrays, and he was damn right!

```c
*(_QWORD *)v8 = 472446402657LL;
  *(_DWORD *)(v8 + 8) = 'n';
  *(_QWORD *)&input_buff = v8;
  _mm_storeu_si128((__m128i *)((char *)&input_buff + 8), _mm_load_si128((const __m128i *)&xmmword_9A5E0));
  v9 = _rust_alloc(12LL, 4LL);
  if ( !v9 )
    goto LABEL_201;
  *(_QWORD *)v9 = 0x6900000072LL;
  *(_DWORD *)(v9 + 8) = 'g';
  *(_QWORD *)(v6 + 16) = v148;
  *(_OWORD *)v6 = output_strings_2;
  *(_QWORD *)(v6 + 40) = v132;
  *(_OWORD *)(v6 + 24) = input_buff;
  *(_QWORD *)(v6 + 48) = v9;
  v11 = _mm_load_si128((const __m128i *)&xmmword_9A5E0);
  _mm_storeu_si128((__m128i *)(v6 + 56), v11);
```

This code may look a little confusing, but you can see that `v9` is being used
as the base of this array, and setup as a series of characters... as is `v8` and
`v6`. We have start breaking apart the memsets to discover a table being
created:

```
z e l 
a n n
r i g
```

That's a boggle table! SOOOOO, we must be playing boggle as inputs to our hashing algorithm.
If you follow every call to `get_input` you will notice that something similar
is happening before each call. Below are the tables from each call:

```python
0: ["zel", "ann", "rig"],
1: ["tkl", "bui", "nrf"],
2: ["fri", "pen", "uad"],
3: ["emz", "bna", "xeh", "wtv"],
4: ["evo", "rux", "com", "gni"],
5: ["plz", "asi", "son"],
```

We finally approach the endgame! We need to find out what strings are possible
for boggle table, then try every combination to the `SHA256` sum. After trying
this initially, and failing, I speculated that we needed to use one more piece
of information to finish this boi... the hackdex dictionary file!

## Getting that Flag

Here is the full recap of the rundown to solving for the flag:
1. We discovered the pro version 
2. We needed 6 strings that sha to `0xF550BAA8068D9C17669E140626A9D7BF13EF0A66662AEB5910FC406BE196A287` 
3. We discovered the boggle tables used to generate the strings
4. The strings can only be strings located in the `hacktm.hdex` file. 

After doing this, we get the final message:

`As we all know, CTFs are not only about winning. They are also about learning,
team work, overcoming yourself, passion and making friends. Have fun!
HackTM{wh4t_4_cur10us_pow3r_w0rds_h4v3}` 

The main portion of the solve is below:
```python
def sha256sum(s):
    sha256 = hashlib.sha256()
    sha256.update(s.encode("ascii"))
    return sha256.digest()
def main():
    # ===== GENERATE BOGGLE WORDS ===== #
    for i in range(6):
        board = BOARDS[i]
        X = len(board[0])
        Y = len(board)
        grid = {}
        for x in range(X):
            for y in range(Y):
                grid[(x, y)] = board[y][x]
        neighbours = get_neighbours(X, Y, grid)
        dictionary, stems = get_dictionary()
        paths = []
        print_grid(X, Y, grid)
        words = get_words(grid, paths, stems, dictionary, neighbours)
        B[i] = [ w for w in words if len(w) > 1 ]

    # ===== FILTER WORDS ===== #
    with open("hacktm.hdex", "rb") as f:
        data = f.read()
    lines = data.split(b"\n")
    words = set()
    for line in lines:
        if not line.strip():
            continue
        word = line[ : line.find(b"<br>")]
        words.add(word.decode("ascii"))
    for i in range(6):
        print(len(B[i]), "...")
        B[i] = list(set(B[i]).intersection(words))
        print(len(B[i]))
    target = binascii.unhexlify("F550BAA8068D9C17669E140626A9D7BF13EF0A66662AEB5910FC406BE196A287")

    # ===== FIND THE HASH ====== #
    for word_0 in B[0]:
        print(word_0 + "...")
        for word_1 in B[1]:
            for word_2 in B[2]:
                for word_3 in B[3]:
                    for word_4 in B[4]:
                        for word_5 in B[5]:
                            s = word_0 + word_1 + word_2 + word_3 + word_4 + word_5
                            if len(s) <= 31:
                                continue
                            h = sha256sum(s)
                            if h == target:
                                print(h, word_0, word_1, word_2, word_3, word_4, word_5)
if __name__ == "__main__":
    main()
```

The full solve script can be found
[here](https://github.com/mahaloz/mahaloz.re/tree/master/writeup_code/hacktm-20)


## Thanks where thanks is due

Big thanks to @fish for the help in this solve!

