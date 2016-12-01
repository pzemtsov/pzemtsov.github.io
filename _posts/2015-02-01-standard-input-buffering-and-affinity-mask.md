---
layout: post
title:  "Standard input, buffering and affinity mask"
date:   2015-02-01 12:00:00
tags: Java optimisation performance streams buffering stdin stdout affinity
story: buffering
story-title: "Effect of stream buffering"
ART-BUFFER: /2015/01/19/on-the-benefits-of-stream-buffering-in-Java.html
REPO-BUFFER: https://github.com/pzemtsov/article-buffering-streams
---

In the [previous article]({{ page.ART-BUFFER }}) we measured effect of input data buffering on TCP sockets,
using both **Java** `InputStream`s and `Channel`s. The measurements were taken on a `1 Gbit` Ethernet adapter, a `10 Gbit`
Ethernet adapter and a loopback IP adapter. The latter triggered the idea to look at communication over
standard input and standard output. Is it faster than over TCP sockets? How does it respond to buffering?

Stream-based code
-----------------

**Java** exposes standard input and output to programs via `System.in` and `System.out` variables, of types
`InputStream` and `PrintStream` (which is also an `OutputStream`) respectively. Neither of them provides any way to
convert it into a channel, so there isn't a portable way to access standard streams using channels. We'll have
to limit ourselves to stream-based operation for now. The classes `StdOutServer` and `StdInClient` can be easily produced
from the socket-based versions (see in [repository]({{ page.REPO-BUFFER }}/commit/239ae47ff8cbbf7d955e7a826452587acf307409)).

This is the result for the original version (the results vary quite a bit, I choose the most common one):

    # java StdOutServer | java StdInClient
    Time for 10000000 msg: 1675; speed: 5970149 msg/s; 644.8 MB/s

This is very good, it is faster than a buffered stream-based solution for TCP sockets (`520 MB/s`).

Now let's introduce a `BufferedInputStream`:

    Time for 10000000 msg: 3546; speed: 2820078 msg/s; 304.6 MB/s

Buffering of the `System.in` made the program slower -- it now runs at less than half the speed. Why is it so?
What is the `System.in` anyway? Let's print it:

{% highlight Java %}
System.out.println (System.in);
{% endhighlight %}

We get:

    java.io.BufferedInputStream@15db9742

