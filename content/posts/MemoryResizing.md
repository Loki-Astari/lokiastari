---
title: Memory Resizing
date: 2016-03-25T05:53:07-0700
author: Loki Astari (C)2016
comments: true
categories:
- C++
- Resource-Management
- C++-By-Example
- Coding
subtitle: So you want to learn C++
description: So why is the constant resize factor of the array 1.5 or 2?
draft: false
math: true
cover:
  image: /images/post/post-2.png
  hidden: false
  caption: Photo by ThisisEngineering RAEng
---

So I never really considered why the resize of vector used a constant expansion of 1.5 or 2 (in some popular implementations). That was until I did my previous article series ["Vector"]({{config.site}}/blog/2016/02/27/vector/) where I concentrated a lot on resource management and did a section on [resizing the vector]({{config.site}}/blog/2016/03/12/vector-resize/). Initially, I tried to be clever in the code, a mistake. I used a resize value of 1.62 (an approximation of `Phi`) because I vaguely remembered reading an article that this was the optimum resize factor. When I put this out for code review, it was pointed out that this value was too large, the optimum value must be less than or equal to `Phi` (1.6180339887), and that exceeding this limit made things much worse.

So, I had to know why....

The theory goes: You have a memory resource of size `B`. If you resize this resource by a constant factor `r` by re-allocating a new block and then releasing the old block, if the value of `r` is smaller than or equal to `Phi`, you will eventually be able to reuse memory that has previously been released; otherwise, the new block of memory being allocated will always be larger than the previously released memory.

So, I thought, let's try that:
Test one `r > Phi`:

```text
    B=10
    r=2.0

                Sum Memory      Memory      Memory Needed       Difference
                 Released     Allocated     Next Iteration
    Start            0            10              20                 20
    Resize 1        10            20              40                 30
    Resize 2        30            40              80                 50
    Resize 3        70            80             160                 90
    Resize 4       150           160             320                170
```

OK. That seems to be holding (at least in the short term), so let's try a smaller value.
Test two `r < Phi`:

```text
    B=10
    r=1.5

                Sum Memory      Memory      Memory Needed       Difference
                 Released     Allocated     Next Iteration
    Start            0            10              15                 15
    Resize 1        10            15              22                 12
    Resize 2        25            22              33                  8
    Resize 3        47            33              48                  1
    Resize 4        80            48              72                 -8 // Reuse released memory next iteration
```

OK. That also seems to be holding. But can we show that it holds for all values of B? Also, this is a bit anecdotal. Can we show this relationship holds? Time to break out some maths (not math as my American cousins seem to insist on for the shortening of mathematics).


So the size `S` of any block after `n` resize operations will be:

$$S = Br^n$$

Thus, the size of `Released Memory` can be expressed as:

$$\sum_{k=0}^{n-1} Br^k$$

Also, the size of the next block will be:

$$Br^{n+1}$$

So if the amount of `Released Memory` >= the amount required for the next block, then we can reuse the `Released Memory`.

$$\sum_{k=0}^{n-1} Br^k \geq Br^{n+1}$$

$$B \sum_{k=0}^{n-1} r^k \geq Br^{n+1}$$

$$\sum_{k=0}^{n-1} r^k \geq r^{n+1}$$

$$\frac{1-r^{(n-1)+1}}{1-r} \geq r^{n+1}$$

$$\frac{1-r^n}{1-r} \geq r^{n+1}$$

$$1-r^n \geq r^{n+1} (1-r)$$

$$1-r^n \geq r^{n+1} - r^{n+2}$$

$$1 + r^{n+2} - r^{n+1} - r^n \geq 0$$

$$1 + r^n (r^2 - r - 1) \geq 0$$

This is where my maths broke down. So after talking to some smart people. They noticed that:

$$\sqrt{r^2 - r - 1} = 0 \quad \text{when} \quad r = \Phi$$

We find that the first root of the equation is 1. The second root of the equation depends on `n`, as `n` tends to `infinity`, and the other root tends towards `Phi`. From this, we can infer the following:

$$1 < r \leq \Phi$$

Thus, if `r` remains in the above range, then the theory holds.

