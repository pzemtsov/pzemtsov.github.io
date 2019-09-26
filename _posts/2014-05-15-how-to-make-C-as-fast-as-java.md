---
layout: post
title:  How to make C run as fast as Java
date:   2014-05-15 12:00:00
tags: C C++ Java GCC optimisation
story: e1-demux
story-title: "De-multiplexing of E1 streams"
ART-UNSTABLE: /2014/05/12/mystery-of-unstable-performance.html
---

No, there isn't a mistake in the title. Everyone expects that **C** programs always run faster than **Java**
programs, but in the recent articles (["{{ site.TITLE-E1-C }}"]({{ site.ART-E1-C }})
and ["The mystery of an unstable performance"]({{ page.ART-UNSTABLE }}))
we came across the example of the opposite.

These are the versions in question and the performance data, as published at the end of
[the last article]({{ page.ART-UNSTABLE }}):

<table class="numeric">
<tr><th>     Method               </th> <th>  Results in Java </th> <th>  Results in C  </th></tr>
<tr><td class="label">Dst_First_1 </td> <td> 1155 </td><td> 1457 </td></tr>
<tr><td class="label">Dst_First_2 </td> <td> 2093 </td><td> 1454 </td></tr>
<tr><td class="label">Dst_First_3 </td> <td> 1022 </td><td> 1464 </td></tr>
</table>

We can see that `Dst_First_1` and `Dst_First_3` are faster in **Java** than in **C**. There are other irregularities
as well: in **Java** `Dst_First_3` is quite a bit faster than `Dst_First_1`, while in **C** it is a bit slower.
In **Java** `Dst_First_2` is much slower while in **C** it runs at the same speed as `Dst_First_1`. Why is this?

Looking at the code: pointer aliasing
-------------------------------------

Let's recall what these three versions of E1 de-multiplexors are:

- `Dst_First_1`: the simplest, unoptimised version;

- `Dst_First_2`: the result of manual optimisation of the `Dst_First_1`. Specifically, loop invariant code motion
   and operation strength reduction were performed.

- `Dst_First_3`: the version of `Dst_First_1` with a hard-coded size of input and output buffers.

[Here is the code for all three versions]({{ site.REPO-E1-C }}/commit/ed29557be68e178600e7f2139330bcbdd724fe9f):

{% highlight C++ %}
class Dst_First_1 : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        assert (src_length % NUM_TIMESLOTS == 0);

        for (unsigned dst_num = 0; dst_num < NUM_TIMESLOTS; ++ dst_num) {
            for (unsigned dst_pos = 0; dst_pos < src_length / NUM_TIMESLOTS; ++ dst_pos) {
                dst [dst_num][dst_pos] = src [dst_pos * NUM_TIMESLOTS + dst_num];
            }
        }
    }
};

class Dst_First_2 : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        assert (src_length % NUM_TIMESLOTS == 0);

        unsigned dst_size = src_length / NUM_TIMESLOTS;
        for (unsigned dst_num = 0; dst_num < NUM_TIMESLOTS; ++ dst_num) {
            byte * d = dst [dst_num];
            unsigned src_pos = dst_num;
            for (unsigned dst_pos = 0; dst_pos < dst_size; ++ dst_pos) {
                d[dst_pos] = src[src_pos];
                src_pos += NUM_TIMESLOTS;
            }
        }
    }
};

class Dst_First_3 : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        assert (src_length == NUM_TIMESLOTS * DST_SIZE);

        for (unsigned dst_num = 0; dst_num < NUM_TIMESLOTS; ++ dst_num) {
            for (unsigned dst_pos = 0; dst_pos < DST_SIZE; ++ dst_pos) {
                dst [dst_num][dst_pos] = src [dst_pos * NUM_TIMESLOTS + dst_num];
            }
        }
    }
};
{% endhighlight %}

Let's compile the program with all necessary flags

    c++ -S -O3 -falign-functions=32 -falign-loops=32 -o e1.asm e1.cpp

