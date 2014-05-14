---
layout: post
title:  The mystery of an unstable performance
date:   2014-05-12 12:00:00
tags: C C++ GCC optimisation
---

Most people love detective stories. It is a bit illogical. The situation where everyone is alive and well, and all
the valuables are where they were left, for logically minded person must be much more enjoyable than the case
when they know exactly who killed half a dozen people and stole the money.

Most programmers love resolving mysteries. For many, like me, this is the biggest fun in programming -- to find
out what went wrong and why. This is also illogical. Logically, the biggest fun should have been when everything just worked.

Obviously, our problems are different. A typical problem is "Why doesn't this program work". Optimisation adds
another one: "Why is this program slow?". Both require full-scale investigation work, and as a result
a programmer feels inside the detective story, as either a victim, a criminal or a detective, depending on the
outcome.

Our investigations have a lot in common with what those clever detectives do in the books.
They come up with the ideas and evaluate them -- we do that too.
They interview wintesses -- we read logfiles. They take samples and run laboratory tests -- so do we.
There is even our equivalent of crowling in mud at the crime scene under moonlight looking for evidence -- it
is reading binary code in the middle of a faulty program, a very exciting activity indeed.

Today I want to share one such investigation story, which was quite entertaining for me. I am going to tell the
story as it unfolds, with the solution right in the end.

The mystery
-----------

As you might remember, in the article ["De-multiplexing of E1 stream: converting to C"](http://pzemtsov.github.io/2014/05/01/demultiplexing-of-e1-converting-to-C.html)
we converted the **Java** code into **C++** and ran it on a Sandy Bridge processor. These were the results we got:

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

Let's concentrate on a 'Dst_First' family of versions:

    11Dst_First_1: 1467
    11Dst_First_2: 1445
    11Dst_First_3: 1761

Let me remind you the nature of these three versions:

- `Dst_First_1`: a simplest, unoptimised version;
- `Dst_First_2`: a result of manual optimisation of the `Dst_First_1`. Specifically, loop invariant code motion
   and operation strength reduction were performed. In **Java** It caused severe performance degradation;
- `Dst_First_3`: a version of `Dst_First_1` with a hard-coded size of input and output buffers.

We couldn't predict relative performance of `Dst_First_1` and `Dst_First_2`. Each compiler may respond differently to
manual optimisations. The generated code could have become better or worse. In our case it seems to have become better, there
is nothing unusual here.

But the performance of `Dst_First_3` is definitely unusual. I expected it to be as fast as `Dst_First_1`, or,
even more likely, faster. And everyone would. This is common sense: programs don't become slower from replacing
of variable loop limits with hard-coded constants -- unless there is something wrong with a compiler. And here it
is 20% slower. This is a mystery, and it requires an investigation. Without knowing the answer, we can't go
forward. What's a point optimising a program by 5% or 10% if some unknown factor can suddenly make it lose 20%
of speed?

Simplifying the program
-----------------------

When facing any kind of a mystery it always helps reducing the situation to the simplest one possible, where all
irrelevant or unnecessary factors have been eliminated. This is what we're trying to achieve now. We'll do the following:

- remove all the versions other than `Dst_First_1` and `Dst_First_3`
- remove call to `check()`, which allows removing of `Reference` as well
- remove asserts

here is the code for two versions in question:

{% highlight C++ %}
class Dst_First_1 : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        for (unsigned dst_num = 0; dst_num < NUM_TIMESLOTS; ++ dst_num) {
            for (unsigned dst_pos = 0; dst_pos < src_length / NUM_TIMESLOTS; ++ dst_pos) {
                dst [dst_num][dst_pos] = src [dst_pos * NUM_TIMESLOTS + dst_num];
            }
        }
    }
};

class Dst_First_3 : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        for (unsigned dst_num = 0; dst_num < NUM_TIMESLOTS; ++ dst_num) {
            for (unsigned dst_pos = 0; dst_pos < DST_SIZE; ++ dst_pos) {
                dst [dst_num][dst_pos] = src [dst_pos * NUM_TIMESLOTS + dst_num];
            }
        }
    }
};
{% endhighlight %}

