---
layout: post
title:  "Building a fast queue between C++ and Java"
date:   2018-07-18 12:00:00
tags: C++ Java optimisation real-time
---

Today we are going to look at one of the of the most often used tools in parallel computing: a queue. There are many types of queues; we'll now concentrate on one of them: 
a single-producer, single-consumer queues. Being the simplest of them all, this type still allows many important uses:

- arranging a pipeline: if some job allows decomposition into sequentially performed operations, we can give each operation its own thread.
In this case a pipe (a single-producer, single-consumer queue) is placed between each two consecutive stages of the pipeline;

- handling abnormal spikes in the incoming data rate (buffering);

- simplifying the program design by replacing callbacks with direct calls or vice versa. It is usually much easier and less error-prone to iterate a complex data structure
  using callbacks than iterators (no need to store complex iterator states). On the other hand, the consumers often benefit from direct iterating, for the same reason.
  Queues can help connect the modules; this seems to be the preferred way of operation in **Go** programming language;

- while this is not a typical use of the queues, this was, in fact, the main motivation for this research. The idea is to use queues as clock sources.
  The plan is to build a fast queue with endpoints in **C++** and **Java**. The **C++** thread will
write tokens to the queue at equal time intervals, while **Java** thread will read them and perform some operation.
Any delay in this operation will increase the amount of queued data,
causing queue overflow and, in the extreme cases, data loss. This will allow measuring the effect of garbage collection and other delays in **Java**.

The safest way to design a multi-threaded program is to assume a relatively slow inter-thread communication (a coarse parallelism).
Does it have to be so? Is there perhaps some implementation of a queue that can be used for a fine-grain parallelism?

We have arrived at the topic of this article: what is the speed that can be achieved
by a single-producer, single-consumer queue? 

