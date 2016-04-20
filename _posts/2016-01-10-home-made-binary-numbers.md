---
layout: post
title:  "Home-made binary numbers"
date:   2016-01-10 12:00:00
tags: Java hashmap Peano
story: peano
story-title: "Numbers from scratch"
---

In the previous article (["{{ site.TITLE-PEANO }}"]({{ site.ART-PEANO }})) we were dealing with a dramatic situation: the hackers from the Anti-Numeric League
damaged our **Java** compiler and made it impossible to use numbers. We needed numbers urgently for a student's appointment, that's why we had to implement
arithmetic from scratch. We tried to implement the  Peano arithmetic directly, using sets as numbers, but that was way too slow (although, surprisingly,
worked when the numbers involved were small, and even produced some results). A sequence of improvements
brought us to the final implementation, where all the numbers produced during program execution were collected into one doubly-linked list, in their natural order.

The most surprising part was that it worked and even was practically usable. We tested it on two problems:
printing all the [Pythagorean triples](https://en.wikipedia.org/wiki/Pythagorean_triple)
and finding all the [perfect numbers](https://en.wikipedia.org/wiki/Perfect_number),
both within the given range. The speed achieved was about 1000 times less than on a native **Java** code, and that speed was achieved only after some
manual optimisation of the user code that imitated the work of the compiler.

Here are the achieved times, in seconds, for **Pythagorean**, for several upper boundaries:

<table class="numeric">
<tr><th>Version</th><th>100</th><th>200</th><th>500</th><th>1,000</th></tr>
<tr><td>Unoptimised </td><td> 7.2  </td>     <td>203 </td>                <td>466 in 14000s </td><td></td></tr>
<tr><td>Optimised   </td><td> 0.070</td>     <td>0.73</td>                <td>22.3</td>          <td>  360</td></tr>
<tr><td>Native      </td><td> 0.002</td>     <td>0.021</td>               <td>0.093</td>         <td> 0.571</td></tr>
</table>

And these are the times for `Perfect`:

<table class="numeric">
<tr><th>Version</th><th>100</th><th>1000</th><th>5,000</th><th>10,000</th></tr>
<tr><td>Unoptimised  </td> <td>0.016</td><td>3.2  </td><td>460</td><td>4154 </td></tr>
<tr><td>Optimised</td>     <td>0.011</td><td>0.48 </td><td> 41</td><td>336  </td></tr>
<tr><td>Native</td>        <td>0.001</td><td>0.006</td><td>0.074</td><td>0.280</td></tr>
</table>

Another surprising fact was that, while I considered Peano arithmetic a joke appropriate for the festive season, others take it seriously:
[https://wiki.haskell.org/Peano_numbers](https://wiki.haskell.org/Peano_numbers),
[https://hackage.haskell.org/package/peano-inf-0.6.5/peano-inf-0.6.5.tar.gz](https://hackage.haskell.org/package/peano-inf-0.6.5/peano-inf-0.6.5.tar.gz).

But anyway, the code is way too slow. Now we are going to try another approach. We'll store numbers in the binary format -- as sequences of zeroes and ones.
Let's see if this is faster.

We'll be using the last column of the above tables as a benchmark: 1,000 for `pythagorean` and 10,000 for `perfect`.

The new code will be available [in this repository]({{ site.REPO-PEANO }}/tree/bc99747a4b6df0fe565205d309ecc23035dc6867).

The implementation details
--------------------------

The numbers will be represented as linked lists, where elements contain digits. We'll have to make several technical decisions:

1) The representation of digits. We could use Booleans, but the purists may argue that Booleans are also numbers, especially when we start performing operations on them,
such as "exclusive OR". That's why we'll avoid them for now (perhaps, later try an implementation used on Booleans as well), and rather use the enums:

{% highlight Java %}
enum Digit
{
    ZERO,
    ONE;
}
{% endhighlight %}

2) Singly or doubly linked list? It seems that for our current purpose a singly linked list is sufficient:

{% highlight Java %}
public class Binary
{
    private final Digit lowest;
    private final Binary next;

    private Binary (Digit d, Binary next)
    {
        this.lowest = d;
        this.next = next;
    }

    private Binary (Digit d)
    {
        this.lowest = d;
        this.next = null;
    }
}
{% endhighlight %}

Still, if the implementation shows any advantage in having a doubly-linked list, we may consider using it.

3) Which direction do they grow? Since operations must be synchronised (the digits in the same positions must be processed together),
the list must grow from the lowest to the highest, the first element being the lowest digit of the number.

4) Two classes or one? We can make a separate number object, which encapsulates the list, or we can make the list element itself a number. The potential advantage of the
first approach is that we can make a null list the default representation of zero, which will make the code more regular. The disadvantage is in the extra object.
We'll use the second approach, and will represent zero as a list consisting of just one element (`ZERO`).

