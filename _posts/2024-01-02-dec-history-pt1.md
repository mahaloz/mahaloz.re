---
title: "30 Years of Decompilation and the Unsolved Structuring Problem: Part 1"
layout: post
tags: [research]
permalink: /dec-history-pt1
description: "A two-part series on the history of decompiler research and the fight against the unsolved control flow structuring problem. In part 1, we revisit the history of foundational decompilers and techniques, concluding on a look at modern works. In part 2, we deep-dive into the fundamentals of modern control flow structuring techniques, and their limitations, and look to the future."
toc: true
---

## Retrospection

As 2024 begins, I've taken a moment to think about the last year(s) of decompilation research, since I spent most of my 2023 working on decompilation.
I've largely spent that time working on a fundamental area of decompilation research called control flow structuring. 
Although I find this area critical to decompilation, I realize most security people have only heard of it in passing. 
With that in mind, I wanted to take a moment to recap the (short) history of binary decompilation and structuring, loosely based on my [Ohio State talk](https://icdt.osu.edu/events/2023/03/virtual-event-modern-approaches-human-centric-decompilation) on the same topic. I also mix in some of my advisor's, [Dr. Fish Wang](https://ruoyuwang.me/), [NDSS talk](https://www.youtube.com/watch?v=XasallkPQIA). 

To clarify, this post will be about _binary decompilation_, or decompilation on programs produced by compiled languages like C, C++, Rust, and Go.
I will focus on the history of decompilation as preliminary information needed to understand the control flow structuring. 
Luckily (or unluckily?), decompilation as a research area has a relatively small history, even considering non-academic approaches like IDA Pro. 

## Decompiler Origins

Many researchers credit [Dr. Cristina Cifuentes](https://scholar.google.com/citations?user=iseZ69MAAAAJ&hl=en) with publishing the first work on decompilation in her 1994 dissertation "Reverse Compilation Techniques".
However, other decompilers also existed at this time, and are documented on [program-transformation.org](https://www.program-transformation.org/Transform/HistoryOfDecompilation2.html).
In Cifuentes work, she reaffirms the idea that after we have a disassembled program, in the form of a Control Flow Graph (CFG), we still have more work to do:
>  "The control flow graph ... has no information on high level language control structures such as `if..then..elses` and `while()` loops. Such a graph can be converted into a structured high level language graph by means of a _structuring_ algorithm."

A structuring algorithm takes this CFG, which may already be lifted to an intermediate language (IL), and converts it into high-level structures. 
In Cifuentes work, the structuring algorithm matches patterns in the graph, referred to as graph schema, and marks them as structures.
Some of those schema can be seen below:

![](/assets/images/dec-history/dcc_schema.png)

You keep identifying structure, bottom-up, until you run out of structures to identify. 
Take note that it is always possible to exhaust patterns in the graph and make C code because of the special _goto_ structure. 
Indeed the goto structure can be constructed from any node with an edge, which must be used sparingly for decent code. 

Let's run an example to briefly understand how this works.
Below you will find a five-node CFG, which has been lifted from an assembly graph:
#### Example Graph
```
                            +-----+
                            |  A  |
                            +-----+
                              |   |
                           ~x |   +--+
                              V      |
                         +-----+     |
                         |  B  |     | x
                         +-----+     |
                           |  +--+   |
                        ~y |     | y |
                           V     V   V
                       +-----+  +-----+
                       |  D  |  |  C  |
                       +-----+  +-----+
                            |    |
                            V    V
                            +-----+
                            |  E  |
                            +-----+
```

That diamond pattern encapsulated in `B`-`E` is a classic `if-then-else` structure, where `y` is the condition. 
Unfortunately, there is also an edge slamming right in the middle of that structure: `A -> C`, violating the known pattern.
Intuitively, you can't have an edge entering in one of the scopes of an `if`... unless that edge is a goto! 
Cifuentes doesn't introduce a clean-and-cut method to know which edge to make into a goto, but we assume `A->C` would be best. 
In this case, it results in the "best" possible output for this graph:

#### Example Structured Code
```c
A();
if (x)
    goto C;
B();
if (y) {
    C:
    C();
}
else {
    D();
}
E();
```

Note, there are many ways to output this code and many edges that could've been chosen to make the goto. 
Small choices like what edge becomes the goto, what condition is flipped in an `if`, and how you reduce scopes (like me omitting the `else` of `if (x)`), can drastically affect the quality of the output.
With this in mind, any lifted (or assembly) CFG has many possible structured-C outputs as options. 
At runtime, we don't know which of those options (if any of them) is the original C that generated the CFG. 
We'll explore more of those ideas, `goto` choosing, and alternate algorithms in Part 2 of this post series.

Let's continue with some history. 
Much of this work on control flow structuring was directly inspired and led by the work in compilers. 
One such work was "Compilers: Principles, Techniques, and Tools," published in 1986. 

Here is a snippet of code produced by the 1994 `dcc`, the decompiler introduced in Cifuentes work:
```c
#include "dcc.h"

void proc_1 (int argo, int arg1, int arg2)
{
    int loc1; 
    int loc2; 
    int loc3;
    Loc2 = 0;
    while ((10c2 < 5)) {
        10c3 = 0;
        while ((10c3 < 4)) {
            1001 = 0;
            while ((10c1 < 4)) {
                *(((1002 * 10) + arg2) + (1003<< 1)))= 
                    (*11(1002 < 3) + arg0) + (1001 << 1)))*
                    *((((1001 * 10) + arg1) + (1003<<1)))) +
                    *(1002 * 10) + arg2) + (10c3 <1))));
                10c1 = (10c1 + 1) ;
            ｝
            10c3 = (10c3 + 1);
        ｝
        10c2 = (10c2 + 1);
    }
}

void main (
{ 
    int 10c1; 
    int loc2; 
    int loc3;
    proc_1 (81003, &1002, &1001);
}
```

It's rugged, lacking variable collapsing and any meaningful structure simplification, but hey, it works! 
I think the first thing you may notice is the over-use of `while()` loops in the program. 
This is of course because you can wrap most structures in a `while` and end it with a `break` to guarantee scopes are correct.
You'll notice modern decompilers still do this when scopes get wonky. 

I generally summarize Cifuentes' dissertation as identifying (implicitly) three fundamental pillars of decompilation:
1. CFG recovery (through disassembling) & Lifting
2. Variable recovery (including type inferencing)
3. Control flow structuring 

Although these areas had much work to be done in 1994, other academics did not approach them deeply for many more years.
However, hackers across the world are now ready to make their own decompilers, following the same general ideas. 

## The Dawn of Hacker Decopmilers

It would be impossible to list all the decompilers that popped up post-Cifuentes, so instead, I will list the ones that still live today.
Luckily, you already know one of them: IDA Pro, [which established its decompiler HexRays in 2007](https://hex-rays.com/about-us/our-journey/), and continues to dominate the decompiler game today.
That same year, listed as a few months earlier, [Reko](https://github.com/uxmal/reko) decompiler, an open-source alternative, released its [first decompiler version](https://github.com/uxmal/reko/commit/c686454070ea00b35d7e1c3c358da9d26d0dc7d4).
Both IDA and Reko follow the general ideas introduced in Cifuentes' work, structuring graphs into code by matching known schema that compilers follow. 
Some assume that IDA Pro took this to a new level by doing some condition-checking to reduce structures at this time. 

Much of the work at this time follows the same ideas, with varying success, but research in the academic world started around 4 years later in 2011.
Without jumping too far into the future, it's worth noting [Sonwman](https://github.com/x64dbg/snowman) and [fcd](https://github.com/fay59/fcd) decompiler were also released in 2015.
I like to assume that around the development time of IDA Pro, [Ghidra](https://ghidra-sre.org/) was also developed. 
If you know of any other decompilers that belong here, please let me know on social media or in the comments. 

## The Academic Pickup

Academic work picked up in decompilers again in 2011 with the Carnegie Mellon team publishing ["TIE: Principled reverse engineering of types in binary programs"](https://kilthub.cmu.edu/articles/journal_contribution/TIE_Principled_Reverse_Engineering_of_Types_in_Binary_Programs/6469466/1/files/11898020.pdf), a new way to recover types (and variables) in CFGs. 
I include this paper because it's used extensively in their 2013 decompiler work, ["Native x86 Decompilation Using Semantics-Preserving Structural Analysis and Iterative Control-Flow Structuring"](https://www.usenix.org/system/files/conference/usenixsecurity13/sec13-paper_schwartz.pdf), colloquially known as Phoenix. 
Last August (2023), Phoenix turned 10 years old. 

The Phoenix paper was the first decompilation control flow structuring algorithm to be published in what security academics refer to as the Top 4: [USENIX Security](https://www.usenix.org/conference/usenixsecurity24), [CCS](https://www.sigsac.org/ccs/CCS2023/), [S&P](https://sp2024.ieee-security.org/), and [NDSS](https://www.ndss-symposium.org/), the distinguished security conferences as noted by [CS Rankings](https://csrankings.org/).
This is significant because it signals to other researchers that decompilation is publishable!

After Phoenix, I assumed the Top 4 would be filled with fundamental research in decompilers, like control flow structuring. 
However, the field has remained rather quiet for the last 20 years. 
Using a simple title grepping tool for the Top 4, [top4grep](https://github.com/Kyle-Kyle/top4grep), we can see papers published since 2000:

```bash
top4grep -k decompil
[Top4Grep][INFO]12-29 18:04 Grep based on the following keywords: decompil
[Top4Grep][DEBUG]12-29 18:04 Found 10 papers
2024: USENIX   - Ahoy SAILR! There is No Need to DREAM of C: A Compiler-Aware Structuring Algorithm for Binary Decompilation
2024: USENIX   - A Taxonomy of C Decompiler Fidelity Issues
2024: IEEE S&P - "Len or index or count, anything but v1": Predicting Variable Names in Decompilation Output with Transfer Learning
2023: USENIX   - Decompiling x86 Deep Neural Network Executables.
2023: IEEE S&P - QueryX: Symbolic Query on Decompiled Code for Finding Bugs in COTS Binaries.
2023: IEEE S&P - Pyfet: Forensically Equivalent Transformation for Python Binary Decompilation.
2022: USENIX   - DnD: A Cross-Architecture Deep Neural Network Decompiler.
2022: USENIX   - Decomperson: How Humans Decompile and What We Can Learn From It.
2022: USENIX   - Augmenting Decompiler Output with Learned Variable Names and Types.
2019: CCS      - Kerberoid: A Practical Android App Decompilation System with Multiple Decompilers.
2016: IEEE S&P - Helping Johnny to Analyze Malware: A Usability-Optimized Decompiler and Malware Analysis User Study.
2015: NDSS     - No More Gotos: Decompilation Using Pattern-Independent Control-Flow Structuring and Semantic-Preserving Transformations.
2013: USENIX   - Native x86 Decompilation Using Semantics-Preserving Structural Analysis and Iterative Control-Flow Structuring.
```

Around 13 papers, three of which I added manually from the 2024 cycle. 
For transparency, I'll add that I'm a first author or co-author on two of them. 
Of these papers, the following make improvements or investigate _binary decompilation_:

```bash
2024: USENIX   - Ahoy SAILR! There is No Need to DREAM of C: A Compiler-Aware Structuring Algorithm for Binary Decompilation
2024: USENIX   - A Taxonomy of C Decompiler Fidelity Issues
2024: IEEE S&P - "Len or index or count, anything but v1": Predicting Variable Names in Decompilation Output with Transfer Learning
2022: USENIX   - Decomperson: How Humans Decompile and What We Can Learn From It.
2022: USENIX   - Augmenting Decompiler Output with Learned Variable Names and Types.
2016: IEEE S&P - Helping Johnny to Analyze Malware: A Usability-Optimized Decompiler and Malware Analysis User Study.
2015: NDSS     - No More Gotos: Decompilation Using Pattern-Independent Control-Flow Structuring and Semantic-Preserving Transformations.
2013: USENIX   - Native x86 Decompilation Using Semantics-Preserving Structural Analysis and Iterative Control-Flow Structuring.
```

We are left with only 8 papers since 2000. 
For comparison, if you look at papers with "kernel" in the title, you get a whopping **121** papers. 
If you consider all decompilation papers across all possible conferences (including Top 4), you still only have around 45 papers.
My advisor and I have compiled a [list](https://docs.google.com/spreadsheets/d/162J8Qcev10poAgaXCECUFovQNVd96sJ1AwJC7XDtFPg/edit#gid=0) of all those papers. 

It's easy to see that decompilation research has been a rather slow field for the last 20 years with academics. 
I think most people would find this surprising since decompilation is both important to security researchers and still has many issues. 
You can find many examples across the web of IDA Pro or Ghidra failing to decompile code correctly.

Generally, I think the academic community never picked up decompiler research intensely because it was difficult to get a working developable decompiler. 
Recall, that it is not until 2019 that the NSA's [Ghidra](https://ghidra-sre.org/) decompiler is made public, the first open-source decompiler released by a well-funded organization.

### The Structuring Papers

Of all the decompiler papers, only four are on control flow strucuturing:
```bash
2024: USENIX   - Ahoy SAILR! There is No Need to DREAM of C: A Compiler-Aware Structuring Algorithm for Binary Decompilation
2020: Asia CCS - A Comb for Decompiled C Code
2015: NDSS     - No More Gotos: Decompilation Using Pattern-Independent Control-Flow Structuring and Semantic-Preserving Transformations. 
2013: USENIX   - Native x86 Decompilation Using Semantics-Preserving Structural Analysis and Iterative Control-Flow Structuring.
```

Note, ["A Comb for Decompiled C Code"](https://rev.ng/downloads/asiaccs-2020-paper.pdf) is not in the Top 4, however, it made contributions significant enough to mention. 
Adding to the difficulty of working on decompilers, only two (2015 and 2024) of these four works had open-source implementations.
Additionally, the 2015 work (DREAM) did not make its code public until [2020](https://api.github.com/repos/CodeIntelligenceTesting/dream), and it only worked on top of IDA Pro 6.6. 

It's easy to see why these advancements took so long when you consider each researcher may need to remake an entire decompiler. 
The future is not bleak though! There are open-source decompilers now and we have made progress in structuring, which we will talk about in Part 2!

## Wrap Up

To summarize, there has been growth in the decompiler community, but it has been much slower than others. 
Aside from control flow structuring, there has been significant work in using [machine learning to recover symbols](https://www.atipriya.com/files/papers/varbert_oakland24.pdf), [human studies to inform decompiler developers](https://www.usenix.org/conference/usenixsecurity22/presentation/burk), and some preliminary work in [using machine translation for code output](https://arxiv.org/abs/2212.08950). 
Control flow structuring though is still developing and has many problems to overcome. 
We explore those problems, why they exist, modern approaches, and what the future may look like for decompilers in part 2 of this post series.

If you enjoyed this post, I hope you'll upvote or like it on some [social media](https://twitter.com/mahal0z) platform, since quantitating impact is always nice for researchers :). 
I've also integrated a funny GitHub comment system if you like to use GitHub for interaction or stars. 
Until next time, see you in Part 2!