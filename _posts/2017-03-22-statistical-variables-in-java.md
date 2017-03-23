---
layout: post
title:  "Statistical variables in Java: not quite legal but very fast"
date:   2017-03-22 12:00:00
tags: Java optimisation parallel
story: multithreading
story-title: "Multi-threading performance"
---

Today we are going to do some multi-threaded programming in **Java**. We'll look at some shared data structures and try 
several locking and non-locking approaches. Not all the code, including the final solution, will be completely correct from the **Java** spec point of view;
some will be completely incorrect. We'll check what consequences it will have.

The source code can be found in [this repository]({{ site.REPO-STATS }}).

The problem definition: counters and gauges
-------------------------------------------

The problem originates from the real life. Our project included a statistical sub-system that measured and accumulated some values for diagnostic purposes.
There were two types of statistics: counters and gauges.

A **counter** is an object that accumulates the sum or reported values. Here are the examples:

- a number of requests executed
- a number of bytes processed
- a number of packets received
- a number of times the program executed a specific function or a path in the control flow graph.

We'll call the code that reports counted values **"a client"**. Here is the client-side interface -- the `Counter` class:

{% highlight Java %}
public abstract class Counter
{
    public abstract void add (long increment);

    public void add () { add (1); }
}
{% endhighlight %}

The statistic engine (**"the server"**) collects and resets the counter regularly (we'll call the time between collections **"the collection interval"**; it was one minute
in our project). For that it uses the server-side part of the interface:

{% highlight Java %}
public abstract class ServerCounter extends Counter
{
    public abstract long getAndReset ();
}
{% endhighlight %}

A **gauge** is an object that stores a result of an instant measurement of some variable and performs basic aggregations on these values -- minimum, maximum, average.
The examples are:

- the amount of allocated memory
- the disk utilisation
- the CPU utilisation as reported by OS
- the size of some queue

The need to calculate the average requires recording the number of invocations and the sum of values, so each gauge always contains two counters inside. This technically allows us to
implement only gauges and use them as counters. However, we'll see soon that counters may be implemented more efficiently than gauges.

The gauges can be used as "weighted counters" to record events that are accompanied by a number. The examples are:

- the size of an incoming message
- the time it took to serve a request

Here is the client-side `Gauge` definition:

{% highlight Java %}
public abstract class Gauge
{
    public abstract void report (long value);
}
{% endhighlight %}

Some gauges do not require frequent reporting of the value. For instance, it's sufficient to record CPU utilisation several times a minute. Some, however, benefit from frequent
update. Such are the gauges of the second type, but not only them. If we can afford to record the queue size every time we put something into the queue or get something,
the measurement will be much more accurate than random sampling, and we'll be much less likely to miss times when the queue is full or empty.

The server-size interface is more complex than for `Counter`, because all the values must be retrieved at once:

{% highlight Java %}
public abstract class GaugeValue
{
    public abstract long getCount ();
    public abstract long getSum ();
    public abstract long getMax ();
    public abstract long getMin ();
}

public abstract class ServerGauge extends Gauge
{
    public abstract GaugeValue getAndReset ();
}
{% endhighlight %}

These statistics are a bit like trace log files: their purpose is to help developers find what went wrong when something does. They are not shown to the users.
The program logic does not depend on them. They are not mission-critical. This doesn't mean they may be totally wrong, but if one of the counters is out by one or two,
it is not the end of the world. However, they must have one very important feature: be fast.

How fast is fast?
-----------------

Here we must clarify, what is fast? This is the reasoning.

Let's assume we are making a handler of some incoming requests. It doesn't perform any sophisticated processing, only initial parsing and some pre-checks.
It figures out who must serve the request and dispatches it further down the processing chain. A typical performance of such a handler on modern CPUs is somewhere between one million and ten million
operations per second per core, which translates to 100 to 1000 nanoseconds per request (which, on a typical Xeon, means between 250 and 2500 cycles). Several statistic
variables may be updated during each of these operations. Moreover, we should never hesitate to add another one if that's needed.
This gives us a rough estimate of the time we can afford to spend on a counter update: 5 to 10 ns is ideal, and 20 ns is probably still reasonable.
Anything above 50 ns is too much, and 100 ns is totally unacceptable. We don't want to convert our event handler into a stat logger. So the boundary between good and bad
is somewhere between 20 and 50 ns.

Another obvious requirement for the stats objects is that they must support a multi-threaded operation. Multiple client threads may exist, plus one server thread.

The tests
---------

We'll start with counters because they are a bit simpler than gauges; however, we'll keep gauges
in mind and, at the end of the article, run the gauge test as well.

We'll start several threads that will do their own work (perform some calculations) and update the counters at given intervals.
These intervals will vary between zero and several hundred nanoseconds, controlled by a parameter called the **delay factor**. We expect the counters to cause the biggest performance penalty at the smallest delays,
because this is where the conflicts are most probable (several threads updating the counter at the same time). That's why one of the tests will use the delay of zero,
even though this isn't very realistic.

Each thread will count how many counter update operations it performed -- this will be our performance metric.

Here is the thread body we'll be using:

{% highlight Java %}
public class CounterTest extends Thread
{
    private final Counter counter;
    private final int delayFactor;
    static volatile boolean running = true;
    long N;
    long time;
    double sum;
    
    public CounterTest (Counter counter, int delayFactor)
    {
        this.counter = counter;
        this.delayFactor = delayFactor;
    }
    
    @Override
    public void run ()
    {
        long N = 0;
        long t0 = System.currentTimeMillis ();
        double sum = 0;
        while (running) {
            for (int j = 1; j <= delayFactor; j++) {
                sum += 1.0/j;
            }
            counter.add ();
            ++ N;
        }
        long t1 = System.currentTimeMillis ();
        time = t1 - t0;
        this.N = N;
        this.sum = sum;
    }
}
{% endhighlight %}

The test spawns a given number of threads, all initialised with the same counter, then runs them for 10 sec, querying the counter every one second
(it is one minute in the real-life code, but we want our tests to run quickly). After the threads
stop, it collects the performance data as well as the number of times `counter.add()` was called. The average performance is then reported, in nanosecond per
counter operation:

{% highlight Java %}
public static void counterTest (Counter counter, int nthreads, int delayFactor)
                   throws InterruptedException
{
    ArrayList<CounterTest> threads = new ArrayList<Test> ();
    for (int i = 0; i < nthreads; i++) {
        threads.add (new CounterTest (counter, delayFactor));
    }
    running = true;
    for (CounterTest t : threads) {
        t.start ();
    }
    long sum = 0;
    for (int sec = 0; sec < 10; sec ++) {
        Thread.sleep (1000);
        long v = counter.getAndReset ();
        sum += v;
        System.out.printf ("Counter retrieved: %11d\n", v);
    }
    running = false;
    for (Thread t : threads) {
        t.join ();
    }
    long lastval = counter.getAndReset ();
    sum += lastval;
    System.out.printf ("Lastval retrieved: %11d\n", lastval);
    
    long testsum = 0;
    double nssum = 0;
    for (CounterTest test : threads) {
        double ns = test.time * 1.0E6 / test.N;
        System.out.printf ("Time = %d; N = %d; time/op: %6.2f ns\n",
                            test.time, test.N, ns);
        testsum += test.N;
        nssum += ns;
    }
    System.out.printf ("Time/op avg: %6.2f ns\n", nssum / nthreads);
    System.out.printf ("Counter sum: %12d\n", sum);
    System.out.printf ("Correct sum: %12d\n", testsum);
}
{% endhighlight %}

We'll be running the test using **Java 1.8** on a dual Xeon with 12 physical cores, with hyper-threading switched off.
The turbo-boost [must also be switched off]({{ site.ART-TURBO-BOOST }}).
We'll run it with `nthreads` equal to 1, 2, 6 and 12.
Each test will be repeated several times, two slowest results discarded as possibly performed on the VM that hasn't yet warmed up.
The average value of the rest will be reported as the result of the test.

The empty counter
-----------------

The first counter we test is an empty counter. It counts nothing:

{% highlight Java %}
public final class EmptyCounter extends ServerCounter
{
    @Override
    public void add (long increment)
    {
    }
    
    @Override
    public long getAndReset ()
    {
        return 0;
    }
}
{% endhighlight %}

This test shows base times, which are the times taken by the imaginary event handler without updating any counters.
Here are the times for some delay factors and thread counts:

<table class="numeric">
<tr><th rowspan="2">Counter name</th><th rowspan="2">Delay factor</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr><td class="label"> EmptyCounter </td><td>   0 </td><td>    1</td><td>    1</td><td>    1</td><td>    1</td></tr>
<tr><td class="label"> EmptyCounter </td><td>  10 </td><td>   47</td><td>   47</td><td>   47</td><td>   47</td></tr>
<tr><td class="label"> EmptyCounter </td><td>  50 </td><td>  374</td><td>  373</td><td>  374</td><td>  374</td></tr>
<tr><td class="label"> EmptyCounter </td><td> 100 </td><td>  887</td><td>  887</td><td>  883</td><td>  884</td></tr>
</table>

The measured times are very stable, and the performance does not depend on the number of threads, as long as they fit into physical cores.
As the delay factor changes, the base time varies in a wide range, resembling the realistic values common for real-world event processing.

The times are rounded to nanoseconds, as this accuracy is good enough for our problem.

The trivial counter
-------------------

The next one is the simplest possible implementation, so simple that everybody knows it is incorrect:

{% highlight Java %}
public final class TrivialCounter extends ServerCounter
{
    private long count = 0;
    
    @Override
    public void add (long increment)
    {
        count += increment;
    }
    
    @Override
    public long getAndReset ()
    {
        long result = count;
        count = 0;
        return result;
    }
}
{% endhighlight %}

Why is it incorrect? It isn't thread-safe, and even our simplest use case is multi-threaded (there is at least one client thread plus one server thread).
There are three issues that prevent it from running well in such case:

1) The `count` isn't `volatile`. This means that both the compiler and the processor are allowed to keep the thread-local view of this variable out of sync with
memory for a substantial time, or, if we're especially unlucky, permanently. When the variable is modified, the compiler may keep the new value in a register, and the only reason it writes it to memory at all is
that it must appear there at the next synchronisation point. Even when the variable is written to memory, the processor is allowed to keep it in a store buffer until
forced to flush it. Meanwhile, all the other cores will see the previous value of the variable. If multiple cores
modify the variable, the new values may be written to memory in any order. Likewise, when the variable is read, the compiler is allowed to
use the cached copy instead of fetching the externally modified value.

2) The `count` isn't protected from concurrent modification by two clients.

3) The `count` isn't protected from concurrent modification by a client and a server.

The most common way of resolving issues 2 and 3 makes issue 1 irrelevant: a variable that's read or modified in a `synchronized` block is kept in sync with memory automatically;
it does not have to be `volatile`.

In theory, lack of synchronisation may cause any result. A VM may optimise the program in an arbitrary way, which can literally cause any value to be produced. However,
given a typical code generation, we are most likely to observe two effects:

1) Loss of counts due to simultaneous update from two clients, or one client and one server, in a scenario similar to this:

<table>
<tr><th>Client thread 1</th><th>Client thread 2</th></tr>
<tr><td>tmp = count; </td><td>tmp = count;</td></tr>
<tr><td>++ tmp;      </td><td>++ tmp;     </td></tr>
<tr><td>             </td><td>count = tmp;</td></tr>
<tr><td>count = tmp; </td><td>            </td></tr>
</table>

This effect causes the counted value to be less than what it should be. More than  one increment of `count` (and in more than one thread) may happen while
the client thread 1 is holding on to its value, so the loss can be quite big.

2) Missing the reset of the counter:

<table>
<tr><th>Client thread</th><th>Server thread</th></tr>
<tr><td>tmp = count; </td><td>result = count;</td></tr>
<tr><td>++ tmp;      </td><td>count = 0;</td></tr>
<tr><td>count = tmp; </td><td>            </td></tr>
</table>

This may cause the entire interval of counter updates between two retrievals to be counted twice, making the counted value much bigger than it should be.

Do we really observe these effects, and, as they are working against each other, which of them wins? Let's run the test. I know, many people will say: "What's the point of running
an incorrect test? You can't use any of its results anyway". We'll try it just out of curiosity. Among other things, we are interested in its speed. Does the lack of
synchronisation make this test fast?

Running the tests, we see that the sum of retrieved values is bigger than the total number of times `Counter.add()` was invoked. For one thread:

    Counter sum:  24954272194
    Correct sum:   3827471278

If we look at the retrieve history, we see the increasing values:

    Counter retrieved:   357673732
    Counter retrieved:   706667851
    Counter retrieved:  1047647376
    Counter retrieved:  1387938007
    Counter retrieved:  1714498868

Very infrequently the value does drop:

    Counter retrieved:  2349630076
    Counter retrieved:  2512206688
    Counter retrieved:   220113172

This means that the second effect really occurs, and is very prominent. 

For six threads:

    Counter sum:    893999846
    Correct sum:    961599387

The second effect is still winning over the first one.
 
For twelve threads:

    Counter sum:    903239369
    Correct sum:    977822510

Finally, the first effect wins over the second one.

How fast does this test run? For convenience, we'll publish the times subtracting the values for the `EmptyCounter`:

<table class="numeric">
<tr><th rowspan="2">Counter name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr><td class="label"> TrivialCounter </td><td>   0 </td><td>   1</td><td> 2</td><td>  15</td><td>  62</td><td> 105</td></tr>
<tr><td class="label"> TrivialCounter </td><td>  10 </td><td>  47</td><td> 0</td><td>  46</td><td> 316</td><td> 693</td></tr>
<tr><td class="label"> TrivialCounter </td><td>  50 </td><td> 374</td><td> 1</td><td>  57</td><td> 254</td><td> 781</td></tr>
<tr><td class="label"> TrivialCounter </td><td> 100 </td><td> 887</td><td>-3</td><td>  15</td><td>  96</td><td> 569</td></tr>
</table>

We expected the trivial counter to be very fast, and it was -- but only when the thread count was one. For bigger thread counts the slowdown is quite big.
It shows that frequent update of the same memory location from different processor cores is inefficient. It prevent local caching. 
When one core updates the variable, the new value is stored in its cache and invalidated in all the other caches.
When other cores need the value, they must request it from the cache of the first core. This may still be faster than retrieving the value from the main memory,
but definitely slower than getting it from the local cache.

One special case is when the delay factor is zero. Then the introduced delay is not as big. I think it is because in this case the next update of the counter
happens while the previous one is still in the write buffer; but I may be wrong here.

Now we start worrying: if the most trivial counter can be so slow, what can we expect from a properly synchronised one?

The volatile counter
--------------------

One of the problems with the `TrivialCounter` was that its main value wasn't `volatile`. Let's try another one, called `VolatileCounter`:

{% highlight Java %}
public final class VolatileCounter extends ServerCounter
{
    private volatile long count = 0;
    
    @Override
    public void add (long increment)
    {
        count += increment;
    }
    
    @Override
    public long getAndReset ()
    {
        long result = count;
        count = 0;
        return result;
    }
}
{% endhighlight %}

Note that the `volatile` modifier does not provide atomic modification of the value. It only makes sure that any read of this variable is
translated into a proper memory read instruction, and any write is flushed from the processor's write buffer before executing the next **Java** statement.
As a result, the time between `tmp = count;` and `count = tmp;` becomes shorter and more predictable, but that does not prevent either of the effects
mentioned above, only reducing them a bit.

Here are the results for one thread, delay factor zero:

    Counter sum:   1357072743
    Correct sum:   1231492832

For six threads:

    Counter sum:    143726338
    Correct sum:    192685609

The first effect took over.

Even when the delay factor is big and the number of threads is small, the effects are visible. Here are two outputs with delay factor of 100 and one thread:

    Counter sum:     17381285
    Correct sum:     17381322

with the first effect and

    Counter sum:     20901042
    Correct sum:     17459768

with the second one.

Here are the times:

<table class="numeric">
<tr><th rowspan="2">Counter name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr><td class="label"> VolatileCounter </td><td>   0 </td><td>   1</td><td> 12</td><td> 138</td><td> 542</td><td>1556</td></tr>
<tr><td class="label"> VolatileCounter </td><td>  10 </td><td>  47</td><td>  1</td><td> 128</td><td> 651</td><td>1360</td></tr>
<tr><td class="label"> VolatileCounter </td><td>  50 </td><td> 374</td><td>  1</td><td>  50</td><td> 608</td><td>1643</td></tr>
<tr><td class="label"> VolatileCounter </td><td> 100 </td><td> 887</td><td> -2</td><td>  25</td><td> 149</td><td>1121</td></tr>
</table>

As we can see, the volatile variables really make things slower. We'll see why now.

Off-topic: the code with and without volatile
---------------------------------------------

It isn't really relevant for our tests, but at this point I feel curious: what exactly is the difference in the code generated for `TrivialCounter` and `VolatileCounter`?

Here is the code of the counter update from the non-volatile case:

{% highlight asm %}
  0x0000000002513572: mov    0x188(%rbx),%r9d   ;*getfield counter
  0x0000000002513579: test   %r9d,%r9d
  0x000000000251357c: je     0x00000000025135b5
  0x000000000251357e: incq   0x10(%r12,%r9,8)   ;*invokevirtual add
{% endhighlight %}

And this is the volatile case:

{% highlight asm %}
  0x00000000024431f0: mov    0x188(%rbp),%r10d  ;*getfield counter
  0x00000000024431f7: mov    0x10(%r12,%r10,8),%r8  ; implicit exception: dispatches to 0x0000000002443251
  0x00000000024431fc: add    $0x1,%r8
  0x0000000002443200: mov    %r8,0x10(%r12,%r10,8)
  0x0000000002443205: lock addl $0x0,(%rsp)     ;*putfield count
{% endhighlight %}

In both cases the `counter.add()` was inlined. The non-volatile code checked for `null` explicitly, while the volatile one made use of the hardware trap -- I don't know why.
The actual incrementing of a value is also done differently: `inc` in one case and load-`add`-store in another one. These differences, however, are not important.
What's important is the `addl` instruction with the `lock` prefix in the volatile case. The `lock` prefix here does not protect the `volatile` variable. In fact, it does not
protect anything -- the memory location the instruction operates on is thread-local (`%rsp`), and even that one isn't really updated (zero added). What this instruction actually
does is flush the processor write buffers.

The description of the `lock` prefix is a bit confusing. The corresponding part of the Intel manual begins very scary:

> Causes the processor's `LOCK#` signal to be asserted during execution of the accompanying
instruction (turns the instruction into an atomic instruction). In a multiprocessor
environment, the `LOCK#` signal ensures that the processor has exclusive use
of any shared memory while the signal is asserted

If this was indeed the case, any multi-core programming would have been a disaster. Any locked instruction would deny access to memory to all the other cores, and, since
there are twelve cores (and there are systems with many more), the chances of one of them issuing such an instruction are very high. The entire execution would stall.
Fortunately, later in the article comes a relief:

> Beginning with the P6 family processors, when the `LOCK` prefix is prefixed to an
instruction and the memory area being accessed is cached internally in the
processor, the `LOCK#` signal is generally not asserted. Instead, only the processor's
cache is locked. Here, the processor's cache coherency mechanism ensures that the
operation is carried out atomically with regards to memory

This is much better. The area being accessed (the top of the stack) is very likely to be cached locally, so nothing really stalls.
The cache coherency mechanism takes care of propagating any change in the cached data (on the cache line basis) to other caches.
It does not, however, affect the processor write buffers; that's why all of them must be flushed to the processor cache when any
locked instruction is performed. This is not limited to the memory area addressed by the locked instruction -- all of them are flushed:

> For the P6 family processors, locked operations serialize all outstanding load and
store operations (that is, wait for them to complete). This rule is also true for the
Pentium 4 and Intel Xeon processors

The JVM uses this to achieve memory consistency required for the `volatile` variables.