5) Leading zeroes conventions. It takes some effort to get rid of leading zeroes (the subtraction code becomes more complex), but it helps elsewhere (for example, in comparisons).
Our numbers will always have the highest digit `ONE`, with the single exception of the number zero. The test for zero looks like this:

{% highlight Java %}
public boolean isZero ()
{
    return lowest == Digit.ZERO && next == null;
}
{% endhighlight %}

6) Mutable or immutable? Sometimes it looks attractive to make the class mutable. For instance, a value 10 (the four-element list consisting of `0`-`1`-`0`-`1`)
could increment itself by just replacing the lowest bit. However, we can do almost as well if we allocate a new element with the digit "1" and point it to the second
element of number 10:

<img src="{{ site.url }}/images/binary-number10.png" width="679" height="112">

This way we can use both numbers. This requires immutable design, because the number that is referenced more than once mustn't modify itself. This promises to save a lot of
space, that's why we'll make our numbers immutable and declare all the fields as `final`.

The implementation
------------------

First, we define the smallest numbers:

{% highlight Java %}
public static Binary ZERO = new Binary (Digit.ZERO);
public static Binary ONE = new Binary (Digit.ONE);
public static Binary TWO = new Binary (Digit.ZERO, ONE);
{% endhighlight %}

Then we need the operation to increment a number:

{% highlight Java %}
public Binary S ()
{
    if (lowest == Digit.ZERO) return new Binary (Digit.ONE, next);
    if (next == null) return TWO;
    return new Binary (Digit.ZERO, next.S ());
}
{% endhighlight %}

This code shows the mentioned idea of re-using parts of numbers. When we call `S()` on 10 (binary 1010), it will create the element with the digit `ONE` and link it to the
high part of 10 (binary 101)

Subtracting one is more complex, because we must take care of leading zeroes:

{% highlight Java %}
private Binary D0 ()
{
    if (lowest == Digit.ONE)
        return next == null ? null : new Binary (Digit.ZERO, next);
    if (next == null)
        throw new IllegalArgumentException ("Negative number");
    return new Binary (Digit.ONE, next.D0());
}

public Binary D ()
{
    Binary r = D0 ();
    return r == null ? ZERO : r;
}
{% endhighlight %}

We are temporarily representing zeroes as nulls, until we reach the top level of recursion. We'll use the same approach in subtraction.

Now we need a routine to add two numbers. We must iterate the lists, each time adding three bits: two from the source numbers, and a carry. This addition produces two
bits -- the new result bit and the new carry. The code to add three bits is ugly and difficult to read, that's why it helps to split the routine into two cases: adding
with carry and adding without:

{% highlight Java %}
private static Binary add_carry (Binary x, Binary y)
{
    if (x == null) {
        if (y == null) return ONE;
        return y.S ();
    }
    if (y == null) return x.S ();

    Digit d = x.lowest == y.lowest ? Digit.ONE : Digit.ZERO;
    Binary s = (x.lowest == Digit.ZERO && y.lowest == Digit.ZERO) ?
        add_no_carry (x.next, y.next) :
        add_carry (x.next, y.next);
    return new Binary (d, s);
}

private static Binary add_no_carry (Binary x, Binary y)
{
    if (x == null) return y;
    if (y == null) return x;

    Digit d = x.lowest == y.lowest ? Digit.ZERO : Digit.ONE;
    Binary s = (x.lowest == Digit.ONE && y.lowest == Digit.ONE) ?
        add_carry (x.next, y.next) :
        add_no_carry (x.next, y.next);
    return new Binary (d, s);
}

public Binary plus (Binary other)
{
    return add_no_carry (this, other);
}
{% endhighlight %}

The same approach works for subtraction, except we must take care of leading zeroes:

{% highlight Java %}
private static Binary sub_borrow (Binary x, Binary y)
{
    if (x == null)
        throw new IllegalArgumentException ("Negative number");
    if (y == null) return x.D0 ();

    Digit d = x.lowest == y.lowest ? Digit.ONE : Digit.ZERO;
    Binary s = (x.lowest == Digit.ONE && y.lowest == Digit.ZERO) ?
        sub_no_borrow (x.next, y.next) :
        sub_borrow (x.next, y.next);
    return (s == null && d == Digit.ZERO) ? null : new Binary (d, s);
}

