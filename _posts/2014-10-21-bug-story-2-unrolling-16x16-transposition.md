---
layout: post
title:  "Bug story 2: Unrolling the 16&times;16 matrix transposition, or be careful with macros"
date:   2014-10-21 12:00:00
tags: C C++ GCC optimisation SSE AVX bug
story: e1-demux
story-title: "De-multiplexing of E1 streams"
---

Background
----------

Last time (in ["{{ site.TITLE-TRANSPOSING-16X16 }}"]({{ site.ART-TRANSPOSING-16X16 }})) we developed a routine
for transposition of a 16&times;16 byte matrix using SSE and applied it to the de-multiplexing of SSE streams.

This solution was the fastest on the i7 notebook, but wasn't too fast on Xeon. The suggested way to improve the speed
was to unroll the inner loop. This is what we'll do today.

Original code
-------------

The code is based on a procedure to transpose a 16&times;16 matrix: 

{% highlight C++ %}
inline void transpose_16x16 (
                __m128i&  x0, __m128i&  x1, __m128i&  x2, __m128i&  x3,
                __m128i&  x4, __m128i&  x5, __m128i&  x6, __m128i&  x7,
                __m128i&  x8, __m128i&  x9, __m128i& x10, __m128i& x11,
                __m128i& x12, __m128i& x13, __m128i& x14, __m128i& x15)
{
    __m128i w00, w01, w02, w03;
    __m128i w10, w11, w12, w13;
    __m128i w20, w21, w22, w23;
    __m128i w30, w31, w32, w33;

    transpose_4x4_dwords ( x0,  x1,  x2,  x3, w00, w01, w02, w03);
    transpose_4x4_dwords ( x4,  x5,  x6,  x7, w10, w11, w12, w13);
    transpose_4x4_dwords ( x8,  x9, x10, x11, w20, w21, w22, w23);
    transpose_4x4_dwords (x12, x13, x14, x15, w30, w31, w32, w33);
    w00 = transpose_4x4 (w00);
    w01 = transpose_4x4 (w01);
    w02 = transpose_4x4 (w02);
    w03 = transpose_4x4 (w03);
    w10 = transpose_4x4 (w10);
    w11 = transpose_4x4 (w11);
    w12 = transpose_4x4 (w12);
    w13 = transpose_4x4 (w13);
    w20 = transpose_4x4 (w20);
    w21 = transpose_4x4 (w21);
    w22 = transpose_4x4 (w22);
    w23 = transpose_4x4 (w23);
    w30 = transpose_4x4 (w30);
    w31 = transpose_4x4 (w31);
    w32 = transpose_4x4 (w32);
    w33 = transpose_4x4 (w33);
    transpose_4x4_dwords (w00, w10, w20, w30,  x0,  x1,  x2, x3);
    transpose_4x4_dwords (w01, w11, w21, w31,  x4,  x5,  x6, x7);
    transpose_4x4_dwords (w02, w12, w22, w32,  x8,  x9, x10, x11);
    transpose_4x4_dwords (w03, w13, w23, w33, x12, x13, x14, x15);
}
{% endhighlight %}

It is called in a way that is common for our sub-matrix-based solutions:

