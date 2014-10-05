---
layout: post
title:  "A bug story: testing your test code"
date:   2014-09-21 12:00:00
tags:   C C++ bug
---

Background
----------

This story happened when I was preparing the article ["{{ site.TITLE-E1-C-SSE }}"]({{ site.ART-E1-C-SSE }})". For those who are out of context, here is a brief
introduction. We were de-multiplexing the E1 stream, first [in **Java**]({{ site.ART-E1 }}), then
[in **C++**]({{ site.ART-E1-C }}).
The de-multiplexing routine takes one long byte array as an input and writes its output into a collection of shorter byte arrays,
one per each timeslot. The operation of the de-multiplexer is best described by the following code:

{% highlight C++ %}
class Reference : public Demux
{
public:
    void demux (const byte * src, size_t src_length, byte ** dst) const
    {
        assert (src_length % NUM_TIMESLOTS == 0);

        size_t dst_pos = 0;
        size_t dst_num = 0;
        for (size_t src_pos = 0; src_pos < src_length; src_pos++) {
            dst [dst_num][dst_pos] = src [src_pos];
            if (++ dst_num == NUM_TIMESLOTS) {
                dst_num = 0;
                ++ dst_pos;
            }
        }
    }
};
{% endhighlight %}

This code is called the reference implementation, because all the versions we write are tested against it.
They must produce identical result to that of the `Reference` on some random input.

The objective of the entire exercise is to develop the fastest version of the de-multiplexer. The speed is measured
by running the method some big number of times and timing it:

{% highlight C++ %}
byte * src;
byte ** dst;

void measure (const Demux & demux)
{
    check (demux);

    uint64_t t0 = currentTimeMillis ();
    for (int i = 0; i < ITERATIONS; i++) {
        demux.demux (src, SRC_SIZE, dst);
    }
    uint64_t t = currentTimeMillis () - t0;
    cout << typeid (demux).name() << ": " << t << endl;
}
{% endhighlight %}

The source and destination arrays are allocated once and re-used so that all the speed tests
run in the same area of memory. This is how we allocate memory and run the tests:

{% highlight C++ %}
byte * generate ()
{
    byte * buf = new byte [SRC_SIZE];
    srand (0);
    for (size_t i = 0; i < SRC_SIZE; i++) buf[i] = (byte) (rand () % 256);
    return buf;
}
    
byte ** allocate_dst ()
{
    byte ** result = new byte * [NUM_TIMESLOTS];
    for (size_t i = 0; i < NUM_TIMESLOTS; i++) {
        result [i] = new byte [DST_SIZE];
    }
    return result;
}

void delete_dst (byte ** dst)
{
    for (size_t i = 0; i < NUM_TIMESLOTS; i++) {
        delete dst [i];
    }
    delete dst;
}

int main (void)
{
    src = generate ();
    dst = allocate_dst ();

    measure (Class_With_Implementation_To_Test ());

    return 0;
}
{% endhighlight %}

The code
--------

The implementation I was testing was the `Read4_Write4`. Instead of moving individual bytes, it reads and writes
four bytes at a time using 32-bit operations. This way isn't portable (it depends on the processor
[**endianness**](http://en.wikipedia.org/wiki/Endianness), or
the order in which bytes of the word are stored in memory), but it promises performance gains over traditional
implementations.

Here is the code:

{% highlight C++ %}
inline uint32_t byte0 (uint32_t x) 
{
    return x & 0xFF;
}

inline uint32_t byte1 (uint32_t x) 
{
    return (x >> 8) & 0xFF;
}

inline uint32_t byte2 (uint32_t x) 
{
    return (x >> 16) & 0xFF;
}

inline uint32_t byte3 (uint32_t x) 
{
    return (x >> 24) & 0xFF;
}

