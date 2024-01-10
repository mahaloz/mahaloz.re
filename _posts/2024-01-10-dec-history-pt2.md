---
title: "30 Years of Decompilation and the Unsolved Structuring Problem: Part 2"
layout: post
tags: [research]
permalink: /dec-history-pt2
description: "A two-part series on the history of decompiler research and the fight against the unsolved control flow structuring problem. In part 1, we revisit the history of foundational decompilers and techniques, concluding on a look at modern works. In part 2, we deep-dive into the fundamentals of modern control flow structuring techniques, and their limitations, and look to the future."
toc: true
---

## A Recap

In [Part 1](/dec-history-pt1) of this series, we looked at the past 30 years of decompilation, providing a broad overview of how the field has been shaped.
We also briefly covered how Cifuentes-based structuring worked: [graph-schema matching](/dec-history-pt1#example-graph), where an algorithm identifies patterns in CFGs to make code.
Part 1 also garnered some interesting discussion across the web on [Hacker News](https://news.ycombinator.com/item?id=38859219), [Reddit](https://www.reddit.com/r/ReverseEngineering/comments/18xrsgn/30_years_of_decompilation_and_the_unsolved/), and [Twitter](https://twitter.com/mahal0z/status/1742576555153355223) among others. 
Generally, I think the community is very alive and excited about research and posts in this area, which is encouraging :). 

In this post, we focus on only four papers from the last 11 years: 
```bash
2024: USENIX   - Ahoy SAILR! There is No Need to DREAM of C: A Compiler-Aware Structuring Algorithm for Binary Decompilation
2020: Asia CCS - A Comb for Decompiled C Code
2015: NDSS     - No More Gotos: Decompilation Using Pattern-Independent Control-Flow Structuring and Semantic-Preserving Transformations. 
2013: USENIX   - Native x86 Decompilation Using Semantics-Preserving Structural Analysis and Iterative Control-Flow Structuring.
```

These papers, together, make up the entire history of decompilation structuring research in the security community. 
We will briefly overview the methods proposed in each of these works and how they influenced the next paper. 
Admittedly, the final paper in this series is SAILR, my paper. 
As such, I'll be using many of the same arguments and questions I used in my paper to understand previous works mentioned here.
As said in the summary of this post, Part 2 will be more technical than Part 1, but it should be accessible to newcomers. 

## Phoenix: Condition-Aware Schema Matching

The 2013 work [Phoenix](https://www.usenix.org/system/files/conference/usenixsecurity13/sec13-paper_schwartz.pdf) came at an interesting time since when it was published IDA Pro had an openly working decompiler for over 6 years. 
IDA Pro had been generating code that was better than other decompilers, but no one in the academic community knew why for certain (because it's closed-source).
In many ways, Phoenix was an attempt to make public the techniques that IDA Pro had been using to beat others, though Phoenix itself was never open-sourced. 
As such, the Phoenix decompiler core is much like Cifuentes' work on schema-matching, but with improved condition awareness and more schema.

Phoenix elaborated on many of the modern mechanics of structuring, such as loop refinement, switch refinement, condition handling, and even some special if-else case structuring.
This was important enough that some hacker decompilers, like Reko, [implemented much of the Phoenix paper](https://github.com/uxmal/reko/blob/63ba0cc20449ef51b23d3c8e89aa903715cf0714/src/Decompiler/Structure/StructureAnalysis.cs#L1154).
However, some of the largest impacts Phoenix had on the future of decompiler research can be surmised in three proposed points:
1. When you run out of good schema to match, you must use a _goto_. The process of turning an edge into a goto is called **virtualization**. 
2. Gotos in decompilation are bad, reduce them. 
3. [Coreutils](https://www.gnu.org/software/coreutils/) is a good dataset for the evaluation of decompilers.

To understand some of those claims, let's look back at our example graph from Part 1:

#### Example Graph
![](/assets/images/dec-history/struct_ex.png)

Recall, that `B-E` makes a good `if-else`, but `A->C` violates that structure.
In Cifuentes' work, you're allowed to also remove `A->B`, which will result in something structurable. 
Phoenix introduces more rules to choosing which edge to _virtualize_ (make into a goto): 
> "... remove edges whose source does not dominate its target, nor whose target dominates its source."

Under these rules, you must choose to virtualize `A->C`.
After virtualizing, your graph loses an edge and it's recorded as a goto.
The resulting graph looks like this:

#### Example Virtualized Graph
![](/assets/images/dec-history/virt_cfg.png)

Now the graph is only full of schema we can match, [resulting in structured code](/dec-history-pt1#example-structured-code) with **only 1 goto**.
Had you chosen to virtualize `A->B` instead, you could end with **2 gotos**.
Indeed, in the case of this CFG, 1 goto was better than 2 gotos. 
According to Phoenix, this phenomenon remains consistent as you reduce gotos: [gotos are considered harmful](https://homepages.cwi.nl/~storm/teaching/reader/Dijkstra68.pdf). 

The most popular decompilers today follow the same principles outlined in Phoenix.
That list includes [Ghidra](https://ghidra-sre.org/), [IDA Pro](https://hex-rays.com/ida-pro/), and [Binary Ninja](https://binary.ninja/).
I largely attribute their differences in output to be either based on better schema (IDA case) or better lifting (Ghidra case). 

The idea that you should reduce the number of gotos emitted in decompilation had a profound effect on the research direction future academics took.
Gotos became the prime metric for making your decompiler better.
With a metric to min-max, the next two papers focused on doing just that: getting gotos emitted to **0**.

## DREAM: Schemaless Condition-Based Structuring

2015 was an active year for decompilation, amounting to many [hacker decompilers](/dec-history-pt1#the-dawn-of-hacker-decompilers) and a new structuring algorithm. 
The paper ["No More Gotos: Decompilation Using Pattern-Independent Control-Flow Structuring and Semantic-Preserving Transformations"](https://net.cs.uni-bonn.de/fileadmin/ag/martini/Staff/yakdan/dream_ndss2015.pdf), colloquially referred to as DREAM, boasted a novel algorithm that could output decompilation with **0 gotos**.
This is a significant feat, considering, according to the DREAM paper, Phoenix had over **4,231** gotos in their output of Coreutils. 

To make such a quantitatively large jump required a drastically different structuring algorithm.
DREAM proposed the idea that you don't _need schema_ when making decompilation.
Instead, you can use the conditions on statements to generate semantically equivalent code. 
Taking a look at our [example CFG](#example-graph), DREAM would initially generate the following code based on its conditions:
```c
A();
if (~x)
  B();
if ((~x && y) || x)
  C();
if (~x && ~y)
  D();
E();
```

Note, DREAM would avoid making conditions for `E`, since it dominates everyone. After that, DREAM **simplifies the boolean expressions** to get the following:

```c
A();
if (~x)
  B();
if (x || y) {
  C();
}
else {
  D();
}
E();
```

DREAM accomplished a very interesting and effective way to make code that is always goto-less.
However, DREAM also came with two fundamental drawbacks to its approach:
1. [Simplifying arbitrary boolean expressions is NP-Hard](https://en.wikipedia.org/wiki/Boolean_satisfiability_problem)
2. Destroying every goto means gotos that existed in the source get eliminated

First, in practice, it's easy to find code with many conditions in it.
For instance, here is some DREAM ([implemented in angr](https://github.com/angr/angr/blob/0fa30cd21b4814c3ee299244cb5601c4c32b5c54/angr/analyses/decompiler/structuring/dream.py#L53)) decompiled code from `tail` in Coreutils:
#### tail: ignore_fifo_and_pipe
```c
        if (!v2 && !a0->field_34 && a0->field_38 >= 0 && (a0->field_30 & 0xf000) == 0x1000)
        {
            a0->field_38 = -1;
            a0->field_34 = 1;
        }
        if (a0->field_38 < 0 || v2 || a0->field_34 || (a0->field_30 & 0xf000) != 0x1000)
            v1 += 1;
```

If the SAT problem was solved, you could have reduced those two expressions. 
Because you can't guarantee you can simplify an expression, you are left with many overlapping booleans. 
In a [slide](https://docs.google.com/presentation/d/1qiZ9TuMZoC8dGqIATininf13PRk4YLG04Ne69kp-1LY/edit#slide=id.g1f608501ff2_0_10042) from my Ohio State Talk, I counted the number of Boolean operators in DREAM and other algorithms output.
DREAM had around **9,600** Booleans. Phoenix had **342**. The source code (Coreutils) had **1,256**. 

Though with its flaws, DREAM still had an important effect on decompilation. 
Namely, it introduced the importance of [Single-Entry Single-Exit (SESE) Regions](https://iss.oden.utexas.edu/Publications/Papers/PLDI1994.pdf), which played a fundamental role in future papers. 
It also had a [follow-up paper](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=7546501) that explored how this algorithm might work on C++ with a human study. 

So, what could a goto-less output be like if we got around the boolean expression simplification problem? 

## rev.ng: Code Cloning Schema-Matching

About five years after DREAM, in 2020, the paper ["A Comb for Decompiled C Code"](https://rev.ng/downloads/asiaccs-2020-paper.pdf) was published about its decompiler `rev.ng`, a closed-source commercial decompiler. 
The rev.ng decompiler targeted the same end goal as DREAM: **0 gotos** in decompilation.
However, it achieved this through a different means, sticking with schema-based algorithms as its base. 
Learning from DREAM, the rev.ng approach avoids simplifying booleans and instead uses a new, but [intuitive](https://www.reddit.com/r/Compilers/comments/18xtvam/comment/kgf1lcm/?utm_source=share&utm_medium=web2x&context=3), algorithm called Combing. 

Where DREAM duplicated conditions (as a result) to eliminate gotos, Combing duplicates actual code to fix CFGs _before_ they are structured. 
In this way, rev.ng uses the same structuring algorithm base as Phoenix, but fixes the CFG before structuring. 
Combing attempts to turn every graph into a series of layered diamonds, which results in nicely structured `if-else` trees. 
Using our [example CFG](#example-graph) once again, we can see the result of "Combing" the CFG:

#### Combing Example Graph
![](/assets/images/dec-history/comb_ex.png)

Take note, that the original graph only had one `C` node, but now there are two. 
Previously, an edge from `A->C` caused a goto, so Combing copied the code at `C` and made it a new node.
This new structure results in a perfect layering of diamonds, resulting in the following code through Phoenix now:
```c
A()
if (x) {
  C();
}
else {
  B();
  if (y)
    C();
  else
    D();
}
E();
```

As some Redditors on [r/Compilers discussed](https://www.reddit.com/r/Compilers/comments/18xtvam/comment/kggunv8/?utm_source=share&utm_medium=web2x&context=3), this concept can lead to an exponential increase in code. 
This also so happens to duplicate code when you may have had the opportunity to solve it with more schema. 
Similar to DREAM, the largest issue with this approach is its divergence from the source that contains gotos. 

---

At this point in the blog post, we approach the final paper in this series of structuring, SAILR. 
As the first author of this paper, I'll be talking about this section a little differently from the others.
Much of the progression of this story, as well as its fueling questions, was produced by my work on SAILR. 

## SAILR: Compiler-Aware Structuring

After understanding the fundamental gains each of these papers had, you may still have some lingering questions.
In Phoenix, we reduced gotos with more schema.
In DREAM, we reduced gotos at the expense of duplicating conditions. 
In rev.ng, we reduced gotos at the expense of duplicating code. 
But one thing never answered was, _why do gotos even exist in decompilation?_
Also, is goto reduction the _best way to measure_ when decompilers are improving? 

In our 2024 work ["Ahoy SAILR! There is No Need to DREAM of C: A Compiler-Aware Structuring Algorithm for Binary Decompilation"](https://www.zionbasque.com/files/publications/sailr_usenix24.pdf), referred to as SAILR, we explored these questions.
We also [open-sourced the entire decompiler](https://github.com/mahaloz/angr-sailr), [evaluation pipeline](https://github.com/mahaloz/sailr-eval), and [reimplementations of all previous work](https://github.com/mahaloz/sailr-eval/blob/e1af48353c1c5b32cc53cbaa015722d57767bd6e/sailreval/decompilers/angr_dec.py#L45) mentioned here. 


Back to our first question, **why do gotos exist in decompilation?**
In schema-based structuring algorithms, we know that gotos appear because there are no more schemas to match a section of the CFG.
But why do the CFGs generate _unstructurable_ patterns in the first place? 
Surely, if you compiled code with no gotos you should always be able to structure the graph without gotos? 

As you likely guessed, when you compile goto-free code you _can get_ decompilation with gotos.
Take this simplified example based on the SAILR paper:

#### Comparing Code and Decompilers

<!-- SOME INSANE TABLE CODE BELOW, see dec-history/code.md --->
<table>
<tr>
<td> Source Code </td> <td> IDA Pro (and others)* </td> <td> DREAM </td>
</tr>
<tr>
<td> 
<div class="language-c highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1">// ====================</span>
<span class="k">if</span> <span class="p">(</span><span class="n">a</span> <span class="o">&amp;&amp;</span> <span class="n">b</span><span class="p">)</span>
  <span class="n">puts</span><span class="p">(</span><span class="s">"first"</span><span class="p">);</span>

<span class="n">puts</span><span class="p">(</span><span class="s">"second"</span><span class="p">);</span>
<span class="k">if</span> <span class="p">(</span><span class="n">b</span><span class="p">)</span>
  <span class="n">puts</span><span class="p">(</span><span class="s">"third"</span><span class="p">);</span>

<span class="n">sleep</span><span class="p">(</span><span class="mi">1</span><span class="p">);</span>
<span class="n">puts</span><span class="p">(</span><span class="s">"fourth"</span><span class="p">);</span>
</code></pre></div></div>
</td>


<td> 
<div class="language-c highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1">// ====================</span>
<span class="k">if</span> <span class="p">(</span> <span class="n">a</span> <span class="o">&amp;&amp;</span> <span class="n">b</span> <span class="p">)</span>
<span class="p">{</span>
  <span class="n">puts</span><span class="p">(</span><span class="s">"first"</span><span class="p">);</span>
  <span class="n">puts</span><span class="p">(</span><span class="s">"second"</span><span class="p">);</span>
  <span class="k">goto</span> <span class="n">LABEL_6</span><span class="p">;</span>
<span class="p">}</span>
<span class="n">puts</span><span class="p">(</span><span class="s">"second"</span><span class="p">);</span>
<span class="k">if</span> <span class="p">(</span> <span class="n">b</span> <span class="p">)</span> <span class="p">{</span>
<span class="nl">LABEL_6:</span>
  <span class="n">puts</span><span class="p">(</span><span class="s">"third"</span><span class="p">);</span>
<span class="p">}</span>
<span class="n">sleep</span><span class="p">(</span><span class="mi">1u</span><span class="p">);</span>
<span class="n">puts</span><span class="p">(</span><span class="s">"fourth"</span><span class="p">);</span>
</code></pre></div></div>
</td>

<td>
<div class="language-c highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1">// ====================</span>
<span class="k">if</span> <span class="p">(</span><span class="n">a</span> <span class="o">&amp;&amp;</span> <span class="n">b</span><span class="p">)</span> <span class="p">{</span>
    <span class="n">puts</span><span class="p">(</span><span class="s">"first"</span><span class="p">);</span>
    <span class="n">puts</span><span class="p">(</span><span class="s">"second"</span><span class="p">);</span>
<span class="p">}</span>
<span class="k">if</span> <span class="p">(</span><span class="o">!</span><span class="n">a</span> <span class="o">||</span> <span class="o">!</span><span class="n">b</span><span class="p">)</span>
    <span class="n">puts</span><span class="p">(</span><span class="s">"second"</span><span class="p">);</span>
<span class="k">if</span> <span class="p">(</span><span class="n">b</span><span class="p">)</span>
    <span class="n">puts</span><span class="p">(</span><span class="s">"third"</span><span class="p">);</span>
<span class="n">sleep</span><span class="p">(</span><span class="mh">0x1</span><span class="p">);</span>
<span class="n">puts</span><span class="p">(</span><span class="s">"fourth"</span><span class="p">);</span>
</code></pre></div></div>
</td>


</tr>
</table>
<br>

*Others include any Phoenix-based decompiler like Ghidra and Binja.
SAILR is the exception, which produces the source code exactly in this example.

The code was compiled with `gcc -O2`, the default compiler optimization level for packages in Linux.
Indeed, IDA Pro has both a goto and duplicated code (`puts("second")`).
In SAILR, we found that the cause of _most_ gotos is just **9 compiler optimizations**, most found in `O2`.
The only way to ever get rid of these gotos, while maintaining structure like the source, is to revert them precisely. 
 
This realization was important because up until this point, we had been removing gotos from decompilation without knowing why.
Doing something without understanding the root cause always results in side effects, as seen in DREAM and rev.ng. 
However, doing nothing results in decompilation with gotos that did not exist in the source, as seen in Phoenix. 

With this in mind, we fought on two fronts.
First, we made our decompiler revert all 9 compiler opts precisely to undo their transformations that made gotos. 
Second, we measured which decompiler was doing the best by comparing their decompilation to the source using a sped-up version of [Graph Edit Distance](https://en.wikipedia.org/wiki/Graph_edit_distance) (called CFGED).
With Control-Flow Graph Edit Distance (CFGED), a decompiler can still have gotos in its output and get a good score if it matches the source. 

A perfect decompiler should produce a **0 CFGED**, meaning 0 graph edit distance, and the same gotos as the source.
Here is how everyone did in a condensed graph derived from the paper on 26 C packages:

![](/assets/images/dec-history/cfged_graph.png)

As you might've guessed, goto-less algorithms do the worst when you compare them to their original source. 
IDA Pro has the best CFGED score, but still produces a huge amount of _fake_ gotos. 
Our algorithm to produce a CFGED score on source for your own decompiler is [open-sourced](https://github.com/mahaloz/cfgutils/blob/main/cfgutils/similarity/cfged.py), so get yours evaluated today!

In conclusion, **SAILR can be summarized into three points**:
1. Gotos are caused by compiler optimizations, many of which you can revert.
2. CFGED is a new algorithm to fairly measure your distance from source.
3. Not all gotos are evil.

For more information on how CFGED works, how you might revert optimizations, or how we discovered which optimizations cause gotos, read our paper. 
The appendix also has a nifty step-by-step of the CFGED algorithm in practice.
Additionally, if you want to try out any of these previous works in the real world, check out our [sailr-eval](https://github.com/mahaloz/sailr-eval#using-sailr-on-angr-decompiler) repo for how to use angr with previous structuring algorithms. 

## Wrap Up

And here we are, at the end of this whirlwind of a post series.
Many issues still exist in decompilation aside from structuring, such as low fidelity to source, weak editability, no recompilability, and other sub-topics such as poor symbol recovery. 
Of these issues, I find, possibly biasedly, that control flow structuring is the most interesting.
I hope after reading this, you find it interesting too, since it's still unsolved! 

Even with our work on SAILR, which got us much closer to source, there is still so much more the community needs to discover. 
Notice that in the graph on `CFGED v Gotos`, SAILR is still nearly double the gotos of the source and nowhere near **0** CFGED.
The future of _perfect decompilation_ is tied to improving these two scores. 
I believe with open and community-driven work, we will achieve near-perfect decompilation. 

I'd like to acknowledge the [DARPA AMP](https://www.darpa.mil/program/assured-micropatching) program for funding this research. 
Lastly, I'd like to thank you, the reader. 
Your support motivates us to keep pushing the boundaries of what is possible. 