and look at the [assembly]({{ site.REPO-E1-C }}/commit/d8d7ca6e71ae558e09098742e51b9c8963d12f15):

{% highlight c-objdump %}
_ZNK11Dst_First_15demuxEPKhjPPh:
.LFB1013:
        .cfi_startproc
        subq    $8, %rsp
        .cfi_def_cfa_offset 16
        testb   $31, %dl
        jne     .L46
        shrl    $5, %edx
        xorl    %r10d, %r10d
        .p2align 5
.L47:
        xorl    %eax, %eax
        testl   %edx, %edx
        movl    %r10d, %edi
        je      .L50
        .p2align 5
.L52:
        movl    %edi, %r8d
        addl    $32, %edi
        movzbl  (%rsi,%r8), %r9d
        movq    (%rcx,%r10,8), %r8
        movb    %r9b, (%r8,%rax)
        addq    $1, %rax
        cmpl    %eax, %edx
        ja      .L52
.L50:
        addq    $1, %r10
        cmpq    $32, %r10
        jne     .L47
        addq    $8, %rsp
        .cfi_remember_state
        .cfi_def_cfa_offset 8
        ret
.L46:
        .cfi_restore_state
        movl    $_ZZNK11Dst_First_15demuxEPKhjPPhE19__PRETTY_FUNCTION__, %ecx
        movl    $103, %edx
        movl    $.LC0, %esi
        movl    $.LC2, %edi
        call    __assert_fail
        .cfi_endproc
{% endhighlight %}

A quick look at the inner loop (the code between `.L52` and `.L50`) reveals that there is something that can be optimised there:

{% highlight c-objdump %}

        movzbl  (%rsi,%r8), %r9d
        movq    (%rcx,%r10,8), %r8
        movb    %r9b, (%r8,%rax)
{% endhighlight %}

There are three memory access instructions here. The first one reads a byte from the address `%rsi` at the offset `%r8`,
where `%r8` is a copy of a loop variable (`%edi`), which is incremented by 32 for each loop iteration. Clearly, this is

{% highlight C++ %}
    src [dst_pos * NUM_TIMESLOTS + dst_num]
{% endhighlight %}

The second one reads a quadword (8 bytes) from the address `%rcx` at the offset `%r10*8`, where `%r10` is the outer loop variable
(it is incremented by one each iteration). This is clearly `dst[dst_num]`.

And the third one writes the byte read by the first instructin to the address just read from `dst[dst_num]` at the
offset `%rax`, where `%rax` is another inner loop variable. This is writing to `dst[dst_num][dst_pos]`.

This means that `dst[dsn_num]` was not identified as an expression, which is invariant in the inner loop, and it
wasn't moved out of the loop. The reason is in potential [**pointer aliasing**](http://en.wikipedia.org/wiki/Pointer_aliasing).

Unlike **Java**, **C** and **C++** are quite tolerant towards pointer behaviour. The original version of **C** was
absolutely liberal -- pointers were allowed to point anywhere, and the same area of memory could be accessed via
multiple pointers of arbitrary types. This reduced the optimisation oppotrunity, since for any two pointers of
unknown nature the compiler should assume that they could pointer to the same memory. This is what GCC is still doing
when strict pointer aliasing is switched off. Later (in the first **C** standard, '**ANSI/ISO 9899:1990**', commonly referred
to as '**ANSI C**') the aliasing rules became stricter. An object now can only be accessed by two pointers of
"similar" types (`int` and `unsigned` are considered similar), and by a `char` pointer. This allows a compiler
to optimise pointer dereferences where types are different and neither of the pointers is a `char*`. In GNU C
this is called a **strict aliasing rule** and is switched on by the `-fstrict-aliasing` switch, which is a part of
the `-O2` optimisation pack.

The impact of the strict aliasing rule can be seen in the following examples, where `b()` models `Dst_First_1::demux`:

{% highlight C++ %}
void a (int ** p)
{
    p[0][0] = 0;
    p[0][1] = 1;
}

void b (char ** p)
{
    p[0][0] = 0;
    p[0][1] = 1;
}
{% endhighlight %}

This is how they are compiled:

{% highlight c-objdump %}
_a:
        movl    4(%esp), %eax
        movl    (%eax), %eax
        movl    $0, (%eax)
        movl    $1, 4(%eax)
        ret

_b:
        movl    4(%esp), %eax
        movl    (%eax), %edx
        movb    $0, (%edx)
        movl    (%eax), %eax
        movb    $1, 1(%eax)
        ret
{% endhighlight %}

We can see that the code for `b()` contains an extra instruction 

{% highlight c-objdump %}
        movl    (%eax), %eax
{% endhighlight %}

The reason is that the compiler assumed aliasing of `p[0]` (of type `char*`) and `p` (of type `char**`), and did not optimise the second `p[0]`.
When compiling `a()` this did not happen because `int*` does not alias `int**`.

Unfortunately, our `Dst_First_1::demux` is similar to `b()`: `dst + dst_num` is of type `char**`,
while `dst[dst_num] + dst_pos` is a `char*`, and they are considered aliased. As a result, the compiler 
did not optimise repetitive access to `dst[dst_num]`.

The analysis of the code for `Dst_First_3` shows the same problem.

The only way I know to improve the code is to optimise the program manually.
This is what we've done in `Dst_First_2`, but there we also performed strength reduction, which could affect the
code somehow. Let's write versions of `Dst_First_1` and `Dst_First_2` with only `dst[dst_num]` optimised,
but no strength reduction (the code is [here]({{ site.REPO-E1-C }}/commit/c976a9f9d44345859ac6b4c4b81dc20527842575)):

{% highlight C++ %}
class Dst_First_1a : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        assert (src_length % NUM_TIMESLOTS == 0);

        for (unsigned dst_num = 0; dst_num < NUM_TIMESLOTS; ++ dst_num) {
            byte * d = dst [dst_num];
            for (unsigned dst_pos = 0; dst_pos < src_length / NUM_TIMESLOTS; ++ dst_pos) {
                d [dst_pos] = src [dst_pos * NUM_TIMESLOTS + dst_num];
            }
        }
    }
};

class Dst_First_3a : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        assert (src_length == NUM_TIMESLOTS * DST_SIZE);

        for (unsigned dst_num = 0; dst_num < NUM_TIMESLOTS; ++ dst_num) {
            byte * d = dst [dst_num];
            for (unsigned dst_pos = 0; dst_pos < DST_SIZE; ++ dst_pos) {
                d [dst_pos] = src [dst_pos * NUM_TIMESLOTS + dst_num];
            }
        }
    }
};
{% endhighlight %}

Results

    11Dst_First_1: 1455
    11Dst_First_2: 1453
    11Dst_First_3: 1463
    12Dst_First_1a: 1454
    12Dst_First_3a: 1443

do not show anything exceptional. The speed of `Dst_First_1` did not improve due to the optimisation, while the
speed of `Dst_First_3` did.

The code for `Dst_First_1a` looks better (I'll only show the inner loop, the rest you can see 
[in the repository, file e1.asm]({{ site.REPO-E1-C }}/blob/c976a9f9d44345859ac6b4c4b81dc20527842575/e1.asm)):

{% highlight c-objdump %}
.L37:
        movl    %edi, %r8d
        addl    $32, %edi
        movzbl  (%rsi,%r8), %r8d
        movb    %r8b, (%r9,%rax)
        addq    $1, %rax
        cmpl    %eax, %edx
        ja      .L37
{% endhighlight %}

