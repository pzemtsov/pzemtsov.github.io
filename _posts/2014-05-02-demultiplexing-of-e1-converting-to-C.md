---
layout: post
title:  "De-multiplexing of E1 stream: converting to C"
date:   2014-05-02 00:00:00
tags: optimisation performance de-multiplexing C C++ macro meta-programming
---

In the [previous article](http://pzemtsov.github.io/2014/04/14/demultiplexing-of-e1.html)
we were de-multiplexing the **[E1 stream](http://en.wikipedia.org/wiki/E-carrier#E1_frame_structure)** in **Java**.
Now I want to try **C**/**C++**. General expectation is that it will be faster
than in **Java**, possibly much faster; we'll check if this is true.

By "**C**/**C++**" I mean "**C** with a little bit of **C++**". There will be just enough **C++** to make a program look a bit nicer
and add a bit of safety.
I mean small features, such as declaring of variable where it is first used rather that at the top. Perhaps occasional
class here and there. There won't be any [multiple inheritance](http://en.wikipedia.org/wiki/Multiple_inheritance)
or [curiously recurring template pattern](http://en.wikipedia.org/wiki/Curiously_recurring_template_pattern),
and the reader won't be forced to google the difference between [static](http://en.wikipedia.org/wiki/Static_cast)
and [dynamic](http://en.wikipedia.org/wiki/Dynamic_cast) cast.

It seems natural to convert the **Java** code to **C++** directly first, to compare the compilers and execution environments.
This is what I'm going to do today; I'll leave using **C**-specific low-level features for later articles.
As a result, today's discussion will be more about syntax of **C**/**C++** than about optimisation. In particular, you will see
lots of macro definitions and invocations. If you are not interested in it or if you are already a **C** expert,
you are welcome to jump straight to the "**Running it**" section.

Conversion of non-unrolled versions
-----------------------------------

The syntax of **Java** is close enough to that of **C++** that we can almost convert one to another using
find-and-replace. I will only give one example, the rest can be seen in the repository.

**Java** code:

{% highlight Java %}
interface Demux
{
    public void demux (byte[] src, byte[][] dst);
}

static final class Reference implements Demux
{
    public void demux (byte[] src, byte[][] dst)
    {
        assert src.length % NUM_TIMESLOTS == 0;

        int dst_pos = 0;
        int dst_num = 0;
        for (byte b : src) {
            dst [dst_num][dst_pos] = b;
            if (++ dst_num == NUM_TIMESLOTS) {
                dst_num = 0;
                ++ dst_pos;
            }
        }
    }
};
{% endhighlight %}

**C++** code:

{% highlight C++ %}
typedef unsigned char byte;

class Demux
{
public:
    virtual void demux (const byte * src, unsigned src_length, byte ** dst) const = 0;
};

class Reference : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        assert (src_length % NUM_TIMESLOTS == 0);

        unsigned dst_pos = 0;
        unsigned dst_num = 0;
        for (unsigned src_pos = 0; src_pos < src_length; src_pos++) {
            dst [dst_num][dst_pos] = src [src_pos];
            if (++ dst_num == NUM_TIMESLOTS) {
                dst_num = 0;
                ++ dst_pos;
            }
        }
    }
};
{% endhighlight %}

Since we're not using class libraries, the buffers are represented as `byte` pointers, which, unlike **Java**'s byte arrays,
do not keep the data size and do not provide index checking. That's why the input length must be passed as an additional
parameter. It is the caller's responsibility to ensure that memory is correctly allocated for the output buffers.

I'm using `unsigned`, rather than `int` for lengths and loop variables out of habit: in **C++** the sizes are
traditionally stored in variables of type `size_t`, which is unsigned.

Another habit explains declaration of type `byte`. I prefer reserving the name `char` for character data, and here we
are dealing with bytes of general nature.

We also need a timer (replacement for **Java**'s `System.currentTimeMillis ()`). I've moved it to separate header file
`"timer.h"`. We'll copy the measurement and correctness test code from **Java** with only minor changes.
We don't need to run test five times, since **C++** doesn't have any warm-up effect. We'll still run it twice,
just to be safe. Another change is triggered by abcense of a garbage collection in **C++**. All the memory we allocate
must be freed. Finally, we'll run all out versions at once, so to improve stability of results, we'll run them in
the same memory, re-using the buffers.

We are ready to run the tests, but let us convert unrolled versions first.

Conversion of unrolled versions: macro magic
--------------------------------------------

We could convert unrolled versions just as we did all the others, by find and replace. However, in **C**/**C++** we can do better,
since **C**/**C++** have a rudimental meta-programming feature: macros.

This is how Unrolled_1 can be written down using macros:

{% highlight C++ %}

#define MOVE_BYTE(i,j) d[i] = src [(j)+(i)*32]

#define MOVE_BYTES_64(j) do {\
        MOVE_BYTE ( 0, j); MOVE_BYTE ( 1, j); MOVE_BYTE ( 2, j); MOVE_BYTE ( 3, j);\
        MOVE_BYTE ( 4, j); MOVE_BYTE ( 5, j); MOVE_BYTE ( 6, j); MOVE_BYTE ( 7, j);\
        MOVE_BYTE ( 8, j); MOVE_BYTE ( 9, j); MOVE_BYTE (10, j); MOVE_BYTE (11, j);\
        MOVE_BYTE (12, j); MOVE_BYTE (13, j); MOVE_BYTE (14, j); MOVE_BYTE (15, j);\
        MOVE_BYTE (16, j); MOVE_BYTE (17, j); MOVE_BYTE (18, j); MOVE_BYTE (19, j);\
        MOVE_BYTE (20, j); MOVE_BYTE (21, j); MOVE_BYTE (22, j); MOVE_BYTE (23, j);\
        MOVE_BYTE (24, j); MOVE_BYTE (25, j); MOVE_BYTE (26, j); MOVE_BYTE (27, j);\
        MOVE_BYTE (28, j); MOVE_BYTE (29, j); MOVE_BYTE (30, j); MOVE_BYTE (31, j);\
        MOVE_BYTE (32, j); MOVE_BYTE (33, j); MOVE_BYTE (34, j); MOVE_BYTE (35, j);\
        MOVE_BYTE (36, j); MOVE_BYTE (37, j); MOVE_BYTE (38, j); MOVE_BYTE (39, j);\
        MOVE_BYTE (40, j); MOVE_BYTE (41, j); MOVE_BYTE (42, j); MOVE_BYTE (43, j);\
        MOVE_BYTE (44, j); MOVE_BYTE (45, j); MOVE_BYTE (46, j); MOVE_BYTE (47, j);\
        MOVE_BYTE (48, j); MOVE_BYTE (49, j); MOVE_BYTE (50, j); MOVE_BYTE (51, j);\
        MOVE_BYTE (52, j); MOVE_BYTE (53, j); MOVE_BYTE (54, j); MOVE_BYTE (55, j);\
        MOVE_BYTE (56, j); MOVE_BYTE (57, j); MOVE_BYTE (58, j); MOVE_BYTE (59, j);\
        MOVE_BYTE (60, j); MOVE_BYTE (61, j); MOVE_BYTE (62, j); MOVE_BYTE (63, j);\
    } while (0)

#define MOVE_TIMESLOT(j) do {\
        byte * const d = dst[j];\
        MOVE_BYTES_64 (j);\
    } while (0)

class Unrolled_1 : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        assert (NUM_TIMESLOTS == 32);
        assert (DST_SIZE == 64);
        assert (src_length == NUM_TIMESLOTS * DST_SIZE);

        for (unsigned j = 0; j < NUM_TIMESLOTS; j++) {
            MOVE_TIMESLOT (j);
        }
    }
};
{% endhighlight %}

