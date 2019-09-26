---
layout: post
title:  "Making a char searcher in C"
date:   2019-09-26 12:00:00
tags: C optimisation search
---

In the previous article (["{{ site.TITLE-INDEXOF }}"]({{ site.ART-INDEXOF }}))
we studied various methods to search for a byte sequence in another byte sequence in **Java**.
We want to try the same in **C++**. All of our **Java** approaches are well applicable in
**C++**, but the common idea is that you can do better in **C++**. Not because **C++** compilers are
better (this days this is a hot debatable question), but because **C++** allows better access to
the computer's low level operations, such as direct in-line assembly programming.

Specifically, one of the options we have in mind is to find the first character of the pattern
using some techniques not available in **Java**, and then compare the patterns.
If this can be performed very fast, we can forget about our fancy search methods, such as
shift tables and search trees.

So, today we'll be studying very narrow technical question: how fast can we find a single character
in a long character array? The issue looks very simple; some of the solutions that we'll be looking at,
however, are not simple at all.

Obviously, there are readily available library functions for the job, which we are unlikely to outperform --
they are usually well-designed and highly optimised. We'll measure them, too, and check how well
our DIY implementations approach them.

Just for a difference, we'll write our tests in **C**, or, more precisely, **C99**.

The code is available [in this repository]({{ site.REPO-CHAR-SEARCHER }}).

The test
--------

The library function in **C** to search for a character in a memory block looks like this:

{% highlight C %}
void* memchr (const void* ptr, int ch, size_t count);
{% endhighlight %}

Here `count` is the number of bytes to look at. The function returns the position of the character if found, or `NULL` if not. We'll follow the same
signature. Our searchers will satisfy the type

{% highlight C %}
typedef void* Searcher (const void* ptr, int ch, size_t count);
{% endhighlight %}

We'll allocate a big enough block, fill it with some non-matching data, and run several tests where
the char to search will be placed at various distances from the start. The measured value will be
the time it took to search, in nanoseconds per byte searched.

One important variable in a test like this is data alignment. It affects data reading using all
techniques, except for single byte access. It is especially important for SSE, where
most of instructions require pointer alignment by 16. That's why we'll run an alignment-averaging test:
we'll allocate our array at a 64-byte boundary (using `_mm_alloc`), and run our tests with alignments
0 to 63, averaging the results.

Another variable is the exact choice of the text length. Some solutions involve loop unrolling
or multi-byte operations and may be sensitive to the alignment of this length. That's why we'll
run tests with multiple lengths around the published central value, and average the results.

Strictly speaking, another variable is the size of the entire array: if we place the required byte
at the position 4, it may be important if we call `memchr` with the total length of 8 or 8192;
however, we'll ignore this for now and always call the function with some big number.

The null test
-------------

First, we want to know the cost of our measurement framework. We'll use a null searcher for that:

{% highlight C %}
void* null_index(const void* ptr, int ch, size_t count)
{
    return NULL;
}
{% endhighlight %}

Here are the times (the column headers are the numbers of bytes searched):

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">Null                </td><td>0.63</td><td>0.14</td><td>0.035</td><td>0.0086</td><td>0.0022</td><td>0.0005</td><td>0.0001</td></tr>
</table>

The simple index-based searcher
-------------------------------

This is, probably, the simplest solution one can think of:

{% highlight C %}
void* simple_index (const void* ptr, int ch, size_t count)
{
    const char * p = (const char *) ptr;
    for (size_t i = 0; i < count; i ++) {
        if (p [i] == (char) ch) return (void*) (p + i);
    }
    return NULL;
}
{% endhighlight %}

Here are the times:

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">Null                </td><td>0.63</td><td>0.14</td><td>0.035     </td><td>0.0086          </td><td>0.0022          </td><td>0.0005          </td><td>0.0001          </td></tr>
<tr><td class="ttext">Simple              </td><td>1.24</td><td>0.61</td><td>0.47&nbsp;</td><td>0.44&nbsp;&nbsp;</td><td>0.44&nbsp;&nbsp;</td><td>0.43&nbsp;&nbsp;</td><td>0.42&nbsp;&nbsp;</td></tr>
</table>

As expected, the test framework is introducing significant overhead on small arrays, and has virtually 
no influence on big ones.

Here is the percentage of time the test framework takes:

<table class="numeric">
<tr><th> Fraction </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">Test code</td><td>50.8%</td> <td>23.0%</td><td>7.4%</td><td>2.0%</td><td>0.5%</td><td>0.1%</td><td>0.02%</td></tr>
</table>

We'll have to keep this overhead in mind when working with small input sizes.

In absolute terms, the results don't look great: 0.42 ns is about one CPU cycle. Surely we must
be able to do better than this.

The pointer-based simple searcher
---------------------------------

This days, using indices is considered normal, due to the modern processor design, which makes indexing
cheap, and to sophisticated compilers, which can convert indexing to pointer manipulations and other way
around, whichever is the best for the target architecture.

In the old days, however, the pointer arithmetic was praised as the winning optimisation technique
and one of the main selling features of a language like **C** as compared to something like
**Pascal**, **Ada**, **Algol-68** or **PL/1** (today, **Java** would top this list).

Let's try using pointer arithmetic and see if it makes any difference:

{% highlight C %}
void* simple_ptr(const void* ptr, int ch, size_t count)
{
    const char * p = (const char *)ptr;
    while (count) {
        if (*p == (char) ch) return (void*) p;
        ++p;
        --count;
    }
    return NULL;
}
{% endhighlight %}

And here are the results:

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">Simple              </td><td>1.24</td><td>0.61</td><td>0.47</td><td>0.44</td><td>0.44</td><td>0.43</td><td>0.42</td></tr>
<tr><td class="ttext">Simple Pointer      </td><td>1.20</td><td>0.61</td><td>0.47</td><td>0.43</td><td>0.43</td><td>0.42</td><td>0.42</td></tr>
</table>

There is no difference outside of the measurement error margin. Let's try another approach:

{% highlight C %}
void* simple_ptr2(const void* ptr, int ch, size_t count)
{
    const char * p = (const char *)ptr;
    const char * q = p + count;
    for (; p != q; p++) {
        if (*p == (char)ch) return (void*)p;
    }
    return NULL;
}
{% endhighlight %}

The results are a bit better on small sizes:

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">Simple Pointer      </td><td>1.20</td><td>0.61</td><td>0.47</td><td>0.43</td><td>0.43</td><td>0.42</td><td>0.42</td></tr>
<tr><td class="ttext">Simple Pointer v2   </td><td>1.17</td><td>0.53</td><td>0.43</td><td>0.41</td><td>0.42</td><td>0.41</td><td>0.41</td></tr>
</table>

The improvement, however, isn't worth bothering. It is really a matter of taste which approach to choose.

The standard searcher
---------------------

Now, having tested the simplest approaches, let's see what the standard library has to offer.
We'll measure `memchr`:

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">Simple              </td><td>1.24</td><td>0.61</td><td>0.47</td><td>0.44&nbsp;</td><td>0.44&nbsp;</td><td>0.43&nbsp;</td><td>0.42&nbsp;</td></tr>
<tr><td class="ttext">memchr              </td><td>1.25</td><td>0.32</td><td>0.10</td><td>0.063     </td><td>0.035     </td><td>0.031     </td><td>0.027     </td></tr>
</table>
                                                                                                                                         
The results are much, much better. The speedup grows with array size, eventually reaching 
15 times. The absolute result is also impressive: our processor runs at 2.4 GHz,
so 0.027 ns/byte means 0.065 cycles/byte, or 15.4 bytes per cycle. This looks so fast that it's unlikely
to be beaten, no matter what we do. It also suggests that the SSE is involved, so we'll try it later.

How close can we come to these results with home-made searchers?

The SCAS-based searcher
-----------------------

Since the Day One of Intel x86 family of processors (namely, since 8086), there was a string search
instruction, `SCAS`. It compared `al` to the byte pointed to by `[edi]`, incremented
that pointer and decremented `cx`, making it suitable to use with the `REP` prefix.

There are rumours that these instructions are not so efficient anymore. However, there are cases
when `REP MOVS` is still the most efficient way to copy a block of bytes (although, even there it is
`MOVSQ`, rather than `MOVSB`). Let's measure. We'll write the function straight in assembly:


{% highlight asm %}
        .p2align 5,,31
        .globl	index_scas
        .type	index_scas, @function

# const void* index_scas(const void* ptr, int ch, size_t count)
# RDI = ptr
# RSI = ch
# RDX = count

index_scas:
        .cfi_startproc
        mov     %esi, %eax
        mov     %rdx, %rcx
        repne scasb
        jnz     not_found
        lea     -1(%rdi), %rax
        ret
not_found:
        xorq    %rax, %rax
        ret
	.cfi_endproc
{% endhighlight %}

This can, probably, be improved by replacing the conditional jump with `cmove` or `setz`,
but this instruction is only executed once after the end of the search. It can only affect
performance on very short repetition counts. Here are the results we get:

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">Simple              </td><td>1.24</td><td>0.61</td><td>0.47</td><td>0.44</td><td>0.44</td><td>0.43</td><td>0.42</td></tr>
<tr><td class="ttext">SCAS                </td><td>7.12</td><td>2.26</td><td>1.19</td><td>0.93</td><td>0.86</td><td>0.85</td><td>0.85</td></tr>
</table>

The solution is really slow, much slower even than the simple loop. It gets a bit faster on longer
arrays, but it's unlikely to ever catch up. This probably means that the rumours were correct,
and we can forget about the `SCAS`-based solutions. This instruction is there as a 40-years old
legacy and is not intended for any practical use today. So enjoy your retirement, good old `SCAS`.

The simple DWORD searcher
-------------------------

This is a variation of the Simple Pointer searcher, reading four bytes from memory at a time instead of one:

{% highlight C %}
void* simple_dword(const void* ptr, int ch, size_t count)
{
    const char * p = (const char *)ptr;
    while (count >= 4) {
        uint32_t v;
        memcpy(&v, p, 4);
        for (size_t i = 0; i < 4; i++) {
            if ((char)v == (char)ch) return (void *)(p + i);
            v >>= 8;
        }
        p += 4;
        count -= 4;
    }
    while (count) {
        if (*p == ch) return (void*)p;
        ++p;
        --count;
    }
    return NULL;
}
{% endhighlight %}

This version is x86-specific. It doesn't put any alignment requirements on the input (reading
data using `memcpy` takes care of that, see
 ["{{ site.TITLE-ALIGNMENT-BUG }}"]({{ site.ART-ALIGNMENT-BUG }})).
However, it makes use of the little-endian nature of the x86. Some additional checks for endianness
are nesessary if we ever want to convert it into an industrial solution, and it's not going to be
completely portable, for neither `C99`, nor `C++` has good built-in endianness support.

Let's check the speed:

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">Simple              </td><td>1.24</td><td>0.61</td><td>0.47</td><td>0.44</td><td>0.44</td><td>0.43</td><td>0.42</td></tr>
<tr><td class="ttext">Simple dword        </td><td>0.99</td><td>0.52</td><td>0.42</td><td>0.40</td><td>0.41</td><td>0.39</td><td>0.39</td></tr>
</table>

The speed is a bit higher (10-20%). This shows that even reading from well-cached memory isn't
as fast as accessing the registers. However, this improvement is clearly not sufficient.