It is difficult to say why this code isn't faster than the original one. Perhaps an extra instruction got
absorbed between the latencies of some other instructions, and the extra dependency that it creates was eliminated
by dynamic instruction reordering. It is interesting to check how many CPU cycles it takes to perform the loop.
This loop is executed `32 * 64 * 1000000`
times in 1454 ms, which makes it 0.71 ns per loop. The CPU runs at 2.6 GHz, or at
0.38 ns per cycle. This means that these 7 instructions take 1.87 CPU cycles to execute. For this to happen,
a lot of parallel execution must be going on, and who knows what the limit is? Perhaps, an extra instruction can be
squeezed in somewhere as well.

The code for `Dst_First_2` looks exactly like `Dst_First_1a` (except for one insignificant instruction swap),
which proves that GNU C is good enough in performing operation strength reduction, and there is no point helping it
to do it. We should rather try writing more easily readable code, and I believe that `Dst_First_1a` is easier
to read than `Dst_First_2`.

The code for `Dst_First_3a` looks the best of all, you can see it in [`e1.asm`]({{ site.REPO-E1-C }}/blob/c976a9f9d44345859ac6b4c4b81dc20527842575/e1.asm) in the repository.

But the code still runs slowly. The optimisation has not helped -- at least, not on its own.

Looking at the code again: loop unrolling
-----------------------------------------

The code for both `Dst_First_1a` and `Dst_First_3a` seems almost ideal. I can't rewrite the assembly version
manually and make it much faster; most likely, I can't make it faster at all.

Unless I unroll the loops.

We've been talking a lot about loop unrolling in the original article (["{{ site.TITLE-E1 }}"]({{ site.ART-E1 }})).
We performed manual loop unrolling, sometimes taking it to extremes. But we know that some compilers are capable
of performing some level of loop unrolling automatically. The only way to explain why **Java** executes this
code in 1022 ms while **C**-compiled code runs for 1443 ms is that **Java** performs loop unrolling while **C++** does not.

Why does GNU C not do it? Could it be that it is just a bad compiler? No. Unrolling all the loops in the program
is probably as bad an idea as not unrolling anything at all, if not worse. Loop unrolling increases code size
big time, and bigger code often runs slower. In addition, when the exact loop iteration count is unknown, some
extra code is required at loop entry or exit to compensate for possible "leftovers". In short, loop unrolling
must be performed where it definitely benefits the program, and the **C** compiler does not know where this is, unless
it has access to profiling data. **Java** has access to profiling data by design, that's why it can perform
such optimisations more efficiently.

GNU C has several switches that control loop unrolling:

<table>
<tr><td><pre>-floop-optimize</pre></td>
<td>
Perform loop optimizations: move constant expressions out of loops, simplify exit test conditions and optionally
do strength-reduction and loop unrolling as well. Enabled at levels <code>-O</code>, <code>-O2</code>, <code>-O3</code>, <code>-Os</code>
</td></tr>

<tr><td><pre>-funroll-loops</pre></td>
<td>
Unroll loops whose number of iterations can be determined at compile time or upon entry to the loop.
<code>-funroll-loops</code> implies <code>-frerun-cse-after-loop</code>. It also turns on complete loop peeling (i.e. complete removal of
loops with small constant number of iterations). This option makes code larger, and may or may not make it run faster. 
</td></tr>

<tr><td><pre>-funroll-all-loops</pre></td>
<td>
Unroll all loops, even if their number of iterations is uncertain when the loop is entered. This usually makes
programs run more slowly. <code>-funroll-all-loops</code> implies the same options as <code>-funroll-loops</code>
</td></tr>

</table>

There are also some switches that control maximal size of unrolled loop, maximal number of times loops are unrolled
and so on. We won't use them, relying on the default values.

As you can see, `-floop-optimize` is capable of performing some unrolling, and it is invoked with `-O3` option.
But it didn't unroll our loops.

The option `-funroll-all-loops` looks dangerous, it can easily make the program run slower.