Here I feel necessary to include some practical points on macro usage. Readers with good **C** experience can skip the next section:

  - If a macro has limited lifetime, you should `#undef` it as soon as it becomes unnecessary; this can save you from
    accidentally declaring macro with the same name later. This is especially applicable to temporary macros in
    include files. In this example I didn't `#undef` the macros, because they will be used in other functions.

  - If a macro is only used inside one function, it is a good idea to `#define` it right inside that function and
    `#undef` it after the use.

  - Don't forget that no white space is allowed between the macro name and open parenthesis. If there is any whitespace
    there, the parenthesis together with the argument list will be considered part of a macro body.

  - The backslash (`'\'`) at the end of a line indicates that a macro is continued on the next line.
    But for that it must really be at the end of the line, no white space is allowed after it
    (although some compilers allow that).

  - It is always a good idea to wrap the arguments into round brackets inside macro definition (see the `MOVE_BYTE`
    definition). You never know what will come as parameters. Imagine the brackets were not there and the macro was
    invoked like this:

{% highlight C++ %}
    MOVE_BYTE (x+1, y);
{% endhighlight %}

It would have been expanded as

{% highlight C++ %}
    d[x+1] = src[y+x+1*32]
{% endhighlight %}

and this is not quite what was intended.

  - For those who do not know the significance of a "do" statement in macro definitions, I'll explain it in one of the
    following articles.

