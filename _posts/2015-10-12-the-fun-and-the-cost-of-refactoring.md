---
layout: post
title:  "The fun and the cost of refactoring"
date:   2015-10-12 12:00:00
tags: Java optimisation hashmap life refactoring design-patterns
story: life
story-title: "Conway's Game of Life"
---

In ["{{ site.TITLE-HASH }}"]({{ site.ART-HASH }}) we implemented [Conway's game of Life](http://en.wikipedia.org/wiki/Conway's_Game_of_Life)
using **Java**'s `HashMap`. This implementation stored Life's field as a hash map with cell co-ordinates as keys.
We tried several classes to represent those co-ordinates, mainly differing from each other by the hash function.

Each such class required its own implementation of Life algorithm. This is very unfortunate, because it causes a lot
of code duplication, and requires additional effort introducing new hash functions. Since then (in ["{{ site.TITLE-DIVISION-HASH }}"]({{ site.ART-DIVISION-HASH }}))
we added two of them, and more are coming. I got tired of this duplication effort, so I want to investigate
what can be done here.

In other words, we are now embarking on a process that most managers hate but most programmers love: refactoring. Most programmers
consider refactoring the ultimate fun and are willing to spend their entire time doing it. Because of that it is important to make sure the process
eventually stops. In real life this happens due to natural factors, such as a manager, a deadline and a year-end bonus, but in experiments like mine I have
to rely on self-discipline.

We must also keep in mind that this process is usually the opposite to the process of optimisation. The programs become shorter and clearer, but hardly faster.
One should make sure the cost in performance loss isn't too high.

The code
--------

The entire code is in [this repository]({{ site.REPO-LIFE }}/tree/d59b6b9a681d60ff011a6b800f8d5f63851eeb83). Here are the
examples of the classes: one key class and one Life implementation based on it.

Here is the key class, [`LongPoint3`]({{ site.REPO-LIFE }}/blob/d59b6b9a681d60ff011a6b800f8d5f63851eeb83/LongPoint3.java):

{% highlight Java %}
final class LongPoint3 extends LongUtil
{
    final long v;
    
    public LongPoint3 (long v)
    {
        this.v = v;
    }

    public LongPoint3 (int x, int y)
    {
        this.v = fromPoint (x, y);
    }
    
    public Point toPoint ()
    {
        return toPoint (v);
    }
    
    @Override
    public boolean equals (Object v2)
    {
        return ((LongPoint3) v2).v == v;
    }
    
    @Override
    public int hashCode ()
    {
        return hi(v) * 11 + lo(v) * 17;
    }
    
    @Override
    public String toString ()
    {
        return toPoint().toString ();
    }
}
{% endhighlight %}

And here is the corresponding Life implementation, [`Hash_LongPoint3`]({{ site.REPO-LIFE }}/blob/d59b6b9a681d60ff011a6b800f8d5f63851eeb83/Hash_LongPoint3.java):

{% highlight Java %}
import java.util.ArrayList;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Set;

final class Hash_LongPoint3 extends LongUtil implements Worker
{
    public static final int HASH_SIZE = 8192;

    private HashSet<LongPoint3> field = new HashSet<LongPoint3> (HASH_SIZE);
    private HashMap<LongPoint3, Integer> counts = new HashMap<LongPoint3, Integer> (HASH_SIZE);
    
    @Override
    public Set<Point> get ()
    {
        HashSet<Point> result = new HashSet<Point> ();
        for (LongPoint3 p : field) {
            result.add (p.toPoint ());
        }
        return result;
    }
    
    private void inc (long w)
    {
        LongPoint3 key = new LongPoint3 (w);
        Integer c = counts.get (key);
        counts.put (key, c == null ? 1 : c+1);
    }

    private void dec (long w)
    {
        LongPoint3 key = new LongPoint3 (w);
        int c = counts.get (key)-1;
        if (c != 0) {
            counts.put (key, c);
        } else {
            counts.remove (key);
        }
    }
    
    void set (LongPoint3 k)
    {
        long w = k.v;
        inc (w-DX-DY);
        inc (w-DX);
        inc (w-DX+DY);
        inc (w-DY);
        inc (w+DY);
        inc (w+DX-DY);
        inc (w+DX);
        inc (w+DX+DY);
        field.add (k);
    }
    