class Read4_Write4 : public Demux
{
public:
    void demux (const byte * src, size_t src_length, byte ** dst) const
    {
        assert (src_length == NUM_TIMESLOTS * DST_SIZE);
        assert (DST_SIZE % 4 == 0);
        assert (NUM_TIMESLOTS % 4 == 0);

        for (size_t dst_num = 0; dst_num < NUM_TIMESLOTS; dst_num += 4) {
            byte * d0 = dst [dst_num + 0];
            byte * d1 = dst [dst_num + 1];
            byte * d2 = dst [dst_num + 2];
            byte * d3 = dst [dst_num + 3];
            for (size_t dst_pos = 0; dst_pos < DST_SIZE; dst_pos += 4) {
                uint32_t w0 = * (uint32_t*) &src [(dst_pos + 0) * NUM_TIMESLOTS + dst_num];
                uint32_t w1 = * (uint32_t*) &src [(dst_pos + 1) * NUM_TIMESLOTS + dst_num];
                uint32_t w2 = * (uint32_t*) &src [(dst_pos + 2) * NUM_TIMESLOTS + dst_num];
                uint32_t w3 = * (uint32_t*) &src [(dst_pos + 3) * NUM_TIMESLOTS + dst_num];
                * (uint32_t*) &d0 [dst_pos] = make_32 (byte0 (w0), byte0 (w1), byte0 (w2), byte0 (w3));
                * (uint32_t*) &d1 [dst_pos] = make_32 (byte1 (w0), byte1 (w1), byte1 (w2), byte1 (w3));
                * (uint32_t*) &d2 [dst_pos] = make_32 (byte2 (w0), byte2 (w1), byte2 (w2), byte2 (w3));
                * (uint32_t*) &d3 [dst_pos] = make_32 (byte3 (w0), byte3 (w1), byte3 (w2), byte3 (w3));
            }
        }
    }
};
{% endhighlight %}

The entire code can be found [here]({{ site.REPO-BUG-TEST }}/commit/1235970a0dcca2910a52f58b7c20f506051d6bd0) (I made an extract of the original file with only the relevant code and called it
`e1-bug1.cpp`). The code looks a bit complicated at first, but it is easy to see the main idea: we iterate over both input and output
with step 4, effectively dividing our entire matrix into the grid of 4x4 blocks. Each block is read into four
32-bit variables (hopefully, residing in registers), transposed in registers by means of shifts, and then written
to the output using 32-bit write instructions.

Obviously, we must instantiate the new class in `main()` and call `measure()`:

{% highlight C++ %}
int main (void)
{
    src = generate ();
    dst = allocate_dst ();

    measure (Read4_Write4 ());

    return 0;
}
{% endhighlight %}

Now we compile the program and run it:

    # ./e1-bug1 
    12Read4_Write4: 688

Going unrolled
--------------

The inspection of the generated code showed that the compiler didn't unroll the inner loop of the `read4_write4`
procedure, [so I decided to do it manually]({{ site.REPO-BUG-TEST }}/commit/746de80e363fd0eee6784476bfaae68fb4b5e9e3):

{% highlight C++ %}
class Read4_Write4_Unroll : public Demux
{
public:
    void demux (const byte * src, size_t src_length, byte ** dst) const
    {
        assert (src_length == NUM_TIMESLOTS * DST_SIZE);
        assert (DST_SIZE == 64);
        assert (NUM_TIMESLOTS % 4 == 0);

        for (size_t dst_num = 0; dst_num < NUM_TIMESLOTS; dst_num += 4) {
            byte * d0 = dst [dst_num + 0];
            byte * d1 = dst [dst_num + 1];
            byte * d2 = dst [dst_num + 2];
            byte * d3 = dst [dst_num + 3];
#define MOVE16(dst_pos) do {\
                uint32_t w0 = * (uint32_t*) &src [(dst_pos + 0) * NUM_TIMESLOTS + dst_num];\
                uint32_t w1 = * (uint32_t*) &src [(dst_pos + 1) * NUM_TIMESLOTS + dst_num];\
                uint32_t w2 = * (uint32_t*) &src [(dst_pos + 2) * NUM_TIMESLOTS + dst_num];\
                uint32_t w3 = * (uint32_t*) &src [(dst_pos + 3) * NUM_TIMESLOTS + dst_num];\
                * (uint32_t*) &d0 [dst_pos] = make_32 (byte0 (w0), byte0 (w1), byte0 (w2), byte0 (w3));\
                * (uint32_t*) &d1 [dst_pos] = make_32 (byte1 (w0), byte1 (w1), byte1 (w2), byte1 (w3));\
                * (uint32_t*) &d2 [dst_pos] = make_32 (byte2 (w0), byte2 (w1), byte2 (w2), byte2 (w3));\
                * (uint32_t*) &d3 [dst_pos] = make_32 (byte3 (w0), byte3 (w1), byte3 (w2), byte3 (w3));\
            } while (0)
            MOVE16 (0);
            MOVE16 (4);
            MOVE16 (8);
            MOVE16 (12);
            MOVE16 (16);
            MOVE16 (20);
            MOVE16 (24);
            MOVE16 (28);
#undef MOVE16
        }
    }
};
{% endhighlight %}

