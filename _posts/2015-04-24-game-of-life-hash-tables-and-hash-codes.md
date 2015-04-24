---
layout: post
title:  "Game of Life, hash tables and hash codes"
date:   2015-04-24 12:00:00
tags: Java optimisation hashmap life
story: life
story-title: "Conway's Game of Life"
---

<script>
$(document).ready(function(){
  function life (canvas, button, shapename) {
    var SX = 64;
    var SY = 64;
    var D = 4;
    canvas.width = SX * D - 1;
    canvas.height = SX * D - 1;
    var field = new Array (SY);
    var field2 = new Array (SY);
    var timer;

    for (var y = 0; y < SY; y++) {
      field [y] = new Array (SX);
      field2 [y] = new Array (SX);
      for (var x = 0; x < SX; x++) {
        field[y][x] = 0;
        field2[y][x] = 2;
      }
    }

    function calc () {
      for (var y = 0; y < SY; y ++) {
        for (var x = 0; x < SX; x ++) {
          var x1 = (x-1) & (SX-1);
          var x2 = (x+1) & (SX-1);
          var y1 = (y-1) & (SY-1);
          var y2 = (y+1) & (SY-1);
          var n = field[y1][x1] + field[y1][x] + field[y1][x2] + field[y][x1] + field[y][x2] + field[y2][x1] + field[y2][x] + field[y2][x2];
          field2[y][x] = (n == 3 || (n == 2 && field[y][x])) ? 1 : 0;
        }
      }
      var t = field2;
      field2 = field;
      field = t;
    }

    function draw_init () {
      var ctx = canvas.getContext ("2d");
      ctx.fillStyle="gray";
      var w = SX*D-1;
      var h = SY*D-1;
      ctx.fillRect (0, 0, w, h);
    }

    function draw_update () {
      var ctx = canvas.getContext ("2d");
      ctx.fillStyle="black";
      for (var y = 0; y < SY; y++) {
        for (var x = 0; x < SX; x++) {
          if (field[y][x]!=field2[y][x]) {
            ctx.fillStyle=field[y][x]?"black":"white";
            ctx.fillRect (x*D, y*D, D-1, D-1);
          }
        }
      }
    }

    function step ()
    {
      calc ();
      draw_update ();
    }

    function start () {
      if (timer) return;
      timer = window.setInterval (step, 100);
      set_stop ();
    }

    function stop () {
      clearInterval (timer);
      timer = null;
      set_start ();
    }

    function set_start ()
    {
      button.text ("Start");
      button.click (start);
    }

    function set_stop ()
    {
      button.text ("Stop");
      button.click (stop);
    }

    function putshape (v) {
      var sx = v[0].length;
      var sy = v.length;
      var x0 = (SX - sx) >> 1;
      var y0 = (SY - sy) >> 1;
      for (var y = 0; y < v.length; y++) {
        for (var x = 0 ; x < v[y].length; x++) {
          if (v[y].charAt(x) != ' ') field [y0+y][x0+x] = 1;
        }
      }
    }

    var shapes = { rpenta : [" * ",
                              " **",
                              "** "],
                   acorn  : ["**  ***",
                             "   *   ",
                             " *     "],
                   gun: ["                        *             ",
                         "                      * *             ",
                         "            **      **            **",
                         "           *   *    **            **",
                         "**        *     *   **",
                         "**        *   * **    * *",
                         "          *     *       *",
                         "           *   *",
                         "            **"]
                 };

    putshape (shapes[shapename]);
    draw_init ();
    draw_update ();
    set_start ();
  }

  $("canvas.life").each (function () {
    var id = $(this).attr ("id");
    var button = $("button#" + id);
    life ($(this)[0], button, $(this).attr("shape"));
  });
});
</script>


Introduction: The Game of Life
------------------------------