    void reset (LongPoint3 k)
    {
        long w = k.v;
        dec (w-DX-DY);
        dec (w-DX);
        dec (w-DX+DY);
        dec (w-DY);
        dec (w+DY);
        dec (w+DX-DY);
        dec (w+DX);
        dec (w+DX+DY);
        field.remove (k);
    }
    
    @Override
    public void put (int x, int y)
    {
        set (new LongPoint3 (x, y));
    }
    
    @Override
    public void step ()
    {
        ArrayList<LongPoint3> toReset = new ArrayList<LongPoint3> ();
        ArrayList<LongPoint3> toSet = new ArrayList<LongPoint3> ();
        for (LongPoint3 w : field) {
            Integer c = counts.get (w);
            if (c == null || c < 2 || c > 3) toReset.add (w);
        }
        for (LongPoint3 w : counts.keySet ()) {
            if (counts.get (w) == 3 && ! field.contains (w)) toSet.add (w);
        }
        for (LongPoint3 w : toSet) {
            set (w);
        }
        for (LongPoint3 w : toReset) {
            reset (w);
        }
    }
}
{% endhighlight %}

Here [`Point`]({{ site.REPO-LIFE }}/blob/d59b6b9a681d60ff011a6b800f8d5f63851eeb83/Point.java)
is the original (coming from the reference implementation) class, representing the co-ordinates as a pair of integers,
and [`LongUtil`]({{ site.REPO-LIFE }}/blob/d59b6b9a681d60ff011a6b800f8d5f63851eeb83/LongUtil.java) is a collection of
useful methods to operate on `long` values -- split, combine, convert to `Point`, etc.

Why duplicate?
--------------

Each class `LongPointX` is accompanied by its own Life implementation `Hash_LongPointX`. Why? It seems that
encapsulation, which is part of a classic object-oriented design, should make it unnecessary.
If the main program only uses methods from some public interface, it can work with objects of any classes that implement it.
However, one operation can't be part of object interface: the object creation. Since we need to create objects
in the main program (we have three instances of `new LongPointX()` there), a simplistic object-oriented approach
won't work.

The **Java** generics won't help either, for the same reason: they can't create objects of the parameter classes.
This isn't valid **Java**:

{% highlight Java %}
class A<T>
{
    public T create ()
    {
        return new T ();
    }
}
{% endhighlight %}

Note that similar code is perfectly valid in **C++**:

{% highlight C++ %}
template<class T> class A
{
    public:

    T create ()
    {
        return new T ();
    }
}
{% endhighlight %}

In **C++** we could make a `LongPointX` class a template parameter and instantiate `Hash_LongPoint` with appropriate
values of these parameters. It would create specialised versions of `Hash_LongPoint`. Perhaps, I will try it later.
Unfortunately, this is not possible in **Java**.

A factory
---------

There are two simple ways to resolve this problem in **Java**. In one we add a method "create new instance" to each
`LongPointX` class. Then we start the whole simulation with creating one dummy object, and then make sure we have
existing object around each time we create a new one.

Another way it to use factories. A factory is a companion class whose job is
to create objects of our class. Just one instance of a factory is necessary; it can be kept inside main Life
implementation.

Both solutions introduce a virtual call where a new object must be created. Both introduce some extra complexity
to a program. That's why in [the original article]({{ site.ART-LIFE }}) I wrote:

<ul><em>We could work that around by introducing
factory classes, but this would complicate our code and hardly improve performance.
</em></ul>

However, a need to get rid of
multiple implementations of Life is stronger. Besides, it's interesting to test this statement.

Even though the factory-based solution seems a bit more complicated, I still prefer it to the one with "create new instance"
method, because it clearly separates the functionality. One class is responsible for creation, another one for operations.
We can even make `LongPointX` constructors `private` or `protected` to make sure that the factory is the only way to create these objects.

We'll keep our `Reference` and `HashLong` implementations unchanged, but make a unified `Hash_LongPoint` one.
This implementation will operate on objects of `LongPoint` class, which will be the base class of all our `LongPointX`
classes. Some functionality (such as `toPoint()`, `equals()`, `toString()` and the value `v` itself) can now
be implemented in the base class. The derived classes must only provide specific methods - in our case
hash functions.

First, we need an abstract class `LongPointFactory`:

{% highlight Java %}
public abstract class LongPointFactory
{
    public abstract LongPoint create (long v);

