---
layout: post
title:  "More optimisations of division-based hash: Karatsuba's method"
date:   2015-10-20 12:00:00
tags: Java optimisation hashmap life
story: life
story-title: "Conway's Game of Life"
---

In ["{{ site.TITLE-DIVISION-HASH }}"]({{ site.ART-DIVISION-HASH }}) we optimised a hash function working on `long` numbers and based on calculating a remainder.
The idea was to replace a division operation with a multiplication by a reciprocal. In pseudo-code this replacement looked like this. The original hash function:

{% highlight java %}
    public int hashCode ()
    {
        return (int) (v % 946840871);
    }
{% endhighlight %}

The optimised hash function (for simplicity, we'll limit today's consideration to an unsigned version):

{% highlight java %}
    public int hashCode ()
    {
        long div = (v ** 2614885092524444427L) >> 91;
        return (int) (v - div * 946840871);
    }
{% endhighlight %}

This is a pseudo-code and not regular **Java**, because **Java** does not have operation `**`, which multiplies two 64-bit numbers and produces a 128-bit result.
We've emulated this operation using the following code:

{% highlight java %}
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

    public int hashCode ()
    {
        long div = mult_unsigned_hipart (v, 2614885092524444427L) >>> 27;
        return (int) (v - div * 946840871);
    }
{% endhighlight %}

This implementation  uses five multiplications: four for a 64-bit multiplication, and one for computing a remainder. It also uses several additions.
This is what is added, as a picture:

<table>
 <tr><th colspan="2"> high 64 bits of xy </th> <th colspan="2"> low 64 bits of xy </th> </tr>
 <tr><td> hi (A*C)         </td><td> lo (A*C)</td> </tr>
 <tr><td class="noborder"> </td><td> hi (A*D)</td>         <td> lo (A*D) </td> </tr>
 <tr><td class="noborder"> </td><td> hi (B*C)</td>         <td> lo (B*C) </td> </tr>
 <tr><td class="noborder"> </td><td class="noborder"> </td><td> hi (B*D) </td> <td> lo (B*D)</td> </tr>
</table>

where

 <div class="formula">
  x = 2<sup>32</sup>A + B<br>
  y = 2<sup>32</sup>C + D
 </div>

Since the result of a 64-bit multiplication is shifted right by 91, we only need the high part of a product.
A rather complex way to add all these values together is caused by a possibility that adding `A*D`, `B*C` and `hi(B*D)` may
produce a 65-bit (or even a 66-bit) result.

Since publishing this code I came across two more improvements, which I'll show now.

The first idea: Specialisation
------------------------------

The `mult_unsigned_hipart` above is designed for arbitrary `x` and `y` values. We are, however, working with a special case,
where we know the value of `y`: it is 2614885092524444427. Then  we have

 <div class="formula">
   C = 608825379 = 0x2449F023 < 2<sup>30</sup><br/>
   D = 744639243 = 0x2C624B0B < 2<sup>30</sup>
 </div>

In other words, both C and D are 30-bit numbers.

A 30-bit number multiplied by a 32-bit number produces a 62-bit number, which means

 <div class="formula">
   AD < 2<sup>62</sup><br/>
   BC < 2<sup>62</sup><br/>
   BD < 2<sup>62</sup><br/>
   hi(BD) < 2<sup>30</sup></sup><br/>
   AD + BC + hi(BD) < 2<sup>62</sup> + 2<sup>62</sup> + 2<sup>30</sup> < 2<sup>63</sup> + 2<sup>63</sup> = 2<sup>64</sup>
 </div>

In short, the sum of two 62-bit numbers and a high part of a 62-bit number is at most a 64-bit number, so there is no carry bits
to worry about. As a result, the multiplication routine can be simplified like this:

{% highlight java %}
    static long mult_unsigned_hipart_special (long x, long y)
    {
        long A = uhi (x);
        long B = ulo (x);
        long C = uhi (y);
        long D = ulo (y);
                   
        return A*C + uhi (A*D + B*C + uhi (B*D));
    }
{% endhighlight %}

Note that we can't ignore the term `uhi (B*D)`. It may cause carry when added to the lower part of `A*D + B*C`. Let's assume

 <div class="formula">
  x = 4294967295 = 0xFFFFFFFF<br/>
 </div>

Then

 <div class="formula">
  A = 0<br/>
  B = 4294967295 = 0xFFFFFFFF<br/>
  ulo (AD + BC) = 0xDBB60FDD<br/>
  uhi (BD) = 0x2C624B0A<br/>
  AC + uhi (AD + BC + uhi (BD)) = 608825379<br/>
  AC + uhi (AD + BC) = 608825378<br/>
 </div>

