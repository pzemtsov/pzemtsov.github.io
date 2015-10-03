---
layout: post
title:  "Optimising division-based hash functions"
date:   2015-09-17 12:00:00
tags: Java optimisation hashmap life
story: life
story-title: "Conway's Game of Life"
---

In ["{{ site.TITLE-HASH }}"]({{ site.ART-HASH }}) we implemented [Conway's game of Life](http://en.wikipedia.org/wiki/Conway's_Game_of_Life)
using **Java**'s `HashMap` and discovered that some hash functions cause higher execution speed than the others.
We introduced several hash functions and measured their distribution of hash codes. In the next article
(["{{ site.TITLE-MEASURING-HASH }}"]({{ site.ART-MEASURING-HASH }})) we learned how to achieve stable performance of
micro-benchmarks (those that measure execution speed of individual hash functions), and measured the speed of all those
functions. We found several abnormalities, one of which was a big difference in performance of the division-based
hash function between **Java 7** and **Java 8**. Let's look at this case.

The function in question
------------------------

Let's repeat the [source code]({{ site.REPO-LIFE }}/blob/28d4e13343b954bdaa4eab3ab27693debd8ce348/LongPoint6.java) of `LongPoint6.hashCode()`:

{% highlight Java %}
    public int hashCode ()
    {
        return (int) (v % 946840871);
    }
{% endhighlight %}

The constant 946840871 was a completely arbitrary choice of a divisor. It's a prime number and it is relatively big
(close to one billion), bigger than any realistic hash table size. There is nothing special about this number, some other number
may work better. It is also possible that some other number will allow faster calculations. We don't use any specific
properties of this number here, all following arguments are equally applicable to any number. 

The test involving this method ran for 578 ms on **Java 7** and 1661 ms on **Java 8**. The only way to find out why is
by looking at the generated code (see the [previous article]({{ site.ART-MEASURING-HASH }}) for the instructions how to do it).

Let's start with the **Java 7**:

{% highlight c-objdump %}
  0x00007f88e10684a0: sub    rsp,0x18
  0x00007f88e10684a7: mov    QWORD PTR [rsp+0x10],rbp  ;*synchronization entry
                                                ; - LongPoint6::hashCode@-1 (line 30)
  0x00007f88e10684ac: mov    r10,QWORD PTR [rsi+0x10]  ;*getfield v
                                                ; - LongPoint6::hashCode@1 (line 30)
  0x00007f88e10684b0: mov    r11,r10
  0x00007f88e10684b3: sar    r11,0x3f
  0x00007f88e10684b7: mov    rax,0x2449f0232c624b0b
  0x00007f88e10684c1: imul   r10
  0x00007f88e10684c4: sar    rdx,0x1b
  0x00007f88e10684c8: sub    rdx,r11
  0x00007f88e10684cb: imul   r11,rdx,0x386fa527
  0x00007f88e10684d2: sub    r10,r11
  0x00007f88e10684d5: mov    eax,r10d           ;*l2i  ; - LongPoint6::hashCode@8 (line 30)
  0x00007f88e10684d8: add    rsp,0x10
  0x00007f88e10684dc: pop    rbp
  0x00007f88e10684dd: test   DWORD PTR [rip+0xb74fb1d],eax        # 0x00007f88ec7b8000
                                                ;   {poll_return}
  0x00007f88e10684e3: ret    
{% endhighlight %}
 
There aren't any division instructions in the code. The pseudo-code looks like this:

{% highlight Java %}
    public int hashCode ()
    {
        long sign = v >> 63;
        long div = ((v ** 2614885092524444427L) >> 91) - sign;
        return (int) (v - div * 946840871);
    }
{% endhighlight %}
 
I call this pseudo-code rather than just code because it contains an operation that is not accessible in **Java**:
a 128-bit multiplication, which takes two 64-bit numbers and produces a 128-bit result (I wrote it as `**`). There is such operation
in the instruction set -- both `MUL` and `IMUL` have 128-bit versions. The result
is placed into `rdx` and `rax`, we are using `rdx`, that's why the total shift is 0x1B + 64 = 91.