It is very easy to do, all that's needed is to made a macro out of the inner loop code (don't forget trailing
backslashes -- they allow a macro to occupy several source lines), and then call this macro appropriate number
of times.

Obviously, I had to add it to the `main` function:

{% highlight C++ %}
int main (void)
{
    src = generate ();
    dst = allocate_dst ();

    measure (Read4_Write4 ());
    measure (Read4_Write4_Unroll ());

    return 0;
}
{% endhighlight %}

And this is what I got:

    # ./e1-bug1
    12Read4_Write4: 688
    19Read4_Write4_Unroll: 321

This looks very good: we've achieved great speed improvement.

Too good to be true
-------------------

The unrolled version runs more than twice as fast as the original one. Can this result be correct?  Generally, yes.
You never know what a specific optimisation can do. Sometimes a small change can cause huge effect. In this particular
case -- highly unlikely. The `MOVE16` macro contains much more code than a typical set of loop control instructions. And
it must be a one to one ratio to cause this speed-up due to unrolling.

No, something must be wrong in the code. And it must be wrong in very obscure way to pass my correctness check. It's time
to use some debugging tools.

The ultimate debugging tool
---------------------------

A long time ago one of my friends made a statement that the ultimate debugging tool is your own head, and the best
debugging method is code inspection. Some intuition can help, too, and in our case the intuition suggests that if
one piece of code runs half the time of the other one, the first thing to check is if it isn't by any chance doing
half the work.

And yes, the code inspection shows that this is exactly what is happening. Our outputs are 64 bytes long (the value
of `DST_SIZE` is 64), and the inner loop runs from 0 to 64. Each iteration is writing four output bytes, so there
are 16 iterations, which must be converted into 16 calls of the macro, while I only made 8. As a result,
the program wrote only 32 bytes into each output vector.

[This is the correct version of this routine]({{ site.REPO-BUG-TEST }}/commit/34e66e68eeedd989e3692d312178b33c023b10ef):

{% highlight C++ %}
class Read4_Write4_Unroll : public Demux
{
public:
    void demux (const byte * src, size_t src_length, byte ** dst) const
    {
        assert (src_length == NUM_TIMESLOTS * DST_SIZE);
        assert (DST_SIZE == 64);
        assert (NUM_TIMESLOTS % 4 == 0);

        for (size_t dst_num = 0; dst_num < NUM_TIMESLOTS; dst_num += 4) {
            byte * d0 = dst [dst_num + 0];
            byte * d1 = dst [dst_num + 1];
            byte * d2 = dst [dst_num + 2];
            byte * d3 = dst [dst_num + 3];
#define MOVE16(dst_pos) do {\
                uint32_t w0 = * (uint32_t*) &src [(dst_pos + 0) * NUM_TIMESLOTS + dst_num];\
                uint32_t w1 = * (uint32_t*) &src [(dst_pos + 1) * NUM_TIMESLOTS + dst_num];\
                uint32_t w2 = * (uint32_t*) &src [(dst_pos + 2) * NUM_TIMESLOTS + dst_num];\
                uint32_t w3 = * (uint32_t*) &src [(dst_pos + 3) * NUM_TIMESLOTS + dst_num];\
                * (uint32_t*) &d0 [dst_pos] = make_32 (byte0 (w0), byte0 (w1), byte0 (w2), byte0 (w3));\
                * (uint32_t*) &d1 [dst_pos] = make_32 (byte1 (w0), byte1 (w1), byte1 (w2), byte1 (w3));\
                * (uint32_t*) &d2 [dst_pos] = make_32 (byte2 (w0), byte2 (w1), byte2 (w2), byte2 (w3));\
                * (uint32_t*) &d3 [dst_pos] = make_32 (byte3 (w0), byte3 (w1), byte3 (w2), byte3 (w3));\
            } while (0)
            MOVE16 (0);
            MOVE16 (4);
            MOVE16 (8);
            MOVE16 (12);
            MOVE16 (16);
            MOVE16 (20);
            MOVE16 (24);
            MOVE16 (28);
            MOVE16 (32);
            MOVE16 (36);
            MOVE16 (40);
            MOVE16 (44);
            MOVE16 (48);
            MOVE16 (52);
            MOVE16 (56);
            MOVE16 (60);
#undef MOVE16
        }
    }
};
{% endhighlight %}