Right, but this is a one-bit difference, and in our particular use case the result of multiplication is shifted right by 27 bits.
Is it possible that one carry bit from adding BD portion gets propagated far enough to the left to
affect the result of this shift? It seems highly unlikely, but the study shows that it can happen.

Let's fix the same value of B as above (0xFFFFFFFF) and look for A that makes the following true:

 <div class="formula">
  (ulo (AC) + uhi (AD + BC)) mod 2<sup>27</sup> = 2<sup>27</sup>&minus;1<br/>
  ulo (AD + BC) + uhi (BD) > 2<sup>32</sup>
 </div>

Finding a suitable value of A would have required some sophisticated mathematics just twenty years ago. Thanks to the Moore's law, these days we can run a simple
brute force search --  we can test all four billion candidates for A in seconds.
The search finds five suitable values of A, the first of which is A = 755792740.

 <div class="formula">
  AC + uhi (AD + BC + uhi (BD)) = 0x662C42B48000000<br/>
  AC + uhi (AD + BC) = 0x662C42B47FFFFFF<br/>
  (AC + uhi (AD + BC + uhi (BD))) >>> 27 = 3428353385<br/>
  (AC + uhi (AD + BC)) >>> 27  = 3428353384
 </div>

The value `w(A,B)` = 3246105105149198335 produces 0 as a result of `hashCode()` listed above and 946840871 if `B*D` is removed from `mult_unsigned_hipart_special ()`.
Our attempt to perform a 64-bit multiplication using only three 32-bit multiplications has failed.

The second idea: Karatsuba's multiplication
-------------------------------------------