{% highlight C++ %}
class Read16_Write16_SSE : public Demux
{
public:
    void demux (const byte * src, size_t src_length, byte ** dst) const
    {
        assert (src_length == NUM_TIMESLOTS * DST_SIZE);
        assert (DST_SIZE % 16 == 0);
        assert (NUM_TIMESLOTS % 16 == 0);

        for (size_t dst_num = 0; dst_num < NUM_TIMESLOTS; dst_num += 16) {
            byte * d0 = dst [dst_num + 0];
            byte * d1 = dst [dst_num + 1];
            byte * d2 = dst [dst_num + 2];
            byte * d3 = dst [dst_num + 3];
            byte * d4 = dst [dst_num + 4];
            byte * d5 = dst [dst_num + 5];
            byte * d6 = dst [dst_num + 6];
            byte * d7 = dst [dst_num + 7];
            byte * d8 = dst [dst_num + 8];
            byte * d9 = dst [dst_num + 9];
            byte * d10= dst [dst_num +10];
            byte * d11= dst [dst_num +11];
            byte * d12= dst [dst_num +12];
            byte * d13= dst [dst_num +13];
            byte * d14= dst [dst_num +14];
            byte * d15= dst [dst_num +15];
            for (size_t dst_pos = 0; dst_pos < DST_SIZE; dst_pos += 16) {

#define LOADREG(i) __m128i w##i = _128i_load (&src [(dst_pos + i) * NUM_TIMESLOTS + dst_num])
#define STOREREG(i) _128i_store (&d##i [dst_pos], w##i)

                LOADREG (0);  LOADREG (1);  LOADREG (2);  LOADREG (3);
                LOADREG (4);  LOADREG (5);  LOADREG (6);  LOADREG (7);
                LOADREG (8);  LOADREG (9);  LOADREG (10); LOADREG (11);
                LOADREG (12); LOADREG (13); LOADREG (14); LOADREG (15);
                transpose_16x16 (w0, w1, w2, w3, w4, w5, w6, w7, w8, w9, w10, w11, w12, w13, w14, w15);
                STOREREG (0);  STOREREG (1);  STOREREG (2);  STOREREG (3);
                STOREREG (4);  STOREREG (5);  STOREREG (6);  STOREREG (7);
                STOREREG (8);  STOREREG (9);  STOREREG (10); STOREREG (11);
                STOREREG (12); STOREREG (13); STOREREG (14); STOREREG (15);
#undef LOADREG
#undef STOREREG 
           }
        }
    }
};
{% endhighlight %}

Unrolling the code
------------------

The unrolling technique is quite standard as well. All we do is convert the inner loop body into a macro and call it
appropriate number of times:

{% highlight C++ %}
class Read16_Write16_SSE_Unroll : public Demux
{
public:
    void demux(const byte * src, size_t src_length, byte ** dst) const
    {
        assert(src_length == NUM_TIMESLOTS * DST_SIZE);
        assert(DST_SIZE == 64);
        assert(NUM_TIMESLOTS % 16 == 0);

        for (size_t dst_num = 0; dst_num < NUM_TIMESLOTS; dst_num += 16) {
            byte * d0 = dst[dst_num + 0];
            byte * d1 = dst[dst_num + 1];
            byte * d2 = dst[dst_num + 2];
            byte * d3 = dst[dst_num + 3];
            byte * d4 = dst[dst_num + 4];
            byte * d5 = dst[dst_num + 5];
            byte * d6 = dst[dst_num + 6];
            byte * d7 = dst[dst_num + 7];
            byte * d8 = dst[dst_num + 8];
            byte * d9 = dst[dst_num + 9];
            byte * d10 = dst[dst_num + 10];
            byte * d11 = dst[dst_num + 11];
            byte * d12 = dst[dst_num + 12];
            byte * d13 = dst[dst_num + 13];
            byte * d14 = dst[dst_num + 14];
            byte * d15 = dst[dst_num + 15];

#define LOADREG(dst_pos, i) __m128i w##i = _128i_load (&src [(dst_pos + i) * NUM_TIMESLOTS + dst_num])
#define STOREREG(dst_pos, i) _128i_store (&d##i [dst_pos], w##i)

#define MOVE256(dst_pos) do {\
                LOADREG (dst_pos, 0);  LOADREG (dst_pos, 1);  LOADREG (dst_pos, 2);  LOADREG (dst_pos, 3);\
                LOADREG (dst_pos, 4);  LOADREG (dst_pos, 5);  LOADREG (dst_pos, 6);  LOADREG (dst_pos, 7);\
                LOADREG (dst_pos, 8);  LOADREG (dst_pos, 9);  LOADREG (dst_pos, 10); LOADREG (dst_pos, 11);\
                LOADREG (dst_pos, 12); LOADREG (dst_pos, 13); LOADREG (dst_pos, 14); LOADREG (dst_pos, 15);\
                transpose_16x16 (w0, w1, w2, w3, w4, w5, w6, w7, w8, w9, w10, w11, w12, w13, w14, w15);\
                STOREREG (dst_pos, 0);  STOREREG (dst_pos, 1);  STOREREG (dst_pos, 2);  STOREREG (dst_pos, 3);\
                STOREREG (dst_pos, 4);  STOREREG (dst_pos, 5);  STOREREG (dst_pos, 6);  STOREREG (dst_pos, 7);\
                STOREREG (dst_pos, 8);  STOREREG (dst_pos, 9);  STOREREG (dst_pos, 10); STOREREG (dst_pos, 11);\
                STOREREG (dst_pos, 12); STOREREG (dst_pos, 13); STOREREG (dst_pos, 14); STOREREG (dst_pos, 15);\
            } while (0)

            MOVE256(0);
            MOVE256(16);
            MOVE256(32);
            MOVE256(48);
#undef MOVE256
#undef LOADREG
#undef STOREREG 
        }
    }
};
{% endhighlight %}