    public LongPoint create (int x, int y)
    {
        return create (fromPoint (x, y));
    }
}
{% endhighlight %}

Next, the `LongPoint` class is modified like this:

{% highlight Java %}
class LongPoint
{
    public static class Factory extends LongPointFactory
    {
        @Override
        public LongPoint create (long v)
        {
            return new LongPoint (v);
        }
    }
    
    final long v;
    
    public LongPoint (long v)
    {
        this.v = v;
    }

    public LongPoint (int x, int y)
    {
        this.v = fromPoint (x, y);
    }
    
    public Point toPoint ()
    {
        return LongUtil.toPoint (v);
    }
    
    @Override
    public boolean equals (Object v2)
    {
        return ((LongPoint) v2).v == v;
    }
    
    @Override
    public int hashCode ()
    {
        return hi(v) * 3 + lo(v) * 5;
    }
    
    @Override
    public String toString ()
    {
        return toPoint().toString ();
    }
}
{% endhighlight %}

And this is what happens to the derived class:

{% highlight Java %}
final class LongPoint3 extends LongPoint
{
    public static class Factory extends LongPoint.Factory
    {
        @Override
        public LongPoint3 create (long v)
        {
            return new LongPoint3 (v);
        }
    }
    
    public LongPoint3 (long v)
    {
        super (v);
    }

    @Override
    public int hashCode ()
    {
        return hi(v) * 11 + lo(v) * 17;
    }
}
{% endhighlight %}

The object creation in the main implementation must be replaced by calls to a factory object:

{% highlight Java %}
final class Hash_LongPoint extends Worker
{
    public static final int HASH_SIZE = 8192;

    private final LongPointFactory factory;
    private HashSet<LongPoint> field = new HashSet<LongPoint> (HASH_SIZE);
    private HashMap<LongPoint, Integer> counts =
        new HashMap<LongPoint, Integer> (HASH_SIZE);
    
    public Hash_LongPoint (LongPointFactory factory)
    {
        this.factory = factory;
    }

    @Override
    public String getName ()
    {
        return factory.create (0).getClass ().getName ();
    }

    @Override
    public void reset ()
    {
        field.clear ();
        counts.clear ();
    }
    
    @Override
    public Set<Point> get ()
    {
        HashSet<Point> result = new HashSet<Point> ();
        for (LongPoint p : field) {
            result.add (p.toPoint ());
        }
        return result;
    }
    
    private void inc (long w)
    {
        LongPoint key = factory.create (w);
        Integer c = counts.get (key);
        counts.put (key, c == null ? 1 : c+1);
    }

    private void dec (long w)
    {
        LongPoint key = factory.create (w);
        int c = counts.get (key)-1;
        if (c != 0) {
            counts.put (key, c);
        } else {
            counts.remove (key);
        }
    }
    
    void set (LongPoint k)
    {
        long w = k.v;
        inc (w-DX-DY);
        inc (w-DX);
        inc (w-DX+DY);
        inc (w-DY);
        inc (w+DY);
        inc (w+DX-DY);
        inc (w+DX);
        inc (w+DX+DY);
        field.add (k);
    }
    
    void reset (LongPoint k)
    {
        long w = k.v;
        dec (w-DX-DY);
        dec (w-DX);
        dec (w-DX+DY);
        dec (w-DY);
        dec (w+DY);
        dec (w+DX-DY);
        dec (w+DX);
        dec (w+DX+DY);
        field.remove (k);
    }
    
    @Override
    public void put (int x, int y)
    {
        set (factory.create (x, y));
    }
    
    @Override
    public void step ()
    {
        ArrayList<LongPoint> toReset = new ArrayList<LongPoint> ();
        ArrayList<LongPoint> toSet = new ArrayList<LongPoint> ();
        for (LongPoint w : field) {
            Integer c = counts.get (w);
            if (c == null || c < 2 || c > 3) toReset.add (w);
        }
        for (LongPoint w : counts.keySet ()) {
            if (counts.get (w) == 3 && ! field.contains (w)) toSet.add (w);
        }
        for (LongPoint w : toSet) {
            set (w);
        }
        for (LongPoint w : toReset) {
            reset (w);
        }
    }
}
{% endhighlight %}

Finally, the main class can now run

{% highlight Java %}
        test (new Hash_LongPoint (new LongPoint3.Factory ()));
{% endhighlight %}

where `test()` is a test method that creates a Life structure, runs a given number of iterations and measures time.

