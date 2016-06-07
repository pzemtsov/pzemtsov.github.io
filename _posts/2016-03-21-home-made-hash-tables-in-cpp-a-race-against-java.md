---
layout: post
title:  "Home-made hash tables in C++: a race against Java"
date:   2016-03-21 12:00:00
tags: Java optimisation hashmap life
story: life
story-title: "Conway's Game of Life"
---

This is the third article in a row that has a title starting with "Home-made". While for the first one (on binary numbers) this is just a coincidence,
for the other two (on hash tables) this illustrates specialisation as an optimisation idea. The code specially written for the task may perform better than the generic code.
I say "may", because this isn't an absolute rule. The cases when this rule does not apply include:

- very complex algorithms where re-implementing them is likely to cause plain bugs (it is surprising how many people make mistakes even writing a binary search from scratch);

- algorithms that contain special performance cases, for which the developers have already provided a special treatment;

- cases where the language is powerful enough to provide ways to implement generic code that efficiently performs any specific tasks.

I'm assuming here that we need optimisation in the first place, that's why I don't mention obvious cases where this particular piece of code isn't a bottleneck,
or the performance isn't critical at all.

I always believed that the last of the three points above is applicable to **C++**. Let's verify this using the problem that we have just solved in **Java**: a Game of Life simulation
using hash maps.

The first implementation was done here: ["{{ site.TITLE-HASH }}"]({{ site.ART-HASH }}). We studied the effect of the hash functions and found some that worked well.

Then we applied some improvements to the program: ["{{ site.TITLE-OPTIMISE-LIFE }}"]({{ site.ART-OPTIMISE-LIFE }}). Most of the improvements dealt with replacing of **Java**
standard classes with the classes of our own, which helped improve performance. We didn't, however, replace the standard `HashMap` class.

Finally, in ["{{ site.TITLE-HASH-HOMEMADE }}"]({{ site.ART-HASH-HOMEMADE }}) we implemented our own version of the `HashMap` class, which gave additional boost of performance.
The overall increase from when we started was about 15 times; the last 80% came from this rewriting. The article contains a full list of applied improvements and
their effect.	

Now I'd like to apply similar approach to **C++**. Two major questions for today's study are:

- Does **C++** templating make custom-made implementation of the standard classes completely unnecessary or even harmful?

- How does performance of **C++** code compare with that of the similar **Java** code? Most people (me included) expect much higher performance from **C++**. How correct is
  this expectation?

We'll be comparing the results achieved on JRE 1.8.0.45 with those of GCC 4.5.4.

The latest version of **Java** code used as the basis of this development is [here]({{ site.REPO-LIFE }}/tree/36d73f731497d0aa84b8587a862b410cb77121da).
The new **C++** code is [here]({{ site.REPO-LIFE }}/tree/36f962059a6853711f2a3668be8bcf54418ce566).

The test
--------

The articles mentioned above give full explanation of the hash-based approach to the Life simulation. The last one (["{{ site.TITLE-HASH-HOMEMADE }}"]({{ site.ART-HASH-HOMEMADE }}))
contains the full list of performance figures.

We're using the [`ACORN`](http://conwaylife.appspot.com/pattern/acorn) pattern (see the [previous article]({{ site.ART-HASH-HOMEMADE }}) for the demonstration) and run its simulation
for 10,000 steps. The original time for this simulation in **Java** was 4961 ms. The improvements reduced it to 327 ms, so the simulation rate went up from 2,015 to 30,500 steps
per second.