When we run it, however, we get the following:

    18Read16_Write16_SSE: 176
    25Read16_Write16_SSE_Unroll: 194

The program slowed down as a result of loop unrolling.

Analysis of the code reveals why it happened. I won't show the entire inner loop code here, just a small fragment:

{% highlight c-objdump %}
        vmovdqa %xmm8, 480(%rsp)
        vmovdqa %xmm7, 496(%rsp)
        vmovdqa %xmm6, 512(%rsp)
        vmovdqa %xmm5, 528(%rsp)
        vmovdqa %xmm4, 544(%rsp)
        vmovdqa %xmm3, 560(%rsp)
        vmovdqa %xmm2, 576(%rsp)
        vmovdqa %xmm1, 592(%rsp)
        vmovdqa %xmm0, 112(%rsp)
        call    _Z15transpose_16x16RU8__vectorxS0_S0_S0_S0_S0_S0_S0_S0_S0_S0_S0_S0_S0_S0_S0_
{% endhighlight %}

The actual assembly listing contains much more of this horrible code before this call -- passing 16 parameters by
reference isn't an easy job. What happened was exactly what we were scared of when discussing the original code
in the [previous article]({{ site.ART-TRANSPOSING-16X16 }}): one call to `transpose_16x16()` wasn't inlined.
The inner loop contains four of them. Three were inlined and the fourth one wasn't: probably, it exceeded some
inlining budget of the compiler. The `inline` keyword is only a recommendation; it isn't a definite instruction to
the compiler to inline a function. And, as we discussed, this function is an ultimate disaster when not inlined.

Converting to a macro
---------------------

There are two ways to rectify the situation. The first one is to play with compiler command line switches, and
the second one is to convert the `inline` function into a macro. The second one is more reliable (there is no way a
macro can't be inlined), that's why we'll go this route.

Conversion of a function to a macro is straightforward -- it is just a punctuation exercise:

{% highlight C++ %}
#define _transpose_16x16(x0, x1, x2, x3, x4, x5, x6, x7, x8, x9, x10, x11, x12, x13, x14, x15) do {\
    __m128i w00, w01, w02, w03;\
    __m128i w10, w11, w12, w13;\
    __m128i w20, w21, w22, w23;\
    __m128i w30, w31, w32, w33;\
    \
    transpose_4x4_dwords(x0, x1, x2, x3, w00, w01, w02, w03);\
    transpose_4x4_dwords(x4, x5, x6, x7, w10, w11, w12, w13);\
    transpose_4x4_dwords(x8, x9, x10, x11, w20, w21, w22, w23);\
    transpose_4x4_dwords(x12, x13, x14, x15, w30, w31, w32, w33);\
    w00 = transpose_4x4(w00);\
    w01 = transpose_4x4(w01);\
    w02 = transpose_4x4(w02);\
    w03 = transpose_4x4(w03);\
    w10 = transpose_4x4(w10);\
    w11 = transpose_4x4(w11);\
    w12 = transpose_4x4(w12);\
    w13 = transpose_4x4(w13);\
    w20 = transpose_4x4(w20);\
    w21 = transpose_4x4(w21);\
    w22 = transpose_4x4(w22);\
    w23 = transpose_4x4(w23);\
    w30 = transpose_4x4(w30);\
    w31 = transpose_4x4(w31);\
    w32 = transpose_4x4(w32);\
    w33 = transpose_4x4(w33);\
    transpose_4x4_dwords(w00, w10, w20, w30, x0, x1, x2, x3);\
    transpose_4x4_dwords(w01, w11, w21, w31, x4, x5, x6, x7);\
    transpose_4x4_dwords(w02, w12, w22, w32, x8, x9, x10, x11);\
    transpose_4x4_dwords(w03, w13, w23, w33, x12, x13, x14, x15);\
} while (0)
{% endhighlight %}

