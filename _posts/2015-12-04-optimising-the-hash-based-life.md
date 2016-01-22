---
layout: post
title:  "Optimising the hash-based Life"
date:   2015-12-04 12:00:00
tags: Java optimisation hashmap life
story: life
story-title: "Conway's Game of Life"
---

In ["{{ site.TITLE-HASH }}"]({{ site.ART-HASH }}) we implemented [Conway's game of Life](http://en.wikipedia.org/wiki/Conway's_Game_of_Life) using **Java**'s `HashMap`.
We spent a lot of time (seven articles) optimising hash functions and choosing the best one. The best candidate so far is the function that divides a `long` value by a big
prime number and returns a remainder; this is the function we'll be using now. However, the multiplication-based functions also featured quite well, so we mustn't
forget about them.

Until now we haven't touched the main algorithm. Now it's time to do it.

The algorithm
-------------

The whole program is [in the repository]({{ site.REPO-LIFE }}/tree/fc67c70d0f43e2890273b2d37d5312a6f3a15165).
The main class is called `Life`, and the actual Life implementation is called `Hash_LongPoint`. The code is short, so we can quote it here fully:

{% highlight Java %}
public abstract class LongPoint
{
    public static abstract class Factory
    {
        public abstract LongPoint create (long v);

        public LongPoint create (int x, int y)
        {
            return create (fromPoint (x, y));
        }
    }
    
    final long v;
    
    protected LongPoint (long v)
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
    public String toString ()
    {
        return toPoint().toString ();
    }
}

final class Hash_LongPoint extends Worker
{
    public static final int HASH_SIZE = 8192;

    private final LongPoint.Factory factory;
    private HashSet<LongPoint> field = new HashSet<LongPoint> (HASH_SIZE);
    private HashMap<LongPoint, Integer> counts
            = new HashMap<LongPoint, Integer> (HASH_SIZE);
    
