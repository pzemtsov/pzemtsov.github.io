---
layout: post
title:  "To allocate or to pool?"
date:   2019-01-17 12:00:00
tags: Java C++ optimisation real-time
---

In the [previous article]({{ site.ART-QUEUE }}) we developed a fast queue between native (**C++**) and **Java** code, which can be used as a clock generator with a fairly small
tick interval (20 ns). Today we'll use it as a measurement tool.

Today's question applies to message processors: is it better to
allocate a fresh buffer for each arriving message, or to allocate them all upfront, collect in a pool and re-use them?
I witnessed numerous discussions on this issue.  The old-school **C** programmers prefer not to allocate
any memory if it is possible, while the modern-days **Java** paradigm suggests allocating objects
left, right and centre.

Let's try to investigate it and get a definite answer.

This problem is applicable to both batch and real-time (or near-real-time) processors. In the former case the only
important factor is the overall throughput of the program; in the latter case we also have the latency issue: we can miss incoming
messages if processing some of them takes too long.

We'll look at both of these cases.

There is in fact a third case: a network request processor, such as an HTTP server. Like a batch processor,
it must be reliable (not to drop messages), and the throughput must be high. Like a real-time processor,
it imposes latency restrictions, although they aren't so strict. We won't look at this case here,
concentrating on the other two.

Note on real-time programming
-----------------------------

Some readers may have wondered why someone even tried doing anything real-time in **Java**.
Everyone knows that **Java** isn't a real-time platform. For that matter, ordinary Windows
or Linux are not real-time operation systems, either.
No one would write a true real-time program (such as an auto-pilot) in **Java**. 
In this article by real-time programs we'll mean near-real-time ones: those where small event loss is
permitted. A good example is a network traffic analyser,
where it is usually not a tragedy if a couple hundred packets out of a million is lost (unless this
is a security monitor, of course). Such programs can be developed quite successfully in just about any language, including **Java**,
and run on normal operation systems. We'll use a very simplified model of such an analyser as
a sample program.

The effects of the garbage collector
------------------------------------

Why is the choice between allocating buffers and pooling them important at all? In the case of **Java**, the most important factor is the Garbage Collector (**GC**),
because it really can pause the entire program execution (it is called "Stop the world").

In its simplest form, the garbage collector:

- is invoked when a memory allocation request fails, which happens at a frequency proportional to the rate memory is allocated;

- runs for a time proportional to the number of live objects (this may seem to make the name "Garbage collection" a bit misleading. Something like "Valuables evacuation"
may seem more appropriate. However, in real life paying for garbage removal depending on your wealth is not unheard of).
	
Real garbage collectors employ various tricks to improve performance, eliminate long pauses and reduce sensitivity to the number of live objects:

- they split objects into two or more spaces ("generations"), according to their age, under the assumption that what has lived for a while, is likely to live
a bit more, whereas a freshly allocated item is likely to die soon. This is (statistically) true in **Java**, where programs allocate lots of temporary objects;

- they run a quick and lightweight version of the  garbage collector, which works on younger generations, and which is invoked very often and long before memory is exhausted. This
makes full GC invocation much less frequent, but does not eliminate it completely;

- they perform a big part of their work in parallel with the user program, making execution pauses shorter, or, in the luckiest cases,
avoiding them.

These improvements don't come for free and often require some expensive code generation overheads, such as write-barriers and even read-barriers. This slows execution
down but, in many cases, is still justified by reducing garbage collection pauses.

However, these improvements, in general, don't affect the two basic GC rules: the GC is invoked more often when more memory is allocated, and it runs longer when there
are more live objects.

Our two approaches regarding the buffers differ in their memory allocation patterns. The allocating strategy usually keeps the number of live objects small,
but causes the GC to be invoked often. The pooling option reduces memory allocation but keeps all the buffers as live **Java** objects (sometimes more than one object per buffer).
The GC is called less often, but runs longer.

The reason we create many buffers in the pooling version 
is that we're dimensioning for a high demand of them -- for a situation, hopefully short in time,
when many of them are used at the same time. In the case of the allocation solution this means frequent
and long-running GC -- a super-challenge.

Obviously, buffers are not the only objects allocated by programs. Some programs keep so many permanently allocated data structures (maps, caches, transaction logs),
that buffers fall into insignificance. Some others allocate so many temporary objects (e.g. some message parsers tend to create an object for each entity parsed), that
buffer allocation falls into insignificance. This problem and this investigation are not applicable to these cases. However, it is worthwhile to establish the boundaries.

Other possible problems
-----------------------

Memory allocation on its own (ignoring the GC) has some costs. Obtaining the address of a new object is usually quick (especially for **Java** implementations that
employ thread-local memory pools). However, there is also zeroing the memory and calling the constructor if there is one.

On the other hand, pooling involves some overheads, too. Usage of every buffer must be carefully tracked, so that it can be returned to the free pool
as soon as it becomes vacant. This can become quite tricky in the multi-threaded case. Failure to
track buffers may lead to buffers leak, similar to a classic memory leak in **C** programs.

Mixed version (partial pooling)
-------------------------------

One often used approach is to keep a pool of certain capacity, and allocate buffers when demand exceeds this capacity. The freed buffers are only returned into
the pool if it isn't full, otherwise discarded. This approach offers a good compromise between pooling and allocating, so it is worth testing.

The test
--------

We'll imitate a typical network analyser -- an application that captures packets from a network interface, decodes protocols and collects statistics. We'll use
a very simplified model of it, which consists of:

- a packet class, which consists of a byte buffer and its parsing results. We'll be using a byte buffer, not an array,
  to make it easier to switch to native (direct) byte buffers later. For now, our byte buffer will be backed
  by an array, which will increase the number of allocated objects;

- a data source, which acquires buffers and populates them with some random IP-related information (addresses, ports and protocols). Apart from the buffers,
  the data source does not allocate any memory;

- a queue, where these buffers will await processing. In our initial model this will be just a FIFO-structure (an `ArrayDeque`) of size `INTERNAL_QUEUE_SIZE`,
  whose only purpose will be to store that number of live buffers. No memory is allocated at this stage either;

- a processor, which will parse buffers, allocating some temporary objects in the process. Some selected objects (TCP packets, which in our model will occur with
  the frequency of 1/16), together with their parsing artefacts, will be stored for longer in a structure of size `STORED_COUNT`
  (imitating the work of a TCP stream reconstructor). This structure will use a random
  replacement strategy.