private static Binary sub_no_borrow (Binary x, Binary y)
{
    if (y == null) return x;
    if (x == null) throw new IllegalArgumentException ("Negative number");

    Digit d = x.lowest == y.lowest ? Digit.ZERO : Digit.ONE;
    Binary s = (x.lowest == Digit.ZERO && y.lowest == Digit.ONE) ?
        sub_borrow (x.next, y.next) :
        sub_no_borrow (x.next, y.next);
    return (s == null && d == Digit.ZERO) ? null : new Binary (d, s);
}

public Binary minus (Binary other)
{
    Binary result = sub_no_borrow (this, other);
    return result == null ? ZERO : result;
}
{% endhighlight %}

Now we need comparisons:

{% highlight Java %}
enum CompareResult
{
    LT,
    EQU,
    GT
}

private static CompareResult compare (Binary x, Binary y)
{
    if (x == y)    return CompareResult.EQU;
    if (x == null) return CompareResult.LT;
    if (y == null) return CompareResult.GT;
    CompareResult r = compare (x.next, y.next);
    if (r != CompareResult.EQU) {
        return r;
    }
    return x.lowest == y.lowest ? CompareResult.EQU :
           x.lowest == Digit.ZERO ? CompareResult.LT : CompareResult.GT;
}

public boolean eq (Binary other)
{
    return compare (this, other) == CompareResult.EQU;
}

@Override
public boolean equals (Object other)
{
    return eq ((Binary) other);
}

public boolean neq (Binary other)
{
    return ! eq (other);
}

public boolean lt (Binary other)
{
    return compare (this, other) == CompareResult.LT;
}

public boolean leq (Binary other)
{
    return compare (this, other) != CompareResult.GT;
}

public boolean geq (Binary other)
{
    return compare (this, other) != CompareResult.LT;
}

public boolean gt (Binary other)
{
    return compare (this, other) == CompareResult.GT;
}
{% endhighlight %}

Unfortunately, comparisons are not very cheap: we must traverse both numbers to find out which one is bigger. Even if one is longer, we don't know that until we
traverse the lists (we can't keep the list sizes, for this requires numbers).

Now we come to multiplication and division. We'll perform both using classic school textbook approach. First, we need shifts:

{% highlight Java %}
public Binary shift_left ()
{
    return isZero() ? this : new Binary (Digit.ZERO, this);
}

public Binary shift_right ()
{
    return next == null ? ZERO : next;
}
{% endhighlight %}

And now we are ready for multiplication:

{% highlight Java %}
public Binary mult (Binary other)
{
    Binary result = ZERO;
    Binary x = this;
    if (! x.isZero ()) {
        for (Binary y = other; y != null; y = y.next) {
            if (y.lowest == Digit.ONE) {
                result = result.plus (x);
            }
            x = x.shift_left ();
        }
    }
    return result;
}

public Binary square ()
{
    return mult (this);
}
{% endhighlight %}

and for division:

{% highlight Java %}
public static class DivResult
{
    final Binary quotient;
    final Binary remainder;
    
    public DivResult (Binary quotient, Binary remainder)
    {
        this.quotient = quotient;
        this.remainder = remainder;
    }
}

public DivResult divrem (Binary other)
{
    Binary a = this;
    Binary b = other;
    Binary digit = ONE;
    while (a.gt (b)) {
        b = b.shift_left ();
        digit = digit.shift_left ();
    }
    Binary result = ZERO;
    while (true) {
        if (a.lt (other)) {
            return new DivResult (result, a);
        }
        if (a.geq (b)) {
            a = a.minus (b);
            result = result.plus (digit);
        }
        b = b.shift_right ();
        digit = digit.shift_right ();
    }
}

public Binary div (Binary other)
{
    return divrem (other).quotient;
}

public Binary rem (Binary other)
{
    return divrem (other).remainder;
}
{% endhighlight %}

The conversion to strings and back is the same as in the original number implementation (["{{ site.TITLE-PEANO }}"]({{ site.ART-PEANO }})). The test routines are also
almost identical:

{% highlight Java %}
public static void pythagorean (Binary N)
{
    Debug.Timer.start ();
    for (Binary c = ONE; c.leq (N); c = c.S ())
        for (Binary b = ONE; b.lt (c); b = b.S ())
            for (Binary a = ONE; a.lt (b); a = a.S ())
                if (a.square().plus (b.square()).equals (c.square())) {
                    System.out.println (a + " " + b + " " + c);
                    Debug.Timer.stop ();
                    Debug.Timer.dump ();
                }
}

public static void perfect (Binary N)
{
    Debug.Timer.start ();
    for (Binary i = ONE; i.leq (N); i = i.S ()) {
        Binary sum = ZERO;
        for (Binary j = ONE; j.lt (i); j = j.S ()) {
            if (i.rem (j).isZero ())
                sum = sum.plus (j);
        }
        if (i.equals (sum)) {
            System.out.println (i);
        }
    }
    System.out.println ("Total: ");
    Debug.Timer.stop ();
    Debug.Timer.dump ();
}

public static void main (String [] args)
{
    pythagorean (Binary.parseInt ("1000"));
    perfect (Binary.parseInt ("10000"));
}
{% endhighlight %}

The first test
--------------

Now we are ready to run the first test. We are using the original (unoptimised) versions of the user code (by which we mean `pythagorean` and `perfect`), and also
we haven't yet applied any optimisations to our algorithms. Here are the times:

<table class="numeric">
<tr><th>Version</th><th>Comment</th><th>Pythagorean, time for 1,000, sec</th><th>Perfect, time for 10,000, sec</th></tr>
<tr><td class="label">Peano6</td><td class="ttext">Optimised Peano implementation</td> <td>360</td><td>336</td></tr>
<tr><td class="label">Binary</td><td class="ttext">Original binary implementation</td> <td>275</td><td> 25</td></tr>
</table>

The results look good, especially for `perfect`. We have already outperformed the best `Peano` version.

The equality comparison
-----------------------

Last time we got some (albeit small) improvement by replacing "less than" comparison in the loops with "not equals". The reason it helped is that equality comparison
was much faster (direct reference comparison instead of a list traversal). In our case it is not so: both call `compare()`. However, numbers can be compared for equality
faster: any bit difference means "not equal", so there is no need to check higher bits if the lower ones differ. Let's try that (the new class is `Binary2` and the new
methods are `pythagorean2` and `perfect2`):