Other implementations (such as [JET](https://www.excelsiorjet.com/)) use the `MFENCE` instruction for the same purpose.

If we took a non-volatile version of the code and added the `LOCK` prefix to the `incq` instruction, we would have produced a properly synchronised counter.
Unfortunately, we can't force JVM to do it for us.


The simple counter
------------------

Now we move to the simplest counter that actually works. All it takes is to make the relevant methods `synchronized`. In this case the `volatile` modifier is no longer
necessary. All the variables will be saved to memory at the end of the `synchronized` block:

{% highlight Java %}
public final class SimpleCounter extends ServerCounter
{
    private long count = 0;
    
    @Override
    public synchronized void add (long increment)
    {
        count += increment;
    }
    
    @Override
    public synchronized long getAndReset ()
    {
        long result = count;
        count = 0;
        return result;
    }
}
{% endhighlight %}

The results are now correct. Here is the output for 12 threads, delay factor zero:

    Counter sum:     40933218
    Correct sum:     40933218

Here are the times:

<table class="numeric">
<tr><th rowspan="2">Counter name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr><td class="label"> SimpleCounter </td><td>   0 </td><td>1</td><td>  39</td><td> 843</td><td>1578</td><td>5565</td></tr>
<tr><td class="label"> SimpleCounter </td><td>  10 </td><td>  47</td><td>  70</td><td> 724</td><td>2000</td><td>7008</td></tr>
<tr><td class="label"> SimpleCounter </td><td>  50 </td><td> 374</td><td>  25</td><td> 365</td><td>2512</td><td>7981</td></tr>
<tr><td class="label"> SimpleCounter </td><td> 100 </td><td> 887</td><td>  20</td><td> 332</td><td>2344</td><td>8550</td></tr>
</table>

The times are awful. When the number of threads is one, the resuls are not so bad, but not great, either. Any higher number of threads ruins the picture completely.
What is remarkable is that the synchronisation overhead is high even when delay factor in big. The chances of monitor conflicts are lower in this case, but the
synchronisation overhead is still high. In short, this counter is totally unusable.

It is really amazing how slow the `synchronized` blocks really are. The time of 8550 ns corresponds to only 117,000 operations per seconds -- just for one counter update.

The atomic counter
------------------

In the `SimpleCounter` implementation we don't perform a lot of operations inside the `synchronized` block. It is either increment of a value or swap it with zero.
There is more lightweight way to do it than the `synchronized` block: using atomic operations. A lot of articles have been written on lock-free multithreaded programming in
**Java**. What's meant by "lock-free" is usually "atomic" (sometimes the desired result can be achieved by just using `volatile` variables, but not in our case). Let's try it
and check if it really is faster:


{% highlight Java %}
import java.util.concurrent.atomic.AtomicLong;

public final class AtomicCounter extends Counter
{
    private AtomicLong count = new AtomicLong (0);
    
    @Override
    public void add (long increment)
    {
        count.addAndGet (increment);
    }
    
    @Override
    public long getAndReset ()
    {
        return count.getAndSet (0);
    }
}
{% endhighlight %}

This approach is only applicable for counters, not for gauges. Gauges require several operations to be performed atomically (calculating the sum, the maximum and the minimum,
and updating the counter).

The reported values are still correct:

    Counter sum:    155891214
    Correct sum:    155891214

Here are the times:

<table class="numeric">
<tr><th rowspan="2">Counter name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr><td class="label"> AtomicCounter </td><td>   0 </td><td>   1</td><td> 11</td><td> 135</td><td> 498</td><td> 926</td></tr>
<tr><td class="label"> AtomicCounter </td><td>  10 </td><td>  47</td><td>  1</td><td> 102</td><td> 440</td><td>1162</td></tr>
<tr><td class="label"> AtomicCounter </td><td>  50 </td><td> 374</td><td>  3</td><td>  72</td><td> 581</td><td>1559</td></tr>
<tr><td class="label"> AtomicCounter </td><td> 100 </td><td> 887</td><td> -1</td><td>  27</td><td> 170</td><td> 968</td></tr>
</table>

The times look much better than for the `synchronized` counter. We've got really wonderful values for one thread, almost reasonable values for two threads and high delays.
Unfortunately, the values for higher thread counts are still prohibitively bad (although not as ridiculous as for the `synchronized` counter).

Off-topic: how it works
-----------------------

What is the code generated for the atomic counter? Let's look at the appropriate portion of the `run()` method (everything was inlined there):

{% highlight asm %}
0x000000000227e5a4: test   %eax,-0x216e5aa(%rip)        # 0x0000000000110000
                                              ; - java.util.concurrent.atomic.AtomicLong::addAndGet@23 (line 251)
0x000000000227e5aa: mov    0x10(%r12,%rdi,8),%rax
0x000000000227e5af: mov    %rax,%r10
0x000000000227e5b2: add    $0x1,%r10
0x000000000227e5b6: lock cmpxchg %r10,0x10(%r12,%rdi,8)
0x000000000227e5bd: sete   %r10b
0x000000000227e5c1: movzbl %r10b,%r10d        ;*invokevirtual compareAndSwapLong
0x000000000227e5c5: test   %r10d,%r10d
0x000000000227e5c8: je     0x000000000227e5a4  ;*lload
{% endhighlight %}

Three instructions (`sete`, `movzbl` and `test`) are in fact unnecessary. I'm not sure what the first `test` instruction is for (I suspect this is the way to interrupt
the loop: for that the VM must unmap the page this instruction points to (`0x110000`)). Apart from this, the code is very simple: we get the value, add one and write it back
on condition that the memory location still contains the value we found initially. If not, we repeat the procedure. The compare and swap are done atomically using the instruction kindly
provided by Intel: `cmpxchg`. It has a `lock` prefix, with usual consequences discussed above. The counter value is likely to be cached, so the `lock` prefix does not have
to lock anything but the local cache. It will, however, serialise the reads and writes, which has its price.

The code above directly corresponds to the **Java** code in the `AtomicLong` class:

{% highlight Java %}
    public final long addAndGet(long delta) {
        for (;;) {
            long current = get();
            long next = current + delta;
            if (compareAndSet(current, next))
                return next;
        }
    }
{% endhighlight %}

This loop contains no provision for fairness. One thread can get stuck there forever. I made a test to verify how many times this loop usually runs. For that, I made my
own version of `AtomicLong` (using `sun.misc.Unsafe`), which counted the iterations, stored the counts and printed them eventually. I won't publish the code here,
for it is straightforward. The test showed that almost every time the loop count is one, very seldom two. I never saw any bigger values even on delay factor of zero and
twelve threads.

This means that the cost of updating the atomic counter isn't much higher than that of the `VolatileCounter`. The execution times show the atomic counter is actually faster.

Compound counters
-----------------

We've looked at two bad solutions, which don't produce correct results, and two good ones, which do. Neither of them was really fast enough on high thread counts. Even
the ones that lacked any synchronisation were slow. We need some other approach to implement
counters suitable for highly-parallel applications. We'll try thread-local counters.

The idea is that each thread, when started, requests thread-local views of all the counters it is going to use. The base counter keeps track of all such views and,
when requested a value, asks them all for their local values and adds all of those together.

Let's first add a method to the `Counter` base class:

{% highlight Java %}
public Counter getThreadLocalView ()
{
    return this;
}
{% endhighlight %}

Then we'll introduce the `CompoundCounter` class, which will be parameterised with the class of the thread-local counter:

{% highlight Java %}
public final class CompoundCounter extends ServerCounter
{
    private ArrayList<ServerCounter> views = new ArrayList<ServerCounter> ();
    private Class<? extends ServerCounter> clazz;
    private long localValue = 0;
    
    public CompoundCounter (Class<? extends ServerCounter> clazz)
    {
        this.clazz = clazz;
    }
    
    @Override
    public synchronized Counter getThreadLocalView ()
    {
        try {
            ServerCounter view = clazz.newInstance ();
            views.add (view);
            return view;
        } catch (Exception e) {
            throw new RuntimeException (e);
        }
    }

    @Override
    public synchronized void add (long increment)
    {
        localValue += increment;
    }

    @Override
    public synchronized long getAndReset ()
    {
        long sum = localValue;
        localValue = 0;
        for (ServerCounter view : views) {
            sum += view.getAndReset ();
        }
        return sum;
    }
}
{% endhighlight %}

This is a simplistic implementation. The proper one must take care of threads starting and stopping. The counters created by the stopped threads must be removed from the list --
either by means of a finalizer, or by using a `WeakReference`, We'll skip this complication for now.

The compound counter has a properly synchronised local value as well, so it can be used directly, too.

We need to modify the `counterTest` method, where the threads are created:

{% highlight Java %}
for (int i = 0; i < nthreads; i++) {
    threads.add (new Test (counter.getThreadLocalView (), delayFactor));
}
{% endhighlight %}

Let's try this `CompoundCounter` with all four of the counter types created before:

<table class="numeric">
<tr><th rowspan="2">Counter name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>

<tr class="even"><td class="label"> Compound&nbsp;Trivial </td><td>   0 </td><td>   1</td><td> 3</td><td> 3</td><td> 3</td><td> 3</td></tr>
<tr class="even"><td class="label"> Compound&nbsp;Trivial </td><td>  10 </td><td>  47</td><td> 1</td><td> 0</td><td> 0</td><td> 0</td></tr>
<tr class="even"><td class="label"> Compound&nbsp;Trivial </td><td>  50 </td><td> 374</td><td> 1</td><td> 2</td><td> 1</td><td> 1</td></tr>
<tr class="even"><td class="label"> Compound&nbsp;Trivial </td><td> 100 </td><td> 887</td><td> 3</td><td>-3</td><td>-2</td><td>-2</td></tr>

<tr><td class="label"> Compound&nbsp;Volatile </td><td>   0 </td><td>   1</td><td> 11</td><td> 11</td><td> 11</td><td> 11</td></tr>
<tr><td class="label"> Compound&nbsp;Volatile </td><td>  10 </td><td>  47</td><td>  1</td><td>  1</td><td>  1</td><td>  1</td></tr>
<tr><td class="label"> Compound&nbsp;Volatile </td><td>  50 </td><td> 374</td><td>  2</td><td>  2</td><td>  4</td><td>  4</td></tr>
<tr><td class="label"> Compound&nbsp;Volatile </td><td> 100 </td><td> 887</td><td> -1</td><td> -3</td><td>  1</td><td>  1</td></tr>

<tr class="even"><td class="label"> Compound&nbsp;Simple </td><td>   0 </td><td>1</td><td>  67</td><td>  73</td><td>  72</td><td>  85</td></tr>
<tr class="even"><td class="label"> Compound&nbsp;Simple </td><td>  10 </td><td>  47</td><td>  56</td><td>  53</td><td>  55</td><td>  57</td></tr>
<tr class="even"><td class="label"> Compound&nbsp;Simple </td><td>  50 </td><td> 374</td><td>  24</td><td>  46</td><td>  37</td><td>  51</td></tr>
<tr class="even"><td class="label"> Compound&nbsp;Simple </td><td> 100 </td><td> 887</td><td>  17</td><td>  24</td><td>  23</td><td>  26</td></tr>

<tr><td class="label"> Compound&nbsp;Atomic </td><td>   0 </td><td>   1</td><td> 11</td><td> 11</td><td> 11</td><td> 11</td></tr>
<tr><td class="label"> Compound&nbsp;Atomic </td><td>  10 </td><td>  47</td><td>  1</td><td>  1</td><td>  1</td><td>  1</td></tr>
<tr><td class="label"> Compound&nbsp;Atomic </td><td>  50 </td><td> 374</td><td>  5</td><td>  6</td><td>  4</td><td>  4</td></tr>
<tr><td class="label"> Compound&nbsp;Atomic </td><td> 100 </td><td> 887</td><td>  -1</td><td>-6</td><td>  0</td><td>  1</td></tr>
</table>

This looks much better. All the results improved a lot, including those that didn't use any synchronisation. This is the result of not sharing of variables
between threads. The synchronised and atomic counters benefit also from absence of thread synchronisation collisions.

The performance of the compound atomic counter is already within the target limits that we established in the beginning (10 ns). Unfortunately, this solution
is only applicable for counters, not for gauges (or, rather, I didn't manage to come up with a suitable solution for gauges using atomics; it may still exist).

The performance of the synchronised counter (`SimpleCounter`) is at the boundary. We probably can afford one synchronised update operation per event handler
at these times, but not more. The times for gauges are likely to be similar, which means that we don't have any suitable solution for gauges yet, and must
try something else. Perhaps, we can improve the counters as well?


Indirect counters
-----------------

Let's try a different approach. Instead of storing the value inside the counter, we'll store it in its own object and swap the reference. We'll call this version
"**an indirect counter**".

We'll see later that other code may benefit from getting access to that separate object in a generic way, that's why we'll create several abstract classes before
introducing the real solution.

First, we need an interface to the object that stores the value:

{% highlight Java %}
public abstract class CounterValue
{
    public abstract long get ();
}
{% endhighlight %}

Then the base class of all the indirect counters:

{% highlight Java %}
public abstract class IndirectCounter extends ServerCounter
{
    public abstract CounterValue getAndResetValue ();
    
    @Override
    public long getAndReset ()
    {
        return getAndResetValue ().get ();
    }
}
{% endhighlight %}

Finally, the actual implementation, which we'll call `LazyCounter`, because it allows delayed fetching of its value:

{% highlight Java %}
public final class MutableCounterValue extends CounterValue
{
    private long value = 0;
    
    public void add (long increment)
    {
        value += increment;
    }

    @Override
    public long get ()
    {
        return value;
    }
}

public final class LazyCounter extends IndirectCounter
{
    private volatile MutableCounterValue current = new MutableCounterValue ();
    
    @Override
    public void add (long increment)
    {
        current.add (increment);
    }

    @Override
    public CounterValue getAndResetValue ()
    {
        CounterValue result = current;
        current = new MutableCounterValue ();
        return result;
    }
}
{% endhighlight %}


The reference to the object is stored in a `volatile` field, which prohibits keeping it in a register: it must be read from memory each time it's used, even if the same counter
is updated several times in a row. This adds one extra memory access per counter update compared to non-indirect counters. However, as we saw, unlike writing, reading
of `volatile` does not incur any additional costs compared to ordinary fields. This one is likely to be cached, so the cost is low. The benefit is that the client always
sees the latest value of this reference, and, when it is swapped, starts updating the new one, while the server is still holding on to the old value.

The nice part of this strategy is that the referenced value may contain more than
one field that must be updated in one transaction. We'll see it in our implementation of the `Gauge` later.

There is no synchronisation anywhere, so the class is still not suitable for simultaneous multi-threaded update. In a multi-client environment it can only be used as a thread-local
view of a compound counter. It does, however, improve over the `TrivialCounter` and `VolatileCounter`: it completely eliminates the second effect
(missing the reset of the counter). At some point after the swap (and rather quickly, due to the `volatile` reference), the counter will start updating the new value. This means that
our results can't be totally out any more.

Does it mean that this is a completely correct solution? We know it can't be. No multi-threaded **Java** program can be completely correct without using some
synchronisation primitives (although, as we'll see later, some degree of cheating may help incorrect program produce sufficiently correct results).
There must be something wrong with this version, and there is.
There are at least two problems that are not resolved:

 1) When the server thread replaces the value object, there is a short interval of time when both the client and the server threads have access to it.
    The client may change the value after is has been collected by the server, in which case that update will be lost. In the case of a gauge, where more than one
    value is kept, there is also a possibility that the server uses a half-updated gauge value.

 2) Even if the client has updated the reference and isn't busy using the old one, the processor may still do it: the last value written into the counter variable may
    still be in a processor write buffer. We don't know how long this may last. Probably, not that long (the primary reason why a value may get stuck in that buffer is
    a cache miss, and our value has just been read and is therefore cached).