Here the usage of macros does not save you much typing - you must still invoke `MOVE_BYTE` manually 64 times.
But at least the actual code of `MOVE_BYTE` is in one place only, and can be modified easily.

By the way, you can avoid writing down all 64 invocations of `MOVE_BYTE` by creating multi-layered macros:

{% highlight C++ %}
#define MOVE_BYTES_4(i,j) do {\
  MOVE_BYTE(i,j); MOVE_BYTE(i+1,j); MOVE_BYTE(i+2,j); MOVE_BYTE(i+3,j);\
} while (0)
#define MOVE_BYTES_16(j) do {\
  MOVE_BYTES_4(i,j); MOVE_BYTES_4(i+4,j); MOVE_BYTES_4(i+8,j); MOVE_BYTES_4(i+12,j);\
} while (0)
#define MOVE_BYTES_64(j) do {\
  MOVE_BYTES_16(0,j); MOVE_BYTES_16(16,j); MOVE_BYTES_16(32,j); MOVE_BYTES_16(48,j);\
} while (0)
{% endhighlight %}

This way you have to write 12 invocations of some macros rather than 64. The same number will result from de-composition
into six layers of two invocations each. A general formula for the minimal number of invocations for a given total
number doesn't seem to be simple; perhaps some mathematically inclined reader can suggest one.

The macros we've just created will be useful for other versions of our method, too. For instance, a four times unrolled version looks like this:

{% highlight C++ %}
class Unrolled_1_4 : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        assert (NUM_TIMESLOTS == 32);
        assert (DST_SIZE == 64);
        assert (src_length == NUM_TIMESLOTS * DST_SIZE);

        for (unsigned j = 0; j < NUM_TIMESLOTS; j+=4) {
            MOVE_TIMESLOT (j);
            MOVE_TIMESLOT (j+1);
            MOVE_TIMESLOT (j+2);
            MOVE_TIMESLOT (j+3);
        }
    }
};
{% endhighlight %}

And fully unrolled version looks like this:

{% highlight C++ %}
class Unrolled_2_Full : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        assert (NUM_TIMESLOTS == 32);
        assert (DST_SIZE == 64);
        assert (src_length == NUM_TIMESLOTS * DST_SIZE);

        MOVE_TIMESLOT ( 0); MOVE_TIMESLOT ( 1); MOVE_TIMESLOT ( 2); MOVE_TIMESLOT ( 3);
        MOVE_TIMESLOT ( 4); MOVE_TIMESLOT ( 5); MOVE_TIMESLOT ( 6); MOVE_TIMESLOT ( 7);
        MOVE_TIMESLOT ( 8); MOVE_TIMESLOT ( 9); MOVE_TIMESLOT (10); MOVE_TIMESLOT (11);
        MOVE_TIMESLOT (12); MOVE_TIMESLOT (13); MOVE_TIMESLOT (14); MOVE_TIMESLOT (15);
        MOVE_TIMESLOT (16); MOVE_TIMESLOT (17); MOVE_TIMESLOT (18); MOVE_TIMESLOT (19);
        MOVE_TIMESLOT (20); MOVE_TIMESLOT (21); MOVE_TIMESLOT (22); MOVE_TIMESLOT (23);
        MOVE_TIMESLOT (24); MOVE_TIMESLOT (25); MOVE_TIMESLOT (26); MOVE_TIMESLOT (27);
        MOVE_TIMESLOT (28); MOVE_TIMESLOT (29); MOVE_TIMESLOT (30); MOVE_TIMESLOT (31);
    }
};
{% endhighlight %}

