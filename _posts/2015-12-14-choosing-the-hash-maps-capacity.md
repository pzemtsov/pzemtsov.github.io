---
layout: post
title:  "Choosing the hash map's capacity"
date:   2015-12-14 12:00:00
tags: Java optimisation hashmap life
story: life
story-title: "Conway's Game of Life"
---

In ["{{ site.TITLE-HASH }}"]({{ site.ART-HASH }}) we implemented [Conway's game of Life](http://en.wikipedia.org/wiki/Conway's_Game_of_Life) using **Java**'s `HashMap`.
We found that performance depends on the choice of a hash function and studied how to measure their quality and speed. We found one hash function that is suitable.
In ["{{ site.TITLE-OPTIMISE-LIFE }}"]({{ site.ART-OPTIMISE-LIFE }}) we improved the original implementation. It now runs twice as fast as when  we started.
The code can be seen in [the repository]({{ site.REPO-LIFE }}/tree/0a84db39335b49aa6c83543c1afdce6ba8d491c1). 
Now I'm going to study the effect of the hash map's capacity.

The naming convention
---------------------

The `HashMap` class is backed by an array that contains the roots of the lists of hash map entries (which we'll call "chains").
It is tempting to call the size of that array the size of the hash map.
This would, however, be incorrect, since the word _size_ is reserved for the number of elements in the hash map (as returned by the `size()` method). The correct `HashMap`
slang for the array size is _capacity_.

The hash map and its capacity
-----------------------------

The underlying array is created and managed by the `HashMap` class itself, and usually the programmer doesn't have to bother about this array and its size.
The default constructor of the `HashMap` sets the capacity to 16; the user may specify another value, which will be
rounded up to the nearest power of two. Another parameter that regulates the capacity is called a `loadFactor`; it controls how many elements can be inserted into the
table before its array is expanded. The default value is 0.75. When the table size exceeds its capacity multiplied by this factor, the array is doubled in size
and all the elements copied to the new array. The table never shrinks. After some time the capacity stabilises at some reasonable value, which is bigger than the
table size, but not too much bigger.

In our implementations we preferred to avoid array resizing and allocated the tables with the initial capacity of 8192. The initial implementation used two hash maps.
One, called `field` (actually a `HashSet`, which is backed by a `HashMap`), contained the live cells. It reached the maximal size of 1034.
Another one, called `counts`, contained the numbers of the live neighbours for the cells. This one reached the size of 3938, so the `HashMap` with the default
capacity would have stabilised at the capacity of 8192 anyway. The `field` could use smaller table (2048).

The optimised implementation uses just one hash table, called `field`, which stores objects that contain both cell status (live or not) and their neighbour number.
This table reached the size of 4041. This size is bigger than the highest neighbour count (3938), because the new table contains live cells with neighbour count of zero.
Anyway, this table would also settle at the capacity of 8192.

The default load factor of 0.75 makes sure that the capacity is never too big. The biggest it gets is 8/3 of the
table size right after the resize. This approach seems too conservative. Why not use bigger capacity? This is what we'll study today.

Intuitively it seems that the bigger the capacity, the more efficient is the hash table. The number of comparisons performed for every operation is determined by the chain length
(the number of entries sharing the same array slot). If the hash function is not too bad, this chain length must decrease with grows of the capacity. We saw in
["{{ site.TITLE-HASH }}"]({{ site.ART-HASH }}) that the random hash function caused the average number of slots used after insertion of 3938 elements to be 3126,
which corresponds to the average chain length of 1.259. If we manage to bring this value down close to 1, we can eliminate 20% of all comparisons. It seems obvious that the
chain length will approach 1 as we increase the capacity, but it can also be proven formally.

In the same article we've produced the formula for the expected value of <i>Num<sub>k</sub></i> (the number of used array elements after _k_ insertions):

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
  
where _M_ is the capacity.

When _M_ grows infinitely, we can approximate the innermost expression with two terms of its binomial expansion:

  <div class="formula">
  <table>
  <tr>
    <td rowspan="2"><span class="big">(</span>1&nbsp;&minus;&nbsp;</td>
    <td class="num">1</td>
    <td rowspan="2">&nbsp;<span class="big">)</span></td>
    <td rowspan="2"><span class="big"><sup><sup>k</sup></sup></span></td>
    <td rowspan="2">&nbsp;=&nbsp;1&nbsp;&minus;&nbsp;</td>
    <td class="num">k</td>
    <td rowspan="2">&nbsp;+&nbsp;O&nbsp;<span class="big">(</span></td>
    <td class="num">1</td>
    <td rowspan="2"><span class="big">)</span></td>
  </tr>
  <tr>
    <td class="denom">M</td>
    <td class="denom">M</td>
    <td class="denom">M<sup>2</sup></td>
  </tr>
  </table>
  </div>

After substitution and reduction, we get the following for _E[Num<sub>k</sub>]_:

  <div class="formula">
  <table>
  <tr>
    <td rowspan=2>E[Num<sub>k</sub>]&nbsp;=&nbsp;k&nbsp;+&nbsp;O&nbsp;<span class="big">(</span></td>
    <td class="num">1</td>
    <td rowspan="2"><span class="big">)</span></td>
  </tr>
  <tr>
    <td class="denom">M</td>
  </tr>
  </table>
  </div>
  
The average chain length <i>Len<sub>k</sub></i> if inversely proportional to the number of used slots:

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

This value approaches 1 as _M_ grows. This is what the behaviour of _Len<sub>k</sub>_ looks like for k&nbsp;=&nbsp;4000:

<img src="{{ site.url }}/images/life-hash-size-chain-size-over-capacity.png" width="647" height="321">

This graph shows that we should be able to do better than with current value 8K. It seems to make sense to increase the capacity to at least 256K,
which would make the chain length very close to 1. Further increase of capacity doesn't promise any improvements.

Of course, other factors such as RAM cache may affect the results. We can't claim anything about the real impact of the capacity until we measure it. Let's do it.

Measuring the capacity impact
-----------------------------

The `HashMap` has a constructor that takes both the initial capacity and the load factor. We can give it some ridiculously big load factor (say, ten million), which will
effectively prohibit resizing. The capacity will stay as initialised. All we need is to modify the `HashMap` allocation:

{% highlight Java %}
public static final int HASH_SIZE = 8192;

private HashMap<LongPoint, Value> field =
    new HashMap<LongPoint, Value> (HASH_SIZE);
{% endhighlight %}

becomes something like this:

{% highlight Java %}
public static final int HASH_SIZE = 4096;

private HashMap<LongPoint, Value> field =
    new HashMap<LongPoint, Value> (HASH_SIZE, 10000000.0f));
{% endhighlight %}