This explains the slow speed. The stream is already buffered. Obviously, wrapping the `BufferedInputStream` into
another `BufferedInputStream` can only reduce speed (although I didn't expect so much of the speed loss).

Replacing the `readInt` from the optimised version (see [here]({{ page.REPO-BUFFER }}/commit/854a9a87d56114eb01011c01332f1368ddd2c796)), we get:

    Time for 10000000 msg: 1669; speed: 5991611 msg/s; 647.1 MB/s

Surprisingly, the optimisation wasn't very successful in our case. The speed improved marginally.
I'm not sure why.

[The version without memory allocation]({{ page.REPO-BUFFER }}/commit/cd1153d9d1e821e090bb25c74618cc45210abf30) runs like this:

    Time for 10000000 msg: 1456; speed: 6868131 msg/s; 741.8 MB/s

Again there is a difference with the socket case. This optimisation did make execution faster (in the socket case
it did not).

Channel-based code
------------------

There is no portable way to create channels based on the standard input and output. However, there is an unportable way.
In Linux, the files corresponding to all file handles are available via paths in the `/proc` filesystem.
This is how we can access standard input as a channel:

{% highlight Java %}
FileChannel chan = new FileInputStream ("/proc/self/fd/0")
                   .getChannel ();
{% endhighlight %}

And this is how we access standard output:

{% highlight Java %}
FileChannel chan = new FileOutputStream ("/proc/self/fd/1")
                   .getChannel ();
{% endhighlight %}

We'll now modify both the server and the client to work with channels. We'll test two versions of the client.
First, the one based on the improved channel version from the previous article ([here]({{ page.REPO-BUFFER }}/commit/18767998b9eca38382adc0fb4921aa3c0fe6d3c2)):

    Time for 10000000 msg: 1496; speed: 6684491 msg/s; 721.9 MB/s
    Time for 10000000 msg: 1615; speed: 6191950 msg/s; 668.7 MB/s
    Time for 10000000 msg: 1506; speed: 6640106 msg/s; 717.1 MB/s
    Time for 10000000 msg: 1533; speed: 6523157 msg/s; 704.5 MB/s

Next, the zero-copy version which serves data for processing in a byte buffer ([here]({{ page.REPO-BUFFER }}/commit/42e481ff3abbf923050dafe79dc305f6191a9e19)):

    Time for 10000000 msg: 1383; speed: 7230657 msg/s; 780.9 MB/s
    Time for 10000000 msg: 1325; speed: 7547169 msg/s; 815.1 MB/s
    Time for 10000000 msg: 1349; speed: 7412898 msg/s; 800.6 MB/s
    Time for 10000000 msg: 1473; speed: 6788866 msg/s; 733.2 MB/s

The speed of the last two versions is very unstable. It looks like the zero-copy one is faster than both the regular one
and the best stream-based one (no-alloc version, `741 MB/s`), but it is very difficult to
verify it on this data.

Managing process affinity 
-------------------------

One possible reason for the unstable performance is the variation in scheduling of the client and the server processes
to available processor cores. If this schedulling affects the execution speed, and the processes are moved from one
core to another while running, this could explain the instability.

Analysing the output of Linux `top` command shows that the processes do in fact get moved from one core to another
(although not as often as on Windows). We need a clean test, where each process runs on its own dedicated core.
This can be achieved using process affinity mask. This mask is a bit string (often represented as a single integer
value), where each bit enables or disables execution of a process on corresponding core. There are functions to
control process and thread affinity programmatically, even from the process of thread itself, but in our case
the easiest is to use Linux `taskset` utility. It accepts a mask (int value; it allows decimal and hexadecimal
input) and a command line; it runs the specified command using the specified affinity.

The system I use for testing has two physical processors with eight cores each (four real cores and four hyper-threading
cores). This makes `16` logical cores, numbered from `0` to `15`. Even numbers correspond to one physical processor
and odd numbers to another one. This isn't guarranteed; another machine where I run my tests allocates logical
cores `0` to `7` to one processor and those from `8` to `15` to the other one. The exact numbering can be established
by analysing file `/proc/cpuinfo` -- that's what I did to establish mine.

The way to run the last test (the zero-copy buffer-based solution) on the same processor is

    # taskset 1 java StdOutServer | taskset 4 java StdInClient
    Time for 10000000 msg: 1040; speed: 9615384 msg/s; 1038.5 MB/s
    Time for 10000000 msg: 1072; speed: 9328358 msg/s; 1007.5 MB/s
    Time for 10000000 msg: 1066; speed: 9380863 msg/s; 1013.1 MB/s
    Time for 10000000 msg: 1074; speed: 9310986 msg/s; 1005.6 MB/s

The way to run it of different processors is

    # taskset 1 java StdOutServer | taskset 2 java StdInClient
    Time for 10000000 msg: 1250; speed: 8000000 msg/s; 864.0 MB/s
    Time for 10000000 msg: 1245; speed: 8032128 msg/s; 867.5 MB/s
    Time for 10000000 msg: 1242; speed: 8051529 msg/s; 869.6 MB/s
    Time for 10000000 msg: 1239; speed: 8071025 msg/s; 871.7 MB/s

We can see that there is definitely a big difference in the results. Besides, the speed is more stable now --
there are still variations, but they are much smaller than before. The affinity mask really matters!

There are two possible reasons for this speed difference. One is that each processor has its own `L3` cache,
which is shared between all the cores there. If internally the pipe that serves as the standard input for one process
and the standard output for another one is represented as some memory buffer, the data written by one processor may still
be in the cache by the time it is read by another one. Another possible reason may be the
[**NUMA**](http://en.wikipedia.org/wiki/Non-uniform_memory_access) effects. We'll
talk about **NUMA** in one of the future articles.

Updated measurements
--------------------

A substantial effect of the process affinity means that all our previous measurements were done incorrectly.
In fact, any measurement of the inter-thread or inter-process interaction must be done in stable affinity
environment. Threads jumping from one core to another may reduce accuracy of measurements greatly.

In addition to our two cases (processes running on the same or on different processors), there is a third case:
both processes running on the same core. This is hardly practical, but for the sake of completeness we'll measure
this as well.  Let's redo all previous tests for all three cases:


<table class="numeric">
<tr><th rowspan="2"> Test </th><th colspan="3"> Speed, MB/s </th></tr>
<tr><th> Different processors</th><th> Same processor </th><th> Same core </th></tr>

<tr><td class="ttext">Original                  </td><td>  587 </td><td>  736 </td><td>  570 </td></tr>
<tr><td class="ttext">Fixed <code>readInt</code></td><td>  539 </td><td>  680 </td><td>  515 </td></tr>
<tr><td class="ttext">No memory allocation      </td><td>  638 </td><td>  828 </td><td>  572 </td></tr>
<tr><td class="ttext">Channel-based             </td><td>  651 </td><td>  798 </td><td>  1531 </td></tr>
<tr><td class="ttext">Channel-based, no copy    </td><td>  867 </td><td> 1013 </td><td>  1782 </td></tr>
</table>


As we see, all the versions benefit from running on the same processor. Channel-based versions benefit
further from running on the same core. I think, the reason for the speedup is that the processes share per-core
caches (`L1` and `L2`), not only `L3`. On the other hand, they share a core, and therefore have fewer CPU cycles
available to each. It looks like the stream-based versions require more cycles (they copy more data around),
that's why they do not show benefits from cache sharing.

These observations have more of a theoretical value and mustn't be regarded as an instruction to place processes
communicating over a pipe onto the same processor. After all, processes usually do much more than just communicating
with each other, so their allocation to the processors must be decided on other factors. However, I can imagine
cases when such allocation makes sense. For instance, if we run several clients and several servers, each client
connected to one server and each process requiring exactly one core, then it makes perfect sense to place
the pairs together rather than, say, put all the clients on one processor and all the servers to the other one.

We can also see that, except for the zero-copy versions, channels do not perform much faster than the streams here.
It was different in the socket case.

Unbuffered operation
--------------------

We never had a chance to check how standard input and standard output respond to buffering. Out initial
stream-based test was already buffered, thanks to **Java** library, channel-based tests also provided some
buffering. What would the performance have been if there was no buffering? The trick using `/proc/self/fd/0`
can answer this question. We'll modify the initial version to work with this file
([see here]({{ page.REPO-BUFFER }}/commit/b869482c4959d4e27d7e3dddd5d30627207e4401)):

<table class="numeric">
<tr><th rowspan="2"> Test </th><th colspan="2"> Speed, MB/s </th></tr>
<tr><th> Different processors</th><th> Same processor </th></tr>

<tr><td class="ttext">Original                  </td><td>  41.5 </td><td>  50.8 </td></tr>
<tr><td class="ttext">Fixed <code>readInt</code></td><td>  78.4 </td><td>  80.6 </td></tr>
</table>

The performance is quite poor, no matter what process affinity is. The **Java** runtime designers were right
when providing `System.in` stream with default buffering.


Conslusions
-----------

- Just like socket operations, reading from the standard input benefits a lot from stream buffering.

- **Java** streams representing standard input (`System.in`) are buffered
  already, so there is no need to buffer them again.

- The speed of these streams is quite reasonable, but the ultimate speed achieved using TCP sockets is much higher
  (see the [previous article]({{ page.ART-BUFFER }})). However, before converting pipes to sockets one should make completely sure that this
  communication is the bottleneck. The measured speed is satisfactory for most practical needs.

- If the program is not written in **Java**, the streams may happen to be unbuffered, which will slow things down a lot
  (our unbuffered test proves it). Check the behaviour of the I/O library in your language and, if necessary,
  add buffering.

- Affinity is an important factor for performance of multi-processed and multi-threaded programs. Some affinity
  masks may be better than the others, so controlling them can sometimes improve performance quite a bit.
  But, even more importantly, setting fixed affinity helps maintain consistency of the tests.