What we see is the result of a well-known optimisation: replacing of division by a constant divisor with a multiplication
and some shifts. It is well described in [Agner Fog. Optimizing subroutines in assembly
language. An optimization guide for x86 platforms](http://agner.org/optimize/optimizing_assembly.pdf). The HotSpot
uses slightly different algorithm (they use two bits less, and compensate for signed values). If the code was written
for unsigned numbers, it could save three instructions here.

The **Java 8** code looks like this:

{% highlight c-objdump %}
  0x0000000002989a00: mov    DWORD PTR [rsp-0x6000],eax
  0x0000000002989a07: push   rbp
  0x0000000002989a08: sub    rsp,0x50           ;*aload_0
                                                ; - LongPoint6::hashCode@0 (line 30)

  0x0000000002989a0c: mov    rcx,QWORD PTR [rdx+0x10]  ;*getfield v
                                                ; - LongPoint6::hashCode@1 (line 30)

  0x0000000002989a10: mov    rdx,rcx
  0x0000000002989a13: movabs rcx,0x386fa527
  0x0000000002989a1d: mov    rsi,rcx
  0x0000000002989a20: mov    rcx,rsi
  0x0000000002989a23: cmp    rsi,0x0
  0x0000000002989a27: je     0x0000000002989a40
  0x0000000002989a2d: call   0x000000006fe379a0  ;*lrem
                                                ; - LongPoint6::hashCode@7 (line 30)
                                                ;   {runtime_call}
  0x0000000002989a32: mov    eax,eax
  0x0000000002989a34: add    rsp,0x50
  0x0000000002989a38: pop    rbp
  0x0000000002989a39: test   DWORD PTR [rip+0xfffffffffd7a66c1],eax        # 0x0000000000130100
                                                ;   {poll_return}
  0x0000000002989a3f: ret
  0x0000000002989a40: call   0x0000000002959c80  ; OopMap{off=101}
                                                ;*lrem
                                                ; - LongPoint6::hashCode@7 (line 30)
                                                ;   {runtime_call}
{% endhighlight %}

This code does not employ any optimisations. It does not even compile the remainder operation into `DIV` or `IDIV`
instructions. It calls a runtime routine. It even goes as far as checking that freshly loaded constant isn't zero.
In short, this is really awful code, no wonder it runs slowly. Perhaps, HotSpot developers considered division by
a constant a rare and unimportant case, and removed corresponding optimisation to keep their code short.


Optimising division: unsigned case
----------------------------------

The code **Java 7** generates is perfect. The only way it could be improved is by using unsigned division instead of
signed one, but this can't be done in **Java**.

Is there any way to improve the **Java 8** case? If our numbers were 32-bit, there would be an obvious way -- to write
code similar to the pseudo-code above; instead of 128-bit multiplication, we would need a 64-bit one, and this is a
normal `long` operation. Unfortunately, we can't do it that easily in the 64-bit case, because, as I said, there is
no `**` operation in **Java**.

This doesn't mean we must stop here: after all, we have a general-purpose computer, and there is nothing mystical
about 128-bit multiplication. It can be programmed in **Java** using available arithmetic.

Let's first simplify the problem by considering our numbers unsigned.

Let `x` and `y` be unsigned 64-bit values, and `A`, `B`, `C` and `D` be unsigned 32-bit values,
where

 <div class="formula">
  x = 2<sup>32</sup>A + B<br>
  y = 2<sup>32</sup>C + D
 </div>

Then

 <div class="formula">
  xy = (2<sup>32</sup>A + B) (2<sup>32</sup>C + D) = 2<sup>64</sup>AC + 2<sup>32</sup>(AD + BC) + BD
 </div>

or, graphically,

<table>
 <tr><th colspan="2"> high 64 bits of xy </th> <th colspan="2"> low 64 bits of xy </th> </tr>
 <tr><td> hi (A*C)         </td><td> lo (A*C)</td> </tr>
 <tr><td class="noborder"> </td><td> hi (A*D)</td>         <td> lo (A*D) </td> </tr>
 <tr><td class="noborder"> </td><td> hi (B*C)</td>         <td> lo (B*C) </td> </tr>
 <tr><td class="noborder"> </td><td class="noborder"> </td><td> hi (B*D) </td> <td> lo (B*D)</td> </tr>
</table>

This is a 128-bit unsigned number, which we will represent as two `long`s, one for high and one for low part of the result.
Both will be interpreted as unsigned 64-bit values. Each of the terms `AC`, `AD`, `BC`, `BD` is a 64-bit number.
The low part of the result can be calculated by the obvious code:

{% highlight Java %}
    long mult_unsigned_lopart (long x, long y)
    {
        long A = uhi (x);
        long B = ulo (x);
        long C = uhi (y);
        long D = ulo (y);
        return B*D + ((A*D + B*C) << 32);
    }

    long uhi (long x)
    {
        return hi (x) & 0xFFFFFFFFL;
    }

    long ulo (long x)
    {
        return lo (x) & 0xFFFFFFFFL;
    }

{% endhighlight %}

Note the `uhi` and `ulo` functions ("unsigned `hi`" and "unsigned `lo`") and their difference from `hi` and `lo` that
we used to extract integer parts of a value stored as `long`:

{% highlight Java %}
    public static int hi (long w)
    {
        return (int) (w >>> 32);
    }

    public static int lo (long w)
    {
        return (int) w;
    }
{% endhighlight %}

These functions return `int`, which, if directly converted to `long`, will be sign-extended, which will corrupt the results.
Here is a simple example. Let `x` be 0x0000000000000001 and `y` be 0xFFFFFFFFFFFFFFFF. Obviously, the 128-bit result should be
0x0000000000000000FFFFFFFFFFFFFFFF, and `mult_unsigned_lopart()` must return 0xFFFFFFFFFFFFFFFF (which will be printed as -1).
However, with the sign-extended versions of `hi` and `lo` we would have

 <div class="formula">
  A = 0 <br>
  B = 1 <br>
  C = &minus;1 <br>
  D = &minus;1 <br>
  BD + ((AD + BC) << 32) <br>
  <ul> = &minus;1 + ((0 + &minus;1) << 32) <br>
       = 0xFFFFFFFFFFFFFFFF + 0xFFFFFFFF00000000 <br>
       = 0xFFFFFFFEFFFFFFFF
 </div>

The code using `uhi`, `ulo` works correctly:

 <div class="formula">
  A = 0 <br>
  B = 1 <br>
  C = 0xFFFFFFFF <br>
  D = 0xFFFFFFFF <br>
  BD + ((AD + BC) << 32) <br>
  <ul> = 0xFFFFFFFF + ((0 + 0xFFFFFFFF) << 32) <br>
       = 0xFFFFFFFF + 0xFFFFFFFF00000000 <br>
       = 0xFFFFFFFFFFFFFFF
 </div>

The `mult_unsigned_lopart` was shown here for completeness; we don't need it for our remainder problem.
The pseudo-code of `hashCode()` above (the one using the `**` operation) shifts
the multiplication result by 91, which totally ignores the low part of the product.

It may seem that the following code will calculate the high part:

{% highlight Java %}
    static long mult_unsigned_hipart (long x, long y)
    {
        long A = uhi (x);
        long B = ulo (x);
        long C = uhi (y);
        long D = ulo (y);
        return A*C + ((A*D + B*C) >>> 32);
    }
{% endhighlight %}

This, however, does not work --  for two reasons. One is that `(A*D + B*C)` can occupy 65 bits (a carry bit may be produced
during addition), and another one is that, although `B*D` term is not included directly into the result, it can
produce carry when its high part is added to the low part of `A*D + B*C`. We have to perform addition in 32-bit blocks
and add all the carry bits manually, according to the picture above:

{% highlight Java %}
    static long mult_unsigned_hipart (long x, long y)
    {
        long A = uhi (x);
        long B = ulo (x);
        long C = uhi (y);
        long D = ulo (y);

        long AC = A * C;
        long AD = A * D;
        long BC = B * C;
        long BD = B * D;

        long ADl_BCl_BDh = ulo (AD) + ulo (BC) + uhi (BD);
        return AC + uhi (AD) + uhi (BC) + uhi (ADl_BCl_BDh);
    }
{% endhighlight %}

Now it is very easy to program the hash function:

{% highlight Java %}
    public int hashCode ()
    {
        long div = mult_unsigned_hipart (v, 2614885092524444427L) >> 27;
        return (int) (v - div * 946840871);
    }
{% endhighlight %}

It seems very unlikely that this monster will work faster than the original code. It contains five multiplications, one
shift and six additions. This makes it a very interesting test: how fast can it possibly run?

Signed or unsigned?
-------------------

We have just calculated an unsigned version of the remainder-based hash function. However, the code of  `LongPoint6.hashCode()`
uses regular **Java**'s arithmetic, which is signed. How important is that? What impact does it have on the quality of hashing?
In **Java** dividing a negative number by a positive one produces negative remainder:

 <div class="formula">
  25 % 3 = 1<br>
  &minus;25 % 3 = &minus;1
 </div>

Strictly speaking, this isn't what is usually called "a remainder" in mathematics. Proper mathematical remainder isn't ever negative:

 <div class="formula">
  0 &le; x % y &lt; y
 </div>

and should be 2 for &minus;25 % 3.
 
To check the quality of unsigned hashing, we must emulate unsigned calculations in **Java**.
A positive signed 64-bit number keeps its value when interpreted as unsigned. A negative number becomes a big
positive, 2<sup>64</sup> added to its value, so we must add (2<sup>64</sup> mod 946840871) = 518863773 to the
remainder. Since the remainder was zero or negative before, it won't exceed the divisor, but it may stay negative,
in which case we need to add a divisor again:

{% highlight Java %}
    public int hashCode ()
    {
        int r = (int) (v % 946840871);
        if (v < 0) r += 518863773;
        if (r < 0) r += 946840871;
        return r;
    }
{% endhighlight %}

Here are the slot counts we got with signed division (see [here]({{ site.ART-LIFE }})):

    Field size: 1034; slots used:  982; avg= 1.05
    Count size: 3938; slots used: 3236; avg= 1.22

And here are the results for unsigned division:

    Field size: 1034; slots used:  968; avg= 1.07
    Count size: 3938; slots used: 3133; avg= 1.26

The results haven't changed much for `field` but have changed for `count`: they became worse. It does not mean they became
very bad. Previously, the result was equal to E + 5.28&sigma;, where E is the expected value 3126.69, and &sigma;
is the standard deviation 20.68. Now the result is E + 0.3&sigma;, which still falls well within good statistical
limits and performs better than some other hash functions. It just does not
perform as well as the signed division, which I remarked on back then as demonstrating unusually good behaviour.

This means that if we manage to write an incredibly fast version of the hash function based on the unsigned division,
we have chances to improve the overall speed, otherwise the signed version is a better choice.

Another approach is to change our `OFFSET`, which is now 0x80000000, to some positive value. This will leave less bits
for `x` and `y` parts, but the `long` values will always be positive, and unsigned version will produce the same result as
the signed one. Some of these values perform better than the others: for instance, 0x40000000 gives 3118 (E &minus; 0.42&sigma;)
as `count` size, while 0x8000000 gives 3234 (E + 5.18&sigma;). We won't do it to keep the story short.

Optimising division: signed case
--------------------------------

We've seen that the signed version has some potential advantages. Can we modify the code to work in signed case?

The formulae we used above:

 <div class="formula">
  x = 2<sup>32</sup>A + B<br>
  y = 2<sup>32</sup>C + D<br>
  xy = (2<sup>32</sup>A + B) (2<sup>32</sup>C + D)<br>
  <ul>= 2<sup>64</sup>AC + 2<sup>32</sup>(AD + BC) + BD</ul>
 </div>

are still applicable if _x_ and _y_ are signed 64-bit numbers. In this case A and C are signed 32-bit values,
while B and D are unsigned. The picture stays the same, except for important change: AD and BC can now be negative,
so the values must be sign-extended before addition:

<table>
 <tr><th colspan="2"> high 64 bits of xy </th> <th colspan="2"> low 64 bits of xy </th> </tr>
 <tr><td> hi (A*C)   </td><td> lo (A*C)</td> </tr>
 <tr><td> sign (A*D) </td><td> hi (A*D)</td>       <td> lo (A*D) </td> </tr>
 <tr><td> sign (B*C) </td><td> hi (B*C)</td>       <td> lo (B*C) </td> </tr>
 <tr><td class="noborder"> </td><td class="noborder"> </td><td> hi (B*D) </td> <td> lo (B*D)</td> </tr>
</table>

Here sign(v) is a 32-bit sign extension of v. Note that BD must not be sign-extended, since it is a positive value.
These sign manipulations can be implemented very easily by using `hi` and `lo` instead of `uhi` and `ulo` where appropriate:

{% highlight Java %}
    static long mult_signed_hipart (long x, long y)
    {
        long A = hi (x);
        long B = ulo (x);
        long C = hi (y);
        long D = ulo (y);
                   
        long AC = A * C;
        long AD = A * D;
        long BC = B * C;
        long BD = B * D;
        
        long ADl_BCl_BDh = ulo (AD) + ulo (BC) + uhi (BD);
        return AC + hi (AD) + hi (BC) + uhi (ADl_BCl_BDh);
    }
{% endhighlight %}

Finally, the signed version of the hash function:

{% highlight Java %}
    public int hashCode ()
    {
        long sign = v >> 63;
        long div = (mult_signed_hipart (v, 2614885092524444427L) >> 27) - sign;
        return (int) (v - div * 946840871);
    }
{% endhighlight %}

Performance measurements
------------------------

Let's add new unsigned and signed versions to our hash function test suit (they'll be called
[`LongPoint60`]({{ site.REPO-LIFE }}/blob/d59b6b9a681d60ff011a6b800f8d5f63851eeb83/LongPoint60.java)
 and
[`LongPoint61`]({{ site.REPO-LIFE }}/blob/d59b6b9a681d60ff011a6b800f8d5f63851eeb83/LongPoint61.java)
to indicate the fact that they are modified versions of `LongPoint6`).

