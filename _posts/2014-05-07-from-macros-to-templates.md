---
layout: post
title:  "From macros to templates"
date:   2014-05-07 00:00:00
tags: C C++ meta-programming templates
---

In [one of the previous articles](http://pzemtsov.github.io/2014/05/01/demultiplexing-of-e1-converting-to-C.html)
I was using **C** macros to implement deep loop unrolling. I also mentioned **C++** templates and stated that it
was possible to use those for the same purpose, but that the code would look complex and ugly. Now I want to test this statement.

Simple unrolled methods
-----------------------

Let's look at the first piece of unrolled code using macros, namely `Unrolled_1`
([you can see it in a repository](https://github.com/pzemtsov/article-E1-demux-C/commit/b98c060e8f01ef58b5ae6bb382b3ae1213333d8c)):

{% highlight C++ %}
#define MOVE_BYTE(i,j) d[i] = src [(j)+(i)*32]

#define MOVE_BYTES_64(j) DUP2_64 (MOVE_BYTE,j)

#define MOVE_TIMESLOT(j) do {\
        byte * const d = dst[j];\
        MOVE_BYTES_64 (j);\
    } while (0)

class Unrolled_1 : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        assert (NUM_TIMESLOTS == 32);
        assert (DST_SIZE == 64);
        assert (src_length == NUM_TIMESLOTS * DST_SIZE);

        for (unsigned j = 0; j < NUM_TIMESLOTS; j++) {
            MOVE_TIMESLOT (j);
        }
    }
};
{% endhighlight %}

Using templates, we can write it like this:

{% highlight C++ %}
template<unsigned N> inline void move_bytes (const byte * src, byte * d, unsigned j)
{
    move_bytes<N-1> (src, d, j);
    d[N-1] = src [j + (N-1) * 32];
}

template<> inline void move_bytes<0> (const byte * src, byte * d, unsigned j) {}

inline void move_timeslot (const byte * src, byte ** dst, unsigned j)
{
    move_bytes<DST_SIZE> (src, dst [j], j);
}

class Unrolled_1 : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        assert (src_length == NUM_TIMESLOTS * DST_SIZE);

        for (unsigned j = 0; j < NUM_TIMESLOTS; j++) {
            move_timeslot (src, dst, j);
        }
    }
};
{% endhighlight %}

Some comments on this:

- Each `move_bytes<N>` is a separate function, with its own generated code; each function gets instantiated when used
  somewhere at compile time. In our case each `move_bytes<N>` causes instantiation of `move_bytes<N-1>`.
  Here the parameter `N` indicated the number of bytes to be moved.

- This chain of instantiations can go for ever, eventually causing stack overflow in the template processor, so
  a terminating definition is required. The syntax `template<> inline void move_bytes<0>` means that the definition of
  `move_bytes<0>` does not follow the general pattern. It is implemented as an empty function, which is reasonable
  for a function that moves zero bytes.

- There is no need for separate `MOVE_BYTE` function or macro any more: its code can be easily placed inside `move_bytes<N>`.

- Hopefully the functions will be inlined into each other (I asked for that by using the `inline` keyword, but the
  compiler can do some ilininng even without such a request), and in the end we'll get an unrolled loop.

- Macros can access any variables from the context they are invoked; functions can't. That's why the entire
  context these functions operate one must be passed to them as parameters

- Unlike macros, the template functions are truly generic and do not depend on exact values of variables.
  In our case we could even use the `DST_SIZE` (our tuning parameter) as a byte count. With macros we could
  construct macro name dynamically (using the `'#'` operator), but all the required macros must exist already.

- The templated version works with any values of `DST_SIZE` and `NUM_TIMESLOTS`, so two of the three asserts
  can be removed.

- The functional languages enthusiasts will probably find it exciting. I still prefer the macro solution as the
  more straightforward one.

- However I must agree that it does not look as ugly as I though it would, so my original statement is probably not
  correct.

Deeper unrolling
----------------

This is what fully unrolled version looked like using macros:

{% highlight C++ %}
class Unrolled_2_Full : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        assert (NUM_TIMESLOTS == 32);
        assert (DST_SIZE == 64);
        assert (src_length == NUM_TIMESLOTS * DST_SIZE);

        DUP_32 (MOVE_TIMESLOT);
    }
};
{% endhighlight %}

We have `move_timeslot` function already, now we must provide a way to call it repeatedly. We'll do it in exactly
the same way we did `move_bytes`:

{% highlight C++ %}
template<unsigned N> inline void move_timeslots (const byte * src, byte ** dst, unsigned j)
{
    move_timeslots<N-1> (src, dst, j);
    move_timeslot (src, dst, j+N-1);
}

template<> inline void move_timeslots<0> (const byte * src, byte ** dst, unsigned j) {}

class Unrolled_2_Full : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        assert (src_length == NUM_TIMESLOTS * DST_SIZE);
        move_timeslots<NUM_TIMESLOTS> (src, dst, 0);
    }
};
{% endhighlight %}

Parameter `j` defines the first timeslot to be moved. It's been added to make `move_timeslots<N>()` suitable for
partialy unrolled versions, too:

{% highlight C++ %}
class Unrolled_1_4 : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        assert (src_length == NUM_TIMESLOTS * DST_SIZE);
        for (unsigned j = 0; j < NUM_TIMESLOTS; j+=4) {
            move_timeslots<4> (src, dst, j);
        }
    }
};
{% endhighlight %}

We can go further if we observe that the partially unrolled versions are now using the unroll factor (which is 4
for `Unroll_1_4` directly rather that as part of a macro name, and this factor is the only difference between them.
It makes it possible to template the whole function:

{% highlight C++ %}
template<unsigned F> class Unrolled_1_F : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        assert (src_length == NUM_TIMESLOTS * DST_SIZE);

        for (unsigned j = 0; j < NUM_TIMESLOTS; j+=F) {
            move_timeslots<F> (src, dst, j);
        }
    }
};
{% endhighlight %}

Later, instead of using classes `Unrolled_1_2`, `Unrolled_1_4`, etc., we can instantiate templates: `Unrolled_1_F<2>`,
`Unrolled_1_F<4>`, etc. Just be prepared to see funny class names as results of `typeid()` call.

Class `Unrolled_2_Full` can be constructed as `Unrolled_1_F<32>`, but it looks clearer as it is.

Other versions
--------------

The remaining two versions (`Unrolled_3` and `Unrolled_4`) both involve repeatitive calling of functions from
`demux()`. The difference was that in `Unrolled_4` it was the same function called 32 times, while in `Unrolled_3` there were
32 different specialised functions. Using templates, we can do both: we can create service templated functions
to implement loops, and we can create 32 instantiations of templated function. We can make all the servce templates
private members of a class, so there won't be any need to `#undef` them after use. But it is easy to show that in
our specific case all of that is unnecessary.

Here is the macro version of `Unrolled_4`:

{% highlight C++ %}
class Unrolled_4 : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        assert (NUM_TIMESLOTS == 32);
        assert (DST_SIZE == 64);
        assert (src_length == NUM_TIMESLOTS * DST_SIZE);

#define DEMUX(i) demux_0 (src, dst[i], i)

        DUP_32 (DEMUX);

#undef DEMUX
    }

private:
    inline void demux_0 (const byte * src, byte * d, unsigned i) const
    {
        MOVE_BYTES_64 (i);
    }
};
{% endhighlight %}

The `demux_0()` here contains a fully unrolled loop that copies 64 bytes. The way to write such a loop in the templated
implementation will be to call `move_bytes<64>()`. But there is no point creating and calling a special function
`demux_0()`, whose sole purpose is calling another function. Instead, we can call that other function directly,
and that is exactly what `Unrolled_2_Full` does.

Similar observation is applicable to `Unrolled_3`, except there we have 32 functions, each of which just calls
`move_bytes`.

In short, both `Unrolled_3` and `Unrolled_4` are unnecessary and can be removed from our **C++** code. In fact,
they can be removed from the macro-based code as well. The only reason they were created in the first place was that
in **Java** the compiler panicked at long functions.

