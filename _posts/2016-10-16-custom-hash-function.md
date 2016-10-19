---
layout: post
title:  "Searching in a constant set using conflict-free hashing"
date:  2016-10-16 12:00:00
tags: C++ optimisation hashmap search matching
---

Unlike previous articles, most of which were based on made-up problems, this one originates from a real-life story: fast detection of [SIP](https://en.wikipedia.org/wiki/Session_Initiation_Protocol)
packets in a network-monitoring application.

Full analysis of SIP requires quite a bit of parsing, and most of the packets in the monitored network were not SIP. Some pre-filtering was required -- a procedure that would:

- filter out most of non-SIP packets
- work very fast in the non-SIP case
- work reasonably fast in the SIP case.

SIP is a text-based protocol. The SIP requests start with a method name, such as `OPTIONS` or `INVITE`. The SIP responses start with `SIP/`, followed by a version and a status.
Some method names, such as `ACK`, are short -- four characters, including the following space. This means that the first four bytes belong to the fixed set of 15 combinations
(14 requests and `SIP/` for responses). Checking these four bytes looks good enough for a quick pre-filtering. The probability of a false positive match is non-zero, but
intuitively it seems small enough.

What is the fastest way to check if given four bytes belong to a given set of 15 four-byte combinations?

It is assumed that the bytes come from a memory buffer that's been pre-fetched into cache (the filtering cost is negligible compared to the cost of a cache miss, so the
problem makes no sense in the uncached case).

The test
--------

We'll write our tests in **C++**, compile using GCC 4.8.3 and run on a Haswell Xeon machine (E5-2620 v3 @ 2.40GHz). The code is available in
[the repository](https://github.com/pzemtsov/article-custom-hash).

Our matchers will extend the base class

{% highlight C++ %}
class Matcher
{
public:
    virtual bool match (const char * p) const = 0;
};
{% endhighlight %}

In this article we won't even consider working with four bytes separately, going instead straight to operating on 32-bit numbers.
However, I'd like to keep this interface to retain the possibility to add other implementations. I could have split it in two (this method plus another one that
takes a number as parameter), but it would be an unnecessary complication. Besides, many of the matchers implemented below are hard-coded for specific SIP constants.

Operating on 32-bit numbers introduces non-portability (dependency on the processor endianness). We'll ignore that for now, concentrating on making it
fast on our current platform (Xeon), but provide a more portable version of the best solution in the end.

The test code will iterate the matcher many times with two types of input: "bad" cases (those with negative result; our primary use case)
and "good" cases (those that match; our secondary use case). It will report the time, in nanoseconds per one input matched.

The null matcher
----------------

First, let's try the null matcher -- the one that always returns `false`:

{% highlight C++ %}
class NullMatcher : public Matcher
{
    bool match (const char * p) const
    {
        return false;
    }
};
{% endhighlight %}

This matcher shows the cost of the method dispatch plus the test loop overhead: 1.77 ns.

<table class="numeric">
<tr><th>Version</th>              <th>Time, good test</th><th>Time, bad test</th></tr>
<tr><td class="label">NullMatcher</td>                   <td> 1.77 ns</td><td>  1.78 ns</td></tr>
</table>

Vector-based linear search
--------------------------

Our first version will collect the values into a vector and perform a linear search there:

{% highlight C++ %}
class VectorLinearMatcher : public Matcher
{
    std::vector<uint32_t> values;

    bool match (const char * p) const
    {
        uint32_t v = *(const uint32_t*) p;
        for (uint32_t x : values) {
            if (x == v) return true;
        }
        return false;
    }

public:
    VectorLinearMatcher ()
    {
        values.push_back ('/PIS');
        // skip...
        values.push_back ('LBUP');
    }
};
{% endhighlight %}

Note that the strings had to be inverted due to the little-endian architecture. Here are the times:

<table class="numeric">
<tr><th>Version</th>              <th>Time, good test</th><th>Time, bad test</th></tr>
<tr><td class="label">VectorLinearMatcher</td>           <td> 6.48 ns</td><td>  8.95 ns</td></tr>
</table>

Vector-based functional linear search
-------------------------------------

These days writing your own loops, even the "for each" type ones, is out of fashion. Let's do the same, function style:

{% highlight C++ %}
bool match (const char * p) const
{
    uint32_t v = *(const uint32_t*) p;
    return std::any_of (values.cbegin (), values.cend (),
                        [v](uint32_t x) { return x == v; });
}
{% endhighlight %}