    public Hash_LongPoint (LongPoint.Factory factory)
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

The classes like `LongUtil` and `Point` are not shown here, since they are trivial.

The `LongPoint` class represents a pair of co-ordinates `x` and `y` stored in one `long` value (32 bits for each). This representation allows moving from a cell to its
neighbours by adding a single offset (see `set()` and `reset()`). Various classes derived from `LongPoint` contain different `hashCode()` methods. The one we are going to use now
is called `LongPoint6`, where this method looks like this:

{% highlight Java %}
@Override
public int hashCode ()
{
    return (int) (v % 946840871);
}
{% endhighlight %}
  
The Life implementation maintains two hash-based structures: a hash set `field` that contains `LongPoint` objects for all the live cells, and a hash map `counts` that keeps
number of live neighbours for all the cells where this number is non-zero. Technically it is implemented as a `HashMap` mapping from `LongPoint` to `Integer`. Both structures
are updated when new cells are added or existing cells are removed. Each simulation step (the method `step()`) analyses `field` and `counts`, identifies cells that must be set
and reset (that is, where live cells must be created or destroyed). It uses temporary lists `toSet` and `toReset` to delay  performing the actions, because it needs the discovery
process to run on unmodified structures.

Since `Integer` is an immutable class, we can't increment the values in the `counts` map when adding a live neighbour. We must allocate a new object with a new value.

We test this code by running 10,000 iterations starting with the configuration called [`ACORN`](http://conwaylife.appspot.com/pattern/acorn).
Here are the execution times, in milliseconds:

<table class="numeric">
<tr><th>Class name</th>                   <th>Comment</th>              <th>Time, <b>Java&nbsp;7</b></th><th>Time, <b>Java&nbsp;8</b></th></tr>
<tr><td class="label">Hash_LongPoint</td>  <td class="ttext">Original version           </td><td>  2032</td><td> 1650</td></tr>
</table>

Going mutable
-------------

The first idea is to replace `Integer` in `counts` structure with something mutable. Since there isn't any readily-made mutable version of `Integer`, we'll have to create
[our own one called `Count`]({{ site.REPO-LIFE }}/blob/6cb7e1e6e751df04aaab0cdfb718c2f01c1d2e3c/Count.java). We'll also move some utility methods into it:

{% highlight Java %}
public final class Count
{
    public int value;
    
    public Count (int value)
    {
        this.value = value;
    }
    
    public void inc ()
    {
        ++ value;
    }
    
    public boolean dec ()
    {
        return --value != 0;
    }
}
{% endhighlight %}

The new Life implementation is called `Hash1` (see [here]({{ site.REPO-LIFE }}/blob/6cb7e1e6e751df04aaab0cdfb718c2f01c1d2e3c/Hash1.java)).
The first thing we need is a new definition for `counts`:

{% highlight Java %}
private HashMap<LongPoint, Count> counts
  = new HashMap<LongPoint, Count> (HASH_SIZE);
{% endhighlight %}

Then we must modify `inc()` and `dec()`:

{% highlight Java %}
private void inc (long w)
{
    LongPoint key = factory.create (w);
    Count c = counts.get (key);
    if (c == null) {
        counts.put (key, new Count (1));
    } else {
        c.inc ();
    }
}

private void dec (long w)
{
    LongPoint key = factory.create (w);
    if (! counts.get (key).dec ()) {
        counts.remove (key);
    }
}
{% endhighlight %}

Unlike `Integer`, the new class `Count` does not support automatic boxing and unboxing, that's why `step()` must be changed as well:

{% highlight Java %}
public void step ()
{
    ArrayList<LongPoint> toReset = new ArrayList<LongPoint> ();
    ArrayList<LongPoint> toSet = new ArrayList<LongPoint> ();
    for (LongPoint w : field) {
        Count c = counts.get (w);
        if (c == null || c.value < 2 || c.value > 3) toReset.add (w);
    }
    for (LongPoint w : counts.keySet ()) {
        if (counts.get (w).value == 3 && ! field.contains (w)) {
            toSet.add (w);
        }
    }
    for (LongPoint w : toSet) {
        set (w);
    }
    for (LongPoint w : toReset) {
        reset (w);
    }
}
{% endhighlight %}

This is the new execution time:

<table class="numeric">
<tr><th>Class name</th>                   <th>Comment</th>              <th>Time, <b>Java&nbsp;7</b></th><th>Time, <b>Java&nbsp;8</b></th></tr>
<tr><td class="label">Hash_LongPoint</td> <td class="ttext">Original version                  </td><td> 2032</td><td> 1650</td></tr>
<tr><td class="label">Hash1</td>          <td class="ttext">Mutable Count                     </td><td> 1546</td><td> 1437</td></tr>
</table>

This is not bad, especially on **Java 7**.

Merging the hash maps
---------------------

Why do we have two hash maps (a hash set is internally a hash map) in the first place? The primary motivation for this decision was the clarity of code. Besides,
keeping counts separately from the information on live cells allowed using standard Java class `Integer` as the value type of a hash map. As we have introduced
our own class, nothing prevents us from adding to it whatever other fields we consider useful. It would have been beneficial to keep two hash maps separately if
we only ever iterated the smaller one of them (`field`). We, however, iterate both, which is another argument in favour of merging them and iterating both at once.
Let's introduce [a new class, called `Value`]({{ site.REPO-LIFE }}/blob/da3d0c678097ffe828448c6504baf39f8859973f/Value.java),
which will contain both neighbour information and the status of the cell:

{% highlight Java %}
public final class Value
{
    public int count;
    public boolean live;
    
    public Value (int value, boolean present)
    {
        this.count = value;
        this.live = present;
    }

    public Value (int value)
    {
        this (value, false);
    }
    
    public void inc ()
    {
        ++ count;
    }
    
    public boolean dec ()
    {
        return --count != 0;
    }
    
    public void set ()
    {
        live = true;
    }

    public void reset ()
    {
        live = false;
    }
}
{% endhighlight %}

Note that both live cells with zero live neighbours, and dead cells with live neighbours occur in real life, so there is no redundancy in the fields. Care should be taken
to remove cells only when `count == 0 && !live`. Let's make [a new implementation, `Hash2`]({{ site.REPO-LIFE }}/blob/da3d0c678097ffe828448c6504baf39f8859973f/Hash2.java).
First, we replace two hash structures with one:

{% highlight Java %}
private HashMap<LongPoint, Value> field
  = new HashMap<LongPoint, Value> (HASH_SIZE);
{% endhighlight %}

The methods `inc()` and `dec()` must now work with the new class:

{% highlight Java %}
private void inc (long w)
{
    LongPoint key = factory.create (w);
    Value c = field.get (key);
    if (c == null) {
        field.put (key, new Value (1));
    } else {
        c.inc ();
    }
}

private void dec (long w)
{
    LongPoint key = factory.create (w);
    Value v = field.get (key);
    if (! v.dec () && ! v.live) {
        field.remove (key);
    }
}
{% endhighlight %}

`set()` and `reset()` must also change: modifying of the live status of a cell now requires a couple of extra lines:

{% highlight Java %}
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
    Value c = field.get (k);
    if (c == null) {
        field.put (k,  new Value (0, true));
    } else {
        c.live = true;
    }
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
    Value v = field.get (k);
    if (v.count == 0) {
        field.remove (k);
    } else {
        v.live = false;
    }
}
{% endhighlight %}

Finally, the main iteration function must only iterate one structure:

{% highlight Java %}
public void step ()
{
    ArrayList<LongPoint> toReset = new ArrayList<LongPoint> ();
    ArrayList<LongPoint> toSet = new ArrayList<LongPoint> ();
    for (LongPoint w : field.keySet ()) {
        Value c = field.get (w);
        if (c.live) {
            if (c.count < 2 || c.count > 3) toReset.add (w);
        } else {
            if (c.count == 3) toSet.add (w);
        }
    }
    for (LongPoint w : toSet) {
        set (w);
    }
    for (LongPoint w : toReset) {
        reset (w);
    }
}
{% endhighlight %}

Here are the results:

<table class="numeric">
<tr><th>Class name</th>                   <th>Comment</th>              <th>Time, <b>Java&nbsp;7</b></th><th>Time, <b>Java&nbsp;8</b></th></tr>
<tr><td class="label">Hash1</td>          <td class="ttext">Mutable Count                     </td><td> 1546</td><td> 1437</td></tr>
<tr><td class="label">Hash2</td>          <td class="ttext">A unified hash map                </td><td> 1309</td><td> 1163</td></tr>
</table>

This change has also resulted in a good speed improvement.

A minor improvement: unnecessary check
--------------------------------------

The `set()` method contains a null check:

{% highlight Java %}
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
    Value c = field.get (k);
    if (c == null) {
        field.put (k,  new Value (0, true));
    } else {
        c.live = true;
    }
}
{% endhighlight %}