This is much smaller in source size than the version in **Java**.

A sad comment on meta-programming
---------------------------------

Meta-programming is when some part of a program is executed at compile-time, producing real program, which is then
compiled into the executable code. As you could see in the previous section, it is very useful feature for
high-performance programming, which involves lots of inlining and unrolling. Unfortunately, such a feature
is a rare commodity in modern programming languages. **Java** does not have it (generics do not count and dynamic
code generation is quite troublesome and heavy-weight), neither does **Go** or **Python**. **C** has a rudimentary version
of it in the form of macro expansion. In addition, **C++** employs templates, which are very powerful but not easy
to use and a bit ugly. They are not actually designed for full-scale meta-programming, but rather for generic
type-independent programming, that's why the primary working mechanism there is pattern matching, which for
me isn't the programming tool of choice. It is not impossible to implement the above examples using templates
but the code won't look nice and it is yet to be seen how fast it would run.

In the old days meta-programming was more common. It was widely available in macro-assemblers, where main language
(assembler) was enriched with macro-features, which included variables, loops, branches and other high-level features.
But the best by far in that respect was the approach of **PL/I**, where macro-language was **PL/I** itself, just the lines were
marked to be run at compile time. Should such a feature be available in **C++**, out program would have looked much nicer:

{% highlight C++ %}
class Unrolled_2_Full : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        assert (src_length % NUM_TIMESLOTS == 0);
        %for (unsigned %dst_num = 0; %dst_num < NUM_TIMESLOTS; ++ %dst_num) {
            %for (unsigned %dst_pos = 0; %dst_pos < DST_SIZE; ++ %dst_pos) {
                dst [%dst_num][%dst_pos] = src [%dst_pos * NUM_TIMESLOTS + %dst_num];
            %}
        %}
    }
};
{% endhighlight %}

I assume imaginary version of **C++** where percentage mark indicates code that runs at compile time. As you can see,
the code looks very close to `Dst_First_1`. Unfortunately, such a version does not exist, and we have to write ugly
macro definitions. What a pity.

More macro magic: parameter concatenation
-----------------------------------------

In our **Java** code there is a version that involves 32 small functions (`Unrolled_3`). Quite frankly,
I don't expect its performance to be any better than performance of other unrolled versions in **C++**;
in addition you can recall that the very reason for its existence was HotSpot limitation of method size.
However, for the sake of completeness I'll convert this function as well, using another handy feature of
**C**'s macro language: token concatenation. A hash character (`'#'`), used in macro, causes concatenation
of text to the left and to the right of it without any whitespaces, allowing creation of new objects such
as function names.

