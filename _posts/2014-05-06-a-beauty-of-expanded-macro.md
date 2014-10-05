---
layout: post
title:  "Beauty of expanded macro"
date:   2014-05-06 00:00:00
tags: C C++ macro meta-programming
---

In [one of the previous articles]({{ site.ART-E1-C }})
we've made a macro multiplier, which is a macro that calls another macro given number of times.
This multiplier was used for writing unrolled loops. The main unrolled macro looked like this:

{% highlight C++ %}
#define MOVE_BYTE(i,j) d[i] = src [(j)+(i)*32]

#define MOVE_BYTES_64(j) DUP_2_64 (MOVE_BYTE,j)
#define MOVE_TIMESLOT(j) do {\
        byte * const d = dst[j];\
        MOVE_BYTES_64 (j);\
    } while (0)
{% endhighlight %}

We had two versions of DUP2_64: [one that iterated indices manually]({{ site.REPO-E1-C }}/blob/1999ca98b6308ff9107e8ebf789e5615f822012e/mymacros.h):

{% highlight C++ %}
#define DUP2_2(m,j)  do {               m(0,j); m(1,j); } while (0)
#define DUP2_4(m,j)  do { DUP2_2(m,j);  m(2,j); m(3,j); } while (0)
#define DUP2_8(m,j)  do { DUP2_4(m,j);  m(4,j); m(5,j); m(6,j); m(7,j); } while (0)
// and so on until DUP2_64
{% endhighlight %}

This is the one we finally used in the project.
[Another version was relying on calling previously defined macros](({{ site.REPO-E1-C }}/blob/f8f03136427227494847db0d52cc726fc0e629d8/mymacros.h)):

{% highlight C++ %}
#define DUP2_2_(m, index, j)  do { m (index, j); m (index+1, j); } while (0)
#define DUP2_4_(m, index, j)  do { DUP2_2_ (m, index, j); DUP2_2_ (m, index+2, j); } while (0)
#define DUP2_8_(m, index, j)  do { DUP2_4_ (m, index, j); DUP2_4_ (m, index+4, j); } while (0)
// and so on until DUP2_64

#define DUP2_2(m)  DUP_2_ (m, 0)
#define DUP2_4(m)  DUP_4_ (m, 0)
#define DUP2_8(m)  DUP_8_ (m, 0)
// and so on until DUP2_64
{% endhighlight %}

It is interesting to see what our macros expand to. Let's see what a simple invocation `MOVE_TIMESLOT (j)` produces
using our current set of macros from `mymacros.h`. A `-E` key in `gcc` command line helps with that:

Both versions are expanded into a piece of code of great elegance and beauty. See here:

The first version produces the following (I've broken the text into lines, originally it was a single line
of 1857 characters):

{% highlight C++ %}
do { byte * const d = dst[j]; do { do { do { do { do { do { d[0] = src [(j)
+(0)*32]; d[1] = src [(j)+(1)*32]; } while (0); d[2] = src [(j)+(2)*32];
d[3] = src [(j)+(3)*32]; } while (0); d[4] = src [(j)+(4)*32]; d[5] = src
[(j)+(5)*32]; d[6] = src [(j)+(6)*32]; d[7] = src [(j)+(7)*32]; } while (0);
d[8] = src [(j)+(8)*32]; d[9] = src [(j)+(9)*32]; d[10] = src [(j)+(10)*32];
d[11] = src [(j)+(11)*32]; d[12] = src [(j)+(12)*32]; d[13] = src [(j)+(13)*
32]; d[14] = src [(j)+(14)*32]; d[15] = src [(j)+(15)*32]; } while (0); d[16]
= src [(j)+(16)*32]; d[17] = src [(j)+(17)*32]; d[18] = src [(j)+(18)*32];
d[19] = src [(j)+(19)*32]; d[20] = src [(j)+(20)*32]; d[21] = src [(j)+(21)*
32]; d[22] = src [(j)+(22)*32]; d[23] = src [(j)+(23)*32]; d[24] = src [(j)+
(24)*32]; d[25] = src [(j)+(25)*32]; d[26] = src [(j)+(26)*32]; d[27] = src
[(j)+(27)*32]; d[28] = src [(j)+(28)*32]; d[29] = src [(j)+(29)*32]; d[30]
= src [(j)+(30)*32]; d[31] = src [(j)+(31)*32]; } while (0); d[32] = src
[(j)+(32)*32]; d[33] = src [(j)+(33)*32]; d[34] = src [(j)+(34)*32]; d[35]
= src [(j)+(35)*32]; d[36] = src [(j)+(36)*32]; d[37] = src [(j)+(37)*32];
d[38] = src [(j)+(38)*32]; d[39] = src [(j)+(39)*32]; d[40] = src [(j)+(40)
*32]; d[41] = src [(j)+(41)*32]; d[42] = src [(j)+(42)*32]; d[43] = src [(j)
+(43)*32]; d[44] = src [(j)+(44)*32]; d[45] = src [(j)+(45)*32]; d[46] = src
[(j)+(46)*32]; d[47] = src [(j)+(47)*32]; d[48] = src [(j)+(48)*32]; d[49] =
src [(j)+(49)*32]; d[50] = src [(j)+(50)*32]; d[51] = src [(j)+(51)*32]; d
[52] = src [(j)+(52)*32]; d[53] = src [(j)+(53)*32]; d[54] = src [(j)+(54)
*32]; d[55] = src [(j)+(55)*32]; d[56] = src [(j)+(56)*32]; d[57] = src
[(j)+(57)*32]; d[58] = src [(j)+(58)*32]; d[59] = src [(j)+(59)*32]; d
[60] = src [(j)+(60)*32]; d[61] = src [(j)+(61)*32]; d[62] = src [(j)+(62)
*32]; d[63] = src [(j)+(63)*32]; } while (0); } while (0)
{% endhighlight %}