This check is unnecessary during iterations: `set()` is always called on a cell that is present in the `field` structure (its neighbour count is 3).
We can remove this check (see [`Hash3`]({{ site.REPO-LIFE }}/blob/7b79fc5c66e23164a57e55ad380d33f39aa9a6bb/Hash3.java)):

{% highlight Java %}
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
    Value c = field.get (k);
    c.live = true;
}
{% endhighlight %}

This check, however, is needed during initial population of the field. We'll have to modify the initial population routine accordingly, from

{% highlight Java %}
public void put (int x, int y)
{
    set (factory.create (x, y));
}
{% endhighlight %}

to

{% highlight Java %}
public void put (int x, int y)
{
    LongPoint k = factory.create (x, y);
    if (! field.containsKey (k)) {
        field.put (k, new Value (0));
    }
    set (k);
}
{% endhighlight %}

<table class="numeric">
<tr><th>Class name</th>                   <th>Comment</th>              <th>Time, <b>Java&nbsp;7</b></th><th>Time, <b>Java&nbsp;8</b></th></tr>
<tr><td class="label">Hash2</td>          <td class="ttext">A unified hash map                </td><td> 1309</td><td> 1163</td></tr>
<tr><td class="label">Hash3</td>          <td class="ttext">Check in <code>set</code> removed </td><td> 1312</td><td> 1151</td></tr>
</table>

This hasn't achieved much, but it was, after all, a minor change (unlike the two previous ones).