Note that this solution (and most of the solutions that follow) is very strict regarding its memory access.
It never reads a single byte past the memory area it was called to search in (or, for that matter,
before that area -- we'll see later that this can be useful, too). If we could be certain
that there are always a few bytes available past the end, we could avoid the trailing loop. This, however,
would only make a visible difference on very short strings.

The DWORD searcher with zeroes detection
----------------------------------------

In the previous solution we read a double-word (32 bits) from memory and then fetched the bytes
one by one from that double-word to find a match. We could save if we could quickly establish if
there was any match in this double-word.

Let's first make a mask containing the byte we are searching for in its every byte, and `XOR` with
that mask:

{% highlight C %}
    uint32_t value = ch * 0x01010101;
    uint32_t v;
    memcpy(&v, p, 8);
    v ^= value;
{% endhighlight %}

Now we need to find out if any byte in `v` is zero. The answer comes from the famous
[Bit Twiddling Hacks](https://graphics.stanford.edu/~seander/bithacks.html#ZeroInWord) page.

This page offers two solutions. The first one works like this:

- first `AND` each byte with `0x7F`;
- this allows adding `0x7F` to resulting bytes without producing carry bits;
- the high bit of the result will be set iff any of the lower 7 bits of `v` were non-zero;
- we `OR` it with the source (`v`). Now the high bit is 1 iff the corresponding byte in `v` is non-zero;
- finally, leave only the high bit and invert it;
- now bytes that were zero in `v` have value `0x80` and all the other bytes are zeroes. The entire
doubleword is non-zero when `v` contains a zero byte.

Here is the formula, which uses five operations:

{% highlight C %}
#define hasZeroByte(v) ~(((((v) & 0x7F7F7F7F) + 0x7F7F7F7F) | (v)) | 0x7F7F7F7F);
{% endhighlight %}

The second solution is less accurate but uses only four operations:

{% highlight C %}
#define hasZeroByte2(v) (((v) - 0x01010101) & ~(v) & 0x80808080)
{% endhighlight %}

Again, we treat the high bit of every byte separately. The `~(v) & 0x80808080` portion
requires it to be zero to return `0x80`. The `(v) - 0x01010101` part subtracts one from every
byte, which sets its high bit to one in three cases:

- it was one before
- the whole byte was zero
- the byte was one and there was a borrow bit from the previous byte.

In the last two cases a borrow bit is set.

The borrow bits can distort the picture, but for them to appear in the first place, a zero byte must
have occured somewhere before. This means that this method also allows to establish if the source has zero bytes
but does not show where they are. Here are some examples:

<table class="numeric">
<tr><th> v </th><th>hasZeroByte</th><th>hasZeroByte2</th></tr>
<tr><td>00000000</td><td>80808080</td><td> 80808080</td></tr>
<tr><td>10100020</td><td>00008000</td><td> 00008000 </td></tr>
<tr><td>01010100</td><td>00000080</td><td> 80808080 </td></tr>
<tr><td>01000001</td><td>00808000</td><td> 80808000 </td></tr>
<tr><td>01010001</td><td>00008000</td><td> 80808000 </td></tr>
</table>

The second method is sufficient for our purpose, so our next solution will be based on it:

{% highlight C %}
void* dword(const void* ptr, int ch, size_t count)
{
    const char * p = (const char *)ptr;
    uint32_t value = ch * 0x01010101L;
    while (count >= 8) {
        uint32_t v;
        memcpy(&v, p, 8);
        v ^= value;
        if ((v - 0x01010101L) & ~v & 0x80808080L) {
            for (size_t i = 0; i < 4; i++) {
                if (!(char)v) return (void *)(p + i);
                v >>= 8;
            }
        }
        p += 4;
        count -= 4;
    }
    while (count) {
        if (*p == ch) return (void*)p;
        ++p;
        --count;
    }
    return NULL;
}
{% endhighlight %}

The results, however, haven't improved:

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">Simple              </td><td>1.24</td><td>0.61</td><td>0.47</td><td>0.44</td><td>0.44</td><td>0.43</td><td>0.42</td></tr>
<tr><td class="ttext">Simple dword        </td><td>0.99</td><td>0.52</td><td>0.42</td><td>0.40</td><td>0.41</td><td>0.39</td><td>0.39</td></tr>
<tr><td class="ttext">Dword               </td><td>1.51</td><td>0.66</td><td>0.48</td><td>0.45</td><td>0.39</td><td>0.38</td><td>0.38</td></tr>
</table>

The BSF Dword searcher
----------------------

When we finally find a doubleword that contains our byte, is there any way to locate this byte
quickly, without running a loop? The methods described above produce 0x80 in the bytes where
zeroes were. The first one writes 0x00 in all the other bytes, while the second one may fill some
of the others with 0x80 as well. This, however, requires a borrow bit to be generated, which only
happens to the left (that is, on the higher side) of the zero byte. This means that on
little-endian processors both methods are suitable to locate the first zero byte. All that's needed
is to find the least significant non-zero byte in the computed value. The Bit Twiddling Hacks has
[code for that, too](https://graphics.stanford.edu/~seander/bithacks.html#ZerosOnRightLinear),
but these days it is unnecessary, since we have an efficient CPU instruction `BSF` (**bit scan forward**).
It was not always fast, but now it is. It returns the position of the rightmost (least significant) bit set in a number.

{% highlight C %}
void* dword_bsf(const void* ptr, int ch, size_t count)
{
    const char * p = (const char *)ptr;
    uint32_t value = ch * 0x01010101L;
    while (count >= 4) {
        uint32_t v;
        memcpy(&v, p, 8);
        v ^= value;
        uint32_t bits = (v - 0x01010101L) & ~v & 0x80808080L;
        if (bits) {
            return (void *)(p + (bsf(bits) / 8));
        }
        p += 4;
        count -= 4;
    }
    while (count) {
        if (*p == ch) return (void*)p;
        ++p;
        --count;
    }
    return NULL;
}
{% endhighlight %}

Here `bsf()` is our own wrapper that takes care of different intrinsics in different compilers:

{% highlight C %}
size_t bsf(uint32_t bits)
{
#ifdef __GNUC__
    return __builtin_ctz(bits);
#else
#ifdef _MSC_VER
    unsigned long index;
    _BitScanForward(&index, bits);
    return index;
#else
#error Undefined BSF
#endif
#endif
}
{% endhighlight %}

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">Simple dword        </td><td>0.99</td><td>0.52</td><td>0.42</td><td>0.40</td><td>0.41</td><td>0.39</td><td>0.39</td></tr>
<tr><td class="ttext">Dword               </td><td>1.51</td><td>0.66</td><td>0.48</td><td>0.45</td><td>0.39</td><td>0.38</td><td>0.38</td></tr>
<tr><td class="ttext">Dword BSF           </td><td>1.28</td><td>0.58</td><td>0.40</td><td>0.36</td><td>0.32</td><td>0.29</td><td>0.29</td></tr>
</table>

This trick helped, except for very small sizes.

Quadword searchers
------------------

On a 64-bit architecture we can do the same using 64-bit words (qwords). The code looks pretty much the same,
so we'll only show one piece here -- that of `qword_bsf`:

{% highlight C %}
void* qword_bsf(const void* ptr, int ch, size_t count)
{
    const char * p = (const char *)ptr;
    uint64_t value = ch * 0x0101010101010101LL;
    while (count >= 8) {
        uint64_t v;
        memcpy(&v, p, 8);
        v ^= value;
        uint64_t bits = (v - 0x01010101010101010ULL)
                        & ~v
                        & 0x8080808080808080LL;
        if (bits) {
            return (void *)(p + (bsf_64(bits) / 8));
        }
        p += 8;
        count -= 8;
    }
    return dword_bsf(p, ch, count);
}
{% endhighlight %}

This code makes use of the `dword_bsf` implementation for leftovers (when there are less than 8 bytes left),
so it will benefit from a good inline engine, if present in the compiler.

Here are the results:

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">Simple qword        </td><td>0.93</td><td>0.46</td><td>0.42</td><td>0.41</td><td>0.42</td><td>0.41</td><td>0.41</td></tr>
<tr><td class="ttext">Qword               </td><td>1.59</td><td>0.49</td><td>0.24</td><td>0.19</td><td>0.16</td><td>0.15</td><td>0.15</td></tr>
<tr><td class="ttext">Qword BSF           </td><td>1.23</td><td>0.42</td><td>0.19</td><td>0.14</td><td>0.13</td><td>0.13</td><td>0.13</td></tr>
</table>

The simple qword is running pretty much at the same speed as the simple dword, if not slower.
However, the other two show good improvement, especially the BSF version. So wider memory access plus
wider operations really help. This makes it attractive to try SSE.

The SSE searcher
----------------

The SSE (starting from SSE2) can operate on integer components of various sizes, including individual bytes.
We can thus load 16 bytes into an SSE register and compare them all with the character we search.
The compare instruction (`PCMPEQB`) sets corresponding bytes in the result to 0xFF if equality
is detected and to 0 if not. To locate the first equal byte, we apply `PMOVMSKB`, which sets bits
in a normal register to one or zero depending on the high bit of each byte, and then apply `BSF` to
that register.

The endianness works just right: the byte in an XMM register that is the first in memory, produces
the lowest bit, for which `BSF` will return zero.

A comment must be made on data alignment. Generally, SSE prefers 16-byte alignment. All operations
with operands in memory and all normal loads (`MOVAPS`, `MOVDQA`) require this alignment, and only special
unaligned loads (`MOVUPS`, `MOVDQU`) can do without it. In older generation of processors,
`MOVUPS` was much slower than `MOVAPS`, even if the data was aligned. It's not like that anymore,
with one exception of crossing cache line boundaries: this is still a bit slow. In this version we'll
go for simplicity and use unaligned loads:

{% highlight C %}
void* sse(const void* ptr, int ch, size_t count)
{
    const char * p = (const char *)ptr;
    __m128i value = _mm_set1_epi8((char)ch);
    while (count >= 16) {
        __m128i v = _mm_loadu_si128((__m128i *) p);
        int eq_mask = _mm_movemask_epi8(_mm_cmpeq_epi8(v, value));
        if (eq_mask) {
            return (void *)(p + bsf(eq_mask));
        }
        p += 16;
        count -= 16;
    }
    return qword_bsf(p, ch, count);
}
{% endhighlight %}

This function reverts to `qword_bsf` for leftovers.

Here are the times:

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">Qword BSF           </td><td>1.23</td><td>0.42</td><td>0.19</td><td>0.14&nbsp;</td><td>0.13&nbsp;</td><td>0.13&nbsp;</td><td>0.13&nbsp;</td></tr>
<tr><td class="ttext">memchr              </td><td>1.25</td><td>0.32</td><td>0.10</td><td>0.063     </td><td>0.035     </td><td>0.031     </td><td>0.027     </td></tr>
<tr><td class="ttext">sse                 </td><td>1.01</td><td>0.28</td><td>0.11</td><td>0.065     </td><td>0.044     </td><td>0.041     </td><td>0.040     </td></tr>
</table>

The times look very good and approach those of `memchr` (beating it on small sizes).

The assembly code of this function is surprisingly long, due to GCC's loop unrolling. It doesn't contain
calls to other functions, which means that the inline engine performed well.

The unlimited SSE searcher 
--------------------------

All functions shown above are written conservatively with regards to the upper boundary of the
source array. They don't access any memory beyond that boundary. If we know that our RAM does not end
there, and some bytes are available for reading after the end pointer, we can save on complexity:

{% highlight C %}
void* sse_unlimited(const void* ptr, int ch, size_t count)
{
    const char * p = (const char *)ptr;
    __m128i value = _mm_set1_epi8((char)ch);
    for (ptrdiff_t cnt = (ptrdiff_t)count; cnt > 0; cnt -= 16, p += 16) {
        __m128i v = _mm_loadu_si128((__m128i *) p);
        int eq_mask = _mm_movemask_epi8(_mm_cmpeq_epi8(v, value));
        if (eq_mask) {
            size_t offset = bsf(eq_mask);
            return offset >= (size_t) cnt ? NULL : (void *)(p + offset);
        }
    }
    return NULL;
}
{% endhighlight %}

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">sse                 </td><td>1.01</td><td>0.28</td><td>0.11</td><td>0.065</td><td>0.044</td><td>0.041</td><td>0.040</td></tr>
<tr><td class="ttext">sse_unlimited       </td><td>0.88</td><td>0.26</td><td>0.11</td><td>0.068</td><td>0.050</td><td>0.049</td><td>0.048</td></tr>
</table>

Except for very small arrays, this version didn't produce any significant improvement, and it's not quite correct as far as **C**
standard goes, so we won't use it in practice. It is still unclear, however, why this happened.

Aligned SSE version
-------------------

As discussed before, aligned access to memory in SSE operations can potentially be faster than unaligned one,
due to better code generation (wider choice of instructions), lack of cache boundary crossing, and,
sometimes, naturally (as a feature of the processor). Let's try to align the pointer before running
the main loop. We'll use the unaligned version for the first loop iteration:

{% highlight C %}
void* sse_aligned(const void* ptr, int ch, size_t count)
{
    if (count < 16) return qword_bsf(ptr, ch, count);
        
    const char * p = (const char *)ptr;
    __m128i value = _mm_set1_epi8((char)ch);

    size_t align = (size_t)p & 0x0F;
    if (align > 0) {
        align = 16 - align;
        __m128i v = _mm_loadu_si128((__m128i *) p);
        int eq_mask = _mm_movemask_epi8(_mm_cmpeq_epi8(v, value));
        if (eq_mask) {
            return (void *)(p + bsf(eq_mask));
        }
        p += align;
        count -= align;
    }
    while (count >= 16) {
        __m128i v = _mm_load_si128((__m128i *) p);
        int eq_mask = _mm_movemask_epi8(_mm_cmpeq_epi8(v, value));
        if (eq_mask) {
            return (void *)(p + bsf(eq_mask));
        }
        p += 16;
        count -= 16;
    }
    return qword_bsf(p, ch, count);
}
{% endhighlight %}

There is some improvement, although not very big:

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">sse                 </td><td>1.01</td><td>0.28</td><td>0.11</td><td>0.065</td><td>0.044</td><td>0.041</td><td>0.040</td></tr>
<tr><td class="ttext">sse_aligned         </td><td>1.00</td><td>0.27</td><td>0.13</td><td>0.062</td><td>0.043</td><td>0.039</td><td>0.036</td></tr>
</table>

Chasing the standard library: unrolling the loop
------------------------------------------------

The standard library's function `memchr` is still faster, despite all our effort. What do they do
there that makes it so fast?

To see the code of `memchr`, we run our test program in `gdb` and print the disassembly of this
function (we could instead download the code from GCC code repository, but it wouldn't make much
difference, since the code is written in assembly). The code is rather long
(see [here]({{ site.REPO-CHAR-SEARCHER }}/blob/master/memchr.s)),
and generally follows the pattern of our aligned SSE solution, but it contains a portion that looks
particularily interesting:

{% highlight asm %}
   <+410>:   movdqa xmm0,XMMWORD PTR [rdi]
   <+414>:   movdqa xmm2,XMMWORD PTR [rdi+0x10]
   <+419>:   movdqa xmm3,XMMWORD PTR [rdi+0x20]
   <+424>:   movdqa xmm4,XMMWORD PTR [rdi+0x30]
   <+429>:   pcmpeqb xmm0,xmm1
   <+433>:   pcmpeqb xmm2,xmm1
   <+437>:   pcmpeqb xmm3,xmm1
   <+441>:   pcmpeqb xmm4,xmm1
   <+445>:   pmaxub xmm3,xmm0
   <+449>:   pmaxub xmm4,xmm2
   <+453>:   pmaxub xmm4,xmm3
   <+457>:   pmovmskb eax,xmm4
   <+461>:   add    rdi,0x40
   <+465>:   test   eax,eax
   <+467>:   je     0x7ffff7aa7920 <memchr+400>
{% endhighlight %}

The loop is unrolled four times: we load four SSE registers and perform comparison in parallel.
Then we combine four comparison results (as we remember, these are SSE values where 255 indicates
equal bytes and 0 means unequal), by performing a byte MAX function (`PMAXUB`). Frankly, bitwise
`OR` (`POR` instruction) seems more natural for this purpose, but `PMAXUB` also works, and there
is no performance difference.

The aligned access is also used (`MOVDQA` instruction).

Let's build a similar solution on top of our "Aligned SSE":

{% highlight C %}
void* sse_aligned64(const void* ptr, int ch, size_t count)
{
    if (count < 16) return qword_bsf(ptr, ch, count);

    const char * p = (const char *)ptr;
    __m128i value = _mm_set1_epi8((char)ch);

    size_t align = (size_t)p & 0x0F;
    if (align > 0) {
        align = 16 - align;
        __m128i v = _mm_loadu_si128((__m128i *) p);
        int eq_mask = _mm_movemask_epi8(_mm_cmpeq_epi8(v, value));
        if (eq_mask) {
            return (void *)(p + bsf(eq_mask));
        }
        p += align;
        count -= align;
    }
    while (count >= 64) {
        __m128i v0 = _mm_load_si128((__m128i *) p);
        __m128i v1 = _mm_load_si128((__m128i *) (p + 16));
        __m128i v2 = _mm_load_si128((__m128i *) (p + 32));
        __m128i v3 = _mm_load_si128((__m128i *) (p + 48));

        __m128i m0 = _mm_cmpeq_epi8(v0, value);
        __m128i m1 = _mm_cmpeq_epi8(v1, value);
        __m128i m2 = _mm_cmpeq_epi8(v2, value);
        __m128i m3 = _mm_cmpeq_epi8(v3, value);

        __m128i max_val = _mm_max_epu8(_mm_max_epu8(m0, m1),
                                       _mm_max_epu8(m2, m3));

        int eq_mask = _mm_movemask_epi8(max_val);
        if (eq_mask) {
            eq_mask = _mm_movemask_epi8(m0);
            if (eq_mask) {
                return (void *)(p + bsf(eq_mask));
            }
            eq_mask = _mm_movemask_epi8(m1);
            if (eq_mask) {
                return (void *)(p + bsf(eq_mask) + 16);
            }
            eq_mask = _mm_movemask_epi8(m2);
            if (eq_mask) {
                return (void *)(p + bsf(eq_mask) + 32);
            }
            eq_mask = _mm_movemask_epi8(m3);
            return (void *)(p + bsf(eq_mask) + 48);
        }
        p += 64;
        count -= 64;
    }
    while (count >= 16) {
        __m128i v = _mm_load_si128((__m128i *) p);
        int eq_mask = _mm_movemask_epi8(_mm_cmpeq_epi8(v, value));
        if (eq_mask) {
            return (void *)(p + bsf(eq_mask));
        }
        p += 16;
        count -= 16;
    }
    return qword_bsf(p, ch, count);
}
{% endhighlight %}

On long inputs this version is faster than our previous SSE-based solutions, but still doesn't
reach the `memchr` (except for length 256):

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">memchr              </td><td>1.25</td><td>0.32</td><td>0.10</td><td>0.063</td><td>0.035</td><td>0.031</td><td>0.027</td></tr>
<tr><td class="ttext">sse                 </td><td>1.01</td><td>0.28</td><td>0.11</td><td>0.065</td><td>0.044</td><td>0.041</td><td>0.040</td></tr>
<tr><td class="ttext">sse_aligned         </td><td>1.00</td><td>0.27</td><td>0.13</td><td>0.062</td><td>0.043</td><td>0.039</td><td>0.036</td></tr>
<tr><td class="ttext">sse_aligned64       </td><td>1.14</td><td>0.33</td><td>0.12</td><td>0.053</td><td>0.037</td><td>0.034</td><td>0.034</td></tr>
</table>


The AVX solutions
-----------------

All our SSE solutions can be easily converted to AVX. Let's try this.
We'll only show the simplest of the AVX solutions, the others can be seen in the repository:

{% highlight C %}
void* avx(const void* ptr, int ch, size_t count)
{
    const char * p = (const char *)ptr;
    __m256i value = _mm256_set1_epi8((char)ch);
    while (count >= 32) {
        __m256i v = _mm256_loadu_si256((__m256i *) p);
        int eq_mask = _mm256_movemask_epi8(_mm256_cmpeq_epi8(v, value));
        if (eq_mask) {
            return (void *)(p + bsf_64(eq_mask));
        }
        p += 32;
        count -= 32;
    }
    return sse(p, ch, count);
}
{% endhighlight %}

The results are quite a bit better than those of SSE, and generally better than the `memchr`.
This, however, mustn't make us very proud, since we were not using the very latest version of GNU C.
Surely, later versions contain a version of that library optimised for AVX.

Here are the times:

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">sse_aligned64       </td><td>1.14</td><td>0.33</td><td>0.12</td><td>0.053</td><td>0.037</td><td>0.034</td><td>0.034</td></tr>
<tr><td class="ttext">memchr              </td><td>1.25</td><td>0.32</td><td>0.10</td><td>0.063</td><td>0.035</td><td>0.031</td><td>0.027</td></tr>
<tr><td class="ttext">avx                 </td><td>1.18</td><td>0.27</td><td>0.10</td><td>0.045</td><td>0.028</td><td>0.024</td><td>0.023</td></tr>
<tr><td class="ttext">avx_aligned         </td><td>1.14</td><td>0.26</td><td>0.12</td><td>0.049</td><td>0.028</td><td>0.020</td><td>0.019</td></tr>
<tr><td class="ttext">avx_aligned64       </td><td>1.15</td><td>0.26</td><td>0.12</td><td>0.041</td><td>0.024</td><td>0.018</td><td>0.017</td></tr>
</table>

The standard library revisited
------------------------------

We managed to outperform the standard `memchr` using an unfair advantage (AVX), but haven't beaten it
using normal SSE code, which rises a question: what exactly is the magic in that code?

The only feature of that code that we paid attention to so far was the unrolled loop where it
tested four 16-byte blocks at a time. A deeper inspection of this code reveals two more features:

- the code does not contain any single-byte, four-byte or eight-byte operations; everything 
happens in SSE. No special treatment of leftovers is visible in this code;

- the code starts with a strange sequence:

<p></p>

{% highlight asm %}
   <+5>:     mov    rcx,rdi
             ...
   <+25>:    and    rcx,0x3f
   <+34>:    cmp    rcx,0x30
   <+38>:    ja     0x7ffff7aa7800 <memchr+112>
{% endhighlight %}

In other words, we check the alignment of the source pointer with respect to a 64-byte boundary.
Why do we do it?

The code happened to be more interesting and more complex than it looked at first. As it is written
in assembly, it can't be directly converted into **C**. The closest we could get to is shown here:

{% highlight C %}
#define TEST16_cmp_known_nz(p, cmp) \
    do {\
        int eq_mask = _mm_movemask_epi8(cmp);\
        return (void *)((p)+bsf(eq_mask));\
    } while (0)

#define TEST16_cmp(p, cmp) \
    do {\
        int eq_mask = _mm_movemask_epi8(cmp);\
        if (eq_mask) {\
            return (void *)((p)+bsf(eq_mask));\
        }\
    } while (0)

#define TEST16_v(p, v) TEST16_cmp (p, _mm_cmpeq_epi8(v, value))
#define TEST16(p) TEST16_v (p, _mm_load_si128((__m128i *) (p)))

#define TEST16_v_count(p, v) \
    do {\
        int eq_mask = _mm_movemask_epi8(_mm_cmpeq_epi8(v, value));\
        if (eq_mask) {\
            size_t pos = bsf(eq_mask);\
            if (pos >= count) return NULL;\
            return (void *)(p + pos);\
        }\
    } while (0)

#define TEST16_count(p) TEST16_v_count (p, _mm_load_si128((__m128i *) (p)))

void* sse_memchr(const void* ptr, int ch, size_t count)
{
    if (count == 0) return NULL;
    const char * p = (const char*)ptr;
    __m128i value = _mm_set1_epi8((char)ch);

    size_t align = (size_t) p & 0x3F;
    if (align <= 48) {
        TEST16_v_count(p, _mm_loadu_si128((__m128i *) p));
        if (count <= 16) return NULL;
        count -= 16;
        p += 16;
        align &= 0x0F;
        p -= align;
        count += align;
    } else {
        align &= 0x0F;
        p -= align;
        __m128i v = _mm_load_si128((__m128i *) p);
        unsigned eq_mask =
            (unsigned) _mm_movemask_epi8(_mm_cmpeq_epi8(v, value));
        eq_mask >>= align;
        if (eq_mask) {
            size_t pos = bsf(eq_mask);
            if (pos >= count) return NULL;
            return (void *)(p + pos + align);
        }
        count += align;
        if (count <= 16) return NULL;
        p += 16;
        count -= 16;
    }
    if (count > 64) {
        TEST16(p);
        TEST16(p + 16);
        TEST16(p + 32);
        TEST16(p + 48);
        p += 64;
        count -= 64;

        if (count > 64) {
            align = (size_t) p & 0x3F;
            if (align) {
                TEST16(p);
                TEST16(p + 16);
                TEST16(p + 32);
                TEST16(p + 48);
                p += 64;
                count -= 64;
                p -= align;
                count += align;
            }

            for (; count > 64; count -= 64, p += 64) {
                __m128i v0 = _mm_load_si128((__m128i *) p);
                __m128i v1 = _mm_load_si128((__m128i *) (p + 16));
                __m128i v2 = _mm_load_si128((__m128i *) (p + 32));
                __m128i v3 = _mm_load_si128((__m128i *) (p + 48));
                __m128i cmp0 = _mm_cmpeq_epi8(v0, value);
                __m128i cmp1 = _mm_cmpeq_epi8(v1, value);
                __m128i cmp2 = _mm_cmpeq_epi8(v2, value);
                __m128i cmp3 = _mm_cmpeq_epi8(v3, value);
                __m128i max_val = _mm_max_epu8(_mm_max_epu8(cmp0, cmp1),
                                               _mm_max_epu8(cmp2, cmp3));
                int eq_mask = _mm_movemask_epi8(max_val);
                if (eq_mask == 0) continue;

                TEST16_cmp(p, cmp0);
                TEST16_cmp(p + 16, cmp1);
                TEST16_cmp(p + 32, cmp2);
                TEST16_cmp_known_nz(p + 48, cmp3); // returns from function
            }
        }
    }
    if (count > 32) {
        TEST16(p);
        TEST16(p + 16);
        count -= 32;
        p += 32;
    }

    TEST16_count(p);
    if (count > 16); {
        count -= 16;
        TEST16_count(p + 16);
    }
    return NULL;
}
{% endhighlight %}

So what is actually going on here?

1) We check how the pointer is aligned against a 64-byte boundary. The idea here is that, since it is
a built-in function, and it is written in assembly anyway, it does not have to follow the **C**
standard strictly. Specifically, it makes use of the fact that the available memory does not start or end at
an address not aligned by 64 (actually, the memory boundary is always aligned by 4096, so perhaps
we could save there). So, if the alignment is 48 or less, there are definitely 16 bytes available,
so we can read them using `MOVDQU`. We must handle correctly the case when the string is shorter
than 16: that's why we return `NULL` if the result of `bsf` is bigger than `count`.

2) If the alignment is above 48, we read the 16-byte vector at alignment 48 (actually reading data
__before__ our pointer). Here we must deal with possible matches before the start (for that we
shift the `eq_mask` right by `align`), and after the end (for that we compare the result of `bsf()`
with `count`).

3) In both cases, unless the end of the input is detected, we end up at the next 16-byte boundary
(in the first case some of the bytes immediately after that boundary have already been tested,
so we are going to perform some redundant computations).