And this is the result of the second version, it was 4052 characters long:

{% highlight C++ %}
do { byte * const d = dst[j]; do { do { do { do { do { do { d[0] = src [(j)
+(0)*32]; d[0 +1] = src [(j)+(0 +1)*32]; } while (0); do { d[0 +2] = src
[(j)+(0 +2)*32]; d[0 +2 +1] = src [(j)+(0 +2 +1)*32]; } while (0); } while
(0); do { do { d[0 +4] = src [(j)+(0 +4)*32]; d[0 +4 +1] = src [(j)+(0 +4
+1)*32]; } while (0); do { d[0 +4 +2] = src [(j)+(0 +4 +2)*32]; d[0 +4 +2
+1] = src [(j)+(0 +4 +2 +1)*32]; } while (0); } while (0); } while (0); do
{ do { do { d[0 +8] = src [(j)+(0 +8)*32]; d[0 +8 +1] = src [(j)+(0 +8 +1)
*32]; } while (0); do { d[0 +8 +2] = src [(j)+(0 +8 +2)*32]; d[0 +8 +2 +1]
 = src [(j)+(0 +8 +2 +1)*32]; } while (0); } while (0); do { do { d[0 +8 +
4] = src [(j)+(0 +8 +4)*32]; d[0 +8 +4 +1] = src [(j)+(0 +8 +4 +1)*32]; }
while (0); do { d[0 +8 +4 +2] = src [(j)+(0 +8 +4 +2)*32]; d[0 +8 +4 +2 +1]
= src [(j)+(0 +8 +4 +2 +1)*32]; } while (0); } while (0); } while (0); }
while (0); do { do { do { do { d[0 +16] = src [(j)+(0 +16)*32]; d[0 +16
+1] = src [(j)+(0 +16 +1)*32]; } while (0); do { d[0 +16 +2] = src [(j)+
(0 +16 +2)*32]; d[0 +16 +2 +1] = src [(j)+(0 +16 +2 +1)*32]; } while (0); }
while (0); do { do { d[0 +16 +4] = src [(j)+(0 +16 +4)*32]; d[0 +16 +4 +1] =
src [(j)+(0 +16 +4 +1)*32]; } while (0); do { d[0 +16 +4 +2] = src [(j)+(0 +
16 +4 +2)*32]; d[0 +16 +4 +2 +1] = src [(j)+(0 +16 +4 +2 +1)*32]; } while
(0); } while (0); } while (0); do { do { do { d[0 +16 +8] = src [(j)+(0 +16
+8)*32]; d[0 +16 +8 +1] = src [(j)+(0 +16 +8 +1)*32]; } while (0); do { d[0
+16 +8 +2] = src [(j)+(0 +16 +8 +2)*32]; d[0 +16 +8 +2 +1] = src [(j)+(0 +16
+8 +2 +1)*32]; } while (0); } while (0); do { do { d[0 +16 +8 +4] = src [(j)
+(0 +16 +8 +4)*32]; d[0 +16 +8 +4 +1] = src [(j)+(0 +16 +8 +4 +1)*32]; }
while (0); do { d[0 +16 +8 +4 +2] = src [(j)+(0 +16 +8 +4 +2)*32]; d[0 +16
+8 +4 +2 +1] = src [(j)+(0 +16 +8 +4 +2 +1)*32]; } while (0); } while (0);
} while (0); } while (0);} while (0); do { do { do { do { do { d[0 +32] =
src [(j)+(0 +32)*32]; d[0 +32 +1] = src [(j)+(0 +32 +1)*32]; } while (0);
do { d[0 +32 +2] = src [(j)+(0 +32 +2)*32]; d[0 +32 +2 +1] = src [(j)+(0
+32 +2 +1)*32]; } while (0); } while (0); do { do { d[0 +32 +4] = src [(j)
+(0 +32 +4)*32]; d[0 +32 +4 +1] = src [(j)+(0 +32 +4 +1)*32]; } while (0);
do { d[0 +32 +4 +2] = src [(j)+(0 +32 +4 +2)*32]; d[0 +32 +4 +2 +1] = src
[(j)+(0 +32 +4 +2 +1)*32]; } while (0); } while (0); } while (0); do { do
{ do { d[0 +32 +8] = src [(j)+(0 +32 +8)*32]; d[0 +32 +8 +1] = src [(j)+(0
+32 +8 +1)*32]; } while (0); do { d[0 +32 +8 +2] = src [(j)+(0 +32 +8 +2)*
32]; d[0 +32 +8 +2 +1] = src [(j)+(0 +32 +8 +2 +1)*32]; } while (0); }
while (0); do { do { d[0 +32 +8 +4] = src [(j)+(0 +32 +8 +4)*32]; d[0 +32
+8 +4 +1] = src [(j)+(0 +32 +8 +4 +1)*32]; } while (0); do { d[0 +32 +8 +4
+2] = src [(j)+(0 +32 +8 +4 +2)*32]; d[0 +32 +8 +4 +2 +1] = src [(j)+(0 +32
+8 +4 +2 +1)*32]; } while (0); } while (0); } while (0); } while (0); do {
do { do { do { d[0 +32 +16] = src [(j)+(0 +32 +16)*32]; d[0 +32 +16 +1] =
src [(j)+(0 +32 +16 +1)*32]; } while (0); do { d[0 +32 +16 +2] = src [(j)
+(0 +32 +16 +2)*32]; d[0 +32 +16 +2 +1] = src [(j)+(0 +32 +16 +2 +1)*32];
} while (0); } while (0); do { do { d[0 +32 +16 +4] = src [(j)+(0 +32 +16
+4)*32]; d[0 +32 +16 +4 +1] = src [(j)+(0 +32 +16 +4 +1)*32]; } while (0);
do { d[0 +32 +16 +4 +2] = src [(j)+(0 +32 +16 +4 +2)*32]; d[0 +32 +16 +4
+2 +1] = src [(j)+(0 +32 +16 +4 +2 +1)*32]; } while (0); } while (0); }
while (0); do { do { do { d[0 +32 +16 +8] = src [(j)+(0 +32 +16 +8)*32];
d[0 +32 +16 +8 +1] = src [(j)+(0 +32 +16 +8 +1)*32]; } while (0); do { d
[0 +32 +16 +8 +2] = src [(j)+(0 +32 +16 +8 +2)*32]; d[0 +32 +16 +8 +2 +1]
= src [(j)+(0 +32 +16 +8 +2 +1)*32]; } while (0); } while (0); do { do {
d[0 +32 +16 +8 +4] = src [(j)+(0 +32 +16 +8 +4)*32]; d[0 +32 +16 +8 +4 +1]
= src [(j)+(0 +32 +16 +8 +4 +1)*32]; } while (0); do { d[0 +32 +16 +8 +4
+2] = src [(j)+(0 +32 +16 +8 +4 +2)*32]; d[0 +32 +16 +8 +4 +2 +1] = src
[(j)+(0 +32 +16 +8 +4 +2 +1)*32]; } while (0); } while (0); } while (0);
} while (0);} while (0);} while (0); } while (0)
{% endhighlight %}

Isn't it impressive and very, very beautiful?