We'll do the same for **C++**. We'll make a corresponding **C++** version for each **Java** one, except for the cases where it is not applicable. This becomes an exciting race.
Who comes first, **Java** or **C++**? And finally, after the finish of this race, we'll have another one, the super-race: we'll run the program with the
[Gosper Glider Gun](http://www.conwaylife.com/wiki/Gosper_glider_gun) as our original pattern. What makes this one special is that it produces more and more gliders
and thus makes the colony grow indefinitely.

Let's go.

The reference implementation
----------------------------

The general organisation of the new **C++** program follows that of the **Java** program. We have a class `Point` that represents co-ordinates, and an interface `Worker`
for Life  implementations. All the implementations will be checked against the default one, called `Hash_Reference`. This implementation works directly on `Point` values
and uses the direct **C++** equivalents of **Java** classes `HashMap` and `HashSet`:

{% highlight C++ %}
std::unordered_set<Point, PointHash> field;
std::unordered_map<Point, int, PointHash> counts;

typedef uint32_t HashType;

class PointHash
{
public:
    inline HashType operator() (Point p) const
    {
        return (HashType) (p.x * 3 + p.y * 5);
    }
};
{% endhighlight %}

The conversion of **Java** code is straightforward. This is what setting and resetting of cells looks like:

{% highlight C++ %}
void inc(Point w)
{
    ++ counts[w];
}

void dec(Point w)
{
    if (!--counts[w]) {
        counts.erase(w);
    }
}

void set(Point w)
{
    for (Point p : w.neighbours()) {
        inc(p);
    }
    field.insert(w);
}

void reset(Point w)
{
    for (Point p : w.neighbours()) {
        dec(p);
    }
    field.erase(w);
}
{% endhighlight %}

This relies on the `Point` class to implement method

{% highlight C++ %}
std::array<Point, 8> neighbours() const
{% endhighlight %}

Here is the result:

<table class="numeric">
<tr><th>Class name</th>                   <th>Comment</th><th>Time, <b>Java&nbsp;8</b></th><th>Time, <b>C++</b></th></tr>
<tr><td class="label">Hash_Reference</td> <td class="ttext">Initial version with a trivial hash function  </td><td> 4961</td><td>1372</td></tr>
</table>

**C++** won the start of the race with a big advantage.

`Hash_LongPoint`: Storing positions as 64-bit numbers
-----------------------------------------------------

Just like in **Java**, we switch from using a class (`Point`) to using a 64-bit number, only in **C++** case it will be an unsigned number:

{% highlight C++ %}
typedef uint64_t LongPoint;
{% endhighlight %}

In **Java** we had to wrap this value in a special class, so that `HashMap` could call `hashCode()` on the instances of that class. In **C++** this isn't necessary,
because we can specify a hash function as a template parameter of `unordered_map`. We can also make this function a template parameter of our Life implementation,
which makes a factory class unnecessary:

{% highlight C++ %}
template<class HASH = std::hash<LongPoint> > class Hash_LongPoint
                                                   : public Worker
{
private:
    std::unordered_set<LongPoint, HASH> field;
    std::unordered_map<LongPoint, int, HASH> counts;
{% endhighlight %}

We can now test `Hash_LongPoint` on the same set of hash functions as we did in **Java** (this set includes the standard hash function for a 64-bit number).

Here are the results:

<table class="numeric">
<tr><th>Class name</th>                <th>Comment</th>                                    <th>Time, <b>Java 8</b></th><th>Time, <b>C++</b></th></tr>
<tr><td class="label">Long</td>        <td class="ttext"><code>Long</code> default         </td><td> 5486</td><td> 1043 </td></tr>
<tr><td class="label">LongPoint</td>   <td class="ttext">Multiply by 3, 5                  </td><td> 4809</td><td> 1355 </td></tr>
<tr><td class="label">LongPoint3</td>  <td class="ttext">Multiply by 11, 17                </td><td> 1928</td><td> 1183 </td></tr>
<tr><td class="label">LongPoint4</td>  <td class="ttext">Multiply by two big primes        </td><td> 1611</td><td> 1302 </td></tr>
<tr><td class="label">LongPoint5</td>  <td class="ttext">Multiply by one big prime         </td><td> 1744</td><td> 1256 </td></tr>
<tr><td class="label">LongPoint6</td>  <td class="ttext">Modulo big prime                  </td><td> 1650</td><td> 1201 </td></tr>
<tr><td class="label">LongPoint7</td>  <td class="ttext">CRC32                             </td><td> 3254</td><td> 1244 </td></tr>
</table>

The **C++** equivalent of the `Long` default hash function is the default one for `LongPoint`, which is `uint64_t`. It is available via the standard class
`std::hash<LongPoint>`. This function performs much better than its **Java** counterpart. It is the best of the lot, which is impressive.

Our best option in **Java** (`LongPoint6`, which divides the number by a big prime number and returns the reminder) also performs reasonably well, but, surprisingly,
`LongPoint3` performs better.

The CRC-based function isn't as bad as in **Java**'s case, because this time we use the hardware-based implementation:

{% highlight C++ %}
#include <nmmintrin.h>

class LongPointHash7
{
public:
    HashType operator() (LongPoint p) const
    {
        return (HashType)~_mm_crc32_u64(~(uint64_t)0, p);
    }
};
{% endhighlight %}

We'll test all our implementations with the three best hash functions: the default one for `int64_t`, the `LongPoint3` (multiplying by 11 and 17), and our **Java** favourite,
the division-based `LongPoint6`.


`Hash1`: using mutable objects as hash table values
---------------------------------------------------

In **Java** we used values of class `Integer` as our initial values, because this class is standard in **Java** and allows boxing and unboxing, which makes the code clearer.
In **C++** we can make hash maps of anything, including type `int`. No boxing is needed, and the values are immediately mutable, which makes the code very nice:

{% highlight C++ %}
void inc(Point w)
{
    ++ counts[w];
}
{% endhighlight %}

(indexation of the hash map creates an empty element if the index is not found. In the case of `int` it is zero, which is exactly what we need).

This means that the **C++** version of `Hash_LongPoint` already employs the `Hash1` improvement, so there is no equivalent of `Hash1` in **C++**.
In **Java** this change improved time from 1650 to 1437 ms; all our **C++** versions are still faster.

`Hash2`: a unified hash map
---------------------------

We merge two hash maps (a real hash map and a hash set) into one. For that, we create the class `Value`, exactly like we did in **Java**.
All the other changes are also straightforward.

<table class="numeric">
<tr><th>Description</th><th>Time, <b>Java&nbsp;8</b></th><th>Time, <b>C++</b></th></tr>
<tr><td class="ttext">A unified hash map, default hash      </td><td>     </td><td> 845 </td></tr>
<tr><td class="ttext">A unified hash map, multiply by 11, 17</td><td>     </td><td> 961 </td></tr>
<tr><td class="ttext">A unified hash map, modulo big prime  </td><td> 1163</td><td> 1003 </td></tr>
</table>

The standard hash function is still the winner, and **C++** is still ahead (although **Java** is catching up).

`Hash3`: a check in `set()` removed
-----------------------------------

In **Java**, the code of `set()` looked like this:

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

The improvement made use of the fact that `c == null` condition is never true. The corresponding fragment in **C++** just looks like this:

{% highlight Java %}
    field[w].set();
{% endhighlight %}

The check happens inside the implementation of `unordered_map`, so we can't remove it. As a result, the `Hash3` improvement is not applicable.
In **Java** it reduced the time from 1163 ms to 1151 ms; **C++** is still ahead.


`Hash4`: `Entry` iterator in the loop
-------------------------------------

This improvement was very specific for **Java**. The **Java**'s hash map has a concept of a key set and that of an entry set, so there are two ways of iterating the map:

{% highlight Java %}
for (LongPoint w : field.keySet ()) {
    Value c = field.get (w);
{% endhighlight %}

and

{% highlight Java %}
for (Entry<LongPoint, Value> w : field.entrySet ()) {
    Value c = w.getValue ();
{% endhighlight %}

Replacing the first one with the second one produced big effect: the time went down from 1151 ms to 976 ms.

In **C++** there is only one way to iterate the map, and that is the one we've already been using:

{% highlight C++ %}
for (auto w : field) {
    Value& v = w.second;
{% endhighlight %}

where `auto` actually stands for `std::pair<LongPoint, Value&>`.

This means that we are already iterating over hash map entries, so this improvement is not applicable.
For the first time, **Java** runs faster than **C++** (the **C++** time is 1003 ms). However, this applies to the division-based hash function, which was the best in **Java**.
The **C++** still has two reserves: two other hash functions that made the time 845 ms and 961 ms.

`Hash5`: `ArrayList<Entry>` introduced
--------------------------------------

This has a direct equivalent in **C++**. Instead of collecting `LongPoint` values into a `std::vector`, we can collect values produced by the map iterator,
which are `std::pair<LongPoint, Value&>`. The rest of the program must be modified exactly as in **Java**.

<table class="numeric">
<tr><th>Description</th><th>Time, <b>Java&nbsp;8</b></th><th>Time, <b>C++</b></th></tr>
<tr><td class="ttext"><code>ArrayList&lt;Entry&gt;</code>, default hash </td><td>     </td><td> 812 </td></tr>
<tr><td class="ttext"><code>ArrayList&lt;Entry&gt;</code>, multiply by 11, 17</td><td>     </td><td> 909 </td></tr>
<tr><td class="ttext"><code>ArrayList&lt;Entry&gt;</code>, modulo big prime  </td><td> 975</td><td>  923 </td></tr>
</table>

**C++** has regained leadership. The race becomes more and more interesting.

`Hash6`: re-using `ArrayList` objects
-------------------------------------

In **C++** the equivalent of `ArrayList` is `std::vector`, and it benefits from the same modification as the `ArrayList`: re-using the object may reduce the
amount of memory allocations. The change is very similar to that in **Java**: `toSet` and `toReset` become fields, and `clear()` is called on them in `step()`.

<table class="numeric">
<tr><th>Description</th><th>Time, <b>Java&nbsp;8</b></th><th>Time, <b>C++</b></th></tr>
<tr><td class="ttext">Re-used <code>ArrayList&lt;Entry&gt;</code>, default hash      </td><td>     </td><td> 726 </td></tr>
<tr><td class="ttext">Re-used <code>ArrayList&lt;Entry&gt;</code>, multiply by 11, 17</td><td>     </td><td> 842 </td></tr>
<tr><td class="ttext">Re-used <code>ArrayList&lt;Entry&gt;</code>, modulo big prime  </td><td> 947</td><td>  857 </td></tr>
</table>

This change really helped in **C++** -- even more than it did in **Java**. The **C++** is gaining advantage in this race.

`Hash7`: replacing `ArrayList` with a plain array
-------------------------------------------------

This change helped in **Java**; let's see if the equivalent change helps in **C++**. However, when trying to implement a similar scheme, we immediately
discover that there isn't a fully equivalent change. The **C++**'s version of the `Entry` class is `std::pair<LongPoint, Value&>`, which does not have
a default constructor, so we can't allocate arrays of this type:

{% highlight C++ %}
toReset = new Entry[INITIAL_ARRAY_SIZE];
{% endhighlight %}

just won't compile. We could resolve  this by using low-level memory manipulation functions in a **C**-like manner, but that would be too much for our task. Besides,
this is what the standard library already does inside its implementation of `std::vector`.

We'll have to create another `Entry` class:

{% highlight C++ %}
typedef std::pair<LongPoint, Value*> Entry;
{% endhighlight %}

and then convert from one to another:

{% highlight C++ %}
for (auto it = field.begin(); it != field.end(); it++) {
    Value& v = it->second;
    if (v.live) {
        if (v.count < 2 || v.count > 3) toReset[reset_count++] =
                                          Entry(it->first, &v);
    }
    else {
        if (v.count == 3) toSet[set_count++] = Entry(it->first, &v);
    }
}
{% endhighlight %}

Here is the result:

<table class="numeric">
<tr><th>Description</th><th>Time, <b>Java&nbsp;8</b></th><th>Time, <b>C++</b></th></tr>
<tr><td class="ttext"><code>ArrayList</code> replaced with an array, default hash      </td><td>     </td><td> 753 </td></tr>
<tr><td class="ttext"><code>ArrayList</code> replaced with an array, multiply by 11, 17</td><td>     </td><td> 869 </td></tr>
<tr><td class="ttext"><code>ArrayList</code> replaced with an array, modulo big prime  </td><td> 904</td><td>  891 </td></tr>
</table>

This is a disappointment. The times got worse, although still better than those in **Java**. This is the example of the idea opposite to the one we started with: in **C++**
using the standard classes may be more beneficial than re-implementing them. I wish it stays like that.

`Hash8`: using the `LinkedHashMap` and increasing the hash map capacity
----------------------------------------------------------------------

The standard **C++** library does not have an equivalent of the `LinkedHashMap` class. Probably, the library designers decided that whoever needs to arrange data in some other
container, such as a linked list, can do so easily. We can test what happens when we increase the hash map capacity of our existing solution (we'll use the `Hash6` as the best so far):

<table class="numeric">
<tr><th>Description</th><th>Time, <b>Java&nbsp;8</b></th><th>Time, <b>C++</b></th></tr>
<tr><td class="ttext">Capacity made bigger, default hash      </td><td>     </td><td> 2075 </td></tr>
<tr><td class="ttext">Capacity made bigger, multiply by 11, 17</td><td>     </td><td> 1847 </td></tr>
<tr><td class="ttext">Capacity made bigger, modulo big prime  </td><td> 591</td><td>  2090 </td></tr>
</table>

This is the first time where **Java** performance is much higher than that of the **C++**. Of course, the comparison isn't completely fair, since we compare different algorithms,
but still in **Java** the improved algorithm is available as a part of the standard library, while in **C++** it is not.

The first home-made implementation
----------------------------------

Now we are going to replace the standard hash map implementation with our own one. We'll copy the **Java** implementation almost exactly.
The class `Field` implements the hash table functionality, while `Hash_MomeMade` uses it to implement Life.
The most notable differences are:

- in **Java** we had to pass a hasher object into a constructor, while in **C++** it became a template parameter, removing any overheads when using it;

- in **Java** we passed the hash map capacity to the constructor, while in **C++** we also made it a template parameter -- a constant is always more efficient to use than a variable;

- **Java** version employs the optimisation of `Hash7` (replacing of `ArrayList` collections with plain arrays), while the **C++** version does not -- this change didn't improve
performance in **C++**;

- **C++** does not have a garbage collection, so we must explicitly delete all objects that become unused. We must also delete the main array in the destructor.

Since we were testing on both **Java 7** and **Java 8**, we weren't using the lambda syntax in **Java**. As a result the main method (`step`) looked a bit ugly, In **C++** we
can use this syntax. Here is what the `step` method looks like:

{% highlight C++ %}
void step()
{
    toSet.clear();
    toReset.clear();

    field.iterate([this](Cell * cell) {
        if (cell->live) {
            if (cell->neighbours < 2 || cell->neighbours > 3)
               toReset.push_back(cell);
        }
        else {
            if (cell->neighbours == 3) toSet.push_back(cell);
        }
    });
    for (Cell * c : toSet) {
        set(c);
    }
    for (Cell * c : toReset) {
        reset(c);
    }
}
{% endhighlight %}

Here `Cell` is the class that represents the value stored in the hash table, together with the key and hash table artefacts (the index value and the `table_next` pointer
to arrange values in a linked list). Our vectors are defined as

{% highlight C++ %}
std::vector<Cell*> toReset;
std::vector<Cell*> toSet;
{% endhighlight %}

As in **Java**, we first test it on a small table capacity (8192). Here are the results:

<table class="numeric">
<tr><th>Description</th><th>Time, <b>Java&nbsp;8</b></th><th>Time, <b>C++</b></th></tr>
<tr><td class="ttext">The first home-made hash map implementation (small capacity), default hash      </td><td>     </td><td> 1143 </td></tr>
<tr><td class="ttext">The first home-made hash map implementation (small capacity), multiply by 11, 17</td><td>     </td><td> 629 </td></tr>
<tr><td class="ttext">The first home-made hash map implementation (small capacity), modulo big prime  </td><td> 465</td><td>  611 </td></tr>
</table>

This result is sensational: **Java** runs faster than **C++** with exactly the same algorithm. The only explanation I can think of is the garbage collector. While usually
we tend to think about it as a performance-hindering feature, here it can actually help, provided that the heap size is big enough. Instead of deallocating every single unused object,
we get rid of all of them, also at some cost, but rather infrequently.

Another interesting feature of this result is that for the first time the hash function that was the best in **Java** also became the best in **C++**. Moreover, the previous
**C++**'s best (the default 64-bit integer hash) is now performing worse than before. This is something to investigate later.

`Hash_HomeMade2`: the hash map code is built into the main code
---------------------------------------------------------------

In **Java** this helped because it removed callback-based iteration. Virtual calls in **Java** are fast but directly-inserted code is even faster. Does it work the same way
in **C++**?

Mostly, the code transformation is about merging the two classes (`Field` and `Hash_HomeMade`), and moving all the relevant methods into one combined class. We can then
provide some degree of encapsulation by collecting all the `Field` methods together and visually separating them from the code using them. It works with all the methods except
for iteration (and more efficient iteration was the point of this change). In **Java** we didn't have a choice but to put the iteration code directly where it was used:

{% highlight Java %}
for (Cell cell : table) {
    for (Cell c = cell; c != null; c = c.table_next) {
        if (c.live) {
            if (c.neighbours < 2 || c.neighbours > 3) toReset[resetPtr ++] = c;
        } else {
            if (c.neighbours == 3) toSet[setPtr ++] = c;
        }
    }
}
{% endhighlight %}

In **C++** we can define the iterator code as a macro:

{% highlight C++ %}
#define ITERATE(cell) \
    for (size_t i = 0; i < CAPACITY; i++) {\
        for (Cell* cell = table[i]; cell; cell = cell->table_next)

#define END_ITERATE }

ITERATE (cell) {
    if (cell->live) {
        if (cell->neighbours < 2 || cell->neighbours > 3)
            toReset.push_back(cell);
    }
    else {
        if (cell->neighbours == 3) toSet.push_back(cell);
    }
} END_ITERATE
{% endhighlight %}
 
There are different opinions on if this macro trick really improves readability or makes it worse. It helps keep all the relevant code
in one place without sacrificing performance, but for many eyes it looks ugly.

Here are the times:

<table class="numeric">
<tr><th>Comment</th><th>Time, <b>Java&nbsp;8</b></th><th>Time, <b>C++</b></th></tr>
<tr><td class="ttext">The hash map code is built into the main code, default hash      </td><td>     </td><td> 1107 </td></tr>
<tr><td class="ttext">The hash map code is built into the main code, multiply by 11, 17</td><td>     </td><td> 622 </td></tr>
<tr><td class="ttext">The hash map code is built into the main code, modulo big prime  </td><td> 449</td><td>  576 </td></tr>
</table>

The speed has improved, but **Java** version is still faster.

`Hash_HomeMade3`: collecting cells in a double-linked list
----------------------------------------------------------

In **Java** this imitated the activity of `LinkedHashMap`. We can translate this solution to **C++** directly. Instead of class `Cell`, we use class `ListedCell` with
`prev` and `next` pointers. Obviously, we must add appropriate list modifications where the cells are inserted into the map or removed from it.

The macros `ITERATE` and `END_ITERATE` introduced in the previous section, can help again. We simply rewrite this macros as

{% highlight C++ %}
#define ITERATE(c) \
    for (ListedCell* c = full_list.next; c != &full_list; c = c->next)

#define END_ITERATE
{% endhighlight %}

and the client code (in our case, that of `step()`) does not have to change at all.

It would have been nice to have macros as language objects with usual visibility rules, rather than a pure text pre-processing feature. Then we would place all the
hash map functionality in a separate class and export the iteration macros from there.

Here are the times:

<table class="numeric">
<tr><th>Description</th><th>Time, <b>Java&nbsp;8</b></th><th>Time, <b>C++</b></th></tr>
<tr><td class="ttext">Cells collected in a list, default hash      </td><td>     </td><td> 1286 </td></tr>
<tr><td class="ttext">Cells collected in a list, multiply by 11, 17</td><td>     </td><td> 567 </td></tr>
<tr><td class="ttext">Cells collected in a list, modulo big prime  </td><td> 433</td><td>  552 </td></tr>
</table>

The default hash result became worse, the rest improved. **C++** is still behind, but the improvement is better than in **Java**. **C++** is slowly catching up.

`Hash_HomeMade4`: The hash map capacity is set to 256K
------------------------------------------------------

In **Java** the hash map capacity was defined as a constant to improve performance, so we had to make anothe class to change this value. In **C++** it is a template parameter,
so the **C++**'s version of the **Java**'s `Hash_HomeMade4` is `Hash_HomeMade3<256*1024>`.

Here are the times:

<table class="numeric">
<tr><th>Description</th><th>Time, <b>Java&nbsp;8</b></th><th>Time, <b>C++</b></th></tr>
<tr><td class="ttext">Big hashmap capacity, default hash      </td><td>     </td><td> 1184 </td></tr>
<tr><td class="ttext">Big hashmap capacity, multiply by 11, 17</td><td>     </td><td> 500 </td></tr>
<tr><td class="ttext">Big hashmap capacity, modulo big prime  </td><td> 353</td><td>  424 </td></tr>
</table>

**C++** keeps catching  up (using the fastest hash function), but is still behind.

`Hash_HomeMade5`: the action lists were introduced
--------------------------------------------------

Here we replace our `vector` containers with usual linked lists. For that, we move on from the `ListedCell` to the `ActilnCell` class, which has one extra reference field
(`next_action`), and introduce the obvious transformations of the code. Again, the garbage collection issue becomes important. This time the code change is in favour 
of **C++**. In **Java**, the iteration over the action lists looked like this:

{% highlight Java %}
for (ActionCell c = toSet; c != null; c = next_action) {
    set (c);
    next_action = c.next_action;
    c.next_action = null;
}
for (ActionCell c = toReset; c != null; c = next_action) {
    reset (c);
    next_action = c.next_action;
    c.next_action = null;
}
{% endhighlight %}

It was important to set `c.next_action` to `null`, to remove an extra reference to another `Cell` object. If we didn't do that, a lot of unwanted objects would be considered
live and escape garbage collection.

In **C++** this is unnecessary, but there is another consideration:

{% highlight C++ %}
for (ActionCell* c = toSet; c; c = c->next_action) {
    set(c);
}
for (ActionCell* c = toReset; c;) {
    ActionCell* next_action = c->next_action;
    reset(c);
    c = next_action;
}
{% endhighlight %}

Since `reset()` can delete the object, we have to make sure to extract the `next_action` field before this call. There is no such problem for `set()`.

Here are the results:

<table class="numeric">
<tr><th>Description</th><th>Time, <b>Java&nbsp;8</b></th><th>Time, <b>C++</b></th></tr>
<tr><td class="ttext">Action lists, default hash      </td><td>     </td><td> 1162 </td></tr>
<tr><td class="ttext">Action lists, multiply by 11, 17</td><td>     </td><td> 478 </td></tr>
<tr><td class="ttext">Action lists, modulo big prime  </td><td> 354</td><td>  425 </td></tr>
</table>

It improved all the cases except for the fastest one. It did the same in **Java**, though.

`Hash_HomeMade6`: hard-coded hash function
------------------------------------------

Since our hash function is a template parameter, it is as good as already hard-coded. We can skip this version. It didn't help much in **Java** either: the time
went down from 354 ms to 353 ms (recovering from the action lists damage).


`Hash_HomeMade7`: the memory allocation optimisation
----------------------------------------------------

This is the first time we'll be doing something in **C++** that isn't applicable in **Java**. We allocate and deallocate memory every time we insert a new cell into
the hash table or remove existing one. We could keep the objects ane re-use them. We could do that in **Java**, too, but in **Java** memory allocation is cheap, so it
didn't promise any improvements.

We'll use the same `ActionCell` as before and collect all unallocated elements in a linked list, using the `next` pointer. Calls to `new` and `delete` must be modified in a
straightforward way:

{% highlight C++ %}
ActionCell * allocate(LongPoint position, int neighbours, bool live = false)
{
    if (!free_list) {
        return new ActionCell(position, neighbours, live);
    }
    ActionCell * result = free_list;
    free_list = result->next;
    result->set(position, neighbours, live);
    return result;
}

void deallocate(ActionCell * cell)
{
    cell->next = free_list;
    free_list = cell;
}
{% endhighlight %}

Since this is the last of our versions that is applicable to various hash functions, we'll run it with all of them:

<table class="numeric">
<tr><th>Hash function</th><th>Time, <b>Java&nbsp;8</b></th><th>Time, <b>C++</b></th></tr>
<tr><td class="ttext">Default hash               </td><td>     </td><td> 1065 </td></tr>
<tr><td class="ttext">Multiply by 3, 5           </td><td>     </td><td> 542 </td></tr>
<tr><td class="ttext">Multiply by 11, 17         </td><td>     </td><td> 411 </td></tr>
<tr><td class="ttext">Multiply by two big primes </td><td>     </td><td> 441 </td></tr>
<tr><td class="ttext">Multiply by one big prime  </td><td>     </td><td> 437 </td></tr>
<tr><td class="ttext">Modulo big prime           </td><td> 353 </td><td> 359 </td></tr>
<tr><td class="ttext">CRC32                      </td><td>     </td><td> 438 </td></tr>
</table>

The division-based hash function is the best. It almost caught up with **Java**, only a little bit is left to go.

`Hash_Additive`: The additive division-based hash function
----------------------------------------------------------

We've finished the generic algorithms (suitable for any hash functions), and the next thing to try is the set of additive algorithms, based on the calculation
of the hash value based on the previous hash value. It worked in **Java** (at least, for one version), we'll try this in **C++**.

The first version operates using the division-based hash function and makes use of the fact that when a constant is added to the divisible, the remainder changes in a
predictable way. The rest stays the same as in `Hash_HomeMade7`.


<table class="numeric">
<tr><th>Description</th><th>Time, <b>Java&nbsp;8</b></th><th>Time, <b>C++</b></th></tr>
<tr><td class="ttext">The additive implementation of "Modulo big prime" hash </td><td> 327 </td><td> 312 </td></tr>
</table>

Finally, we've made it: the **C++** version runs faster than **Java**'s one.

`Hash_Additive2`: The additive multiplication-based hash function
-----------------------------------------------------------------

The next version is based on the `LongPointHash4` hash function, which multiplies two parts of a 64-bit number by two big prime numbers and adds the results. The transition
from one hash value to another one when a constant is added to the key is trivial. However, the quality of hashing of this function is somewhat lower than that of the
division-based one, so we can't predict how this version will perform.

<table class="numeric">
<tr><th>Description</th><th>Time, <b>Java&nbsp;8</b></th><th>Time, <b>C++</b></th></tr>
<tr><td class="ttext">The additive implementation of the "Multiply by two big primes" hash  </td><td> 351 </td><td> 360 </td></tr>
</table>

**C++** is behind again, but, anyway, this version isn't the fastest.

`Hash_Additive3`: The ultimate additive multiplication-based hash function
--------------------------------------------------------------------------

This version does not keep original co-ordinates at all, only their values multiplied by prime numbers. This makes calculating hash values very fast but requires special
procedure to recover the co-ordinates when reporting the simulation results.

<table class="numeric">
<tr><th>Description</th><th>Time, <b>Java&nbsp;8</b></th><th>Time, <b>C++</b></th></tr>
<tr><td class="ttext">Improved additive implementation of the "Multiply by two big primes" hash  </td><td> 361 </td><td> 348 </td></tr>
</table>


**Java** became slower, **C++** became faster and caught up, but this is still not the fastest version.

The first race: The summary
---------------------------

This is the end of the first race. Finally, the **C++** has won, but just marginally (312 ms against 327 in **Java**). Here is the full scorecard
(we show the best results in each row): 

<table class="numeric">
<tr><th>Java class name</th>              <th>Comment</th><th>Time, <b>Java&nbsp;8</b></th><th>Time, <b>C++</b></th></tr>
<tr><td class="label">Hash_Reference</td> <td class="ttext">Initial version with a trivial hash function  </td><td> 4961</td><td> 1372</td></tr>
<tr><td class="label">Hash_LongPoint</td> <td class="ttext">Storing position in a 64-bit value and
                                                            using a good hash function                    </td><td> 1650</td><td> 1043</td></tr>
<tr><td class="label">Hash1</td>          <td class="ttext">Mutable Count                                 </td><td> 1437</td><td>     </td></tr>
<tr><td class="label">Hash2</td>          <td class="ttext">A unified hash map                            </td><td> 1163</td><td>  845</td></tr>
<tr><td class="label">Hash3</td>          <td class="ttext">Check in <code>set</code> removed             </td><td> 1151</td><td>     </td></tr>
<tr><td class="label">Hash4</td>          <td class="ttext"><code>Entry</code> iterator in the loop       </td><td>  976</td><td>     </td></tr>
<tr><td class="label">Hash5</td>          <td class="ttext"><code>ArrayList&lt;Entry&gt;</code> introduced</td><td>  975</td><td>  812</td></tr>
<tr><td class="label">Hash6</td>          <td class="ttext">Re-used <code>ArrayList&lt;Entry&gt;</code>   </td><td>  947</td><td>  726</td></tr>
<tr><td class="label">Hash7</td>          <td class="ttext"><code>ArrayList</code> replaced with an array </td><td>  904</td><td>  753</td></tr>
<tr><td class="label">Hash8</td>          <td class="ttext"><code>LinkedHashMap</code> used,
                                                            capacity made bigger                          </td><td>  591</td><td> 1847</td></tr>
<tr><td class="label">Hash_HomeMade</td>  <td class="ttext">The first home-made hash map implementation
                                                            (small capacity)                              </td><td>  465</td><td>  611</td></tr>
<tr><td class="label">Hash_HomeMade2</td> <td class="ttext">The hash map code is built into the main code </td><td>  449</td><td>  576</td></tr>
<tr><td class="label">Hash_HomeMade3</td> <td class="ttext">The cells are collected into
                                                            a double-linked list                          </td><td>  433</td><td>  552</td></tr>
<tr><td class="label">Hash_HomeMade4</td> <td class="ttext">The hash map capacity is set to 256K          </td><td>  353</td><td>  424</td></tr>
<tr><td class="label">Hash_HomeMade5</td> <td class="ttext">The action lists were introduced              </td><td>  354</td><td>  425</td></tr>
<tr><td class="label">Hash_HomeMade6</td> <td class="ttext">The division-based hash function, hard coded  </td><td>  353</td><td>     </td></tr>
<tr><td class="label">Hash_Additive</td>  <td class="ttext">The division-based hash function, additive    </td><td>  327</td><td>  312</td></tr>
<tr><td class="label">Hash_Additive2</td> <td class="ttext">Multiplication by two big numbers, additive   </td><td>  351</td><td>  360</td></tr>
<tr><td class="label">Hash_Additive3</td> <td class="ttext">Multiplication by two big numbers,
                                                            ultimate additive                             </td><td>  361</td><td>  348</td></tr>
</table>


The big race: the glider gun
----------------------------

We've finished the [previous article]({{ site.ART-HASH-HOMEMADE }}) with the simulation of the [Gosper Glider Gun](http://en.wikipedia.org/wiki/Gun_%28cellular_automaton%29).
We'll do the same in **C++** now. This is the ultimate race. If the Acorn simulations can be compared with separate Formula-1 laps, the Glider Gun is more like
Le Mans 24 hours -- literally: it really runs for 24 hours and more (2,500,000 iterations ran for 33 hours in **Java**).

We'll be running the `Hash_Additive` version, as this is the best one we've produced so far.

Here are the results (I didn't have patience to finish 2.5M steps):

<table class="numeric">
<tr><th rowspan="2">Steps</th><th colspan="2">Time for 10,000, sec</th><th colspan="2">Total time, sec</th></tr>
<tr>               <th><b>Java</b></th><th><b>C++</b></th><th><b>Java</b></th><th><b>C++</b></th></tr>
<tr><td>   10,000</td><td> 0.75</td><td> 0.77</td><td>   0.75</td><td>   0.77</td></tr>
<tr><td>   20,000</td><td>  1.6</td><td>  2.7</td><td>    2.3</td><td>    3.5</td></tr>
<tr><td>   30,000</td><td>  2.7</td><td>  4.9</td><td>    5.0</td><td>    8.4</td></tr>
<tr><td>  500,000</td><td>  125</td><td>  317</td><td>  2,206</td><td>  6,301</td></tr>
<tr><td>1,000,000</td><td>  324</td><td>  980</td><td> 13,577</td><td> 37,160</td></tr>
<tr><td>1,500,000</td><td>  553</td><td>1,670</td><td> 35,203</td><td>101,968</td></tr>
<tr><td>2,000,000</td><td>  834</td><td>2,436</td><td> 70,420</td><td>187,174</td></tr>
<tr><td>2,500,000</td><td>1,126</td><td>     </td><td>119,481</td><td>       </td></tr>
</table>

This result is sensational. The **C++** is losing dramatically. The execution speed in **C++** becomes three times lower than in **Java** as the structure grows.
Why this happens is a topic of another big investigation. We can straight away come with some theories:

- One possible explanations involves a different strategy of memory allocation (direct allocate/deallocate vs garbage collection);

- Different memory layout patterns could have caused better utilisation of the RAM cache in **Java**'s case;

- **Java** could have generated much better code;

- This could be a result of **Java** using compressed pointers (objects become shorter and more of them fit into the cache).

We won't investigate it here, however, as this article is already too long. We'll return to this issue later.

Conclusions
-----------

- The general myth "**C++**  is always faster than **Java**" is busted.

- The default hash map implementation in **C++** is better than the default one in **Java**.

- However, **Java**'s hash map performance can be improved by using higher capacity and the `LinkedHashMap`, while the **C++**'s can't (or, rather, it can, but by
  custom coding).

- Most of the optimisations applicable in **Java** are also applicable in  **C++**. This specifically applies to implementing a custom-made hash map.
  In our case it performed three times faster than the standard one (although the standard one worked three times faster than the **Java**'s standard one).

- The custom implementation of hash maps makes both **Java** and **C++** faster, but helps **Java** more (in fact, it helps it so much that it outperforms **C++**).

- The **C++**'s default hash function for integer numbers (`uint32_t` in our test) works much better than the one in **Java**. It performed better than any of the custom
  ones until we implemented a custom hash map. Why this happened is still a mystery.

- While **C++** caught up with **Java** performance on the small test, it failed miserably on the big one.

Coming soon
-----------

The **C++** program being three times slower than the **Java** version is an abnormal situation and a mystery. It must be investigated. I'll do that in one of the next articles.