{% highlight Java %}
public boolean eq (Binary2 other)
{
    if (this == other) return true;
    if (lowest != other.lowest) return false;
    if (next == null) return other.next == null;
    if (other.next == null) return false;
    return next.eq (other.next);
}
    
@Override
public boolean equals (Object other)
{
    return eq ((Binary2) other);
}
{% endhighlight %}

We've got a small improvement again:

<table class="numeric">
<tr><th>Version</th><th>Comment</th><th>Pythagorean, time for 1,000, sec</th><th>Perfect, time for 10,000, sec</th></tr>
<tr><td class="label">Binary</td> <td class="ttext">Original binary implementation</td> <td>275</td><td> 25</td></tr>
<tr><td class="label">Binary2</td><td class="ttext">Equality test improved</td> <td>263</td><td> 24</td></tr>
</table>

Improvement of multiplication and division
------------------------------------------

Let's return to the code of multiplication:

{% highlight Java %}
public Binary mult (Binary other)
{
    Binary result = ZERO;
    Binary x = this;
    if (! x.isZero ()) {
        for (Binary y = other; y != null; y = y.next) {
            if (y.lowest == Digit.ONE) {
                result = result.plus (x);
            }
            x = x.shift_left ();
        }
    }
    return result;
}
{% endhighlight %}

Tis code has a problem: it shifts the first argument left as it proceeds with multiplication, and then spends time adding these additional zeroes to the result.
It is better to add the unshifted argument and shift the result, but this requires performing multiplication starting from high bits, which requires recursion:

{% highlight Java %}
public Binary3 mult (Binary3 other)
{
    if (other.isZero ()) return ZERO;
    Binary3 r = mult (other.shift_right ()).shift_left ();
    if (other.lowest == Digit.ONE) r = r.plus (this);
    return r;
}
{% endhighlight %}

Division suffers from the same problem:

{% highlight Java %}
public DivResult divrem (Binary other)
{
    Binary a = this;
    Binary b = other;
    Binary digit = ONE;
    while (a.gt (b)) {
        b = b.shift_left ();
        digit = digit.shift_left ();
    }
    Binary result = ZERO;
    while (true) {
        if (a.lt (other)) {
            return new DivResult (result, a);
        }
        if (a.geq (b)) {
            a = a.minus (b);
            result = result.plus (digit);
        }
        b = b.shift_right ();
        digit = digit.shift_right ();
    }
}
{% endhighlight %}

We shift the divisor left until it becomes as long as the dividend, and then subtract it from the dividend, together with all the trailing zeroes.
We could instead shift the dividend right, and subtract short numbers. Here is the new code:

{% highlight Java %}
public DivResult divrem (Binary3 other)
{
    if (this.lt (other)) {
        return new DivResult (ZERO, this);
    }
    DivResult dr = this.shift_right ().divrem (other);
    Binary3 r = dr.remainder.shift_left ();
    Binary3 q = dr.quotient.shift_left ();
    if (lowest == Digit.ONE) {
        r = r.S ();
    }
    if (r.geq (other)) {
        r = r.minus (other);
        q = q.S ();
    }
    return new DivResult (q, r);
}
{% endhighlight %}

Here are the results:

<table class="numeric">
<tr><th>Version</th><th>Comment</th><th>Pythagorean, time for 1,000, sec</th><th>Perfect, time for 10,000, sec</th></tr>
<tr><td class="label">Binary2</td><td class="ttext">Equality test improved</td> <td>263</td><td> 24</td></tr>
<tr><td class="label">Binary3</td><td class="ttext">Multiplication and division improved</td> <td>221</td><td> 18</td></tr>
</table>

We've got a really remarkable result: the recursive solutions work faster than iterative ones. Sure, the improvement didn't come just because we replaced iteration with recursion.
In both cases it was due to different algorithm, but both new algorithms (multiplication and division) are very difficult to write down without recursion.


Keeping the links
-----------------

Until now, all the number objects were independent from each other. They were just linked lists of bits, created fresh when they were needed. Sometimes a portion of the list
could be re-used as a part of another number. The fundamental numbers `ZERO`, `ONE` and `TWO` are often
re-used. However, unused numbers are not kept, which saves memory, but costs time -- we have to re-create the numbers that we had before.
It is attractive to keep and re-use all the numbers, although this approach isn't applicable to all computational programs. Some of them use billions of different numbers.
In our test programs, however, the number of numbers used is at most millions, so we can use this approach.

What we'll do is keep the numbers resulting from appending `ZERO` or `ONE` to a given number, as fields in the object, populated on demand. We'll replace all calls to constructors
with calls to a special method:

{% highlight Java %}
private Binary4 link0 = null;
private Binary4 link1 = null;

private static Binary4 newBinary (Digit d, Binary4 next)
{
    if (next != null) {
        if (d == Digit.ONE) {
            if (next.link1 == null) {
                next.link1 = new Binary4 (d, next);
            }
            return next.link1;
        } else {
            if (next.link0 == null) {
                next.link0 = new Binary4 (d, next);
            }
            return next.link0;
        }
    }
    return d == Digit.ONE ? ONE : ZERO;
}
{% endhighlight %}

This is the example of a use of the new method:

{% highlight Java %}
public Binary4 S ()
{
    if (lowest == Digit.ZERO) return newBinary (Digit.ONE, next);
    if (next == null) return TWO;
    return newBinary (Digit.ZERO, next.S ());
}
{% endhighlight %}

The numbers `ZERO` and `ONE` are created as before (calling the constructor), but `TWO` must be created using new method, to make sure that `ONE` gets
correct value in its `link0`:

{% highlight Java %}
public static Binary4 TWO = newBinary (Digit.ZERO, ONE);
{% endhighlight %}

Now there is at most one object for each number, and that object is the head of a link list that ends at `ONE`, unless the number is `ZERO`. These are the only two numbers with
`next` set to `null`.

Since the numbers are now singletons, we can replace the equality test with comparison of references:

{% highlight Java %}
public boolean isZero ()
{
    return this == ZERO;
}

@Override
public boolean equals (Object other)
{
    return this == other;
}
{% endhighlight %}

Here are the new times:


<table class="numeric">
<tr><th>Version</th><th>Comment</th><th>Pythagorean, time for 1,000, sec</th><th>Perfect, time for 10,000, sec</th></tr>
<tr><td class="label">Binary3</td><td class="ttext">Multiplication and division improved</td> <td>221</td><td> 18</td></tr>
<tr><td class="label">Binary4</td><td class="ttext">Making numbers singletons</td> <td>199</td><td> 17.5</td></tr>
</table>

Caching the increments
----------------------

The singleton nature of numbers allows another improvement: caching the results of `S()`, in the same way we were doing it in `Peano` numbers:

{% highlight Java %}
private Binary5 s = null;

public Binary5 S ()
{
    if (s == null) {
        s = lowest == Digit.ZERO ? newBinary (Digit.ONE, next) :
              next == null ? TWO : newBinary (Digit.ZERO, next.S ());
    }
    return s;
}
{% endhighlight %}

We won't cache results of `D()` for now, because `D()` is currently never called.

Here are the times:

<table class="numeric">
<tr><th>Version</th><th>Comment</th><th>Pythagorean, time for 1,000, sec</th><th>Perfect, time for 10,000, sec</th></tr>
<tr><td class="label">Binary4</td><td class="ttext">Making numbers singletons</td> <td>199</td><td> 17.5</td></tr>
<tr><td class="label">Binary5</td><td class="ttext">Caching results of <code>S()</code></td> <td>190</td><td> 16.9</td></tr>
</table>

Manual optimisations of the user code: `pythagorean`
----------------------------------------------------

While working with Peano numbers, we've performed manually some optimisations that the compiler usually does automatically. We can now apply the same transformations
again:


<table class="numeric">
<tr><th>Version</th><th>Comment</th><th>Pythagorean, time for 1,000, sec</th></tr>
<tr><td class="label">Binary5.pythagorean2</td><td class="ttext">Caching results of <code>S()</code></td> <td>190</td></tr>
<tr><td class="label">Binary5.pythagorean3</td><td class="ttext">Some code moved out of loops</td> <td>106.6</td></tr>
<tr><td class="label">Binary5.pythagorean4</td><td class="ttext">More code moved out of loops</td> <td>60.7</td></tr>
<tr><td class="label">Binary5.pythagorean5</td><td class="ttext">Strength reduction</td><td>30.9</td></tr>
<tr><td class="label">Binary5.pythagorean6</td><td class="ttext">More strength reduction</td><td>33.4</td></tr>
</table>

We see some performance degradation in `pythagorean6`. The difference between the last two versions is that `pythagorean5` maintains variables `c_square`, `b_square` and
`a_square`, and calculates

{% highlight Java %}
c_square_minus_b_square = c_square.minus (b_square)
{% endhighlight %}

while `pythagorean6` maintains `c_square_minus_b_square` instead of `b_square`.
Possibly, maintenance of this variable is more expensive, because it requires subtraction (two of those per loop), which is slower than addition.

Improving remainder calculation
-------------------------------

In `Peano` project we improved the speed of `perfect` by creating a specialised version of division: the routine that checks if one number is divisible by another one
without reporting quotient or remainder. We can do something similar in our case, too. When only the remainder is required, there is no need to calculate the quotient.
This makes the `DivResult` structure unnecessary and saves some memory allocations. There  is no need to create a special `divisible` method, because internally it would
calculate the remainder anyway. This means that we don't need a special version of `perfect()`, which we had in `Peano` (`perfect3()`), we'll rather make a new version of
`rem()` (see `Binary6.java`):

{% highlight Java %}
public Binary6 rem (Binary6 other)
{
    if (this.lt (other)) {
        return this;
    }
    Binary6 r = this.shift_right ().rem (other).shift_left ();
    if (lowest == Digit.ONE) {
        r = r.S ();
    }
    if (r.geq (other)) {
        r = r.minus (other);
    }
    return r;
}
{% endhighlight %}

This is the time:

<table class="numeric">
<tr><th>Version</th><th>Comment</th><th>Perfect, time for 10,000, sec</th></tr>
<tr><td class="label">Binary5</td><td class="ttext">Caching results of <code>S()</code></td> <td> 16.9</td></tr>
<tr><td class="label">Binary6</td><td class="ttext">Improved <code>rem()</code></td> <td> 15.9</td></tr>
</table>

I expected better improvement, but 6% isn't insignificant.

The final touch
---------------

In `Peano` project the last improvement we made in `perfect()` was to run the internal loop to `i/2` instead of `i`. Since division by 2 was slow, we implemented it by
introducing the induction variable that equals double the loop variable.

Now division by 2 is fast, so our job is easier:

{% highlight Java %}
public static void perfect4 (Binary6 N)
{
    Debug.Timer.start ();
    for (Binary6 i = ONE; i.leq (N); i = i.S ()) {
        Binary6 sum = ZERO;
        Binary6 k = i.shift_right ().S ();
        for (Binary6 j = ONE; j.neq (k); j = j.S ()) {
            if (i.rem (j).isZero ())
                sum = sum.plus (j);
        }
        if (i.eq (sum)) {
            System.out.println (i);
        }
    }
    System.out.println ("Total: ");
    Debug.Timer.stop ();
    Debug.Timer.dump ();
}
{% endhighlight %}

Here is the result:

<table class="numeric">
<tr><th>Version</th><th>Comment</th><th>Perfect, time for 10,000, sec</th></tr>
<tr><td class="label">Binary6.perfect2</td><td class="ttext">Improved <code>rem()</code></td> <td> 15.9</td></tr>
<tr><td class="label">Binary6.perfect4</td><td class="ttext">Running the inner loop to <code>i/2</code></td> <td> 10.3</td></tr>
</table>

The binary tree
---------------