Here are the results we get for a range of the capacities  (these are the times for 10,000 steps of Life simulation, in milliseconds):

<table class="numeric">
<tr><th>Capacity</th><th><b>Java 7</b></th><th><b>Java 8</th></tr>
<tr><td>     1 </td><td> 256372 </td><td>  1947 </td></tr>
<tr><td>     2 </td><td> 110658 </td><td>  1917 </td></tr>
<tr><td>     4 </td><td>  52715 </td><td>  1915 </td></tr>
<tr><td>     8 </td><td>  24597 </td><td>  1904 </td></tr>
<tr><td>    16 </td><td>  12032 </td><td>  1898 </td></tr>
<tr><td>    32 </td><td>   6941 </td><td>  1838 </td></tr>
<tr><td>    64 </td><td>   3721 </td><td>  1835 </td></tr>
<tr><td>   128 </td><td>   2432 </td><td>  1770 </td></tr>
<tr><td>   256 </td><td>   1763 </td><td>  1490 </td></tr>
<tr><td>   512 </td><td>   1425 </td><td>  1395 </td></tr>
<tr><td>  1024 </td><td>   1218 </td><td>  1161 </td></tr>
<tr><td>  2048 </td><td>   1104 </td><td>  1036 </td></tr>
<tr><td>  4096 </td><td>   1040 </td><td>   923 </td></tr>
<tr><td>  8192 </td><td>   1020 </td><td>   904 </td></tr>
<tr><td>   16K </td><td>   1089 </td><td>   806 </td></tr>
<tr><td>   32K </td><td>   1268 </td><td>   852 </td></tr>
<tr><td>   64K </td><td>   1640 </td><td>   935 </td></tr>
<tr><td>  128K </td><td>   2476 </td><td>  1186 </td></tr>
<tr><td>  256K </td><td>   3721 </td><td>  1602 </td></tr>
<tr><td>  512K </td><td>   6523 </td><td>  2516 </td></tr>
<tr><td>    1M </td><td>  13795 </td><td>  4313 </td></tr>
<tr><td>    2M </td><td>  26650 </td><td>  7916 </td></tr>
<tr><td>    4M </td><td>  52221 </td><td> 15160 </td></tr>
<tr><td>    8M </td><td> 110515 </td><td> 39127 </td></tr>
<tr><td>   16M </td><td> 228141 </td><td> 76980 </td></tr>
</table>