4) Then we test the next 64 bytes, one 16-byte block at a time, using aligned access.

5) If the match hasn't been found, we read enough 16-byte blocks to get to the 64-byte alignment.

6) Then we run the loop reading 64 bytes at a time (as implemented in `sse_aligned64` version).

7) Finally, we deal with the leftovers (possible 32 bytes followed by possible up to 32 bytes).

Here are the results, which leave mixed feelings. The code is faster than `memchr` on small sizes,
but loses to it on big ones:

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">sse_aligned64       </td><td>1.14</td><td>0.33</td><td>0.12&nbsp;</td><td>0.053</td><td>0.037</td><td>0.034</td><td>0.034</td></tr>
<tr><td class="ttext">memchr              </td><td>1.25</td><td>0.32</td><td>0.10&nbsp;</td><td>0.063</td><td>0.035</td><td>0.031</td><td>0.027</td></tr>
<tr><td class="ttext">sse_memchr          </td><td>1.19</td><td>0.30</td><td>0.092     </td><td>0.058</td><td>0.038</td><td>0.033</td><td>0.033</td></tr>
</table>

It looks like re-writing the carefully written assembly code in **C** makes it worse. What if we simplify it?

{% highlight C%}
void* sse_memchr2(const void* ptr, int ch, size_t count)
{
    if (count == 0) return NULL;

    const char * p = (const char*)ptr;

    __m128i value = _mm_set1_epi8((char)ch);
    size_t align = (size_t)p & 0xFFF;

    if (align <= 4096 - 16) {
        TEST16_v_count(p, _mm_loadu_si128((__m128i *) p));
        if (count <= 16) return NULL;
        count -= 16;
        p += 16;

        align &= 0x0F;
        p -= align;
        count += align;
    }
    else {
        align &= 0x0F;
        p -= align;
        __m128i v = _mm_load_si128((__m128i *) p);
        unsigned eq_mask = _mm_movemask_epi8(_mm_cmpeq_epi8(v, value));
        eq_mask >>= align;
        if (eq_mask) {
            size_t pos = bsf(eq_mask);
            if (pos >= count) return NULL;
            return (void *)(p + pos + align);
        }
        count += align;
        if (count <= 16) return NULL;
        p += 16;
        count -= 16;
    }
    for (; count >= 64; count -= 64, p += 64) {

        __m128i v0 = _mm_load_si128((__m128i *) p);
        __m128i v1 = _mm_load_si128((__m128i *) (p + 16));
        __m128i v2 = _mm_load_si128((__m128i *) (p + 32));
        __m128i v3 = _mm_load_si128((__m128i *) (p + 48));

        __m128i cmp0 = _mm_cmpeq_epi8(v0, value);
        __m128i cmp1 = _mm_cmpeq_epi8(v1, value);
        __m128i cmp2 = _mm_cmpeq_epi8(v2, value);
        __m128i cmp3 = _mm_cmpeq_epi8(v3, value);

        __m128i max_val = _mm_max_epu8(_mm_max_epu8(cmp0, cmp1),
                          _mm_max_epu8(cmp2, cmp3));
        int eq_mask = _mm_movemask_epi8(max_val);
        if (eq_mask == 0) continue;

        TEST16_cmp(p, cmp0);
        TEST16_cmp(p + 16, cmp1);
        TEST16_cmp(p + 32, cmp2);
        TEST16_cmp_known_nz(p + 48, cmp3); // returns from function
    }
    if (count >= 32) {
        TEST16(p);
        TEST16(p + 16);
        count -= 32;
        p += 32;
    }
    if (count >= 16) {
        TEST16(p);
        count -= 16;
        p += 16;
    }
    if (count > 0) {
        TEST16_count(p);
    }
    return NULL;
}
{% endhighlight %}