It is interesting to look at the structure that our numbers form. The number `0` is separate -- it consists of a single node with the digit `0`.
Everything else is arranged into a binary tree, with the number `1` as a root.

<img src="{{ site.url }}/images/binary-tree.png" width="637" height="337">

Each node has a parent reference `next` (node `1` has it equal to `null`), and two children references `link0` and `link1`, which are drawn as pointing to the left
and the right child, respectively. For a node representing the number _n_ these children represent numbers _2n_ and _2n + 1_.
The nodes are marked with digits corresponding to their position relative to the parent node: left nodes are marked "0", right ones "1".
This is the lowest digit in the binary representation of the number. The path from the number to node `1` gives the entire binary representation,
the root node being the highest digit. Every number, except for `0`, has the highest digit one, which agrees with the convention of avoiding the leading zeroes.

The tree doesn't have to be fully built; the children are added as they are needed. For instance, while running `pythagorean5`, the biggest number seen
is 1,000,000, while the size of the tree is 430,743. 

This tree is a pretty picture, but it doesn't help performing operations with numbers. Some algorithms might exist that perform computations faster using graph processing
techniques, but I don't know them. Moreover, the value of keeping this tree in memory isn't great -- when we introduced it in `Binary4`, the times only improved from 221 sec to
199 sec and from 18 sec to 17.5 sec. If we undo the changes introduced in `Binary4` and `Binary5` and run the latest version of the user code, we get

<table class="numeric">
<tr><th>Version</th><th>Comment</th><th>Pythagorean, time for 1,000, sec</th><th>Perfect, time for 10,000, sec</th></tr>
<tr><td class="label">Binary6</td><td class="ttext">Best versions of both tests</td> <td>30.9</td><td> 10.3</td></tr>
<tr><td class="label">Binary6X</td><td class="ttext">Best versions of both tests, no tree in memory</td> <td>32.7</td><td> 10.7</td></tr>
</table>

Additional versions: null as zero
---------------------------------

While discussing possible strategies for implementing binary numbers, we mentioned representing number zero as an empty list. This could make the program more regular --
for instance, subtraction wouldn't have to check for `null` and convert it to `ZERO`. The need for separare `D0()` and `D()` would disappear, and so on. The problem is that
we can't invoke any methods on `null`. Either all the methods must be replaced with ordinary static binary functions, or another class must be introduced that would
encapsulate the list.

We've tried the first approach (see class `BinaryN`). The conversion is straightforward. I'll show some of the new methods. This is the new `D()`:

{% highlight Java %}
private static BinaryN D (BinaryN x)
{
    if (x == null)
        throw new IllegalArgumentException ("Negative number");
    if (x == ONE)
        return null;
    if (x.lowest == Digit.ONE)
        return newBinary (Digit.ZERO, x.next);
    return newBinary (Digit.ONE, D(x.next));
}
{% endhighlight %}

And this is multiplication:

{% highlight Java %}
public static BinaryN mult (BinaryN x, BinaryN y)
{
    if (isZero (y)) return ZERO;
    BinaryN r = shift_left (mult (x, shift_right (y)));
    if (y.lowest == Digit.ONE) r = plus (r, x);
    return r;
}
{% endhighlight %}

The rest of the code is converted in a similar fashion.

The user code must be converted, too, and this is where the disadvantage of the new structure becomes clearly visible. The new code looks ugly. To illustrate this, I'll
show a single line of the old code and its version in the new code.

This like from `pythagorean6`