This doesn't mean that such calculation is impossible. The trick, called [Karatsuba's multiplication](https://en.wikipedia.org/wiki/Karatsuba_algorithm),
after [Anatoly Alexeevich Karatsuba](https://ru.wikipedia.org/wiki/%D0%9A%D0%B0%D1%80%D0%B0%D1%86%D1%83%D0%B1%D0%B0,_%D0%90%D0%BD%D0%B0%D1%82%D0%BE%D0%BB%D0%B8%D0%B9_%D0%90%D0%BB%D0%B5%D0%BA%D1%81%D0%B5%D0%B5%D0%B2%D0%B8%D1%87), who discovered it in 1960 and published in 1962, and is based on a simple observation:

 <div class="formula">
  (A + B)(C + D) = AC + BC + AD + BD
 </div>

which implies

 <div class="formula">
  AD + BC = (A + B) (C + D) &minus; (AC + BD)
 </div>

Then we can modify the original formula as

 <div class="formula">
  (2<sup>32</sup>A + B)(2<sup>32</sup>C + D) = 2<sup>64</sup>AC + 2<sup>32</sup>(AD + BC) + BD = <br/>
  2<sup>64</sup>AC + 2<sup>32</sup>((A + B) (C + D) &minus; (AC + BD)) + BD
 </div>

Interestingly, Knuth (_The art of computer programming, vol. 2, 4.3.3.A_) suggests different method, based on a subtraction rather than addition:

 <div class="formula">
  (2<sup>32</sup>A + B)(2<sup>32</sup>C + D) = <br/>
  2<sup>64</sup>AC + 2<sup>32</sup>(AC + BD &minus; (A &minus; B) (C &minus; D)) + BD = <br/>
  (2<sup>64</sup> + 2<sup>32</sup>)AC + 2<sup>32</sup>(A &minus; B) (D &minus; C) + (2<sup>32</sup> + 1) BD
 </div>

He mentions Karatsuba's method too, with a remark that his method is "similar, but more complex". It seems other way around, since Knuth's method requires signed arithmetic,
while Karatsuba's can be fully done in unsigned numbers (the single subtraction that it contains is guaranteed to produce non-negative result).

Both formulae contain only three multiplications. They were not designed just to multiply 64-bit integers -- the primary application is multiplication of very
long numbers. The methods reduce multiplication of two 2N-bit numbers to three N-bit multiplications (more precisely, two N-bit multiplications and a (N+1)-bit one)
and some additions. The additions are performed in O&nbsp;(N) time and do not
therefore affect the overall complexity of algorithms, which is O&nbsp;(N<sup>log<sub>2</sub>3</sup>). The example of recursive Karatsuba's multiplication can be seen in
the version of `java.math.BigInteger` that comes with **Java 8**. Look at `multiplyKaratsuba()` function, which is used for numbers between 80 and 240 `int`s long
(that is, between 2560 and 7680 bits). The class uses [Toom-Cook algorithm](https://en.wikipedia.org/wiki/Toom%E2%80%93Cook_multiplication) for longer numbers and a classic
square-complexity grade school algorithm for shorter numbers. The `BigInteger` in **Java 7** always uses the grade school algorithm and works much slower when the inputs are long.

The original Karatsuba's formula looks very attractive, but one obstacle makes it difficult to use it for general 64-bit multiplication. Both A + B and C + D are 33-bit numbers,
and their product can potentially occupy 66 bits (in Knuth's version A &minus; B and D &minus; C are 33-bit signed numbers, with the similar implication for their product). We can emulate a 33-bit multiplication
but that will require additional operations and, probably, branches.

However, the specialisation trick that we employed in the previous section is applicable again. We know that C and D are 30-bit numbers. Their sum

 <div class="formula">
  C + D = 0x2449F023 + 0x2C624B0B = 0x50AC3B2E
 </div>

is a 31-bit number. And, as we know, A + B is a 33-bit number.

This means that (A + B) (C + D) is at most a 64-bit number, and the multiplication can be performed in standard **Java**'s `long`s. Similarly,
AD + BC is at most a 63-bit number, and it won't cause an overflow when added to the high part of BD. Here is the code:

{% highlight java %}
    static long mult_unsigned_hipart_special (long x, long y)
    {
        long A = uhi (x);
        long B = ulo (x);
        long C = uhi (y);
        long D = ulo (y);
        
        long AC = A * C;
        long BD = B * D;
        long AD_BC = (A+B) * (C+D) - (AC + BD);
        return AC + uhi (AD_BC + uhi (BD));
    }
{% endhighlight %}

Giving it a try
---------------

Let's add two more classes ([`LongPoint62`]({{ site.REPO-LIFE }}/blob/c7ace1da74dbe139b5aa0abc952b8d69e1d0be92/LongPoint62.java) and 
[`LongPoint63`]({{ site.REPO-LIFE }}/blob/c7ace1da74dbe139b5aa0abc952b8d69e1d0be92/LongPoint63.java)) to our tests
(the entire code is [here]({{ site.REPO-LIFE }}/tree/c7ace1da74dbe139b5aa0abc952b8d69e1d0be92)). Let's run a micro-benchmark (`HashTime`):

<table class="numeric">
<tr><th>Class name</th>                <th>Comment</th>                               <th>Time, <b>Java 7</b></th><th>Time, <b>Java 8</b></th></tr>
<tr><td class="label">LongPoint6</td>  <td class="ttext">Original version             </td><td>  578</td><td> 1655</td></tr>
<tr><td class="label">LongPoint60</td> <td class="ttext">Optimised, unsigned          </td><td>  640</td><td>  610</td></tr>
<tr><td class="label">LongPoint61</td> <td class="ttext">Optimised, signed            </td><td>  729</td><td>  727</td></tr>
<tr><td class="label">LongPoint62</td> <td class="ttext">Optimised and specialised    </td><td>  595</td><td>  547</td></tr>
<tr><td class="label">LongPoint63</td> <td class="ttext">Karatsuba's multiplication   </td><td>  622</td><td>  565</td></tr>
<tr><td class="label">NullPoint</td>   <td class="ttext">Null function (does nothing) </td><td>  532</td><td>  455</td></tr>
</table>


Both new versions are faster than the original optimised one; `LongPoint62` is approaching the speed of the original version on **Java 7**. However, we've got a surprise:
Karatsuba's version is slower than the first specialised one. For anyone who was brought up with the idea that multiplication is an expensive operation to be avoided
at all costs, this is nonsense. Let's look at the assembly outputs. This is the disassembly of `LongPoint62`:


{% highlight c-objdump %}
  0x00007f101930c610: mov    r11,r10
  0x00007f101930c613: shr    r11,0x20
  0x00007f101930c617: mov    r8d,r10d
  0x00007f101930c61a: imul   r9,r11,0x2449f023    ; C = 608825379
  0x00007f101930c621: imul   rcx,r8,0x2c624b0b    ; D = 744639243
  0x00007f101930c628: imul   r8,r8,0x2449f023     ; C = 608825379
  0x00007f101930c62f: shr    rcx,0x20
  0x00007f101930c633: imul   r11,r11,0x2c624b0b   ; D = 744639243
  0x00007f101930c63a: add    r11,r8
  0x00007f101930c63d: add    r11,rcx
  0x00007f101930c640: shr    r11,0x20
  0x00007f101930c644: add    r9,r11
  0x00007f101930c647: sar    r9,0x1b
  0x00007f101930c64b: imul   r11,r9,0x386fa527    ; 946840871
  0x00007f101930c652: sub    r10,r11
  0x00007f101930c655: mov    eax,r10d
{% endhighlight %}
 
And this is the code for `LongPoint63`:

{% highlight c-objdump %}
  0x00007fac89309750: mov    r11,r10
  0x00007fac89309753: shr    r11,0x20
  0x00007fac89309757: mov    r8d,r10d
  0x00007fac8930975a: mov    r9,r11
  0x00007fac8930975d: add    r9,r8
  0x00007fac89309760: imul   r11,r11,0x2449f023   ; C = 608825379
  0x00007fac89309767: imul   r9,r9,0x50ac3b2e     ; C + D
  0x00007fac8930976e: imul   r8,r8,0x2c624b0b     ; D = 744639243
  0x00007fac89309775: mov    rcx,r11
  0x00007fac89309778: add    rcx,r8
  0x00007fac8930977b: sub    r9,rcx
  0x00007fac8930977e: shr    r8,0x20
  0x00007fac89309782: add    r9,r8
  0x00007fac89309785: shr    r9,0x20
  0x00007fac89309789: add    r9,r11
  0x00007fac8930978c: shr    r9,0x1b
  0x00007fac89309790: imul   r11,r9,0x386fa527    ; 946840871
  0x00007fac89309797: sub    r10,r11
  0x00007fac8930979a: mov    eax,r10d
{% endhighlight %}

The first code has more multiplications (five instead of four), but less instructions in total (16 vs 19). Three ordinary instructions can't compensate for one multiplication,
if you look at the total clock count alone. However, according to [Agner Fog](http://agner.org/optimize/optimizing_assembly.pdf), one must look at two parameters:
instruction throughput and latency. While multiplication has high latency (3 cycles on Sandy Bridge), it also has high throughput (one cycle), which means that the processor
can start (and finish) another multiplication every cycle, provided that the arguments are ready.
That's the case in both snippets above -- the multiplications do not depend on each other. As a result, the three `imul` instructions in the first
case can be executed in 5 cycles, while the four in the second one take 6. This difference will be totally neutralised by the extra three instructions in the second case,
especially taking into account that the shifts and additionss that follow the multiplications form a perfect dependency chain.

The full test
-------------

Now we are ready to run the full test, our best hopes now placed on `LongPoint62`. Here are the results for our new classes, together with previous division-based ones:

<table class="numeric">
<tr><th>Class name</th>                <th>Comment</th>                           <th>Time, <b>Java 7</b></th><th>Time, <b>Java 8</b></th></tr>
<tr><td class="label">LongPoint6</td>  <td class="ttext">Original version           </td><td>  2032</td><td> 1650</td></tr>
<tr><td class="label">LongPoint60</td> <td class="ttext">Optimised, unsigned        </td><td>  2247</td><td> 1877</td></tr>
<tr><td class="label">LongPoint61</td> <td class="ttext">Optimised, signed          </td><td>  2368</td><td> 1885</td></tr>
<tr><td class="label">LongPoint62</td> <td class="ttext">Optimised and specialised  </td><td>  2191</td><td> 1726</td></tr>
<tr><td class="label">LongPoint63</td> <td class="ttext">Karatsuba's multiplication </td><td>  2239</td><td> 1776</td></tr>
</table>

As expected, the new versions are both faster than the first optimised unsigned version, Karatsuba's one being the slower one of the two. However, the original division-based
one (`LongPoint6`) is still the fastest on both  **Java 7** and **Java 8**. Our optimisation attempt has failed again.

Conclusions
-----------

- It is possible to perform a double-precision multiplication using three single-precision ones

- However, this technique, being useful for very long numbers, does not help improve performance on very short ones

- In general, unlike division, multiplication operation isn't as expensive as it used to be, and can be used without much hesitation. The overheads of avoiding it may
exceed its own cost

- Specialised functions (those that are designed to handle only a subset of possible inputs) can be made faster than the generic ones. Obviously, there is a price:
the entire solution becomes more error-prone and difficult to maintain. This is why this approach must be used with care and only when absolutely necessary. For instance,
in our case reducing execution time from 2247 ms to 2191 (by 2.5%) hardly justifies the costs.

- The original solution (using normal division operation) performed very badly as a micro-benchmark on **Java 8**, and could be improved a lot. However,
all attempts to optimise it in a full test failed miserably. This is a mystery.

Coming soon
-----------

I'm going to address this mystery in my next article.