This is what we get when we run it:

    # ./e1-bug1
    12Read4_Write4: 688
    19Read4_Write4_Unroll: 638

This looks much more realistic. The improvement is 50 ms, or 7%, which is very reasonable for loop unrolling.

By the way, neither `Read4_Write4` nor `Read4_Write4_Unroll` shows any extraordinary performance. `Write4` is faster than both.
SSE-based improvements were needed to achieve a real performance increase. See details in ["{{ site.TITLE-E1-C-SSE }}"]
({{ site.ART-E1-C-SSE }}).

Correctness check
-----------------

Here comes the real mystery. What about our correctness check? All our versions passed this check,
producing the result identical to that of the `Reference`. The incorrect version also passed it,
and we know that it wrote only half the result to each output vector. How could this happen?

Here is the check procedure:

{% highlight C++ %}
void check (const Demux & demux)
{
    byte * src = generate ();
    byte ** dst0 = allocate_dst ();
    byte ** dst = allocate_dst ();
    Reference().demux (src, SRC_SIZE, dst0);
    demux.demux (src, SRC_SIZE, dst);

    for (int i = 0; i < NUM_TIMESLOTS; i++) {
        if (memcmp (dst0[i], dst[i], DST_SIZE)) {
            cout << "Results not equal: line " << i << "\n";
            exit (1);
        }
    }
    delete src;
    delete_dst (dst0);
    delete_dst (dst);
}
{% endhighlight %}

The procedure seems correct. We generate the input buffer and allocate two sets of outputs -- for `Reference` and
for the implementation in question. Then we run both and compare. What's wrong?

Maybe the comparison isn't done right? [Let's restore the incorrect unrolled version and dump the outputs]
({{ site.REPO-BUG-TEST }}/commit/282c52ad06fffa5e3797ef0d3ccde9ed6af5a10a):

{% highlight C++ %}
void dump (byte ** dst)
{
    for (int i = 0; i < 32; i++) {
        for (int j = 0; j < 64; j++) {
            printf("%02X ", dst[i][j]);
        }
        printf("\n");
    }
}

void check (const Demux & demux)
{
    byte * src = generate ();
    byte ** dst0 = allocate_dst ();
    byte ** dst = allocate_dst ();
    Reference().demux (src, SRC_SIZE, dst0);
    demux.demux (src, SRC_SIZE, dst);
    cout << "Result\n";
    dump (dst);

    for (int i = 0; i < NUM_TIMESLOTS; i++) {
        if (memcmp (dst0[i], dst[i], DST_SIZE)) {
            cout << "Results not equal: line " << i << "\n";
            exit (1);
        }
    }
    delete src;
    delete_dst (dst0);
    delete_dst (dst);
}
{% endhighlight %}