Today we'll limit the study to the simplest, single-threaded, setup: the processor and the data source will run in the same thread.
We'll consider the multi-threaded case later.

To write less code, we'll implement the mixed solution in the same way as the pooled one, the only difference being the pool size:
`MIX_POOL_SIZE` or `POOL_SIZE`.

Two data sources will be used:

- a batch data source, which generates objects as fast as it can. We'll be interested in the maximal achieved throughput;

- a real-time data source, which starts a native ("source") thread and sets up a native-to-**Java** queue ("input queue"), where this thread will write clock
signals (sequence numbers) every `SOURCE_INTERVAL_NS` nanoseconds. Upon receiving each signal, the data source will generate one packet
(we won't bother with actually reading the packet from the input queue).
The total source queue capacity will be defined
in milliseconds as `MAX_CAPTURING_DELAY_MS`.
The source packets will be lost if the queue isn't serviced for this time (this loss will be detected using sequence numbers).
Here we'll be interested in the smallest value of `SOURCE_INTERVAL_NS` where the packets are not lost. We'll be assuming the `MAX_CAPTURING_DELAY_MS` of 100 ms.
This isn't actually as small as it sounds. At the data rate of one million packets per second this
means the queue size of 100,000 messages, which may already be too long for a hardware probe, and we
are aiming for higher data rates (perhaps, not quite ten million, but maybe five million per second).

We'll consider three cases for the tests:

- Case **A**: packets are received, parsed and mostly dropped. Very few are kept in memory;

- Case **B**: Substantial number of packets are used, but still much less than were pre-allocated;

- Case **C**: Almost all the pre-allocated packets will be used.

These cases correspond to different real-world scenarios of a message processor:

- **A**: the load is abnormally low, or we are performing a functionality (not load) test; alternatively,
  the data rate may be high but most packets are filtered out early and not stored anywhere.

- **B**: the load is realistic; this case is to be expected most of the time;

- **C**; the load is abnormally high; we don't expect this to last for very long, but must still handle it. This case is the primary reason
so many buffers are allocated in the first place.

Here are the values we'll use:

<table class="numeric">
<tr><th> Variable</th><th> Case <b>A</b> </th><th> Case <b>B</b> </th><th> Case <b>C</b> </th></tr>
<tr><td class="ttext"><code>MAX_CAPTURING_DELAY_MS</code></td><td>        100 </td><td>        100 </td><td>        100 </td></tr>
<tr><td class="ttext"><code>POOL_SIZE</code>             </td><td>  1,000,000 </td><td>  1,000,000 </td><td>  1,000,000 </td></tr>
<tr><td class="ttext"><code>MIX_POOL_SIZE</code>         </td><td>    200,000 </td><td>    200,000 </td><td>    200,000 </td></tr>
<tr><td class="ttext"><code>INTERNAL_QUEUE_SIZE</code>   </td><td>     10,000 </td><td>    100,000 </td><td>    495,000 </td></tr>
<tr><td class="ttext"><code>STORED_COUNT</code>          </td><td>     10,000 </td><td>    100,000 </td><td>    495,000 </td></tr>
</table>

To save space, I won't publish code snippets here. The full source code is in [the repository]({{ site.REPO-ALLOC-OR-POOL }})

Batch test
----------

To measure the cost of our test framework, we'll introduce one more buffer allocating strategy: a dummy one, where we allocate just one packet and use it everywhere.

We'll be running on a dual Xeon&reg; CPU E5-2620 v3 @ 2.40GHz, using Linux with a 3.17.4 kernel and **Java** 1.8.142
with the heap of 2 Gb. We'll start by using a traditional CMS garbage collector, with the following command line:

    # java -Xloggc:gclog -Xms2g -Xmx2g -server Main X [strategy] batch

where `X` is the case label (`A`, `B`, or `C`), and `[strategy]` is the name of the strategy (`dummy`, `alloc`, `mix` or `pool`).

Let's run all the cases and measure time per packet, in nanoseconds:

<table class="numeric">
<tr><th> Strategy </th><th> Case <b>A</b> </th><th> Case <b>B</b> </th><th> Case <b>C</b> </th></tr>
<tr><td class="ttext">Dummy      </td><td>   59 </td><td>         57 </td><td>        66 </td></tr>
<tr><td class="ttext">Allocation </td><td>  400 </td><td>        685 </td><td>      4042 </td></tr>
<tr><td class="ttext">Mix        </td><td>  108 </td><td>        315 </td><td>       466 </td></tr>
<tr><td class="ttext">Pooling    </td><td>  346 </td><td>        470 </td><td>       415 </td></tr>
</table>

The allocating strategy is by far the worst (awful in case **C**), which seems to answer our question (to allocate or to pool): to pool!
The mixed strategy is the best in cases **A** and **B**, and just a bit worse on **C**, which makes it a perfect strategy for the batch test.

We also see that our test code works rather fast (60 ns), so memory allocation, zeroing and garbage collection really slow things down a lot.

Three possible factors can contribute to poor performance of this test: frequent allocation, frequent garbage collection and high garbage collection cost.
The allocation strategy in case **C** suffers from all three; no wonder its performance is so pathetic.

In the case **A** we witnessed a competition between frequent, but fast GC and rare, but slow one (allocation and zeroing costs added in the first option). The slow GC
has won.

The overall picture, however, becomes much less optimistic, and the victory of pooling becomes much less obvious when we look at the garbage collection statistics
(written to `gclog` file by the `-Xloggc` option in the command line). Let's look at these files. All of them contain lots of records about GC pauses,
differing only in their duration, frequency, and type ("`Allocation Failure`" and "`Full GC (ergonomics)`"). Here are the analysis results on these files:

<table class="numeric">
<tr><th> Case</th><th> Strategy</th><th> Max GC pause, ms</th><th>Avg GC pause, ms</th><th>GC count&nbsp;/ sec</th><th>GC fraction</th><th>Object count, mil</th><th>GC time&nbsp;/ object, ns</th></tr>
<tr><td rowspan="3"><b>A&nbsp;</b></td><td class="ttext">Allocation </td><td>   44 </td><td>   9 </td><td>  4.5 </td><td>  4% </td><td> 0.045 </td><td> 194 </td></tr>
<tr>                                   <td class="ttext">Mix        </td><td>   35 </td><td>   6 </td><td>  1.9 </td><td>  1% </td><td> 0.639 </td><td>  10 </td></tr>
<tr>                                   <td class="ttext">Pooling    </td><td>  940 </td><td> 823 </td><td>  0.8 </td><td> 67% </td><td> 3.039 </td><td> 271 </td></tr>

<tr class="even">
    <td rowspan="3"><b>B&nbsp;</b></td><td class="ttext">Allocation </td><td>  176 </td><td>  66 </td><td>  4.5 </td><td> 30% </td><td> 1.134 </td><td>  58 </td></tr>
<tr class="even">                      <td class="ttext">Mix        </td><td>   63 </td><td>  40 </td><td>  0.8 </td><td>  3% </td><td> 1.134 </td><td>  34 </td></tr>
<tr class="even">                      <td class="ttext">Pooling    </td><td>  911 </td><td> 712 </td><td>  0.6 </td><td> 40% </td><td> 3.534 </td><td> 201 </td></tr>
                                                                                                                                                          
<tr><td rowspan="3"><b>C&nbsp;</b></td><td class="ttext">Allocation </td><td>  866 </td><td> 365 </td><td>  2.3 </td><td> 89% </td><td> 5.454 </td><td>  67 </td></tr>
<tr>                                   <td class="ttext">Mix        </td><td>  790 </td><td> 488 </td><td>  0.6 </td><td> 27% </td><td> 5.478 </td><td>  89 </td></tr>
<tr>                                   <td class="ttext">Pooling    </td><td>  576 </td><td> 446 </td><td>  0.6 </td><td> 29% </td><td> 5.508 </td><td>  81 </td></tr>
</table>

Here "GC count" is the average number of GC invocations per second, and "GC fraction" is the percentage of execution time spent in GC.

According to the GC pauses data, the pooling strategy is actually the worst one when it comes to the real-time operation. It simply does not work -- nothing that shows
pauses of nearly one second can be considered real-time. We also see that there is no good solution for case **C**. Neither of our strategies work there.

The allocation case **C** is an exceptionally bad one: it spends 89% of its time in GC. Some more is spent in allocation and zeroing. Pooling, however, can also be bad
(67% in case **A**). It is unclear why the GC load is so much heavier in case **A** than in **B** and **C**.

Just out of curiosity I measured the number of live objects (as reported by **JVisualVM** after invoking GC) and the average GC time per object (the last two columns).
The GC time isn't exactly proportional to the live count, but the overall rule that higher count makes it slower holds. The time per object is surprisingly high.
Roughly 100 ns per object makes it 100ms per one million objects, and one million objects isn't really that many. Most of realistic **Java** programs have more. Some
have much more (hundreds of millions). It looks like those programs can't ever perform anywhere close to real-time -- at least, using the CMS garbage collector.

Real-time test
--------------

Now it's time to check our predictions for the real-time test. This is what the command line and the output looks like for the real-time test with
the allocation strategy and the source interval of 1000 ns for case **A**:

    # java -Djava.library.path=. -Xloggc:gclog -Xms2g -Xmx2g \
           -server Main A alloc 1000
    Test: ALLOC
    Input queue size: 100000
    Input queue capacity, ms: 99.99999999999999
    Internal queue: 1000 = 1 ms
    Stored: 1000
      6.0;   6.0; lost:        0
      7.0;   1.0; lost:        0
      8.0;   1.0; lost:     5717
      9.0;   1.0; lost:        0
     10.1;   1.0; lost:        0
     11.0;   1.0; lost:        0
     12.0;   1.0; lost:        0

No packets are lost afterwards, which means that the test program can handle the load (we tolerate some initial
lack of performance).

Can we, perhaps, increase the incoming packet rate? We can, with gradual deterioration of results. At 500 ns, we get about 80K lost packets after 27 sec and no loss
later. The output for 300 ns looks like this:

     5.5;   5.5; lost:   279184
     5.8;   0.3; lost:   113569
     6.2;   0.3; lost:   111238
     6.5;   0.4; lost:   228014
     6.9;   0.3; lost:   143214
     7.5;   0.6; lost:   296348
     8.1;   0.6; lost:  1334374
 
Experiments show that the minimal delay where packets are not lost is 400 ns (2.5M packets/sec), which matches the result of the batch test very well.


Let's now look at the pooling strategy:

    # java -Djava.library.path=. -Xloggc:gclog -Xms2g -Xmx2g \
           -server Main A pool 1000
    Test: POOL, size = 1000000
    Input queue size: 100000
    Input queue capacity, ms: 99.99999999999999
    Internal queue: 1000 = 1 ms
    Stored: 1000
      6.0;   6.0; lost:        0
      7.0;   1.0; lost:        0
      8.0;   1.0; lost:        0
     10.3;   2.3; lost:  1250212
     11.3;   1.0; lost:        0
     12.3;   1.0; lost:        0
     13.3;   1.0; lost:        0
     15.0;   1.8; lost:   756910
     16.0;   1.0; lost:        0
     17.0;   1.0; lost:        0
     18.0;   1.0; lost:        0
     19.8;   1.8; lost:   768783


This is what we predicted from our batch test experience: the pooling packet processor can't handle the load, because its GC pauses are longer than the input queue capacity.
A quick look at the `gclog` file shows that typical pauses are the same as in the batch test (about 800 ms), and GC runs roughly once every four seconds.

This is very disappointing. No matter what we do, the pooling strategy can't handle even case **A**, not to mention **B** or **C**. Increasing the heap size will make
GC less frequent but won't affect its duration. Increase in source packet interval won't help, either -- for instance, at the interval of 10,000 ns we lose about 80K packets every 40 seconds.
The only thing that helps is increasing the capacity of the source queue to some value above our GC pauses (one second and more), but this isn't always possible.

Here is the combined result for all the tests. The following legend is used:

- the normal value (such as 600) is the minimal source interval, in nanoseconds, where we don't lose packets;

- where such interval doesn't exist (some packets are always lost), the cell contains the word "lost" and two
  values: the percentage of packets lost using the heap of 2 Gb and of 10 Gb, at the source interval of 1000 ns.


<table class="numeric">
<tr><th> Strategy </th><th> Case <b>A</b> </th><th> Case <b>B</b> </th><th> Case <b>C</b> </th></tr>
<tr><td class="ttext">Allocation </td><td>  600       </td><td>  lost:&nbsp;0.8%&nbsp;(0.3%)</td><td> lost:&nbsp;75%&nbsp;(20%)</td></tr>
<tr><td class="ttext">Mix        </td><td>  150       </td><td>        350  </td><td> lost:&nbsp;9%&nbsp;(0.6%)</td></tr>
<tr><td class="ttext">Pooling    </td><td>  lost:&nbsp;17%&nbsp;(0.5%)</td><td>  lost:&nbsp;17%&nbsp;(0.5%)</td><td> lost:&nbsp;9%&nbsp;(0.6%) </td></tr>
</table>

Note that our pool was allocated to handle case **C**. The same pool, but dimensioned for case **B** is called "Mix" and works quite well. This means that pooling
is still better than allocating for the cases that we can handle, and there are cases that we can't handle.

Increasing the heap size reduces losses to almost bearable and "almost solves" the problem (although it doesn't reach the packet loss
level of a couple hundred out of a million discussed above; the losses look more like 5000 per million). As one could expect,
it works much better in the pooling case. However, this approach looks ridiculous: who wants to use 10 Gb of RAM instead of
2 Gb just to reduce the packet loss count from 17% to 0.5%?

G1 garbage collector
--------------------

Until now, we've been using the CMS (concurrent mark-sweep) garbage collector. These days it's considered outdated, and the standard one is the G1 ("garbage first")
collector. It became standard in **Java&nbsp;9** but was available in **Java&nbsp;8** as a run-time option. This one is supposed to be much more real-time-friendly. For instance,
you can specify the maximal allowed GC pause in the command line. This seems to be the way to go, so let's repeat our tests using G1.

Here is the command line for a batch test:

    java -Xloggc:gclog -Xms2g -Xmx2g -XX:+UseG1GC -XX:MaxGCPauseMillis=80 \
         -server Main alloc batch

And here are the results for the batch test (the legend: G1 time / CMS time):

<table class="numeric">
<tr><th> Strategy </th><th> Case <b>A</b> </th><th> Case <b>B</b> </th><th> Case <b>C</b> </th></tr>
<tr><td class="ttext">Dummy      </td><td>   78 / 59  </td><td>         70 / 57  </td><td>        81 / 66 </td></tr>
<tr><td class="ttext">Allocation </td><td>  424 / 400 </td><td>        640 / 685 </td><td>      4300 / 4042 </td></tr>
<tr><td class="ttext">Mix        </td><td>  134 / 108 </td><td>        364 / 315 </td><td>       625 / 466 </td></tr>
<tr><td class="ttext">Pooling    </td><td>  140 / 346 </td><td>        355 / 470 </td><td>       740 / 415 </td></tr>
</table>

In most cases, the execution became slower, between 10% and 130%, with one notable exception: pooling became faster in cases **A** and **B**.

Now let's analyse the garbage collector logs. It is more complex now, since not every line in a G1 log indicates a pause. Some report asynchronous operation, which
doesn't stop the program execution.

<table class="numeric">
<tr><th> Case</th><th> Strategy</th><th> Max GC pause, ms</th><th>Avg GC pause, ms</th><th>GC count&nbsp;/ sec</th><th>GC fraction</th><th>Object count, mil</th><th>GC time&nbsp;/ object, ns</th></tr>
<tr><td rowspan="3"><b>A&nbsp;</b></td><td class="ttext">Allocation </td><td>   56 </td><td>  20 </td><td>  2.4 </td><td>  5% </td><td> 0.045 </td><td> 444 </td></tr>
<tr>                                   <td class="ttext">Mix        </td><td>   43 </td><td>  24 </td><td>  0.5 </td><td>  1% </td><td> 0.639 </td><td>  38 </td></tr>
<tr>                                   <td class="ttext">Pooling    </td><td>   47 </td><td>  21 </td><td>  1.3 </td><td>  3% </td><td> 3.039 </td><td>   7 </td></tr>
<tr class="even">
    <td rowspan="3"><b>B&nbsp;</b></td><td class="ttext">Allocation </td><td>   85 </td><td>  48 </td><td>  5.8 </td><td> 28% </td><td> 1.134 </td><td>  42 </td></tr>
<tr class="even">                      <td class="ttext">Mix        </td><td>   81 </td><td>  65 </td><td>  0.3 </td><td>  2% </td><td> 1.134 </td><td>  57 </td></tr>
<tr class="even">                      <td class="ttext">Pooling    </td><td>   76 </td><td>  62 </td><td>  0.6 </td><td>  3% </td><td> 3.534 </td><td>  17 </td></tr>
<tr><td rowspan="3"><b>C&nbsp;</b></td><td class="ttext">Allocation </td><td>  732 </td><td> 118 </td><td>  2.4 </td><td> 28% </td><td> 5.454 </td><td>  21 </td></tr>
<tr>                                   <td class="ttext">Mix        </td><td>  172 </td><td> 110 </td><td>  2.3 </td><td> 25% </td><td> 5.478 </td><td>  20 </td></tr>
<tr>                                   <td class="ttext">Pooling    </td><td>  173 </td><td> 117 </td><td>  2.0 </td><td> 23% </td><td> 5.508 </td><td>  21 </td></tr>
</table>

This looks much better than with CMS and promises a working solution for case **B**. Let's run the real-time test:

<table class="numeric">
<tr><th> Strategy </th><th> Case <b>A</b> </th><th> Case <b>B</b> </th><th> Case <b>C</b> </th></tr>
<tr><td class="ttext">Allocation </td><td>  750 </td><td>       2000 </td><td>    lost: 76% (13%) </td></tr>
<tr><td class="ttext">Mix        </td><td>  200 </td><td>        600 </td><td>    lost: 4% (1%) </td></tr>
<tr><td class="ttext">Pooling    </td><td>  200 </td><td>        600 </td><td>    lost: 4.4% (0.8%)</td></tr>
</table>

The G1 collector causes mixed feelings: more tests produce results; however, those that do, perform much worse than using traditional CMS. The G1 isn't a magical
tool that solves all the problems: we still have no solution for case **C**.

Pooling still works better than allocating.

ZGC
---

Let's now go through a time warp and skip from **Java&nbsp;8** straight to **Java&nbsp;11**, which has a completely new feature:
[the garbage collector called **ZGC**](https://www.opsian.com/blog/javas-new-zgc-is-very-exciting/),
advertised as capable of handling hundreds of million objects in terabyte heaps.

At the time of writing this article, this garbage collector was only available on Linux and only as an experimental feature.
Let's try it.

The command line looks like this:

    java -Xloggc:gclog -Xms2g -Xmx2g -XX:+UnlockExperimentalVMOptions -XX:+UseZGC 
         -server Main A alloc batch

Here are the batch test results (the legend is ZGC time / G1 time):

<table class="numeric">
<tr><th> Strategy </th><th> Case <b>A</b> </th><th> Case <b>B</b> </th><th> Case <b>C</b> </th></tr>
<tr><td class="ttext">Dummy      </td><td>   72 / 78</td><td>          66 / 70</td><td>        84 / 81 </td></tr>
<tr><td class="ttext">Allocation </td><td>  523 / 424 </td><td>        800 / 640 </td><td>      1880 / 4300 </td></tr>
<tr><td class="ttext">Mix        </td><td>  108 / 134 </td><td>        403 / 364 </td><td>       436 / 625 </td></tr>
<tr><td class="ttext">Pooling    </td><td>  109 / 140 </td><td>        403 / 355 </td><td>       453 / 740 </td></tr>
</table>

The performance went a bit down in some cases and a bit up in some others. In general, it feels like this GC really
handles bigger heaps better than the previous ones.

I haven't found an option to dump the full ZGC log with pause times, so I skip this study for now. Here are the real-time results for ZGC:

<table class="numeric">
<tr><th> Strategy </th><th> Case <b>A</b> </th><th> Case <b>B</b> </th><th> Case <b>C</b> </th></tr>
<tr><td class="ttext">Allocation </td><td>  540 </td><td>        820 </td><td>    lost: 44% (1.7%) </td></tr>
<tr><td class="ttext">Mix        </td><td>  120 </td><td>        420 </td><td>    450        </td></tr>
<tr><td class="ttext">Pooling    </td><td>  130 </td><td>        420 </td><td>    460        </td></tr>
</table>

This is great! Congratulations to **Java** developers. All our cases are covered. One can argue that 450 ns  per packet is too much (only
two million packets per second), but previously we couldn't do this. The figures for other cases also
look good. Pooling still looks better than allocating.

Using pre-allocated native buffers, CMS
---------------------------------------

Although **ZGC** seems to solve our problems, we don't want to stop here. After all, it is still new
and experimental. Firstly, what if we can improve performance of traditional garbage collectors?
Secondly, can we get better throughput with ZGC?

The observed GC performance for traditional collectors seems a bit low, and delays per object figures seem rather high. One idea of why it could be so is that our memory allocation patterns are
different from those the GCs were tuned at. **Java** programs allocate objects whenever they feel like it, that's normal.
However, usually they allocate "normal" objects (structures of several fields), not big arrays. What if this makes a difference?

Here's a plan. We'll move these buffers off heap and make the heap smaller. We allocate them in native memory using direct byte buffers. Then we try our approaches,
with one important difference. Allocation of direct byte buffers is rather expensive (among other things, it invokes `System.gc()`), and deallocation isn't
exactly cheap either (it is done using finalizers that get stored in a special queue and run from a special thread). That's why in both our allocating and pooling
versions we will pool these buffers, and we'll do it off-heap. Apart from that, the allocating version will allocate packet objects each time they are needed
and the pooling version will keep them in a collection. Although the number of packets will be the same as before, the total number of objects will be smaller, because
where previously we had a byte buffer and a byte array, now we only have a byte buffer.

One can argue that the "allocation" strategy now isn't a true "allocation" anymore: we must still implement
some pooling scheme for native buffers. This is true; but we'll still test its performance.

Let's start with CMS GC, batch test. Here is the command line:

    java -Xloggc:gclog -Xms1g -Xmx1g -XX:MaxDirectMemorySize=2g -server \
          Main A native-alloc batch

The **Java** heap size has been reduced by the total size of all the arrays (one gigabyte).

Here are the batch results:

<table class="numeric">
<tr><th> Strategy </th><th> Case <b>A</b> </th><th> Case <b>B</b> </th><th> Case <b>C</b> </th></tr>
<tr><td class="ttext">Dummy      </td><td>   50 </td><td>         53 </td><td>        58 </td></tr>
<tr><td class="ttext">Allocation </td><td>   89 </td><td>        253 </td><td>       950 </td></tr>
<tr><td class="ttext">Mix        </td><td>   83 </td><td>        221 </td><td>       298 </td></tr>
<tr><td class="ttext">Pooling    </td><td>   79 </td><td>        213 </td><td>       260 </td></tr>
</table>

The results (except for Allocation case **C**) look very good and all of them are much better than anything we've seen so far. This seems to be a perfect option for batch processing.

Let's look at real-time results:

<table class="numeric">
<tr><th> Strategy </th><th> Case <b>A</b> </th><th> Case <b>B</b> </th><th> Case <b>C</b> </th></tr>
<tr><td class="ttext">Allocation </td><td>  140       </td><td>  lost: 0.8% </td><td> lost: 34%  </td></tr>
<tr><td class="ttext">Mix        </td><td>  130       </td><td>  250; lost: 0.0025% </td><td> lost: 0.7% </td></tr>
<tr><td class="ttext">Pooling    </td><td>  120       </td><td>  300; lost: 0.03% </td><td> lost: 0.7% </td></tr>
</table>

Note the new notation: "250; lost: 0.0025%" means that, while we still lose packets (loss is measured, as usual, at 1000 ns interval),
the loss is small enough to raise the question of the smallest suitable interval. In short, this is an "almost working" solution.
We can see why if we look at the GC logs. 

The GC log for pooling case C looks like this:

    60.618: [GC (Allocation Failure)  953302K->700246K(1010688K), 0.0720599 secs]
    62.457: [GC (Allocation Failure)  973142K->717526K(1010176K), 0.0583657 secs]
    62.515: [Full GC (Ergonomics)  717526K->192907K(1010176K), 0.4102448 secs]
    64.652: [GC (Allocation Failure)  465803K->220331K(1011712K), 0.0403231 secs]

Roughly every two seconds there is a short run of the GC, which collects about 200MB, but memory usage still climbs by 20MB each time.
Eventually, we run out of memory and every 60 sec there is a 400-msec GC, which causes loss of about 350K packets. This isn't too bad.

The "**B**" case is even better: the full GC is only called every 1100 seconds, which roughly corresponds to 0.03% of lost packets
(300 out of one million). Even less so for the mixing scheme. This makes even production use of this solution possible.

Native buffers, G1
------------------

Here are the batch results:

<table class="numeric">
<tr><th> Strategy </th><th> Case <b>A</b> </th><th> Case <b>B</b> </th><th> Case <b>C</b> </th></tr>
<tr><td class="ttext">Dummy      </td><td>   62 </td><td>         63 </td><td>        79 </td></tr>
<tr><td class="ttext">Allocation </td><td>  108 </td><td>        239 </td><td>      1100 </td></tr>
<tr><td class="ttext">Mix        </td><td>  117 </td><td>        246 </td><td>       432 </td></tr>
<tr><td class="ttext">Pooling    </td><td>  111 </td><td>        249 </td><td>       347 </td></tr>
</table>

The results are better than without native buffers, but worse than the CMS batch results.

Here are the results for the real-time tests:

<table class="numeric">
<tr><th> Strategy </th><th> Case <b>A</b> </th><th> Case <b>B</b> </th><th> Case <b>C</b> </th></tr>
<tr><td class="ttext">Allocation </td><td>  150       </td><td>  350 </td><td> lost: 6.5% </td></tr>
<tr><td class="ttext">Mix        </td><td>  150       </td><td>  400 </td><td> 800; lost: 0.075% </td></tr>
<tr><td class="ttext">Pooling    </td><td>  160       </td><td>  500 </td><td> 700 </td></tr>
</table>

Although it looks a bit worse than the CMS results in case A, and the achieved results in case B are worse than "almost achieved" results there,
these results demonstrate one incredible feature we've never seen before: they cover all the cases. Even pooling case **C** is now working.

Native buffers, ZGC
-------------------

Now let's try the ZGC at the batch test (results are compared with the ZGC results without native buffers):

<table class="numeric">
<tr><th> Strategy </th><th> Case <b>A</b> </th><th> Case <b>B</b> </th><th> Case <b>C</b> </th></tr>
<tr><td class="ttext">Dummy      </td><td>   63 / 72</td><td>          76 / 66</td><td>        102 / 84 </td></tr>
<tr><td class="ttext">Allocation </td><td>  127 / 523 </td><td>        290 / 800 </td><td>       533 / 1880 </td></tr>
<tr><td class="ttext">Mix        </td><td>  100 / 108 </td><td>        290 / 403 </td><td>       400 / 436 </td></tr>
<tr><td class="ttext">Pooling    </td><td>  118 / 109 </td><td>        302 / 403 </td><td>       330 / 453 </td></tr>
</table>

There is a significant improvement almost everywhere, especially in allocation tests. However,
G1, and especially CMS, results are still much better.

Finally, here are the real-time results:

<table class="numeric">
<tr><th> Strategy </th><th> Case <b>A</b> </th><th> Case <b>B</b> </th><th> Case <b>C</b> </th></tr>
<tr><td class="ttext">Allocation </td><td>  170 </td><td>        380 </td><td>    550 </td></tr>
<tr><td class="ttext">Mix        </td><td>  120 </td><td>        320 </td><td>    440 </td></tr>
<tr><td class="ttext">Pooling    </td><td>  130 </td><td>        320 </td><td>    460 </td></tr>
</table>

Again, congratulations to the **Java** engineers. For the first time ever, we've got a working solution for all strategies
and all cases. Even allocating strategy in case **C** works now (although when native buffers are used it is not completely "allocating"
strategy; besides, the pooling solution still shows better results).

Trying C++
----------

We've seen that memory management is really affecting performance of **Java** programs.
We could try to reduce these overheads by employing our own off-heap memory manager (I'm still going to explore this technique
in one of the following articles). However, if we're prepared to go that far, we could just as well try writing in **C++**.

The garbage collection issue does not exist in **C++**; we can keep as many live objects as we wish, it won't introduce any pauses.
It may reduce performance due to poor caching, but that is another story.

This makes the choice between allocation and pooling obvious: no matter how small the allocation costs are, pooling costs are zero.
So pooling is bound to win. Let's test it.

Our first version will be a direct translation of the **Java** version, with the same design features. Specifically,
we'll allocate `IPHeader` and `IPV4Address` objects when needed. This makes the dummy version leak
memory, as the same buffer object is re-used many times without returning to the pool, and no one
deletes those header objects in the process.

Here are the batch results:

<table class="numeric">
<tr><th> Strategy </th><th> Case <b>A</b> </th><th> Case <b>B</b> </th><th> Case <b>C</b> </th></tr>
<tr><td class="ttext">Dummy      </td><td>  145 </td><td>        164</td><td>        164 </td></tr>
<tr><td class="ttext">Allocation </td><td>  270 </td><td>        560 </td><td>       616 </td></tr>
<tr><td class="ttext">Mix        </td><td>  115 </td><td>        223 </td><td>       307 </td></tr>
<tr><td class="ttext">Pooling    </td><td>  111 </td><td>        233 </td><td>       274 </td></tr>
</table>

The results look good but, surprisingly, not great. We've seen better results at native-buffers CMS
solution in **Java**. Some other **Java** results are also better. The allocation strategy results
are as bad as most of those in **Java**, and, surprisingly, dummy results are bad. This points
to memory allocation being quite expensive in **C++**, much more expensive than in **Java**, even
without the GC.

Here are the real-time figures:

<table class="numeric">
<tr><th> Strategy </th><th> Case <b>A</b> </th><th> Case <b>B</b> </th><th> Case <b>C</b> </th></tr>
<tr><td class="ttext">Allocation </td><td>  520 </td><td>        950 </td><td>    950 </td></tr>
<tr><td class="ttext">Mix        </td><td>  280 </td><td>        320 </td><td>    550 </td></tr>
<tr><td class="ttext">Pooling    </td><td>  250 </td><td>        420 </td><td>    480 </td></tr>
</table>

The figures don't look bad (at least, all the cases are covered), but the figures from **Java** using ZDC and native buffers look better.
The way to go in **C++** must be to reduce memory allocation wherever possible.

C++: no allocation
------------------

The previous solution was implemented in a **Java** fashion: allocate an object (such as an `IPv4Address`), when it is needed.
In **Java** there was no alternative, but in **C++** there is: we can reserve the memory
for the most commonly used objects right inside our buffers.
This will result in reducing memory allocation during packet processing to zero. We'll call it the
flat **C++** version.

Here are the batch results:

<table class="numeric">
<tr><th> Strategy </th><th> Case <b>A</b> </th><th> Case <b>B</b> </th><th> Case <b>C</b> </th></tr>
<tr><td class="ttext">Dummy      </td><td>  16   </td><td>       16 </td><td>       16 </td></tr>
<tr><td class="ttext">Allocation </td><td>  163  </td><td>      409 </td><td>      480 </td></tr>
<tr><td class="ttext">Mix        </td><td>  35   </td><td>      153 </td><td>      184 </td></tr>
<tr><td class="ttext">Pooling    </td><td>  34   </td><td>      148 </td><td>      171 </td></tr>
</table>

All the figures are much better than those of our **Java** tests. The mixed and pooling
results are also very good in the absolute sense.

And the real-time results look like this:

<table class="numeric">
<tr><th> Strategy </th><th> Case <b>A</b> </th><th> Case <b>B</b> </th><th> Case <b>C</b> </th></tr>
<tr><td class="ttext">Allocation </td><td>  220 </td><td>        650 </td><td>    700 </td></tr>
<tr><td class="ttext">Mix        </td><td>   50 </td><td>        220 </td><td>    240 </td></tr>
<tr><td class="ttext">Pooling    </td><td>   50 </td><td>        190 </td><td>    230 </td></tr>
</table>

Some **Java** versions provide better results for the allocating strategy; one (native ZGC) even
performs better in the case **C**, which can be attributed to slow and unpredictable nature of
the **C++** memory manager. All the other versions, however, perform very well. The pooling
version can handle four million packets per second in case **C** and five million in case **B**,
which reaches our desired value. The speed of case **A** is absolutely fantastic (twenty million),
but we must remember that in this case we discard the packets.

Since no memory allocation is performed at all in pooling versiuon, the difference
in speed between cases **A**, **B** and **C** can only be explained by different total volume
of used memory -- more used memory and random access patterns reduce cache efficiency.

Summary
-------

Let's summarise all the results in one table. We'll leave out dummy strategy results and the results
obtained with ridiculously high memory heap sizes.

Let's first look at the batch test:

<table class="numeric">
<tr><th> Solution </th><th> Strategy </th><th> Case <b>A</b> </th><th> Case <b>B</b> </th><th> Case <b>C</b> </th></tr>

<tr class="even"><td class="ttext" rowspan="3"> CMS </td>
    <td class="ttext">Allocation </td><td>  400 </td><td>        685 </td><td>      4042 </td></tr>
<tr class="even"><td class="ttext">Mix        </td><td>  108 </td><td>        315 </td><td>       466 </td></tr>
<tr class="even"><td class="ttext">Pooling    </td><td>  346 </td><td>        470 </td><td>       415 </td></tr>
<tr><td class="ttext" rowspan="3"> G1 </td>
    <td class="ttext">Allocation </td><td>  424 </td><td>        640 </td><td>      4300 </td></tr>
<tr><td class="ttext">Mix        </td><td>  134 </td><td>        364 </td><td>       625 </td></tr>
<tr><td class="ttext">Pooling    </td><td>  140 </td><td>        355 </td><td>       740 </td></tr>
<tr class="even"><td class="ttext" rowspan="3"> ZGC </td>
    <td class="ttext">Allocation </td><td>  523 </td><td>        800 </td><td>      1880</td></tr>
<tr class="even"><td class="ttext">Mix        </td><td>  108 </td><td>        403 </td><td>       436 </td></tr>
<tr class="even"><td class="ttext">Pooling    </td><td>  109 </td><td>        403 </td><td>       453 </td></tr>
<tr><td class="ttext" rowspan="3"> Native CMS </td>
    <td class="ttext">Allocation </td><td>   89 </td><td>        253 </td><td>       950 </td></tr>
<tr><td class="ttext">Mix        </td><td class="red">      83 </td><td class="red">     221 </td><td class="red">    298 </td></tr>
<tr><td class="ttext">Pooling    </td><td class="yellow">   79 </td><td class="yellow">  213 </td><td class="yellow"> 260 </td></tr>
<tr class="even"><td class="ttext" rowspan="3"> Native G1 </td>
    <td class="ttext">Allocation </td><td>  108 </td><td>        239 </td><td>      1100 </td></tr>
<tr class="even"><td class="ttext">Mix        </td><td>  117 </td><td>        246 </td><td>       432 </td></tr>
<tr class="even"><td class="ttext">Pooling    </td><td>  111 </td><td>        249 </td><td>       347 </td></tr>
<tr><td class="ttext" rowspan="3"> Native ZGC </td>
    <td class="ttext">Allocation </td><td>  127 </td><td>        290 </td><td>       533 </td></tr>
<tr><td class="ttext">Mix        </td><td>  100 </td><td>        290 </td><td>       400 </td></tr>
<tr><td class="ttext">Pooling    </td><td>  118 </td><td>        302 </td><td>       330 </td></tr>
<tr class="even"><td class="ttext" rowspan="3"> C++ </td>
    <td class="ttext">Allocation </td><td>  270 </td><td>        560 </td><td>       616 </td></tr>
<tr class="even"><td class="ttext">Mix        </td><td>  115 </td><td>        223 </td><td>       307 </td></tr>
<tr class="even"><td class="ttext">Pooling    </td><td>  111 </td><td>        233 </td><td>       274 </td></tr>
<tr><td class="ttext" rowspan="3"> C++ flat </td>
    <td class="ttext">Allocation </td><td>  163 </td><td>        409 </td><td>       480 </td></tr>
<tr><td class="ttext">Mix        </td><td>  35 </td><td>        153 </td><td>        184 </td></tr>
<tr><td class="ttext">Pooling    </td><td class="green">  34 </td><td class="green">  148 </td><td class="green"> 171 </td></tr>
</table>

The absolute best result in every column is marked green, and all three of them happened to
be from the flat **C++**.

The best and the second best **Java** results are marked yellow and red. They come from "Native CMS"
family, which shows that it is too early to write off the good old CMS garbage collector. It still
works well for batch processing.
 
And finally, here is the summary of the results that were our main interest, those of the real-time test:

<table class="numeric">
<tr><th> Strategy </th><th> Solution </th><th> Case <b>A</b> </th><th> Case <b>B</b> </th><th> Case <b>C</b> </th></tr>

<tr class="even"><td class="ttext" rowspan="3"> CMS </td>
    <td class="ttext">Allocation </td><td>                          600 </td><td class="gray">     lost:&nbsp;0.8% </td><td class="gray"> lost:&nbsp;75%</td></tr>
<tr class="even"><td class="ttext">Mix        </td><td>                          150 </td><td class="red">                  350 </td><td class="gray"> lost:&nbsp;9%</td></tr>
<tr class="even"><td class="ttext">Pooling    </td><td class="gray">  lost:&nbsp;17%</td><td class="gray">       lost:&nbsp;17% </td><td class="gray"> lost:&nbsp;9 </td></tr>
<tr><td class="ttext" rowspan="3"> G1 </td>
    <td class="ttext">Allocation </td><td>  750 </td><td>          2000 </td><td class="gray">           lost: 76% </td></tr>
<tr><td class="ttext">Mix        </td><td>  200 </td><td>           600 </td><td class="gray">            lost: 4% </td></tr>
<tr><td class="ttext">Pooling    </td><td>  200 </td><td>           600 </td><td class="gray">          lost: 4.4% </td></tr>
<tr class="even"><td class="ttext" rowspan="3"> ZGC </td>
    <td class="ttext">Allocation </td><td>  540 </td><td>           820 </td><td class="gray">           lost: 44% </td></tr>
<tr class="even"><td class="ttext">Mix        </td><td class="yellow">           120 </td><td>                              420 </td><td class="red">    450 </td></tr>
<tr class="even"><td class="ttext">Pooling    </td><td class="red">              130 </td><td>                              420 </td><td>                460 </td></tr>
<tr><td class="ttext" rowspan="3"> Native CMS </td>
    <td class="ttext">Allocation </td><td>                          140 </td><td class="gray">  lost: 0.8%         </td><td class="gray"> lost: 34%  </td></tr>
<tr><td class="ttext">Mix        </td><td class="red">              130 </td><td class="gray">  lost:&nbsp;0.0025% </td><td class="gray"> lost: 0.7% </td></tr>
<tr><td class="ttext">Pooling    </td><td class="yellow">           120 </td><td class="gray">  lost:&nbsp;0.03%   </td><td class="gray"> lost: 0.7% </td></tr>
<tr class="even"><td class="ttext" rowspan="3"> Native G1 </td>
    <td class="ttext">Allocation </td><td>                          150 </td><td class="red">                  350 </td><td class="gray"> lost: 6.5% </td></tr>
<tr class="even"><td class="ttext">Mix        </td><td>                          150 </td><td>                              400 </td><td class="gray"> lost:&nbsp;0.075% </td></tr>
<tr class="even"><td class="ttext">Pooling    </td><td>                          160 </td><td>                              500 </td><td>                700 </td></tr>
<tr><td class="ttext" rowspan="3"> Native ZGC </td>
    <td class="ttext">Allocation </td><td>                          170 </td><td>                              380 </td><td>                550 </td></tr>
<tr><td class="ttext">Mix        </td><td class="yellow">           120 </td><td class="yellow">               320 </td><td class="yellow"> 440 </td></tr>
<tr><td class="ttext">Pooling    </td><td class="red">              130 </td><td class="yellow">               320 </td><td>                460 </td></tr>
<tr class="even"><td class="ttext" rowspan="3"> C++ </td>
    <td class="ttext">Allocation </td><td>                          520 </td><td>                              950 </td><td>                950 </td></tr>
<tr class="even"><td class="ttext">Mix        </td><td>                          280 </td><td>                              320 </td><td>                550 </td></tr>
<tr class="even"><td class="ttext">Pooling    </td><td>                          250 </td><td>                              420 </td><td>                480 </td></tr>
<tr><td class="ttext" rowspan="3"> C++ flat </td>
    <td class="ttext">Allocation </td><td>                          220 </td><td>                              650 </td><td>                700 </td></tr>
<tr><td class="ttext">Mix        </td><td>                           50 </td><td>                              220 </td><td>                240 </td></tr>
<tr><td class="ttext">Pooling    </td><td class="green">             50 </td><td class="green">                190 </td><td class="green">  230 </td></tr>
</table>

The dark grey blocks indicate absence of solution (packets are always lost). Otherwise, the
colour scheme is the same. The flat **C++** version is the best again, while the best and the second-best
**Java** versions come from all possible solutions, the best being the Native ZGC.

Conclusions
-----------

- If you are writing a true real-time system, write it in **C** or **C++** and avoid memory allocation.

- However, some reasonably good approximation to real-time can be implemented in **Java**, too.

- It helps to reduce memory allocations in this case as well.

- **This answers our initial question (to allocate or to pool): to pool.** In every single test that we ran,
  pooling performed better than allocation; moreover, in most **Java** tests allocation performed
  awfully in batch mode and did not perform at all in real-time mode.

- Mixed approach proved very good when packet utilisation is low. However, if it grows, pooling
  becomes better.

- The garbage collector is indeed the biggest risk factor. Pooling introduces very many live objects
  that causes infrequent, but long GC delays. Allocation, however, overloads GC completely. Besides,
  at the times of high load (our case **C**) there may be many live objects anyway, and there
  allocation fails miserably. So pooling is still better.

- There is a reason behind G1 and ZGC collectors. While often (however, not always) performing worse
  in batch mode, they really improve things in real-time mode. The ZGC performed especially well;
  it made it possible to handle even case **C** with reasonable performance (2 million packets per second).

- Everything would have been much worse if we allocated ten million buffers instead of one million,
  or if the program used some other big data structures. One possible solution is to move these
  structures, where under our control, off-heap.

- Where immediate response to the incoming messages is not required, increasing the input queue
  size may help. Where it is impossible (input hardware limitation), we can consider introducing
  another layer in **C** and another intermediate queue of higher capacity. If our source queue
  could contain one second worth of packets, pooling version would work well even with CMS.

Comments are welcome below or on [reddit](https://www.reddit.com/r/java/comments/ahaedh/to_allocate_or_to_pool/).