Running it, we get the following:

    # ./e1-16x16 
    18Read16_Write16_SSE: 178
    Results not equal: line 0

 Something must have gone wrong. The  conversion of a function into a macro has broken something. Now it's time
for some debugging.

Debugging a macro
-----------------

There must be some readers that already know exactly what went wrong. Congratulations to them.
I wasn't so clever. At this point I was totally confused, even suspecting a compiler bug (I know this is the last
thing a programmer must suspect, but, having written a couple of compilers before, I know that compiler bugs do exist,
and quite many of them as well).

Let's debug the program. My favourite way of debugging is debug outputs. Let's put some into this program. Since
the only change we've made is in the way we do matrix transposition, let's check if this is in order. We'll print
the 16&times;16 matrix before and after the transposition. The matrix is kept in SSE registers -- so we need a
routine to dump an SSE register:

{% highlight C++ %}
void dump(__m128i x)
{
    unsigned char v[16];
    _mm_storeu_si128((__m128i*) v, x);
    for (unsigned i = 0; i < 16; i++) {
        printf("%02X ", v[i]);
    }
    printf("\n");
}

#define DUMP(w0, w1, w2, w3, w4, w5, w6, w7, w8, w9, w10, w11, w12, w13, w14, w15) do {\
            dump(w0); dump(w1); dump(w2); dump(w3); \
            dump(w4); dump(w5); dump(w6); dump(w7); \
            dump(w8); dump(w9); dump(w10); dump(w11); \
            dump(w12); dump(w13); dump(w14); dump(w15); \
        } while (0)

{% endhighlight %}

The main loop is modified in the following way:

{% highlight C++ %}
    printf ("Before: \n");\
    DUMP (w0, w1, w2, w3, w4, w5, w6, w7, w8, w9, w10, w11, w12, w13, w14, w15);\
    _transpose_16x16 (w0, w1, w2, w3, w4, w5, w6, w7, w8, w9, w10, w11, w12, w13, w14, w15);\
    printf("After: \n");\
    DUMP(w0, w1, w2, w3, w4, w5, w6, w7, w8, w9, w10, w11, w12, w13, w14, w15); \
    exit (0);\
{% endhighlight %}

The call to `exit()` is there to prevent flooding of the output. Just one dump of a matrix before and after transposition
is enough, we don't need millions of them.

And one last convenient thing: let's fill the source with some easily readable data rather than the random numbers.
Consecutive byte values will work.