I've put the templated code into separate file, `e1-template.cpp`.
[The code is available in the repository](https://github.com/pzemtsov/article-E1-demux-C/commit/561026d395eb57962a4a5167939ae2d39477e3a1).

Running it
----------

Now we are ready to compile and run our templated code. But first let's recall the results for the macro-based code:

    $ ./e1
    9Reference: 1939
    11Src_First_1: 1885
    11Src_First_2: 1924
    11Src_First_3: 1892
    11Dst_First_1: 1467
    11Dst_First_2: 1445
    11Dst_First_3: 1761
    10Unrolled_1: 633
    12Unrolled_1_2: 634
    12Unrolled_1_4: 654
    12Unrolled_1_8: 650
    13Unrolled_1_16: 635
    15Unrolled_2_Full: 635
    10Unrolled_3: 635
    10Unrolled_4: 655

    $ c++ -O3 -o e1-template e1-template.cpp -lrt
    $ ./e1-template
    9Reference: 1940
    10Unrolled_1: 634
    12Unrolled_1_FILj2EE: 633
    12Unrolled_1_FILj4EE: 653
    12Unrolled_1_FILj8EE: 650
    12Unrolled_1_FILj16EE: 646
    15Unrolled_2_Full: 655

We see is that the templated versions are fast. `Unrolled_1`, `Unrolled_2`, `Unrolled_4` and
`Unrolled_8` execute in exactly the same time as the macro-based versions. The `Unrolled_1_16` and `Unrolled_2_Full`
slow down a bit. This is most likely caused by the function inline limit and can possibly be improved by using
appropriate compiler options. We won't do it here, because `Unrolled_1` is already fast enough.

The assembly output (which one can get by invoking the compiler with `-S` option) for `Unrolled_1` is nearly identical
for macro and templated versions. Here it is for curious readers (without the assertion failure handler):

{% highlight cpp-objdump %}
_ZNK10Unrolled_15demuxEPKhjPPh:
.LFB1013:
        .cfi_startproc
        subq        $8, %rsp
        .cfi_def_cfa_offset 16
        cmpl        $2048, %edx
        jne        .L13
        xorl        %edx, %edx
        .p2align 4,,10
        .p2align 3
.L11:
        movzbl        (%rsi), %edi
        movq        (%rcx,%rdx), %rax
        addq        $8, %rdx
        movb        %dil, (%rax)
        movzbl        32(%rsi), %edi
        movb        %dil, 1(%rax)
        movzbl        64(%rsi), %edi
        movb        %dil, 2(%rax)
        movzbl        96(%rsi), %edi
        movb        %dil, 3(%rax)
        movzbl        128(%rsi), %edi
        movb        %dil, 4(%rax)
        movzbl        160(%rsi), %edi
        movb        %dil, 5(%rax)
        movzbl        192(%rsi), %edi
        movb        %dil, 6(%rax)
        movzbl        224(%rsi), %edi
        movb        %dil, 7(%rax)
        movzbl        256(%rsi), %edi
        movb        %dil, 8(%rax)
        movzbl        288(%rsi), %edi
        movb        %dil, 9(%rax)
        movzbl        320(%rsi), %edi
        movb        %dil, 10(%rax)
        movzbl        352(%rsi), %edi
        movb        %dil, 11(%rax)
        movzbl        384(%rsi), %edi
        movb        %dil, 12(%rax)
        movzbl        416(%rsi), %edi
        movb        %dil, 13(%rax)
        movzbl        448(%rsi), %edi
        movb        %dil, 14(%rax)
        movzbl        480(%rsi), %edi
        movb        %dil, 15(%rax)
        movzbl        512(%rsi), %edi
        movb        %dil, 16(%rax)
        movzbl        544(%rsi), %edi
        movb        %dil, 17(%rax)
        movzbl        576(%rsi), %edi
        movb        %dil, 18(%rax)
        movzbl        608(%rsi), %edi
        movb        %dil, 19(%rax)
        movzbl        640(%rsi), %edi
        movb        %dil, 20(%rax)
        movzbl        672(%rsi), %edi
        movb        %dil, 21(%rax)
        movzbl        704(%rsi), %edi
        movb        %dil, 22(%rax)
        movzbl        736(%rsi), %edi
        movb        %dil, 23(%rax)
        movzbl        768(%rsi), %edi
        movb        %dil, 24(%rax)
        movzbl        800(%rsi), %edi
        movb        %dil, 25(%rax)
        movzbl        832(%rsi), %edi
        movb        %dil, 26(%rax)
        movzbl        864(%rsi), %edi
        movb        %dil, 27(%rax)
        movzbl        896(%rsi), %edi
        movb        %dil, 28(%rax)
        movzbl        928(%rsi), %edi
        movb        %dil, 29(%rax)
        movzbl        960(%rsi), %edi
        movb        %dil, 30(%rax)
        movzbl        992(%rsi), %edi
        movb        %dil, 31(%rax)
        movzbl        1024(%rsi), %edi
        movb        %dil, 32(%rax)
        movzbl        1056(%rsi), %edi
        movb        %dil, 33(%rax)
        movzbl        1088(%rsi), %edi
        movb        %dil, 34(%rax)
        movzbl        1120(%rsi), %edi
        movb        %dil, 35(%rax)
        movzbl        1152(%rsi), %edi
        movb        %dil, 36(%rax)
        movzbl        1184(%rsi), %edi
        movb        %dil, 37(%rax)
        movzbl        1216(%rsi), %edi
        movb        %dil, 38(%rax)
        movzbl        1248(%rsi), %edi
        movb        %dil, 39(%rax)
        movzbl        1280(%rsi), %edi
        movb        %dil, 40(%rax)
        movzbl        1312(%rsi), %edi
        movb        %dil, 41(%rax)
        movzbl        1344(%rsi), %edi
        movb        %dil, 42(%rax)
        movzbl        1376(%rsi), %edi
        movb        %dil, 43(%rax)
        movzbl        1408(%rsi), %edi
        movb        %dil, 44(%rax)
        movzbl        1440(%rsi), %edi
        movb        %dil, 45(%rax)
        movzbl        1472(%rsi), %edi
        movb        %dil, 46(%rax)
        movzbl        1504(%rsi), %edi
        movb        %dil, 47(%rax)
        movzbl        1536(%rsi), %edi
        movb        %dil, 48(%rax)
        movzbl        1568(%rsi), %edi
        movb        %dil, 49(%rax)
        movzbl        1600(%rsi), %edi
        movb        %dil, 50(%rax)
        movzbl        1632(%rsi), %edi
        movb        %dil, 51(%rax)
        movzbl        1664(%rsi), %edi
        movb        %dil, 52(%rax)
        movzbl        1696(%rsi), %edi
        movb        %dil, 53(%rax)
        movzbl        1728(%rsi), %edi
        movb        %dil, 54(%rax)
        movzbl        1760(%rsi), %edi
        movb        %dil, 55(%rax)
        movzbl        1792(%rsi), %edi
        movb        %dil, 56(%rax)
        movzbl        1824(%rsi), %edi
        movb        %dil, 57(%rax)
        movzbl        1856(%rsi), %edi
        movb        %dil, 58(%rax)
        movzbl        1888(%rsi), %edi
        movb        %dil, 59(%rax)
        movzbl        1920(%rsi), %edi
        movb        %dil, 60(%rax)
        movzbl        1952(%rsi), %edi
        movb        %dil, 61(%rax)
        movzbl        1984(%rsi), %edi
        movb        %dil, 62(%rax)
        movzbl        2016(%rsi), %edi
        addq        $1, %rsi
        cmpq        $256, %rdx
        movb        %dil, 63(%rax)
        jne        .L11
        addq        $8, %rsp
        .cfi_remember_state
        .cfi_def_cfa_offset 8
        ret
{% endhighlight %}

Conclusions
-----------

Using templates for meta-programming is not that bad. Effectively we are replacing loops with recursion,
which is not the most intuitive thing to do. However, resulting code is quite small, more flexible and,
most important, it is efficient. A powerful function inlining engine that is built into the C++ compiler makes
the generated machine code identical to that of macro version.