Since the `LazyCounter` can't be used on its own with more than one client thread, we'll run our tests with a `CompoundCounter` based on it.
We see that the problem does indeed occur. With the delay factor of zero, we see many counter errors even on one thread:

    Counter sum:   3860183598
    Correct sum:   3860183606

The error, obviously, only gets bigger with more threads. On 12 threads we get:

    Counter sum:  34326496626
    Correct sum:  34326497846

The error gets smaller with the delay factor growing, but it never becomes zero. Even for the delay of 100 and one thread, we can still see errors:

    Counter sum:     16797154
    Correct sum:     16797155

If the error was always one or two counts, we could perhaps ignore it (as I said in the beginning, the counters aren't mission-critical). However, we sometimes observe
much bigger differences. We'll first try to fix those, and only then look at the times.

Indirect counter: lazy retrieval
--------------------------------

As said before, the reason for the counter difference is that the last update by the client thread
may coincide with the retrieval by the server thread. The time window for that is very small, though -- most likely, not more than 10 CPU cycles. Here comes a solution:
the server must delay using the value until quite sure it is completely stable and isn't going to change any more -- we'll call it the **cool-down period**.
A couple of hundred of nanoseconds should be enough. Let's, however, be completely paranoid and make this period half the collection interval (30 seconds in the real program, half a second in our tests).
The server thread will run twice as often as before and collect the values on the even iterations, while using them on odd ones. This is where our `IndirectCounter` interface
becomes handy. We'll change our test loop in the following way:

{% highlight Java %}
    long sum = 0;
    for (int sec = 0; sec < 10; sec ++) {
        Thread.sleep (500);
        CounterValue val = dc.getAndResetValue ();
        Thread.sleep (500);
        long v = val.get ();
        System.out.printf ("Counter retrieved: %11d\n", v);
        sum += v;
    }
    running = false;
    for (Thread t : threads) {
        t.join ();
    }
    CounterValue lastvalue = dc.getAndResetValue ();
    long lastval = lastvalue.get ();
    sum += lastval;
    System.out.printf ("Lastval retrieved: %11d\n", lastval);
{% endhighlight %}

We call it a "lazy retrieval", because the object being retrieved isn't used immediately, but rather stored for future use.

The tests of `LazyCounter` with one thread look promising: no updates are lost. However, the higher thread counts are still not covered: the
`LazyCounter` doesn't support multiple clients, and `CompoundCounter` does not support delayed operation. We need one last step.

Lazy compound indirect counters
-------------------------------

This counter will resemble the `CompoundCounter` that we saw before, with one important difference: it won't touch the values until really forced to do so.
In the meantime it will keep the value objects in an array. This is really what is called "lazy operation" --
very common trick in functional programming, when the expression is kept in its symbolic form until the value is really needed.

Here are the classes:

{% highlight Java %}
public final class CompoundCounterValue extends CounterValue
{
    private final CounterValue [] values;
    private final long ownValue;
    
    public CompoundCounterValue (CounterValue [] values, long ownValue)
    {
        this.values = values;
        this.ownValue = ownValue;
    }

    @Override
    public long get ()
    {
        long sum = ownValue;
        for (CounterValue v : values) sum += v.get();
        return sum;
    }
}

public class LazyCompoundCounter extends IndirectCounter
{
    private List<IndirectCounter> views = new ArrayList<IndirectCounter> ();
    private Class<? extends IndirectCounter> clazz;
    private long localValue = 0;

    public LazyCompoundCounter (Class<? extends IndirectCounter> clazz)
    {
        this.clazz = clazz;
    }
    
    @Override
    public synchronized Counter getThreadLocalView ()
    {
        try {
            IndirectCounter view = clazz.newInstance ();
            views.add (view);
            return view;
        } catch (Exception e) {
            throw new RuntimeException (e);
        }
    }

    @Override
    public synchronized void add (long increment)
    {
        localValue += increment;
    }

    @Override
    public synchronized CounterValue getAndResetValue ()
    {
        CounterValue [] values = new CounterValue [views.size ()];
        for (int i = 0; i < values.length; i++) {
            values [i] = views.get (i).getAndResetValue ();
        }
        long ownValue = localValue;
        localValue = 0;
        return new CompoundCounterValue (values, ownValue);
    }
}

{% endhighlight %}

Let's run it giving it `LazyCounter.class` as a parameter. We don't see any counter losses. This is the result for delay factor zero, 12 threads:

    Counter sum:  34882489418
    Correct sum:  34882489418

Although this solution is, as we admitted, not completely corresponding to the **Java** spec, it is still practically usable. And it is, we hope, fast. Let's look at the times:

<table class="numeric">
<tr><th rowspan="2">Counter name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr><td class="label"> Compound Lazy </td><td>   0 </td><td>   1</td><td> 2</td><td>  21</td><td>   2</td><td>   3</td></tr>
<tr><td class="label"> Compound Lazy </td><td>  10 </td><td>  47</td><td> 1</td><td>  13</td><td>  48</td><td>  60</td></tr>
<tr><td class="label"> Compound Lazy </td><td>  50 </td><td> 374</td><td>-2</td><td>  21</td><td>  98</td><td> 122</td></tr>
<tr><td class="label"> Compound Lazy </td><td> 100 </td><td> 887</td><td>-8</td><td>  -3</td><td>  20</td><td>  41</td></tr>
</table>

Here comes the disappointment: the solution is in fact not fast. At least, not faster than the compound counter based on the atomic counter. What went wrong?

The false sharing problem
-------------------------

We've mentioned simultaneous update of a variable by multiple threads as a source of a big inefficiency. We saw how it slowed down all our counters before we introduced
the compound one.

The problem, however, is much bigger. The caches operate on the cache line basis (64 bytes on our architecture). When a variable is updated, not only its value is
invalidated in other caches, but also values of all the variables that share the same cache line. This is called ["False sharing"](https://en.wikipedia.org/wiki/False_sharing).
Can it be the reason of the observed performance degradation?

The counter value objects are allocated by the server thread in sequence, one immediately after another. They can easily end up in the same cache lines. Let's check if
this happens by printing the addresses of allocated objects. This is how we do it:

{% highlight Java %}
import java.lang.reflect.Constructor;
import sun.misc.Unsafe;

public class Dump
{
    static Dump dump = new Dump ();
    static Unsafe unsafe;
    static int offset;
    public CounterValue v;
    static int addresses [] = new int [100];
    static int cnt = 0;
    
    static {
          try {
              Constructor<Unsafe> unsafeConstructor
                = Unsafe.class.getDeclaredConstructor();
              unsafeConstructor.setAccessible(true);
              unsafe = unsafeConstructor.newInstance();
              offset = unsafe.fieldOffset (Dump.class.getField ("v"));
          } catch (Exception e) {
              e.printStackTrace ();
          }
    }
    
    public static void addValue (CounterValue value)
    {
        dump.v = value;
        addresses [cnt++] = unsafe.getInt (dump, offset);
        dump.v = null;
        if (cnt == addresses.length) {
            System.out.println (addresses [0] + ": ");
            for (int i = 1; i < addresses.length; i++)
                System.out.print (addresses [i] - addresses [i-1] + " ");
            System.out.println ();
            System.exit (0);
        }
    }
}
{% endhighlight %}

This class uses `sun.misc.Unsafe` to get hold of the objects` addresses, collects these addresses into an array and prints the contents of that array when it has enough.
To make its output easier to read, it prints the first address and then the differences.

The Eclipse refused to compile this class, at least at its standard error settings. Fortunately, the **javac** compiles it, just gives four warnings.

The `LazyCounter` must call the `addValue` after allocating a new object:

{% highlight Java %}
    public CounterValue getAndResetValue ()
    {
        CounterValue result = current;
        current = new MutableCounterValue ();
        Dump.addValue (current);
        return result;
    }
{% endhighlight %}

This is the output we get after several seconds of running with 12 threads:

    -177946356:
    987 3 3 3 3 3 3 3 3 3 3 17881 3 3 3 3 3 3 3 3 3 3 3 175 3 3 3 3 3 3 3 3 3 3 3
    175 3 3 3 3 3 3 3 3 3 3 3 175 3 3 3 3 3 3 3 3 3 3 3 175 3 3 3 3 3 3 3 3 3 3 3
    175 3 3 3 3 3 3 3 3 3 3 3 175 3 3 3 3 3 3 3 3 3 3 3 175 3 3 3

The address difference of 3 looks confusing, but is explained easily. By default, JVM uses compressed pointers. These pointers occupy four bytes (that's why we read the `v` variable
using `unsafe.getInt()`, not `getLong()`) and serve as indices. They are multiplied by 8 and added to a base address. This way a 32-bit address space can address
up to 32Gb heap. What we see in the output is that the objects are allocated 24 bytes apart from each other. Every 12<sup>th</sup> difference is bigger (175, or 1400 bytes),
because the server allocates some other objects after collecting the values from all the thread local views.

If we prefer uncompressed pointers, we can replace `int` with `long` in the `Dump` class and run the test with pointer compression off:

    # java -XX:-UseCompressedOops Test
    8288454184:
    9656 24 24 24 24 24 24 24 24 24 24 178712 24 24 24 24 24 24 24 24 24 24 24
    1832 24 24 24 24 24 24 24 24 24 24 24 1832 24 24 24 24 24 24 24 24 24 24 24
    1832 24 24 24 24 24 24 24 24 24 24 24 1832 24 24 24 24 24 24 24 24 24 24 24
    1832 24 24 24 24 24 24 24 24 24 24 24 1832 24 24 24 24 24 24 24 24 24 24 24
    1832 24 24 24


We've learned that an object with a single `long` field takes up 24 bytes in memory, and that the objects are allocated next to each other. This means that two of them will
always share the same cache line, sometimes three. The false sharing is really happening here.

One can think of many ways to improve the situation. For instance, we can pre-allocate a lot of the value objects and give them out in an order that's different from the order
they were allocated. We'll look at some of these options later. For now, since we are not writing production code, let's follow the easiest route and make the object bigger -- pad
it to 64 bytes:

{% highlight Java %}
public final class MutableCounterValue extends CounterValue
{
    private long value = 0;
    public long v1 = 0;
    public long v2 = 0;
    public long v3 = 0;
    public long v4 = 0;
    public long v5 = 0;

    public void add (long increment)
    {
        value += increment;
    }

    @Override
    public long get ()
    {
        return value;
    }
}
{% endhighlight %}

The address dump now reports the difference of 64:

    8300513448:
    9696 64 64 64 64 64 64 64 64 64 64 178752 64 64 64 64 64 64 64 64 64 64 64
    1872 64

Why didn't we suffer from false sharing in our `CompoundCounter` test? We just got lucky. The thread local views were allocated in a loop:

{% highlight Java %}
    for (int i = 0; i < nthreads; i++) {
        threads.add (new Test (counter.getThreadLocalView (), delayFactor));
    }
{% endhighlight %}

Other objects were allocated between calls to `getThreadLocalView()` -- the `Test` object and whatever the class `Thread` allocated in its constructor.
Measurements using similar tool to the `Dump` class show that the distance between objects was 520 bytes. That's why it was so fast.

Now it's time to check the speed:

<table class="numeric">
<tr><th rowspan="2">Counter name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr><td class="label"> Compound Lazy </td><td>   0 </td><td>   1</td><td> 4</td><td> 4</td><td> 4</td><td> 4</td></tr>
<tr><td class="label"> Compound Lazy </td><td>  10 </td><td>  47</td><td> 0</td><td> 0</td><td> 0</td><td> 0</td></tr>
<tr><td class="label"> Compound Lazy </td><td>  50 </td><td> 374</td><td>-1</td><td> 1</td><td> 2</td><td> 3</td></tr>
<tr><td class="label"> Compound Lazy </td><td> 100 </td><td> 887</td><td>-6</td><td>-3</td><td> 1</td><td> 1</td></tr>
</table>

This is an excellent result. All the times are well below the target value of 10 ns, and generally better than the compound solution based on the atomic counter.
We finally found a solution that really works, and is incredibly fast -- all at the cost of not being completely legal.

A thread-independent solution
-----------------------------

The solution we've just presented is very good, but it has one obvious deficiency. It depends on the threads explicitly obtaining the thread-local views of the compound
counter. This requires extra effort and may be error-prone (one thread may accidentally use a counter from another one; there isn't any syntax checking for that).
Sometimes thread creation can be outside of our control. For instance, a third-party HTTP server may call our code from whichever thread it finds suitable. There are also
general-purpose libraries and other utility code that can be called from anywhere. Can such a code still benefit from the thread-local solution?

For the cases like that we'll create a special adapter that is built on top of the `LazyCompoundCounter` and is using the **Java**'s `ThreadLocal` utility class.
We'll call it simply a `FastCounter`:

{% highlight Java %}
public class FastCounter extends LazyCompoundCounter
{
    private ThreadLocal<Counter> counters = new ThreadLocal<Counter> ();

    public FastCounter ()
    {
        super (LazyCounter.class);
    }
    
    @Override
    public Counter getThreadLocalView ()
    {
        return this;
    }
    
    @Override
    public void add (long increment)
    {
        Counter counter = counters.get ();
        if (counter == null) {
            counter = super.getThreadLocalView ();
            counters.set (counter);
        }
        counter.add (increment);
    }
}
{% endhighlight %}

The thread local views are stored in the `ThreadLocal` utility collection, which is implemented as a hash map (a different kind from the standard `HashMap` -- it uses
re-hashing rather than entry chains) associated with the current `Thread` object. The operations on this collection are completely unsynchronised, and don't cost too much.
On every `add()` operation this class checks if there is a thread-local view associated with the current thread, and requests one if not. It adds additional operations,
but does not require any locking.

The example class above re-defines `getThreadLocalView()` as a trivial function, just because our test code calls it, and we want this code to behave as if there are
no thread-local views. In real-life code this function will be implemented properly, so that the clients that know their threads can request a thread-local view and save
those hash map lookups.

Here are the times:

<table class="numeric">
<tr><th rowspan="2">Counter name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr><td class="label"> FastCounter </td><td>   0 </td><td>   1</td><td> 7</td><td> 7</td><td> 7</td><td> 7</td></tr>
<tr><td class="label"> FastCounter </td><td>  10 </td><td>  47</td><td> 1</td><td> 1</td><td> 1</td><td> 1</td></tr>
<tr><td class="label"> FastCounter </td><td>  50 </td><td> 374</td><td>-1</td><td>-1</td><td> 3</td><td> 3</td></tr>
<tr><td class="label"> FastCounter </td><td> 100 </td><td> 887</td><td>-6</td><td>-7</td><td>-3</td><td>-1</td></tr>
</table>

This also looks very good. It runs slower that the compound counter with explicit thread-local views only on the delay factors 0 and 10, and even then it is fast enough.

The fast atomic counter
-----------------------

As we remember, the atomic counters also performed well, but only in the thread-local setup. Let's implement a fast atomic counter, based on the usual
(non-lazy) compound counter and the `ThreadLocal` map:

{% highlight Java %}
public class FastAtomicCounter extends CompoundCounter
{
    private ThreadLocal<Counter> counters = new ThreadLocal<Counter> ();
    
    public FastAtomicCounter ()
    {
        super (AtomicCounter.class);
    }

    @Override
    public synchronized Counter getThreadLocalView ()
    {
        return this;
    }

    @Override
    public void add (long increment)
    {
        Counter counter = counters.get ();
        if (counter == null) {
            counter = super.getThreadLocalView ();
            counters.set (counter);
        }
        counter.add (increment);
    }
}
{% endhighlight %}

The results are a bit worse than those of `FastCounter`, but the solution is still usable:

<table class="numeric">
<tr><th rowspan="2">Counter name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr><td class="label"> FastAtomicCounter </td><td>   0 </td><td>   1</td><td>  15</td><td>  15</td><td>  15</td><td>  15</td></tr>
<tr><td class="label"> FastAtomicCounter </td><td>  10 </td><td>  47</td><td>   6</td><td>   7</td><td>  7</td><td>    5</td></tr>
<tr><td class="label"> FastAtomicCounter </td><td>  50 </td><td> 374</td><td>  16</td><td>  27</td><td>  21</td><td>  21</td></tr>
<tr><td class="label"> FastAtomicCounter </td><td> 100 </td><td> 887</td><td>  20</td><td>  24</td><td>  27</td><td>  26</td></tr>
</table>

The positive feature of this solution is that it will satisfy any **Java** purist: it is completely correct and properly synchronised.


Gauges
------

Now it's time to look at the gauges. This time we won't implement obviously incorrect solutions. We also won't need a special `IndirectGauge`
interface, because gauge's `getAndReset()` method already returns an object rather than a primitive value. There isn't any `Atomic` solution. The classes
implemented for gauges include:

- `EmptyGauge`: does nothing

- `SimpleGaugeValue`: an immutable implementation of `GaugeValue`. It is returned by ordinary (non-lazy) implementations

- `SimpleGauge`: the most obvious `synchronized` solution

- `CompoundGauge`: a non-lazy compound solution based on an arbitrary gauge implementation and the idea of a thread-local view

- `LazyGauge`: a gauge that keeps its value inside a `MutableGaugeValue` referenced by a `volatile` reference

- `LazyCompoundGauge`: a lazy compound solution, which collects the values from its thread-local views into a collection called `CompoundGaugeValue`, so that they could
  be added together later

- `FastGauge`: a thread-independent facade of the `LazyCompoundGauge`, based on **Java**'s `ThreadLocal`.

The implementations are straightforward and look very similar to those of the counters. I won't show them all here; they are available in the repository. I'll only
show the core of the lazy compound solution -- the thread local views:

{% highlight Java %}
public class MutableGaugeValue extends GaugeValue
{
    private long count = 0;
    private long sum = 0;
    private long max = Long.MIN_VALUE;
    private long min = Long.MAX_VALUE;
 
    @Override
    public long getCount () { return count;}
    
    @Override
    public long getSum () { return sum;}
    
    @Override
    public long getMax () { return max; }
    
    @Override
    public long getMin () { return min; }
    }
    
    public void report (long value)
    {
        sum += value;
        max = Math.max (max, value);
        min = Math.min (min, value);
        ++ count;
    }
}

public class LazyGauge extends ServerGauge
{
    private volatile MutableGaugeValue current = new MutableGaugeValue ();

    @Override
    public void report (long value)
    {
        current.report (value);
    }

    @Override
    public GaugeValue getAndReset ()
    {
        GaugeValue result = current;
        current = new MutableGaugeValue ();
        return result;
    }
}
{% endhighlight %}

All the versions produced correct results -- the counts and the sums were exactly the same as expected. Here are the times:

<table class="numeric">
<tr><th rowspan="2">Gauge name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr class="even"><td class="label"> Simple </td><td>   0 </td><td>   2</td><td>  14</td><td> 218</td><td> 704</td><td>1884</td></tr>
<tr class="even"><td class="label"> Simple </td><td>  10 </td><td>  47</td><td>  16</td><td> 400</td><td>1067</td><td>3706</td></tr> 
<tr class="even"><td class="label"> Simple </td><td>  50 </td><td> 394</td><td>  35</td><td> 107</td><td>1884</td><td>5866</td></tr>
<tr class="even"><td class="label"> Simple </td><td> 100 </td><td> 898</td><td>  40</td><td>  83</td><td>1574</td><td>6047</td></tr>
<tr><td class="label"> Compound Simple </td><td>   0 </td><td>   2</td><td> 14</td><td> 15</td><td> 16</td><td> 19</td></tr>
<tr><td class="label"> Compound Simple </td><td>  10 </td><td>  47</td><td> 17</td><td> 17</td><td> 17</td><td> 17</td></tr>
<tr><td class="label"> Compound Simple </td><td>  50 </td><td> 394</td><td> 42</td><td> 41</td><td> 43</td><td> 42</td></tr>
<tr><td class="label"> Compound Simple </td><td> 100 </td><td> 898</td><td> 49</td><td> 49</td><td> 49</td><td> 52</td></tr>
<tr class="even"><td class="label"> Compound Lazy </td><td>   0 </td><td>   2</td><td>  8</td><td> 18</td><td> 39</td><td> 44</td></tr>
<tr class="even"><td class="label"> Compound Lazy </td><td>  10 </td><td>  47</td><td>  5</td><td> 30</td><td> 90</td><td> 86</td></tr>
<tr class="even"><td class="label"> Compound Lazy </td><td>  50 </td><td> 394</td><td>  6</td><td> 23</td><td> 81</td><td> 99</td></tr>
<tr class="even"><td class="label"> Compound Lazy </td><td> 100 </td><td> 898</td><td> 13</td><td> 20</td><td> 33</td><td> 41</td></tr>
</table>

Just like with the counters, the ordinary `SimpleGauge` is very slow, while `CompoundGauge` built on top of it is better, but still not ideal.

The lazy compound version, which we hoped for so much, is even worse. This time we don't need to look far for the reason: obviously, we've got caught by the false sharing again.
The `MutableGaugeValue` object is 48 bytes long and can share the same cache line with another such object. We need to pad it to 64 bytes by two extra `long`s:

{% highlight Java %}
{
    private long count = 0;
    private long sum = 0;
    private long max = Long.MIN_VALUE;
    private long min = Long.MAX_VALUE;
    public long dummy1;
    public long dummy2;
}
{% endhighlight %}

Here are the results:
<table class="numeric">
<tr><th rowspan="2">Gauge name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr><td class="label"> Compound Lazy </td><td>   0 </td><td>   2</td><td>  3</td><td>  4</td><td>   7</td><td>   5</td></tr>
<tr><td class="label"> Compound Lazy </td><td>  10 </td><td>  47</td><td>  0</td><td> 10</td><td>   7</td><td>  16</td></tr>
<tr><td class="label"> Compound Lazy </td><td>  50 </td><td> 394</td><td> 10</td><td> 33</td><td>  56</td><td>  63</td></tr>
<tr><td class="label"> Compound Lazy </td><td> 100 </td><td> 898</td><td> 14</td><td> 23</td><td>  34</td><td>  49</td></tr>
</table>

They are a bit better but still far from ideal. It is easy to see why. One gauge update involves update to four fields. Each of them can cause false sharing with any other.
Making the object 64 bytes long will only help if all the objects are aligned by 64 bytes. In practice, they are not, which is easy to check if we use our `Dump` tool
again and modify it to print the address modulo 64 as well as the distance to the previous object. Here is the output (with compressed objects off):

    9720:24 64:24 64:24 64:24 64:24 64:24 64:24 64:24 64:24 64:24 64:24 3408:40
    64:40 64:40 64:40 64:40 64:40 64:40 64:40 64:40 64:40 64:40 64:40 1648:24

Because of this, the last of the four fields can share the cache line with the first field of the following object. To make sure this doesn't happen, we need to provide
the distance of at least seven `long`s (56 bytes) between the blocks of fields. That requires three more dummy fields, which makes the size of our object 88 bytes.
And here are the times:

<table class="numeric">
<tr><th rowspan="2">Gauge name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr><td class="label"> Compound Lazy </td><td>   0 </td><td>   2</td><td> 3</td><td>   3</td><td>  3.</td><td>  3</td></tr>
<tr><td class="label"> Compound Lazy </td><td>  10 </td><td>  47</td><td> 6</td><td>   5</td><td>  5</td><td>   6</td></tr>
<tr><td class="label"> Compound Lazy </td><td>  50 </td><td> 394</td><td> 7</td><td>  11</td><td> 11</td><td>  14</td></tr>
<tr><td class="label"> Compound Lazy </td><td> 100 </td><td> 898</td><td> 9</td><td>  19</td><td> 20</td><td>  21</td></tr>
</table>

This is much better. It is still not as fast the counters, but already within our "almost good" range (under 20 ns). It has a disadvantage, though.
This scheme causes each value to occupy two cache lines, preventing any other data to be placed there, and cache lines are valuable resources: there are
only 512 of them in a 32K data cache. Aligning the values by 64 bytes would be better, and storing the values used by the same thread next to each other --
even better.

Now we are ready to introduce a `FastGauge` (the `LazyCompoundGauge` with a `ThreadLocal`-based facade):

<table class="numeric">
<tr><th rowspan="2">Gauge name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr><td class="label"> FastGauge </td><td>   0 </td><td>   2</td><td>  14</td><td>  14</td><td>  14</td><td>  14</td></tr>
<tr><td class="label"> FastGauge </td><td>  10 </td><td>  47</td><td>  10</td><td>  11</td><td>  11</td><td>  11</td></tr>
<tr><td class="label"> FastGauge </td><td>  50 </td><td> 394</td><td>  15</td><td>  15</td><td>  17</td><td>  22</td></tr>
<tr><td class="label"> FastGauge </td><td> 100 </td><td> 898</td><td>  22</td><td>  16</td><td>  26</td><td>  32</td></tr>
</table>

This is a bit slow on high thread counts and high delay factors, but still usable.

Possible problems
-----------------

Our reference variable is `volatile` while our value is not. **Java** promises that modified non-volatile variables will be flushed to memory before next
synchronisation point (entering or leaving the `synchronized` block or writing a `volatile` variable). Our solution may fail in the very unlikely case when
the program does not reach any synchronisation point before we start using collected data. Most likely, it won't fail even then -- typical code generators
prefer not to keep values in registers for such a long time if they can avoid that. However, a theoretical possibility exists.

Thread scheduling may also play a role. When talking about the effects of thread synchronisation, we usually look at multi-core scenarios, when the pieces of
code really run in parallel, but in our case the danger lies in the opposite case -- when several threads share one core. A client thread may be removed from the processor just before
it is ready to write the updated counter back into memory, and only scheduled back after 30 seconds. In the real life this is only a theoretical possibility.
Proper thread schedulers allocate CPU to threads without causing such delays (unless this is a very low-priority thread; probably, a low priority should be
a counter-indication for the use of lazy thread-local statistical variables).

Another case worth mentioning is a garbage collection pause. Let's imagine the garbage collector to kick in right after the counter update but before the memory write.
All the threads are suspended. The GC operation runs for 30 seconds. When the threads continue, the server thread wakes up first and, having discovered that 30 seconds
have passed, it proceeds to collect the data and happily uses incorrect values. This means that long garbage collection pauses must also be considered a counter-indication for lazy thread-local
variables. However, they should be considered a counter-indication for everything; if the GC delays are so long, there is something wrong with the program.
One should consider changing the JVM, or the GC, or its parameters, or the heap size, or, possibly, modifying the program. Perhaps, only some long-running batch processors
may tolerate such long GC delays, but programs like that do not require high-performance real-time statistical counters.

Anyway, the worst that may happen is a loss of some counts, or a partially updated gauge value, which will result in incorrect averages. However, since 
these variables aren't mission-critical, this unlucky event won't cause a real damage. I agree that any really mission-critical code must rely on proper
synchronisation and comply to the **Java** standard fully. And if it is slow, so be it.

If these problems are really experienced, one can consider increasing the cool-down period. It does not have to be half the collection
interval. It does not even have to be smaller than that interval. We can make it 10 minutes, although it would require storing the last 10 results somewhere.
If some thread suspends for 10 minutes or does not perform any synchronised operation for 10 minutes, the program has a problem
much bigger than just some counters being a bit out.

The volatile option
-------------------

If we feel really paranoid about our main counter variable not being declared `volatile`, we can declare it as such.
Let's add a `MutableVolatileCounterValue` class where the `value` is `volatile`. In the case of a gauge, we don't need to make all four values `volatile`, we can get away
with just one -- the last one updated (`counter` in our case). See `MutableVolatileGaugeValue` class.

We'll build a `LazyVolatileCounter` and `LazyVolatileGauge` classes on top of these, and test the lazy compound solutions with them. Here are the times for a counter:

<table class="numeric">
<tr><th rowspan="2">Counter name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr><td class="label"> Compound&nbsp;Lazy&nbsp;Volatile </td><td>   0 </td><td>   1</td><td>  10</td><td> 10</td><td>  10</td><td>  10</td></tr>
<tr><td class="label"> Compound&nbsp;Lazy&nbsp;Volatile </td><td>  10 </td><td>  47</td><td>   1</td><td>  1</td><td>   1</td><td>   1</td></tr>
<tr><td class="label"> Compound&nbsp;Lazy&nbsp;Volatile </td><td>  50 </td><td> 374</td><td>   4</td><td>  5</td><td>   7</td><td>  10</td></tr>
<tr><td class="label"> Compound&nbsp;Lazy&nbsp;Volatile </td><td> 100 </td><td> 887</td><td>   3</td><td>  5</td><td>  10</td><td>  18</td></tr>
</table>

And these are the times for gauges:

<table class="numeric">
<tr><th rowspan="2">Gauge name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr><td class="label"> Compound&nbsp;Lazy&nbsp;Volatile </td><td>   0 </td><td>   2</td><td> 6</td><td> 5</td><td>   5</td><td>   5</td></tr>
<tr><td class="label"> Compound&nbsp;Lazy&nbsp;Volatile </td><td>  10 </td><td>  47</td><td> 4</td><td> 4</td><td>   4</td><td>   5</td></tr>
<tr><td class="label"> Compound&nbsp;Lazy&nbsp;Volatile </td><td>  50 </td><td> 394</td><td> 6</td><td> 7</td><td>  11</td><td>  19</td></tr>
<tr><td class="label"> Compound&nbsp;Lazy&nbsp;Volatile </td><td> 100 </td><td> 898</td><td> 9</td><td> 8</td><td>  14</td><td>  23</td></tr>
</table>

The counters are a bit slower that the non-`volatile` ones, while with gauges the times are roughly the same. This means that both `volatile` and non-`volatile` solutions
are practically usable. Note that both are incorrect with respect to the **Java** spec. The `volatile` one is a bit more robust against the compiler keeping values in registers,
but both are vulnerable to the thread scheduling oddities and long GC pauses.

Personally, I still prefer the non-`volatile` solution. Our test was a bit artificial. All the test function was doing was calculating some floating-point
function. The real-life code is more likely to read and write some memory, which may cause the operations with the `lock` prefix to be more expensive. But I agree that
I may be wrong  here -- after all, I didn't manage to make an example where `volatile` would make running much slower.

Alternative solutions
---------------------

In our implementation of the `LazyCounter` and `LazyGauge` we allocate new object each time we request the value. I don't think this is a big expense, but some people
suggested that we can re-use the old ones. Specifically, if the cool-down period is set as half of the collection interval, we could get away with exactly two value objects.
At each point in time, the client thread is using one and the server thread is using the other one, and at some point they are swapped. We can keep both references inside the
counter object:

{% highlight Java %}
public class LazyCounter2 extends IndirectCounter
{
    private volatile MutableCounterValue current = new MutableCounterValue ();
    private MutableCounterValue other = new MutableCounterValue ();
    
    @Override
    public void add (long increment)
    {
        current.add (increment);
    }

    @Override
    public CounterValue getAndResetValue ()
    {
        CounterValue result = current;
        other.reset ();
        current = other;
        other = result;
        return result;
    }
}
{% endhighlight %}

We could combine these two values into an array:

{% highlight Java %}
public class LazyCounter3 extends IndirectCounter
{
    private MutableCounterValue [] values = new MutableCounterValue [2];
    private volatile currentIndex = 0;

    @Override
    public void add (long increment)
    {
        values [currentIndex].add (increment);
    }

    @Override
    public CounterValue getAndResetValue ()
    {
        values [1 - currentIndex].reset ();
        currentIndex = 1 - currentIndex;
        return values [1 - currnetIndex];
    }
}
{% endhighlight %}

We could make `currentIndex` global and share it between all the counters.

The additional benefit of all these versions is that they help prevent the false sharing problem: all the objects are allocated before the time and not necessarily
immediately after each other.

We could even go as far as place both values right inside the counter object and change the interface:

{% highlight Java %}
public class LazyCounter4 extends SwappableCounter
{
    private long value0 = 0;
    private long value1 = 0;
    private volatile currentIndex = 0;

    @Override
    public void add (long increment)
    {
        if (currentIndex == 0) {
            value0 += increment;
        } else {
            value1 += increment;
        }
    }

    @Override
    public void swap ()
    {
        if (currentIndex == 0) {
            value1 = 0;
        } else {
            value0 = 0;
        }
        currentIndex = 1 - currentIndex;
    }

    @Override
    public int get ()
    {
        return currentIndex == 0 ? value1 : value0;
    }
}
{% endhighlight %}

These are all valid solutions, but they share common disadvantages:

- The schedule of value-swapping is hard-coded into the counter or gauge code

- Previously, the "second effect" (see the `TrivialCounter` section) was eliminated. The abnormal thread scheduling could cause loss of some counts but not totally
  incorrect results. Once we start re-using the values, the scenarios with incorrect results become possible. No matter what we do, at some point we must reset the
counter value, and the client thread can update it right after that.

This, by the way, makes it difficult to implement a similar scheme in **C++**. Allocation of new objects doesn't work there, because no one can ever reliably destroy
the object; there is always a possibility that the client thread is still using it. Perhaps, the reference must be deposited into some kind of "purgatory", until next round of
requesting the values makes sure the old ones aren't used. This is one of very few cases where the garbage collector has some advantage
over the traditional memory management.

One other approach worth mentioning is the lazy allocation of value objects inside a counter:

{% highlight Java %}
public class LazyAllocCounter extends IndirectCounter
{
    private volatile MutableCounterValue current = null;
    
    @Override
    public void add (long increment)
    {
        MutableCounterValue cur = current;
        if (cur == null) {
            cur = new MutableCounterValue ();
            current = cur;
        }
        cur.add (increment);
    }

    @Override
    public CounterValue getAndResetValue ()
    {
        CounterValue result = current;
        current = null;
        return result;
    }
}
{% endhighlight %}

The `CompoundCounterValue` must  be modified to take care of `null` values. The memory is allocated in the client threads, and we can hope that the memory
manager allocates the values belonging to different threads far enough from each other, thus resolving the false sharing issue. We can then remove additional padding.
The times look quite good:

<table class="numeric">
<tr><th rowspan="2">Counter name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr><td class="label"> Compound&nbsp;LazyAlloc </td><td>   0 </td><td>   1</td><td> 2</td><td> 2</td><td> 2</td><td> 2</td></tr>
<tr><td class="label"> Compound&nbsp;LazyAlloc </td><td>  10 </td><td>  47</td><td>-1</td><td> 0</td><td> 0</td><td> 0</td></tr>
<tr><td class="label"> Compound&nbsp;LazyAlloc </td><td>  50 </td><td> 374</td><td> 1</td><td> 5</td><td> 4</td><td> 6</td></tr>
<tr><td class="label"> Compound&nbsp;LazyAlloc </td><td> 100 </td><td> 887</td><td> 4</td><td> 4</td><td> 8</td><td> 9</td></tr>
</table>

Similar solution can be implemented for gauges:

<table class="numeric">
<tr><th rowspan="2">Gauge name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr><td class="label"> Compound&nbsp;LazyAlloc </td><td>   0 </td><td>   2</td><td>  5</td><td>   4</td><td>    4</td><td>    4</td></tr>
<tr><td class="label"> Compound&nbsp;LazyAlloc </td><td>  10 </td><td>  47</td><td>  0</td><td>   0</td><td>    1</td><td>   1</td></tr>
<tr><td class="label"> Compound&nbsp;LazyAlloc </td><td>  50 </td><td> 394</td><td>  14</td><td>  18</td><td>  18</td><td>  18</td></tr>
<tr><td class="label"> Compound&nbsp;LazyAlloc </td><td> 100 </td><td> 898</td><td>  12</td><td>  14</td><td>  18</td><td>  22</td></tr>
</table>

Unfortunately, these solutions suffer from a race condition that can't be resolved by increasing the collection interval. Here is the sequence of events
that causes one counter loss. In the beginning `current` is `null`:

<table>
<tr><th>Client thread</th><th>Server thread</th></tr>
<tr><td>cur = current; // null  </td><td>result = current; // null</td></tr>
<tr><td>cur = new MutableCounterValue ();  </td><td></td></tr>
<tr><td>current = cur;  </td><td></td></tr>
<tr><td>cur.add (increment);  </td><td>current = null;</td></tr>
</table>

This is a very rare case: there must be no counter updates for the entire collection interval, and one must happen exactly at the time of collection. Still, it is
possible, and, since this is the only counter update in the entire collection interval, we don't want to miss it.

A very exotic solution
----------------------

We had to make our gauge object 88 bytes long because there was no way to tell JVM to align it at 64 bytes. We mentioned that this was inefficient with regards to cache lines.
We can arrange our own heap and make sure it is properly aligned; we'll allocate our own man-made objects and ignore the **Java** heap management. Our heap
will live inside a direct byte buffer. The object addresses inside this heap will be ordinary `int`s.

The `BufferGaugeValue` class will contain a `LongBuffer`, an object allocator and a set of static access methods:

{% highlight Java %}
public class BufferGaugeValue extends GaugeValue
{
    private static final int SIZE_LONGS = 1024;

    private static final LongBuffer heap =
        ByteBuffer.allocateDirect (8 * SIZE_LONGS)
                  .order (ByteOrder.LITTLE_ENDIAN)
                  .asLongBuffer ();
    private static int ptr0 = 0;
    private static int ptr = 0;
    
    private static final int COUNT_OFFSET = 0;
    private static final int SUM_OFFSET = 1;
    private static final int MAX_OFFSET = 2;
    private static final int MIN_OFFSET = 3;
    
    static {
        long addr = ((DirectBuffer) heap).address ();
        int offset = (int) (addr & 63) / 8;
        if (offset != 0) ptr0 += (8 - offset);
        ptr = ptr0;
        heap.limit (heap.capacity ());
    }
    
    public static int alloc ()
    {
        if (ptr + 4 > heap.capacity ()) {
            ptr = ptr0;
        }
        return (ptr += 8) - 8;
    }
       
    public static long getCount (int addr)
    {
        return heap.get (addr + COUNT_OFFSET);
    }

    public static void setCount (int addr, long val)
    {
        heap.put (addr + COUNT_OFFSET, val);
    }
{% endhighlight %}

On top of these operations we'll implement updating of the gauge value:

{% highlight Java %}
    public static void report (int addr, long value)
    {
        setCount (addr, getCount (addr) + 1);
        setSum (addr, getSum (addr) + value);
        long max = getMax (addr);
        long min = getMin (addr);
        if (value > max) setMax (addr, value);
        if (value < min) setMin (addr, value);
    }
{% endhighlight %}

And then we need a proper **Java** object to encapsulate our artificial object inside a `CompoundGaugeValue`:

{% highlight Java %}
    private int addr;
    
    public BufferGaugeValue (int addr)
    {
        this.addr = addr;
    }

    @Override
    public long getCount ()
    {
        return getCount (addr);
    }
{% endhighlight %}

Finally, we create a `LazyBufferGauge`:

{% highlight Java %}
    private volatile int current = BufferGaugeValue.alloc ();

    @Override
    public void report (long value)
    {
        BufferGaugeValue.report (current, value);
    }

    @Override
    public GaugeValue getAndReset ()
    {
        GaugeValue result = new BufferGaugeValue (current);
        current = BufferGaugeValue.alloc ();
        return result;
    }
{% endhighlight %}

Obviously, the buffer operations introduce additional delays, and there are other drawbacks, such as limited hard-coded heap size, but our goal of having our objects
aligned by 64 bytes is achieved, and the times are good:

<table class="numeric">
<tr><th rowspan="2">Gauge name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr><td class="label"> Compound&nbsp;LazyBufferGauge </td><td>   0 </td><td>   2</td><td> 6</td><td>   5</td><td>   5</td><td>   5</td></tr>
<tr><td class="label"> Compound&nbsp;LazyBufferGauge </td><td>  10 </td><td>  47</td><td> 4</td><td>   4</td><td>   4</td><td>   5</td></tr>
<tr><td class="label"> Compound&nbsp;LazyBufferGauge </td><td>  50 </td><td> 394</td><td> 8</td><td>  10</td><td>  15</td><td>  17</td></tr>
<tr><td class="label"> Compound&nbsp;LazyBufferGauge </td><td> 100 </td><td> 898</td><td> 9</td><td>  12</td><td>  17</td><td>  20</td></tr>
</table>

I wouldn't recommend this solution for this specific problem, but it mustn't be completely discarded as something completely ridiculous. We'll keep it in
our toolbox and possibly find some use for it in later articles.

A combined statistics approach
------------------------------

The best way of allocating the thread-local values would be to put together values belonging to the same thread but to different counters and gauges. This would
make the most efficient use of cache lines. It would, however, make the overall design of the statistical system too complex, so I won't follow this idea up
here.

Summary
-------

Let's bring together all the solutions that produce correct or almost correct results. To make the lists shorter, let's choose one realistic delay factor, 50
(base time 370&nbsp;--&nbsp;390 ns). Here are the times for the counters:

<table class="numeric">
<tr><th rowspan="2">Counter name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr><td class="label"> SimpleCounter                    </td><td>  50 </td><td> 374</td><td>  25</td><td> 365</td><td> 2512</td><td> 7981</td></tr>
<tr><td class="label"> AtomicCounter                    </td><td>  50 </td><td> 374</td><td>   3</td><td>  72</td><td>  581</td><td> 1559</td></tr>
<tr><td class="label"> Compound&nbsp;Simple             </td><td>  50 </td><td> 374</td><td>  24</td><td>  46</td><td>   37</td><td>   51</td></tr>
<tr><td class="label"> Compound&nbsp;Atomic             </td><td>  50 </td><td> 374</td><td>   5</td><td>   6</td><td>    4</td><td>    4</td></tr>
<tr><td class="label"> Compound Lazy                    </td><td>  50 </td><td> 374</td><td>  -1</td><td>   1</td><td>    2</td><td>    3</td></tr>
<tr><td class="label"> FastCounter                      </td><td>  50 </td><td> 374</td><td>  -1</td><td>  -1</td><td>    3</td><td>    3</td></tr>
<tr><td class="label"> FastAtomicCounter                </td><td>  50 </td><td> 374</td><td>  16</td><td>  27</td><td>   21</td><td>   21</td></tr>
<tr><td class="label"> Compound&nbsp;Lazy&nbsp;Volatile </td><td>  50 </td><td> 374</td><td>   4</td><td>   5</td><td>    7</td><td>   10</td></tr>
<tr><td class="label"> Compound&nbsp;LazyAlloc          </td><td>  50 </td><td> 374</td><td>   1</td><td>   5</td><td>    4</td><td>    6</td></tr>
</table>


And here are the times for gauges:

<table class="numeric">
<tr><th rowspan="2">Gauge name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr><td class="label"> Simple                           </td><td>  50 </td><td> 394</td><td> 35</td><td> 107</td><td>1884</td><td>5866</td></tr>
<tr><td class="label"> Compound Simple                  </td><td>  50 </td><td> 394</td><td> 42</td><td>  41</td><td>  43</td><td>  42</td></tr>
<tr><td class="label"> Compound Lazy                    </td><td>  50 </td><td> 394</td><td>  7</td><td>  11</td><td>  11</td><td>  14</td></tr>
<tr><td class="label"> FastGauge                        </td><td>  50 </td><td> 394</td><td> 15</td><td>  15</td><td>  17</td><td>  22</td></tr>
<tr><td class="label"> Compound&nbsp;Lazy&nbsp;Volatile </td><td>  50 </td><td> 394</td><td>  6</td><td>   7</td><td>  11</td><td>  19</td></tr>
<tr><td class="label"> Compound&nbsp;LazyAlloc          </td><td>  50 </td><td> 394</td><td> 14</td><td>  18</td><td>  18</td><td>  18</td></tr>
<tr><td class="label"> Compound&nbsp;Lazy&nbsp;Buffer   </td><td>  50 </td><td> 394</td><td>  8</td><td>  10</td><td>  15</td><td>  17</td></tr>
</table>

Conclusions
-----------

- The ill-effects of not using synchronisation aren't just theoretical possibility; they do in fact happen in real life, and are very prominent.

- Absence of synchronisation does not yet make programs fast.

- However, its presence may make it slow. While `synchronized` blocks are a convenient way to address shared data structures, they aren't really suitable for high-performance
  computing.

- Atomic objects of **Java** perform much better than `synchronized` blocks, and should be preferred to those wherever possible.

- However, even they aren't fast enough for high-performance operation. High operation speed can only be achieved by using thread-local objects, not shared ones.

- Updating of a variable from multiple threads is slow and should be avoided if ultimate speed is needed.

- False sharing is really a problem. While real sharing is under our control, false sharing is not. It may happen in completely unexpected places, and it is
  difficult to get rid of reliably. It really slows down multi-threaded programs a lot.

- We've made very fast implementations of counters and gauges, which are not completely trivial and also not 100% legal. But they work.

- Counters allow completely legal implementation based on atomic values. It is only a little bit slower than the illegal one. Gauges, however, do not allow such
  solution.

- We expected the biggest overhead when running with delay factor of zero; in reality, the biggest overhead happened at the highest delay factor. Why this happens, requires
  additional investigation, but the effect is good: the overheads are increased exactly when we can afford it.

- Current programming fashion is immutable objects. We, however, achieved substantial performance gain by using mutable ones. This shows that no rule is absolute.

- When programming in **Java**, one still has to worry about cache lines and LOCK prefixes. This is sad, because **Java** is positioned as an application
  language, not a system programming one. Ideally, all such issues should be taken care of by the language implementation.

- The last point is especially bad, because it undermines the platform-independent status of **Java**. Who knows what the cache structure is on other processors.
  Tomorrow Intel may increase the cache line size to 128 bytes, and we'll have to update the solution.

- If someone knows a suitable solution for gauges using atomics, please let me know.

Coming soon
-----------

I want to try all these approaches in **C++**. Some might still be applicable there.

I also want to investigate some other structures typically used in high-performance computing, such as queues. Our exprerience with synchronised objects so far was
not very encouraging -- what if the classes we usually use are not optimal? What is the performance we can achieve?

Update: `LongAccumulator` and `LongAdder`
-----------------------------------------

This article is based on the relatively old project. Since that time some new functionality has been added to **Java**. During discussion
on [reddit](https://www.reddit.com/r/programming/comments/6118ef/statistical_variables_in_java_not_quite_legal_but/), [kevinherron](https://www.reddit.com/user/kevinherron)
suggested two implementations based on the new classes, which were introduced in **Java 1.8** -- `LongAccumulator` and `LongAdder`, both from `java.util.concurrent.atomic` package.

Here are the versions he suggested:

{% highlight Java %}
import java.util.concurrent.atomic.LongAccumulator;

public class LongAccumulatorCounter extends ServerCounter {

    private final LongAccumulator accumulator = new LongAccumulator(
        (left, right) -> left + right,
        0L
    );

    @Override
    public void add(long increment) {
        accumulator.accumulate(increment);
    }

    @Override
    public long getAndReset() {
        return accumulator.getThenReset();
    }
}
{% endhighlight %}

and

{% highlight Java %}
import java.util.concurrent.atomic.LongAdder;

public class LongAdderCounter extends ServerCounter {

    private final LongAdder adder = new LongAdder();

    @Override
    public void add(long increment) {
        adder.add(increment);
    }

    @Override
    public long getAndReset() {
        return adder.sumThenReset();
    }
}
{% endhighlight %}

Both of the new utility classes offer distributed thread-random cache-aware accumulation, just the `Accumulator` is capable of performing any operations on the incoming
numbers, while the `Adder` can only add. Both are suitable for counters, and both promise to be fast. Let's try them. Here are the times:

<table class="numeric">
<tr><th rowspan="2">Counter name</th><th rowspan="2">Delay factor</th><th rowspan="2">Base time</th><th colspan="4">Time, ns, for thread count</th></tr>
<tr><th>1</th><th>2</th><th>6</th><th>12</th></tr>
<tr class="even"><td class="label"> LongAccumulatorCounter </td><td>   0 </td><td>   1</td><td>  13</td><td>  13</td><td>  13</td><td>  13</td></tr>
<tr class="even"><td class="label"> LongAccumulatorCounter </td><td>  10 </td><td>  47</td><td>   5</td><td>   6</td><td>   6</td><td>   7</td></tr>
<tr class="even"><td class="label"> LongAccumulatorCounter </td><td>  50 </td><td> 347</td><td>  12</td><td>  13</td><td>  17</td><td>  20</td></tr>
<tr class="even"><td class="label"> LongAccumulatorCounter </td><td> 100 </td><td> 887</td><td>   8</td><td>   7</td><td>  16</td><td>  21</td></tr>
<tr><td class="label"> LongAdderCounter </td><td>   0 </td><td>   1</td><td>  14</td><td>  14</td><td>  14</td><td>  14</td></tr>
<tr><td class="label"> LongAdderCounter </td><td>  10 </td><td>  47</td><td>   3</td><td>   4</td><td>   5</td><td>   5</td></tr>
<tr><td class="label"> LongAdderCounter </td><td>  50 </td><td> 387</td><td>  12</td><td>  14</td><td>  17</td><td>  20</td></tr>
<tr><td class="label"> LongAdderCounter </td><td> 100 </td><td> 887</td><td>   8</td><td>   8</td><td>  16</td><td>  21</td></tr>
</table>

The times are reasonable and they don't deteriorate much with the thread count -- exactly what we need. It is also worth mentioning that the `Accumulator` isn't slower than
the `Adder` -- JIT has done a good job here.

Unfortunately, the results are not correct. Here are some sample numbers for the `Adder` on
12 threads and delay factor 0:

    Counter sum:   8019220551
    Correct sum:   8019220615

And here are the results for the `Accumulator`:

    Counter sum:   8478867056
    Correct sum:   8478867115

Even on one thread and delay of 100 we see errors -- for example, on `Accumulator`:

    Counter sum:     11320241
    Correct sum:     11320242

This is to be expected, since neither the `Adder`, nor `Accumulator` offer correct real-time retrieval. They offer correct multi-threaded counting, but the results
must be collected after the threads finish, otherwise some counts may be lost. This is from the `Adder` Javadoc:

> If there are
updates concurrent with this method, the returned value is
**not** guaranteed to be the final value occurring before
the reset.

This, however, is very easy to fix. If some updates that happened right at the collection time, it is normal for them to be added to either of the two intervals,
the one just finished or the one just started. After all, thread scheduling could have caused collection to happen a bit later or earlier. As long as the counts are
not lost, it's all right. All we need to do is to run the adder (or accumulator) continuously and never reset it. The counter can compute and report the delta to the
previous value:

{% highlight Java %}
public class LongAccumulatorCounter extends ServerCounter {

    private long start = 0;
    
    private final LongAccumulator accumulator = new LongAccumulator(
        (left, right) -> left + right,
        0L
    );

    @Override
    public void add(long increment) {
        accumulator.accumulate(increment);
    }

    private long start = 0;

    @Override
    public long getAndReset() {
        long current = accumulator.get();
        long result = current - start;
        start = current;
        return result;
    }
}
{% endhighlight %}

and

{% highlight Java %}
public class LongAdderCounter extends ServerCounter {

    private long start = 0;

    private final LongAdder adder = new LongAdder();

    @Override
    public void add(long increment) {
        adder.add(increment);
    }

    @Override
    public long getAndReset() {
        long current = adder.sum ();
        long result = current - start;
        start = current;
        return result;
    }
}
{% endhighlight %}

Note that we don't need to protect the `start` variable or make it `volatile`: it is only ever accessed from the server thread.

The results are now correct.

We could make continuous accumulating the regular feature of the counter and move calculating of the delta to the server side. This would eliminate competition between server
and client threads. It would, however, complicate our compound solutions.

This means that my statement above ("When programming in **Java**, one still has to worry about cache lines and LOCK prefixes") is not completely correct. In **Java 1.8**
there are built-in classes that take care of these complexities, and we can go quite far using these classes. Two points, however, keep the stament alive:

- Our lazy compound implementation is still significantly faster (3 ns for `FastCounter` on 12 threads, delay 50 vs 20 ns for these two)

- These two solutions are only applicable to counters, not for gauges.

Comments are welcome below or on [reddit](https://www.reddit.com/r/programming/comments/6118ef/statistical_variables_in_java_not_quite_legal_but/).