Here is the result (I skip most of the output):

    # ./e1-bug1
    Result
    67 66 70 02 05 3B 0B AC 08 A3 3E 49 25 3F 22 89 95 6E C9 75 02 9B 56 BC F3 60
    0D 75 5A 61 E2 E8 2B 68 08 7A FF B2 C9 AF 90 49 FC 68 D4 1B 27 23 A1 E2 D2 B3
    D3 8A 74 B8 7B 44 DB E6 9D FB D6 D8 
    C6 32 E9 1A EF 70 E1 86 70 84 05 69 CF 62 FC D4 AA 03 2A A7 F4 BF 89 F7 58 22
    A7 71 31 E0 A0 15 8C 2A 57 28 D9 C8 78 36 EF 70 AD B1 E2 1B CD A0 01 1C E5 2A
    51 2F 01 28 D6 3D 99 A4 7D 35 A1 13 
    <skip>
    12Read4_Write4: 704
    Result
    67 66 70 02 05 3B 0B AC 08 A3 3E 49 25 3F 22 89 95 6E C9 75 02 9B 56 BC F3 60
    0D 75 5A 61 E2 E8 2B 68 08 7A FF B2 C9 AF 90 49 FC 68 D4 1B 27 23 A1 E2 D2 B3
    D3 8A 74 B8 7B 44 DB E6 9D FB D6 D8 
    C6 32 E9 1A EF 70 E1 86 70 84 05 69 CF 62 FC D4 AA 03 2A A7 F4 BF 89 F7 58 22
    A7 71 31 E0 A0 15 8C 2A 57 28 D9 C8 78 36 EF 70 AD B1 E2 1B CD A0 01 1C E5 2A
    51 2F 01 28 D6 3D 99 A4 7D 35 A1 13 
    <skip>
    19Read4_Write4_Unroll: 321

The first output is from `Read4_Write4`, the second from `Read4_Write4_Unroll`, and they are identical. The check
procedure seems to be working correctly.

Next thing to look at is how the procedures update the memory. What was there before? Let's dump `dst` before the
call to `demux` (the code is [here]({{ site.REPO-BUG-TEST }}/commit/4678bbd6cff1c878ff2d08f0d114390959513bd2)):

{% highlight C++ %}
    cout << "dst\n";
    dump (dst);
    demux.demux (src, SRC_SIZE, dst);
    cout << "Result\n";
    dump (dst);
{% endhighlight %}

This is the output:

    # ./e1-bug1
    dst
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    00 00 00 00 00 00 00 00 00 00 00 00 
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    00 00 00 00 00 00 00 00 00 00 00 00 
    <Skip>
    Result
    67 66 70 02 05 3B 0B AC 08 A3 3E 49 25 3F 22 89 95 6E C9 75 02 9B 56 BC F3 60
    0D 75 5A 61 E2 E8 2B 68 08 7A FF B2 C9 AF 90 49 FC 68 D4 1B 27 23 A1 E2 D2 B3
    D3 8A 74 B8 7B 44 DB E6 9D FB D6 D8 
    C6 32 E9 1A EF 70 E1 86 70 84 05 69 CF 62 FC D4 AA 03 2A A7 F4 BF 89 F7 58 22
    A7 71 31 E0 A0 15 8C 2A 57 28 D9 C8 78 36 EF 70 AD B1 E2 1B CD A0 01 1C E5 2A
    51 2F 01 28 D6 3D 99 A4 7D 35 A1 13 
    <Skip>
    12Read4_Write4: 691
    dst
    F0 95 CF 01 00 00 00 00 08 A3 3E 49 25 3F 22 89 95 6E C9 75 02 9B 56 BC F3 60
    0D 75 5A 61 E2 E8 2B 68 08 7A FF B2 C9 AF 90 49 FC 68 D4 1B 27 23 A1 E2 D2 B3
    D3 8A 74 B8 7B 44 DB E6 9D FB D6 D8 
    50 97 CF 01 00 00 00 00 70 84 05 69 CF 62 FC D4 AA 03 2A A7 F4 BF 89 F7 58 22
    A7 71 31 E0 A0 15 8C 2A 57 28 D9 C8 78 36 EF 70 AD B1 E2 1B CD A0 01 1C E5 2A
    51 2F 01 28 D6 3D 99 A4 7D 35 A1 13 
    <Skip>
    Result
    67 66 70 02 05 3B 0B AC 08 A3 3E 49 25 3F 22 89 95 6E C9 75 02 9B 56 BC F3 60
    0D 75 5A 61 E2 E8 2B 68 08 7A FF B2 C9 AF 90 49 FC 68 D4 1B 27 23 A1 E2 D2 B3
    D3 8A 74 B8 7B 44 DB E6 9D FB D6 D8 
    C6 32 E9 1A EF 70 E1 86 70 84 05 69 CF 62 FC D4 AA 03 2A A7 F4 BF 89 F7 58 22
    A7 71 31 E0 A0 15 8C 2A 57 28 D9 C8 78 36 EF 70 AD B1 E2 1B CD A0 01 1C E5 2A
    51 2F 01 28 D6 3D 99 A4 7D 35 A1 13 
    <Skip>
    19Read4_Write4_Unroll: 320