You can hardly find a programmer who heard about [Conway's game of Life](http://en.wikipedia.org/wiki/Conway's_Game_of_Life)
and didn't ever try to program it. Google's programmers were no exception, otherwise they wouldn't have included a
Life interpreter into the [search result page](https://www.google.co.za/?gfe_rd=cr&ei=CbwRVaTCCeKr8wfg-oKQDg&gws_rd=ssl#q=conway%27s+game+of+life).
I was no exception either. The first time I tried to do it was on [one ancient computer](http://www.computer-museum.ru/english/minsk_22.php),
which you are unlikely to see even in a museum these days. The processor was so slow and the field was so small that
I didn't even bother to measure performance. Next time it was on a [8-bit 1 MHz processor](http://en.wikipedia.org/wiki/MOS_Technology_6502).
If I remember correctly, an assembly program could process 256&times;256 matrix in 6s. It seemed fast in those days.
Since the processing was matrix-based (it worked right inside the video memory), half the size meant four times the
speed. A 32&times;32 matrix produced a movie with 10 fps. I never tried it on modern machines and/or in **Java**.
The clock speed became 2500 times faster. How much will the speed improve? If the speed improvement keeps up
with the CPU, we should be able to process 256&times;256 at 400 fps. But our processor is 64-bit, so in theory
it could do 3200 fps. Is it possible?

The rules
---------

The Life universe is an infinite two-dimensional square grid, whose squares can be occupied by living cells. Each square in this
grid has eight neighbours, direct and diagonal. The time runs in clock ticks. The following events happen on each tick:

- A number of living neighbours is calculated for each square
- All the cells having less than two living neighbours die of solitude
- All the cells having more than three living neighbours die of overpopulation
- A new cell is born on any empty square that has three living neighbours.

The game received attention from researchers, and a lot of interesting structures have been found. There are static
structures, oscillators, moving patterns ([gliders](http://en.wikipedia.org/wiki/Glider_%28Conway's_Life%29)), and
a lot of more complex objects. Some small patterns can multiply for a while and produce non-trivial results. Here is an example: this simple structure
(R-pentamino piece) starts small, grows, lives a turbulent life and then deteriorates and fades away, never dying
completely:

<canvas class="life" id="R" shape="rpenta" width="255" height="255" style="outline: 1px black solid;"></canvas>
<br/>
<button class="life" id="R">Start</button>
<br/>

You can click "Start" and see how this one evolves.

What about hash tables?
-----------------------

But wait, the article must have something to do with hash tables and hash codes. What do they have to do with Life?

There are two major classes of Life algorithms. The matrix-based ones (such as my previous experiments)
keep a part of the universe as a matrix (possibly considered wrapped-around, as in the examples above).
The entire matrix is re-calculated on each step. These algorithms are very easy to implement but have two disadvantages.
Firstly, they are limited by the size of a matrix, and Life structures are known to be able to grow indefinitely (for
example, to spawn mentioned gliders). This means the programs can only
produce approximate results, such as the result we saw for R-pentamino above. Secondly, most Life colonies contain much fewer cells than the entire
matrix, so it should be much faster to keep track of the occupied cells in some sort of a data structure and only
process those. This is the second class of Life programs. Very sophisticated data structures have been used for
Life simulation, some with extraordinary performance. For instance, the [HashLife](http://en.wikipedia.org/wiki/Hashlife)
approach makes use of identifying repeating patterns both in space and time, with great results.

I want to do something much simpler than HashLife --  use the data structure that is readily available in **Java** -- the `HashMap`
(and its buddy `HashSet`). This has additional advantage that a hash map is a very common
data structure in typical **Java** programs. It will help if we learn something about it in the process.

The timing results, however, won't be directly comparable with my old results, since the algorithms are different.
I'll try the matrix approach later, too.

The test
--------

It is not trivial to find a suitable test input for a Life program. An arbitrary set of cells is very likely
to die very quickly. We've seen one simple structure that lived for a while, the R-pentamino. Here is another one,
the acorn:

<canvas class="life" id="A" shape="acorn" width="255" height="255" style="outline: 1px black solid;"></canvas>
<br/>
<button class="life" id="A">start</button>
<br/>

We'll be using the acorn as it lives a bit longer, and grows bigger -- it stabilises at 633 living cells while
the R-pentamino does at 116.

The plan
--------

The algorithm will be trivial, and the initial implementation will be based directly on the classes provided by **Java** library.
We'll keep locations on the plain as values of class `Point`, encapsulating two co-ordinates, `x` and `y`. The class
will be suited for use as a hash map key (will have `equals()` and `hashCode()` methods).
Then we can keep a field as simply a set of `Point` objects (`HashSet` will do). We'll use `int` values as our
co-ordinates, allowing both positive and negative ones. This means that the field is still finite and will wrap at some
point, so simulations of unbounded structures, such as gliders, will be limited in time (although I'd be very happy
if I could hit this limitation in real life -- that would mean that the program performed several billion iterations in
reasonable time, and this is something yet to be achieved). Another consideration
is memory. There are configurations that grow indefinitely, such as [Gosper Glider Gun](http://en.wikipedia.org/wiki/Gun_%28cellular_automaton%29):

<canvas class="life" id="gun" shape="gun" width="255" height="255" style="outline: 1px black solid;"></canvas>
<br/>
<button class="life" id="gun">start</button>
<br/>

If we track this gun long enough, we'll run out of memory. However, a million or two iterations should still work.
Perhaps, we'll do the gun next time.

The code
--------

The code can be found [here]({{ site.REPO-LIFE }}).

First of all, we need a `Point` class. Here it is:

{% highlight Java %}
class Point
{
    public final int x, y;
    
    public Point (int x, int y)
    {
        this.x = x;
        this.y = y;
    }
    
    @Override
    public final boolean equals (Object obj)
    {
        Point p = (Point) obj;
        return x == p.x && y == p.y;
    }
    
    @Override
    public final int hashCode ()
    {
        return x * 3 + y * 5;
    }
    
    @Override
    public String toString ()
    {
        return "(" + x + "," + y + ")";
    }

    public Point[] neighbours ()
    {
        return new Point[] {
            new Point (x-1, y-1),
            new Point (x-1, y),
            new Point (x-1, y+1),
            new Point (x, y-1),
            new Point (x, y+1),
            new Point (x+1, y-1),
            new Point (x+1, y),
            new Point (x+1, y+1)
        };
    }
}
{% endhighlight %}

The only not so trivial part of this class is the `hashCode()`. Why such hash function? It was a completely
arbitrary choice. We need something that depends on both numbers in some not-completely-trivial way. Multiplying
by some prime numbers seems good enough. We'll come back to this issue later.

We'll keep a field in a `HashSet`:

{% highlight Java %}
public static final int HASH_SIZE = 8192;
private HashSet<Point> field = new HashSet<Point> (HASH_SIZE);
{% endhighlight %}

Specifying the initial size (8192) avoids spending time growing the table. This isn't very important, it just
simplifies performance comparisons.

We'll also maintain another structure, `counts`:

{% highlight Java %}
private HashMap<Point, Integer> counts =
        new HashMap<Point, Integer> (HASH_SIZE);
{% endhighlight %}

It will keep a number of neighbours for each point (zero assumed for points not included in this map). We'll
update this structure each time we create or destroy a living cell:

{% highlight Java %}
private void inc (Point w)
{
    Integer c = counts.get (w);
    counts.put (w, c == null ? 1 : c + 1);
}

private void dec (Point w)
{
    int c = counts.get (w) - 1;
    if (c != 0) {
        counts.put (w, c);
    } else {
        counts.remove (w);
    }
}

void set (Point w)
{
    for (Point p : w.neighbours ()) {
        inc (p);
    }
    field.add (w);
}

void reset (Point w)
{
    for (Point p : w.neighbours ()) {
        dec (p);
    }
    field.remove (w);
}
{% endhighlight %}

We can immediately see some inefficiencies in the code. For instance, `set()` calls `neighbours()`, which allocates
an array and eight `Point` objects. We can't avoid creating these `Point` objects (we need something as hash map keys),
but could do without array creation. Another example is the `counts` hash map. A code like

{% highlight Java %}
int c = counts.get (w) - 1;
counts.put (w, c);
{% endhighlight %}

completely conceals the fact that what's kept in the map isn't `int` -- it is `Integer`, which is an object. **Java**'s
automatic boxing makes it look like the map keeps primitive values, but in the actual fact it does not. A new object must
be allocated each time the values changes -- this is hardly the most efficient way to keep numbers in the hash table.
But we'll ignore these inefficiencies for now. Let's first see what speed we can achieve without any optimisations.

We still need the main step operation:

{% highlight Java %}
public void step ()
{
    ArrayList<Point> toReset = new ArrayList<Point> ();
    ArrayList<Point> toSet = new ArrayList<Point> ();
    for (Point w : field) {
        Integer c = counts.get (w);
        if (c == null || c < 2 || c > 3) toReset.add (w);
    }
    for (Point w : counts.keySet ()) {
        if (counts.get (w) == 3 && ! field.contains (w)) toSet.add (w);
    }
    for (Point w : toReset) {
        reset (w);
    }
    for (Point w : toSet) {
        set (w);
    }
}
{% endhighlight %}

Methods `set()` and `reset()` modify both `field` and `counts`, so we can't call them inside two main loops. We have
to memorise the cells that must be created and destroyed and perform the operations later. The `reset()` pass
is performed before `set()` in a hope to make is a little bit faster (hash table operations run faster when there are
fewer elements in the table).

There is also usual correctness check code, which isn't needed now but will be important later, and the performance
measurement code. We'll run each performance test three times. Here is the run result:

    # java -server Life
    Hash_Reference: time for 10000:  2726  2196  2199: 4547.5 frames/sec

How good is the result? It is interesting that it is of the same order of magnitude as the 3200 fps I was talking
about in the beginning of the article, even though the algorithms are of different nature, so the results aren't directly comparable.

Going Long
----------

A very obvious optimisation comes to mind when one looks at this code: getting rid of the `Point` class. Why keep two
32-bit values separately if they can be kept in one 64-bit value? This looks especially attractive on a 64-bit
platform, where such values can be operated on using a single CPU instruction. We can immediately see two
advantages: we don't need a special class (built-in `long` type is sufficient), and we need a single CPU instruction
comparing the values instead of two (in fact, three, if you count an extra branch). There is also another, not so obvious
advantage: we can get a neighbouring position by adding a statically known offset to a `long`.

There is a disadvantage, too: the code loses some clarity and type-safety. A `Point` is much more informative than
a `long`. You can't by mistake pass a loop index or current time as a `Point`, but you can as a `long`. A `Point`
gives immediate access to its components (`x` and `y`), while `long` must still be decoded, and it is much easier
to make a mistake there. In short, if we get speed increase (and we're not even sure we do), it comes with a
price tag. This is common case in optimisation.

There are some interesting technical details, too. Here is the code we use to combine two `int` values into one `long`
and split it back:

{% highlight Java %}
public static long w (int hi, int lo)
{
    return ((long) hi << 32) | lo & 0xFFFFFFFFL;
}

public static int hi (long w)
{
    return (int) (w >>> 32);
}

public static int lo (long w)
{
    return (int) w;
}
{% endhighlight %}

The significance of `& 0xFFFFFFFFL` (and of that trailing `L`) is known to every experienced **Java** programmer,
because few of them didn't make a mistake in such code at least once. The `int` values are sign-extended when converting
to `long`. The entire high part of a `long` result will be filled with ones for any negative `lo` if we do not `AND` it
with the appropriate mask; this mask must be of type `long` (this is what `L` is for), otherwise the result of `AND` will
still be of type `int` and will be sign-extended anyway.

We'll keep `x` portion in the high 32 bits of a `long`, and `y` portion in the low one. This makes it easy to
calculate positions of the neighbours. This is what `set()` method looks like using `long`s:

{% highlight Java %}
public static final long DX = 0x100000000L;
public static final long DY = 1;

void set (long w)
{
    inc (w-DX-DY);
    inc (w-DX);
    inc (w-DX+DY);
    inc (w-DY);
    inc (w+DY);
    inc (w+DX-DY);
    inc (w+DX);
    inc (w+DX+DY);
    field.add (w);
}
{% endhighlight %}

These formulas, however, do not tolerate transitions across zero. For instance, a formula for the top left
neighbour is `w-DX-DY`. For location (1,&nbsp;1) it produces correct result (0,&nbsp;0), but for (0,&nbsp;0) it produces

  <div class="formula">
  0&nbsp;&minus;&nbsp;0x10000001&nbsp;=&nbsp;0xFFFFFFFEFFFFFFFF,
  </div>

which is (-2,&nbsp;-1) instead of (-1,&nbsp;-1). To resolve this, we'll add
an offset `0x80000000` to all co-ordinates. This offset is only applied at input or output time, and does not
affect main algorithm:

{% highlight Java %}
public static final int offset = 0x8000000;

public static int x (long w)
{
    return hi (w) - OFFSET;
}

public static int y (long w)
{
    return lo (w) - OFFSET;
}
    
public static long fromPoint (int x, int y)
{
    return w (x + OFFSET, y + OFFSET);
}
{% endhighlight %}

The rest of the program stays virtually unchanged. Here is the result:

    # java -server Life
          Hash_Long: time for 10000:  3878  3248  3240: 3086.4 frames/sec

This is a surprise! The performance got worse. What went wrong? It seems very unlikely that operations with `long`,
even boxed as `Long`, are slower than with our `Point` class. The general structure of the program is the same. What is the difference?

The difference is in the `hashCode()` method. This method in class `Point` looked like this:

{% highlight Java %}
public final int hashCode ()
{
    return x * 3 + y * 5;
}
{% endhighlight %}

And this is the same method from `Long` class:

{% highlight Java %}
public int hashCode() {
    return (int)(value ^ (value >>> 32));
}
{% endhighlight %}

Is this the reason?
-------------------

Could the difference in hash codes have caused such performance difference? This is easy to test. We must write
a version similar to the one based on `long` but with the same hash code as the original one. Extending the `Long`
class would have done the trick, but this is impossible, because that class is `final` (besides, boxing would have created values of
the original class anyway). We need to write our own version of `Long`, with modified hash code. Obviously,
automatic boxing and unboxing will not happen. In fact, this class has very little in common with `Long`,
so it's better to call it a different version of `Point`:

{% highlight Java %}
final class LongPoint extends LongUtil
{
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
        return toPoint (v);
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

A pedantic reader would point out that this class isn't equivalent to the original version, since its `hashCode()` uses `x` and `y`
values with added `OFFSET`. However, this won't affect either the time to compute the hash or the quality of hashing. In
fact, the hash value will be exactly the same: 0x80000000 multiplied by 3 (or 5) as `int` is still 0x80000000,
and the two added together will cancel each other out.

Since there is no more boxing, the code is a little bit longer, but it is also clearer and safer -- arbitrary number
can't be passed as a point any more. Here is the result (I'll quote all three):

     Hash_Reference: time for 10000:  2726  2196  2199: 4547.5 frames/sec
          Hash_Long: time for 10000:  3878  3248  3240: 3086.4 frames/sec
     Hash_LongPoint: time for 10000:  2312  1946  1933: 5173.3 frames/sec

This proves our theory. The version using `long` with the original hash code is much faster than the `Long`-based
version and a little bit faster than the original one.

How good are your hash codes?
-----------------------------

Finally, we arrived at the main topic of the article: the hash codes. Very few programmers fine-tune their hash codes,
usually relying on default implementations, or on those that intuitively seem good. And, as we have just seen, the choice of hash code has significant impact on
performance (50% slowdown in our case). Why?

**Java** uses classic hash map structure, which features an array of certain big enough size, with a direct mapping
of hash codes to positions in this array, called slots. Typically the hash value is divided by the array size and a remainder is
used as an index. This is how the `HashTable` was implemented. In `HashMap` the array size is chosen as a power of two,
so the division operation is saved and replaced with a bitwise `AND` -- the required number of lowest bits is taken.
It would be great to develop a hash function, which, provided that the number of elements is less than the array size,
mapped them all to different array indices. Perhaps, this can be achieved in some special cases, but hardly in our case
(and definitely not in general case). This means that hash collisions must be processed. That's why data elements
are kept in entry objects, several such objects per slot. The simplest, and most common, way to keep these objects
is a linked list, growing from the array elements, but **Java 8** uses binary trees for long chains (above 8 elements).
We'll concentrate on **Java 7** for now (most of findings are also applicable to **Java 8**, since chains of 8 and more
elements are a pathological case).

Each insert operation requires full traversal of the entire list, and so does each unsuccessful lookup. An average successful
lookup requires traversal of half the list. In short, the number of entries looked at is roughly
proportional to the average length of the list, which equals to the number of elements in the hash table
divided by the number of non-empty slots. So we need the hash function to spread values evenly across the array.
Both above hash functions would work well if `x` and `y` values were truly random 32-bit numbers. This is not, however,
our case. Some hash functions are better for our case than others.

The full hash value is kept in each entry. When traversing a chain, the hash map implementation checks it first,
only calling `equals()` on objects if hashes are equal. As a result, a full hash collision (where two hash values are equal)
is worse than a partial collision (where two hashes are different but still fall into the same slot), although both
reduce performance.

**Java 7** uses a technique to reduce the number of partial collisions. The `HashMap` does not directly use the lowest bits of a hash code. It first applies
some additional bit manipulations:

{% highlight Java %}
final int hash(Object k) {
    int h = hashSeed;
    if (0 != h && k instanceof String) {
        return sun.misc.Hashing.stringHash32((String) k);
    }

    h ^= k.hashCode();

    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
{% endhighlight %}

This bit shuffling helps against hash codes where sufficient number of significant bits is available, but these bits
occupy the higher parts of the numbers. This shuffling was removed in **Java 8** -- the binary trees were supposed to
take care of this. However, neither bit shuffling nor binary trees help against full hash conflicts, which originate
from poor quality hash functions that generate too few hash values. And the `XOR`-based hash code of `Long` is exactly like that. It maps the entire
diagonal line (0,0), (1,1), (2,2), ... into the single value 0. It does not work well with other diagonals, either.
Every second hash code for diagonal line `y = x + 1` will be 1, every fourth -- 3, and so on. Patterns in `Life`
game often contain diagonal lines or develop along diagonal lines, so this behaviour is bad for them. Our original hash
function (`x * 3 + y * 5`) behaves much better -- it always maps a diagonal of size 100 into 100 different numbers.

The default hash function also behaves badly with patterns bounded by small squares. Consider a 32&times;32 square block,
where 0 &le; x &le; 31, 0 &le; y &le; 31. Each co-ordinate fits into 5 bits. A `XOR` of two co-ordinates also fits in 5 bits,
which means that the entire block (1024 cells) is mapped to 32 hash codes. It can be a bit better for other blocks
(for instance, a 32&times;32 block positioned at (3, 38) maps into 100 values), but we want it to be as close to 1024
as possible. The patterns in `Life` often fall apart into relatively small components, such as individual gliders or
pulsars, so this is important.

Our original hash function maps every 32&times;32 block into exactly 241 values, which is better, but still very far
from optimal, which suggests that this function is not very good either. There are also cases where it fails miserably.
For instance, it maps the entire line

{% highlight Java %}
y = (int) (x * -3.0/5.0) + y0;
{% endhighlight %}

into just five values. We need proper measurement to choose our hash function.

Measuring hash functions
------------------------

We don't have access to `HashMap` internals. The relevant fields and methods are package-private. The easiest is
to run our own simulation. We'll start with printing the sizes and bounding rectangles as a function of time:

{% highlight Java %}
private static void sizeDump ()
{
    Hash_Reference r = new Hash_Reference ();
    put (r, ACORN);
    
    for (int i = 0; i < 10000; i++) {
        r.step ();
        int xmin = 0, xmax = 0, ymin = 0, ymax = 0;
        for (Point p : r.field) {
            xmin = Math.min (xmin,  p.x);
            xmax = Math.max (xmax,  p.x);
            ymin = Math.min (ymin,  p.y);
            ymax = Math.max (ymax,  p.y);
        }
        System.out.printf
            ("%5d: %5d - %5d; %5d - %5d; field size: %5d; counts size: %5d\n",
             i, xmin, xmax, ymin, ymax, r.field.size (), r.counts.size ());
    }
}
{% endhighlight %}

We had to modify `Hash_Reference` and made `field` and `counts` public.

As we can see, the colony grows at first, then shrinks a bit and then stabilises at
633 living cells and 2755 non-zero neighbours:

<img src="{{ site.url }}/images/life-sizes.png" width="585" height="380">

The maximum is reached roughly at step 4400:

    4400:  -922 ..  1000; -1046 ..  1048; field size:  1034; counts size:  3938

The colony isn't dead after the number stabilises -- the bounding rectangle keeps expanding:

    9997: -2321 ..  2399; -2445 ..  2447; field size:   633; counts size:  2755
    9998: -2321 ..  2400; -2445 ..  2447; field size:   633; counts size:  2755
    9999: -2322 ..  2400; -2445 ..  2448; field size:   633; counts size:  2755

This indicates that the colony isn't dead; there are some gliders flying away.

Let's apply various hash functions at step 4400 and see how many slots in the 8192-sized table are used:

{% highlight Java %}
private static int shuffle (int h)
{
    h ^= (h >>> 20) ^ (h >>> 12);
    h ^= (h >>> 7) ^ (h >>> 4);
    return h & (HASH_SIZE-1);
}
    
private interface Hash
{
    public int hash (Point p);
}

private static void hashtest (Hash h)
{
    Hash_Reference r = new Hash_Reference ();
    put (r, ACORN);
    
    int NSTEPS = 4400;
    
    for (int i = 0; i <= NSTEPS; i++) {
        r.step ();
    }

    BitSet b1 = new BitSet (HASH_SIZE);
    for (Point p : r.field) {
        b1.set (shuffle (h.hash (p)));
    }
    int nonzero_field = b1.cardinality ();
    double avg_field = (double) r.field.size () / nonzero_field;

    BitSet b2 = new BitSet (HASH_SIZE);
    for (Point p : r.counts.keySet ()) {
        b2.set (shuffle (h.hash (p)));
    }
    int nonzero_counts = b2.cardinality ();
    double avg_counts = (double) r.counts.size () / nonzero_counts;

    System.out.printf ("Field size: %4d; slots used: %4d; avg=%5.1f\n",
                       r.field.size (), nonzero_field, avg_field);
    System.out.printf ("Count size: %4d; slots used: %4d; avg=%5.1f\n",
                       r.counts.size (), nonzero_counts, avg_counts);
}
{% endhighlight %}

Now let's try using it with different hashes:

First, the default hash for `Long`:

{% highlight Java %}
hashtest (new Hash () { public int hash (Point p) {
                          return p.x ^ p.y;}} );
{% endhighlight %}

This is where Lambda-definitions of **Java 8** would be handy, but I'm testing it on **Java 7**.

Here is the result:

    Field size: 1034; slots used:  240; avg= 4.31
    Count size: 3938; slots used:  302; avg=13.04

The result really looks bad. List size of 13 is too big. With the table half empty we should be able to do better.

Now let's look at our original `Point` hash:

{% highlight Java %}
hashtest (new Hash () { public int hash (Point p) {
                          return p.x * 3 + p.y * 5;}} );
{% endhighlight %}

    Field size: 1034; slots used:  595; avg= 1.74
    Count size: 3938; slots used: 1108; avg= 3.55

This is much better, so we've got the explanation of poor performance of `Long` compared to `Point`. However, in
absolute terms the result is still bad. Surely, we must be able to spread 3938 entries in more than 1108 slots in a 8192-sized hash table

It looks like multiplying by 3 and 5 does not shuffle the bits well enough.
How about some other prime numbers? Let's try 11 and 17.

{% highlight Java %}
hashtest (new Hash () { public int hash (Point p) {
                          return p.x * 11 + p.y * 17;}} );
{% endhighlight %}

    Field size: 1034; slots used:  885; avg= 1.17
    Count size: 3938; slots used: 2252; avg= 1.75

That's better. How about even bigger numbers?

{% highlight Java %}
hashtest (new Hash () { public int hash (Point p) {
                          return p.x * 1735499 + p.y * 7436369;}} );
{% endhighlight %}

    Field size: 1034; slots used:  972; avg= 1.06
    Count size: 3938; slots used: 3099; avg= 1.27

This already looks reasonable. Let's now try some other approaches.

How about multiplying the entire `long` by some big enough prime number?

{% highlight Java %}
hashtest (new Hash () { public int hash (Point p) {
                          long x = w(p.x, p.y) * 541725397157L;
                          return lo(x) ^ hi(x);}} );
{% endhighlight %}

    Field size: 1034; slots used:  970; avg= 1.07
    Count size: 3938; slots used: 3096; avg= 1.27

One of the methods Knuth (_The art of computer programming, vol. 3, 6.4_) suggests is dividing by some prime number and getting a remainder.
We need a number small enough to distribute remainders evenly, but big enough to generate sufficient number of
these remainders and prevent full hash conflicts.

{% highlight Java %}
hashtest (new Hash () { public int hash (Point p) {
                          long x = w (p.x,  p.y);
                          return (int) (x % 946840871);}} );
{% endhighlight %}

    Field size: 1034; slots used:  978; avg= 1.06
    Count size: 3938; slots used: 3201; avg= 1.23

It performs slightly better. However, division is expensive, so I expect the
division-based hashing to be slower than multiplication-based one.

Knuth also suggested using polynomial binary codes. There is one such code readily available in **Java** library,
it is `CRC32` (in package `java.util.zip`). 

{% highlight Java %}
static int crc_hash (long n)
{
    CRC32 crc = new CRC32 ();
    crc.update ((int)(n >>> 0) & 0xFF);
    crc.update ((int)(n >>> 8) & 0xFF);
    crc.update ((int)(n >>> 16) & 0xFF);
    crc.update ((int)(n >>> 24) & 0xFF);
    crc.update ((int)(n >>> 32) & 0xFF);
    crc.update ((int)(n >>> 40) & 0xFF);
    crc.update ((int)(n >>> 48) & 0xFF);
    crc.update ((int)(n >>> 56) & 0xFF);
    return (int) crc.getValue ();
}

hashtest (new Hash () { public int hash (Point p) {
                          return crc_hash (w(p.x, p.y));}} );
{% endhighlight %}

    Field size: 1034; slots used:  981; avg= 1.05
    Count size: 3938; slots used: 3228; avg= 1.22

We've got some more progress. However, 1.22 still seems a bit high. We want 1.0! Is `CRC32` still not shuffling bits well?
Let's try the ultimate test: random numbers. Obviously, we're not planning of using them as real hash values, we'll
just measure the slots number and chain size:

{% highlight Java %}
final Random r = new Random (0);
hashtest (new Hash () { public int hash (Point p) {
                          return r.nextInt ();}} );
{% endhighlight %}

    Field size: 1034; slots used:  968; avg= 1.07
    Count size: 3938; slots used: 3151; avg= 1.25

Even truly random hash didn't reduce chain size to 1. Moreover, it performed worse than `CRC32`. Why is that?
In fact, this is exactly what we'd expect from random numbers. Truly random selection of a slot will chose already
occupied slot with the same probability as any other slot. The hash function must be rather _non-random_ to spread
all the values to different slots. Let's look at probabilities.

Let's do some calculations
--------------------------

Let's define two random variables. Let <i>Num<sub>k</sub></i> be the number of non-empty slots in the table
after _k_ insertions, and <i>Len<sub>k</sub></i> be the average chain size.
There is an obvious relation between them:

  <div class="formula">
  <table>
  <tr>
    <td rowspan="2">Len<sub>k</sub>&nbsp;=&nbsp;</td>
    <td class="num">k</td>
  </tr>
  <tr>
    <td class="denom">Num<sub>k</sub></td>
  </tr>
  </table>
  </div>

Both variables are good for our purposes. <i>Len<sub>k</sub></i> is directly relevant to the `HasMap` performance, but
<i>Num<sub>k</sub></i> is directly reflecting the quality of the hash function. The latter seems to be easier to
work with, at least we can calculate its mathematical expectation E[<i>Num<sub>k</sub></i>] straight away:

Let _M_ be the table size (_M = 8192_ in our case). The probability of hitting a specific slot while inserting one element is

  <div class="formula">
  <table>
  <tr>
    <td rowspan="3">p&nbsp;=&nbsp;</td>
    <td class="num">1</td>
  </tr>
  <tr>
    <td class="denom">M</td>
  </tr>
  </table>
  </div>

The probability of not hitting a slot is then

  <div class="formula">
  <table>
  <tr>
    <td rowspan="3">q&nbsp;=&nbsp;1&nbsp;&minus;&nbsp;</td>
    <td class="num">1</td>
  </tr>
  <tr>
    <td class="denom">M</td>
  </tr>
  </table>
  </div>

The probability of not hitting a specific slot in _k_ tries is

  <div class="formula">
  <table>
  <tr>
    <td rowspan=3>q<sub>k</sub>&nbsp;=&nbsp;<span class="big">(</span>1&nbsp;&minus;&nbsp;</td>
    <td class="num">1</td>
    <td rowspan=3>&nbsp;<span class="big">)</span></td>
    <td rowspan=3><span class="big"><sup><sup>k</sup></sup></span></td>
  </tr>
  <tr>
    <td class="denom">M</td>
  </tr>
  </table>
  </div>

The probability of hitting a specific slot at least once (in other words, of a slot in the table to be non-empty after _k_ inserts) is

  <div class="formula">
  <table>
  <tr>
    <td rowspan=3>p<sub>k</sub>&nbsp;=&nbsp;1&nbsp;&minus;&nbsp;<span class="big">(</span>1&nbsp;&minus;&nbsp;</td>
    <td class="num">1</td>
    <td rowspan=3>&nbsp;<span class="big">)</span></td>
    <td rowspan=3><span class="big"><sup><sup>k</sup></sup></span></td>
  </tr>
  <tr>
    <td class="denom">M</td>
  </tr>
  </table>
  </div>

and the expected value of the number of non-empty slots (random variable <i>Num<sub>k</sub></i>) is then

  <div class="formula">
  <table>
  <tr>
    <td rowspan=2>E[Num<sub>k</sub>]&nbsp;=&nbsp;M&nbsp;<span class="big">(</span>1&nbsp;&minus;&nbsp;<span class="big">(</span>1&nbsp;&minus;&nbsp;</td>
    <td class="num">1</td>
    <td rowspan=2>&nbsp;<span class="big">)</span></td>
    <td rowspan=2><span class="big"><sup><sup>k</sup></sup></span></td>
    <td rowspan=2><span class="big">)</span></td>
  </tr>
  <tr>
    <td class="denom">M</td>
  </tr>
  </table>
  </div>

For the expected values of numbers of populated slots in `field` and `counts` tables we get:

  <div class="formula">
  <p>  E[Num<sub>1034</sub>]&nbsp;=&nbsp;971.46
  <br> E[Num<sub>3938</sub>)&nbsp;=&nbsp;3126.69
  </p>
  </div>

These numbers correspond to average chain size of 1.064 and 1.259 respectively.

How close are our measured values to these expected ones? In statistics, this is measured is "sigmas", a sigma
(<i>&sigma;</i>) being the standard deviation of a random variable. To calculate it, we need to know the distribution of
the variable in question, in our case, <i>Num<sub>k</sub></i>. This is less trivial than calculating the mean value.

Let <i>N<sub>1</sub>, &hellip;, N<sub>k</sub></i> be independent discrete random variables taking _M_ different values (<i>0 &hellip; M-1</i>)
with equal probabilities. Let <i>P<sub>k</sub>(n)</i> be the probability that these variables take exactly _n_ different values.
We are interested in the values <i>P<sub>k</sub>(n)</i>, for all <i>0&le;n&le;k&nbsp;&and;&nbsp;n&le;M</i>.

First <i>k</i> variables take exactly <i>n</i> different values in two cases:

- if <i>k&minus;1</i> variables took exactly <i>n</i> different values (which happens with probability <i>p<sub>k&minus;1</sub>&nbsp;(n)</i>),
and the <i>k</i><sup>th</sup> variable also took one of these values (the probability is <i>n/M</i>);

- if <i>k&minus;1</i> variables took exactly <i>n&minus;1</i> different values (the probability is <i>p<sub>k&minus;1</sub>&nbsp;(n&minus;1)</i>),
and the <i>k</i><sup>th</sup> variable didn't take one of those values (the probability is <i>(M&minus;n+1)/M</i>).

This gives us following recurrent formula:

  <div class="formula">
  <table>
  <tr>
    <td rowspan="2">P<sub>k</sub>(n)&nbsp;=&nbsp;P<sub>k-1</sub>(n)&nbsp;</td>
    <td class="num">n</td>
    <td rowspan="2">&nbsp;+&nbsp;P<sub>k-1</sub>(n-1)&nbsp;</td>
    <td class="num">M&nbsp;&minus;&nbsp;n&nbsp;+&nbsp;1</td>
  </tr>
  <tr>
    <td class="denom">M</td>
    <td class="denom">M</td>
  </tr>
  </table>
  <br>
    P<sub>0</sub>(0)&nbsp;=&nbsp;1
  </div>

I don't know any easy way to convert it into a direct formula. Knuth (_The art of computer programming, vol. 1,
1.2.10, exercise 9_) gives a solution based on
[Stirling numbers](http://en.wikipedia.org/wiki/Stirling_number):

  <div class="formula">
  <table>
  <tr>
    <td rowspan="2">P<sub>k</sub>(n)&nbsp;=&nbsp;
    <td class="num">M!<sup> </sup></td>
    <td rowspan="2"><span class="big">{</span></td>
    <td>k</td>
    <td rowspan="2"><span class="big">}</span></td>
  </tr>
  <tr>
    <td class="denom">(M&minus;n)!M<sup>k</sup></td>
    <td>n</td>
  </tr>
  </table>
  <br>
    P<sub>0</sub>(0)&nbsp;=&nbsp;1
  </div>

This does not help much, for Stirling numbers are also calculated using recurrent formulas. Fortunately,
our formula is good enough to compute distributions and standard deviations:

  <div class="formula">
  <p> E[Num<sub>1034</sub>]&nbsp;=&nbsp;971.46
  <br>D[Num<sub>1034</sub>]&nbsp;=&nbsp;52.86
  <br>&sigma;[Num<sub>1034</sub>]&nbsp;=&nbsp;7.27
  <br>E[Num<sub>3938</sub>]&nbsp;=&nbsp;3126.69
  <br>D[Num<sub>3938</sub>]&nbsp;=&nbsp;427.57
  <br>&sigma;[Num<sub>3938</sub>]&nbsp;=&nbsp;20.68
  </p>
  </div>

The values for <i>E[Num]</i> are the same as computed before, which inspires confidence in our results.

Let's have a look how far the values measured for various hash functions are from the expected values.
Here are the data for `field` structure (element count 1034) collected in a table. The last column shows the distance, in sigmas, from
expected value (E[Num]&nbsp;=&nbsp;971.46):

<table class="numeric">
<tr><th>Label</th><th>Hash</th><th>Num</th><th>Num&minus;E</th><th>(Num&minus;E)/&sigma;</th></tr>
<tr><td class="label">1</td><td class="ttext"><b>Long</b> default        </td><td>240</td><td>-731.46</td><td>-100.61</td></tr>
<tr><td class="label">2</td><td class="ttext">Multiply by 3, 5           </td><td>595</td><td>-376.46</td><td> -51.78</td></tr>
<tr><td class="label">3</td><td class="ttext">Multiply by 11, 17         </td><td>885</td><td> -86.46</td><td> -11.89</td></tr>
<tr><td class="label">4</td><td class="ttext">Multiply by two big primes </td><td>972</td><td>  +0.54</td><td>  +0.07</td></tr>
<tr><td class="label">5</td><td class="ttext">Multiply by one big prime  </td><td>970</td><td>  -1.46</td><td>  -0.20</td></tr>
<tr><td class="label">6</td><td class="ttext">Modulo big prime           </td><td>978</td><td>  +6.54</td><td>  +0.90</td></tr>
<tr><td class="label">7</td><td class="ttext">CRC32                      </td><td>981</td><td>  +9.54</td><td>  +1.31</td></tr>
<tr><td class="label">8</td><td class="ttext">Random                     </td><td>968</td><td>  -3.46</td><td>  -0.48</td></tr>
</table>

Here are the same points plotted on a graph of the probability density function of the _Num_ random variable (as calculated
using our recurrent formula):

<img src="{{ site.url }}/images/1034.png" width="604" height="455">

The solid vertical line shows the expected value; the thin vertical lines indicate one, two and three sigmas distance
from it in both directions. Red dots correspond to the values from the table above (some were so far away that they didn't fit
into the picture).

Here are the results for the `counts` structure (element count 3938, E&nbsp;[Num]&nbsp;=&nbsp;3126.69), first as a table:

<table class="numeric">
<tr><th>Label<th>Hash</th><th>Num</th><th>Num&minus;E</th><th>(Num&minus;E)/&sigma;</th></tr>
<tr><td class="label">1</td><td class="ttext"><b>Long</b> default               </td><td> 302</td><td>-2824.69</td><td> -136.52</td></tr>
<tr><td class="label">2</td><td class="ttext">Multiply by 3, 5                  </td><td>1108</td><td>-2018.69</td><td>  -97.57</td></tr>
<tr><td class="label">3</td><td class="ttext">Multiply by 11, 17                </td><td>2252</td><td> -901.69</td><td>  -43.58</td></tr>
<tr><td class="label">4</td><td class="ttext">Multiply by two big primes        </td><td>3099</td><td>  -27.69</td><td>   -1.34</td></tr>
<tr><td class="label">5</td><td class="ttext">Multiply by one big prime         </td><td>3096</td><td>  -30.69</td><td>   -1.48</td></tr>
<tr><td class="label">6</td><td class="ttext">Modulo big prime                  </td><td>3201</td><td>  +74.31</td><td>   +3.59</td></tr>
<tr><td class="label">7</td><td class="ttext">CRC32                             </td><td>3228</td><td> +101.31</td><td>   +4.90</td></tr>
<tr><td class="label">8</td><td class="ttext">Random                            </td><td>3151</td><td>  +24.31</td><td>   +1.17</td></tr>
</table>

and as a graph:

<img src="{{ site.url }}/images/3938.png" width="605" height="456">

In statistics, a three sigma distance from the mean value is usually considered a boundary between likely and unlikely.
Normally distributed random variable falls within 3&sigma; from its mean value with probability 99.7%. The
probabilities for 1&sigma; and 2&sigma; are 68.2% and 95.4%. Physicists usually require the difference to be 5&sigma; to be
really sure that they have discovered something new. This was the achieved confidence in Higgs boson search.

In our case we want the opposite -- the result to be close to the mean value. The first three hash functions,
as we see, perform very badly, the `Long` one being the absolute negative champion. No wonder the program ran so slowly.
Our "fast" version, the one multiplying by 3 and 5, however, also turned out to be bad. We'll need to replace it with
something better. All the other functions seem to behave reasonably well, mostly measuring within 1&nbsp;--&nbsp;2&sigma;.
We can see that the random number test gave &minus;0.48&sigma; in one case and 1.17&sigma; in another one, which shows that
such variations are quite achievable by chance.

There are, however, two interesting cases, both for `counts`, where "modulo big prime" and `CRC32` produced exceptionally
good results, both differing from the mean value by over 3&sigma; positively. The point for `CRC32` (number `7`)
didn't even fit into the graph, so far right it is positioned. Why this happens and whether it means that
these two hash functions are in general especially well (above average) suited for our task, I don't know. Perhaps
some mathematically inclined reader can answer this.

The graph of the probability density function of <i>Num<sub>k</sub></i> looks very much like the bell curve of a
[Normal distribution](http://en.wikipedia.org/wiki/Normal_distribution), and if we plot such a curve, the graph will be indeed very close. This suggests that our
distribution asymptotically approaches a Normal distribution. If this is the case, it can simplify calculations,
since instead of our recurring formula, we will be able to use the Gaussian function, which is available in most mathematical
libraries. I'm not sure if this is the case, and, if it is, why. Some statisticians (and most natural scientists)
take it for granted that any unknown distribution is normal by default (and what isn't normal, is logarithmically normal);
others, with whom I tend to agree, look for conditions of some form
of the [Central Limit Theorem](http://en.wikipedia.org/wiki/Central_limit_theorem).
This theorem requires the variable in question to be the appropriately scaled sum of a big number of
independent variables (identically distributed in the simplest forms of the theorem). If such theorem is applicable to
our case and what are these basic variables is another question for a mathematically inclined reader.

Note on the test data
---------------------

It is important that the hash functions are tested on realistic input data. For instance, in our case the entire
colony fits into a bounding rectangle sized about 5000&times;5000, both `x` and `y` being below 2500 in absolute value.
It could affect the behaviour of some hash functions (for instance, it is possible that the exceptionally good
behaviour of a division-based function has something to do with it). It is not a problem in our case, since we want to optimise
our test, but if we want a program to perform well on broader set of inputs, we must check the behaviour of hash
functions on those inputs.

How fast are our hash functions?
--------------------------------

Any of the functions starting from number 4 in the tables above are suitable (obviously, except the random one). Which one is the best,
depends on their own speed -- a well-hashing, but slow function is a bad choice. Let's measure. We'll create
several versions of the `LongPoint` class, their names ending with labels from the table above. Special versions of
`Hash_LongPoint` will also be made, using these `LongPoint`s. We'll test all of them on **Java 7** and **Java 8**.

Various versions of `LongPoint` only differ in hash function, and in theory could be implemented as classes derived
from `LongPoint`. However, the amount of re-used code will be very small and will hardly justify making
that class not final. Various versions of `Hash_LongPoint` only differ in the versions of `LongPoint` they use. They could
have been implemented as one generic class if they didn't need to instantiate `LongPoint` objects. We could work it around
by introducing factory classes, but this would complicate code and hardly improve performance. This is where **C++**
templates work better than **Java** generics (I haven't yet found examples of the opposite).

Here are the results as a table:

<table class="numeric">
<tr><th>Label<th>Hash</th><th>Time, <b>Java 7</b></th><th>Time, <b>Java 8</b></th></tr>
<tr><td class="label">Ref</td><td class="ttext">Multiply by 3, 5                </td><td>2199</td><td>4161</td></tr>
<tr><td class="label">1</td><td class="ttext"><b>Long</b> default               </td><td>3240</td><td>6454</td></tr>
<tr><td class="label">2</td><td class="ttext">Multiply by 3, 5                  </td><td>1933</td><td>2317</td></tr>
<tr><td class="label">3</td><td class="ttext">Multiply by 11, 17                </td><td>1642</td><td>1913</td></tr>
<tr><td class="label">4</td><td class="ttext">Multiply by two big primes        </td><td>1628</td><td>1632</td></tr>
<tr><td class="label">5</td><td class="ttext">Multiply by one big prime         </td><td>1645</td><td>1609</td></tr>
<tr><td class="label">6</td><td class="ttext">Modulo big prime                  </td><td>1567</td><td>1550</td></tr>
<tr><td class="label">7</td><td class="ttext">CRC32                             </td><td>8757</td><td>3234</td></tr>
</table>

And as a graph:

<img src="{{ site.url }}/images/life-results.png" width="576" height="287">

Here are the immediate observations:

- When hash functions are bad, **Java 8** performs worse than **Java 7**. I can only attribute it to them abolishing bit
  shuffling of the hash in favour of optimising the chains (binary threes instead of lists). Perhaps, our case is especially bad for
  such implementation, but I'm not convinced that that was a good idea to begin with.

- The original hash function (multiplying by 3 and 5) is indeed not very good; multiplying by 11 and 17 is better,
  especially in **Java 8**; multiplying by two big numbers is even better on both versions.

- Multiplying by one big prime is so close to multiplying by two big primes that neither has definite advantage.

- This is the biggest surprise: modulo big prime performs best in both versions of **Java**. It was above average on
  the distribution curve (point `6` on the graphs), but the division operation is so slow that I didn't expect
  much; still, it performs well in our case.

- `CRC32` is slow on **Java 8** and very slow on **Java 7**. The cost of calculating this function seem to defeat
  all its benefits regarding the quality of hashing, However, there are ways to optimise this function; we'll look
  at them in the next article.

Conclusions
-----------

- Quality of hash functions is important. For applications with heavy usage of hash tables the correct choice of
  hash functions can be the single most significant factor determining the performance.

- The default hash function of `Long` isn't friendly towards packed values. If several values are kept in one
  `long`, and this `long` is often used as a hash key, hand-written special classes with well-chosen custom hash functions
  may work better. The same probably applies to `Integer`.

- It makes sense to check, and possibly fine-tune, the hash function for the specific case. It mustn't produce
  results much worse than the expected value of the <i>Num<sub>k</sub></i> random variable. The above formulas
  can be used to find both the expected value and the variance.

- There is always a trade-off between quality and speed of hash functions. Sometimes very good hash function may turn
  out to be too slow.

- Time to run the test went down by 29% for **Java 7** and 63% for **Java 8**, compared with the original version
  (much more if compared with the first `long`-based version).

Coming soon
-----------

In the next article I'll try to optimise `CRC32` and see if it can be made faster. I also want to test the influence
of the hash table size, replace basic types with some mutable classes and check what else can be done to speed up
calculations. Finally, I want to replace **Java** standard hash tables with custom-made ones and see if it helps.