The numbers grow so high that the only way to show both **Java 7** and **Java 8** results on the same graph is by using the logarithmic scale:

<img src="{{ site.url }}/images/life-hash-size.png" width="642" height="372">

This is the central part of the **Java 7** graph (in traditional scale):

<img src="{{ site.url }}/images/life-hash-size-java7.png" width="645" height="375">

And this is the graph for **Java 8**:

<img src="{{ site.url }}/images/life-hash-size-java8.png" width="651" height="377">

Here are the three major observations from these graphs:

- Times on **Java 8** are always better than on **Java 7** (two to three times in the right side of the graph; much more on the left)

- Times on **Java 7** grow almost to infinity as the capacity decreases, while times on **Java 8** do not; the time stabilises at about 128 and stays roughly the same all
the way down. The time stays reasonable even when the capacity is set to 1.

- Times on both versions grow almost linearly on big capacities (above 64K) -- contrary to our expectations.

Let's look at these observations.

The difference in performance at small capacity values
------------------------------------------------------

We expect the speed to drop as the capacity decreases, so the behaviour on **Java 7** isn't a surprise.
Moreover, the observed times are roughly in accordance with our formula for _Len<sub>k</sub>_ (see above).
Let's assume k&nbsp;=&nbsp;2000 as a representative table size, and plot the value _time&nbsp;/&nbsp;Len<sub>k</sub>_ for small values of _M_ for **Java 7**:

<img src="{{ site.url }}/images/life-hash-size-time-over-len-java7.png" width="649" height="367">

The graph looks almost horizontal up to the capacity of 64, which means that the **Java 7** implementation behaves in the expected way.

The **Java 8** behaviour is different. It keeps running reasonably fast even when chains become very long. This can only be due to the innovation that has been
introduced in **Java 8**'s implementation of the `HashMap`: the binary trees of the entries. When entry lists exceed the length of 8, they are converted to binary trees
with full hash codes as search keys. This conversion takes time, but it improves the speed of the searches. As we see, the speed stays good even at the array size of 1, when
the entire table is stored in just one entry. Effectively, the entire hash map becomes a tree map in this case, which suggests that perhaps we must try this solution as well.

The average chain length (assuming k=2000) is 7.8 at the capacity of 256 and 15.6 at 128, which means that the capacity of 128 is roughly the point where
the binary trees start to kick in massively. This is also the point (see the "Time on **Java 8**" graph above) where the time graph behaviour changes; it becomes roughly
horizontal to the left of this point.

The performance at high capacity values
---------------------------------------

