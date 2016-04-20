---
layout: post
title:  "How to transpose a 16&times;16 byte matrix using SSE"
date:   2014-10-01 12:00:00
tags: C C++ GCC optimisation SSE AVX
story: e1-demux
story-title: "De-multiplexing of E1 streams"
---

Background
----------

In ["{{ site.TITLE-E1-C }}"]({{ site.ART-E1-C }}) we tried different ways to de-multiplex E1 streams using SSE and AVX.
This problem can be described as transposition of a matrix. In our case the dimensions of the source matrix
are 32&times;64 (32 columns and 64 rows, where 32 is the number of timeslots and 64 is the size of each timeslot's output).
The matrix is assumed stored in memory along the rows.

The usual way to transpose this matrix is to divide it into small blocks that fit into available registers,
and transpose each block separately. We tried
this using blocks of size 1&times;4, 1&times;8, 4&times;4, 4&times;16, 8&times;16, 4&times;32 and 8&times;32.
The best results were achieved using 8&times;16 blocks with SSE (120 ms for 1,000,000 iterations),
and 8&times;32 blocks with AVX (109 ms). The times were measured on the Intel&reg; Xeon&reg; CPU E5-2670 @ 2.60GHz.

In the same article I briefly mentioned an attempt to use 16&times;16 blocks. I didn't go into the details then,
to keep the article shorter. Now I changed my mind. Looking later at the code, I considered it rather beautiful. So
today we'll be transposing a 16&times;16 byte matrix.


Transposition
-------------

We'll assume that the matrix is represented as 16 SSE variables (of type `__m128i`), where each variable keeps one
row (16 bytes). The result must be stored the same way.

When designing high-performance matrix manipulations using SSE, we developed two utility functions performing two
important manipulations. The first one considered a single SSE value as a 4&times;4 matrix and transposed it:

{% highlight C++ %}
inline __m128i transpose_4x4 (__m128i m)
{
    return _mm_shuffle_epi8 (m, _mm_setr_epi8 (0, 4, 8, 12,
                                               1, 5, 9, 13,
                                               2, 6, 10, 14,
                                               3, 7, 11, 15));
}
{% endhighlight %}

The second one took four SSE variables, interpreted them as a 4&times;4 matrix of double-words (4-byte entities)
and transposed this matrix:

{% highlight C++ %}
inline void transpose_4x4_dwords (__m128i w0, __m128i w1,
                                  __m128i w2, __m128i w3,
                                  __m128i &r0, __m128i &r1,
                                  __m128i &r2, __m128i &r3)
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

`_128i_shuffle` is our own macro defined in `sse.h`. It is a more convenient version of `_mm_shuffle_ps`.

Now here is the plan: We'll apply the second procedure to four groups of variables, each representing a 16&times;4
matrix, then apply the first procedure to 16 variables, each representing a 4&times;4 matrix, and then apply the
second procedure again to four groups of four registers along the columns. And that's it, the job is done.

It is easy to see why it works. Let's assume that the matrix contains hex values from `0x00` to `0xFF`.
Look at the first four rows of the matrix, stored in four SSE registers, `x0` to `x3`:

<table class="matrix">
<tr><td class="mlabel">x0: </td><td class="l">00</td><td>01</td><td>02</td><td class="r">03</td><td class="l">04</td><td>05</td><td>06</td><td class="r">07</td><td class="l">08</td><td>09</td><td>0A</td><td class="r">0B</td><td class="l">0C</td><td>0D</td><td>0E</td><td class="r">0F</td></tr>
<tr><td class="mlabel">x1: </td><td class="l">10</td><td>11</td><td>12</td><td class="r">13</td><td class="l">14</td><td>15</td><td>16</td><td class="r">17</td><td class="l">18</td><td>19</td><td>1A</td><td class="r">1B</td><td class="l">1C</td><td>1D</td><td>1E</td><td class="r">1F</td></tr>
<tr><td class="mlabel">x2: </td><td class="l">20</td><td>21</td><td>22</td><td class="r">23</td><td class="l">24</td><td>25</td><td>26</td><td class="r">27</td><td class="l">28</td><td>29</td><td>2A</td><td class="r">2B</td><td class="l">2C</td><td>2D</td><td>2E</td><td class="r">2F</td></tr>
<tr><td class="mlabel">x3: </td><td class="l">30</td><td>31</td><td>32</td><td class="r">33</td><td class="l">34</td><td>35</td><td>36</td><td class="r">37</td><td class="l">38</td><td>39</td><td>3A</td><td class="r">3B</td><td class="l">3C</td><td>3D</td><td>3E</td><td class="r">3F</td></tr>
</table>

We want to collect the leftmost 4&times;4 submatrix (from `00` to `33`) into one SSE variable, the next one
(from `04` to `37`) into the next one, and so on. The first sub-matrix consists of first double-words of
all of the source values, the second one of the second ones, and so on. In short, this procedure is in fact transposition
of a 4&times;4 double-word matrix, and it can be done like this:

{% highlight C++ %}
transpose_4x4_dwords (x0, x1, x2, x3, w00, w01, w02, w03);
{% endhighlight %}

This is the result of this procedure:

<table class="matrix">
<tr><td class="mlabel">w00: </td><td class="l">00</td><td>01</td><td>02</td><td class="r">03</td><td class="l">10</td><td>11</td><td>12</td><td class="r">13</td><td class="l">20</td><td>21</td><td>22</td><td class="r">23</td><td class="l">30</td><td>31</td><td>32</td><td class="r">33</td></tr>
<tr><td class="mlabel">w01: </td><td class="l">04</td><td>05</td><td>06</td><td class="r">07</td><td class="l">14</td><td>15</td><td>16</td><td class="r">17</td><td class="l">24</td><td>25</td><td>26</td><td class="r">27</td><td class="l">34</td><td>35</td><td>36</td><td class="r">37</td></tr>
<tr><td class="mlabel">w02: </td><td class="l">08</td><td>09</td><td>0A</td><td class="r">0B</td><td class="l">18</td><td>19</td><td>19</td><td class="r">1A</td><td class="l">28</td><td>29</td><td>2A</td><td class="r">2A</td><td class="l">38</td><td>39</td><td>3A</td><td class="r">3B</td></tr>
<tr><td class="mlabel">w03: </td><td class="l">0C</td><td>0D</td><td>0E</td><td class="r">0F</td><td class="l">1C</td><td>1D</td><td>1E</td><td class="r">1F</td><td class="l">2C</td><td>2D</td><td>2E</td><td class="r">2F</td><td class="l">3C</td><td>3D</td><td>3E</td><td class="r">3F</td></tr>
</table>

After we run this procedure for all four 16&times;4 strips, we'll have the entire source matrix stored in SSE variables
in the following way:

<table class="bigmatrix">
<tr><td><table class="grid"><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr></table><div><p>w00</p></div></td>
    <td><table class="grid"><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr></table><div><p>w01</p></div></td>
    <td><table class="grid"><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr></table><div><p>w02</p></div></td>
    <td><table class="grid"><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr></table><div><p>w03</p></div></td>
</tr>
<tr><td><table class="grid"><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr></table><div><p>w10</p></div></td>
    <td><table class="grid"><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr></table><div><p>w11</p></div></td>
    <td><table class="grid"><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr></table><div><p>w12</p></div></td>
    <td><table class="grid"><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr></table><div><p>w13</p></div></td>
</tr>
<tr><td><table class="grid"><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr></table><div><p>w20</p></div></td>
    <td><table class="grid"><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr></table><div><p>w21</p></div></td>
    <td><table class="grid"><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr></table><div><p>w22</p></div></td>
    <td><table class="grid"><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr></table><div><p>w23</p></div></td>
</tr>
<tr><td><table class="grid"><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr></table><div><p>w30</p></div></td>
    <td><table class="grid"><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr></table><div><p>w31</p></div></td>
    <td><table class="grid"><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr></table><div><p>w32</p></div></td>
    <td><table class="grid"><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr><tr><td></td><td></td><td></td><td></td></tr></table><div><p>w33</p></div></td>
</tr>
</table>

We named the new variables according to their positions in this matrix.

The next step is to transpose each of the `w00` ... `w33` as a 4&times;4 matrix. The `transpose_4x4` will do the job.

Now we are ready to collect the results. We want to put columns of the original 16&times;16 matrix back into
`x0` ... `x15` variables. Before transposition of `w00` ... `w33`, the first column of the source matrix
consisted of the first columns of `w00` ... `w30`, the second one from the second coluns snd so on.
After the transposition we must use the rows, which are exactly the double-word components of the variables.
This means that we need to transpose a double-word matrix again:

{% highlight C++ %}
transpose_4x4_dwords (w00, w10, w20, w30, x0, x1, x2, x3);
{% endhighlight %}

Here is the content of `w00` ... `w30` before this operation:

<table class="matrix">
<tr><td class="mlabel">w00: </td><td class="l">00</td><td>10</td><td>20</td><td class="r">30</td><td class="l">01</td><td>11</td><td>21</td><td class="r">31</td><td class="l">02</td><td>12</td><td>22</td><td class="r">32</td><td class="l">03</td><td>13</td><td>23</td><td class="r">33</td></tr>
<tr><td class="mlabel">w10: </td><td class="l">40</td><td>50</td><td>60</td><td class="r">70</td><td class="l">41</td><td>51</td><td>61</td><td class="r">71</td><td class="l">42</td><td>52</td><td>62</td><td class="r">72</td><td class="l">43</td><td>53</td><td>63</td><td class="r">73</td></tr>
<tr><td class="mlabel">w20: </td><td class="l">80</td><td>90</td><td>A0</td><td class="r">B0</td><td class="l">81</td><td>91</td><td>A1</td><td class="r">B1</td><td class="l">82</td><td>92</td><td>A2</td><td class="r">B2</td><td class="l">83</td><td>93</td><td>A3</td><td class="r">B3</td></tr>
<tr><td class="mlabel">w30: </td><td class="l">C0</td><td>D0</td><td>E0</td><td class="r">F0</td><td class="l">C1</td><td>D1</td><td>E1</td><td class="r">F1</td><td class="l">C2</td><td>D2</td><td>E2</td><td class="r">F2</td><td class="l">C3</td><td>D3</td><td>E3</td><td class="r">F3</td></tr>
</table>

And this is the result (stored back into `x0` ... `x3`):

<table class="matrix">
<tr><td class="mlabel">x0: </td><td class="l">00</td><td>10</td><td>20</td><td class="r">30</td><td class="l">40</td><td>50</td><td>60</td><td class="r">70</td><td class="l">80</td><td>90</td><td>A0</td><td class="r">B0</td><td class="l">C0</td><td>D0</td><td>E0</td><td class="r">F0</td></tr>
<tr><td class="mlabel">x1: </td><td class="l">01</td><td>11</td><td>21</td><td class="r">31</td><td class="l">41</td><td>51</td><td>61</td><td class="r">71</td><td class="l">81</td><td>91</td><td>A1</td><td class="r">B1</td><td class="l">C1</td><td>D1</td><td>E1</td><td class="r">F1</td></tr>
<tr><td class="mlabel">x2: </td><td class="l">02</td><td>12</td><td>22</td><td class="r">32</td><td class="l">42</td><td>52</td><td>62</td><td class="r">72</td><td class="l">82</td><td>92</td><td>A2</td><td class="r">B2</td><td class="l">C2</td><td>D2</td><td>E2</td><td class="r">F2</td></tr>
<tr><td class="mlabel">x3: </td><td class="l">03</td><td>13</td><td>23</td><td class="r">33</td><td class="l">43</td><td>53</td><td>63</td><td class="r">73</td><td class="l">83</td><td>93</td><td>A3</td><td class="r">B3</td><td class="l">C3</td><td>D3</td><td>E3</td><td class="r">F3</td></tr>
</table>

You can see that the variables contain four first columns of the original matrix. If we perform this operation
on other groups of four variables, we'll get the entire matrix.

Here is the code:

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

De-multiplexing of the E1 stream using the 16&times;16 transposition
--------------------------------------------------------------------

This is what the de-multiplexing procedure looks like:

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

#define LOADREG(i) __m128i w##i = \
                _128i_load (&src [(dst_pos + i) * NUM_TIMESLOTS + dst_num])

#define STOREREG(i) _128i_store (&d##i [dst_pos], w##i)

                LOADREG (0);  LOADREG (1);  LOADREG (2);  LOADREG (3);
                LOADREG (4);  LOADREG (5);  LOADREG (6);  LOADREG (7);
                LOADREG (8);  LOADREG (9);  LOADREG (10); LOADREG (11);
                LOADREG (12); LOADREG (13); LOADREG (14); LOADREG (15);
                transpose_16x16 (w0,  w1, w2,  w3,  w4,  w5,  w6,  w7,
                                 w8,  w9, w10, w11, w12, w13, w14, w15);
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

At first this code may look very inefficient. The `transpose_16x16` function is called with 16 parameters passed
by reference. In general case this requires all the variables to be stored in memory and their addresses passed
into the function. The same applies to calls to `transpose_4x4_dwords`, which returns results via the reference
parameters. If those calls aren't inlined, this is exactly what will happen, so for this code to be efficient,
the inlining is absolutely essential.

If all the calls are inlined, the functions act as safe macros, and all the references are optimised out.
There is no need to store all the variables in memory then. The code may end up relatively efficient, although some performance penalties are inevitable.

First of all, no matter how the compiler reorders the instructions, there is always some place in the code where
there are at least 16 live SSE variables. In addition, some registers are needed for temporary values. Since SSE
only has 16 registers, there is definite register shortage here, and some values will end up in memory.

Secondly,
the same applies to pointers `d0` ... `d15`, which are all alive inside the inner loop. The processor has 16
general-purpose registers, so there is a shortage here as well. The impact of this shortage may depend on the
compiler and the processor in use.

Achieved performance
--------------------

On our test machine this methods takes 176ms, which is good result, but not the best. The best time was achieved using
8&times;32 blocks and AVX instructions.

I also tried it on my notebook (Intel(R) Core(TM) i7-4800MQ CPU @2.70 GHz), using Microsoft Visual C++ 2013 in 64-bit
mode. This is what I got:

    >e1-new.exe
    class Reference: 1991
    class Write4: 578
    class Write8: 545
    class Read4_Write4: 685
    class Read4_Write4_Unroll: 649
    class Read4_Write4_SSE: 317
    class Read4_Write16_SSE: 235
    class Read8_Write16_SSE: 236
    class Read8_Write16_SSE_Unroll: 236
    class Read16_Write16_SSE: 208
    class Read16_Write16_SSE_Unroll: 204
    class Read4_Write32_AVX: 275
    class Read8_Write32_AVX: 260
    class Read8_Write32_AVX_Unroll: 256

We can see that on Windows our code is the fastest.
This is why we mustn't discard it -- it can still be useful.
Besides, it allows easy adaptation for AVX2 once it becomes available for testing. On AVX2 we can keep two rows
of the matrix in one AVX register, and the entire matrix will require 8 registers. We need AVX2 for that, because
AVX does not have byte manipulation instruction (`PSHUFB`).

Conclusions
-----------

- The code for this method is quite beautiful (perhaps, the most beautiful of all).

- However, the most beautiful isn't always the fastest. But sometimes it is.

- Different methods may be the best for different execution environments. Perhaps, the correct thing to do is to
  keep several versions and choose the best one at run time -- either by means of static configuration, or by
  automatic discovery.

Coming soon
-----------

This method is still not unrolled. I'll do it in the next article.