The full new test suite is [here]({{ site.REPO-LIFE }}/tree/d59b6b9a681d60ff011a6b800f8d5f63851eeb83).
The class to run is `HashTime1`.

Here are the results (in milliseconds for 100M iterations):

<table class="numeric">
<tr><th>Class name</th>                <th>Comment</th>                           <th>Time, <b>Java 7</b></th><th>Time, <b>Java 8</b></th></tr>
<tr><td class="label">Point</td>       <td class="ttext">Multiply by 3, 5                  </td><td>  548</td><td>  485</td></tr>
<tr><td class="label">Long</td>        <td class="ttext"><code>Long</code> default         </td><td>  547</td><td>  486</td></tr>
<tr><td class="label">LongPoint</td>   <td class="ttext">Multiply by 3, 5                  </td><td>  547</td><td>  486</td></tr>
<tr><td class="label">LongPoint3</td>  <td class="ttext">Multiply by 11, 17                </td><td>  547</td><td>  485</td></tr>
<tr><td class="label">LongPoint4</td>  <td class="ttext">Multiply by two big primes        </td><td>  547</td><td>  486</td></tr>
<tr><td class="label">LongPoint5</td>  <td class="ttext">Multiply by one big prime         </td><td>  547</td><td>  486</td></tr>
<tr><td class="label">LongPoint6</td>  <td class="ttext">Modulo big prime                  </td><td>  578</td><td> 1655</td></tr>
<tr><td class="label">LongPoint60</td> <td class="ttext">Modulo optimised, unsigned        </td><td>  640</td><td>  610</td></tr>
<tr><td class="label">LongPoint61</td> <td class="ttext">Modulo optimised, signed          </td><td>  729</td><td>  727</td></tr>
<tr><td class="label">LongPoint7</td>  <td class="ttext"><code>java.util.zip.CRC32</code>  </td><td>10109</td><td> 1882</td></tr>
<tr><td class="label">NullPoint</td>   <td class="ttext">Null function (does nothing)      </td><td>  532</td><td>  455</td></tr>
</table>