It is debatable if this code is more readable. Interestingly, it is faster:

<table class="numeric">
<tr><th>Version</th>              <th>Time, good test</th><th>Time, bad test</th></tr>
<tr><td class="label">VectorFunctionalLinearMatcher</td> <td> 4.52 ns</td><td>  5.89 ns</td></tr>
</table>

A quick look at the assembly shows that the code of `any_of` has been inlined into `match`, and so was the code of our lambda. As a result, the overhead of the functional
style was zero. In addition, the main loop has been unrolled four times.

Vector-based binary search
--------------------------

Frankly, I don't believe that this volume of data justifies the binary search, but let's try it anyway:

{% highlight C++ %}
class VectorBinaryMatcher : public Matcher
{
    std::vector<uint32_t> values;

    bool match (const char * p) const
    {
        uint32_t v = *(const uint32_t*) p;
        return binary_search (values.cbegin (), values.cend (), v);
    }

public:
    VectorBinaryMatcher ()
    {
        values.push_back ('/PIS');
        // skip...
        values.push_back ('LBUP');
        sort (values.begin(), values.end());
    }
};
{% endhighlight %}

It is indeed slower:

<table class="numeric">
<tr><th>Version</th>              <th>Time, good test</th><th>Time, bad test</th></tr>
<tr><td class="label">VectorBinaryMatcher</td>           <td> 8.54 ns</td><td>  8.37 ns</td></tr>
</table>

Simple linear matcher
---------------------

We can hard-code all the values in the body of `match()` and remove the vector:

{% highlight C++ %}
class LinearMatcher : public Matcher
{
    bool match (const char * p) const
    {
        uint32_t v = *(const uint32_t*) p;
        if (v == '/PIS') return true;
        if (v == 'IVNI') return true;
        if (v == ' KCA') return true;
        if (v == 'CNAC') return true;
        if (v == ' EYB') return true;
        if (v == 'CARP') return true;
        if (v == 'IGER') return true;
        if (v == 'ITPO') return true;
        if (v == 'OFNI') return true;
        if (v == 'ADPU') return true;
        if (v == 'SBUS') return true;
        if (v == 'ITON') return true;
        if (v == 'SSEM') return true;
        if (v == 'EFER') return true;
        if (v == 'LBUP') return true;
        return false;
    }
};
{% endhighlight %}

<table class="numeric">
<tr><th>Version</th>              <th>Time, good test</th><th>Time, bad test</th></tr>
<tr><td class="label">LinearMatcher</td>                 <td> 4.00 ns</td><td>  5.81 ns</td></tr>
</table>

Simple binary matcher
---------------------

We can do the same with the binary matcher. I do not favour this solution, for it is complex and error-prone, unless the code is generated automatically.
Still, completeness requires testing this as well:

{% highlight C++ %}
class BinaryMatcher : public Matcher
{
    bool match (const char * p) const
    {
        uint32_t v = *(const uint32_t*) p;

#define IF_LESS(x) \
        if (v == x) return true;\
        if (v < x)

        IF_LESS ('IGER') {
            IF_LESS ('ADPU') {
                IF_LESS (' KCA') {
                    return v == ' EYB';
                } else {
                    return v == '/PIS';
                }
            } else {
                IF_LESS ('CNAC') {
                    return v == 'CARP';
                } else {
                    return v == 'EFER';
                }
            }
        } else {
            IF_LESS ('LBUP') {
                IF_LESS ('ITPO') {
                    return v == 'ITON';
                } else {
                    return v == 'IVNI';
                }
            } else {
                IF_LESS ('SBUS') {
                    return v == 'OFNI';
                } else {
                    return v == 'SSEM';
                }
            }
        }
#undef IF_LESS
    }
};
{% endhighlight %}

The result is worth the effort: this version is the fastest so far.

<table class="numeric">
<tr><th>Version</th>              <th>Time, good test</th><th>Time, bad test</th></tr>
<tr><td class="label">BinaryMatcher</td>                 <td> 2.90 ns</td><td>  3.33 ns</td></tr>
</table>

Can we produce something even faster? Let's stop using the na&iuml;ve approach and try some collections that are specially designed for searching.

Tree set matcher
----------------

Of all collections, sets are the best to search in. Let's try the (ordered) set, which is implemented as a binary tree:

{% highlight C++ %}
class TreeSetMatcher : public Matcher
{
    std::set<uint32_t> values;