The times grow steadily as the capacity grows above 64K. Two lines on the logarithmic graphs are almost parallel to each other and almost straight, which means that
the time is roughly proportional to the capacity. Let's divide the time by the capacity and plot this value for both **Java 7** and **Java 8**:

<img src="{{ site.url }}/images/life-hash-size-time-over-capacity.png" width="653" height="390">

The graph shows that the time is indeed proportional to the capacity, very accurately so starting at approximately 1M elements and less accurately from 256K. The graphs also
show slight relative slowdown at 8M - probably, this is the effect of RAM caching.

There is nothing in the basic `HashMap` operations (`get`, `put` or `remove`) that depends linearly on the capacity (it is the entire purpose of a hash table to avoid
such dependency). However, there is such dependency in iterating the `HashMap`, and this is something we do in `step()`. The only way to iterate a hash map is to iterate its
underlying array and for the non-empty elements iterate corresponding chains (linked lists). Starting at some point, iterating over the empty array elements will dominate
everything else. It seems reasonable to collect the `HashMap` values into some other collection that allows easy insertion and deletion as well as quick iteration.
The linked list is the ideal option.
We don't need to do it by hand: **Java** has a standard class with this functionality: `LinkedHashMap`. The elements of this hash map are linked together in a doubly-linked
list. The main purpose of this class is different, though: it is to iterate the elements of the map in the order they were inserted, which the classic `HashMap` does not provide.
Still, we can try using this class and check if it helps.

This time we could run tests at bigger capacities -- up to 1G:

<table class="numeric">
<tr><th rowspan="2">Capacity</th><th colspan="2">Old time</th><th colspan="2">New time</tr>
<tr><th><b>Java&nbsp;7</b></th><th><b>Java&nbsp;8</b></th><th><b>Java&nbsp;7</b></th><th><b>Java&nbsp;8</b></th></tr>
<tr><td>     1 </td><td> 256372 </td><td>  1947 </td><td> 261389 </td><td> 2083 </td></tr>
<tr><td>     2 </td><td> 110658 </td><td>  1917 </td><td> 108080 </td><td> 1855 </td></tr>
<tr><td>     4 </td><td>  52715 </td><td>  1915 </td><td>  55500 </td><td> 1970 </td></tr>
<tr><td>     8 </td><td>  24597 </td><td>  1904 </td><td>  26517 </td><td> 1951 </td></tr>
<tr><td>    16 </td><td>  12032 </td><td>  1898 </td><td>  12690 </td><td> 1969 </td></tr>
<tr><td>    32 </td><td>   6941 </td><td>  1838 </td><td>   6643 </td><td> 1970 </td></tr>
<tr><td>    64 </td><td>   3721 </td><td>  1835 </td><td>   3795 </td><td> 1992 </td></tr>
<tr><td>   128 </td><td>   2432 </td><td>  1770 </td><td>   2447 </td><td> 1710 </td></tr>
<tr><td>   256 </td><td>   1763 </td><td>  1490 </td><td>   1728 </td><td> 1533 </td></tr>
<tr><td>   512 </td><td>   1425 </td><td>  1395 </td><td>   1361 </td><td> 1553 </td></tr>
<tr><td>  1024 </td><td>   1218 </td><td>  1161 </td><td>   1135 </td><td> 1174 </td></tr>
<tr><td>  2048 </td><td>   1104 </td><td>  1036 </td><td>    985 </td><td>  912 </td></tr>
<tr><td>  4096 </td><td>   1040 </td><td>   923 </td><td>    939 </td><td>  834 </td></tr>
<tr><td>  8192 </td><td>   1020 </td><td>   904 </td><td>    902 </td><td>  742 </td></tr>
<tr><td>   16K </td><td>   1089 </td><td>   806 </td><td>    878 </td><td>  640 </td></tr>
<tr><td>   32K </td><td>   1268 </td><td>   852 </td><td>    870 </td><td>  651 </td></tr>
<tr><td>   64K </td><td>   1640 </td><td>   935 </td><td>    820 </td><td>  601 </td></tr>
<tr><td>  128K </td><td>   2476 </td><td>  1186 </td><td>    844 </td><td>  593 </td></tr>
<tr><td>  256K </td><td>   3721 </td><td>  1602 </td><td>    873 </td><td>  572 </td></tr>
<tr><td>  512K </td><td>   6523 </td><td>  2516 </td><td>    843 </td><td>  581 </td></tr>
<tr><td>    1M </td><td>  13795 </td><td>  4313 </td><td>    836 </td><td>  611 </td></tr>
<tr><td>    2M </td><td>  26650 </td><td>  7916 </td><td>    852 </td><td>  629 </td></tr>
<tr><td>    4M </td><td>  52221 </td><td> 15160 </td><td>    959 </td><td>  695 </td></tr>
<tr><td>    8M </td><td> 110515 </td><td> 39127 </td><td>    973 </td><td>  697 </td></tr>
<tr><td>   16M </td><td> 228141 </td><td> 76980 </td><td>    952 </td><td>  668 </td></tr>
<tr><td>   32M </td><td>        </td><td>       </td><td>    892 </td><td>  685 </td></tr>
<tr><td>   64M </td><td>        </td><td>       </td><td>    887 </td><td>  673 </td></tr>
<tr><td>  128M </td><td>        </td><td>       </td><td>    962 </td><td>  748 </td></tr>
<tr><td>  256M </td><td>        </td><td>       </td><td>   1032 </td><td>  876 </td></tr>
<tr><td>  512M </td><td>        </td><td>       </td><td>   1336 </td><td> 1270 </td></tr>
<tr><td>    1G </td><td>        </td><td>       </td><td>   2146 </td><td> 1935 </td></tr>
</table>