Here we don't insist that the 64-byte loop runs over the 64-byte-aligned memory blocks.
We also relaxed our alignment requirements from 64 to 4096 for the original test.

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">memchr               </td><td>1.25</td><td>0.32</td><td>0.10&nbsp;</td><td>0.063</td><td>0.035</td><td>0.031</td><td>0.027</td></tr>
<tr><td class="ttext">sse_memchr           </td><td>1.19</td><td>0.30</td><td>0.092     </td><td>0.058</td><td>0.038</td><td>0.033</td><td>0.033</td></tr>
<tr><td class="ttext">sse_memchr2          </td><td>1.11</td><td>0.32</td><td>0.11&nbsp;</td><td>0.052</td><td>0.036</td><td>0.033</td><td>0.033</td></tr>
</table>

The results got slightly worse on sizes 16 and 64, but either improved, or stayed the same on all
the other sizes, which makes the simplified version viable. It still, however, loses to `memchr` on
long inputs. Why?

Loop unrolling
--------------

The code generated for `sse_memchr` and `sse_memchr2` differs quite a bit from that of the `memchr`.
The original code is much neater. It is also shorter (although not by far). Let's show the code size
for all the versions presented so far:

<table class="numeric">
<tr><th> Searcher </th><th>Code size, bytes</th></tr>
<tr><td class="ttext">simple_index   </td><td> 385 </td></tr>
<tr><td class="ttext">simple_ptr     </td><td> 385 </td></tr>
<tr><td class="ttext">simple_ptr2    </td><td> 241 </td></tr>
<tr><td class="ttext">memchr         </td><td> 836 </td></tr>
<tr><td class="ttext">index_scas     </td><td> 14 </td></tr>
<tr><td class="ttext">simple_dword   </td><td> 977 </td></tr>
<tr><td class="ttext">dword          </td><td> 956 </td></tr>
<tr><td class="ttext">dword_bsf      </td><td> 900 </td></tr>
<tr><td class="ttext">simple_qword   </td><td> 997 </td></tr>
<tr><td class="ttext">qword          </td><td> 1200 </td></tr>
<tr><td class="ttext">qword_bsf      </td><td> 848 </td></tr>
<tr><td class="ttext">sse            </td><td> 1157 </td></tr>
<tr><td class="ttext">sse_unlimited  </td><td> 599 </td></tr>
<tr><td class="ttext">sse_aligned    </td><td> 1716 </td></tr>
<tr><td class="ttext">sse_aligned64  </td><td> 2326 </td></tr>
<tr><td class="ttext">sse_memchr     </td><td> 1221 </td></tr>
<tr><td class="ttext">sse_memchr2    </td><td> 1220 </td></tr>
</table>