This looks interesting. The destination memory, freshly allocated in both cases, is filled with zeroes before the first
run and is not before the second one. This is normal, because, unlike in **Java**, **C++**'s `new` operator does not clear
allocated memory (or, rather, does not have such obligation -- MSVC seems to clear it).
Most users fill allocated objects and arrays with something useful (which is the point of allocating them in the first place);
clearing them would have been a
waste of CPU cycles. **Java**'s way is safer, but less efficient (if the word "safer" is appropriate here is a matter of
opinion; after all, programs that use uninitialised data are usually incorrect; "safety" makes them behave consistently
incorrect rather than demonstrate some variety).

However, the arrays are not only filled with non-zeroes -- they are filled with almost correct results. Only the first
eight bytes differ, the rest are the same. No wonder the incorrect implementation seemed to produce correct result -- it
wrote 32 correct bytes into the outputs, overwriting the eight broken bytes.

It is not completely impossible that some arbitrary piece of memory is just by chance filled by (almost) correct data.
I saw this happening. However, it is highly unlikely in our case.
It's natural to suppose that this is in fact the same memory as in the first case. This is easy to check, [we'll just
print the addresses]({{ site.REPO-BUG-TEST }}/commit/016827e9ce0a1507d0134fbea590c4617c8cfbbe):

{% highlight C++ %}
void dump (byte ** dst)
{
    for (int i = 0; i < 32; i++) {
        printf ("%p: ", dst[i]);
        for (int j = 0; j < 64; j++) {
            printf("%02X ", dst[i][j]);
        }
        printf("\n");
    }
}
{% endhighlight %}

    # ./e1-new 
    dst
    0x20c3760: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
    0x20c37b0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
    <skip>
    Result
    0x20c3760: 67 66 70 02 05 3B 0B AC 08 A3 3E 49 25 3F 22 89 95 6E C9 75 02 9B
    56 BC F3 60 0D 75 5A 61 E2 E8 2B 68 08 7A FF B2 C9 AF 90 49 FC 68 D4 1B 27 23
    A1 E2 D2 B3 D3 8A 74 B8 7B 44 DB E6 9D FB D6 D8 
    0x20c37b0: C6 32 E9 1A EF 70 E1 86 70 84 05 69 CF 62 FC D4 AA 03 2A A7 F4 BF
    89 F7 58 22 A7 71 31 E0 A0 15 8C 2A 57 28 D9 C8 78 36 EF 70 AD B1 E2 1B CD A0
    01 1C E5 2A 51 2F 01 28 D6 3D 99 A4 7D 35 A1 13 
    <skip>
    12Read4_Write4: 708
    dst
    0x20c3760: F0 35 0C 02 00 00 00 00 08 A3 3E 49 25 3F 22 89 95 6E C9 75 02 9B
    56 BC F3 60 0D 75 5A 61 E2 E8 2B 68 08 7A FF B2 C9 AF 90 49 FC 68 D4 1B 27 23
    A1 E2 D2 B3 D3 8A 74 B8 7B 44 DB E6 9D FB D6 D8 
    0x20c37b0: 50 37 0C 02 00 00 00 00 70 84 05 69 CF 62 FC D4 AA 03 2A A7 F4 BF
    89 F7 58 22 A7 71 31 E0 A0 15 8C 2A 57 28 D9 C8 78 36 EF 70 AD B1 E2 1B CD A0
    01 1C E5 2A 51 2F 01 28 D6 3D 99 A4 7D 35 A1 13 
    <skip>
    Result
    0x20c3760: 67 66 70 02 05 3B 0B AC 08 A3 3E 49 25 3F 22 89 95 6E C9 75 02 9B
    56 BC F3 60 0D 75 5A 61 E2 E8 2B 68 08 7A FF B2 C9 AF 90 49 FC 68 D4 1B 27 23
    A1 E2 D2 B3 D3 8A 74 B8 7B 44 DB E6 9D FB D6 D8 
    0x20c37b0: C6 32 E9 1A EF 70 E1 86 70 84 05 69 CF 62 FC D4 AA 03 2A A7 F4 BF
    89 F7 58 22 A7 71 31 E0 A0 15 8C 2A 57 28 D9 C8 78 36 EF 70 AD B1 E2 1B CD A0
    01 1C E5 2A 51 2F 01 28 D6 3D 99 A4 7D 35 A1 13 
    <skip>
    19Read4_Write4_Unroll: 321