Let's try `-funroll-loops`:

    c++ -O3 -falign-functions=32 -falign-loops=32 -funroll-loops -o e1 e1.cpp -lrt
    ./e1

    9Reference: 1604
    11Src_First_1: 1119
    11Src_First_2: 1101
    11Src_First_3: 1458
    11Dst_First_1: 1079
    11Dst_First_2: 880
    11Dst_First_3: 1009
    12Dst_First_1a: 882
    12Dst_First_3a: 727
    10Unrolled_1: 635
    12Unrolled_1_2: 633
    12Unrolled_1_4: 651
    12Unrolled_1_8: 648
    13Unrolled_1_16: 635
    15Unrolled_2_Full: 635

Let's put all the results into one table. The old results are taken from
[The mystery of an unstable performance]({{ page.ART-UNSTABLE }}).

<table class="numeric">
<tr><th>     Version              </th> <th>  Time in Java </th> <th> Old time in C</th><th> New time in C</th></tr>
<tr><td class="label">Reference      </td><td >  2860 </td><td > 1939</td><td > 1604</td></tr>
<tr><td class="label">Src_First_1    </td><td >  2481 </td><td > 1888</td><td > 1119</td></tr>
<tr><td class="label">Src_First_2    </td><td >  2284 </td><td > 1341</td><td > 1101</td></tr>
<tr><td class="label">Src_First_3    </td><td >  4360 </td><td > 1891</td><td > 1458</td></tr>
<tr><td class="label">Dst_First_1    </td><td >  1155 </td><td > 1457</td><td > 1079</td></tr>
<tr><td class="label">Dst_First_1a   </td><td >       </td><td >     </td><td >  882</td></tr>
<tr><td class="label">Dst_First_2    </td><td >  2093 </td><td > 1454</td><td >  880</td></tr>
<tr><td class="label">Dst_First_3    </td><td >  1022 </td><td > 1464</td><td > 1009</td></tr>
<tr><td class="label">Dst_First_3a   </td><td >       </td><td >     </td><td >  727</td></tr>
<tr><td class="label">Unrolled_1     </td><td >   659 </td><td >  636</td><td >  635</td></tr>
<tr><td class="label">Unrolled_1_2   </td><td >   654 </td><td >  634</td><td >  633</td></tr>
<tr><td class="label">Unrolled_1_4   </td><td >   636 </td><td >  655</td><td >  651</td></tr>
<tr><td class="label">Unrolled_1_8   </td><td >   637 </td><td >  650</td><td >  648</td></tr>
<tr><td class="label">Unrolled_1_16  </td><td > 25904 </td><td >  635</td><td >  635</td></tr>
<tr><td class="label">Unrolled_2_Full</td><td > 15630 </td><td >  635</td><td >  635</td></tr>
</table>


Observations:

- All the versions (excluding manually unrolled) became faster.

- All the versions now run faster than in **Java**.

- `Dst_First` versions now run faster than `Src_First`, it was not so in **C** initially.

- While resolving the pointer aliasing and optimising the loop did not help on its own, it helped a lot
  in combination with the loop unrolling. This is visible by comparing `Dst_First_1` with `Dst_First_1a` and
  `Dst_First_3` with `Dst_First_3a`.

- The same is true for hard-coded loop boundaries. It didn't help much on its own but made a big difference
  in combination with the loop unrolling. Compare `Dst_First_1` with `Dst_First_3` and `Dst_First_1a` with `Dst_First_3a`.

- The `Dst_First_3a` version's speed is very close to the `Unrolled`'s speed, which makes it a perfect choice
  where extreme performance is not necessary

- The `Dst_First_1a` is slower, but its speed is still acceptable for a universal solution (the one without
  hard-coded sizes).

- This is probably all we can do by applying simple changes and tweaking compiler switches. More effort is
  required if we need to improve speed even more.


The unrolled code
-----------------