Obviously, we don't expect the shortest code to be nesessarily faster. The `memchr_scas` is the
most obvious example of the opposite. Our SSE matchers, compared to the simple matchers,
also demonstrate that the longer code is often faster. However, the `sse_memchr` is implementing
the same algorithm as `memchr`, and yet is 50% longer. This shows that the hand-written code can
still outperform the code produced even by the best compiler (and the GCC is definitely very good).

Why is the `sse_memchr` so long? A brief code inspection shows that the main loop (the one that iterates
over 64-byte blocks) has been unrolled. Generally, loop unrolling is a very powerful tool, but here
we have already unrolled the main loop 4 times in a custom way -- is there really a point to unroll it
more, dealing with all possible leftovers?

Let's try to disable loop unrolling forcibly, using GCC pragmas:

{% highlight C%}
#pragma GCC push_options
#pragma GCC optimize ("no-unroll-loops")

// original code

#pragma GCC pop_options
{% endhighlight %}

We'll do it for both `sse_memchr` and `sse_memchr2`:

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">memchr               </td><td>1.25</td><td>0.32</td><td>0.10&nbsp;</td><td>0.063     </td><td>0.035     </td><td>0.031     </td><td>0.027     </td></tr>
<tr><td class="ttext">sse_memchr           </td><td>1.19</td><td>0.30</td><td>0.092     </td><td>0.058     </td><td>0.038     </td><td>0.033     </td><td>0.033     </td></tr>
<tr><td class="ttext">sse_memchr_nounroll  </td><td>1.18</td><td>0.29</td><td>0.091</td><td>0.056</td><td>0.033</td><td>0.029</td><td>0.024</td></tr>
<tr><td class="ttext">sse_memchr2          </td><td>1.11</td><td>0.32</td><td>0.11</td><td>0.052</td><td>0.036</td><td>0.033</td><td>0.033</td></tr>
<tr><td class="ttext">sse_memchr2_nounroll </td><td>1.11</td><td>0.32</td><td>0.10</td><td>0.049</td><td>0.033</td><td>0.027</td><td>0.024</td></tr>
</table>

Finally, we've done it. The SSE-based version written in **C** is faster than the one carefully
written by hand in assembly. Moreover, the simplified version (`sse_memchr2`) isn't, generally,
performing worse that the original `sse_memchr`. So this will be our version of choice unless we
go for AVX.

The sizes also look better: 775 bytes for `sse_memchr_nounroll` and 472 for `sse_memchr2_nounroll`.
The **C** compiler made code shorter than assembly!

The AVX version of the standard library
---------------------------------------

The same approach can be used using AVX. Here is the AVX version of the `sse_memchr2` code
as the shorter one (the macro definitions are omitted here):