The results are surprising. Our optimisation did work! The code above, being rather long and complex, still runs faster than
the modulo operation as implemented by **Java 8**. And quite a bit faster: if we subtract the calling overhead
(measured as the time of `NullPoint`), we get:

<table class="numeric">
<tr><th>Class name</th>                <th>Comment</th>                              <th>Time, <b>Java&nbsp;7</b></th><th>Time, <b>Java&nbsp;8</b></th></tr>
<tr><td class="label">LongPoint6</td>  <td class="ttext">Modulo big prime            </td><td>  46</td><td> 1200</td></tr>
<tr><td class="label">LongPoint60</td> <td class="ttext">Modulo optimised, unsigned  </td><td> 108</td><td>  155</td></tr>
<tr><td class="label">LongPoint61</td> <td class="ttext">Modulo optimised, signed    </td><td> 197</td><td>  272</td></tr>
</table>

Some observations:

- On **Java 8**, the optimised code, with its five multiplications, is still 7.4 times faster than the original code in unsigned case
and 4.4 times faster in signed case.
- The signed case is quite a bit slower than the unsigned case (I'm not sure why).
- **Java 8** is quite a bit slower than **Java 7** in all cases; it just happens that the optimised versions in **Java 8**
are faster than the original one. Nothing beats unoptimised **Java 7** version.
- In CPU cycles the times are: 

<table class="numeric">
<tr><th>Class name</th>                <th>Comment</th>                              <th>Cycles, <b>Java&nbsp;7</b></th><th>Cycles, <b>Java&nbsp;8</b></th></tr>
<tr><td class="label">LongPoint6</td>  <td class="ttext">Modulo big prime            </td><td> 1.2 </td><td> 31.2</td></tr>
<tr><td class="label">LongPoint60</td> <td class="ttext">Modulo optimised, unsigned  </td><td> 2.8  </td><td> 4.0</td></tr>
<tr><td class="label">LongPoint61</td> <td class="ttext">Modulo optimised, signed    </td><td> 5.1  </td><td> 7.0</td></tr>
</table>

Even though **Java 8** figures are bigger than those of **Java 7**, they are still surprisingly small. The processor must
be very advanced to calculate our unsigned version in 4 cycles.

The disassembly
---------------

Just out of curiosity, let's look at the code generated for our functions. Each of them is compiled twice by **Java 8**,
we'll look at the second output. This is the code for the unsigned version (`LongPoint60`):

{% highlight c-objdump %}
0x00000000027be8e0: sub    rsp,0x18
0x00000000027be8e7: mov    QWORD PTR [rsp+0x10],rbp  ;*synchronization entry
                                                ; - LongPoint60::hashCode@-1 (line 46)

0x00000000027be8ec: mov    r10,QWORD PTR [rdx+0x10]  ;*getfield v
                                              ; - LongPoint60::hashCode@1 (line 46)

0x00000000027be8f0: mov    r11,r10
0x00000000027be8f3: shr    r11,0x20           ;*lushr
                                              ; - LongUtil::uhi@3 (line 25)
                                              ; - LongPoint60::mult_unsigned_hipart@1 (line 29)
                                              ; - LongPoint60::hashCode@7 (line 46)

0x00000000027be8f7: mov    r8d,r10d           ;*land
                                              ; - LongUtil::ulo@4 (line 30)
                                              ; - LongPoint60::mult_unsigned_hipart@7 (line 30)
                                              ; - LongPoint60::hashCode@7 (line 46)

0x00000000027be8fa: imul   r9,r11,0x2449f023
0x00000000027be901: imul   rcx,r8,0x2c624b0b
0x00000000027be908: imul   r8,r8,0x2449f023   ;*lmul
                                              ; - LongPoint60::mult_unsigned_hipart@42 (line 36)
                                              ; - LongPoint60::hashCode@7 (line 46)

0x00000000027be90f: shr    rcx,0x20
0x00000000027be913: mov    rbx,r8
0x00000000027be916: shr    rbx,0x20
0x00000000027be91a: mov    r8d,r8d
0x00000000027be91d: imul   r11,r11,0x2c624b0b  ;*lmul
                                              ; - LongPoint60::mult_unsigned_hipart@35 (line 35)
                                              ; - LongPoint60::hashCode@7 (line 46)

0x00000000027be924: mov    edi,r11d
0x00000000027be927: add    rdi,r8
0x00000000027be92a: add    rdi,rcx
0x00000000027be92d: shr    r11,0x20
0x00000000027be931: add    r9,r11
0x00000000027be934: add    r9,rbx
0x00000000027be937: shr    rdi,0x20
0x00000000027be93b: add    r9,rdi
0x00000000027be93e: sar    r9,0x1b
0x00000000027be942: imul   r11,r9,0x386fa527
0x00000000027be949: sub    r10,r11
0x00000000027be94c: mov    eax,r10d           ;*l2i  ; - LongPoint60::hashCode@24 (line 47)

0x00000000027be94f: add    rsp,0x10
0x00000000027be953: pop    rbp
0x00000000027be954: test   DWORD PTR [rip+0xfffffffffd9816a6],eax        # 0x0000000000140000
                                              ;   {poll_return}
0x00000000027be95a: ret    
{% endhighlight %}
  
This is the code for the signed version (`LongPoint61`):

{% highlight c-objdump %}
0x00000000025949e0: sub    rsp,0x18
0x00000000025949e7: mov    QWORD PTR [rsp+0x10],rbp  ;*synchronization entry
                                              ; - LongPoint61::hashCode@-1 (line 46)

0x00000000025949ec: mov    rbx,QWORD PTR [rdx+0x10]  ;*getfield v
                                              ; - LongPoint61::hashCode@1 (line 46)

0x00000000025949f0: mov    r10,rbx
0x00000000025949f3: shr    r10,0x20
0x00000000025949f7: mov    rdi,rbx
0x00000000025949fa: sar    rdi,0x3f
0x00000000025949fe: mov    r11d,r10d
0x0000000002594a01: mov    r10d,ebx           ;*land
                                              ; - LongUtil::ulo@4 (line 30)
                                              ; - LongPoint61::mult_signed_hipart@8 (line 30)
                                              ; - LongPoint61::hashCode@15 (line 47)

0x0000000002594a04: movsxd r11,r11d           ;*i2l  ; - LongPoint61::mult_signed_hipart@4 (line 29)
                                              ; - LongPoint61::hashCode@15 (line 47)

0x0000000002594a07: imul   r8,r10,0x2c624b0b
0x0000000002594a0e: imul   r9,r11,0x2c624b0b  ;*lmul
                                              ; - LongPoint61::mult_signed_hipart@37 (line 35)
                                              ; - LongPoint61::hashCode@15 (line 47)

0x0000000002594a15: shr    r8,0x20
0x0000000002594a19: mov    rcx,r9
0x0000000002594a1c: shr    rcx,0x20
0x0000000002594a20: mov    edx,r9d
0x0000000002594a23: mov    r9d,ecx
0x0000000002594a26: imul   r11,r11,0x2449f023
0x0000000002594a2d: movsxd r9,r9d
0x0000000002594a30: add    r11,r9
0x0000000002594a33: imul   r10,r10,0x2449f023  ;*lmul
                                              ; - LongPoint61::mult_signed_hipart@44 (line 36)
                                              ; - LongPoint61::hashCode@15 (line 47)

0x0000000002594a3a: mov    r9d,r10d
0x0000000002594a3d: add    rdx,r9
0x0000000002594a40: add    rdx,r8
0x0000000002594a43: shr    r10,0x20
0x0000000002594a47: shr    rdx,0x20
0x0000000002594a4b: mov    r8d,r10d
0x0000000002594a4e: movsxd r10,r8d
0x0000000002594a51: add    r11,r10
0x0000000002594a54: add    r11,rdx
0x0000000002594a57: sar    r11,0x1b
0x0000000002594a5b: sub    r11,rdi
0x0000000002594a5e: imul   r10,r11,0x386fa527
0x0000000002594a65: sub    rbx,r10
0x0000000002594a68: mov    eax,ebx            ;*l2i  ; - LongPoint61::hashCode@34 (line 48)
0x0000000002594a6a: add    rsp,0x10
0x0000000002594a6e: pop    rbp
0x0000000002594a6f: test   DWORD PTR [rip+0xfffffffffdc9b58b],eax        # 0x0000000000230000
                                                ;   {poll_return}
0x0000000002594a75: ret    
{% endhighlight %}

Both of these pieces look like a very good code for me. It is a mystery why the second one runs so much slower than the
first one. It is still amazing that both run much faster than a runtime call, which performs s `DIV` instruction.

A full test
-----------

How do the optimised versions perform in the full test (a Life application)? Let's try them.
Here are the times (in milliseconds for 10,000 iterations). The times for other versions are taken from
the [original article]({{ site.ART-HASH }}):

<table class="numeric">
<tr><th>Class name</th>                <th>Comment</th>                           <th>Time, <b>Java 7</b></th><th>Time, <b>Java 8</b></th></tr>
<tr><td class="label">Point</td>       <td class="ttext">Multiply by 3, 5                  </td><td>  2743</td><td> 4961</td></tr>
<tr><td class="label">Long</td>        <td class="ttext"><code>Long</code> default         </td><td>  4273</td><td> 6755</td></tr>
<tr><td class="label">LongPoint</td>   <td class="ttext">Multiply by 3, 5                  </td><td>  2602</td><td> 4836</td></tr>
<tr><td class="label">LongPoint3</td>  <td class="ttext">Multiply by 11, 17                </td><td>  2074</td><td> 2028</td></tr>
<tr><td class="label">LongPoint4</td>  <td class="ttext">Multiply by two big primes        </td><td>  2000</td><td> 1585</td></tr>
<tr><td class="label">LongPoint5</td>  <td class="ttext">Multiply by one big prime         </td><td>  1979</td><td> 1553</td></tr>
<tr><td class="label">LongPoint6</td>  <td class="ttext">Modulo big prime                  </td><td>  1890</td><td> 1550</td></tr>
<tr><td class="label">LongPoint60</td> <td class="ttext">Modulo optimised, unsigned        </td><td>  2124</td><td> 1608</td></tr>
<tr><td class="label">LongPoint61</td> <td class="ttext">Modulo optimised, signed          </td><td>  2198</td><td> 1689</td></tr>
<tr><td class="label">LongPoint7</td>  <td class="ttext"><code>java.util.zip.CRC32</code>  </td><td> 10115</td><td> 3206</td></tr>
</table>


The new versions do not perform badly, compared to other hash functions, but they are not nearly as fast as the microbenchmarks suggest.
In particular, they don't outperform the original division-based version. Why this happend is a mystery, perhaps **Java 8** is capable of some magic
we don't know about yet.

Conclusions
-----------

- A division operation is very slow compared to other arithmetic; it should be avoided in performance-critical code
unless there is evidence that **Java** VM can optimise it.

- On the contrary, a multiplication operation is relatively fast; it still makes sense to replace multiplication by
small constants with shifts, additions or `LEA` instructions, but compilers are usually good at that.

- Replacing division by a constant with a multiplication by a reciprocal is a very useful optimisation; it's a pity it
has been removed from **Java** VM in version 8.

- This optimisation requires a 128-bit multiplication, which isn't available in **Java** directly. It can be implemented
from scratch, although in some ugly way.

- The code we produced was very long. It contained five multiplications and several shifts and additions; still, it
was faster than the original code using division (on **Java 8**) -- although, only in a microbenchmark test.

- The optimisation didn't increase the speed of the full test, or, simply speaking, failed.

Coming soon
-----------

Is there any way to improve division even more? Can we outperform the original division-based version on the full test this way,
and if not, why? We'll see soon.