Now let's run the code:

    #./e1-16x16
    18Read16_Write16_SSE: 176
    Before:
    00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F
    20 21 22 23 24 25 26 27 28 29 2A 2B 2C 2D 2E 2F
    40 41 42 43 44 45 46 47 48 49 4A 4B 4C 4D 4E 4F
    60 61 62 63 64 65 66 67 68 69 6A 6B 6C 6D 6E 6F
    80 81 82 83 84 85 86 87 88 89 8A 8B 8C 8D 8E 8F
    A0 A1 A2 A3 A4 A5 A6 A7 A8 A9 AA AB AC AD AE AF
    C0 C1 C2 C3 C4 C5 C6 C7 C8 C9 CA CB CC CD CE CF
    E0 E1 E2 E3 E4 E5 E6 E7 E8 E9 EA EB EC ED EE EF
    00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F
    20 21 22 23 24 25 26 27 28 29 2A 2B 2C 2D 2E 2F
    40 41 42 43 44 45 46 47 48 49 4A 4B 4C 4D 4E 4F
    60 61 62 63 64 65 66 67 68 69 6A 6B 6C 6D 6E 6F
    80 81 82 83 84 85 86 87 88 89 8A 8B 8C 8D 8E 8F
    A0 A1 A2 A3 A4 A5 A6 A7 A8 A9 AA AB AC AD AE AF
    C0 C1 C2 C3 C4 C5 C6 C7 C8 C9 CA CB CC CD CE CF
    E0 E1 E2 E3 E4 E5 E6 E7 E8 E9 EA EB EC ED EE EF
    After:
    00 20 40 60 80 A0 C0 E0 00 20 80 84 88 8C C0 E0
    01 21 41 61 81 A1 C1 E1 01 21 81 85 89 8D C1 E1
    02 22 42 62 82 A2 C2 E2 02 22 82 86 8A 8E C2 E2
    03 23 43 63 83 A3 C3 E3 03 23 83 87 8B 8F C3 E3
    04 24 44 64 84 A4 C4 E4 04 24 A0 A4 A8 AC C4 E4
    05 25 45 65 85 A5 C5 E5 05 25 A1 A5 A9 AD C5 E5
    06 26 46 66 86 A6 C6 E6 06 26 A2 A6 AA AE C6 E6
    07 27 47 67 87 A7 C7 E7 07 27 A3 A7 AB AF C7 E7
    08 28 48 68 88 A8 C8 E8 08 28 C0 C4 C8 CC C8 E8
    09 29 49 69 89 A9 C9 E9 09 29 C1 C5 C9 CD C9 E9
    40 41 42 43 44 45 46 47 48 49 4A 4B 4C 4D 4E 4F
    60 61 62 63 64 65 66 67 68 69 6A 6B 6C 6D 6E 6F
    80 81 82 83 84 85 86 87 88 89 8A 8B 8C 8D 8E 8F
    A0 A1 A2 A3 A4 A5 A6 A7 A8 A9 AA AB AC AD AE AF
    0E 2E 4E 6E 8E AE CE EE 0E 2E E2 E6 EA EE CE EE
    0F 2F 4F 6F 8F AF CF EF 0F 2F E3 E7 EB EF CF EF

The dump looks interesting. The top left 10&times;10 matrix in the result is correct; most of everything else is not.
Sometimes the result contains source (untransposed) values, such as in rows 10, 11, 12 and 13 (don't forget that we
count from zero). The first 10 positions of rows and columns 14 and 15 are also correct.

The constant `10` isn't in any way natural for SSE, or for our problem at all. In fact, it is quite unnatural constant
in programming outside of human interaction. It is a human constant. One can hardly
expect a compiler to generate correct code for the first 10 outputs but not for the others.

At this point a clever reader has all the information necessary to find the bug, but I required one more iteration.
So I decided to put another dump after the first transposition in `_transpose_16x16`:

{% highlight C++ %}
    transpose_4x4_dwords(x0, x1, x2, x3, w00, w01, w02, w03);\
    transpose_4x4_dwords(x4, x5, x6, x7, w10, w11, w12, w13);\
    transpose_4x4_dwords(x8, x9, x10, x11, w20, w21, w22, w23);\
    transpose_4x4_dwords(x12, x13, x14, x15, w30, w31, w32, w33);\
    printf ("After first transpose\n");\
    DUMP (w00, w01, w02, w03, w10, w11, w12, w13, w20, w21, w22, w23, w30, w31, w32, w33);\
{% endhighlight %}