Another minor improvement: entry iterator
-----------------------------------------

This fragment in `step()`

{% highlight Java %}
    for (LongPoint w : field.keySet ()) {
        Value c = field.get (w);
{% endhighlight %}

is inefficient: we are looking up in the hash map the key value that has just been extracted from the same hash map during iteration. This can be easily eliminated
(see [`Hash4`]({{ site.REPO-LIFE }}/blob/e406657efb9260d4ac6e10ff37e656de794adce4/Hash4.java)):

{% highlight Java %}
public void step ()
{
    ArrayList<LongPoint> toReset = new ArrayList<LongPoint> ();
    ArrayList<LongPoint> toSet = new ArrayList<LongPoint> ();
    for (Entry<LongPoint, Value> w : field.entrySet ()) {
        Value c = w.getValue ();
        if (c.live) {
            if (c.count < 2 || c.count > 3) toReset.add (w.getKey ());
        } else {
            if (c.count == 3) toSet.add (w.getKey ());
        }
    }
    for (LongPoint w : toSet) {
        set (w);
    }
    for (LongPoint w : toReset) {
        reset (w);
    }
}
{% endhighlight %}

<table class="numeric">
<tr><th>Class name</th>                   <th>Comment</th>              <th>Time, <b>Java&nbsp;7</b></th><th>Time, <b>Java&nbsp;8</b></th></tr>
<tr><td class="label">Hash3</td>          <td class="ttext">Check in <code>set</code> removed </td><td> 1312</td><td> 1151</td></tr>
<tr><td class="label">Hash4</td>          <td class="ttext"><code>Entry</code> iterator in the loop       </td><td> 1044</td><td>  976</td></tr>
</table>

Another big improvement.

More use of `HashMap.Entry` objects
---------------------------------

Both `set()` and `reset()` use their `LongPoint` parameter to look up corresponding cell in the `field` structure. This is unnecessary, since the cell is already known:
after all, the reason for call to `set()` or `reset()` is that the program has established some fact about the cell (variable `c` in `step()`). The  reference to this cell
is lost, because we populate `toSet` and `toReset` with the key values. It would be nice to put both keys and values there, and pass both as parameters to `set()` and `reset()`.
This is easy to achieve: all we need is declare `toReset` and `toSet` as `ArrayList<Entry<LongPoint, Value>>`
(see [`Hash5`]({{ site.REPO-LIFE }}/blob/4819ad053b8ef7b86163e48522bff99c052b3468/Hash5.java)):

{% highlight Java %}
void set (LongPoint k, Value v)
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
    v.live = true;
}
    
void reset (LongPoint k, Value v)
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
    if (v.count == 0) {
        field.remove (k);
    } else {
        v.live = false;
    }
}

public void step ()
{
    ArrayList<Entry<LongPoint, Value>> toReset
              = new ArrayList<Entry<LongPoint, Value>> ();
    ArrayList<Entry<LongPoint, Value>> toSet
              = new ArrayList<Entry<LongPoint, Value>> ();

    for (Entry<LongPoint, Value> w : field.entrySet ()) {
        Value c = w.getValue ();
        if (c.live) {
            if (c.count < 2 || c.count > 3) toReset.add (w);
        } else {
            if (c.count == 3) toSet.add (w);
        }
    }
    for (Entry<LongPoint, Value> w : toSet) {
        set (w.getKey (), w.getValue ());
    }
    for (Entry<LongPoint, Value> w : toReset) {
        reset (w.getKey (), w.getValue ());
    }
}
{% endhighlight %}

The initial population routine must be changed as well:

{% highlight Java %}
@Override
public void put (int x, int y)
{
    LongPoint k = factory.create (x, y);
    Value v = field.get (k);
    if (v == null) {
        field.put (k, v = new Value (0));
    }
    set (k, v);
}
{% endhighlight %}