{% highlight C %}
void* avx_memchr2(const void* ptr, int ch, size_t count)
{
    if (count == 0) return NULL;

    const char * p = (const char*)ptr;

    __m256i value = _mm256_set1_epi8((char)ch);
    size_t align = (size_t)p & 0xFFF;

    if (align <= 4096-32) {
        TEST32_v_count(p, _mm256_loadu_si256((__m256i *) p));
        if (count <= 32) return NULL;
        count -= 32;
        p += 32;

        align &= 0x1F;
        p -= align;
        count += align;
    }
    else {
        align &= 0x1F;
        p -= align;
        __m256i v = _mm256_load_si256((__m256i *) p);
        unsigned eq_mask =
            (unsigned)_mm256_movemask_epi8(_mm256_cmpeq_epi8(v, value));
        eq_mask >>= align;
        if (eq_mask) {
            size_t pos = bsf(eq_mask);
            if (pos >= count) return NULL;
            return (void *)(p + pos + align);
        }
        count += align;
        if (count <= 32) return NULL;
        p += 32;
        count -= 32;
    }
    for (; count >= 64; count -= 64, p += 64) {

        __m256i v0 = _mm256_load_si256((__m256i *) p);
        __m256i v1 = _mm256_load_si256((__m256i *) (p + 32));

        __m256i cmp0 = _mm256_cmpeq_epi8(v0, value);
        __m256i cmp1 = _mm256_cmpeq_epi8(v1, value);

        __m256i max_val = _mm256_max_epu8(cmp0, cmp1);
        unsigned eq_mask = (unsigned)_mm256_movemask_epi8(max_val);
        if (eq_mask == 0) continue;

        TEST32_cmp(p, cmp0);
        TEST32_cmp_known_nz(p + 32, cmp1); // returns from function
    }
    if (count > 32) {
        TEST32(p);
        count -= 32;
        p += 32;
    }
    TEST32_count(p);
    return NULL;
}
{% endhighlight %}

The original `avx_memchr` can be seen in the repository.

And here are the times, compared to the previous results:

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">memchr              </td><td>1.25</td><td>0.32</td><td>0.10&nbsp;</td><td>0.063     </td><td>0.035     </td><td>0.031     </td><td>0.027     </td></tr>
<tr><td class="ttext">avx_aligned64       </td><td>1.15</td><td>0.26</td><td>0.12&nbsp;</td><td>0.041</td><td>0.024</td><td>0.018</td><td>0.017</td></tr>
<tr><td class="ttext">avx_memchr          </td><td>1.24</td><td>0.29</td><td>0.085     </td><td>0.045     </td><td>0.026     </td><td>0.018     </td><td>0.017     </td></tr>
<tr><td class="ttext">avx_memchr_nounroll </td><td>1.24</td><td>0.29</td><td>0.088</td><td>0.042</td><td>0.027</td><td>0.024</td><td>0.019</td></tr>
<tr><td class="ttext">avx_memchr2         </td><td>1.23</td><td>0.28</td><td>0.098</td><td>0.041</td><td>0.025</td><td>0.018</td><td>0.017</td></tr>
<tr><td class="ttext">avx_memchr2_nounroll</td><td>1.23</td><td>0.28</td><td>0.092</td><td>0.038</td><td>0.026</td><td>0.023</td><td>0.019</td></tr>
</table>

Surprisinly, the ununrolled versions aren't faster than unrolled ones, and, in general, the versions
aren't faster than `avx_aligned64`. I won't investigate it further, though; we already achieved a lot.

Summary
-------

Here is the total summary of all the methods considered so far, excluding AVX:

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">Null                </td><td>            0.63</td><td>           0.14</td><td>           0.035     </td><td>           0.009     </td><td>           0.002     </td><td>           0.000     </td><td>           0.000     </td></tr>
<tr><td class="ttext">Simple              </td><td>            1.24</td><td>           0.61</td><td>           0.47&nbsp;</td><td>           0.44&nbsp;</td><td>           0.44&nbsp;</td><td>           0.43&nbsp;</td><td>           0.42&nbsp;</td></tr>
<tr><td class="ttext">Simple Ptr          </td><td>            1.20</td><td>           0.61</td><td>           0.47&nbsp;</td><td>           0.43&nbsp;</td><td>           0.43&nbsp;</td><td>           0.42&nbsp;</td><td>           0.42&nbsp;</td></tr>
<tr><td class="ttext">Simple Ptr2         </td><td>            1.17</td><td>           0.53</td><td>           0.43&nbsp;</td><td>           0.41&nbsp;</td><td>           0.42&nbsp;</td><td>           0.41&nbsp;</td><td>           0.41&nbsp;</td></tr>
<tr><td class="ttext">SCAS                </td><td>            7.12</td><td>           2.26</td><td>           1.19&nbsp;</td><td>           0.93&nbsp;</td><td>           0.86&nbsp;</td><td>           0.85&nbsp;</td><td>           0.85&nbsp;</td></tr>
<tr><td class="ttext">Simple dword        </td><td class="p3"> 0.99</td><td>           0.52</td><td>           0.42&nbsp;</td><td>           0.40&nbsp;</td><td>           0.41&nbsp;</td><td>           0.39&nbsp;</td><td>           0.39&nbsp;</td></tr>
<tr><td class="ttext">Dword               </td><td>            1.51</td><td>           0.66</td><td>           0.48&nbsp;</td><td>           0.45&nbsp;</td><td>           0.39&nbsp;</td><td>           0.38&nbsp;</td><td>           0.38&nbsp;</td></tr>
<tr><td class="ttext">Dword BSF           </td><td>            1.28</td><td>           0.58</td><td>           0.40&nbsp;</td><td>           0.36&nbsp;</td><td>           0.32&nbsp;</td><td>           0.29&nbsp;</td><td>           0.29&nbsp;</td></tr>
<tr><td class="ttext">Simple qword        </td><td class="p2"> 0.93</td><td>           0.46</td><td>           0.42&nbsp;</td><td>           0.41&nbsp;</td><td>           0.42&nbsp;</td><td>           0.41&nbsp;</td><td>           0.41&nbsp;</td></tr>
<tr><td class="ttext">Qword               </td><td>            1.59</td><td>           0.49</td><td>           0.24&nbsp;</td><td>           0.19&nbsp;</td><td>           0.16&nbsp;</td><td>           0.15&nbsp;</td><td>           0.15&nbsp;</td></tr>
<tr><td class="ttext">Qword BSF           </td><td>            1.23</td><td>           0.42</td><td>           0.19&nbsp;</td><td>           0.14&nbsp;</td><td>           0.13&nbsp;</td><td>           0.13&nbsp;</td><td>           0.13&nbsp;</td></tr>
<tr><td class="ttext">memchr              </td><td>            1.25</td><td>           0.32</td><td>           0.10&nbsp;</td><td>           0.063     </td><td class="p2">0.035     </td><td class="p3">0.031     </td><td class="p2">0.027     </td></tr>
<tr><td class="ttext">sse                 </td><td>            1.01</td><td class="p3">0.28</td><td>           0.11&nbsp;</td><td>           0.065     </td><td>           0.044     </td><td>           0.041     </td><td>           0.040     </td></tr>
<tr><td class="ttext">sse_unlimited       </td><td class="p1"> 0.88</td><td class="p1">0.26</td><td>           0.11&nbsp;</td><td>           0.068     </td><td>           0.050     </td><td>           0.049     </td><td>           0.048     </td></tr>
<tr><td class="ttext">sse_aligned         </td><td>            1.00</td><td class="p2">0.27</td><td>           0.13&nbsp;</td><td>           0.062     </td><td>           0.043     </td><td>           0.039     </td><td>           0.036     </td></tr>
<tr><td class="ttext">sse_aligned64       </td><td>            1.14</td><td>           0.33</td><td>           0.12&nbsp;</td><td class="p3">0.053     </td><td>           0.037     </td><td>           0.034     </td><td>           0.034     </td></tr>
<tr><td class="ttext">sse_memchr          </td><td>            1.19</td><td>           0.30</td><td class="p2">0.092     </td><td>           0.058     </td><td>           0.038     </td><td>           0.033     </td><td class="p3">0.033     </td></tr>
<tr><td class="ttext">sse_memchr2         </td><td>            1.11</td><td>           0.32</td><td>           0.11&nbsp;</td><td class="p2">0.052     </td><td class="p3">0.036     </td><td>           0.033     </td><td class="p3">0.033     </td></tr>
<tr><td class="ttext">sse_memchr_nounroll </td><td>            1.18</td><td>           0.29</td><td class="p1">0.091     </td><td>           0.056     </td><td class="p1">0.033     </td><td class="p2">0.029     </td><td class="p1">0.024     </td></tr>
<tr><td class="ttext">sse_memchr2_nounroll</td><td>            1.11</td><td>           0.32</td><td class="p3">0.10&nbsp;</td><td class="p1">0.049     </td><td class="p1">0.033     </td><td class="p1">0.027     </td><td class="p1">0.024     </td></tr>
</table>