{% highlight Java %}
for (Binary6 b = ONE, c_square_minus_b_square = c.square ().D (); b.neq (c);
     c_square_minus_b_square =
         c_square_minus_b_square.minus (b).minus (b).D (), b = b.S ()) {
{% endhighlight %}

now looks like this:

{% highlight Java %}
for (BinaryN b = ONE, c_square_minus_b_square = D (square (c)); neq (b, c);
     c_square_minus_b_square =
         D (minus (minus (c_square_minus_b_square, b), b)), b = S (b)) {
{% endhighlight %}

The object notation looks much prettier and easier to read (although, operator overloading, such as in **C++**, would have looked even better).

The results aren't great:

<table class="numeric">
<tr><th>Version</th><th>Comment</th><th>Pythagorean, time for 1,000, sec</th><th>Perfect, time for 10,000, sec</th></tr>
<tr><td class="label">Binary6</td><td class="ttext">Best results so far</td><td>30.9</td><td>10.3</td></tr>
<tr><td class="label">BinaryN</td><td class="ttext">null as zero, no object notation</td> <td>30.8</td><td> 10.4</td></tr>
</table>

Additional versions: booleans as digits
---------------------------------------

We also planned to try using `boolean`s to represent digits. The change is easy. This is the example of the new code (class `BinaryB`):

{% highlight Java %}
private static BinaryB add_no_carry (BinaryB x, BinaryB y)
{
    if (x == null) return y;
    if (y == null) return x;

    boolean d = x.lowest != y.lowest;
    BinaryB s = (x.lowest && y.lowest) ?
        add_carry (x.next, y.next) :
        add_no_carry (x.next, y.next);
    return newBinary (d, s);
}
{% endhighlight %}

Here are the results:

<table class="numeric">
<tr><th>Version</th><th>Comment</th><th>Pythagorean, time for 1,000, sec</th><th>Perfect, time for 10,000, sec</th></tr>
<tr><td class="label">Binary6</td><td class="ttext">Best results so far</td><td>30.9</td><td>10.3</td></tr>
<tr><td class="label">BinaryB</td><td class="ttext">Booleans as digits</td> <td>30.0</td><td> 9.0</td></tr>
</table>

This is the best solution we've produced.

The summary
-----------

Here is the full table of results:

<table class="numeric">
<tr><th>Version</th><th>Comment</th><th>Pythagorean, time for 1,000, sec</th><th>Perfect, time for 10,000, sec</th></tr>
<tr><td class="label">Peano6</td><td class="ttext">Optimised Peano implementation</td> <td>360</td><td>336</td></tr>
<tr><td class="label">Binary</td><td class="ttext">Original binary implementation</td> <td>275</td><td> 25</td></tr>
<tr><td class="label">Binary2</td><td class="ttext">Equality test improved</td> <td>263</td><td> 24</td></tr>
<tr><td class="label">Binary3</td><td class="ttext">Multiplication and division improved</td> <td>221</td><td> 18</td></tr>
<tr><td class="label">Binary4</td><td class="ttext">Making numbers singletons</td> <td>199</td><td> 17.5</td></tr>
<tr><td class="label">Binary5</td><td class="ttext">Caching results of <code>S()</code></td> <td>190</td><td> 16.9</td></tr>
<tr><td class="label">Binary5.pythagorean3</td><td class="ttext">Some code moved out of loops</td> <td>106.6</td><td></td></tr>
<tr><td class="label">Binary5.pythagorean4</td><td class="ttext">More code moved out of loops</td> <td>60.7</td><td></td></tr>
<tr><td class="label">Binary5.pythagorean5</td><td class="ttext">Strength reduction</td><td>30.9</td><td></td></tr>
<tr><td class="label">Binary5.pythagorean6</td><td class="ttext">More strength reduction</td><td>33.4</td><td></td></tr>
<tr><td class="label">Binary6</td><td class="ttext">Improved <code>rem()</code></td> <td></td><td> 15.9</td></tr>
<tr><td class="label">Binary6.perfect4</td><td class="ttext">Running the inner loop to <code>i/2</code></td> <td></td><td> 10.3</td></tr>
<tr><td class="label">Binary6X</td><td class="ttext">Best versions of both tests, no tree in memory</td> <td>32.7</td><td> 10.7</td></tr>
<tr><td class="label">BinaryN</td><td class="ttext">null as zero, no object notation</td> <td>30.8</td><td> 10.4</td></tr>
<tr><td class="label">BinaryB</td><td class="ttext">Booleans as digits</td> <td>30.0</td><td> 9.0</td></tr>
<tr><td class="label">Native</td><td class="ttext">Classic <b>Java</b> implementation</td><td>0.571</td><td>0.28</td></tr>
</table>

Conclusions
-----------

- This is still a valid computation-free implementation: there are no `int` variables or numeric constants in the code.

- This implementation does not use any classes from the standard library.

- As we expected, the new version is faster than the strict `Peano` version: the code runs 10-30 times faster. The speed difference may increase if we move to bigger
numbers.

- Unlike the `Peano` version, the new code isn't limited in number size. Even the caching versions only keep the numbers that were seen, and caching can be switched off
with relatively small performance loss. 

- The code now runs 30 -- 60 times slower than the native version, as if it was running on a CPU with 40-80 MHz clock speed, such as the first Pentium (66 MHz).
Not just student's assignments - serious business applications ran on those computers.
In the previous article we've made a journey into the late 70's - early 80's, now we've been in middle 90's.

Coming soon
-----------

We've had a some fun playing with the aftificial numbers, but this is the end of this story.
Now it's time to get back to our previous activity. We were implementing the game of Life using a hash table.
Many improvements have been applied, but I have some more ideas.