This is the output of the new dump:

    After first transpose
    00 01 02 03 20 21 22 23 40 41 42 43 60 61 62 63
    04 05 06 07 24 25 26 27 44 45 46 47 64 65 66 67
    08 09 0A 0B 28 29 2A 2B 48 49 4A 4B 68 69 6A 6B
    0C 0D 0E 0F 2C 2D 2E 2F 4C 4D 4E 4F 6C 6D 6E 6F
    80 81 82 83 A0 A1 A2 A3 C0 C1 C2 C3 E0 E1 E2 E3
    84 85 86 87 A4 A5 A6 A7 C4 C5 C6 C7 E4 E5 E6 E7
    88 89 8A 8B A8 A9 AA AB C8 C9 CA CB E8 E9 EA EB
    8C 8D 8E 8F AC AD AE AF CC CD CE CF EC ED EE EF
    00 01 02 03 20 21 22 23 80 81 82 83 84 85 86 87
    04 05 06 07 24 25 26 27 A0 A1 A2 A3 A4 A5 A6 A7
    08 09 0A 0B 28 29 2A 2B C0 C1 C2 C3 C4 C5 C6 C7
    0C 0D 0E 0F 2C 2D 2E 2F E0 E1 E2 E3 E4 E5 E6 E7
    88 89 8A 8B 8C 8D 8E 8F C0 C1 C2 C3 E0 E1 E2 E3
    A8 A9 AA AB AC AD AE AF C4 C5 C6 C7 E4 E5 E6 E7
    C8 C9 CA CB CC CD CE CF C8 C9 CA CB E8 E9 EA EB
    E8 E9 EA EB EC ED EE EF CC CD CE CF EC ED EE EF

To check the results, the best is to recall that this transposition places every source variable into a 4&times;4
block inside four variables. As a result, in our case each such block must contain consecutive values. We can see that
this is not the case with the blocks that start at row 8, columns 8 and 12:

    80 81 82 83  84 85 86 87 
    A0 A1 A2 A3  A4 A5 A6 A7 
    C0 C1 C2 C3  C4 C5 C6 C7 
    E0 E1 E2 E3  E4 E5 E6 E7 

These blocks correspond to double words 2 and 3 of variables `w20`, `w21`, `w22` and `w23`, which are produced here:

{% highlight C++ %}
    transpose_4x4_dwords(x8, x9, x10, x11, w20, w21, w22, w23);\
{% endhighlight %}

Here is the code for `transpose_4x4_dwords()`:

{% highlight C++ %}
inline void transpose_4x4_dwords (__m128i w0, __m128i w1, __m128i w2, __m128i w3, __m128i &r0, __m128i &r1, __m128i &r2, __m128i &r3)
{
    // 0  1  2  3
    // 4  5  6  7
    // 8  9  10 11
    // 12 13 14 15

    __m128i x0 = _128i_shuffle (w0, w1, 0, 1, 0, 1); // 0 1 4 5
    __m128i x1 = _128i_shuffle (w0, w1, 2, 3, 2, 3); // 2 3 6 7
    __m128i x2 = _128i_shuffle (w2, w3, 0, 1, 0, 1); // 8 9 12 13
    __m128i x3 = _128i_shuffle (w2, w3, 2, 3, 2, 3); // 10 11 14 15

    r0 = _128i_shuffle (x0, x2, 0, 2, 0, 2);
    r1 = _128i_shuffle (x0, x2, 1, 3, 1, 3);
    r2 = _128i_shuffle (x1, x3, 0, 2, 0, 2);
    r3 = _128i_shuffle (x1, x3, 1, 3, 1, 3);
}
{% endhighlight %}

We can see that the double words 2 and 3 of the result are determined by the values of `x2` and `x3`, and these two
depend on the parameters `w2` and `w3`, which in our case are `x10` and `x11`. It looks
as if someone had modified `x10` and `x11` just before they were used here. And the previous line computes
`w10`, `w11`, `w12` and `w13`:

{% highlight C++ %}
    transpose_4x4_dwords(x4, x5, x6, x7, w10, w11, w12, w13);\
{% endhighlight %}

Now it's time to recall that `_transpose_16x16` is a macro, and the actual parameters in the call to this macro are called
`w0` ... `w15`. The puzzle is solved. We declared intermediate values inside a macro, some of which (`w10`, `w11`, `w12`
and `w13`) had the same names as actual parameters. Unlike function calls, macros do not hide the textual
representation of the actual parameters from their internals. In fact, they do exactly the opposite. The actual
parameters are substituted _as is_. This is what the macro body looks like after the substitution:

{% highlight C++ %}
    transpose_4x4_dwords(w0, w1, w2, w3, w00, w01, w02, w03);
    transpose_4x4_dwords(w4, w5, w6, w7, w10, w11, w12, w13);
    transpose_4x4_dwords(w8, w9, w10, w11, w20, w21, w22, w23);
    transpose_4x4_dwords(w12, w13, w14, w15, w30, w31, w32, w33);
    w00 = transpose_4x4(w00);
    w01 = transpose_4x4(w01);
    w02 = transpose_4x4(w02);
    w03 = transpose_4x4(w03);
    w10 = transpose_4x4(w10);
    w11 = transpose_4x4(w11);
    w12 = transpose_4x4(w12);
    w13 = transpose_4x4(w13);
    w20 = transpose_4x4(w20);
    w21 = transpose_4x4(w21);
    w22 = transpose_4x4(w22);
    w23 = transpose_4x4(w23);
    w30 = transpose_4x4(w30);
    w31 = transpose_4x4(w31);
    w32 = transpose_4x4(w32);
    w33 = transpose_4x4(w33);
    transpose_4x4_dwords(w00, w10, w20, w30, w0, w1, w2, w3);
    transpose_4x4_dwords(w01, w11, w21, w31, w4, w5, w6, w7);
    transpose_4x4_dwords(w02, w12, w22, w32, w8, w9, w10, w11);
    transpose_4x4_dwords(w03, w13, w23, w33, w12, w13, w14, w15);
{% endhighlight %}

No wonder it does not work.

Solving the issue
-----------------

The natural way to resolve this is to rename the internal variables so that they do not conflict with parameters.
I fixed the problem by renaming the internal `w` variables to `m`:

{% highlight C++ %}
#define _transpose_16x16(x0, x1, x2, x3, x4, x5, x6, x7, x8, x9, x10, x11, x12, x13, x14, x15) do {\
    __m128i m00, m01, m02, m03;\
    __m128i m10, m11, m12, m13;\
    __m128i m20, m21, m22, m23;\
    __m128i m30, m31, m32, m33;\
    \
    transpose_4x4_dwords(x0, x1, x2, x3, m00, m01, m02, m03);\
    transpose_4x4_dwords(x4, x5, x6, x7, m10, m11, m12, m13);\
    transpose_4x4_dwords(x8, x9, x10, x11, m20, m21, m22, m23);\
    transpose_4x4_dwords(x12, x13, x14, x15, m30, m31, m32, m33);\
    m00 = transpose_4x4(m00);\
    m01 = transpose_4x4(m01);\
    m02 = transpose_4x4(m02);\
    m03 = transpose_4x4(m03);\
    m10 = transpose_4x4(m10);\
    m11 = transpose_4x4(m11);\
    m12 = transpose_4x4(m12);\
    m13 = transpose_4x4(m13);\
    m20 = transpose_4x4(m20);\
    m21 = transpose_4x4(m21);\
    m22 = transpose_4x4(m22);\
    m23 = transpose_4x4(m23);\
    m30 = transpose_4x4(m30);\
    m31 = transpose_4x4(m31);\
    m32 = transpose_4x4(m32);\
    m33 = transpose_4x4(m33);\
    transpose_4x4_dwords(m00, m10, m20, m30, x0, x1, x2, x3);\
    transpose_4x4_dwords(m01, m11, m21, m31, x4, x5, x6, x7);\
    transpose_4x4_dwords(m02, m12, m22, m32, x8, x9, x10, x11);\
    transpose_4x4_dwords(m03, m13, m23, m33, x12, x13, x14, x15);\
} while (0)
{% endhighlight %}

But this is obviously a limited solution. One can call the macro with parameters that start with `m`. Probably, the most
reliable way to resolve this is to establish a naming convention where all internal variables get some
prefix, depending on the macro name -- something like `_TRANSPOSE_16X16_w00`. This, however, makes the text very poorly
readable. Besides, it is always unpleasant to see the program correctness depend on another convention, rather than on
the language's syntax and semantics. Ideally, the `inline` specifier should just work, then this problem would not
have happened in the first place.

Results
-------

After fixing the bug, the output is

    25Read16_Write16_SSE_Unroll: 163

