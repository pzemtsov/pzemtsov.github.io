---
layout: post
title:  "Implementing Peano arithmetic"
date:   2015-12-22 12:00:00
tags: Java hashmap Peano
story: peano
story-title: "Numbers from scratch"
---

As Christmas and New Year are approaching, let's have a break from the serious stuff and have some fun (not that I consider
[study of Conway's Game of Life]({{ site.ART-HASH }}) serious, but this will be much less serious). We are going to find ourselves in the middle of a horror story.

All the code written along the way will be available [in this repository]({{ site.REPO-PEANO }}/tree/37dc603cdada3752e95c3f16ae9249ee2ce774f2).

Imagine you are a high school student, and you are learning **Java**. The current topic is arithmetic, and your assignment for today consists of two very easy numerical
examples. One is to print all the [Pythagorean triples](https://en.wikipedia.org/wiki/Pythagorean_triple) up to 100,
and another one is to print all the [perfect numbers](https://en.wikipedia.org/wiki/Perfect_number), also up to 100.
The reason these two were assigned was that they utilise both multiplication and division. No optimisation was required, any solution
was acceptable. Since the problems are so easy, you haven't done anything for a week. Finally, you get to a computer and quickly type:

{% highlight Java %}
public class Assignment
{
    public static void pithagorean (int N)
    {
        for (int c = 1; c <= N; c++)
            for (int b = 1; b < c; b++)
                for (int a = 1; a < b; a++)
                    if (a*a + b*b == c*c)
                        System.out.println (a + " " + b + " " + c);
        
    }
    
    public static void perfect (int N)
    {
        for (int i = 1; i <= N; i++) {
            int sum = 0;
            for (int j = 1; j < i; j++)
                if (i % j == 0)
                    sum += j;
            if (i == sum)
                System.out.println (i);
        }
    }
    
    public static void main (String [] args)
    {
        pithagorean (100);
        perfect (100);
    }
}
{% endhighlight %}

You try running, it and here comes the disaster: it does not work. It does not even compile. Soon you learn what happened.
Your computer (and possibly every other computer in the world) was attacked by the hackers from
the Anti-Numeric League, which objects using computers for computations. This sounds odd (after all, why are they called computers?), but the League claims that
computers have done enough computations in last 70 years and now deserve
some rest (here they have a point; however, their idea of "rest", as we'll see soon, isn't quite right).
They couldn't modify the instruction set of your computer, and the
**Java** interpreter seems to be intact. The compiler, however, lost numeric literals and types. You can't write or use any methods taking or returning `int` or other
such types. The runtime library lost classes such as `BigInteger` and `BigDecimal`. You can write native code or create your own class file in a hex editor, but your
assignment must be in **Java**. Your failure is imminent.

The only class you have that performs something numeric is the one you used for some previous assignment to print times, memory sizes, etc. It's called `Debug`, and its functionality
is not rich enough to implement arithmetic.

Hand-made numbers: Peano arithmetic
-----------------------------------

You have no choice but to implement your own numbers. It is possible, despite the famous quote from Leopold Kronecker ("God made the natural numbers; all else is the work of man.")
Kronecker didn't have **Java** VM at his disposal, while we do.

What is a natural number anyway? The prevailing approach in modern mathematics is to define natural numbers as objects that satisfy the set of
[Peano axioms](https://en.wikipedia.org/wiki/Peano_axioms).
The axioms use two basic concepts: the number zero (_0_) and operation _S&nbsp;(n)_, which for every number _n_ produces another number (which we call simply _n+1_).
From that, addition and multiplication are constructed, and all the properties are defined.

A very simple model exists for this set of axioms, based on the set theory. In this model, the empty set (&empty;) serves as zero, and operation _S(n)_ is defined like this:

 <div class="formula">
 S(n) = n&nbsp;&cup;&nbsp;{n}
 </div>

Here are some small numbers in this model:

 <div class="formula">
 0 = &empty;<br/>
 1 = {&empty;}<br/>
 2 = {&empty;,{&empty;}}<br/>
 3 = {&empty;,{&empty;},{&empty;,{&empty;}}}
 </div>

 
Here are some useful properties of the numbers in this model:

- |n| = n (the number of elements in the set denoting a number equals that number)

- a <= b &hArr; a &sub; b  (_less-or-equal_ relation is equivalent to the _subset_ relation)

- a < b &hArr; a &isin; b (_less_ relation is equivalent to the membership relation)

- S (max (x | x &isin; a)) = a (the maximal element in _a_ is _a&minus;1_).

The last property allows us to define the operation _D&nbsp;(n)_, which is the inverse of _S&nbsp;(n)_:

  <div class="formula">
   D (n) = max (x | x &isin; n)
  </div>

We can then define addition and subtraction:

  <div class="formula">
  <table>
  <tr>
    <td rowspan=2>a&nbsp;+&nbsp;b = <span class="big">{</span></td>
    <td class="left">S(a&nbsp;+&nbsp;D(b));</td>
    <td class="space"></td>
    <td class="left">b&nbsp;&ne;&nbsp;0</td>
  </tr>
  <tr>
    <td class="left">a;</td>
    <td></td>
    <td class="left">b&nbsp;=&nbsp;0</td>
  </tr>
  </table>
  </div>
  
  <div class="formula">
  <table>
  <tr>
    <td rowspan=2>a&nbsp;&minus;&nbsp;b = <span class="big">{</span></td>
    <td class="left">D(a&nbsp;&minus;&nbsp;D(b));</td>
    <td class="space"></td>
    <td class="left">b&nbsp;&ne;&nbsp;0</td>
  </tr>
  <tr>
    <td class="left">a;</td>
    <td></td>
    <td class="left">b&nbsp;=&nbsp;0</td>
  </tr>
  </table>
  </div>

Note that _D(n)_ is undefined for _n=0_, and _a&minus;b_ is undefined for _a<b_. The Peano arithmetic does not know negative numbers. We can survive without them for now.
We'll make some plan if we need them.

We can define multiplication, too:

  <div class="formula">
  <table>
  <tr>
    <td rowspan=2>a&nbsp;&times;&nbsp;b = <span class="big">{</span></td>
    <td class="left">a&nbsp;&times;&nbsp;D(b)&nbsp;+&nbsp;a;</td>
    <td class="space"></td>
    <td class="left">b&nbsp;&ne;&nbsp;0</td>
  </tr>
  <tr>
    <td class="left">0;</td>
    <td></td>
    <td class="left">b&nbsp;=&nbsp;0</td>
  </tr>
  </table>
  </div>


The reason I pay so much attention to Peano numbers and their model is that we're now going to implement it.
This model was not really created to be implemented; it has more of a theoretical value -- for instance, it allows easy generalisation to
transfinite numbers. Still, **Java** has all that's needed: it has sets, so we can implement the model. We'll use `HashSet` and follow all the formulas
exactly. This promises to be very inefficient -- in fact, we are now going to write the ultimately inefficient program. The good news is that we don't need
integer values anywhere in the process. They do occur in public API of sets, in methods like `size()`, so we'll have to avoid calling these methods.

Our class will be called `Peano1`, and it will extend `HashSet<Peano1>`. **Java**'s type system allows us to use the class as a type parameter when defining itself.
In our case it makes perfect sense, for `Peano1` will be a set of `Peano1` elements.

{% highlight Java %}
public class Peano1 extends HashSet<Peano1>
{
    public static Peano1 ZERO = new Peano1 ();
    
    private Peano1 () {
        super ();
    }

    public boolean isZero ()
    {
        return this.equals (ZERO);
    }
    
    public Peano1 S()
    {
        Peano1 s = new Peano1 ();
        s.addAll (this);
        s.add (this);
        return s;
    }
}
{% endhighlight %}

Let's print the number `3`:
{% highlight Java %}
    System.out.println (ZERO.S ().S ().S ());
{% endhighlight %}

Here is the output:

    [[], [[]], [[], [[]]]]

Now we need _D()_:

{% highlight Java %}
    public Peano1 D()
    {
        if (isZero ()) throw new IllegalArgumentException ("D called on Zero");
        Peano1 max = null;
        for (Peano1 x : this)
        {
            if (max == null || x.contains (max)) {
                max = x;
            }
        }
        return max;
    }
{% endhighlight %}

Now we need comparisons. The equality is already there (standard `equals()` is good enough). The others can be implemented like this:

{% highlight Java %}
    public boolean lt (Peano1 other)
    {
        return other.contains (this);
    }

    public boolean gt (Peano1 other)
    {
        return other.lt (this);
    }
    
    public boolean leq (Peano1 other)
    {
        return ! gt (other);
    }
    
    public boolean geq (Peano1 other)
    {
        return ! lt (other);
    }
{% endhighlight %}

Note: we also could also have implemented `leq()` like this:

{% highlight Java %}
    public boolean leq (Peano1 other)
    {
        return other.containsAll (this);
    }
{% endhighlight %}

Now we implement `plus` and `minus`. I'd like to call them `add` and `sub`, but `HashSet` already has `add()` method, which does something else.
This is a negative side-effect of extending a class without intent to retain its functionality. Perhaps, aggregation would have been a better option,
but here we follow the Peano model strictly (a number **is** a set, not just is built on top of a set).
The implementation follows the formulas exactly:

{% highlight Java %}
    public Peano1 plus (Peano1 other)
    {
        return other.isZero () ? this : plus (other.D ()).S ();
    }

    public Peano1 minus (Peano1 other)
    {
        return other.isZero () ? this : minus (other.D ()).D ();
    }
{% endhighlight %}

The multiplication is done in the same way:

{% highlight Java %}
    public Peano1 mult (Peano1 other)
    {
        return other.isZero () ? ZERO : mult (other.D ()).plus (this);
    }
    
    public Peano1 square ()
    {
        return mult (this);
    }
{% endhighlight %}

Division will be implemented in more traditional iterative way:

{% highlight Java %}
    public static class DivResult
    {
        final Peano1 quotient;
        final Peano1 remainder;
        
        public DivResult (Peano1 quotient, Peano1 remainder)
        {
            this.quotient = quotient;
            this.remainder = remainder;
        }
    }
    
    public DivResult divrem (Peano1 other)
    {
        Peano1 div = ZERO;
        Peano1 rem = this;
        while (rem.geq (other)) {
            rem = rem.minus (other);
            div = div.S ();
        }
        return new DivResult (div, rem);
    }

    public Peano1 div (Peano1 other)
    {
        return divrem (other).quotient;
    }

    public Peano1 rem (Peano1 other)
    {
        return divrem (other).remainder;
    }
{% endhighlight %}

The natural numbers arithmetic is ready to be used. However, we still need input and output:

{% highlight Java %}
    public static Peano1 ONE = ZERO.S();
    public static Peano1 TWO = ONE.S();
    public static Peano1 THREE = TWO.S();
    public static Peano1 FOUR = THREE.S();
    public static Peano1 FIVE = FOUR.S();
    public static Peano1 SIX = FIVE.S();
    public static Peano1 SEVEN = SIX.S();
    public static Peano1 EIGHT = SEVEN.S();
    public static Peano1 NINE = EIGHT.S();
    public static Peano1 TEN = NINE.S();

    private Peano1 (Peano1 other)
    {
        super (other);
    }
    
    public Peano1 (String str)
    {
        this (parseInt (str));
    }

    private static String digitToString (Peano1 p)
    {
        if (p.equals (ZERO))  return "0";
        if (p.equals (ONE))   return "1";
        if (p.equals (TWO))   return "2";
        if (p.equals (THREE)) return "3";
        if (p.equals (FOUR))  return "4";
        if (p.equals (FIVE))  return "5";
        if (p.equals (SIX))   return "6";
        if (p.equals (SEVEN)) return "7";
        if (p.equals (EIGHT)) return "8";
        if (p.equals (NINE))  return "9";
        throw new IllegalArgumentException ();
    }

    private static Peano1 charToDigit (char c)
    {
        switch (c) {
        case '0' : return ZERO;
        case '1' : return ONE;
        case '2' : return TWO;
        case '3' : return THREE;
        case '4' : return FOUR;
        case '5' : return FIVE;
        case '6' : return SIX;
        case '7' : return SEVEN;
        case '8' : return EIGHT;
        case '9' : return NINE;
        default:
            throw new IllegalArgumentException ();
        }
    }
    
    @Override
    public String toString ()
    {
        DivResult d = this.divrem (TEN);
        String digit = digitToString (d.remainder);
        return d.quotient.isZero () ? digit : d.quotient + digit;
    }
    
    private static Peano1 parseInt (String s)
    {
        Peano1 result = ZERO;
        for (char c : s.toCharArray ()) {
            result = result.mult (TEN).plus (charToDigit (c));
        }
        return result;
    }
{% endhighlight %}

The `parseInt()` method iterates over array, but, fortunately, this can be done without use of integer variables or arithmetic operators.

The first test
--------------

Finally, we are ready to write our assignment:

{% highlight Java %}
    public static void pythagorean (Peano1 N)
    {
        for (Peano1 c = ONE; c.leq (N); c = c.S ())
            for (Peano1 b = ONE; b.lt (c); b = b.S ())
                for (Peano1 a = ONE; a.lt (b); a = a.S ())
                    if (a.square().plus (b.square()).equals (c.square()))
                        System.out.println (a + " " + b + " " + c);
    }

    public static void perfect (Peano1 N)
    {
        for (Peano1 i = ONE; i.leq (N); i = i.S ()) {
            Peano1 sum = ZERO;
            for (Peano1 j = ONE; j.lt (i); i = i.S ())
                if (i.rem (j).isZero ())
                    sum = sum.plus (j);
            if (i.equals (sum))
                System.out.println (i);
        }
    }
    
    public static void main (String [] args)
    {
        pythagorean (new Peano1 ("100"));
        perfect (new Peano1 ("100"));
    }
{% endhighlight %}

When we run it, it just hangs and prints nothing. The program is way too slow. We agreed upfront that we wouldn't pursue high performance, but it looks like poor
performance must have its limits.

Let's use the stack trace as a poor man's profiler. We'll let the program run for a while and then print the stack:

    java.lang.Thread.State: RUNNABLE
        at java.util.AbstractSet.hashCode(Unknown Source)
        at java.util.AbstractSet.hashCode(Unknown Source)
        at java.util.AbstractSet.hashCode(Unknown Source)
        at java.util.AbstractSet.hashCode(Unknown Source)
        at java.util.AbstractSet.hashCode(Unknown Source)
        at java.util.AbstractSet.hashCode(Unknown Source)
        at java.util.AbstractSet.hashCode(Unknown Source)
        at java.util.AbstractSet.hashCode(Unknown Source)
        at java.util.AbstractSet.hashCode(Unknown Source)
        at java.util.AbstractSet.hashCode(Unknown Source)
        at java.util.HashMap.hash(Unknown Source)
        at java.util.HashMap.put(Unknown Source)
        at java.util.HashSet.add(Unknown Source)
        at java.util.AbstractCollection.addAll(Unknown Source)
        at Peano1.S(Peano1.java:40)
        at Peano1.plus(Peano1.java:80)
        at Peano1.plus(Peano1.java:80)
        at Peano1.mult(Peano1.java:90)
        at Peano1.mult(Peano1.java:90)
        at Peano1.mult(Peano1.java:90)
        at Peano1.mult(Peano1.java:90)
        at Peano1.mult(Peano1.java:90)
        at Peano1.mult(Peano1.java:90)
        at Peano1.mult(Peano1.java:90)
        at Peano1.mult(Peano1.java:90)
        at Peano1.parseInt(Peano1.java:176)
        at Peano1.<init>(Peano1.java:29)
        at Peano1.main(Peano1.java:206)

It shows that the program hasn't even started running `pythagorean()`. It is still busy with `new Peano1 ("100")`, performing multiplication.

Let's simplify the problem for now. We'll just print natural numbers, starting from zero and going up:

{% highlight Java %}
    static void inctest ()
        Peano1 x = ZERO;
        while (true) {
            Debug.Timer.start ();
            x = x.S ();
            System.out.println (x);
            Debug.Timer.stop ();
            Debug.Timer.dump ();
        }
    }
{% endhighlight %}

This code slows down at about iteration 23:

 <table class="numeric">
 <tr><th>Iteration</th><th>Time, ms</th></tr>
 <tr><td>23</td><td> 1483</td></tr>
 <tr><td>24</td><td> 3356</td></tr>
 <tr><td>25</td><td> 7134</td></tr>
 <tr><td>26</td><td>12541</td></tr>
 </table>

We see that each iteration takes about twice the time of the previous one, so the forecast is very pessimistic. We'll get to the number 100 in about

 <div class="formula">
 12&times;2<sup>74</sup> s &asymp; 2&times;10<sup>23</sup> s &asymp; 7&times;10<sup>15</sup> years.
 </div>

Of course, some time is spent in `System.out.println (x)`. If we remove it, we can get to bigger numbers:

 <table class="numeric">
 <tr><th>Iteration</th><th>Time with printing</th><th>Time without printing</th></tr>
 <tr><td>23</td><td> 1483</td><td>  141</td></tr>
 <tr><td>24</td><td> 3356</td><td>  231</td></tr>
 <tr><td>25</td><td> 7134</td><td>  442</td></tr>
 <tr><td>26</td><td>12541</td><td>  884</td></tr>
 <tr><td>27</td><td>     </td><td> 1767</td></tr>
 <tr><td>28</td><td>     </td><td> 3540</td></tr>
 <tr><td>29</td><td>     </td><td> 7073</td></tr>
 <tr><td>30</td><td>     </td><td>14138</td></tr>
 </table>

Printing of numbers does indeed take more time than incrementing. However, the exponential law still holds, with the shift of 4. The time to get to number 100 becomes
almost 16 time less, which is much better, but still hardly satisfactory (remember, the assignment must be handed in today).

It shows that a direct implementation of Peano arithmetic isn't really practical when big numbers are involved. It is still possible to get some results with small numbers.
For instance, if we remove the check for the upper boundary and do not create number 100, `perfect()` finds the first perfect number (6) in about 4 ms. It took 4820
seconds to print the next one (28). The first Pythagorean triple (3, 4, 5) is printed after 12.2 sec. This is all it could do, though, because to carry on it needed
the number _6<sup>2</sup> = 36_, which takes too long to compute.

Why is it so slow?
------------------

A look at the above stack trace shows one of the reasons why the execution is so slow. The program spends its time deep inside recursive `hashCode()`. Why?
Consider what happens during `S()` operation.

{% highlight Java %}
public class Peano1 extends HashSet<Peano1>
{
    public Peano1 S()
    {
        Peano1 s = new Peano1 ();
        s.addAll (this);
        s.add (this);
        return s;
    }
}
{% endhighlight %}

We first add all the elements from current set to the new set. For that, we need to take each one of them and calculate its `hashCode()`.
The set does not cache its hash code; it re-calculates it each time it is needed (perhaps, use of sets as hash map keys, or collecting sets into other sets
were not considered important cases).
The hash code of the set is computed as the sum of hash codes of all the elements. Since these elements are sets themselves, the process goes on and on recursively, with the square
complexity.

And the saddest part of all is that all these hash codes are zeroes. When we descend into our recursion, we'll inevitably end up at empty sets, and empty sets have
hash codes of zero. This is the most inefficient way of calculating zero that I ever came across.

After the hash code (zero) is calculated for an element, this element is looked up in the new set. For that, we get the chain (list) of all the elements with this hash code
(which in our case is the entire set), and call `equals()` of the new element with all of them. The implementation of `HashMap.equals()` has two special cases: one when two
references are the same (returns `true`) and one when the sizes differ (returns `false`). While we just call `S()` again and again, one of these cases always happens, so
`equals()` is fast. If, however, we compare `S(x)` for some `x` with the result of another invocation of `S(x)`, the `equals()` will be slow: it will have to iterate all
the elements in the first `S(x)` and check if they belong to the second one (call `hashCode()` and so on). This will inevitably happen when we start real calculations.

However, even in the fast case we still need to compare the new element with all the elements already in the set.

In short, the whole program is not just non-optimal. The program we've just written is the total, absolute feast of inefficiency. It's a miracle it works sometimes.

The `Debug` class I mentioned earlier contained a counter, which I could use to collect some statistics. I counted how many calls to
`hashCode()` and `equals()` were made while executing some `S()` and `D()` operations:

<table class="numeric">
<tr><th>Operation</th><th>Calls to <code>hashCode()</code></th><th>Calls to <code>equals()</code></th></tr>
<tr><td> S(9)</td><td>         1,023</td><td> 45</td></tr>
<tr><td>D(10)</td><td>           511</td><td> 37</td></tr>
<tr><td>S(19)</td><td>     1,048,575</td><td>232</td></tr>
<tr><td>D(20)</td><td>     2,425,472</td><td>158</td></tr>
<tr><td>S(29)</td><td> 1,073,741,823</td><td>538</td></tr>
<tr><td>D(30)</td><td>13,522,436,098</td><td>402</td></tr>
</table>

The `hashCode()` method dominates everything by far. It seems that we could improve everything a lot by just replacing `hashCode()` with a function that always returns zero.
We, however, can't do it (this function returns `int`, and `int`s do not work). Besides, we can do something better.

Caching the numbers
-------------------

Let's first introduce caching: we won't re-create the numbers that we saw before. This way every number will be present in no more than one instance. This will allow replacing
`equals()` simply with the comparison of references. It is unlikely to help much in the `inctest()`, but it may help in other calculations.

The new class is called `Peano2`.

We remove all constructors, except for the default one, so we have to make `parseInt()` public. We also introduce the field `next` that keeps the result of `S()`:

{% highlight Java %}
    private Peano2 next = null;

    @Override
    public boolean equals (Object o)
    {
        return this == o;
    }
    
    public Peano2 S()
    {
        if (next == null) {
            Peano2 s = new Peano2 ();
            s.addAll (this);
            s.add (this);
            next = s;
        }
        return next;
    }
{% endhighlight %}

Note that what we have just done is dangerous and not generally recommended. We've introduced an asymmetric `equals()`: our object will not be equal to anything else
but some other set may find itself equal to it.

As expected, the new class doesn't perform much faster on the `inctest()` (the time is the same as before), but other
calculations improve. It gets to the first Pythagorean triple after 1.036 sec (previously 12.2 sec), then stalls (probably, calculating 6&times;6).
It prints the first perfect number 6 after 2 ms, and reaches 28 in 4382 sec (previous values: 4 ms and 4820 sec).


Using identity hashCode
-----------------------

Since each number is now present in just one instance, we could use the address of this instance as the basis for the object's hash code. This is exactly what the default
implementation of `Object.hashCode()` does, which is also available via `System.identityHashCode()`. We can't, however, write

{% highlight Java %}
    public int hashCode ()
    {
        return System.identityHashCode (this);
    }
{% endhighlight %}

because the keyword `int` still does not work (this Anti-Numeric League is really beginning to annoy me; OK, their goals are fair, but the methods...).

Besides, this code would suffer from similar problem to the one mentioned above (asymmetric `equals()`). Some other set may be `equals()` to our object but still
have different `hashCode()`. This may one day bring us a lot of fun when we put our numbers and other sets into the same `HashMap`.

There is a solution, however: we must ignore the strict requirement of Peano model and replace inheritance with aggregation (it seems that this is almost always
the better way) -- make a separate class that represents a number, which would contain the set as a field.
This will have an additional advantage that the class will not implement all the set methods
and will therefore be safer -- no one will call those methods accidentally. The new class will be called `Peano3`:

{% highlight Java %}
public class Peano3
{
    public static Peano3 ZERO = new Peano3 ();
    
    private final HashSet<Peano3> set;
    private Peano3 next = null;
    
    private Peano3 (HashSet<Peano3> set)
    {
        this.set = set;
    }

    private Peano3 ()
    {
        this (new HashSet<Peano3> ());
    }

    public boolean isZero ()
    {
        return this == ZERO;
    }

    public Peano3 S()
    {
        if (next == null) {
            HashSet<Peano3> newSet = new HashSet<Peano3> ();
            newSet.addAll (set);
            newSet.add (this);
            next = new Peano3 (newSet);
        }
        return next;
    }
{% endhighlight %}

Some more minor modifications must be made, since `Peano3` isn't a `set` anymore.

Since this class doesn't extend anything but `Object`, it inherits the default `hashCode()` as well as the default `equals()`. Use of these defaults must make the entire
implementation faster.

It really does: on `inctest()` it quickly reaches 1000 and only then starts slowing down. If we don't print the number, it starts slowing down much later. Memory use
starts growing, too: number 10,000 took over two gigabytes (it took 121 seconds to produce this number). The garbage collector was called several times, the longest invocation took 5 seconds (perhaps, it helps
that we are running on a machine with 64 Gbytes of RAM).

The first Pythagorean triple was printed in 3 ms instead of 1 second previously. It printed some more, but I stopped it at (33, 44, 55), after the program took 782 seconds.

The new class performed much better with another assignment (`perfect ()`). It got to the number 6 in the same 2 ms as before, to number 28 in 16 ms (instead of 4382 seconds!)
and checked all 100 numbers in 288 ms.
We could run it further. In 187 seconds it found the next perfect number (496), and in 3572 seconds it scanned the first 1000.

Improving a decrement operation
-------------------------------

A typical stack trace of the running program now shows a long chain of recursive calls (`plus`, `minus` or `mult`) with a call to `D()` on top. This is not surprising, since
all the recursive operations are based on `D()`, and `D()` is very inefficient (it looks for the maximal element in the set). There are two ways to improve it: to get rid
of recursion and to improve `D()`. The recursion needs attention anyway, since it has a limit (the stack depth is finite), but we can address it when we hit this limit.
Let's first concentrate on `D()`. If it becomes faster, everything will improve (division too).

We can improve `D()` the same way we've improved `S()`: caching. Currently, the only way to create a new number is in `S()`, and the previous one is known at this point.
We can memorise it. The new class is called `Peano4`:

{% highlight Java %}

    private final HashSet<Peano4> set;
    private Peano4 next = null;
    private final Peano4 prev;
    
    private Peano4 (Peano4 prev, HashSet<Peano4> set)
    {
        this.prev = prev;
        this.set = set;
    }

    private Peano4 ()
    {
        this (null, new HashSet<Peano4> ());
    }
 
    public boolean isZero ()
    {
        return this == ZERO;
    }

    public Peano4 S()
    {
        if (next == null) {
            HashSet<Peano4> newSet = new HashSet<Peano4> ();
            newSet.addAll (set);
            newSet.add (this);
            next = new Peano4 (this, newSet);
        }
        return next;
    }

    public Peano4 D()
    {
        if (isZero ()) throw new IllegalArgumentException ("D called on Zero");
        return prev;
    }
{% endhighlight %}

The `inctest()` hasn't improved much, but the rest has. The `pythagorean()` finished the search up to 100 in 70 sec. It ran out of memory at 113 when left to run longer.
The `perfect()` found number 496 in 372 ms (previously: 187 seconds), and finished testing the first thousand numbers in 2.5 sec (previously 3572 seconds)
and the first 5,000 in 598 sec. It finished the first 10,000 in 6313 sec. This is really a great improvement.

The linked lists
----------------

Now the class has forward and backwards references and looks suspiciously close to a usual doubly-linked list.
A very natural question arises: why do we need a set in the first place? It came as part of a Peano
model, but after re-design of `D()` it is used in one place only: in the `lt()` method. One number is less than another number if it belongs to it as an element.
If we remove all mentioning of the sets, we are left with a linked list, which grows from `ZERO`, and where _n<sup>th</sup>_ element represents the number _n&minus;1_.
How can we compare two numbers in such a setup?

We can immediately offer several strategies:

- Start from zero and go up the list until we find one of the numbers. This will be the smaller one.

- Start from one of the numbers and go up the list until we find the other number (in which case the first one is smaller), or the end of the list (in which case the first one is bigger).

- Perform the same from both numbers at the same time.

- Traverse the list backwards starting from one of the numbers.

- Do the same from both numbers.

- Traverse the list in both directions from one of the numbers, or from both.

Possibly there are more options. Neither of these methods is the optimal, since we don't know what the inputs will be.

Let's try the option three (see the class `Peano5`):

{% highlight Java %}
    public boolean lt (Peano5 other)
    {
        if (this == other) return false;
        Peano5 p = this;
        Peano5 q = other;
        while (true) {
            p = p.next;
            if (p == null) return false;
            if (p == other) return true;
            q = q.next;
            if (q == null) return true;
            if (q == this) return false;
        }
    }
{% endhighlight %}

Here are the test results:

The `inctest()` happily ran to 144,000 in one minute, the time mostly spent in printing (it takes 18 ms to construct the number 1,000,000 and 66 sec to print it).

The `pythagorean()` took 9 sec to get to 100 (previously 70 sec) and slowed down after that. It reached 200 in 312 sec and then -- surprise! -- crashed with StackOverflowException

The `perfect()` reached 1000 in 3.8 sec, 5000 in 571 sec and 10,000 in 5216 sec (the previous values were 2.5 sec, 598 sec and 6317 sec). It 

Surprisingly, the new class isn't much faster than the previous one. Probably, the `lt()` method is called often, and the previous version was faster. The sets use memory but
are still useful, sometimes.

Iteration or recursion?
-----------------------

For most young programmers these days the answer to this question is so straightforward that one can say there is no question. Of course, recursion.
Everyone is overly excited about anything with the word "functional" in it, and this involves using recursion everywhere. The best languages are those that do not even contain
loop constructs. Ironically, my first programming language was like that; perhaps, this made me a bit sceptical about this issue. Maybe, too sceptical. When I got
exposed to the internals of the computer, I became even less convinced that the best way to iterate over a sequence of integer indices is to first push the entire
sequence onto the stack, interleaved with the return addresses and, possibly, stack and frame pointers. Tail recursion is an exception, for it can be compiled to
conventional iteration. Our code, however, isn't using tail recursion:

{% highlight Java %}
    public Peano5 plus (Peano5 other)
    {
        return other.isZero () ? this : plus (other.D ()).S ();
    }
{% endhighlight %}

It is calling `S()` on the result of the recursive call. **Java** does not convert this recursion into iteration, which causes stack overflow when the call nesting level
is too high. Even though this method looks pretty, we'll have to rewrite it into something that, as I think, is even prettier (see `Peano6`):

{% highlight Java %}
    public Peano6 plus (Peano6 other)
    {
        Peano6 result = this;
        while (other != ZERO) {
            result = result.S ();
            other = other.D ();
        }
        return result;
    }
{% endhighlight %}

Note that we could have gone another way and have used a similar transformation without optimisation of `D()` -- we'd just need to avoid using `D()`:

{% highlight Java %}
    public Peano6 plus (Peano6 other)
    {
        Peano6 result = this;
        Peano6 count = ZERO;
        while (count != other) {
            result = result.S ();
            count = sount.S ();
        }
        return result;
    }
{% endhighlight %}

Now such a change is unnecessary, since `D()` is equally efficient as `S()`.

Here are the results:

The `pythagorean()` now took 7.2 sec to get to 100 (instead of 9 sec), 203 sec to 200 and 14000 sec to 466. It never crashed (I killed it instead).

The `perfect()` reached 1000 in 3.2 sec (3.8 sec before) and 10,000 in 4154 sec (previously 5216).

Optimisation of the main program: comparisons
---------------------------------------------

Until now we didn't touch the main program, leaving it exactly as it was written originally, just replacing native numbers with our classes.
From some point of view this is a fair approach: after all, we weren't after the fastest solution of the original problems, and better algorithms may exist for both of them.
On the other hand, the solution using the artificial numbers is in disadvantaged position, as the VM may not optimise it properly. For instance, the compiler is very unlikely
to detect calls without side effects and reduce common subexpressions. It is also unlikely to replace multiplication with addition. It is unaware of the relative
cost of operations. To achieve results that we can compare with those of native integers, we'll have to optimise some things by hand.

Let's start with elimination of "less than" comparisons. We know in our implementation it is inefficient -- requires a loop traversing the list. The most common use
of this comparison (our program is no exception) is in loops:

{% highlight Java %}
    for (int i = low; i < high; i++) {
    }
{% endhighlight %}

This tradition comes from **C**. This is indeed the most reliable way to write down a loop that would work with all combinations of `low` and `high` and (provided that the limits are
low enough to avoid arithmetic overflow) with the step values other than one. However, in simple cases, when the step is one and `low < high` is given, it can be replaced
with the equality test:

{% highlight Java %}
    for (int i = low; i != high; i++) {
    }
{% endhighlight %}

We can do that in our assignment, too:

{% highlight Java %}
    public static void pythagorean2 (Peano6 N)
    {
        for (Peano6 c = ONE; c.leq (N); c = c.S ())
            for (Peano6 b = ONE; b != c; b = b.S ())
                for (Peano6 a = ONE; a != b; a = a.S ())
                    if (a.square().plus (b.square()) == c.square())
                        System.out.println (a + " " + b + " " + c);
    }

    public static void perfect2 (Peano6 N)
    {
        for (Peano6 i = ONE; i.leq (N); i = i.S ()) {
            Peano6 sum = ZERO;
            for (Peano6 j = ONE; j != i; j = j.S ())
                if (i.rem (j) == ZERO)
                    sum = sum.plus (j);
            if (i == sum)
                System.out.println (i);
        }
    }
{% endhighlight %}

The `leq` comparisons in the outer loop do not affect general performance much, so we can leave them as they are.

The `pythagorean2()` gets to 100 in the same 7.2s, and to 100 in 202 sec -- it doesn't run faster. Perhaps, multiplication dominates over everything else.

The `perfect2()` reaches 1000 in 3 sec and 10,000 in 3874 sec -- a bit faster than previously (3.2 and 3874 sec).

Loop-invariant code motion
--------------------------

This it the optimisation that virtually all the compilers perform, and **Java** VM is no exception. We'll have to do it by hand:

{% highlight Java %}
    public static void pythagorean3 (Peano6 N)
    {
        for (Peano6 c = ONE; c.leq (N); c = c.S ()) {
            Peano6 c_square = c.square ();
            for (Peano6 b = ONE; b != c; b = b.S ()) {
                Peano6 b_square = b.square ();
                for (Peano6 a = ONE; a != b; a = a.S ()) {
                    if (a.square().plus (b_square) == c_square) {
                        System.out.println (a + " " + b + " " + c);
                    }
                }
            }
        }
    }
{% endhighlight %}

This code ran up to 100 in 2.1 sec and up to 200 in 58 sec -- good progress from 7.2 sec and 202 sec. It is the first version that managed to get to 500 -- it took 5594 seconds.

Some more code motion
---------------------

This line

{% highlight Java %}
    if (a.square().plus (b_square) == c_square) {
{% endhighlight %}

can be rewritten to create more loop-invariant code (some compilers do that, too):

{% highlight Java %}
    if (a.square() == c_square.minus (b_square)) {
{% endhighlight %}

This code may then be moved out of the loop:

{% highlight Java %}
    public static void pythagorean4 (Peano6 N)
    {
        for (Peano6 c = ONE; c.leq (N); c = c.S ()) {
            Peano6 c_square = c.square ();
            for (Peano6 b = ONE; b != c; b = b.S ()) {
                Peano6 b_square = b.square ();
                Peano6 c_square_minus_b_square = c_square.minus (b_square);
                for (Peano6 a = ONE; a != b; a = a.S ()) {
                    if (a.square() == c_square_minus_b_square) {
                        System.out.println (a + " " + b + " " + c);
                    }
                }
            }
        }
    }
{% endhighlight %}

The effect is surprisingly big. This code runs to 100 in 770 ms, to 200 in 20 sec and to 500 in 1944 sec.
This version has made it to 1000, although it took rather long - 62,259 sec (17 hours).

Operation strength reduction
----------------------------

When a variable is incremented, its square changes in controllable way. Since

 <div class="formula">
  (a&nbsp;+&nbsp;1)<sup>2</sup>&nbsp;=&nbsp;a<sup>2</sup>&nbsp;+&nbsp;2a&nbsp;+1
 </div>

we can maintain a square of the required variable and modify it accordingly:

{% highlight Java %}
    a_square += 2 * a + 1;
    a ++;
{% endhighlight %}

This way we reduce multiplication to additions, which are much cheaper. Most optimising compilers do it automatically. We'll have to do it by hand:

{% highlight Java %}
    public static void pythagorean5 (Peano6 N)
    {
        for (Peano6 c = ONE, c_square = ONE; c.leq (N);
             c_square = c_square.plus (c).plus (c).S (), c = c.S ())
        {
            for (Peano6 b = ONE, b_square = ONE; b != c;
                 b_square = b_square.plus (b).plus (b).S (), b = b.S ())
            {
                Peano6 c_square_minus_b_square = c_square.minus (b_square);
                for (Peano6 a = ONE, a_square = ONE; a != b;
                     a_square = a_square.plus (a).plus (a).S (), a = a.S ())
                {
                    if (a_square == c_square_minus_b_square) {
                        System.out.println (a + " " + b + " " + c);
                    }
                }
            }
        }
    }
{% endhighlight %}

This version gets to 100 in 95 ms, to 200 in 1.28 sec and to 500 in 45 sec. It gets to 1000 in 727 sec (85 times faster than before).

Another small optimisation
--------------------------

The middle loop contains subtraction

{% highlight Java %}
    Peano6 c_square_minus_b_square = c_square.minus (b_square);
{% endhighlight %}

Here both `b_square` and `c_square` are synthesized induction variables. Both are only used in the mentioned subtraction. We can replace one of them (`b_square`)
with a new synthesized variable, containing subtraction result:

{% highlight Java %}
    public static void pythagorean6 (Peano6 N)
    {
        for (Peano6 c = ONE, c_square = ONE; c.leq (N);
             c_square = c_square.plus (c).plus (c).S (), c = c.S ())
        {
            for (Peano6 b = ONE, c_square_minus_b_square = c.square ().D ();
                 b != c;
                 c_square_minus_b_square =
                 c_square_minus_b_square.minus (b).minus (b).D (), b = b.S ())
            {
                for (Peano6 a = ONE, a_square = ONE; a != b;
                     a_square = a_square.plus (a).plus (a).S (), a = a.S ())
                {
                    if (a_square == c_square_minus_b_square) {
                        System.out.println (a + " " + b + " " + c);
                    }
                }
            }
        }
    }
{% endhighlight %}

This new code runs better: it reaches 100 in 70 ms, 200 in 730 ms, 500 in 22.3 sec and 1000 in 360 sec. This will be our final version of `pythagorean()`.

Perfect numbers: optimising division
------------------------------------

The basic operation in `perfect()` is a check if one number is divisible by another one. It is performed by calling a general division method and checking if the remainder is zero.
As often happens, the improvement comes with specialisation: we implement a special method to check exactly this. To help with that, we'll also write a special version of
`minus()` operation, which does not throw exception if the result is negative, but rather reports an error in the form of `null`:

{% highlight Java %}
    public Peano6 minus_special (Peano6 other)
    {
        Peano6 result = this;
        while (other != ZERO) {
            if (result == ZERO)
                return null;
            result = result.D ();
            other = other.D ();
        }
        return result;
    }
{% endhighlight %}

The divisibility test looks like this:

{% highlight Java %}
    public boolean divisible (Peano6 other)
    {
        Peano6 rem = this;
        while (rem != ZERO) {
            rem = rem.minus_special (other);
            if (rem == null) return false;
        }
        return true;
    }
{% endhighlight %}

All we need is to use the new method in `perfect()`:

{% highlight Java %}
    public static void perfect3 (Peano6 N)
    {
        for (Peano6 i = ONE; i.leq (N); i = i.S ()) {
            Peano6 sum = ZERO;
            for (Peano6 j = ONE; j != i; j = j.S ()) {
                if (i.divisible (j))
                    sum = sum.plus (j);
            }
            if (i == sum)
                System.out.println (i);
        }
    }
{% endhighlight %}

This method reaches 1000 in 800 ms and 10,000 in 648 sec, which is a good improvement over 3 sec and 3874 sec before.

The final touch
---------------

Obviously, there are many ways to improve calculation of perfect numbers by applying better algorithms. This is not our point today. That's why we'll limit ourselves
to one very simple modification: running the inner loop to `i/2`. Division by 2 is quick in traditional arithmetics, but not in our simulation. We'll replace it with
multiplication and introduce new induction variable:

{% highlight Java %}
    public static void perfect4 (Peano6 N)
    {
        int count = 0;
        for (Peano6 i = TWO; i.leq (N); i = i.S ()) {
            Peano6 sum = ZERO;
            Peano6 j = ONE;
            Peano6 j_times_2 = TWO;
            while (true) {
                if (i.divisible (j))
                    sum = sum.plus (j);
                if (j_times_2 == i) break;
                j_times_2 = j_times_2.S ();
                if (j_times_2 == i) break;
                j_times_2 = j_times_2.S ();
                j = j.S ();
            }
            if (i == sum)
                System.out.println (i);
        }
    }
{% endhighlight %}

It reaches 1000 in 480 ms and 10,000 in 336 sec, which means it runs roughly twice as fast as before, which is what we expected.

The epiloque
------------

The appointment has been passed successfully, despite all the effort of the hackers to prevent it. We have implemented the number system without using any computing
functionality in **Java**. Our program does not contain any numeric operations, and the only places it contains digits is in class and method names (`Peano6`, `iteration3`).
The manufacturers can as well remove digits from keyboards as unnecessary.

Several days later the totally exhausted sysadmin tells you that the attack is over and the system has been restored.
The numbers are back, and we can now run our original `Assignment` program and check the speed.

- `pythagorean`: it reaches 100 in 2 ms, 200 in 21 ms, 1000 in 571 ms, and 10,000 in 435 sec.

- `perfect`: it reaches 1000 in 6 ms, 10,000 in 280 ms, 100,000 in 21 sec.


Our version of `pythagorean` ran 630 times slower than the proper **Java** program, while our version or `perfect` was 1200 times slower -- as if we were running on a
microprocessor with the clock speed of 2 -- 4 MHz. Many of us still remember the days when such processor
speeds were common -- and these processors didn't have modern Intel instruction set. Most were 8-bit and most didn't have multiplication. We have just gone
on a journey 30 -- 40 years back in time. And back then the students also wrote assignments and some of them passed them.
 

The summary
-----------

Here is the list of the versions we developed:

- Peano1: The original version, the strict implementation of Peano set model using `HashSet`;
- Peano2: Added remembering and re-using the result os `S()`;
- Peano3: Added a separate class with identity hash code;
- Peano4: Added remembering and re-using the result os `D()`;
- Peano5: Removed the sets and used the linked lists;
- Peano6: Replaced iteration with recursion;
- Peano6.2: Replaced comparisons for _less_ with equality comparisons;
- Peano6.3: Moved invariant code out of loop in `Pythagorean`; replacing general division with a divisibility check in `Perfect`;
- Peano6.4: Replaced addition with subtraction to create more invariant code to be moved out of loop in `Pythagorean`; running the inner loop to half the limit
in `Perfect`;
- Peano6.5: Reduction of operation strength (replacing multiplications with additions) in `Pythagorean`;
- Peano6.6: Removing one subtraction by means of incorporating it into another induction variable in `Pythagorean`;
- Standard: The implementation using usual **Java** numbers, with no optimisations.

Here are the results for `Pythagorean`, in seconds:

<table class="numeric">
<tr><th>Version</th><th>First output</th><th>100</th><th>200</th><th>500</th><th>1000</th><th>Max speedup</th></tr>
<tr><td>Peano1  </td><td> 12.2 </td><td></td><td></td><td></td><td></td><td></td></tr>
<tr><td>Peano2  </td><td> 1    </td><td></td><td></td><td></td><td></td><td>12.2</td></tr>
<tr><td>Peano3  </td><td> 0.003</td><td>55 in 782 s</td><td></td><td></td><td></td><td>333</td></tr>
<tr><td>Peano4  </td><td> 0.001</td><td>78    </td>     <td>Out of memory at 113</td><td></td><td></td><td>3</td></tr>
<tr><td>Peano5  </td><td> 0.001</td><td> 9    </td>     <td>321 </td>                <td>Stack overflow</td><td></td><td>8.7</td></tr>
<tr><td>Peano6  </td><td> 0.001</td><td> 7.2  </td>     <td>203 </td>                <td>466 in 14000s </td><td></td><td>1.6</td></tr>
<tr><td>Peano6.2</td><td> 0.001</td><td> 7.2  </td>     <td>202 </td>                <td></td><td></td><td>1</td></tr>
<tr><td>Peano6.3</td><td> 0.001</td><td> 2.1  </td>     <td> 58 </td>                <td>5594</td>          <td></td><td>3.48</td></tr>
<tr><td>Peano6.4</td><td> 0.001</td><td> 0.77 </td>     <td> 20 </td>                <td>1944</td>          <td>62259</td><td>2.9</td></tr>
<tr><td>Peano6.5</td><td>     0</td><td> 0.095</td>     <td>1.28</td>                <td>  45</td>          <td>  727</td><td>85</td></tr>
<tr><td>Peano6.6</td><td>     0</td><td> 0.070</td>     <td>0.73</td>                <td>22.3</td>          <td>  360</td><td>2</td></tr>
<tr><td>Standard</td><td>     0</td><td> 0.002</td>     <td>0.021</td>               <td>0.093</td>         <td> 0.571</td><td>630</td></tr>
</table>

And here are the results for `Perfect`, in seconds:

<table class="numeric">
<tr><th>Version</th><th>First number (6)</th><th>Second number (28)</th><th>100</th><th>1000</th><th>5,000</th><th>10,000</th><th>Max speedup</th></tr>
<tr><td>Peano1  </td><td>0.004</td><td>4820</td><td></td><td></td><td></td><td></td><td></td></tr>
<tr><td>Peano2  </td><td>0.002</td><td>4382</td><td></td><td></td><td></td><td></td><td>2</td></tr>
<tr><td>Peano3  </td><td>0.002</td><td>0.016</td><td>  288</td><td>3572 </td><td></td><td></td><td>274000</td></tr>
<tr><td>Peano4  </td><td>0.001</td><td>0.006</td><td>0.027</td><td>2.5  </td><td>598</td><td>6317 </td><td>10666</td></tr>
<tr><td>Peano5  </td><td>0.002</td><td>0.007</td><td>0.018</td><td>3.8  </td><td>571</td><td>5216 </td><td>1.5</td></tr>
<tr><td>Peano6  </td><td>0.001</td><td>0.004</td><td>0.016</td><td>3.2  </td><td>460</td><td>4154 </td><td>1.25</td></tr>
<tr><td>Peano6.2</td><td>0.001</td><td>0.004</td><td>0.016</td><td>3.0  </td><td>412</td><td>3874 </td><td>1.07</td></tr>
<tr><td>Peano6.3</td><td>0.001</td><td>0.003</td><td>0.013</td><td>0.8  </td><td> 81</td><td>648  </td><td>6</td></tr>
<tr><td>Peano6.4</td><td>0.001</td><td>0.003</td><td>0.011</td><td>0.48 </td><td> 41</td><td>336  </td><td>1.9</td></tr>
<tr><td>Standard</td><td>    0</td><td>    0</td><td>0.001</td><td>0.006</td><td>0.074</td><td>0.280</td><td>1200</td></tr>
</table>


The last column is the maximal observed speedup for the version (the timing of the previous version divided by the timing of the current one).
We see some very high values there. For instance, the value 274,000 indicates that the time to print the second perfect number (28) went down from 2,382 seconds
to 0.016 seconds between versions `Peano2` and `Peano3`. Such was the effect of the identity hash code.

If we want to calculate the total speedup, we must agree on the numbers we measure at. Here is one approach. We'll take the interval from our assignment, which required
to check all the numbers from 1 to 100. Here the results are shocking. Initially just the time required to print these 100 numbers was 7&times;10<sup>15</sup>
years, and now our entire programs finished in 70 and 11 milliseconds respectively. This gives us the speedups of 2.8&times;10<sup>21</sup> and 1.8&times;10<sup>22</sup> --
this is the best optimisation effort I've seen in a long time.

Conclusions
-----------

- It is possible to re-create natural numbers using **Java**'s standard library.

- A direct implementation of Peano set model is very, very inefficient, but it works.

- As in [the study of Life]({{ site.ART-HASH }}), we encountered the problem of poor hash codes. Here comes the lesson: whenever you use hash tables -- check your hash codes!

- Good old linked lists work better.

- The standard optimisations (common subexpression elimination, loop-invariant code motion, and operation strength reduction) add a lot of value. Don't forget to include them
  next time you write a compiler.

- This entire exercise does not have practical value (yet), but it can serve as a test of a programming language and its library. Besides, it is fun.

- As often happens, the Anti-Numeric League achieved exactly the opposite to what it intended. Instead of finishing our assignment in milliseconds,
  the computer worked very hard for hours, running our inefficient code.

- If the League really wants to give computers rest, it should not limit itself to numerical operations; it must disable sets and lists as well.

Coming soon
-----------

Our arithmetic represented the numbers as linear-sized structures, which requires linear space and linear or square time. Some other representation could allow us
to do better than that. We'll try it soon.