Let's have a look at the unrolled code
(I'm excluding the assertion part; 
[the full text is in the repository]({{ site.REPO-E1-C }}/commit/2f0d5d7566ce32993f802c2fe906329e66904b3f)).
This is `Dst_First_1a`:

{% highlight c-objdump %}
_ZNK12Dst_First_1a5demuxEPKhjPPh:
        subq    $8, %rsp
        testb   $31, %dl
        jne     .L77
        shrl    $5, %edx
        xorl    %r10d, %r10d
        .p2align 5
.L45:
        testl   %edx, %edx
        movl    %r10d, %r11d
        movq    (%rcx,%r10,8), %r8
        je      .L43
        movl    %r10d, %edi
        leal    -1(%rdx), %r9d
        movzbl  (%rsi,%rdi), %eax
        leal    32(%r11), %edi
        andl    $7, %r9d
        cmpl    $1, %edx
        movb    %al, (%r8)
        movl    $1, %eax
        jbe     .L43
        testl   %r9d, %r9d
        je      .L44
        cmpl    $1, %r9d
        je      .L71
        cmpl    $2, %r9d
        je      .L72
        cmpl    $3, %r9d
        je      .L73
        cmpl    $4, %r9d
        je      .L74
        cmpl    $5, %r9d
        je      .L75
        cmpl    $6, %r9d
        je      .L76
        movzbl  (%rsi,%rdi), %eax
        leal    64(%r11), %edi
        movb    %al, 1(%r8)
        movl    $2, %eax
.L76:
        movl    %edi, %r11d
        addl    $32, %edi
        movzbl  (%rsi,%r11), %r9d
        movb    %r9b, (%r8,%rax)
        addq    $1, %rax
.L75:
        movl    %edi, %r11d
        addl    $32, %edi
        movzbl  (%rsi,%r11), %r9d
        movb    %r9b, (%r8,%rax)
        addq    $1, %rax
.L74:
        movl    %edi, %r11d
        addl    $32, %edi
        movzbl  (%rsi,%r11), %r9d
        movb    %r9b, (%r8,%rax)
        addq    $1, %rax
.L73:
        movl    %edi, %r11d
        addl    $32, %edi
        movzbl  (%rsi,%r11), %r9d
        movb    %r9b, (%r8,%rax)
        addq    $1, %rax
.L72:
        movl    %edi, %r11d
        addl    $32, %edi
        movzbl  (%rsi,%r11), %r9d
        movb    %r9b, (%r8,%rax)
        addq    $1, %rax
.L71:
        movl    %edi, %r11d
        addl    $32, %edi
        movzbl  (%rsi,%r11), %r9d
        movb    %r9b, (%r8,%rax)
        addq    $1, %rax
        cmpl    %eax, %edx
        jbe     .L43
.L44:
        movl    %edi, %r11d
        movzbl  (%rsi,%r11), %r9d
        leal    32(%rdi), %r11d
        movb    %r9b, (%r8,%rax)
        movzbl  (%rsi,%r11), %r9d
        leal    64(%rdi), %r11d
        movb    %r9b, 1(%r8,%rax)
        movzbl  (%rsi,%r11), %r9d
        leal    96(%rdi), %r11d
        movb    %r9b, 2(%r8,%rax)
        movzbl  (%rsi,%r11), %r9d
        leal    128(%rdi), %r11d
        movb    %r9b, 3(%r8,%rax)
        movzbl  (%rsi,%r11), %r9d
        leal    160(%rdi), %r11d
        movb    %r9b, 4(%r8,%rax)
        movzbl  (%rsi,%r11), %r9d
        leal    192(%rdi), %r11d
        movb    %r9b, 5(%r8,%rax)
        movzbl  (%rsi,%r11), %r9d
        leal    224(%rdi), %r11d
        addl    $256, %edi
        movb    %r9b, 6(%r8,%rax)
        movzbl  (%rsi,%r11), %r9d
        movb    %r9b, 7(%r8,%rax)
        addq    $8, %rax
        cmpl    %eax, %edx
        ja      .L44
{% endhighlight %}