This means that we've achieved some progress, but haven't really beaten the speed record - on Xeon. On the notebook
we actually have:

<table class="numeric">
<tr><th>     Version                </th><th> Time, Xeon</th><th> Time, i7</th></tr>
<tr><td class="label">Reference         </td><td> 1367</td><td> 2040</td></tr>
<tr><td class="label">Src_First_1       </td><td>  987</td><td>  989</td></tr>
<tr><td class="label">Src_First_2       </td><td>  986</td><td>  974</td></tr>
<tr><td class="label">Src_First_3       </td><td> 1359</td><td> 1528</td></tr>
<tr><td class="label">Dst_First_1       </td><td>  991</td><td> 1228</td></tr>
<tr><td class="label">Dst_First_1a      </td><td>  763</td><td> 1137</td></tr>
<tr><td class="label">Dst_First_2       </td><td>  766</td><td> 1227</td></tr>
<tr><td class="label">Dst_First_3       </td><td>  982</td><td>  997</td></tr>
<tr><td class="label">Dst_First_3a      </td><td>  652</td><td>  705</td></tr>
<tr><td class="label">Unrolled_1        </td><td>  636</td><td>  744</td></tr>
<tr><td class="label">Unrolled_1_2      </td><td>  632</td><td>  689</td></tr>
<tr><td class="label">Unrolled_1_4      </td><td>  636</td><td>  703</td></tr>
<tr><td class="label">Unrolled_1_8      </td><td>  635</td><td>  698</td></tr>
<tr><td class="label">Unrolled_1_16     </td><td>  635</td><td>  708</td></tr>
<tr><td class="label">Write_4           </td><td>  491</td><td>  901</td></tr>
<tr><td class="label">Write_8           </td><td>  508</td><td>  775</td></tr>
<tr><td class="label">Read4_Write4      </td><td>  638</td><td>  946</td></tr>
<tr><td class="label">Read4_Write4_SSE  </td><td>  232</td><td>  439</td></tr>
<tr><td class="label">Read4_Write16_SSE </td><td>  141</td><td>  332</td></tr>
<tr><td class="label">Read8_Write16_SSE </td><td>  120</td><td>  323</td></tr>
<tr><td class="label">Read16_Write16_SSE</td><td>  163</td><td>  283</td></tr>
<tr><td class="label">Read4_Write32_AVX </td><td>  138</td><td>  378</td></tr>
<tr><td class="label">Null              </td><td>    1</td><td>    3</td></tr>
<tr><td class="label">Copy              </td><td>   98</td><td>   33</td></tr>
<tr><td class="label">Copy_AVX          </td><td>   56</td><td>   76</td></tr>
</table>

It is interesting that the results on the notebook (Intel(R) Core(TM) i7-4800MQ CPU @ 2.70 GHz) are sometimes very
similar to those on the server (Intel(R) Xeon(R) CPU E5-2670 @ 2.60GHz), but sometimes are much worse. The similar results
can be explained by similar clock speeds. The different results probably demonstrate the significance of other variables --
in our case, the OS and the compiler. I used GNU C 4.6 on Linux in one case and Visual C++ 2013 (as part of Microsoft
Visual Studio Express 2013 for Windows Desktop), both in 64-bit mode. The code looks quite different, which can mean
one of two things: either MSVC isn't as good a compiler as GNU C, or it requires additional tuning.

It's also interesting that `Copy` methods are faster in MSVC than in GNU C, and that `Copy` is faster than `Copy_AVX` there.
The latter is explained by the fact that MSVC implements `memcpy` function using AVX instructions, fully unrolls
and inlines it, and this doesn't happen with hand-made AVX-based copy function.

Conclusions
-----------

- One can't always rely on the automatic function inlining; sometimes it fails.

- Macros are very powerful as an optimisation technique, but quite error-prone; you must be really careful with them.

- Correctness checking is absolutely essential when you optimise programs.

- Different combinations of processor/OS/compiler may cause different versions of the same routines to be the fastest.
  In some cases the difference is big enough to justify keeping several versions and choosing one at runtime by
means of performance testing or static configuration.