As usual, the first, the second and the third place in each category are marked green, yellow and red.

And here are the AVX ones:

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">avx                 </td><td class="p3">1.18</td><td class="p2">0.27</td><td>0.10&nbsp;</td><td>0.045</td><td>0.028</td><td>0.024</td><td class="p3">0.023</td></tr>
<tr><td class="ttext">avx_aligned         </td><td class="p1">1.14</td><td class="p1">0.26</td><td>0.12&nbsp;</td><td>0.049</td><td>0.028</td><td class="p2">0.020</td><td class="p2">0.019</td></tr>
<tr><td class="ttext">avx_aligned64       </td><td class="p2">1.15</td><td class="p1">0.26</td><td>0.12&nbsp;</td><td class="p2">0.041</td><td class="p1">0.024</td><td class="p1">0.018</td><td class="p1">0.017</td></tr>
<tr><td class="ttext">avx_memchr          </td><td>1.25</td><td>0.29</td><td class="p1">0.085</td><td>0.045</td><td class="p3">0.026</td><td class="p1">0.018</td><td class="p1">0.017</td></tr>
<tr><td class="ttext">avx_memchr          </td><td>1.24</td><td>0.29</td><td class="p1">0.085</td><td>0.045</td><td class="p3">0.026</td><td class="p1">0.018</td><td class="p1">0.017</td></tr>
<tr><td class="ttext">avx_memchr_nounroll </td><td>1.24</td><td>0.29</td><td class="p2">0.088</td><td class="p3">0.042</td><td>0.027</td><td>0.024</td><td class="p2">0.019</td></tr>
<tr><td class="ttext">avx_memchr2         </td><td>1.23</td><td class="p3">0.28</td><td>0.098</td><td class="p2">0.041</td><td class="p2">0.025</td><td class="p1">0.018</td><td class="p1">0.017</td></tr>
<tr><td class="ttext">avx_memchr2_nounroll</td><td>1.23</td><td class="p3">0.28</td><td class="p3">0.092</td><td class="p1">0.038</td><td class="p3">0.026</td><td class="p3">0.023</td><td class="p2">0.019</td></tr>
</table>

With the exception of size 4, the AVX versions are always faster than the SSE ones, although they are
never twice as fast as one might expect.

In absolute terms, the results don't look bad: the achieved speed was about 41 bytes per nanosecond
(17 bytes per cycle) for SSE, and a bit higher for AVX (59 and 24 bytes). 

When we were searching for byte strings in **Java** (in ["{{ site.TITLE-INDEXOF }}"]({{ site.ART-INDEXOF }})),
the typical times, per byte, varied depending on nature of data and length of pattern, and for
small patterns (100 bytes) were about 0.04 -- 0.10 ns per byte, which makes the search based on
the first character a viable option. Obviously, searching for a string also requires string
comparison, but it is worth trying anyway.

Results on Windows
------------------

For the sake of completeness, let's run the same test on Windows. Unfortunately, there are two
variables that change. First, it's a compiler (Microsoft Visual Studio 2017, MSVC 19.10.25019).
And then, it's hardware: a notebook with i7-6820HQ @ 2.7 GHz (Windows 10).

Here are the abridged results:

<table class="numeric">
<tr><th> Searcher </th><th>4</th><th>16</th><th>64</th><th>256</th><th>1024</th><th>4096</th><th>16384</th></tr>
<tr><td class="ttext">null_index          </td><td>0.49</td><td>0.11</td><td>0.027</td><td>0.007</td><td>0.002</td><td>0.001</td><td>0.000</td></tr>
<tr><td class="ttext">simple_index        </td><td>1.27</td><td>0.60</td><td>0.43&nbsp;</td><td>0.46&nbsp;</td><td>0.40&nbsp;</td><td>0.38&nbsp;</td><td>0.38&nbsp;</td></tr>
<tr><td class="ttext">simple_ptr          </td><td>1.24</td><td>0.84</td><td>0.72&nbsp;</td><td>0.72&nbsp;</td><td>0.70&nbsp;</td><td>0.69&nbsp;</td><td>0.69&nbsp;</td></tr>
<tr><td class="ttext">dword_bsf           </td><td>0.75</td><td>0.36</td><td>0.28&nbsp;</td><td>0.26&nbsp;</td><td>0.23&nbsp;</td><td>0.22&nbsp;</td><td>0.22&nbsp;</td></tr>
<tr><td class="ttext">qword_bsf           </td><td>0.78</td><td>0.30</td><td>0.13&nbsp;</td><td>0.13&nbsp;</td><td>0.11&nbsp;</td><td>0.10&nbsp;</td><td>0.10&nbsp;</td></tr>
<tr><td class="ttext">memchr              </td><td>0.71</td><td>0.16</td><td>0.072</td><td>0.051</td><td>0.045</td><td>0.045</td><td>0.043</td></tr>
<tr><td class="ttext">sse                 </td><td>0.68</td><td>0.18</td><td>0.083</td><td>0.045</td><td>0.042</td><td>0.041</td><td>0.039</td></tr>
<tr><td class="ttext">sse_aligned64       </td><td>0.81</td><td>0.22</td><td>0.072</td><td>0.032</td><td>0.022</td><td>0.022</td><td>0.022</td></tr>
<tr><td class="ttext">sse_memchr2         </td><td>0.68</td><td>0.21</td><td>0.075</td><td>0.032</td><td>0.024</td><td>0.022</td><td>0.022</td></tr>
<tr><td class="ttext">avx                 </td><td>0.78</td><td>0.15</td><td>0.051</td><td>0.028</td><td>0.030</td><td>0.022</td><td>0.020</td></tr>
<tr><td class="ttext">avx_aligned64       </td><td>0.71</td><td>0.16</td><td>0.069</td><td>0.025</td><td>0.013</td><td>0.011</td><td>0.012</td></tr>
<tr><td class="ttext">avx_memchr2         </td><td>0.68</td><td>0.16</td><td>0.063</td><td>0.030</td><td>0.016</td><td>0.012</td><td>0.011</td></tr>
</table>

The main observations are:

- in MSVC, there is a difference between indexing and pointer arithmetic -- indexing works so much faster.
What a shame;

- DWORD-based solution runs at more than twice the speed of a simple byte-based one; and QWORD-based
solution runs exactly two times faster than that;

- the version of `memchr` that comes with the standard library is not really that fast at all;
it loses even to the very basic SSE solution (still beating the simple solution by the factor of 9),
and is twice as slow as the 64-aligned SSE version.

- our fancy SSE and AVX versions are by far the best; however, the `aligned64` versions are good enough,

- specifically, `aligned64` AVX version runs twice as fast as any SSE version; one kind of expects
this, but it only happens here, with MSVC on Windows.

Conclusions
-----------

- **C** provides more options than **Java** to perform low-level tasks such as search for a character.
Some of them may improve performance quite a bit;

- indexing does not differ from pointer arithmetic in performance when scanning byte arrays in **C**;

- simple solutions, which inspect every byte, are quite slow;

- old-style hardware acceleration (`SCAS`) is even slower;

- reading and processing more than one byte at a time helps, especially in combination with the `BSF`
instruction;

- however, using SSE and AVX provides a real breakthrough. The speed can be as much as 24 times
higher than that of simple solutions;

- the `memchr` function from the GCC standard library is well-written in assembly and performs very well,
so in most practical cases is the best and the easiest option to use;

- we still managed to outperform it -- by 10% in SSE and by 35% in AVX. However, the GCC standard library
is evolving with time and is likely to catch up;

- I'm not so sure about MSVC library, which we managed to outperform by 100% while staying with SSE;
the AVX offered another 100%;

- in the process we learned some tricks of working with SSE, e.g. reading extra bytes after of before
the data of interest helps avoid using any non-SSE operations;

- another trick is performing several comparisons at a time, combining the results using
`OR` or `MAX` operation;

- generic loop unrolling as presented by GCC may harm the performance, especially if manual
unrolling has already been done.

Things to do
------------

Now it's time to look at string searching in **C**/**C++**.