Here are the graphs. This is for **Java 7**:

<img src="{{ site.url }}/images/life-hash-size-java7-old-and-new.png" width="634" height="355">

And this is for **Java 8**:

<img src="{{ site.url }}/images/life-hash-size-java8-old-and-new.png" width="633" height="353">

Some observations:

- The behaviour at big capacities became much better on both versions of **Java**. The graphs are nearly flat for a wide range of capacities (512 to 512M).
The slowdown was indeed caused by the `HashMap` iteration, and introducing the linked list resolved the problem.

- The **Java 8** graph is not completely flat at the ends but never goes too high, either. All the capacities are quite usable in **Java 8**.

- The behaviour at small capacities is roughly the same as before, mostly a bit slower, which is understandable: the lists do not add any benefit but require time for
their maintenance.

- The program previously ran at the capacity of 8192, which is the value the `HashMap` would have settled upon naturally (the element count is about 4000 and the load factor
if 0.75). The added linked lists help there already. In fact, the positive effect of the lists start at capacity 512 on **Java 7** and on capacity 2048 on **Java 8**.

- This means that `LinkedHashMap` is always better than ordinary `HashMap` when the map if often iterated.

- This also explains why the load factor was chosen 0.75 for the standard `HashMap`: the designers of that class didn't know if the map would be iterated, and insured themselves
against very slow execution in the case it would. Since the `LinkedHashMap` is supposed to be iterated (this is its primary purpose), it could use different default
load factor. It is strange it doesn't.

- The best performance is achieved on capacities between 64K and 2M on both systems. The absolute times (roughly 820 and 580 ms) are 20% and 36% better than the best times achieved so far
using standard capacity.

- Both graphs show one step up at about 4M and one incline at 256M. The first one can be explained by the effects of RAM caching (the processor has 20Mb of L3 cache).
The second is probably due to the garbage collector costs (the garbage collector scans the entire array to mark used blocks; scanning the empty slots also takes time).

The effect of the garbage collector
-----------------------------------

