---
title: "Lambda calculus challenges for ASU Undergrads"
layout: post
tags: [ctf, math]
description: "This was a challenge written for a programming languages and compilers course at ASU. This is meant to test your skill in lambda calculus."
toc: true
---
## Rules
This challenge is broken into three parts, each with growing difficulty. To
complete these challenges you *ARE NOT* expected to re-write common lambda
calculus functions such as:
```
pred
succ
pair
fst
snd
tru
fls
church numerals 
...
```
You get the idea. All common lambda functions are game and do not need to be
expanded to lambda header form. For a more expansive list of common Lambda
functions check out the CSE340 lectures. TODO: Add an external link here, open
to suggestions.

## Challenge Description
Consider a stack implementation in $$ \lambda $$-calculus using pairs:
```
(a, (b, (c, (d, fls) ) ) )  
```
We define a stack to recursively have the form:
```
stack = (element, stack_body)
```
To put this in a legitimate syntax, we can say:
```
stack = pair element stack_body
```

A stack body is empty if it is `fls`. Otherwise, the `stack_body` will follow
the same form. 


### Part 1
Any good stack should have a basic function, poping! Write a `pop` function that
gets the element on the top of the stack. 

For instance:
```
pop (a, (b, (c, (d, fls) ) ) )	->	a
``` 

If a stack is empty, return `fls`. 


### Part 2 
How can we pop things if we can't push things! Write a `push` function that,
when given a stack and element, pushes the element on the stack. 

For instance:
```
push a (b, (c, (d, fls) ) ) )  ->  (a, (b, (c, (d, fls) ) ) ) 
``` 

If a stack is empty, return `(pair element fls)`.


### Part 3
Time to turn up the heat! Now that we have a working stack, let's get a little
creative with it. Write a `reverse` function that reverses the entire stack.

For instance:
```
reverse (a, (b, (c, (d, fls) ) ) )	->	(d, (c, (b, (a, fls) ) ) )
```

If a stack is empty, return `fls`. 

 
### Part 4
This will be the holy grail of the challenges. Write a `sort` function that
fully sorts a stack, assuming the stack is full of church numerals. 

For instance:
```
sort (7, (3, (1, (3, fls) ) ) )	 ->	 (1, (3, (3, (7, fls) ) ) )
```

If a stack is empty, return `fls`. 


## Solutions
The solutions will be provided on request, or posted here if it becomes too pain
staking to give everyone the solution. Good luck!