    bool match (const char * p) const
    {
        return values.find (*(const uint32_t*) p) != values.end ();
    }

public:
    TreeSetMatcher ()
    {
        values.insert ('/PIS');
        // skip
        values.insert (' KCA');
    }
};
{% endhighlight %}

Effectively, the tree set does the same thing as the binary search in a vector, but it works slightly faster:

<table class="numeric">
<tr><th>Version</th>              <th>Time, good test</th><th>Time, bad test</th></tr>
<tr><td class="label">TreeSetMatcher</td>                <td> 5.24 ns</td><td>  6.96 ns</td></tr>
</table>

Hash set matcher
----------------

Except, perhaps, a binary set, which is suitable for small numbers as elements, the fastest of all sets has always been the hash set. Let's try this (it is sufficient to
replace the `set` with `unordered_set` in the previous example):

<table class="numeric">
<tr><th>Version</th>              <th>Time, good test</th><th>Time, bad test</th></tr>
<tr><td class="label">HashSetMatcher</td>                <td>19.73 ns</td><td> 13.75 ns</td></tr>
</table>

It is surprising how slowly this works. Perhaps I've done something wrong with the `unordered_set`?

SSE matcher
-----------

The tradition of these articles requires trying, among others, low-level solutions using extended instruction sets. This became especially attractive since
I got hold of a Haswell, which supports AVX2. Let's, however, start with SSE, whic can hold four 32-bit values in a register and perform operations on all four in parallel.

{% highlight C++ %}
class SSEMatcher : public Matcher
{
    bool match (const char * p) const
    {
        uint32_t v = *(const uint32_t*) p;
        __m128i w = _mm_setr_epi32 (v, v, v, v);

#define CMP(x) \
        if (_mm_movemask_epi8 (_mm_cmpeq_epi32 (w, x))) return true

        CMP (_mm_setr_epi32 ('/PIS', 'IVNI', ' KCA', 'CNAC'));
        CMP (_mm_setr_epi32 (' EYB', 'CARP', 'IGER', 'ITPO'));
        CMP (_mm_setr_epi32 ('OFNI', 'ADPU', 'SBUS', 'ITON'));
        CMP (_mm_setr_epi32 ('SSEM', 'EFER', 'LBUP', 'LBUP'));
#undef CMP
        return false;
    }
};
{% endhighlight %}

This code only uses the instructions of the SSE2 instruction set.

How this works: we put four of the values to look for into a register and compare it with the input value duplicated four times.
The comparison instruction (`PCMPEQD`, or `_mm_cmpeq_epi32` intrinsic) puts all ones (`0xFFFFFFFF`) into the components where the inputs are equal, and all zeroes where
they are not. The "move byte mask" instruction (`PMOVMSKB`, or `_mm_movemask_epi8` intrinsic) collects the most significant bits of all the bytes into a single 16-bit value,
making it non-zero if any of comparisons report equality.

Since there're 15 values to look for, we have to do it four times.

<table class="numeric">
<tr><th>Version</th>              <th>Time, good test</th><th>Time, bad test</th></tr>
<tr><td class="label">SSEMatcher</td>                    <td> 2.60 ns</td><td>  2.62 ns</td></tr>
</table>

The result is very good -- it is faster than the binary search.

Improved SSE matcher
--------------------

The SSE matcher tests four values at the time and exits as soon as any of the four match. But, as we remember, our primary use case is the negative one
(the value is not in the set). We can optimise for this case by removing extra branches. These branches are not as harmful as arbitrary branches as they are well-predicted
(usually not executed), but it still helps to replace two instructions (a mask move and a branch) with one `OR` (to be pedantic: the compiler generates only three ORs in total).
We'll accumulate the value and perform a single branch at the end:

{% highlight C++ %}
class SSEMatcher2 : public Matcher
{
    bool match (const char * p) const
    {
        uint32_t v = *(const uint32_t*) p;
        __m128i w = _mm_setr_epi32 (v, v, v, v);
        __m128i d = _mm_setzero_si128 ();

#define CMP(x) \
        d = _mm_or_si128 (d, _mm_cmpeq_epi32 (w, x))

        CMP (_mm_setr_epi32 ('/PIS', 'IVNI', ' KCA', 'CNAC'));
        CMP (_mm_setr_epi32 (' EYB', 'CARP', 'IGER', 'ITPO'));
        CMP (_mm_setr_epi32 ('OFNI', 'ADPU', 'SBUS', 'ITON'));
        CMP (_mm_setr_epi32 ('SSEM', 'EFER', 'LBUP', 'LBUP'));
#undef CMP
        return _mm_movemask_epi8 (d) != 0;
    }
};
{% endhighlight %}