{% highlight C++ %}
class Unrolled_3 : public Demux
{
public:
    void demux (const byte * src, unsigned src_length, byte ** dst) const
    {
        assert (src_length % NUM_TIMESLOTS == 0);

#define CALL_DEMUX(i) demux_##i (src, dst[i])

        CALL_DEMUX ( 0); CALL_DEMUX ( 1); CALL_DEMUX ( 2); CALL_DEMUX ( 3);
        CALL_DEMUX ( 4); CALL_DEMUX ( 5); CALL_DEMUX ( 6); CALL_DEMUX ( 7);
        CALL_DEMUX ( 8); CALL_DEMUX ( 9); CALL_DEMUX (10); CALL_DEMUX (11);
        CALL_DEMUX (12); CALL_DEMUX (13); CALL_DEMUX (14); CALL_DEMUX (15);
        CALL_DEMUX (16); CALL_DEMUX (17); CALL_DEMUX (18); CALL_DEMUX (19);
        CALL_DEMUX (20); CALL_DEMUX (21); CALL_DEMUX (22); CALL_DEMUX (23);
        CALL_DEMUX (24); CALL_DEMUX (25); CALL_DEMUX (26); CALL_DEMUX (27);
        CALL_DEMUX (28); CALL_DEMUX (29); CALL_DEMUX (30); CALL_DEMUX (31);
#undef CALL_DEMUX
    }

private:

#define DEF_DEMUX(i) \
    inline void demux_##i (const byte * src, byte * d) const\
    {\
        MOVE_BYTES (i);\
    }

    DEF_DEMUX ( 0) DEF_DEMUX ( 1) DEF_DEMUX ( 2) DEF_DEMUX ( 3)
    DEF_DEMUX ( 4) DEF_DEMUX ( 5) DEF_DEMUX ( 6) DEF_DEMUX ( 7)
    DEF_DEMUX ( 8) DEF_DEMUX ( 9) DEF_DEMUX (10) DEF_DEMUX (11)
    DEF_DEMUX (12) DEF_DEMUX (13) DEF_DEMUX (14) DEF_DEMUX (15)
    DEF_DEMUX (16) DEF_DEMUX (17) DEF_DEMUX (18) DEF_DEMUX (19)
    DEF_DEMUX (20) DEF_DEMUX (21) DEF_DEMUX (22) DEF_DEMUX (23)
    DEF_DEMUX (24) DEF_DEMUX (25) DEF_DEMUX (26) DEF_DEMUX (27)
    DEF_DEMUX (28) DEF_DEMUX (29) DEF_DEMUX (30) DEF_DEMUX (31)
#undef DEF_DEMUX
};
{% endhighlight %}