The new code is in file [`e1-2.cpp`](https://github.com/pzemtsov/article-E1-demux-C/commit/01988e4f8bdff46fedf0573e54bc43332402a46b). Let's compile it and run:

    $ c++ -O3 -o e1-2 e1-2.cpp -lrt
    $ ./e1-2
    11Dst_First_1: 1747
    11Dst_First_3: 1761

Suddenly both versions run at similar speed. It might seem that we've lost the effect and achieved nothing, but it is
not so. We've established something important: the `Dst_First_1` isn't always much faster then `Dst_First_3`, which means that
its code isn't necessarily better. Besides, it can be fast or slow, depending on the environment. We are not in fact
dealing with performance degradation in `Dst_First_3`, we are dealing with a stability issue. Our investigation
becomes even more important, because we can't seriously embark on any optimisation exercise when our measurements
are so unstable.

Could it be that the compiler generates different code in different circumstances? Or perhaps the code is the same but
some places in memory are better than the others? We'll look at it now. To do it, we don't need two functions any more,
one is enough -- the one that's unstable.

Testing the unstable
--------------------

I'll remove `Dst_First_3` but instead create several copies of `Dst_First_1`. In **C++** it is easy to do using
templates (see [`e1-3.cpp`](https://github.com/pzemtsov/article-E1-demux-C/commit/7a67fe72b9829caf5fcb9d6435e47094c166a29f)):

{% highlight C++ %}
template<int N> class Dst_First_1 : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        for (unsigned dst_num = 0; dst_num < NUM_TIMESLOTS; ++ dst_num) {
            for (unsigned dst_pos = 0; dst_pos < src_length / NUM_TIMESLOTS; ++ dst_pos) {
                dst [dst_num][dst_pos] = src [dst_pos * NUM_TIMESLOTS + dst_num];
            }
        }
    }
};

int main (void)
{
    src = generate ();
    dst = allocate_dst ();

    measure (Dst_First_1<0> ());
    measure (Dst_First_1<1> ());
    measure (Dst_First_1<2> ());
    measure (Dst_First_1<3> ());
    measure (Dst_First_1<4> ());
    measure (Dst_First_1<5> ());

    return 0;
}

{% endhighlight %}

Here is the result:

    $ ./e1-3
    11Dst_First_1ILi0EE: 1749
    11Dst_First_1ILi1EE: 1464
    11Dst_First_1ILi2EE: 1746
    11Dst_First_1ILi3EE: 1464
    11Dst_First_1ILi4EE: 1745
    11Dst_First_1ILi5EE: 1463

This is what we were looking for: an instability in its purest forms. The compiler seems to like versions with
odd values of `N` but hate those with even ones.

What's the difference?
----------------------

Analysing the assembly file helps  checking if the code produced for different versions differed. But this file does not
show where these versions reside in memory. To see that, we need a universal programmer's tool -- a debugger, in our
case `gdb`.

However, we still need the assembly file: we need to know the mangled **C++** symbol names. It is easier to look
at the assembly than learning the rules of this mangling.

A quick look at the assembly output shows the symbol names:

    _ZNK11Dst_First_1ILi5EE5demuxEPKhjPPh
    _ZNK11Dst_First_1ILi4EE5demuxEPKhjPPh
    _ZNK11Dst_First_1ILi3EE5demuxEPKhjPPh
    _ZNK11Dst_First_1ILi2EE5demuxEPKhjPPh
    _ZNK11Dst_First_1ILi1EE5demuxEPKhjPPh
    _ZNK11Dst_First_1ILi0EE5demuxEPKhjPPh

Clearly, the digit after the `"ILi"` indicates the value of the template parameter `N`. Note the order -- the
methods are listed in the reverse order of their instantiation. This seems to be the general rule in GNU C:
the code is generated last function first. This means that the code for a fast version comes first, followed
by a slow version, followed by another fast and so on. Let's look at the first two, `Dst_First_1<5>` and
`Dst_First_1<4>`. For that let's run `gdb`:

    $ gdb ./e1-3
    Reading symbols from /root/cpp/e1-3...(no debugging symbols found)...done.

Sorry, my mistake. We need symbol table, which is not generated by default. Fortunately, this can be done easily:

    $ c++ -g -O3 -o e1-3 e1-3.cpp -lrt
    $ gdb ./e1-3
    Reading symbols from /root/cpp/e1-3...done.

This is better. Let's look at the code now:

{% highlight text %}
(gdb) disassemble _ZNK11Dst_First_1ILi5EE5demuxEPKhjPPh
Dump of assembler code for function Dst_First_1<5>::demux(unsigned char const*, unsigned int, unsigned char**) const:
   0x0000000000400f80 <+0>:     shr    $0x5,%edx
   0x0000000000400f83 <+3>:     xor    %r10d,%r10d
   0x0000000000400f86 <+6>:     nopw   %cs:0x0(%rax,%rax,1)
   0x0000000000400f90 <+16>:    xor    %eax,%eax
   0x0000000000400f92 <+18>:    test   %edx,%edx
   0x0000000000400f94 <+20>:    mov    %r10d,%edi
   0x0000000000400f97 <+23>:    je     0x400fbb <Dst_First_1<5>::demux(unsigned char const*, unsigned int, unsigned char**) const+59>
   0x0000000000400f99 <+25>:    nopl   0x0(%rax)
   0x0000000000400fa0 <+32>:    mov    %edi,%r8d
   0x0000000000400fa3 <+35>:    add    $0x20,%edi
   0x0000000000400fa6 <+38>:    movzbl (%rsi,%r8,1),%r9d
   0x0000000000400fab <+43>:    mov    (%rcx,%r10,8),%r8
   0x0000000000400faf <+47>:    mov    %r9b,(%r8,%rax,1)
   0x0000000000400fb3 <+51>:    add    $0x1,%rax
   0x0000000000400fb7 <+55>:    cmp    %eax,%edx
   0x0000000000400fb9 <+57>:    ja     0x400fa0 <Dst_First_1<5>::demux(unsigned char const*, unsigned int, unsigned char**) const+32>
   0x0000000000400fbb <+59>:    add    $0x1,%r10
   0x0000000000400fbf <+63>:    cmp    $0x20,%r10
   0x0000000000400fc3 <+67>:    jne    0x400f90 <Dst_First_1<5>::demux(unsigned char const*, unsigned int, unsigned char**) const+16>
   0x0000000000400fc5 <+69>:    repz retq 
End of assembler dump.
(gdb) disassemble _ZNK11Dst_First_1ILi4EE5demuxEPKhjPPh
Dump of assembler code for function Dst_First_1<4>::demux(unsigned char const*, unsigned int, unsigned char**) const:
   0x0000000000400fd0 <+0>:     shr    $0x5,%edx
   0x0000000000400fd3 <+3>:     xor    %r10d,%r10d
   0x0000000000400fd6 <+6>:     nopw   %cs:0x0(%rax,%rax,1)
   0x0000000000400fe0 <+16>:    xor    %eax,%eax
   0x0000000000400fe2 <+18>:    test   %edx,%edx
   0x0000000000400fe4 <+20>:    mov    %r10d,%edi
   0x0000000000400fe7 <+23>:    je     0x40100b <Dst_First_1<4>::demux(unsigned char const*, unsigned int, unsigned char**) const+59>
   0x0000000000400fe9 <+25>:    nopl   0x0(%rax)
   0x0000000000400ff0 <+32>:    mov    %edi,%r8d
   0x0000000000400ff3 <+35>:    add    $0x20,%edi
   0x0000000000400ff6 <+38>:    movzbl (%rsi,%r8,1),%r9d
   0x0000000000400ffb <+43>:    mov    (%rcx,%r10,8),%r8
   0x0000000000400fff <+47>:    mov    %r9b,(%r8,%rax,1)
   0x0000000000401003 <+51>:    add    $0x1,%rax
   0x0000000000401007 <+55>:    cmp    %eax,%edx
   0x0000000000401009 <+57>:    ja     0x400ff0 <Dst_First_1<4>::demux(unsigned char const*, unsigned int, unsigned char**) const+32>
   0x000000000040100b <+59>:    add    $0x1,%r10
   0x000000000040100f <+63>:    cmp    $0x20,%r10
   0x0000000000401013 <+67>:    jne    0x400fe0 <Dst_First_1<4>::demux(unsigned char const*, unsigned int, unsigned char**) const+16>
   0x0000000000401015 <+69>:    repz retq 
End of assembler dump.
{% endhighlight %}

The two pieces of code are completely identical. There is no difference in code generation. They are placed close
together: the first one ends at `0x400fc5` and the second one starts at `0x400fd0`. Stop. The first one starts
at `0x400f80`. Maybe this is the difference? One address is a multiple of `0x20` and the other one is only a multiple
of `0x10`. In other words, the first version is aligned by 32 (in fact, also by 64 and by 128), while the other one
is only aligned by 16.

Aligning functions
------------------

Fortunately, it is very easy to test the effect of function alignment. GNU C has a special switch to control it:
`-falign-functions`

    $ c++ -O3 -falign-functions=32 -o e1-3 e1-3.cpp -lrt
    $ ./e1-3
    11Dst_First_1ILi0EE: 1464
    11Dst_First_1ILi1EE: 1464
    11Dst_First_1ILi2EE: 1464
    11Dst_First_1ILi3EE: 1464
    11Dst_First_1ILi4EE: 1464
    11Dst_First_1ILi5EE: 1464

It seems that we have found the culprit. The execution speed benefits from functions being aligned by 32.
If a function isn't explicitly aligned by 32, it is a matter of luck
where it will end up, aligned or not. Its starting address depends on presence of other functions, their sizes and the order of
their generation. No wonder we've got such unstable results.

Back to original problem
------------------------

We have found what was wrong. It's time to finish the article, there is just a formality left: to test the
solution on other examples. Let's go back to `e1-2.cpp` and run it again:

    $ c++ -O3 -falign-functions=32 -o e1-2 e1-2.cpp -lrt
    $ ./e1-2
    11Dst_First_1: 1463
    11Dst_First_3: 1760

As it turns out, our triumph was premature. The solutiom does not work on `Dst_First_3`. So here we are, back in
the depth of the assembly listings.

These are the names of our methods:

    _ZNK11Dst_First_15demuxEPKhjPPh:
    _ZNK11Dst_First_35demuxEPKhjPPh:

the former one being `Dst_First_1::demux` and the latter `Dst_First_3::demux`. Now back to the gdb:

{% highlight text %}
$ c++ -g -O3 -falign-functions=32 -o e1-2 e1-2.cpp -lrt
$ gdb ./e1-2
Reading symbols from /root/cpp/e1-2...done.
(gdb) disassemble _ZNK11Dst_First_15demuxEPKhjPPh
Dump of assembler code for function Dst_First_1::demux(unsigned char const*, unsigned int, unsigned char**) const:
   0x0000000000400f60 <+0>:     shr    $0x5,%edx
   0x0000000000400f63 <+3>:     xor    %r10d,%r10d
   0x0000000000400f66 <+6>:     nopw   %cs:0x0(%rax,%rax,1)
   0x0000000000400f70 <+16>:    xor    %eax,%eax
   0x0000000000400f72 <+18>:    test   %edx,%edx
   0x0000000000400f74 <+20>:    mov    %r10d,%edi
   0x0000000000400f77 <+23>:    je     0x400f9b <Dst_First_1::demux(unsigned char const*, unsigned int, unsigned char**) const+59>
   0x0000000000400f79 <+25>:    nopl   0x0(%rax)
   0x0000000000400f80 <+32>:    mov    %edi,%r8d
   0x0000000000400f83 <+35>:    add    $0x20,%edi
   0x0000000000400f86 <+38>:    movzbl (%rsi,%r8,1),%r9d
   0x0000000000400f8b <+43>:    mov    (%rcx,%r10,8),%r8
   0x0000000000400f8f <+47>:    mov    %r9b,(%r8,%rax,1)
   0x0000000000400f93 <+51>:    add    $0x1,%rax
   0x0000000000400f97 <+55>:    cmp    %eax,%edx
   0x0000000000400f99 <+57>:    ja     0x400f80 <Dst_First_1::demux(unsigned char const*, unsigned int, unsigned char**) const+32>
   0x0000000000400f9b <+59>:    add    $0x1,%r10
   0x0000000000400f9f <+63>:    cmp    $0x20,%r10
   0x0000000000400fa3 <+67>:    jne    0x400f70 <Dst_First_1::demux(unsigned char const*, unsigned int, unsigned char**) const+16>
   0x0000000000400fa5 <+69>:    repz retq 
End of assembler dump.
(gdb) disassemble _ZNK11Dst_First_35demuxEPKhjPPh
Dump of assembler code for function Dst_First_3::demux(unsigned char const*, unsigned int, unsigned char**) const:
   0x0000000000400fc0 <+0>:     xor    %r9d,%r9d
   0x0000000000400fc3 <+3>:     nopl   0x0(%rax,%rax,1)
   0x0000000000400fc8 <+8>:     mov    %r9d,%edx
   0x0000000000400fcb <+11>:    xor    %eax,%eax
   0x0000000000400fcd <+13>:    nopl   (%rax)
   0x0000000000400fd0 <+16>:    mov    %edx,%edi
   0x0000000000400fd2 <+18>:    add    $0x20,%edx
   0x0000000000400fd5 <+21>:    movzbl (%rsi,%rdi,1),%r8d
   0x0000000000400fda <+26>:    mov    (%rcx,%r9,8),%rdi
   0x0000000000400fde <+30>:    mov    %r8b,(%rdi,%rax,1)
   0x0000000000400fe2 <+34>:    add    $0x1,%rax
   0x0000000000400fe6 <+38>:    cmp    $0x40,%rax
   0x0000000000400fea <+42>:    jne    0x400fd0 <Dst_First_3::demux(unsigned char const*, unsigned int, unsigned char**) const+16>
   0x0000000000400fec <+44>:    add    $0x1,%r9
   0x0000000000400ff0 <+48>:    cmp    $0x20,%r9
   0x0000000000400ff4 <+52>:    jne    0x400fc8 <Dst_First_3::demux(unsigned char const*, unsigned int, unsigned char**) const+8>
   0x0000000000400ff6 <+54>:    repz retq 
End of assembler dump.
{% endhighlight %}

We can see that both pieces of code are aligned by 32. The second piece is shorter, for obvious reasons. It does not
have to calculate the number of iterations of the loop and check if is is zero. The inner loops are almost identical,
except the loop in the first code compares the loop variable (`%eax`) to another variable (`%edx`), while the
second code compares it the constant `$0x40` (at `0x400fe6`). The `ja` (resp. `jne`) instructions that follows
is the main loop instruction -- it jumps to the beginning of the loop. The inner loop starts at 0x400f80 (0x20 offset
from the start of the function) in the first case and at 0x400fd0 (0x10 offset from the start) in the second.
These offsets look very round. Is it a coincidence? No, it is not. There are suspicious instructions just
before the loops:

    nopw   %cs:0x0(%rax,%rax,1)

(10 bytes long) in the first case, and

    nopl   (%rax)

(3 bytes long) in another.

These are the modern day versions of a good old `NOP`, or no operation instruction. By using appropriate combination
of arguments and prefixes, we can manufacture these instructions of virtually any length. The compiler uses them
to align some point in the code, in our case the start of the loop body. And, as we can see in the second case,
the loops were aligned by 16. It looks like it was the alignment of the loop that was important. The alignment
of the function just helped align the loop the loop started at aligned offset from the start of the function. We were lucky
with `Dst_First_1` but not in `Dst_First_3`. Funny enough, the source of this luck was longer code preceding the loop
(25 bytes instead of 13).

Aligning loops
--------------

Fortunately, GCC has a special switch controlling loop alignment as well: `-falign-loops`.

    $ c++ -O3 -falign-functions=32 -falign-loops=32 -o e1-2 e1-2.cpp -lrt
    $ ./e1-2
    11Dst_First_1: 1456
    11Dst_First_3: 1462

This looks much better. In this case we don't need `-falign-functions` any more, since it is implied. A compiler
can't align the loops by 32 if the functions aren't aligned by 32. However, if we want to align functions without loops
(such as our fully inlined version), we still need this switch. So let's use both.

Why is the alignment so importent? Apparently, a processor, after executing the jump instruction, starts fetching and
decoding instructions at the new location. The instructions are fetched into a prefetch buffer and then decoded
(more than one at a time). This buffer is always aligned by 16 or 32 bytes (depending on the exact processor model).
If, for instance, the buffer is aligned by 16 and the new instruction pointer location is only 5 bytes before the
next 16-bytes boundary, only these 5 bytes are decoded, and then the processor must fetch again. In this case
instruction fetching will become a major bottleneck, making the entire loop run slowly.

It turns out that the Sandy Bridge has this buffer aligned by 32, as a result it benefits from loops being aligned by 32.
Other processors require alignment by 16. The recommendation is not "always align by 32"; it is "compile for your
target architecture". Unfortunately, the version of GCC I use does not seem to have a switch "compile for Sandy Bridge",
that's why manual specification of important parameters is required.

It is amazing how big a difference this makes -- bigger than many real optimisations.

Is there any cost?
------------------

Aligning of functions is almost free. I say "almost", because there are some second-order effects. Any increase in the
code size makes caching less efficient. If less code fits into a memory page, more pages are required, which affects
the initial page loading and slows down program startup. It also affects [TLB](http://en.wikipedia.org/wiki/Translation_lookaside_buffer)
preformance. But these are all minor effects. What's important is that the alignment is achieved by inserting
bytes that are not executed. The start of a function is just moved to the next aligned location.

Aligning of loops isn't for free. Most loops are entered in "fall through" way, The control naturally arives to the start of
the loop from the top, all preceding code being executed. Let's have a look what this code looks like in our case (`Dst_First_3`).
In e1-2:

{% highlight text %}
(gdb) disassemble /r _ZNK11Dst_First_35demuxEPKhjPPh
Dump of assembler code for function Dst_First_3::demux(unsigned char const*, unsigned int, unsigned char**) const:
   0x0000000000400fe0 <+0>:     45 31 c9        xor    %r9d,%r9d
   0x0000000000400fe3 <+3>:     66 66 66 66 66 2e 0f 1f 84 00 00 00 00 00       data32 data32 data32 data32 nopw %cs:0x0(%rax,%rax,1)
   0x0000000000400ff1 <+17>:    66 66 66 66 66 66 2e 0f 1f 84 00 00 00 00 00    data32 data32 data32 data32 data32 nopw %cs:0x0(%rax,%rax,1)
   0x0000000000401000 <+32>:    44 89 ca        mov    %r9d,%edx
   0x0000000000401003 <+35>:    31 c0   xor    %eax,%eax
   0x0000000000401005 <+37>:    66 66 66 2e 0f 1f 84 00 00 00 00 00     data32 data32 nopw %cs:0x0(%rax,%rax,1)
   0x0000000000401011 <+49>:    66 66 66 66 66 66 2e 0f 1f 84 00 00 00 00 00    data32 data32 data32 data32 data32 nopw %cs:0x0(%rax,%rax,1)
   0x0000000000401020 <+64>:    89 d7   mov    %edx,%edi
   0x0000000000401022 <+66>:    83 c2 20        add    $0x20,%edx
   0x0000000000401025 <+69>:    44 0f b6 04 3e  movzbl (%rsi,%rdi,1),%r8d
   0x000000000040102a <+74>:    4a 8b 3c c9     mov    (%rcx,%r9,8),%rdi
   0x000000000040102e <+78>:    44 88 04 07     mov    %r8b,(%rdi,%rax,1)
   0x0000000000401032 <+82>:    48 83 c0 01     add    $0x1,%rax
   0x0000000000401036 <+86>:    48 83 f8 40     cmp    $0x40,%rax
   0x000000000040103a <+90>:    75 e4   jne    0x401020 <Dst_First_3::demux(unsigned char const*, unsigned int, unsigned char**) const+64>
   0x000000000040103c <+92>:    49 83 c1 01     add    $0x1,%r9
   0x0000000000401040 <+96>:    49 83 f9 20     cmp    $0x20,%r9
   0x0000000000401044 <+100>:   75 ba   jne    0x401000 <Dst_First_3::demux(unsigned char const*, unsigned int, unsigned char**) const+32>
   0x0000000000401046 <+102>:   f3 c3   repz retq 
End of assembler dump.
{% endhighlight %}

The inner loop starts at 0x401020. It is preceded by two `nopw` instructions (`0f 1f 84 00 00 00 00 00`). Each of them
has one `cs:` prefix (`2e`). One has three `data32` prefixes (66), another one has five. In total these two
instructions occupy 27 bytes: one takes 15 bytes and another one takes 12 bytes. These instructions don't
cause emitting any micro-operations, but they must still be fetched and decoded, and, as we've seen, fetching and decoding
can be a bottleneck. If a loop executes just a few times, the penalty of entering the loop through the alignment code
may be higher than the benefit of aligning the loop.

This is why "align your loops" isn't a universal advice, and "align your loops by 32" is even less universal. This
is a point whene **Java** has advantage over **C**: the **Java** VM knows which loops are hot and which are not; in
addition it knows the exact model of the processor the program runs on. **C** compilers don't have such a luxury.

By the way, the outer loop of our example is aligned as well, using even more bytes (29).

Other types of alignment
------------------------

GNU C has two more switches controlling alignment:

- `-falign-jumps` aligns jump targets, that is the locations in code that receive control by jumps only and never
  by fall-through. The cost involved is relatively small -- as with aligning functions, it is limited to the
  problems associated with bigger code size. It might be a good idea to set this to 32 for Sandy Bridge (not for
  our program though, since it does not have such jumps)

- `-falign-labels` aligns any labels, that is the locations in code that receive control by both jumps and
  fall-through, but which are not necesserily starting points of loops. This can help for branches that execute
  frequently (when there is an `if` statement inside a hot loop), but it can slow down other cases; that's why
  this option must be used carefully.

Probably, the best way of using these options is to use them on specific files (by creating custom rules in
`make` files), or even on specific functions (GNU C allows this by means of function attributes).

So what to do?
--------------

The **C/C++** developers have really hard time tuning their programs for target systems. Not only alignment is
specific for processor models: the entire code generation process is specific. For instance, in the old days the best way to
increment a variable by 1 was by using an `inc` instruction. These days `add 1` is usually better, because `inc` causes
a partial flag stall. In addition, processors differ in their support for the new features: `AVX`, `FMA`, `popcnt`
instruction and so on. The programmers have limited options to choose from:

- Compile for "generic" processor and make peace with the fact that the speed won't always be the highest,

- Find out which processors the customers are going to use and tune the program for these processors; do not
  forget to re-tune when the customers upgrade their hardware,

- Pre-compile the program for several processors and chose the version to run dynamically based on processor
  detection; don't forget to expand the set of available versions as new hardware arrives.
  This is exactly what Intel does with their
  [IPP](http://en.wikipedia.org/wiki/Integrated_Performance_Primitives) library,

- Use standard libraries such as mentioned IPP for performance-critical computations and only write the "glue"
  code yourself. Obviously, this has limited applicability.

This really looks as a very poor choice. Fortunately, the problem is usually limited to the most
performance-critical part of the system, and this part is usually small. Most code in production systems is
the "glue" code (configuration, user interaction, health monitoring, etc.), which has very relaxed performance
requirements.

Back to the original program
----------------------------

Let's get back to our original `e1.cpp` and compile it with the alignment options:

    $ c++ -O3 -falign-functions=32 -falign-loops=32 -o e1 e1.cpp -lrt
    ./e1
    9Reference: 1932
    11Src_First_1: 1488
    11Src_First_2: 1338
    11Src_First_3: 1891
    11Dst_First_1: 1455
    11Dst_First_2: 1454
    11Dst_First_3: 1464
    10Unrolled_1: 634
    12Unrolled_1_2: 634
    12Unrolled_1_4: 654
    12Unrolled_1_8: 650
    13Unrolled_1_16: 634
    15Unrolled_2_Full: 635
    10Unrolled_3: 633
    10Unrolled_4: 653

The results look much more stable than before.
[Now we can remove `Unrolled_3` and `Unrolled4`](https://github.com/pzemtsov/article-E1-demux-C/commit/ed29557be68e178600e7f2139330bcbdd724fe9f),
which were found unnecessary in [this article](http://pzemtsov.github.io/2014/05/06/from-macros-to-templates.html).
I tried to do it then but faced exactly the same instability problem with `Dst_First_1`.

    $ c++ -O3  -falign-functions=32 -falign-loops=32 -o e1 e1.cpp -lrt
    9Reference: 1939
    11Src_First_1: 1488
    11Src_First_2: 1341
    11Src_First_3: 1891
    11Dst_First_1: 1457
    11Dst_First_2: 1454
    11Dst_First_3: 1464
    10Unrolled_1: 636
    12Unrolled_1_2: 634
    12Unrolled_1_4: 655
    12Unrolled_1_8: 650
    13Unrolled_1_16: 635
    15Unrolled_2_Full: 635

Let's collect the old and the new results, together with the results from **Java**, in one table:

<table>
<tr><th>     Version              </th> <th>  Results in Java </th> <th> Old results in C</th><th> New results in C</th></tr>
<tr><td><pre>Reference      </pre></td> <td><pre>  2860 </pre></td> <td><pre> 1939</pre></td> <td><pre> 1939</pre></td></tr>
<tr><td><pre>Src_First_1    </pre></td> <td><pre>  2481 </pre></td> <td><pre> 1885</pre></td> <td><pre> 1488</pre></td></tr>
<tr><td><pre>Src_First_2    </pre></td> <td><pre>  2284 </pre></td> <td><pre> 1924</pre></td> <td><pre> 1341</pre></td></tr>
<tr><td><pre>Src_First_3    </pre></td> <td><pre>  4360 </pre></td> <td><pre> 1892</pre></td> <td><pre> 1891</pre></td></tr>
<tr><td><pre>Dst_First_1    </pre></td> <td><pre>  1155 </pre></td> <td><pre> 1467</pre></td> <td><pre> 1457</pre></td></tr>
<tr><td><pre>Dst_First_2    </pre></td> <td><pre>  2093 </pre></td> <td><pre> 1445</pre></td> <td><pre> 1454</pre></td></tr>
<tr><td><pre>Dst_First_3    </pre></td> <td><pre>  1022 </pre></td> <td><pre> 1761</pre></td> <td><pre> 1464</pre></td></tr>
<tr><td><pre>Unrolled_1     </pre></td> <td><pre>   659 </pre></td> <td><pre>  633</pre></td> <td><pre>  636</pre></td></tr>
<tr><td><pre>Unrolled_1_2   </pre></td> <td><pre>   654 </pre></td> <td><pre>  634</pre></td> <td><pre>  634</pre></td></tr>
<tr><td><pre>Unrolled_1_4   </pre></td> <td><pre>   636 </pre></td> <td><pre>  654</pre></td> <td><pre>  655</pre></td></tr>
<tr><td><pre>Unrolled_1_8   </pre></td> <td><pre>   637 </pre></td> <td><pre>  650</pre></td> <td><pre>  650</pre></td></tr>
<tr><td><pre>Unrolled_1_16  </pre></td> <td><pre> 25904 </pre></td> <td><pre>  635</pre></td> <td><pre>  635</pre></td></tr>
<tr><td><pre>Unrolled_2_Full</pre></td> <td><pre> 15630 </pre></td> <td><pre>  635</pre></td> <td><pre>  635</pre></td></tr>
</table>

Some observations:

- In general, it became better. No version got slower, some (all the `Src_First` and `Dst_First_3`) became much faster.

- Unlike in **Java**, `Src_First` version are faster than `Dst_First`

- `Src_First` run faster than in **Java**, while `Dst_First` run slower

- `Dst_First_3` is still slightly slower than `Dst_First_1`. The difference, however, is small and can probably
  be attributed to some unlucky combination of instruction sizes of execution unit allocation.
- This fact also shows that hard-coded loop sizes do not necessarily cause faster execution -- at least, without
  extra optimisations such as loop unrolling

- `Dst_First_2` was first written in **Java** as a hand-optimised version of `Dst_First_1`. In **Java** it was slower
  than the unoptimised version; in **C** it runs at the same speed. It indicates that, while manual optimisation
  in **C** isn't as harmful as in **Java**, it isn't very helpful, either -- or, at least, that additional effort
  is required to make it helpful.

Conclusions
-----------

- We've just gone through very enthralling detective story. The detective team succeeded this time;
  the culprit was found and neutralised.

- Our first find was not completely correct; it is very typical for detective stories.

- Function alignment is very important for speed measurement stability.

- Loop alignment can be crucial for program performance.

- Other alignment options may be useful, too.

- However, any alignment control must be used carefully, for it can also decrease performance.

- Programs in **C++** and other statically compiled languages must be tuned for specific processor models.
  Such tuning may give better yields than many sophisticated optimisations.

- Default function and loop alignment used for 64-bit Intel platforms in GCC 4.6 is 16; the one required for Sandy
  Bridge is 32.

- Dynamic compilation, such as the one employed by the HotSpot (a **Java** VM), allows the compiler to tune the
  program for you.

Coming soon
-----------

Some versions in **C++** are still slower than in **Java** . Why is that, and what can be done? Wait
for new articles.