The garbage collector theory is easy to verify -- we can use the **Java** command-line option `-Xloggc:file` to print the garbage collector invocations.
This is what is printed when running on **Java 8** with the capacity of 8192 (our test program runs the simulation three times):

    0.713: [GC (Allocation Failure)  258048K->984K(989184K), 0.0020325 secs]
    1.069: [GC (Allocation Failure)  259032K->1312K(989184K), 0.0038226 secs]
    1.338: [GC (Allocation Failure)  259360K->1152K(989184K), 0.0046389 secs]
    1.665: [GC (Allocation Failure)  259200K->1010K(989184K), 0.0028201 secs]
    1.919: [GC (Allocation Failure)  259058K->1394K(989184K), 0.0069037 secs]
    2.229: [GC (Allocation Failure)  259442K->916K(1205760K), 0.0033108 secs]


Six GC invocation take 23ms in total, or 7ms per simulation, or roughly 1% of total execution time, which is reasonable.

Now let's look at the execution with the capacity 1G:

    1.761: [GC (Allocation Failure)  4452352K->4195032K(5184000K), 0.7945753 secs]
    3.367: [GC (Allocation Failure)  4453080K->4195216K(5442048K), 0.7838357 secs]
    5.528: [GC (Allocation Failure)  4711312K->4195120K(5442048K), 0.7899669 secs]

<ul>
[A side note: I still can't get used to the heap size of 4Gb; I know these days it isn't even considered big, but I still remember the system in development of which I
once took part: its heap was 8K bytes long, or half a million times smaller than now. And this was as recently as 1982! End of the side note]
</ul>
     
Here we have three invocations, averaging to 790 ms, which is 41% of total execution time (1935 ms). This means that GC does indeed play a role in the slowdown at
super-big capacities.

Here is the graph (in a logarithmic scale) of the total time taken by the GC.

<img src="{{ site.url }}/images/life-hash-size-gctime.png" width="649" height="372">

When the capacity is small, the time varies, while staying relatively small compared to the total execution time. Starting at about 1M elements the GC time grows steadily.

This graph shows the absolute values of the total time, the GC time and the pure time (the former minus the latter):

<img src="{{ site.url }}/images/life-hash-size-time-with-and-without-gc.png" width="652" height="375">

Grows in GC time can't explain the entire observed increase in total time, but it is responsible for a significant portion of it.

The tree map
------------

We mentioned earlier that **Java 8** uses binary trees to represent the chains of `HashMap` entries that occupy the same slot in the array. What if we replace the `HashMap` 
with a `TreeMap`? Let's try it (see [`TreeLife.java`]({{ site.REPO-LIFE }}/blob/2bd59fb0416dc4e0351604ed4b9f75d42f3981fd/TreeLife.java)).

Here are the results:

- on **Java 7**: 2499 ms
- on **Java 8**: 2841 ms.

Strangely, the result on **Java 8** is worse than on **Java 7** and worse than when using `HashMap` of capacity 1.
In any case, the results are bad enough to dismiss the idea.

The best results achieved so far were:

- On **Java 7**: 820 ms (achieved on the capacity of 64K)
- On **Java 8**: 572 ms (achieved on the capacity of 256K).

In general, all capacity values between 64K and 1M give good results.

Conclusions
-----------

- The standard capacity of the `HashMap` was chosen with the general use case in mind, which involves occasional iteration.

- Still, if the hash map is iterated often, it is a good idea to rather use the `LinkedHashMap`.

- If the hash map is not iterated, or if the `LinkedHashMap` is used, it helps to increase the hash map capacity above the standard one; just don't overdo it.

- We've improved the speed by another 20% on **Java 7** and 36% on **Java 8**.

- The values we started at in the very beginning were 2763 ms for **Java 7** and 4961 for **Java 8**. The overall improvement is 70% for **Java 7** and 88%
(yes, the 8.5 times speedup) for **Java 8**.

- The number of iterations per second reached 12K on **Java&nbsp;7** and 17K on **Java&nbsp;8**.

Coming soon
-----------

In the next article we'll check if we can improve speed even more.