This is `Dst_First_3a`:

{% highlight c-objdump %}
_ZNK12Dst_First_3a5demuxEPKhjPPh:
        subq    $8, %rsp
        cmpl    $2048, %edx
        jne     .L40
        xorl    %r9d, %r9d
        .p2align 5
.L28:
        movq    (%rcx,%r9,8), %rdi
        movl    %r9d, %edx
        xorl    %eax, %eax
        .p2align 5
.L29:
        movl    %edx, %r8d
        leal    32(%rdx), %r10d
        movzbl  (%rsi,%r8), %r11d
        movb    %r11b, (%rdi,%rax)
        movzbl  (%rsi,%r10), %r8d
        leal    64(%rdx), %r11d
        movb    %r8b, 1(%rdi,%rax)
        movzbl  (%rsi,%r11), %r10d
        leal    96(%rdx), %r8d
        movb    %r10b, 2(%rdi,%rax)
        movzbl  (%rsi,%r8), %r11d
        leal    128(%rdx), %r10d
        movb    %r11b, 3(%rdi,%rax)
        movzbl  (%rsi,%r10), %r8d
        leal    160(%rdx), %r11d
        movb    %r8b, 4(%rdi,%rax)
        movzbl  (%rsi,%r11), %r10d
        leal    192(%rdx), %r8d
        movb    %r10b, 5(%rdi,%rax)
        movzbl  (%rsi,%r8), %r11d
        leal    224(%rdx), %r10d
        addl    $256, %edx
        movb    %r11b, 6(%rdi,%rax)
        movzbl  (%rsi,%r10), %r8d
        movb    %r8b, 7(%rdi,%rax)
        addq    $8, %rax
        cmpq    $64, %rax
        jne     .L29
        addq    $1, %r9
        cmpq    $32, %r9
        jne     .L28
        addq    $8, %rsp
        .cfi_remember_state
        .cfi_def_cfa_offset 8
        ret
{% endhighlight %}

Observations:

- The inner loop was unrolled 8 times in both cases

- The inner loop of `Dst_First_1a` was 23 bytes long; became 140 bytes

- The inner loop of `Dst_First_3a` was 23 bytes long; became 142 bytes

- The entire `Dst_First_1a` was 123 bytes long; became 451 byte

- The entire `Dst_First_3a` was 123 bytes long; became 246 bytes

- The loop unrolling killed the loop alignment in `Dst_First_1a`: the loop starts at `0x410c2f`,
  still explicitly aligned. A possible reason is that the loop entry is also a target for a normal branch.
  I think this is wrong, possibly a deficiency of GNU C.

- Both loops are explicitly aligned in `Dst_First_3a`

- Comparison instructions with jumps to labels from `.L76` to `.L71` are sorting out leftovers. They check the
  number of iterations modulo 7. If it is zero, the code jumps to the loop entry, if it is one, the code processes
  one byte and then jumps to the loop entry, and so on.

- The code for `Dst_First_3a` does not need these tests and looks much neater

- The code still does not look optimal: `leal` instruction could have been merged with the memory load operation. Perhaps,
  it has something to do with the operand size: the `leal` is a 32-bit instructions. This is something to
  investigate. It might mean that the previous statement "This is probably all we can do by applying simple changes and tweaking compiler switches"
  was not correct.


Conclusions
-----------

- Pointer aliasing requires manual treatment. Even if the performance does not improve, it may improve in
  combination with other optimisations.

- When applied in right places, loop unrolling can improve performance a lot. It may require using additional
  command-line arguments or pragmas/attributes.

- GCC can do strength reduction; there is no need to perform it manually

- Finally, the **C**-compiled program **does** run faster than the one in **Java**

- There is still something wrong with the code

Coming soon
-----------

In the next article I'm going to look at the code generated for `Dst_First_3a` closely and check if it can be improved.