Some notes:

- As before, we have two classes for each `LongPoint` implementation -- a class itself and another class accompanying
it. However, the classes are now smaller and much more code is re-used instead of being copy-pasted.

- The code isn't very ugly after all; it is slightly more complex since it introduces additional concept (a factory),
but it is much better than endless replication of `Hash_LongPoint` class.

- The abstract factory class could be made an interface; I prefer using abstract classes wherever possible. I know that
this is not a recommended **Java** way, but they usually perform better, and this is an optimisation exercise after all.
Besides, our abstract class has one concrete method (`create (int x, int y)`), and methods inside interfaces have only
been introduced in **Java 8**.

- The factory classes do not have to be placed right inside the main classes; they could be made top-level.
It is a matter of personal style.

- The `LongPoint.Factory` could fulfil the role or `LongPointFactory`, then we would have one class less.
However, I prefer current design, because it is safer: it forces the derived classes to implement abstract methods.
If `LongPoint.Factory` was used as the base class, some derived class could by accident inherit its implementation of
`create()` (for that it's enough to misspell the method name or change the signature, and omit the `@Override` annotation)..

Some final touches
------------------

I had to make some more modifications of the code. They are not so important and rather technical; so if the reader is not
interested, I recommend skipping this chapter and going straight to test results.

The main test routine (in the class [`Life`]({{ site.REPO-LIFE }}/blob/d59b6b9a681d60ff011a6b800f8d5f63851eeb83/Life.java))  previously looked like this:

{% highlight Java %}
    private static void measure (Class<? extends Worker> cw) throws Exception
    {
        int K = 10000;
        System.out.printf ("%20s: time for %5d:", cw.getName (), K);
        
        long t = 0;
        for (int n = 0; n < 3; n++) {
            Worker w = cw.newInstance ();
            put (w, ACORN);
            long t1 = System.currentTimeMillis ();
            for (int i = 0; i < K; i++) {
                w.step ();
            }
            long t2 = System.currentTimeMillis ();
            t = t2 - t1;
            System.out.printf (" %5d", t);
        }
        System.out.printf (": %6.1f frames/sec\n", K * 1000.0 / t);
    }

    private static void test (Worker w) throws Exception
    {
        test (new Hash_Reference(), w, 100);
        measure (w.getClass ());
    }
{% endhighlight %}

The Life implementation was passed as a class object rather than the instance object because we wanted clean fresh
object, not the one that has just gone through the test. We can't do it anymore, since some implementations are
now parameterised with a factory, and some others are not. We'll have to use the implementation object itself
and add `reset()` method to its interface, which returns the object to its original state (cleans the field). We
could use the factory technique here again, but that looks like an overkill.

We also can't use the class name as the test output label, because most of the time it will be `Hash_LongPoint`. This
means we need another method, called `getName()`, in the `Worker` interface. In the parameterised class it will return
the parameter class name.

And finally, the last change. Previously, all `LongPointX` and `Hash_LongPointX` classes extended `LongUtil` to get
unqualified access to handy utility methods such as `hi()`, `lo()`, and `toPoint()`. We can't do it anymore,
because our `LongPointX` classes now extend `LongPoint`, and we can't extend two classes in **Java**.

One can argue, however, that extending a class for a sole purpose of using its static members as utility functions
is a bad idea anyway. This is similar to defining constants in interfaces -- a practice called
[Constant Interface Antipattern](https://en.wikipedia.org/wiki/Constant_interface). This practice is discouraged
by **Java** designers. Here is what the [**Java** documentation](https://docs.oracle.com/javase/1.5.0/docs/guide/language/static-import.html) says:

<ul><em>
The problem is that a class's use of the static members of another class is a mere implementation detail.
When a class implements an interface, it becomes part of the class's public API.
Implementation details should not leak into public APIs.
</em></ul>

The Wikipedia article referenced above offers some more arguments against this practice, which are fully applicable
to the static methods in the abstract classes.

Fortunately, **Java** (since version 1.5) has an `import static` construct that allows direct import of static methods
and fields from any class without need to inherit this class. There is just one catch: the class mustn't be in the default package.
In real life this is not a problem, since no one is using default
packages in real life. However, I consider the default package appropriate for the test samples such as this Life project.
Anyway, the `LongUtil` has to leave the default package and go somewhere else (`util` package in our case). The `Point`
class goes there, too.

Finally, since `Hash_LongPoint` does not extend `LongUtil`, `Worker` does not have to be an interface anymore; it can
become an abstract class. This allows it to contain a default implementation of `getName()`, which works for non-`LongPoint`
implementations.

The results
-----------

[Here is the new version of the code]({{ site.REPO-LIFE }}/tree/6d1eaef66b329e1eee8528bcf11e6da4d944738a). It does not look too complex, so the first part of my previous statement is probably
not correct. And here are the execution times, measures on **Java 7**

<table class="numeric">
<tr><th>Class name</th>                <th>Comment</th>                                    <th>Time before</th><th>Time after</th><th>change</th></tr>
<tr><td class="label">Point</td>       <td class="ttext">Multiply by 3, 5                  </td><td> 2743</td><td> 2728</td><td> -0.55% </td></tr>
<tr><td class="label">Long</td>        <td class="ttext"><code>Long</code> default         </td><td> 4273</td><td> 4117</td><td> -3.65% </td></tr>
<tr><td class="label">LongPoint</td>   <td class="ttext">Multiply by 3, 5                  </td><td> 2602</td><td> 2719</td><td> +4.50% </td></tr>
<tr><td class="label">LongPoint3</td>  <td class="ttext">Multiply by 11, 17                </td><td> 2074</td><td> 2257</td><td> +8.82% </td></tr>
<tr><td class="label">LongPoint4</td>  <td class="ttext">Multiply by two big primes        </td><td> 2000</td><td> 2128</td><td> +6.40% </td></tr>
<tr><td class="label">LongPoint5</td>  <td class="ttext">Multiply by one big prime         </td><td> 1979</td><td> 2150</td><td> +8.64% </td></tr>
<tr><td class="label">LongPoint6</td>  <td class="ttext">Modulo big prime                  </td><td> 1890</td><td> 2032</td><td> +7.51% </td></tr>
<tr><td class="label">LongPoint60</td> <td class="ttext">Modulo optimised, unsigned        </td><td> 2124</td><td> 2247</td><td> +5.79% </td></tr>
<tr><td class="label">LongPoint61</td> <td class="ttext">Modulo optimised, signed          </td><td> 2198</td><td> 2368</td><td> +7.73% </td></tr>
<tr><td class="label">LongPoint7</td>  <td class="ttext"><code>java.util.zip.CRC32</code>  </td><td>10115</td><td>10337</td><td> +2.19% </td></tr>
</table>

and on **Java 8**

<table class="numeric">
<tr><th>Class name</th>                <th>Comment</th>                                    <th>Time before</th><th>Time after</th><th>change</th></tr>
<tr><td class="label">Point</td>       <td class="ttext">Multiply by 3, 5                  </td><td> 4961</td><td> 5049</td><td>  +1.77% </td></tr>
<tr><td class="label">Long</td>        <td class="ttext"><code>Long</code> default         </td><td> 6755</td><td> 5486</td><td> -18.79% </td></tr>
<tr><td class="label">LongPoint</td>   <td class="ttext">Multiply by 3, 5                  </td><td> 4836</td><td> 4809</td><td>  -0.56% </td></tr>
<tr><td class="label">LongPoint3</td>  <td class="ttext">Multiply by 11, 17                </td><td> 2028</td><td> 1928</td><td>  -4.93% </td></tr>
<tr><td class="label">LongPoint4</td>  <td class="ttext">Multiply by two big primes        </td><td> 1585</td><td> 1611</td><td>  +1.64% </td></tr>
<tr><td class="label">LongPoint5</td>  <td class="ttext">Multiply by one big prime         </td><td> 1553</td><td> 1744</td><td> +12.30% </td></tr>
<tr><td class="label">LongPoint6</td>  <td class="ttext">Modulo big prime                  </td><td> 1550</td><td> 1650</td><td>  +6.45% </td></tr>
<tr><td class="label">LongPoint60</td> <td class="ttext">Modulo optimised, unsigned        </td><td> 1608</td><td> 1877</td><td> +16.72% </td></tr>
<tr><td class="label">LongPoint61</td> <td class="ttext">Modulo optimised, signed          </td><td> 1689</td><td> 1885</td><td> +11.60% </td></tr>
<tr><td class="label">LongPoint7</td>  <td class="ttext"><code>java.util.zip.CRC32</code>  </td><td> 3206</td><td> 3254</td><td>  +1.48% </td></tr>
</table>

We expected that an extra virtual call would slow execution down, and, in most cases, it did. There are cases when the speed didn't change much or when it
became higher (in one case by as much as 18%), but most common outcome is a 5-15% slowdown. I consider this a fair price for getting rid of code
duplication.

The alternatives
----------------

In our new code a virtual call is made for each object creation. We can reduce the number of virtual calls by increasing their size, for example,
if we create more than one object in each one. If we make virtual methods even bigger, we can reduce the number of calls more, the ultimate
case being our original code where there were no virtual calls at all (except those required by `HashMap`). However, each increase in virtual
method size means extra code duplication. Somewhere between two extreme cases - "All the code is duplicated" and "All the code is generic; the lowest-level
methods are virtual" lies a sweet spot where performance is good and code duplication is still manageable. I want to show two versions that lie between these extremes
and check if they are better than our refactored version.

In our first version, which we'll call "`Neighbours`" we'll have a virtual call that creates eight objects instead of one. We'll revive
`neighbours()` method that existed in `Point` class. The full code is in [its own branch]({{ site.REPO-LIFE }}/tree/Neighbours).
This is what we add to [`LongPoint`]({{ site.REPO-LIFE }}/blob/eb8bf3bf9809f6761eba891c0bea01b804ceb49c/LongPoint.java):

{% highlight Java %}
    public LongPoint[] neighbours ()
    {
        return new LongPoint[] {
                            new LongPoint (v-DX-DY),
                            new LongPoint (v-DX),
                            new LongPoint (v-DX+DY),
                            new LongPoint (v-DY),
                            new LongPoint (v+DY),
                            new LongPoint (v+DX-DY),
                            new LongPoint (v+DX),
                            new LongPoint (v+DX+DY)
        };
    }
{% endhighlight %}

We'll override this method in all the other `LongPoint` classes. Here is the code from [`LongPoint3`]({{ site.REPO-LIFE }}/blob/eb8bf3bf9809f6761eba891c0bea01b804ceb49c/LongPoint3.java):

{% highlight Java %}
    @Override
    public LongPoint[] neighbours ()
    {
        return new LongPoint[] {
                            new LongPoint3 (v-DX-DY),
                            new LongPoint3 (v-DX),
                            new LongPoint3 (v-DX+DY),
                            new LongPoint3 (v-DY),
                            new LongPoint3 (v+DY),
                            new LongPoint3 (v+DX-DY),
                            new LongPoint3 (v+DX),
                            new LongPoint3 (v+DX+DY)
        };
    }
{% endhighlight %}

The factory class and `create()` method is still there -- we need it to initialise the field.

The `set()`, `reset()`, `inc()` and `dec()` methods in [`Hash_LongPoint`]({{ site.REPO-LIFE }}/blob/eb8bf3bf9809f6761eba891c0bea01b804ceb49c/Hash_LongPoint.java) will look like this:
    
{% highlight Java %}
    private void inc (LongPoint key)
    {
        Integer c = counts.get (key);
        counts.put (key, c == null ? 1 : c+1);
    }

    private void dec (LongPoint key)
    {
        int c = counts.get (key)-1;
        if (c != 0) {
            counts.put (key, c);
        } else {
            counts.remove (key);
        }
    }
    
    void set (LongPoint k)
    {
        for (LongPoint p : k.neighbours ()) {
            inc (p);
        }
        field.add (k);
    }
    
    void reset (LongPoint k)
    {
        for (LongPoint p : k.neighbours ()) {
            dec (p);
        }
        field.remove (k);
    }
{% endhighlight %}

Since we don't allocate objects in `inc()` and `dec()` anymore, we had to change their signatures to accept `LongPoint` rather than `long`.

Another alternative version (called `SetReset` and stored in [its own branch]({{ site.REPO-LIFE }}/tree/SetReset)) goes further and pulls `set()` and `reset()` into `LongPoint` class as well, eliminating the need to allocate,
fill and iterate a neighbour array. This is what [`LongPoint`]({{ site.REPO-LIFE }}/38382eebc6528a480391ea1cfe110edad5464567/LongPoint.java) looks like in this version:

{% highlight Java %}
    protected void inc (HashMap<LongPoint, Integer> counts, LongPoint key)
    {
        Integer c = counts.get (key);
        counts.put (key, c == null ? 1 : c+1);
    }

    protected void dec (HashMap<LongPoint, Integer> counts, LongPoint key)
    {
        int c = counts.get (key)-1;
        if (c != 0) {
            counts.put (key, c);
        } else {
            counts.remove (key);
        }
    }

    public void inc (HashMap<LongPoint, Integer> counts)
    {
        inc (counts, new LongPoint (v-DX-DY));
        inc (counts, new LongPoint (v-DX));
        inc (counts, new LongPoint (v-DX+DY));
        inc (counts, new LongPoint (v-DY));
        inc (counts, new LongPoint (v+DY));
        inc (counts, new LongPoint (v+DX-DY));
        inc (counts, new LongPoint (v+DX));
        inc (counts, new LongPoint (v+DX+DY));
    }

    public void dec (HashMap<LongPoint, Integer> counts)
    {
        dec (counts, new LongPoint (v-DX-DY));
        dec (counts, new LongPoint (v-DX));
        dec (counts, new LongPoint (v-DX+DY));
        dec (counts, new LongPoint (v-DY));
        dec (counts, new LongPoint (v+DY));
        dec (counts, new LongPoint (v+DX-DY));
        dec (counts, new LongPoint (v+DX));
        dec (counts, new LongPoint (v+DX+DY));
    }
{% endhighlight %}

Two `protected` methods are identical for all `LongPoint` classes and can be re-used; the other two must be overridden in all derived
classes.

The appropriate place from [`Hash_LongPoint`](({{ site.REPO-LIFE }}/38382eebc6528a480391ea1cfe110edad5464567/Hash_LongPoint.java)) will look like this:

{% highlight Java %}
    void set (LongPoint k)
    {
        k.inc (counts);
        field.add (k);
    }
    
    void reset (LongPoint k)
    {
        k.dec (counts);
        field.remove (k);
    }
{% endhighlight %}

I'll only run these versions for two of the versions -- say, `LongPoint4` (multiply by two big primes) and `LongPoint6` (modulo big prime).
Here are the results on **Java 7**:

<table class="numeric">
<tr><th>Class name</th>                <th>Comment</th>                                    <th>Original</th><th>Refactored</th><th>Neighbours</th><th>SetReset</th></tr>
<tr><td class="label">LongPoint4</td>  <td class="ttext">Multiply by two big primes        </td><td> 2000</td><td> 2128</td><td> 2370</td><td> 2090</td></tr>
<tr><td class="label">LongPoint6</td>  <td class="ttext">Modulo big prime                  </td><td> 1890</td><td> 2032</td><td> 2230</td><td> 2080</td></tr>
</table>

and on **Java 8**

<table class="numeric">
<tr><th>Class name</th>                <th>Comment</th>                                    <th>Original</th><th>Refactored</th><th>Neighbours</th><th>SetReset</th></tr>
<tr><td class="label">LongPoint4</td>  <td class="ttext">Multiply by two big primes        </td><td> 1585</td><td> 1611</td><td> 1794</td><td> 1600</td></tr>
<tr><td class="label">LongPoint6</td>  <td class="ttext">Modulo big prime                  </td><td> 1550</td><td> 1650</td><td> 1749</td><td> 1580</td></tr>
</table>

Observations:

- The results are surprisingly consistent. The tests demonstrate the same pattern.

- The `Neighbours` failed expectations and performed badly in all the cases; I'm not sure why and I'm too lazy to investigate. We've replaced eight virtual calls by one
array allocation and some manipulations with this array. Perhaps, **Java** VM was able to optimise those virtual calls well.

- The `SetReset` mostly behaved as expected (between the performance of the original code and that of the refactored one).

- The `SetReset` solution is very ugly, because it draws the boundary between the algorithm (`Hash_LongPoint`) and the data item (`LongPoint`) classes in very
unusual place. Some implementation details of the `Hash_LongPoint` have been pulled inside `LongPoint`, making the overall solution less readable and
more difficult to maintain. It even went as far as leaking the data structure used in the algorithm (a `HashMap`) into `LongPoint`'s public interface. This
is really a bad idea. I'd rather not use classes at all and write everything in plain **C** than use the code like this. Besides, the amount of code
duplication is rather big.

- The `Neighbours` is better, because the `neighbours()` method can qualify as a reasonable part of a public interface. It is
natural to ask a point for a list of its neighbours, and it does not leak implementation details into fundamental classes. However, this method is long,
and it must be written for every new version of `LongPoint` classes, and the primary point of the entire refactoring exercise was to reduce the amount
of code duplication. I would have used the `neighbours()` method if there were very few of `LongPoint` classes, but not in our situation. In addition, this
solution didn't improve performance.

- In short, both solutions have failed, and I won't bother with them anymore.

Abstract classes are good: improving the class hierarchy
--------------------------------------------------------

While experimenting with the alternative versions described above, I initially modified all the `LongPointX` classes, except for `LongPoint7`. I just forgot to include
the required methods. And guess what? The `LongPoint7` version compiled and ran, but didn't work. It produced incorrect results -- fortunately, it was detected by a correctness check.
It was easy to forget to upgrade a class, because there was nothing that prevented us from it.  All the methods were present in the base class.
This is a good point to remember for the future: if some classes differ from each other in the implementation of some method, this method
must be `abstract`. Otherwise, there is danger of accidentally inheriting the base class implementation, which may not be applicable, By the way, the `Object` class doesn't
follow this guideline, for it has default implementations of `hashCode()` and `equals()`.

This means that the best way to lay out our classes is to introduce some base `LongPoint` class, containing `v` field and all the common methods, and to let all the other `LongPoint`
classes, including the original one (which we'll rename as `LongPoint1`) inherit it. Let's do it. We won't change the `Neighbours` and `SetReset`, since I said I wouldn't bother with
them anymore, but let's fix the main solution (branch [master]({{ site.REPO-LIFE }}/commits/master)). We'll also remove one of the `HashTime` classes (the original one)
and fix the remaining one use refactored classes.

[Here is the code]({{ site.REPO-LIFE }}/tree/894485c5f9f691e7b03a1a1515fae4c7aaeba8a3). The speed didn't change.

Why did the correctness check fail when we forgot to modify `LongPoint7`? This class inherited `neighbours()` from `LongPoint`, which returns an array of `LongPoint`.
It would not have made a difference (except for performance), if the classes didn't differ in the `hashCode()` function. Strictly  speaking, our design isn't correct: our
`equals()` method is not in sync with `hashCode()`. **Java** does not allow objects that are `equals` to have different hash codes. The only reason why we still used
this design was that we planned to use only one class representing a point in the full program. Different classes were not supposed to mix, let alone be inserted into
the same hash map. This, however, happened by accident, so we have to watch out for such cases. In this respect our primary refactoring is better than the alternative
solutions -- it requires less code it the derived class.

One more refactoring
--------------------

The factory classes we've created do not require multiple instances. Exactly one instance of each of them is sufficient. Creating more than one identical factory just adds confusion,
so the best form of a factory is a singleton -- the class with exactly one instance. Such a class doesn't need a name - anonymous class would work. This is what it can looks like
(in `LongPoint3`):

{% highlight Java %}
    public static final LongPoint.Factory factory = new LongPoint.Factory ()
    {
        @Override
        public LongPoint3 create (long v)
        {
            return new LongPoint3 (v);
        }
    };
{% endhighlight %}

The code in the main class must then be changed accordingly:

{% highlight Java %}
        test (new Hash_LongPoint (LongPoint6.factory));
{% endhighlight %}

The new code is [here]({{ site.REPO-LIFE }}/tree/d453e6ce16f603fa9c154102ff772285e26e974c). Again, the performance stays the same.

This is where the self-discipline mentioned in the beginning suggests that we stop refactoring.
Not because there is no more improvements (I can't think of any right now, but there are always some), but because we must stop somewhere, and now is a good time.


Conclusions
-----------

- Moderate use of common design patterns can reduce the total code size and the amount of necessary code duplication. The code becomes neater and more maintainable.

- It comes at a cost: overheads of extra virtual calls cause performance penalties. We saw speed reduction by as much as 16% in some cases.

- There is more than one way to refactor a program, and some are better than another.

- Refactoring is a pleasant and potentially endless process. Unless kept under control, it can use up all your time and energy, so it is important to stop somewhere.
  One must make a firm decision "This is good enough".

Coming soon
-----------

Now we are ready to start adding more hash functions. These include new version of modulo- and CRC-based once. I'm also going to run an investigation
of the strange behaviour of the modulo-based  version, which performs poorly as a microbenchmark but well as a full test.