<table class="numeric">
<tr><th>Class name</th>                   <th>Comment</th>              <th>Time, <b>Java&nbsp;7</b></th><th>Time, <b>Java&nbsp;8</b></th></tr>
<tr><td class="label">Hash4</td>          <td class="ttext"><code>Entry</code> iterator in the loop       </td><td> 1044</td><td>  976</td></tr>
<tr><td class="label">Hash5</td>          <td class="ttext"><code>ArrayList&lt;Entry&gt;</code> introduced      </td><td> 1006</td><td>  975</td></tr>
</table>

Strangely, the improvement isn't great; I expected much better results.

Re-using the `ArrayList`
----------------------

Two new `ArrayList` objects are allocated each time `step` is executed. This on its own isn't too bad: object allocation is fast in **Java**, and `step()` is only
executed 10,000 times. However, there is additional cost: management of the array that backs the `ArrayList`. This array is allocated small (size 10), then it is expanded as
the `ArrayList` grows. The rule for expanding is

{% highlight Java %}
int newCapacity = oldCapacity + (oldCapacity >> 1);
{% endhighlight %}

This means that the array will be allocated of size 10, 15, 22, 33, 49, 73, 109, 163, 244 and so on -- nine allocations for a list size of 250, which is typical for our
program. While staying with `ArrayList` class, we can't avoid checking for sufficient capacity, but can avoid re-allocation. All we need is re-use the arrays:
we put the following in the beginning of the new class ([`Hash6`]({{ site.REPO-LIFE }}/blob/dbfa6e85160d122e9d86be85ed7285937cac491e/Hash6.java)):

{% highlight Java %}
private final ArrayList<Entry<LongPoint, Value>> toReset
        = new ArrayList<Entry<LongPoint, Value>> ();
private final ArrayList<Entry<LongPoint, Value>> toSet
        = new ArrayList<Entry<LongPoint, Value>> ();
{% endhighlight %}

Instead of allocating an array, it is sufficient to clear it:

{% highlight Java %}
public void step ()
{
    toReset.clear ();
    toSet.clear ();
    for (Entry<LongPoint, Value> w : field.entrySet ()) {
    ...
{% endhighlight %}

<table class="numeric">
<tr><th>Class name</th>                   <th>Comment</th>              <th>Time, <b>Java&nbsp;7</b></th><th>Time, <b>Java&nbsp;8</b></th></tr>
<tr><td class="label">Hash5</td>          <td class="ttext"><code>ArrayList&lt;Entry&gt;</code> introduced      </td><td> 1006</td><td>  975</td></tr>
<tr><td class="label">Hash6</td>          <td class="ttext">Re-used <code>ArrayList&lt;Entry&gt;</code>         </td><td> 1097</td><td>  947</td></tr>
</table>

Strangely, the improvement only happened on **Java 8**, and even there it isn't big. Does it mean that object allocation isn't such a big expense?

`ArrayList` vs an array
-----------------------

What if we replace `ArrayList` objects with simply arrays? If we allocate these arrays of right size, we can save on resize checks when we fill the array. We will also save
on the iterator when we iterate it. This is not the recommended way to program in **Java**, since it causes us to write code that duplicates the code of the **Java**
standard library. We will still do it to check if it provides any speed advantage.

Let's define two arrays of appropriate types and of some reasonable size (see [`Hash7`]({{ site.REPO-LIFE }}/blob/0a84db39335b49aa6c83543c1afdce6ba8d491c1/Hash7.java)):

{% highlight Java %}
private Entry<LongPoint, Value> [] toReset = new Entry [128];
private Entry<LongPoint, Value> [] toSet = new Entry [128];
{% endhighlight %}

`javac` gives warning on these lines:

    Hash7.java:17: warning: [unchecked] unchecked conversion
        private Entry<LongPoint, Value> [] toReset = new Entry [128];
                                                     ^
      required: Entry<LongPoint,Value>[]
      found:    Entry[]

I don't know of any way to get rid of this warning (except `@SupressWarnings`), so we'll have to live with it.

In the beginning of `step()` we must make sure both arrays are of sufficient size. We are not short of memory. Why not allocate it of size `field.size()`, or, to prevent
frequent resizing, even bigger?

{% highlight Java %}
public void step ()
{
    if (field.size () > toSet.length) {
        toReset = new Entry [field.size () * 2];
        toSet = new Entry [field.size () * 2];
    }
    int setCount = 0;
    int resetCount = 0;
        
    for (Entry<LongPoint, Value> w : field.entrySet ()) {
        Value c = w.getValue ();
        if (c.live) {
            if (c.count < 2 || c.count > 3) toReset[resetCount ++] = w;
        } else {
            if (c.count == 3) toSet[setCount ++] = w;
        }
    }
    for (int i = 0; i < setCount; i++) {
        set (toSet[i].getKey (), toSet[i].getValue ());
        toSet[i] = null;
    }
    for (int i = 0; i < resetCount; i++) {
        reset (toReset[i].getKey (), toReset[i].getValue ());
        toReset[i] = null;
    }
}
{% endhighlight %}

Note that we set used elements of the arrays to `null`. This is done to improve garbage collection. If we don't clean the arrays, some elements at the high indices may
stay alive and contaminate memory forever. We can, however, clean the arrays now and then rather than each time, but I don't think this cleanup introduces a significant overhead.

Here are the results:

<table class="numeric">
<tr><th>Class name</th>                   <th>Comment</th><th>Time, <b>Java&nbsp;7</b></th><th>Time, <b>Java&nbsp;8</b></th></tr>
<tr><td class="label">Hash6</td>          <td class="ttext">Re-used <code>ArrayList&lt;Entry&gt;</code>         </td><td> 1097</td><td>  947</td></tr>
<tr><td class="label">Hash7</td>          <td class="ttext"><code>ArrayList</code> replaced with an array </td><td> 1020</td><td>  904</td></tr>
</table>

There is some improvement, although not as big as we expected.

The overall results
-------------------

Here are the results for all the versions we tried today:

<table class="numeric">
<tr><th>Class name</th>                   <th>Comment</th><th>Time, <b>Java&nbsp;7</b></th><th>Time, <b>Java&nbsp;8</b></th></tr>
<tr><td class="label">Hash_LongPoint</td> <td class="ttext">Original version                              </td><td> 2032</td><td> 1650</td></tr>
<tr><td class="label">Hash1</td>          <td class="ttext">Mutable Count                                 </td><td> 1546</td><td> 1437</td></tr>
<tr><td class="label">Hash2</td>          <td class="ttext">A unified hash map                            </td><td> 1309</td><td> 1163</td></tr>
<tr><td class="label">Hash3</td>          <td class="ttext">Check in <code>set</code> removed             </td><td> 1312</td><td> 1151</td></tr>
<tr><td class="label">Hash4</td>          <td class="ttext"><code>Entry</code> iterator in the loop       </td><td> 1044</td><td>  976</td></tr>
<tr><td class="label">Hash5</td>          <td class="ttext"><code>ArrayList&lt;Entry&gt;</code> introduced      </td><td> 1006</td><td>  975</td></tr>
<tr><td class="label">Hash6</td>          <td class="ttext">Re-used <code>ArrayList&lt;Entry&gt;</code>         </td><td> 1097</td><td>  947</td></tr>
<tr><td class="label">Hash7</td>          <td class="ttext"><code>ArrayList</code> replaced with an array </td><td> 1020</td><td>  904</td></tr>
</table>

The best results were achieved by:

- introducing mutable counts (24% on **Java 7**; 13% on **Java 8**)

- unifying two hash maps (15% on **Java 7**; 19% on **Java 8**)

- iterating over the set of `Entry` objects (20% on **Java 7**; 15% on **Java 8**)

It is interesting that the main idea of one of these changes (the mutable counts) is
"replace standard **Java** classes with something custom made for the job", one (unifying the maps) is very close to it ("use less of the standard classes"),
and only the `Entry` iterator change is based on much more moderate idea "Use **Java** standard classes optimally".

Here are the results in the graphic form:

<img src="{{ site.url }}/images/life-optimise.png" width="656" height="357">

The overall improvement is quite good, it was 50% for **Java 7** and 45% for **Java 8**, which means that we've made the program almost twice as fast.
    
Some statistics
---------------

Our improved implementation uses much less hash map operations than the initial ones. Here are the stats:

<table class="numeric">
<tr><th rowspan="2">Operation</th><th colspan="2">Old counts</th><th>New counts</th></tr>
<tr><th><code>field</code></th><th><code>counts</code></th><th><code>field</code></th></tr>
<tr><td class="ttext"><code>put()</code>, new      </td><td>1,292,359</td><td>  2,481,224</td><td>  2,470,436</td></tr>
<tr><td class="ttext"><code>put()</code>, update   </td><td>        0</td><td> 15,713,009</td><td>          0</td></tr>
<tr><td class="ttext"><code>get()</code>, success  </td><td>1,708,139</td><td> 48,514,853</td><td> 18,202,363</td></tr>
<tr><td class="ttext"><code>get()</code>, fail     </td><td>1,292,359</td><td>  2,497,339</td><td>  2,470,436</td></tr>
<tr><td class="ttext"><code>remove()</code>        </td><td>1,291,733</td><td>  2,478,503</td><td>  2,467,681</td></tr>
<tr><td class="ttext">All operations               </td><td>5,584,590</td><td> 71,684,928</td><td> 25,610,916</td></tr>
<tr><td class="ttext"><code>hashCode()</code>      </td><td colspan="2">77,269,518</td>        <td>25,610,916</td></tr>
</table>

The total number of times `hashCode()` is called dropped by two thirds.

Other hash functions
--------------------

Since the total number of performed hash map operations has dropped, the fraction of time spent in these operations may have changed, and this may have caused
different relative performance of the hash functions. Let's run the `Hash7` version with some of the previously tested hash functions (excluding very slow ones).
Here are the old and the new results:

<table class="numeric">
<tr><th rowspan="2">Class name</th>   <th rowspan="2">Comment</th>                             <th colspan="2"><code>Hash_LongPoint</code></th><th colspan="2"><code>Hash7</code></tr>
                                                                                               <th>Java&nbsp;7</th><th>Java&nbsp;8</th><th>Java&nbsp;7</th><th>Java&nbsp;8</th></tr>
<tr><td class="label">LongPoint1</td> <td class="ttext">Multiply by 3, 5                  </td><td> 2719</td><td> 4809</td><td> 1344</td><td> 2361</td></tr>
<tr><td class="label">LongPoint3</td> <td class="ttext">Multiply by 11, 17                </td><td> 2257</td><td> 1928</td><td> 1139</td><td> 1109</td></tr>
<tr><td class="label">LongPoint4</td> <td class="ttext">Multiply by two big primes        </td><td> 2128</td><td> 1611</td><td> 1160</td><td>  932</td></tr>
<tr><td class="label">LongPoint5</td> <td class="ttext">Multiply by one big prime         </td><td> 2150</td><td> 1744</td><td> 1189</td><td>  914</td></tr>
<tr><td class="label">LongPoint6</td> <td class="ttext">Modulo big prime                  </td><td> 2032</td><td> 1650</td><td> 1020</td><td>  904</td></tr>
<tr><td class="label">LongPoint75</td><td class="ttext">CRC32, written in <b>Java</b>     </td><td> 3522</td><td> 2664</td><td> 1641</td><td> 1308</td></tr>
<tr><td class="label">LongPoint77</td><td class="ttext">CRC32C, written in <b>C</b> using SSE, one JNI call </td><td> 2409</td><td>2045</td><td> 1254</td><td> 1072</td></tr>
</table>

The relative performance stayed the same. The functions that caused poor performance are still causing poor performance. The division-based function (`LongPoint6`) is still
the best, two multiplication-based ones (`LongPoint4` and `LongPoint5`) being very close.

Conclusions
-----------

- Our initial implementation could be improved a lot; by 50% on **Java 7** and by 45% on **Java 8**.

- The biggest improvement was caused by replacing of standard classes with the hand-made ones and by using the standard classes more efficiently.

- The choice of the hash function is still important.

Coming soon
-----------

This seems to be the limit: it looks as if there is nothing to improve in our program. However, there are some more ideas. We'll start by looking at the hash table size;
that will be the topic of the next article.