It is indeed the same memory. Now we have the entire picture. This is what happened. We allocated memory for the first run, ran correct `demux`,
and freed the memory. When we allocated memory again, the amount and order of allocation was the same, so the same
memory was allocated, already filled with correct results. When memory is freed, it is marked as free and added to
the allocator's data structures, and the usual place to keep these structures is in the free memory itself.
As a result, freeing the block damages its content, but only slightly. In our
case only eight bytes were damaged. The value written there (`F0 35 0C 02 00 00 00 00`) looks suspiciously
like a pointer (`0x020c35f0`), which suggests that the block was inserted into some kind of a list. You can see
that the pointers for consecutive vectors differ by more than 64 (`0x20c37b0 - 0x20c3760 = 0x50 = 80`); this extra 16
bytes are a block header. It seems that these 16 bytes are sufficient to keep the extra information for a busy block
but a free block requires some extra space.

This behaviour is obviously very specific for a memory allocator in use. The allocator doesn't have to behave this
way; it can allocate memory anywhere. For instance, MSVC does not behave like this; as a result, the check
procedure correctly fails on our first version of `Read4_Write4_Unroll`. We got extremely lucky with GCC (or extremely
unlucky, this is again a matter of opinion).

This explanation allows a prediction: if we comment out the test of `Read4_Write4` from `main()`, leaving only
`Read4_Write4_Unroll`, the check will fire. And indeed it does:

    # ./e1-bug1 
    Results not equal: line 0

The fix
-------

The fix in our case is simple: [clear the memory after allocation]
({{ site.REPO-BUG-TEST }}/commit/2f185f8b5e8b46079989caaef1aa5ff22bb203df):

{% highlight C++ %}
byte ** allocate_dst ()
{
    byte ** result = new byte * [NUM_TIMESLOTS];
    for (size_t i = 0; i < NUM_TIMESLOTS; i++) {
        result [i] = new byte [DST_SIZE];
        memset (result [i], 0, DST_SIZE);
    }
    return result;
}
{% endhighlight %}

This causes the correctness check to fire reliably.

Generally, the better idea is to fill the memory with some non-zero value (I usually use `0xEE`), or even with some
random data. If some specific input must cause output of all zeroes, and the output buffer is already filled with zeroes,
the correctness test can fail. However, in our case zero is good enough, since we know that the correct result is not zeroes.

Conclusions
-----------

- When things are too good, it is likely that something is wrong somewhere (the applicability of this statement
  far exceeds program optimisation or programming in general; I would in fact consider it a universal truth).

- Seemingly independent tests may influence each other in some obscure ways; be prepared for this. In many cases
  it makes sense to run the tests one by one.

- What works on one platform may fail on another one.

- Some very unlikely events do happen, so it is better not to try your luck.

- Uninitialised memory is good for performance but bad for testing; fill your output memory with the values that are the
least likely to resemble the correct output.

- Test code is also code; it can contain bugs or drawbacks. It requires the same attention as the production code.
  Possibly, even more, since there are usually no tests for the test code.

- This time I was extremely lucky twice. First, the bug I made caused a big speed improvement; were it smaller,
  I could have attributed it to the effect of the loop unrolling. And then I was lucky enough to pay
  attention to this unrealistic improvement. You can't rely on this much luck -- rather try to improve your
  test procesures.

- The bug would not have been made if instead of writing down the inlined code manually I used some repetition
  instruction having number of repetitions as a parameter. It seems that traditional macros are incapable of that
  (or, at least, I don't know such a way), but templates can help here. Using templates for variable repetitions
  was described in this article: ["{{ site.TITLE-TEMPLATE }}"]({{ site.ART-TEMPLATE }}).

Coming soon
-----------

Unfortunately, this is not the only bug-hunting experience I had when preparing the articles. At least one more account is
coming.