You can see how using `'#'` helps create necessary method names. Note abcense of semicolons between calls to `DEF_DEMUX`:
in **C**/**C++** function definitions do not end with semicolons (but class definitions do!)

Making things shorter: macros as macro parameters
-------------------------------------------------

Our code contains manually written sequences of macro calls of size 64 (as a `MOVE_BYTES` definition), one each of size 2, 4, 8
and 16 (various `Unrolled_1_x` versions) and four of size 32 (all fully unrolled versions).
A code can be made a bit shorter by introducing a generic macro duplicator - a poor man's meta-programming loop.
Such a duplicator (also a macro) can take a macro as a parameter and call it given (constant) number of times.
This is based on a very handy feature of macros: although a macro with parameters can't be invoked with different
number of parameters, using its name without any parameters at all is allowed, the macro won't be expanded then.
This allows passing name of one macro as a parameter to another macro.

There are three ways to write such a multiplier. The simplest (but the longest) involves a lot of typing:

{% highlight C++ %}
#define DUP_2(m)  do { m(0); m(1); } while (0)
#define DUP_4(m)  do { m(0); m(1); m(2); m(3); } while (0)
#define DUP_8(m)  do { m(0); m(1); m(2); m(3); m(4); m(5); m(6); m(7); } while (0)
{% endhighlight %}

And so on - definition of DUP_64 will contain 64 calls to m().

We can save a bit of typing by basing higher-order duplicator on lower-order one. A partial solution looks like this:

{% highlight C++ %}
#define DUP_2(m)  do {            m(0); m(1); } while (0)
#define DUP_4(m)  do { DUP_2(m);  m(2); m(3); } while (0)
#define DUP_8(m)  do { DUP_4(m);  m(4); m(5); m(6); m(7); } while (0)
{% endhighlight %}

Here the lists of m() being invoked are exactly half the size.

Another approach is to call each previous macro twice:

{% highlight C++ %}
#define DUP_2_(m, index)  do { m (index); m (index+1); } while (0)
#define DUP_4_(m, index)  do { DUP_2_ (m, index); DUP_2_ (m, index+2); } while (0)
#define DUP_8_(m, index)  do { DUP_4_ (m, index); DUP_4_ (m, index+4); } while (0)

#define DUP_2(m)  DUP_2_ (m, 0)
#define DUP_4(m)  DUP_4_ (m, 0)
#define DUP_8(m)  DUP_8_ (m, 0)
{% endhighlight %}

This approach has two significant advantages. It allows writing down loops of arbitrary sizes without much typing
(a macro definition for any loop of size that is power of two has exactly two macro invocations; for other loop
sizes the number may be bigger but is always limited by number of bits in the loop size). It also provides,
as a by-product, a set of loops that don't start at zero but at some given number, and we need these for our partially
unrolled loop solutions. It has, however, one disadvantage. The indices in macro expansions appear as arithmetic
expressions rather than as direct numbers; and this makes it impossible to use this macro with `CALL_DEMUX` above:
the `'#'` concatenation operation does not evaluate expressions before concatenation. That's why we will use the
partial solution.

Neither of these approaches works for `DEF_DEMUX`, because `DEF_DEMUX` does not allow semicolons between calls.

The macros above only work with argument macros that take one parameter. For more parameters, separate series of
macros must be created. I called them `DUP2_xx`.

Using this macros, for instance, the sequence of `CALL_DEMUX` in the previous example (`Unrolled_3`) will become just

{% highlight C++ %}
    DUP_32 (CALL_DEMUX)
{% endhighlight %}

Partially unrolled methods require service macro that relies on the name of the loop variable (`j`) and adds it to the
offset parameter:

What is nice is that `DUP_xx` macros are now generic and can be moved to a separate header file and later re-used
in other projects. I've moved them to `mymacros.h` file.

As a result of applying macros and multipliers the entire size of the test program became 383 lines, while in Java
just the `Unrolled_2_Full` was 553 lines long.

Running it
----------

We are ready to run it all. We'll be using the same system as for the **Java** testing: Linux, running on a Dell
blade with Intel(R) Xeon(R) CPU E5-2670 @ 2.60GHz. We'll use gcc 4.6.

    $ c++ -O3 -o e1 e1.cpp -lrt

(`-O3` indicates the highest level of optimisation, and `-lrt` is needed to include relevant run-time library).

    $ ./e1
    9Reference: 1937 1930
    11Src_First_1: 1890 1878
    11Src_First_2: 1920 1916
    11Src_First_3: 1890 1891
    11Dst_First_1: 1465 1465
    11Dst_First_2: 1443 1444
    11Dst_First_3: 1758 1758
    10Unrolled_1: 632 633
    12Unrolled_1_2: 633 633
    12Unrolled_1_4: 655 655
    12Unrolled_1_8: 648 649
    13Unrolled_1_16: 635 633
    15Unrolled_2_Full: 635 634
    10Unrolled_3: 634 634
    10Unrolled_4: 653 653

You might be surprised by the names in the first columns; so am I. We are lucky that we can read it at all.
The **C++** standard does not require any particular format of a type name as returned by `typeid` call, so
the class name prepended by the number of characters in it isn't the worst possible case.

It is clearly visible that the results for two runs are quite stable. It means that running the test twice is unnesessary
and we can remove appropriate code.

    9Reference: 1939
    11Src_First_1: 1885
    11Src_First_2: 1924
    11Src_First_3: 1892
    11Dst_First_1: 1467
    11Dst_First_2: 1445
    11Dst_First_3: 1761
    10Unrolled_1: 633
    12Unrolled_1_2: 634
    12Unrolled_1_4: 654
    12Unrolled_1_8: 650
    13Unrolled_1_16: 635
    15Unrolled_2_Full: 635
    10Unrolled_3: 635
    10Unrolled_4: 655

Let's recall the results for **Java** version and put them together in the same table:

<table>
<tr><th>     Method               </th> <th>  Results in Java </th> <th> Results in C</th></tr>
<tr><td><pre>Reference      </pre></td> <td><pre>  2860 </pre></td> <td><pre> 1939</pre></td></tr>
<tr><td><pre>Src_First_1    </pre></td> <td><pre>  2481 </pre></td> <td><pre> 1885</pre></td></tr>
<tr><td><pre>Src_First_2    </pre></td> <td><pre>  2284 </pre></td> <td><pre> 1924</pre></td></tr>
<tr><td><pre>Src_First_3    </pre></td> <td><pre>  4360 </pre></td> <td><pre> 1892</pre></td></tr>
<tr><td><pre>Dst_First_1    </pre></td> <td><pre>  1155 </pre></td> <td><pre> 1467</pre></td></tr>
<tr><td><pre>Dst_First_2    </pre></td> <td><pre>  2093 </pre></td> <td><pre> 1445</pre></td></tr>
<tr><td><pre>Dst_First_3    </pre></td> <td><pre>  1022 </pre></td> <td><pre> 1761</pre></td></tr>
<tr><td><pre>Unrolled_1     </pre></td> <td><pre>   659 </pre></td> <td><pre>  633</pre></td></tr>
<tr><td><pre>Unrolled_1_2   </pre></td> <td><pre>   654 </pre></td> <td><pre>  634</pre></td></tr>
<tr><td><pre>Unrolled_1_4   </pre></td> <td><pre>   636 </pre></td> <td><pre>  654</pre></td></tr>
<tr><td><pre>Unrolled_1_8   </pre></td> <td><pre>   637 </pre></td> <td><pre>  650</pre></td></tr>
<tr><td><pre>Unrolled_1_16  </pre></td> <td><pre> 25904 </pre></td> <td><pre>  635</pre></td></tr>
<tr><td><pre>Unrolled_2_Full</pre></td> <td><pre> 15630 </pre></td> <td><pre>  635</pre></td></tr>
<tr><td><pre>Unrolled_3     </pre></td> <td><pre>   790 </pre></td> <td><pre>  635</pre></td></tr>
<tr><td><pre>Unrolled_4     </pre></td> <td><pre>   711 </pre></td> <td><pre>  655</pre></td></tr>
</table>

Here comes our first sensation: **C++ is not always faster than Java**. Both `Dst_First_1` and `Dst_First_3` run
quite a bit faster in **Java** than in **C++**. I'll come back to this fenomenon in a later article. Let's look at
other expectations:

- The compiler does not panic when compiling big methods; there are no cases when something ran unusually slow

- In most cases **C++** is still a bit faster than **Java**, with typical the execution time being 70-80% of **Java** time,
sometimes less (43% in `Src_First_3` and 68% in `Reference`). Some of this speed difference can be attributed to
the abcense of index checking in **C++**.

- All fully unrolled versions (`Unrolled_2_Full`, `Unrolled_3`, `Unrolled_4`) run at the same speed, so there is
no need to create complex solutions such as `Unrolled_3`

- The speed of unrolled solutions is virtually identical in **C++** and **Java**. Sometimes **Java** is a bit faster,
sometimes **C++** is.

- Relative speeds of different solutions aren't the same as in **Java**. In **Java** manual optimisation of `Dst_First_1`
produced very slow result (`Dst_First_2`), while in **C++** it even improved speed a bit, which causes suspicion
that optimiser in `gnu C++` isn't as good as one in HotSpot. `Src_First_3` didn't experience big slowdown
either - could be due to array index checking.

- The difference between Dst_First_3 and Dst_First_1 is that the input size is hard-coded in the former one,
which should improve performance. However, the opposite has happened. This is something to investigate.

Notes for myself
----------------

Some points to investigate:

- Why some **C++** versions run slower than in **Java**?

- How well does **C++** compiler optimise programs? Is manual optimisation good or bad there?

- What happens if we use vectors with index checking instead of `byte` pointers? How much will it
  affect performance?

Coming soon
-----------

Can C++ code be made faster without rewriting - perhaps there are some secret compiler keys? Can the speed be
improved any more? Answers will follow soon.