I have to warn the reader that this is going to be a very long journey. I recomment a reader  who is short of time to scroll down straight to the
["Dual-array queue"](#dual).

Blocking vs non-blocking
------------------------

In most cases, when a consumer encounters an empty queue, it has nothing to do and may as well block. Since the wait-notify mechanism is often integrated with the
synchronisation mechanism (like in **Java**), it makes sense to incorporate blocking of the consumer into the queue mechanism itself.

A bigger problem is what to do when a producer encounters a full queue. There are three options then:

- to drop the data
- to grow the queue
- to block.

There are cases where integrity of the data is so important that nothing may be allowed to drop. Often in these cases we have control over the input data rate (e.g.
the data is read from the DB or from the disk). Blocking is a preferred option then. In the list of the use cases above, one case (simplifying iterators)
definitely requires blocking.

However, in real-time programming the incoming data rate is often outside of our control. For instance, when analysing the network traffic, demodulating the radio waves or processing
the streaming video, we must deal with the data at the rate it comes. The queue can help with sudden short bursts of data, but if the data continuously comes faster than we process it,
there is no choice but to drop some portions of it. Dropping the data at the input of a queue is the best and the most commonly used option, and the amount of the data dropped
is the measure of how well-dimensioned the system is.

In short, we'll concentrate on single-producer, single-consumer, producer-dropping, consumer-blocking queues. We'll study the producer-blocking queues later, too.

We'll write in **C++**, with porting the solution to **Java** in mind.

The test
--------

The test will involve two threads, the producer and the consumer, running on two different cores and connected by a queue. We'll try to maximise the queue throughput,
giving it priority over the overall system performance, meaning by it that we won't hesitate to use spin loops where needed, even though they waste CPU cycles and burn
extra joules.

We'll be using very short messages: one single `int32_t` (four bytes). Each message will contain a sequence number, which will be used to detect packet loss.
The consumer will measure the packet loss and display it after some pre-defined number of messages received (10M).

I found it very difficult to measure the performance of the queue on its own, without any client applicaion. Even for the slow implementations, the variation is big,
despite the stabilising effort (see later). The measurement is even more problematic when we approach nanosecond intervals. Often the slight variations in the input
data rate cause big differences in the resulting throughput. This means that all the results achieved must be viewed only as approximations. The top-performers
will probably do better in the real life than the outsiders, but finer comparisons will always be imprecise.

This is what we are going to measure now. The producer will work in two modes:

- a performance mode, when it sends messages as fast as it can; we'll measure how long an average write takes and publish it as the `Writing time` (in nanoseconds).
  This,  on its own, isn't a good performance metric: there is no point of writing fast if we can't read fast. However, this is some measure of the overhead of the
synchronisation mechanism used.

- a clock mode, when it sends messages at certain frequency, determined by the specified time interval. We'll adjust this interval manually to find the shortest one where
 the packets aren't lost.  This interval will be published as `Good interval`, and the corresponding packet rate as `Throughput`.

In the clock mode we'll also measure the average queue size, as measured by the consumer at each read operation. Since it slows down the operation, the size measurement
will be switched on and off by a conditional compilation variable.

[We've learned that time reporting is not always fast in Linux]({{ site.ART_SLOW_CURRENT_TIME_MILLIS }}), and even when it is, it is not as fast as using the processor's time stamp counter (TSC).
This is why we'll be using `RDTSC` instruction to set the clock rate of the producer. This limits the code to Intel processors, and this is where we'll run the test:
a dual Xeon&reg; CPU E5-2620 v3 @ 2.40GHz, using Linux with a 3.17.4 kernel, and GCC 4.8.3.

The queue is defined by the following interface:

{% highlight C++ %}
template<typename E> class Queue
{
public:
    virtual void write(const E& elem) = 0;
    virtual void read(E& elem) = 0;
};
{% endhighlight %}

The test framework consists of the following components, all running in their own threads:

- `DataSource`: writes data to the queue at time intervals provided by a timer. The elements written to the queue contain incrementing sequence numbers;

- `DataSink`: reads elements from the queue as fast as it can and detects data loss using sequence numbers. After each 10M elements it reports the statistics
to the `Reporter` thread (using another queue);

- `Reporter`: a thread that measures time and reports lost packets.

The timer implements the following interface:

{% highlight C++ %}
    void start();
    void iteration(uint64_t i);
{% endhighlight C++ %}

We'll be using two implementations: `empty_timer`, which doesn't wait at all, and `hires_timer`, which looks like this:

{% highlight C++ %}
double freq;

class hires_timer
{
    const unsigned FACTOR = 1024;
    const unsigned interval;
    uint64_t start_time;

public:
    hires_timer (double interval)
                : interval ((unsigned)round(freq * interval * FACTOR))
    {
    }

    inline void start()
    {
        start_time = __rdtsc();
    }

    inline void iteration(uint64_t i)
    {
        sleep_until(interval * i / FACTOR + start_time);
    }

private:
    inline void sleep_until(uint64_t until)
    {
        while (__rdtsc() < until);
    }
};
{% endhighlight %}

This timer uses the `RDTSC` instruction and, therefore, depends on the correct value of the processor's clock rate. This value is calculated upfront using a calibration
procedure (see the source), and stored in the `freq` variable (in GHz).

The timer is configured with the average interval between iterations, in nanoseconds. It can run unevenly (generate a burst of ticks if falling behind).
However, it is worth noticing that it performs at least one `RDTSC` instruction at every iteration, so its rate is limited by the latency of this instruction
(8 ns on this processor).

We'll use the queue size of 100,000. We'll check later how different queue types respond to a change of the queue size.

The full source code can be found in [the repository]({{ site.REPO-QUEUE }})..

Test precautions
----------------

Tests like this often show very inconsistent behaviour. The results vary from one run to another. That's why it is important to eliminate as many variables as possible:

- the Turbo boost [must be switched off]({{ site.ART-TURBO-BOOST }});

- the system clock source must be [set to TSC]({{ site.ART_SLOW_CURRENT_TIME_MILLIS }});

- our system is a multi-processor one. The effects of the multi-processor environment fall outside of the scope of this article; so we'll limit the execution to
just one physical processor. Besides, the results of the `RDTSC` instruction are not consistent across processor cores. So we'll bind our data source, data sink and reporter
threads to specific cores of the same processor using Linux thread affinity control tools (beware of the hypertheading; it must either be switched off, or taken
into account while making thread affinity masks to avoid threads running on the same physical core);

- we'll limit the execution to the NUMA node associated with the processor we run on; the `numactl` utility helps there;

- whenever possible, we will allocate all our data structures at the 64 byte (the cache line size) boundary. For that, we'll add the following to the declaration of our
`struct elem`:

{% highlight C++ %}
    static void* operator new[](size_t sz)
    {
        return _mm_malloc(sz * sizeof(elem), CACHE_LINE_SIZE);
    }

    static void operator delete [](void * p)
    {
        _mm_free(p);
    }
{% endhighlight %}

Here `_mm_malloc` and `_mm_free` come from the SSE library. I would have preferred to use `std::aligned_alloc`, but it has only appeared in `C++17`, which I don't have
available yet.

Strangely, it works on GCC but not on MSVC;

- obviously, make sure as few processes as possible are running on the host.

With all these precautions, I haven't managed to achieve true consistence of the tests. What's interesting, the results are usually the same within one program run,
 but differ from one run to another. I would really appreciate if someone could explain this behaviour and suggest a way to circumvent it.

I'll publish the test results as ranges. I won't bother with averages or standard deviations, because I don't know what the reason for the variance is and,
therefore, how representative the sample is.

Now everything is ready; let's hit the road.

The standard queue
------------------

This is the most obvious of all queue implementations. It is based directly on the synchronisation and container primitives from the standard library:

{% highlight C++ %}
template<typename E> class StandardQueue : public Queue<E>
{
    std::mutex mutex;
    std::condition_variable cv;
    std::deque<E> queue;
    const size_t size;

public:
    StandardQueue(size_t size) : size (size)
    {
    }

    void write(const E& elem)
    {
        {
            std::lock_guard<std::mutex> lock(mutex);
            if (queue.size() >= size)
                return;
            queue.push_back(elem);
        }
        cv.notify_one();
    }

    void read(E& elem)
    {
        std::unique_lock<std::mutex> lock(mutex);
        cv.wait(lock, [this] {return ! queue.empty(); });
        elem = queue.front();
        queue.pop_front();
    }
};
{% endhighlight %}

Here are the run results:

<table class="numeric">
<tr><th>Queue name</th><th>Writing time, ns</th><th>Good interval, ns</th><th>Throughput, Mil/sec</th><th>Avg size</th></tr>
<tr><td class="label"> Standard            </td><td>  280-500 </td><td> 380-550   </td><td> 2.6   </td><td>  230 </td></tr>
</table>

(We showed the highest value for the throughput).

This packet rate is somewhere between the rate typical for a coarse parallelism (microseconds and milliseconds),
and a fine-grain parallelism (tens to hundreds of nanoseconds). This is not bad, but we must be able to do better than this.

The circular queue
------------------

Our next queue is also using standard synchronisation utilities but makes its own implementation of a circular queue inside an array instead of the standard class
`deque`:

{% highlight C++ %}
template<typename E> class CircularQueue : public Queue<E>
{
    std::mutex mutex;
    std::condition_variable cv;
    E * const queue;
    const size_t size;
    volatile size_t read_ptr;
    volatile size_t write_ptr;

public:
    CircularQueue(size_t size) : queue(new E[size]), size(size),
                                 read_ptr(0), write_ptr(0)
    {
    }
    
    ~CircularQueue()
    {
        delete[] queue;
    }

    void write(const E& elem)
    {
        {
            std::lock_guard<std::mutex> lock(mutex);
            size_t w = write_ptr;
            size_t nw = w + 1;
            if (nw >= size) nw = 0;
            if (nw == read_ptr) {
                return;
            }
            queue[w] = elem;
            write_ptr = nw;
        }
        cv.notify_one();
    }

    void read(E& elem)
    {
        std::unique_lock<std::mutex> lock(mutex);
        cv.wait(lock, [this] {return read_ptr != write_ptr; });
        size_t r = read_ptr;
        elem = queue[r];
        if (++r == size) r = 0;
        read_ptr = r;
    }
};
{% endhighlight %}

It is unlikely that in the `std::deque` was the bottleneck in the previous implementation, so we shouldn't expect big performance gain, and, in fact, it got slower:

<table class="numeric">
<tr><th>Queue name</th><th>Writing time, ns</th><th>Good interval, ns</th><th>Throughput, Mil/sec</th><th>Avg size</th></tr>
<tr><td class="label"> Circular            </td><td>  364-395 </td><td> 430       </td><td> 2.3   </td><td>  290 </td></tr>
</table>

We won't, however, revert back to the standard library, because we are going to manipulate the code of the circular queue. In particular, we'll integrate it with the
synchronisation code. That's why we haven't moved the circular queue logic into a separate piece of code, as the usual good coding practice suggests.

No wait queue
-------------

In the above examples, the two sides of a queue used a notification mechanism (condition variables) to notify the reader that there is data. This is completely appropriate when
we are short of processor capacity and can't afford wasting CPU cycles while waiting for data. However, we've decided to put the queue throughput above everything else.
So let's try some spin loops.

Let's start with the simplest of the spin loop solutions, where  the `Circular` code is used as is (including the mutex), but no conditional variable is used:

{% highlight C++ %}
template<typename E> class NoWaitCircularQueue : public Queue<E>
{
    std::mutex mutex;
    E * const queue;
    const size_t size;
    volatile size_t read_ptr;
    volatile size_t write_ptr;

public:
    NoWaitCircularQueue(size_t size) : queue(new E[size]), size(size),
                                       read_ptr(0), write_ptr(0)
    {
    }

    ~NoWaitCircularQueue()
    {
        delete[] queue;
    }

    void write(const E& elem)
    {
        std::lock_guard<std::mutex> lock(mutex);
        size_t w = write_ptr;
        size_t nw = w + 1;
        if (nw >= size) nw = 0;
        if (nw == read_ptr) {
            return;
        }
        queue[w] = elem;
        write_ptr = nw;
    }

    void read(E& elem)
    {
        while (true) {
            {
                std::lock_guard<std::mutex> lock(mutex);
                size_t r = read_ptr;
                if (r != write_ptr) {
                    elem = queue[r];
                    if (++r == size) r = 0;
                    read_ptr = r;
                    return;
                }
            }
            yield();
        }
    }
};
{% endhighlight %}

Here when there is nothing to do the program calls `yield`, which is defined as following:

{% highlight C++ %}
inline void yield ()
{
    _mm_pause();
}
{% endhighlight %}

This is a CPU instruction that hints to the CPU that it is in a spin loop and must slow down a little. It is supposed to increase the performance of memory access.
At least, this is what Intel manual says; I haven't actually seen this happening. In addition, [there are reports that this instruction has become very slow
on the Skylake family on processors](https://aloiskraus.wordpress.com/2018/06/16/why-skylakex-cpus-are-sometimes-50-slower-how-intel-has-broken-existing-code/).
Probably, we'll have to do without it on those.

<table class="numeric">
<tr><th>Queue name</th><th>Writing time, ns</th><th>Good interval, ns</th><th>Throughput, Mil/sec</th><th>Avg size</th></tr>
<tr><td class="label"> No wait             </td><td>  310     </td><td> 340       </td><td> 2.9   </td><td> 1030 </td></tr>
</table>

The result is mixed: the throughput got a bit higher, but so did the average queue size.

Spin-lock queue
---------------

The next step is to replace the mutex with some mechanism that doesn't involve the OS. Let's try a spin-lock. Here is a simple implementation that's made in the usual 
`C++` tradition: the constructor acquires the lock while the destructor releases it:

{% highlight C++ %}
class spinlock
{
    std::atomic_flag &flag;

public:
    spinlock(std::atomic_flag &flag) : flag(flag)
    {
        while (flag.test_and_set(std::memory_order_acquire)) yield();
    }

    ~spinlock()
    {
        flag.clear(std::memory_order_release);
    }
};
{% endhighlight %}

This lock can be used instead of the `lock_guard` in the previous solution:

{% highlight C++ %}
template<typename E> class SpinCircularQueue : public Queue<E>
{
    std::atomic_flag flag;
    E * const queue;
    const size_t size;
    volatile size_t read_ptr;
    volatile size_t write_ptr;

public:
    SpinCircularQueue(size_t size) : queue(new E[size]), size(size),
                                     read_ptr(0), write_ptr(0)
    {
    }

    ~SpinCircularQueue()
    {
        delete[] queue;
    }

    void write(const E& elem)
    {
        spinlock lock (flag);
        size_t w = write_ptr;
        size_t nw = w + 1;
        if (nw >= size) nw = 0;
        if (nw == read_ptr) {
            return;
        }
        queue[w] = elem;
        write_ptr = nw;
    }

    void read(E& elem)
    {
        while (true) {
            spinlock lock(flag);
            size_t r = read_ptr;
            if (r == write_ptr) {
                yield();
                continue;
            }
            elem = queue[r];
            if (++r == size) r = 0;
            read_ptr = r;
            return;
        }
    }
};
{% endhighlight %}

Note that this solution is not guaranteed to work at all. The spin-lock does not even try to implement any kind of fairness, so if a thread releases the lock and
re-acquires it immediately (as our reading thread does when waiting for data), another thread may have no chance to slip through.

Testing confirms that this is indeed very unreliable solution. The writing time varies between 1000 and 200,000 ns, even though the average queue size doesn't
go too high (700-2000). We'll discard this solution and try more reliable options.


Atomic queue
------------

All the implementations so far followed one pattern: there is some data structure (a queue in our case), which is not thread-safe on its own, because it consists of
multiple values that must be kept consistent with each other. We then use a mutual exclusion mechanism to protect the structure from concurrent access.

In a case of some very complex data structure this is the best we can do; in the case of a queue, however, we can do better. A reader and a writer may use the
queue simultaneously as long as they access different parts of it.  This is exactly what happens when the queue is neither full nor empty. Obviously, to establish this fact,
the threads must read some shared variables, but they don't need any mutexes for that: atomic variables are sufficient.

The reader and the writer threads share two pointer (technically, index) variables: `read_ptr` and `write_ptr`, and they can use these two variables for synchronisation:

- if `read_prr == write_ptr`, the queue is empty, and the reader must wait;

- if `(write_ptr + 1) % size == read_ptr`, the queue is full, and the writer must drop its message;

- otherwise they are accessing different elements of the underlying array and can safely proceed to perform their operations.

Using these two pointer variables as synchronisation tools imposes some requirements on them:

- they must be **volatile**, which means that they must be written to memory as soon as they are modified and read when they are needed. In short, they mustn't be placed
into registers for any substantial period of time.

- they must be **atomic**, which means that no thread may ever see any partial modification of them. For instance, 64-bit variables were not atomic on 32-bit
processor architectures. Arrays of 1000 bytes are not atomic anywhere.

- there must be some **memory order** enforced; in particular, we must make sure that the reading thread reads the updated
`write_ptr` only after the new item has already been placed onto the queue, and the writing thread reads the updated `read_ptr` only after the item has been copied from it.

In **C++** all of this can be achieved using atomic variables:

{% highlight C++ %}
template<typename E> class AtomicCircularQueue : public Queue<E>
{
    E * const queue;
    const size_t size;

    std::atomic<size_t> read_ptr;
    std::atomic<size_t> write_ptr;

public:
    AtomicCircularQueue(size_t size) : queue(new E[size]), size(size),
                                       read_ptr(0), write_ptr(0)
    {
    }

    ~AtomicCircularQueue()
    {
        delete[] queue;
    }

    void write(const E& elem)
    {
        size_t w = write_ptr;
        size_t nw = w + 1;
        if (nw >= size) nw = 0;
        if (nw == read_ptr.load (std::memory_order_consume)) {
            return;
        }
        queue[w] = elem;
        write_ptr.store(nw, std::memory_order_release);
    }

    void read(E& elem)
    {
        size_t r = read_ptr;
        size_t w;
        while (r == (w = write_ptr.load (std::memory_order_acquire))) {
            yield();
        }
        elem = queue[r];
        if (++r == size) r = 0;
        read_ptr.store (r, std::memory_order_release);
    }
};
{% endhighlight %}

This promises to be a big step forward compared to the previous solutions. The reader and the writer are still tightly inter-dependent (they access each other's
variables on every call), but any kind of waiting only happens at the reader side and only when the queue is empty, which is the natural time we should wait.
The fairness problem is therefore solved.

<table class="numeric">
<tr><th>Queue name</th><th>Writing time, ns</th><th>Good interval, ns</th><th>Throughput, Mil/sec</th><th>Avg size</th></tr>
<tr><td class="label"> Atomic              </td><td>  23-25   </td><td> 42-46     </td><td> 21-23 </td><td> 26 </td></tr>
</table>

The improvement is indeed impressive. It's a leap from a coarse to a fine parallelism. The results are so much better than anything we've seen so far that we could
stop right here. Let's, however, look closer at the result -- what if it can be improved?

It's interesting to look at the code generated for the queue operations. Here is `write`:

{% highlight C++ %}
_ZN19AtomicCircularQueueI4elemE5writeERKS0_:
	movq	32(%rdi), %rdx
	movl	$0, %ecx
	leaq	1(%rdx), %rax
	cmpq	16(%rdi), %rax
	cmovae	%rcx, %rax
	movq	24(%rdi), %rcx
	cmpq	%rcx, %rax
	je	.L1
	movq	8(%rdi), %rcx
	movl	(%rsi), %esi
	movl	%esi, (%rcx,%rdx,4)
	movq	%rax, 32(%rdi)
.L1:
	rep ret
{% endhighlight %}

Note that here `%rdi` is the first parameter (`this`), and `%rsi` is the second one (the pointer to `elem`). And here are the fields:

-  `32(%rdi)` is `write_ptr`
-  `16(%rdi)` is `size`
-  `24(%rdi)` is `read_ptr`
-  `8(%rdi)` is `queue`.

And here is `read`:

{% highlight C++ %}
_ZN19AtomicCircularQueueI4elemE4readERS0_:
	movq	24(%rdi), %rdx
	leaq	32(%rdi), %rcx
	jmp	.L7
	.p2align 4,,10
	.p2align 3
.L8:
	rep nop
.L7:
	movq	(%rcx), %rax
	cmpq	%rax, %rdx
	je	.L8
	movq	8(%rdi), %rax
	movl	(%rax,%rdx,4), %eax
	addq	$1, %rdx
	cmpq	16(%rdi), %rdx
	movl	%eax, (%rsi)
	movl	$0, %eax
	cmove	%rax, %rdx
	movq	%rdx, 24(%rdi)
	ret
{% endhighlight %}

The strange instruction `rep nop` mustn't confuse the reader: its instruction code (`F3 90`) has been hijacked for `pause`.

One may wonder about the `rep ret` in the first sample. This is in fact more cryptic and is a workaround (or a hack, if you prefer this word) for some branch-prediction
misbehaviour in AMD processors. It is described in this blog post: [repz ret](http://repzret.org/p/repzret/).

The code looks very good if not absolutely perfect. One could wonder if conditional moves are justified there (a branch misprediction will only happen once in 100000 iterations),
but, even if a plain branch is better, the compiler has no way to establish it. The most striking visible feature of this code, however, is the absence of any
synchronisation instructions. No locks, atomic swaps, or fences. The volatile semantics is, however, provided (the `read_ptr` is read from memory on each iteration of the
loop in `read`), and the memory order is also observed: the pointer variables are stored as the last operations in the loops. The strong memory ordering of Intel takes
care of the rest.

If we replace `acomic<size_t>` with a simple `volatile size_t`, we get exactly the same code. No difference.

This is only true on Intel. Other processors (such as ARM) may have much weaker natural memory ordering, and some memory barrier instructions will be required after
reading or before writing the pointer variables.

Out of interest, let's try compiling the code without `atomic` or `volatile`:

{% highlight asm %}
_ZN19SimpleCircularQueueI4elemE5writeERKS0_:
	movq	32(%rdi), %rdx
	movl	$0, %ecx
	leaq	1(%rdx), %rax
	cmpq	16(%rdi), %rax
	cmovae	%rcx, %rax
	cmpq	%rax, 24(%rdi)
	je	.L13
	movl	(%rsi), %esi
	movq	8(%rdi), %rcx
	movl	%esi, (%rcx,%rdx,4)
	movq	%rax, 32(%rdi)
.L13:
	rep ret

_ZN19SimpleCircularQueueI4elemE4readERS0_:
	movq	24(%rdi), %rax
	cmpq	32(%rdi), %rax
	jne	.L29
	.p2align 4,,10
	.p2align 3
.L32:
	rep nop
	cmpq	32(%rdi), %rax
	je	.L32
.L29:
	movq	8(%rdi), %rdx
	movl	(%rdx,%rax,4), %edx
	addq	$1, %rax
	cmpq	16(%rdi), %rax
	movl	%edx, (%rsi)
	movl	$0, %edx
	cmove	%rdx, %rax
	movq	%rax, 24(%rdi)
	ret
{% endhighlight %}

The code looks a bit different but does the same thing. It fulfils volatile, atomic and memory order requirements just as well as the `atomic` and `volatile` versions
did. This is probably just a coincidence: for instance, the **C++** compiler didn't have to generate memory read in the loop in `read` (`.L32`). So this isn't a safe
option.

These code observations kill one possible improvement of the code that I previously had in mind. If reading atomic variables was expensive, one could consider caching
their safe approximations in normal variables. For instance, the `write` routine could store the value of `read_ptr` in a normal variable and perform no atomic reads
until this value is reached, and the `read` routine could do the same. All of this is not necessary when atomic operations are for free (the trick, however, might still be valid for
non-Intel processors).

Overcoming false sharing: aligning the variables
------------------------------------------------

[Previously]({{ site.ART-STATS }}) we faced the effects of [false sharing](https://en.wikipedia.org/wiki/False_sharing).
It is worthwhile checking any multi-threading solution for possible exposure to this problem.

A memory location may be cached in local caches of more than one processor. When one processor
(or one core) modifies it, a signal is sent to other processors to discard their cached values and rather fetch the new value from the new owner when needed.
This is completely reasonable. The problem is that the unit of such discarding and retrieval is a cache line, which on modern Intels is 64 bytes long.
One thread modifying some value causes another thread to re-fetch some other value, whose only fault is to share a cache line with the modified one. This is exactly what may happen
to our `read_ptr` and `write_ptr` values. When the reader updates its pointer, it invalidates the write pointer, and the writer has to fetch it again, and vice versa.
It seems attractive to place `read_ptr` and `wirte_ptr` into different cache lines. The simplest way to achieve this is to align the entire structure and the second one
of the pointers by 64:

{% highlight C++ %}
template<typename E> class AlignedAtomicCircularQueue : public Queue<E>
{
    E * const queue;
    const size_t size;
    std::atomic<size_t> read_ptr;
    alignas (CACHE_LINE_SIZE)
        std::atomic<size_t> write_ptr;
{% endhighlight %}

The results really improved:

<table class="numeric">
<tr><th>Queue name</th><th>Writing time, ns</th><th>Good interval, ns</th><th>Throughput, Mil/sec</th><th>Avg size</th></tr>
<tr><td class="label"> Aligned Atomic      </td><td>  12-23   </td><td> 30-41     </td><td> 24-33 </td><td> 7 </td></tr>
</table>

Aligning the variables even more
--------------------------------

We made sure `read_ptr` and `write_ptr` reside in different cache lines, but what about other variables? Both `queue` and `size` are also fields stored in memory.
As we saw in the assembly listing above, both are read at each invocation of `read()` and `write()`. Both are, therefore, vulnerable to false sharing with the
pointers (or, after the fix, only with the `read_ptr`). Let's align the `read_ptr` as well:

{% highlight C++ %}
template<typename E> class AlignedAtomicCircularQueue : public Queue<E>
{
    E * const queue;
    const size_t size;
    alignas (CACHE_LINE_SIZE)
        std::atomic<size_t> read_ptr;
    alignas (CACHE_LINE_SIZE)
        std::atomic<size_t> write_ptr;
{% endhighlight %}

Here are the results (strangely, no big improvement):

<table class="numeric">
<tr><th>Queue name</th><th>Writing time, ns</th><th>Good interval, ns</th><th>Throughput, Mil/sec</th><th>Avg size</th></tr>
<tr><td class="label"> Aligned More Atomic </td><td>  15-30   </td><td> 35-42     </td><td> 24-28 </td><td> 6 </td></tr>
</table>

Cached atomic circular queue
----------------------------

Let's now revisit one of the statements made above:

> These code observations kill one possible improvement of the code that I previously had in mind. If reading atomic variables was expensive, one could consider caching
> their safe approximation in normal variables.

As we saw, reading atomic variables isn't more expensive than reading normal variables: the code is identical. However, reading variables that are shared with another
thread is more expensive than reading local variables. The idea comes back to life:

{% highlight C++ %}
template<typename E> class CachedAtomicCircularQueue : public Queue<E>
{
    E * const queue;
    const size_t size;
    alignas (CACHE_LINE_SIZE)
        std::atomic<size_t> read_ptr;
    size_t cached_write_ptr;
    alignas (CACHE_LINE_SIZE)
        std::atomic<size_t> write_ptr;
    size_t cached_read_ptr;

public:
    CachedAtomicCircularQueue(size_t size) : queue(new E[size]), size(size),
                                             read_ptr(0), write_ptr(0),
                                             cached_read_ptr(0), 
                                             cached_write_ptr(0)
    {
    }

    ~CachedAtomicCircularQueue()
    {
        delete[] queue;
    }

    void write(const E& elem)
    {
        size_t w = write_ptr;
        size_t nw = w + 1;
        if (nw >= size) nw = 0;
        if (nw == cached_read_ptr) {
            cached_read_ptr = read_ptr.load(std::memory_order_consume);
            if (nw == cached_read_ptr) {
                return;
            }
        }
        queue[w] = elem;
        write_ptr.store(nw, std::memory_order_release);
    }

    void read(E& elem)
    {
        size_t r = read_ptr;
        if (r == cached_write_ptr) {
            size_t w;
            while (r == (w = write_ptr.load(std::memory_order_acquire))) {
                yield();
            }
            cached_write_ptr = w;
        }
        elem = queue[r];
        if (++r == size) r = 0;
        read_ptr.store(r, std::memory_order_release);
    }
};
{% endhighlight %}

This version reduces reading of shared variables to the cases when it is absolutely necessary to exchange information between threads. In our case this is
when the queue is full of empty. The improvement is visible straight away:

<table class="numeric">
<tr><th>Queue name</th><th>Writing time, ns</th><th>Good interval, ns</th><th>Throughput, Mil/sec</th><th>Avg size</th></tr>
<tr><td class="label"> Cached Atomic       </td><td>  6       </td><td> 27-32     </td><td> 31-37 </td><td> 2.2 </td></tr>
</table>

<p id="dual"> </p>

Dual-array queue
----------------

The cached circular queue, being almost ideal, still has two disadvantages:

- as any circular queue, it is not very cache-friendly. This is not so important for a queue of size 100,000 elements, as it fits into the L3 cache, and this is the only
cache shared between two different cores. Even then, the writer often has to write to the memory not cached in its L1 or L2 cache. While this is inevitable for the reader
(after all, it always reads the memory that has just been modified), this can be avoided for the writer. Much longer queues may present even bigger caching problems;

- even if the entire queue fits into a cache of some level, the cache isn't there just for the queue: the rest of the program also wants access to this resource, while
the circular nature of the queue frequently pushes useful data out of the cache;

- if the read and write pointers aren't too far from each other, we can experience false sharing between the data being written and the data being read.

Let's try a completely different approach for the queue design, based on the idea that the writer writes to some part of the queue while the reader reads some other part.
We'll make these parts of the queue two separate arrays, one being written and another one being read, the arrays swapped when the reading one is exhausted.

We'll follow a client-server approach: the only truly shared variable will be a swap request flag (`swap_requested`), set by the reader when it runs out of data.
Upon receiving such a request, the writer swaps the arrays and resets the flag. Note that the writer can only do this on its next write operation, so the current content of the queue can get stuck
there forever if the influx of source data stops. This is a shortcoming of this implementation.

{% highlight C++ %}
template<typename E> class DualArrayQueue : public Queue<E>
{
    const size_t size;
    E * volatile read_buf;
    E * volatile write_buf;

    alignas (CACHE_LINE_SIZE)
        size_t read_ptr;
    alignas (CACHE_LINE_SIZE)
        volatile size_t read_limit;
    alignas (CACHE_LINE_SIZE)
        std::atomic<bool> swap_requested;
    alignas (CACHE_LINE_SIZE)
        size_t write_ptr;

public:
    DualArrayQueue(size_t size) : size(size), read_buf(new E[size]),
                                  write_buf(new E[size]), read_ptr(0),
                                  read_limit(0), swap_requested(false),
                                  write_ptr(0)
    {
    }

    ~DualArrayQueue()
    {
        delete[] read_buf;
        delete[] write_buf;
    }

    void write(const E& elem)
    {
        E * t = write_buf;
        size_t w = write_ptr;
        if (w < size) {
            t[w++] = elem;
        }
        if (swap_requested.load(std::memory_order_acquire)) {
            E* r = read_buf;
            read_buf = t;
            read_limit = w;
            swap_requested.store(false, std::memory_order_release);
            write_buf = r;
            w = 0;
        }
        write_ptr = w;
    }

    void read(E& elem)
    {
        size_t r = read_ptr;
        if (r >= read_limit) {
            swap_requested.store(true, std::memory_order_release);
            while (swap_requested.load(std::memory_order_acquire)) {
                yield();
            }
            r = 0;
        }
        elem = read_buf[r];
        read_ptr = r + 1;
    }
};
{% endhighlight %}

Here we preferred to be safe and aligned every single variable by the cache line boundary. Probably, it is possible to ease these requirements. It is also possible that
the memory order requirements may be relaxed. I will not concentrate on these issues here.

Other strategies can be used for reader and writer co-operation. For example, if the queue is full at the call to `write()`, and the `swap_requested` flag is set,
we can, instead of dropping the element, write it to the new array. I don't think this is a big improvement: we could just make the array size bigger.

Another variation is using a `do while` loop in the reader to perform at least one `yield()` between writing the swap request flag and first reading it. I tried this, it
makes things a bit slower.

Again, the `atomic` operations do not show in the Intel code. The `volatile` keyword would be sufficient on this architecture.

How does this solution compare with the best so far (`CachedAtomicCircularQueue`)? This is not clear. Here are different comparison points:

<table>
<tr> <th> CachedAtomicCircularQueue </th><th> DualArrayQueue </th></tr>
<tr> <td> Cache-unfriendly</td><td> More cache-friendly, unless the queue grows too long </td></tr>
<tr> <td> Two atomic variables </td><td> One atomic variable </td></tr>
<tr> <td> The reader writes one each time </td><td> The reader writes it to signal empty queue </td></tr>
<tr> <td> The writer writes one each time </td><td> The writer writes it to respond </td></tr>
<tr> <td> The reader polls one when the queue is full </td><td> The reader polls it for response </td> </tr>
<tr> <td> The writer reads one when reaching the previous reading point </td><td> The writer reads it each time, although it is well-cached most of the time </td> </tr>
<tr> <td> Threads signal to each other by a single memory write </td><td> The reader starts a whole transaction,
                                                                          with an atomic variable being read and written twice,
                                                                          and two more variables written by the writer
                                                                          and read by the reader (although <code>read_limit</code> only at
                                                                          the next iteration)
                                                                 </td> </tr>
</table>

We have no option but testing, and here are the results:

<table class="numeric">
<tr><th>Queue name</th><th>Writing time, ns</th><th>Good interval, ns</th><th>Throughput, Mil/sec</th><th>Avg size</th></tr>
<tr><td class="label"> Dual                </td><td>  3       </td><td> 23        </td><td> 43.5  </td><td> 1 </td></tr>
</table>

Overall there is a small difference in favour of the dual-array solution. The average queue size looks especially impressive.

Dual-array queue improved
-------------------------

One small improvement of the dual-array queue is reducing the number of variables. We can hijack the `read_limit` as a new `swap_flag`. This variable being zero will
serve as the signal to the writer to provide a new chunk of data.

{% highlight C++ %}
template<typename E> class DualArrayQueue2 : public Queue<E>
{
    const size_t size;
    E * volatile read_buf;
    E * volatile write_buf;

    alignas (CACHE_LINE_SIZE)
        size_t read_ptr;
    alignas (CACHE_LINE_SIZE)
        std::atomic<size_t> read_limit;
    alignas (CACHE_LINE_SIZE)
        size_t write_ptr;

public:
    DualArrayQueue2(size_t size) : size(size), read_buf(new E[size]),
                                   write_buf(new E[size]),
                                   read_ptr(size), read_limit(size),
                                   write_ptr(0)
    {
    }

    ~DualArrayQueue2()
    {
        delete[] read_buf;
        delete[] write_buf;
    }

    void write(const E& elem)
    {
        E * t = write_buf;
        size_t w = write_ptr;
        if (w < size) {
            t[w++] = elem;
        }
        if (read_limit.load(std::memory_order_acquire) == 0) {
            E* r = read_buf;
            read_buf = t;
            read_limit.store(w, std::memory_order_release);
            write_buf = r;
            w = 0;
        }
        write_ptr = w;
    }

    void read(E& elem)
    {
        size_t r = read_ptr;
        if (r >= read_limit) {
            read_limit.store(0, std::memory_order_release);
            while (read_limit.load(std::memory_order_acquire) == 0) {
                yield();
            }
            r = 0;
        }
        elem = read_buf[r];
        read_ptr = r + 1;
    }
};
{% endhighlight %}

We know that atomic variables are not expensive on Intel, and that shared variables are fast when accessed by a single thread. This makes it possible for
`read_limit`, apart from being used as a request flag, to serve its direct purpose: indicate the size of the read buffer. On other platforms one could use
a thread-local copy of this variable, but we won't bother with it here.

Even if there are performance advantages, they aren't visible:

<table class="numeric">
<tr><th>Queue name</th><th>Writing time, ns</th><th>Good interval, ns</th><th>Throughput, Mil/sec</th><th>Avg size</th></tr>
<tr><td class="label"> Dual2               </td><td>  3       </td><td> 26        </td><td> 38    </td><td> 1 </td></tr>
</table>

We'll still keep this version as a lower-impact one.

Thread-local array swapping
---------------------------

In the code of `DualArrayQueue` the writer swaps the read and write buffers, and the reader uses the results of this swap. The reader, however, could have its own copy
of the read buffer and perform swapping locally:

{% highlight C++ %}
template<typename E> class DualArrayQueue3 : public Queue<E>
{
    const size_t size;
    E * const buf1;
    E * const buf2;

    alignas (CACHE_LINE_SIZE)
        size_t read_ptr;
    E * read_buf;

    alignas (CACHE_LINE_SIZE)
        size_t write_ptr;
    E * write_buf;

    alignas (CACHE_LINE_SIZE)
        std::atomic<size_t> read_limit;

public:
    DualArrayQueue3(size_t size) : size(size),
                                   buf1(new E[size]), buf2(new E[size]),
                                   read_ptr(size),
                                   write_ptr(0),
                                   read_limit(size)
    {
        read_buf = buf1;
        write_buf = buf2;
    }

    ~DualArrayQueue3()
    {
        delete[] buf1;
        delete[] buf2;
    }

    void write(const E& elem)
    {
        E * wb = write_buf;
        size_t w = write_ptr;
        if (w < size) {
            wb[w++] = elem;
        }
        if (read_limit.load(std::memory_order_acquire) == 0) {
            read_limit.store(w, std::memory_order_release);
            write_buf = wb == buf1 ? buf2 : buf1;
            w = 0;
        }
        write_ptr = w;
    }

    void read(E& elem)
    {
        size_t r = read_ptr;
        E* t = read_buf;
        if (r >= read_limit) {
            read_limit.store(0, std::memory_order_release);
            t = read_buf = t == buf1 ? buf2 : buf1;
            while (read_limit.load(std::memory_order_acquire) == 0) {
                yield();
            }
            r = 0;
        }
        elem = t[r];
        read_ptr = r + 1;
    }
};
{% endhighlight %}

The results are a little bit berrer:

<table class="numeric">
<tr><th>Queue name</th><th>Writing time, ns</th><th>Good interval, ns</th><th>Throughput, Mil/sec</th><th>Avg size</th></tr>
<tr><td class="label"> Dual3               </td><td>  2       </td><td> 24        </td><td> 41.6    </td><td> 1 </td></tr>
</table>


XOR-based array swapping
------------------------

We didn't use a proper swap (reading two variables and writing them other way around), because it would require two memory updates at every operation.
We used a conditional move instead. It requires two memory reads and a conditional move instruction, or a read, a branch and another read.  The GNU C++ compiler
chose the latter. It still worked fast, because the jump is very well predicted -- it is taken every other time, and processors know how to predict that. We can,
however, perform the swap without branches, using `XOR`:

{% highlight C++ %}
template<typename E> class DualArrayQueue4 : public Queue<E>
{
    const size_t size;
    E * const buf;
    std::uintptr_t diff;

    alignas (CACHE_LINE_SIZE)
        size_t read_ptr;
        const E * read_buf;

    alignas (CACHE_LINE_SIZE)
        size_t write_ptr;
        E * write_buf;

    alignas (CACHE_LINE_SIZE)
        std::atomic<size_t> read_limit;

public:
    DualArrayQueue4(size_t size) : size(size), buf(new E[2 * size]),
                                   read_ptr(size),
                                   write_ptr(0),
                                   read_limit(size)
    {
        read_buf = buf;
        write_buf = buf + size;
        diff = reinterpret_cast<uintptr_t> (read_buf)
             ^ reinterpret_cast<uintptr_t> (write_buf);
    }

    ~DualArrayQueue4()
    {
        delete[] buf;
    }

    void write(const E& elem)
    {
        E * wb = write_buf;
        size_t w = write_ptr;
        if (w < size) {
            wb[w++] = elem;
        }
        if (read_limit.load(std::memory_order_acquire) == 0) {
            read_limit.store(w, std::memory_order_release);
            write_buf = reinterpret_cast<E*>
                       (reinterpret_cast<uintptr_t>(wb) ^ diff);
            w = 0;
        }
        write_ptr = w;
    }

    void read(E& elem)
    {
        size_t r = read_ptr;
        const E* t = read_buf;
        if (r >= read_limit) {
            read_limit.store(0, std::memory_order_release);
            t = read_buf = reinterpret_cast<const E*>
                          (reinterpret_cast<uintptr_t>(t) ^ diff);

            while (read_limit.load(std::memory_order_acquire) == 0) {
                yield();
            }
            r = 0;
        }
        elem = t[r];
        read_ptr = r + 1;
    }
};
{% endhighlight %}

The result is a bit worse than before:

<table class="numeric">
<tr><th>Queue name</th><th>Writing time, ns</th><th>Good interval, ns</th><th>Throughput, Mil/sec</th><th>Avg size</th></tr>
<tr><td class="label"> Dual4               </td><td>  3       </td><td> 25        </td><td> 40    </td><td> 1 </td></tr>
</table>

The code, however, looks very compact and elegant:

{% highlight asm %}
_ZN15DualArrayQueue4I4elemE5writeERKS0_:
	movq	128(%rdi), %rax
	cmpq	8(%rdi), %rax
	movq	136(%rdi), %rcx
	jae	.L560
	movl	(%rsi), %edx
	movl	%edx, (%rcx,%rax,4)
	addq	$1, %rax
.L560:
	movq	192(%rdi), %rdx
	testq	%rdx, %rdx
	jne	.L561
	movq	%rax, 192(%rdi)
	xorq	24(%rdi), %rcx
	xorl	%eax, %eax
	movq	%rcx, 136(%rdi)
.L561:
	movq	%rax, 128(%rdi)
	ret

_ZN15DualArrayQueue4I4elemE4readERS0_:
	movq	64(%rdi), %rax
	movq	72(%rdi), %rcx
	leaq	192(%rdi), %rdx
	movq	192(%rdi), %r8
	cmpq	%r8, %rax
	jae	.L612
	leaq	0(,%rax,4), %rdx
	addq	$1, %rax
	movl	(%rcx,%rdx), %edx
	movl	%edx, (%rsi)
	movq	%rax, 64(%rdi)
	ret
.L612:
	movq	$0, 192(%rdi)
	xorq	24(%rdi), %rcx
	movq	%rcx, 72(%rdi)
	jmp	.L614
.L615:
	rep nop
.L614:
	movq	(%rdx), %rax
	testq	%rax, %rax
	je	.L615
	xorl	%edx, %edx
	movl	$1, %eax
	movl	(%rcx,%rdx), %edx
	movl	%edx, (%rsi)
	movq	%rax, 64(%rdi)
	ret
{% endhighlight %}

Dual-array queue: asynchronous approach
--------------------------------------

In all the dual-array versions so far the reader sent a message to the writer when it encountered an empty queue when entering `read()`. We can improve this by sending
this message at exit of the previous `read()`. This will allow message exchange to run in parallel with some useful activity in the reader.
It probably won't affect our artificial test, which has no such activity, but may help in real applications.

{% highlight C++ %}
template<typename E> class DualArrayAsyncQueue : public Queue<E>
{
    const size_t size;
    E * const buf;
    std::uintptr_t diff;

    alignas (CACHE_LINE_SIZE)
        size_t read_ptr;
    const E * read_buf;

    alignas (CACHE_LINE_SIZE)
        size_t write_ptr;
    E * write_buf;

    alignas (CACHE_LINE_SIZE)
        std::atomic<size_t> read_limit;

public:
    DualArrayAsyncQueue(size_t size) : size(size), buf(new E[size * 2]),
        read_ptr(0),
        write_ptr(0),
        read_limit(0)
    {
        read_buf = buf;
        write_buf = buf;
        diff = reinterpret_cast<uintptr_t> (buf)
             ^ reinterpret_cast<uintptr_t> (buf + size);
    }

    ~DualArrayAsyncQueue()
    {
        delete[] buf;
    }

    void write(const E& elem)
    {
        E * wb = write_buf;
        size_t w = write_ptr;
        if (w < size) {
            wb[w++] = elem;
        }
        if (read_limit.load(std::memory_order_acquire) == 0) {
            read_limit.store(w, std::memory_order_release);
            write_buf = reinterpret_cast<E*>
                       (reinterpret_cast<uintptr_t>(wb) ^ diff);
            w = 0;
        }
        write_ptr = w;
    }

    void read(E& elem)
    {
        size_t lim;

        while ((lim = read_limit.load(std::memory_order_acquire)) == 0) {
            yield();
        }
        size_t r = read_ptr;
        const E* t = read_buf;
        elem = t[r++];
        if (r == lim) {
            read_limit.store(0, std::memory_order_release);
            read_buf = reinterpret_cast<const E*>
                      (reinterpret_cast<uintptr_t>(t) ^ diff);
            r = 0;
        }
        read_ptr = r;
    }
};
{% endhighlight %}

As expected, the result hasn't become better, but it hasn't become worse, either:

<table class="numeric">
<tr><th>Queue name</th><th>Writing time, ns</th><th>Good interval, ns</th><th>Throughput, Mil/sec</th><th>Avg size</th></tr>
<tr><td class="label"> Dual Async          </td><td>  3       </td><td> 25        </td><td> 40    </td><td> 1 </td></tr>
</table>

The summary
-----------

This is where our story of improving the queue in **C++** finishes. We've gone quite far: 

<table class="numeric">
<tr><th>Queue name</th><th>Writing time, ns</th><th>Good interval, ns</th><th>Throughput, Mil/sec</th><th>Avg size</th></tr>
<tr><td class="label"> Standard            </td><td>  280-500 </td><td> 380-550   </td><td> 2.6   </td><td>  230 </td></tr>
<tr><td class="label"> Circular            </td><td>  364-395 </td><td> 430       </td><td> 2.3   </td><td>  290 </td></tr>
<tr><td class="label"> No wait             </td><td>  310     </td><td> 340       </td><td> 2.9   </td><td> 1030 </td></tr>
<tr><td class="label"> Atomic              </td><td>  23-25   </td><td> 42-46     </td><td> 21-23 </td><td> 26 </td></tr>
<tr><td class="label"> Aligned Atomic      </td><td>  12-23   </td><td> 30-41     </td><td> 24-33 </td><td> 7 </td></tr>
<tr><td class="label"> Aligned More Atomic </td><td>  15-30   </td><td> 35-42     </td><td> 24-28 </td><td> 6 </td></tr>
<tr><td class="label"> Cached Atomic       </td><td>  6       </td><td> 27-32     </td><td> 31-37 </td><td> 2.2 </td></tr>
<tr><td class="label"> Dual                </td><td>  3       </td><td> 23        </td><td> 43.5  </td><td> 1 </td></tr>
<tr><td class="label"> Dual2               </td><td>  3       </td><td> 26        </td><td> 38    </td><td> 1 </td></tr>
<tr><td class="label"> Dual3               </td><td>  2       </td><td> 24        </td><td> 41.6    </td><td> 1 </td></tr>
<tr><td class="label"> Dual4               </td><td>  3       </td><td> 25        </td><td> 40    </td><td> 1 </td></tr>
<tr><td class="label"> Dual Async          </td><td>  3       </td><td> 25        </td><td> 40    </td><td> 1 </td></tr>
</table>

Various versions of the dual-array queue are the definite winners. Even though the advantage over atomic versions isn't big, the queue size shows
that dual-array versions are more responsive and cause less latency.

We've been working with the queues of 100,000 elements long, each element being four bytes. Let's see what happens when we change these parameters.

Changing the element size
-------------------------

Let's try changing the element size to 64, with the same queue size (100K):

<table class="numeric">
<tr><th>Queue name</th><th>Writing time, ns</th><th>Good interval, ns</th><th>Throughput, Mil/sec</th><th>Avg size</th></tr>
<tr><td class="label"> Standard            </td><td>  360      </td><td> 470   </td><td> 2.1   </td><td> 830 </td></tr>
<tr><td class="label"> Circular            </td><td>  515      </td><td> 530   </td><td> 1.9   </td><td> 290 </td></tr>
<tr><td class="label"> No wait             </td><td>  430      </td><td> 430   </td><td> 2.3   </td><td> 89  </td></tr>
<tr><td class="label"> Atomic              </td><td>  80       </td><td> 63    </td><td> 15.9  </td><td> 544 </td></tr>
<tr><td class="label"> Aligned Atomic      </td><td>  59       </td><td> 59    </td><td> 16.9  </td><td> 5   </td></tr>
<tr><td class="label"> Aligned More Atomic </td><td>  61       </td><td> 61    </td><td> 16.4  </td><td> 2.6 </td></tr>
<tr><td class="label"> Cached Atomic       </td><td>  60       </td><td> 59    </td><td> 16.9  </td><td> 2.2 </td></tr>
<tr><td class="label"> Dual                </td><td>  26-43    </td><td> 52    </td><td> 19.2  </td><td> 1 </td></tr>
<tr><td class="label"> Dual2               </td><td>  56       </td><td> 56    </td><td> 17.9  </td><td> 1 </td></tr>
<tr><td class="label"> Dual3               </td><td>  32-47    </td><td> 47    </td><td> 21.2  </td><td> 1 </td></tr>
<tr><td class="label"> Dual4               </td><td>  60       </td><td> 45    </td><td> 22.2  </td><td> 1 </td></tr>
<tr><td class="label"> Dual Async          </td><td>  44       </td><td> 44    </td><td> 22.7  </td><td> 1 </td></tr>
</table>

Except for the versions using the OS-based synchronisation, everything went quite a bit slower. However, high throughputs are still available.
We see that the solutions that were the best last time are still the best.

Changing the queue size
-----------------------

I won't show all the results here. Here are the observations:

- the queue size of 100 or less simply doesn't work with any data rate of interest. Some of the atomic and dual-array solutions can still perform at
  1000 ns/packet or more. Everything else just fails.

- the standard, circular and no-wait solutions fail at the queue size of 1000. The atomic and dual-array solutions perform well at 100 ns and more.
  They lose packets at higher speed.

- the queue size of 10,000: occasional packet loss is observed at high speed on atomic solutions, but not on dual-array ones.

- going other direction does not show any difference. Even exceeding the L3 cache size (the queue size of 100M at the element size of 4 bytes)
  does not affect the achieved throughput on any of the solutions. However, one must remember that we didn't do anything else on the machine, so
  there were no useful data to cache.

In any case, the dual-array versions performed the best, followed by the atomic ones, with everything else as the outsiders.

Multi-processor case
--------------------

Studying the effects of true multi-processor environment was not our objective; however, just out of curiosity let's run some of our solutions using affinity
masks that put our data source and data sink onto different processors:

<table class="numeric">
<tr><th>Queue name</th><th>Writing time, ns</th><th>Good interval, ns</th></tr>
<tr><td class="label"> Atomic              </td><td>  5   </td><td> 160     </td></tr>
<tr><td class="label"> Aligned Atomic      </td><td>  5   </td><td> 160     </td></tr>
<tr><td class="label"> Aligned More Atomic </td><td>  4   </td><td> 140     </td></tr>
<tr><td class="label"> Cached Atomic       </td><td>  5   </td><td> 140     </td></tr>
<tr><td class="label"> Dual                </td><td>  3   </td><td> 140     </td></tr>
<tr><td class="label"> Dual2               </td><td>  3   </td><td> 140     </td></tr>
<tr><td class="label"> Dual3               </td><td>  3   </td><td> 140     </td></tr>
<tr><td class="label"> Dual4               </td><td>  3   </td><td> 135     </td></tr>
<tr><td class="label"> Dual Async          </td><td>  4   </td><td> 135     </td></tr>
</table>

All our solutions look poor; however, the dual-array ones are still the best. The typical times exceeding 100 ns indicate that some uncached memory access
takes place.

Let's increase the queue size to one million (it is not cached anyway), and see what happens:

<table class="numeric">
<tr><th>Queue name</th><th>Writing time, ns</th><th>Good interval, ns</th></tr>
<tr><td class="label"> Atomic              </td><td>  5   </td><td> 140     </td></tr>
<tr><td class="label"> Aligned More Atomic </td><td>  5   </td><td> 80      </td></tr>
<tr><td class="label"> Cached Atomic       </td><td>  3   </td><td> 24     </td></tr>
<tr><td class="label"> Dual Async          </td><td>  4   </td><td> 28     </td></tr>
</table>

The results are surprisingly good. It looks like the memory pre-fetch resolves the uncached memory access problem. What is important is the optimisation of the control
values access. The false-sharing reduction (`Aligned More Atomic`) helps a lot, and true-sharing reduction (`Cached Atomic`) helps even more.

The dual-array versions don't look too bad, either.

Still, the high queue size where this result has been achieved shows that the multi-processor setup is not good where low latency is required.

Going Java: dual-array version
------------------------------

Now it's time to test our solutions on **Java**. We will only port one version -- the last one (asynchronous dual-array). The overall test setup will be the same; however,
there are some important differences to keep in mind:

- **Java** does not have a built-in thread affinity control. Although it is possible to write a native-code library for that, we won't bother with it. Let's just run
the entire program on one  physical processor using `taskset` with appropriate parameters (`taskset 0x555` works on my system);

- NUMA considerations stay the same; we'll use the same NUMA control commands;

- **Java** does not have direct access to `rdtsc` instruction. This means we can't implement a precise `hires_timer`. Instead, we'll be running some arithmetic calculations
in a loop to get some controllable delay. Thus the delay value won't be in nanoseconds; however, reported values will still be.

- **Java** does not have any control over field alignment inside an object; we'll have to introduce our own padding where necessary. This is, obviously, not portable --
**Java** does not make any promises regarding the object layout.

Let's now look at the first version, the `DualArrayAsyncIntQueue`:

{% highlight Java %}
public class DualArrayAsyncIntQueue extends IntQueue
{
    private final int size;
    private final int [] buf1;
    private final int [] buf2;
    int x1, x2, x3, x4, x5, x6, x7, x8, x9, x10, x11;
    int [] read_buf;
    int read_ptr;
    int y1, y2, y3, y4, y5, y6, y7, y8, y9, y10, y11, y12;
    int [] write_buf;
    int write_ptr;
    int z1, z2, z3, z4, z5, z6, z7, z8, z9, z10, z11, z12;
    volatile int read_limit = 0;
    
    public DualArrayAsyncIntQueue (int size)
    {
        this.size = size;
        buf1 = new int [size];
        buf2 = new int [size];
        read_buf = write_buf = buf1;
        read_ptr = write_ptr = 0;
        read_limit = 0;
    }
    
    @Override
    public int read ()
    {
        int lim;

        while ((lim = read_limit) == 0) {
            Thread.yield();
        }
        int r = read_ptr;
        int [] rb = read_buf;
        int elem = rb[r++];
        if (r == lim) {
            read_limit = 0;
            read_buf = rb == buf1 ? buf2 : buf1;
            r = 0;
        }
        read_ptr = r;
        return elem;
    }
    
    @Override
    public void write (int value)
    {
        int [] wb = write_buf;
        int w = write_ptr;
        if (w < size) {
            wb[w++] = value;
        }
        if (read_limit == 0) {
            read_limit = w;
            write_buf = wb == buf1 ? buf2 : buf1;
            w = 0;
        }
        write_ptr = w;
    }
}
{% endhighlight %}

The code for `DataSource`, `DataSink` and `Reporter` classes can be seen in [the repository]({{ site.REPO-QUEUE}}/tree/master/java).

Here are the results:

<table class="numeric">
<tr><th>Queue name</th><th>Writing time, ns</th><th>Good interval, ns</th><th>Throughput, Mil/sec</th><th>Avg size</th></tr>
<tr><td class="label"> DualArrayAsyncIntQueue </td><td>  6       </td><td> 16        </td><td> 62.5    </td><td> 1 </td></tr>
</table>

Surprisingly, the throughput is quite a bit higher than in **C++**.

Going Java: dual-buffer version
--------------------------------

This version isn't really there to be used in **Java**. We're going to use it to connect **Java** and **C++**. The **Java** array isn't the best object to be accessed via
JNI; the native buffer is much better. That's why we'll put our data elements into a native buffer, and a field that's going to be shared with **C++** (`read_limit`)
into another native buffer. Let's, however, first implement both ends in **Java**:

{% highlight Java %}
public class ByteBufferAsyncIntQueue extends IntQueue
{
    final int size;
    final IntBuffer buf;
    final IntBuffer read_limit_buf;
    private int read_offset = 0;
    private int write_offset = 0;
    private int read_ptr = 0;
    private int write_ptr = 0;
    private int read_limit = 0;
    
    public ByteBufferAsyncIntQueue (int size)
    {
        this.size = size;
        buf = ByteBuffer.allocateDirect (size * 2 * 4)
                        .order (ByteOrder.nativeOrder ())
                        .asIntBuffer ();
        read_limit_buf = ByteBuffer.allocateDirect (4)
                                   .order (ByteOrder.nativeOrder ())
                                   .asIntBuffer ();
        read_ptr = write_ptr = 0;
        read_limit = 0;
    }
    
    @Override
    public int read ()
    {
        if (read_limit == 0) {
            while ((read_limit = read_limit_buf.get (0)) == 0) {
                Thread.yield();
            }
        }
        int r = read_ptr;
        int elem = buf.get (read_offset + r++);
        if (r == read_limit) {
            read_limit_buf.put (0, 0);
            read_limit = 0;
            read_offset ^= size;
            r = 0;
        }
        read_ptr = r;
        return elem;
    }
    
    @Override
    public void write (int value)
    {
        int w = write_ptr;
        if (w < size) {
            buf.put (write_offset + w++, value);
        }
        if (read_limit_buf.get (0) == 0) {
            read_limit_buf.put (0, w);
            write_offset ^= size;
            w = 0;
        }
        write_ptr = w;
    }
}
{% endhighlight %}

Here we don't bother aligning any data elements, because this queue won't be used in a **Java** to **Java** setup, and no fields will be shared with other threads
when talking to the **C++** code. 

One point in this code isn't completely up to standard: where in **C++** we used atomic variables, here we just write data into the native buffers. This causes an
appropriate memory ordering on Intel; on other processors it might not work. There are two possible workarounds:

 - use `sun.misc.Unsafe`, which has volatile write and fence operations. However, using this class in a client code is these days considered not *comme il faut*;

 - just write something into any dummy `volatile` variable; this will probably work but looks like a hack.

We'll ignore this problem for now.

Here are the results:

<table class="numeric">
<tr><th>Queue name</th><th>Writing time, ns</th><th>Good interval, ns</th><th>Throughput, Mil/sec</th><th>Avg size</th></tr>
<tr><td class="label"> ByteBufferAsyncIntQueue </td><td>  9       </td><td> 25        </td><td> 40    </td><td> 1 </td></tr>
</table>

As expected, the speed dropped a bit, still staying better than our best **C++** version.

Connecting Java to C++: a native data source
------------------------------------------

We finally arrived at our last topic: a queue that connects **Java** and **C++**. Our last **Java** version allows easy integration with native code. First, we'll
need a **Java** class with native functions:

{% highlight Java %}
public class NativeDataSource
{
    private final ByteBufferAsyncIntQueue queue;
    private final int interval;

    static
    {
        System.loadLibrary ("NativeDataSource");
    }

    public NativeDataSource (IntQueue queue, int interval)
    {
        this.queue = (ByteBufferAsyncIntQueue) queue;
        this.interval = interval;
    }
    
    public void start ()
    {
        start (queue.buf, queue.size, queue.read_limit_buf, interval);
    }
    
    private static native void start
            (IntBuffer queue, int size, IntBuffer read_limit, int interval);
}
{% endhighlight %}

Then, a native implementation of the data source:

{% highlight C++ %}
template<class Timer> class DataSource
{
    DualArrayAsyncWriter<uint32_t> writer;
    Timer timer;

public:
    DataSource (uint32_t * buf, size_t size,
                std::atomic<uint32_t> & read_limit,
                int interval_ns)
    : writer(buf, size, read_limit), timer(interval_ns)
    {
    }

    void operator()()
    {
        int32_t seq = 0;
        timer.start();
        for (uint64_t i = 0; ; i++) {
            timer.iteration(i);
            writer.write(seq++);
        }
    }
};

JNIEXPORT void JNICALL Java_NativeDataSource_start
          (JNIEnv * env, jclass, jobject jbuf,
           jint size, jobject jread_limit_buf, jint interval)
{
    std::atomic<uint32_t> * read_limit_ptr =
         (std::atomic<uint32_t> *)env->GetDirectBufferAddress(jread_limit_buf);
    uint32_t * buf = (uint32_t*)env->GetDirectBufferAddress(jbuf);
    if (interval == 0)
        new std::thread(*new DataSource<empty_timer>
                             (buf, (size_t)size, *read_limit_ptr, interval));
    else
        new std::thread(*new DataSource<hires_timer>
                             (buf, (size_t)size, *read_limit_ptr, interval));
}
{% endhighlight %}

(a tricky part here is the conversion of an arbitrary memory pointer to a pointer to `atomic<uint32_t>`. Nothing in the `C++` standard promises this will work;
however, this looks very natural and works on Windows and Linux).

Finally, we need the queue itself (only the writing part of it):

{% highlight C++ %}
template<typename E> class DualArrayAsyncWriter
{
    E * const buf;
    const size_t size;
    std::atomic <uint32_t> & read_limit;
    size_t write_ptr;
    E * write_buf;
    std::uintptr_t diff;

public:
    DualArrayAsyncWriter (E * buf, size_t size,
                          std::atomic <uint32_t> & read_limit)
        : buf (buf), size(size),
          read_limit (read_limit),  write_ptr(0)
    {
        write_buf = buf;
        diff = reinterpret_cast<uintptr_t> (buf) ^
               reinterpret_cast<uintptr_t> (buf + size);
    }

    void write(E elem)
    {
        E * wb = write_buf;
        size_t w = write_ptr;
        if (w < size) {
            wb[w++] = elem;
        }
        if (read_limit.load(std::memory_order_acquire) == 0) {
            read_limit.store((uint32_t) w, std::memory_order_release);
            write_buf = reinterpret_cast<E*>
                       (reinterpret_cast<uintptr_t>(wb) ^ diff);
            w = 0;
        }
        write_ptr = w;
    }
};
{% endhighlight %}

Here are the results achieved when sending data from **C++** to **Java** (still, on one physical processor):

<table class="numeric">
<tr><th>Queue name</th><th>Writing time, ns</th><th>Good interval, ns</th><th>Throughput, Mil/sec</th><th>Avg size</th></tr>
<tr><td class="label"> Native ByteBufferAsyncIntQueue </td><td>  1       </td><td> 20        </td><td> 50    </td><td> 1 </td></tr>
</table>

This is really a very good result. It's better than anything we've achieved when both sides of the queue were in **C++**. This mechanism can be used as a clock 
generator with very fine resolution.

Conclusions
-----------

- We've tried multiple strategies to arrange a single-producer, single-consumer queue. Even the simplest implementation, using the standard tools, could sustain
  the message rate of 2.1 million per second, which is already more than good enough for any coarse parallelism requirements;

- We've improved the throughput by the factor of ten (up to 22 million), which took us to the area of the fine parallelism;

- This happened at a cost of burning CPU cycles in spin loops;

- The OS mechanisms for thread synchronisation and inter-thread communication are relatively slow; high-performance applications should rather use
some other, lower-level, tools, such as atomic variables and test-and-set instructions;

- The data structures designed for multi-threaded access perform much better than generic structures wrapped into mutexes;

- The dual-array design of a queue works very well and demonstrates better performance than the one based on circular arrays;

- Surprisingly, **Java** showed better performance than **C++**

- We've implemented very fast (50M messages/sec) queue between **C++** and **Java**. This creates a good framework to study the real-time behaviour of **Java**.