This does indeed run faster, especially, as planned, on the bad tests:

<table class="numeric">
<tr><th>Version</th>              <th>Time, good test</th><th>Time, bad test</th></tr>
<tr><td class="label">SSEMatcher2</td>                   <td> 2.41 ns</td><td>  2.33 ns</td></tr>
</table>

The AVX2 matcher
----------------

The AVX architecture allows processing 256 bits (8 numbers) at a time, but to process integers we need AVX2 (the regular AVX only works with floats). This is what the improved
SSE solution looks like when converted into AVX2:

{% highlight C++ %}
class AVX2Matcher : public Matcher
{
    bool match (const char * p) const
    {
        uint32_t v = *(const uint32_t*) p;
        __m256i w = _mm256_setr_epi32 (v, v, v, v, v, v, v, v);
        __m256i d = _mm256_setzero_si256 ();

#define CMP(x) \
        d = _mm256_or_si256 (d, _mm256_cmpeq_epi32 (w, x));
        
        CMP (_mm256_setr_epi32 ('/PIS', 'IVNI', ' KCA', 'CNAC',
                                ' EYB', 'CARP', 'IGER', 'ITPO'));
        CMP (_mm256_setr_epi32 ('OFNI', 'ADPU', 'SBUS', 'ITON'
                                'SSEM', 'EFER', 'LBUP', 'LBUP'));
#undef CMP
        return _mm256_movemask_epi8 (d) != 0;
    }
};
{% endhighlight %}

This code produces remarkably small and elegant assembly:

{% highlight C++ %}
_ZNK11AVX2Matcher5matchEPKc:
.LFB5001:
        .cfi_startproc
        vbroadcastss    (%rsi), %ymm0
        vpcmpeqd        .LC4(%rip), %ymm0, %ymm1
        vpcmpeqd        .LC5(%rip), %ymm0, %ymm0
        vpor    %ymm0, %ymm1, %ymm0
        vpmovmskb       %ymm0, %eax
        testl   %eax, %eax
        setne   %al
        vzeroupper
        ret
{% endhighlight %}

It also runs exceptionally fast:

<table class="numeric">
<tr><th>Version</th>              <th>Time, good test</th><th>Time, bad test</th></tr>
<tr><td class="label">AVX2Matcher</td>                   <td> 2.10 ns</td><td>  2.09 ns</td></tr>
</table>

It doesn't look easy to make anything faster than this, but we'll try.
 
The conflict-free hash matcher
------------------------------

Finally, we arrived at the main point of this article: the conflict-free hash matcher.

The generic hash map (and hash set) algorithm involves a hash function that maps keys to their hash values, which are used as indices in an array.
More than one key may be mapped to the same array element (a hash conflict situation), so the hash maps store collections of conflicting keys,
as linked lists or other data structures. Good hash functions make these lists short, but never reliably reduce it to one element.
Anything can be inserted into a general-purpose hash map, so it must always be ready for a hash conflict.

Our case, however, is different. All our keys are known, and it should be possible to create a hash function that does not cause conflicts.
For every set of N numbers and every M &ge; N the set of functions that map all these numbers to different values or range [0 .. M&minus;1] isn't empty. We know this, because it contains at least
one function:

{% highlight C++ %}
uint32_t hash (uint32_t v)
{
    switch (v) {
    case value1: return 0;
    ...
    case valueN: return N-1;
    default: return 0; // can be any value
}
{% endhighlight %}

The function suitable for real-life applications, however, must also be fast. Since this set is finite and non-empty, it contains the fastest function, according to some
definition of "fastest" (for instance, we can use the average, or the maximal, count of CPU cycles over all possible inputs). Finding it, however, requires so much computation,
that we can consider it impossible. Besides, even the best function may still happen to be too slow.

We will go a simpler route and try to find a suitable hash function in a specific class. In particular, we will multiply the hashed 32-bit value by some factor and use several
high-order bits of the lower 32-bit portion of the result. The high-order bits are used because they depend on all the bits of the input. This is what the function will
look like:

{% highlight C++ %}
inline uint32_t hash (uint32_t v) const
{
    return (v * FACTOR) >> (32 - NBITS);
}
{% endhighlight %}

We'd like to make the `NBITS` value as small as possible, to make sure that the entire hash table is well cached.
In our case, where we have 15 values, the ideal would be to have `NBITS` equal to 4 (the hash table size of 16).
Can we find a suitable `FACTOR`? Yes, we can: 239012. This leads to the following solution:

{% highlight C++ %}
class StaticHashMatcher : public Matcher
{
    uint32_t values [16];

    inline uint32_t hash (uint32_t v) const
    {
        return (v * 239012) >> 28;
    }

    bool match (const char * p) const
    {
        uint32_t v = *(const uint32_t*) p;
        return values [hash (v)] == v;
    }

    void insert (uint32_t v)
    {
        uint32_t h = hash (v);
        if (values [h]) exit (1);
        values [h] = v;
    }

public:
    StaticHashMatcher ()
    {
        memset (values, 0, sizeof (values));
        insert ('/PIS');
        // skip...
        insert ('LBUP');
        if (values [0] == 0) exit (1);
    }
};
{% endhighlight %}

There is one point of this code that needs a comment. There are 16 slots in the array, 15 of them are occupied by our numbers (the population code checks that all use
different slots, out of paranoia).
What happens with the remaining one? It, obviously, will keep the value zero, and can accidentally match the input value of zero. Since the hash code of
zero is zero, this can only happen if the last remaining slot is at position zero. Fortunately (and this is also checked at the end of the constructor), this isn't our case.
The slot number zero is occupied by `'OFNI'`, and the vacant slot has number 10. This eliminates a possibility for accidental matching.

The matching consists of one multiplication, which is fast these days, and one lookup in a short array. This has to be fast, and it is:

<table class="numeric">
<tr><th>Version</th>              <th>Time, good test</th><th>Time, bad test</th></tr>
<tr><td class="label">StaticHashMatcher</td>             <td> 1.78 ns</td><td>  1.77 ns</td></tr>
</table>

This code outperformed the current champion (the AVX2 version) and caught up with the `NullMatcher`. It appears as if it uses zero CPU cycles (obviously, it is not true,
just the parallel nature of the processor allowed sharing the same cycles between the matcher code and the test code).

Improving portability
---------------------

There are three issues about this code that make it non-conforming to the standard and non-portable:

- the use of single-quoted multi-byte constants (the byte order isn't defined)
- the conversion of a char pointer to a `uint32_t` pointer (may cause unaligned memory access)
- reading of a `uint32_t` value from this pointer (again, undefined byte order).

The byte order has to be converted to some well-defined one. Unfortunately, (unless we combine four bytes manually), the only byte order for which there are portable
routines to convert to and from, is the "network byte order", which is the same as the big-endian.
The `ntohl` function converts a 32-bit value from this order into that of current hardware. It is not a part of **C** or **C++** standard,
but it is available on all usual platforms, and, hopefully, is efficiently implemented as an intrinsic.

The misaligned access can be avoided by using `memcpy`:

{% highlight C++ %}
class StaticPortableHashMatcher : public Matcher
{
    uint32_t values [16];

    inline uint32_t hash (uint32_t v) const
    {
        return (v * 93564) >> 28;
    }

    bool match (const char * p) const
    {
        uint32_t v;
        memcpy (&v, p, sizeof (v));
        v = ntohl (v);
        return values [hash (v)] == v;
    }

    void insert (const char * p)
    {
        uint32_t v;
        memcpy (&v, p, sizeof (v));
        v = ntohl (v);
        uint32_t h = hash (v);
        if (values [h]) exit (1);
        values [h] = v;
    }

public:
    StaticPortableHashMatcher ()
    {
        memset (values, 0, sizeof (values));
        insert ("SIP/");
        // skip...
        insert ("PUBL");
        if (values [0] == 0) exit (1);
    }
};
{% endhighlight %}

If some purists find some other violations of the standard here, please let me know.

Note that, since the byte order has changed, we're working with different 15 numbers than previously, and previous factor 239012 may stop working. It really does:
both `CANC` and `SUBS` map to 1, `MESS` and `REFE` map to 2, while `PRAC`, `REGI`, `INFO`, `UPDA` and `PUBL` all map to 3, making all 15 values occupy only 9 slots.
We had to look for another factor. Fortunately, it exists: 93564.

The new code looks much longer, and it contains calls into the library. However, the generated instructions look very neat and differ from the code for the non-portable matcher
only in one instruction (`bswap`):

{% highlight asm %}
_ZNK25StaticPortableHashMatcher5matchEPKc:
.LFB5008:
        .cfi_startproc
        movl    (%rsi), %eax
        bswap   %eax
        imull   $93564, %eax, %edx
        shrl    $28, %edx
        cmpl    %eax, 8(%rdi,%rdx,4)
        sete    %al
        ret
{% endhighlight %}

(`%rdi` and `%rsi` are two parameters of the method; `%rdi` is `this`, while `%rsi` is `p`).

This runs slightly slower than the non-portable version, but still much faster than the AVX2 solution.
     
<table class="numeric">
<tr><th>Version</th>              <th>Time, good test</th><th>Time, bad test</th></tr>
<tr><td class="label">StaticPortableHashMatcher</td>     <td> 1.90 ns</td><td>  1.86 ns</td></tr>
</table>

Applicability of this approach
------------------------------

Can an appropriate factor be found for all possible sets of fifteen 32-bit numbers? No, although I only found very artificial counter-examples, where all the numbers
fit in one byte, for example,

{% highlight C++ %}
    {68, 70, 72, 74, 75, 78, 81, 82, 86, 89, 91, 92, 93, 94, 95}
{% endhighlight %}

For this set there isn't a factor that would place all the values into a
hash table of size 16 without conflicts. However, there is such factor (103135728) for the hash table of size 32.

I don't know if size 32 (five-bit hash) is sufficient for all sets of 15, and what is the general rule defining the minimal hash table size suitable for all possible
sets of N numbers.

I expect that this method may be less applicable for bigger sets of the accepted  values, where it is more likely that the appropriate factor does not exist,
I ran several tests, and they confirm it. A set of 50 random numbers required 7 bits, while 100 numbers required 9 bits and 1000 numbers required 15 bits. This makes the
approach practically usable even for these numbers, but also shows that there is a limit, and it is somewhere there (set size of several hundreds).

Some other hash functions may exist (after all, the main reason I used multiplication was that it was the first thing that came to mind). It is also possible
that for really big sets the only option is a traditional hash set.

Result summary
--------------

Here is the final result table:

<table class="numeric">
<tr><th>Version</th>              <th>Time, good test</th><th>Time, bad test</th></tr>
<tr><td class="label">NullMatcher</td>                   <td> 1.77 ns</td><td>  1.78 ns</td></tr>
<tr><td class="label">VectorLinearMatcher</td>           <td> 6.48 ns</td><td>  8.95 ns</td></tr>
<tr><td class="label">VectorFunctionalLinearMatcher</td> <td> 4.52 ns</td><td>  5.89 ns</td></tr>
<tr><td class="label">VectorBinaryMatcher</td>           <td> 8.54 ns</td><td>  8.37 ns</td></tr>
<tr><td class="label">LinearMatcher</td>                 <td> 4.00 ns</td><td>  5.81 ns</td></tr>
<tr><td class="label">BinaryMatcher</td>                 <td> 2.90 ns</td><td>  3.33 ns</td></tr>
<tr><td class="label">TreeSetMatcher</td>                <td> 5.24 ns</td><td>  6.96 ns</td></tr>
<tr><td class="label">HashSetMatcher</td>                <td>19.73 ns</td><td> 13.75 ns</td></tr>
<tr><td class="label">SSEMatcher</td>                    <td> 2.60 ns</td><td>  2.62 ns</td></tr>
<tr><td class="label">SSEMatcher2</td>                   <td> 2.41 ns</td><td>  2.33 ns</td></tr>
<tr><td class="label">AVX2Matcher</td>                   <td> 2.10 ns</td><td>  2.09 ns</td></tr>
<tr><td class="label">StaticHashMatcher</td>             <td> 1.78 ns</td><td>  1.77 ns</td></tr>
<tr><td class="label">StaticPortableHashMatcher</td>     <td> 1.90 ns</td><td>  1.86 ns</td></tr>
</table>

Conclusions
-----------

- Conflict-free hash tables are very powerful for fast matching of numbers against known sets of numbers

- The specific version of these tables, with multiplication-based hash, outperform standard collections (vectors and sets)
and solutions based on SSE and AVX (although the latter ones may be very fast when the set of values is small).

- There is, however, no proof that these tables can be built for any input sets. If there is one for your specific set, you can consider yourself lucky.

