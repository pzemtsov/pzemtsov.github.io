---
layout: post
title:  GCC, C standard and pointer aliasing
date:   2014-07-23 12:00:00
tags:   C C++ Standard aliasing 
ART-ALIASING: /2014/05/09/gcc-optimisations-and-pointer-aliasing.html
---

After [this]({{ page.ART-ALIASING }}) article had been published, I received a very valid comment. Here it is:

<img src="{{ site.url }}/images/c-standard-kentonvarda.png" width="498" height="203">

I'm very grateful to Kenton for pointing it out. His first statement is indeed correct. The last time I read
a **C** standard must have been more than twenty years ago, enough time for the information to go out of active memory,
or, simply speaking, be forgotten.

This is a good opportunity to look back and check what various standards have to say on the issue. So today instead
of sharing a programming experience, I'm going to share a learning experience, which is also good.

These are the documents I'm going to use:

- [ANSI/ISO 9899:1990 (C89)](http://read.pudn.com/downloads133/doc/565041/ANSI_ISO%2B9899-1990%2B[1].pdf)
- [The latest publicly available version of the C99 standard (2007)](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1256.pdf)
- [The C99 Rationale](http://www.open-std.org/JTC1/SC22/WG14/www/C99RationaleV5.10.pdf)
- [The latest publically available version of the C11 standard, dated 2011-04-12](http://www.open-std.org/JTC1/SC22/WG14/www/docs/n1570.pdf)
- [The latest working draft of the C++ standard, dated 2013-05-15](http://open-std.org/JTC1/SC22/WG21/docs/papers/2011/n3242.pdf)

<br>

Aliasing in GCC
---------------

Let me remind you the definition of the strict aliasing that we used in the original article:

<table>
<tr><td><pre>-fstrict-aliasing</pre></td>
<td>
Allows the compiler to assume the strictest aliasing rules applicable to the language being compiled.
For C (and C++), this activates optimizations based on the type of expressions. In particular, an object
of one type is assumed never to reside at the same address as an object of a different type, unless the
types are almost the same. For example, an unsigned int can alias an int, but not a void* or a double.
<strong>A character type may alias any other type</strong>.
</td></tr></table>

The `-fstrict-aliasing` option is off by default; it is activated as part of `-O2` optimisation suite.

This definition contains a hint that the language standard must be covering aliasing as well ("strictest aliasing
rules applicable").

Aliasing in C89
---------------

Mr. Varda is completely right: what we know as the strict aliasing rule has been introduced in the very first
**C** standard, which was commonly referred to as '**ANSI C**'. More correct name is '**ANSI/ISO 9899:1990**'.
It can be found [here](http://read.pudn.com/downloads133/doc/565041/ANSI_ISO%2B9899-1990%2B[1].pdf).
The aliasing rule is well hidden in the beginning of Chapter 6.3 (Expressions). This is what is says:

<img src="{{ site.url }}/images/c-standard-c89.png" width="694" height="323">

We can see that the Standard definition talks about the same things as GNU C description, but there are differences, too.

First of all, what is an optimisation option in GNU C, is a language rule in the Standard. And a very strict one, too.
It contains the word "**shall**", which, earlier in the Standard (3.16), is defined as a requirement whose violation
is punishable by an undefined behaviour. And of the four unusual types of behaviour that **C** defines (undefined behaviour,
unspecified behaviour, implementation-defined behaviour, and locale-specific behaviour) the undefined behaviour is
the worst:

<img src="{{ site.url }}/images/c-standard-c89-2.png" width="685" height="141">

It is quite possible to imagine an execution environment that does indeed cause program crash in case of the
wrongly-typed memory access. In the old days there were machines with
[tagged memory](http://en.wikipedia.org/wiki/Tagged_architecture) (such as various models of Burroughs),
where each memory location kept the data type of the stored value. Nowadays one may expect something like that
in interpreted virtual machines.

Fortunately, the standard does not require the crash. Moreover, as you can see above, it specifically allows the
undefined behaviour to be defined and documented, and this is exactly what GNU C does when strict aliasing is off.

Although it does not explicitly say so, the GNU C definition makes impression of a local rule, applicable within
a function or a block. If within a function there are two pointers of an unknown origin, the compiler may
make some assumptions about their aliasing, based on the types of those pointers, and then perform or not perform
related optimisations. The Standard definition sounds much more global. Once a value of a certain type has been written
to an object, from that moment and until the power-off we are only allowed to access this object as a value of this type and
a couple of related types. Even if there is exactly one pointer pointing to the object and there is no aliasing and no optimisations
in mind. In short, the Standard rule seems a bit too strict.

There is another important difference. The GNU C description says that a character type may alias any type.
But aliasing is a symmetric relation: it means that if there are two pointers, one of which is a `char*`,
the compiler must assume they may point to the same place, no matter which of them is used for reading and which
for writing. The Standard's position is different. What is initially written using non-character type can be accessed
(that is, read or written) using character type, **but not other way around**!

If my understanding of the standard-speak is correct, this must be valid:

{% highlight C++ %}
int x = 5;
char c = *(char*)&x;
{% endhighlight %}

And this must be invalid

{% highlight C++ %}
char c[4] = {1, 2, 3, 4};
int x = *(int*) c;
{% endhighlight %}

This example of writing a protocol message into a byte buffer must also be invalid:

{% highlight C++ %}
typedef struct {
    int x, y;
    float f;
} Message;

int fill (const Message * message, char * buf)
{
    int offset = 0;
    * (int*)   (buf + offset) = message->x; offset += sizeof (int);
    * (int*)   (buf + offset) = message->y; offset += sizeof (int);
    * (float*) (buf + offset) = message->f; offset += sizeof (float);
    return offset;
}
{% endhighlight %}

And this, strangely enough, must be valid:

{% highlight C++ %}
int fill (const Message * message, char * buf)
{
    int offset = 0;
    memcpy (buf + offset, &message->x, sizeof (int)); offset += sizeof (int);
    memcpy (buf + offset, &message->y; sizeof (int)); offset += sizeof (int);
    memcpy (buf + offset, &message->f; sizeof (float)); offset += sizeof (float);
    return offset;
}
{% endhighlight %}

This renders incorrect my [previous guess]({{ page.ART-ALIASING }}) on the reasons why such an exception (for a character type) existed.
I thought that was to allow efficient writing and reading of objects to and from byte (character) arrays, such as
when composing protocol messages. Now it seems that the language designers didn't intend to allow anything like that;
they were only interested in examining the byte content of objects and manipulation that byte content,
including copying the objects as byte blobs from one place in memory to another.
This is expressed in the [C99 Rationale](http://www.open-std.org/JTC1/SC22/WG14/www/C99RationaleV5.10.pdf):

<p style="font-family:Times;padding-left:50px;">
Character pointer types are often used in the bytewise manipulation of objects; a byte
stored through such a character pointer may well end up in an object of any type.
</p>
<br>

There is another interesting observation about this: obviously, bytewise manipulations of objects (even reading, never mind
writing) are not portable, which means that "standard-compliant" is not equal to "portable". Portable programming in **C** is
possible, but this is a special discipline; just following the standard is not enough.



Aliasing in C99
---------------

The C99 standard has modified the definition a bit. Previously all the objects had _declared types_. That included
allocated objects, whose declared types were assumed equal to the types of l-values used to access them. Now allocated objects
lost declared type, which was replaced with an _effective type_:

   <p style="font-family:Times;padding-left:50px;">
   The effective type of an object for an access to its stored value is the declared type of the
   object, if any.<sup>75</sup>) If a value is stored into an object having no declared type through an
   lvalue having a type that is not a character type, then the type of the lvalue becomes the
   effective type of the object for that access and for subsequent accesses that do not modify
   the stored value. If a value is copied into an object having no declared type using
   <code>memcpy</code> or <code>memmove</code>, or is copied as an array of character type, then the effective type
   of the modified object for that access and for subsequent accesses that do not modify the
   value is the effective type of the object from which the value is copied, if it has one. For
   all other accesses to an object having no declared type, the effective type of the object is
   simply the type of the lvalue used for the access.
   <br>
   <sup>75)</sup> Allocated objects have no declared type.
   </p>

This modification explicitly allows copying objects using library functions and character arrays.
The following becomes a valid code:

{% highlight C++ %}
char * p = malloc (sizeof (int));
char * q = malloc (sizeof (double));
int x = 5;
double f = 3.5;
int i;

memcpy (p, &x, sizeof (int));
for (i = 0; i < sizeof (double); i++)
    q[i] = ((char*) &f)[i];
{% endhighlight %}

The `memcpy` and the loop give the objects pointed to by `p` and `q` effective types `int` and `double`
respectively. After this the following access is valid:

{% highlight C++ %}
*(int*) p           /* correct effective type */
*(unsigned int*) p  /* unsigned version of the effective type */
*(double*) q        /* correct effective type */
*p                  /* character type is always welcome */
*q                  /* character type is always welcome */
{% endhighlight %}

and this is invalid:

{% highlight C++ %}
*(int*) q
*(double*) p
{% endhighlight %}

The objects will keep these effective types until the stored values are modified. But since they can only be modified
by values of restricted set of types, the new effective types are limited. Object pointed to by `p` has effective type
`int`. It can assume effective type `unsigned`, but not `float` or `void *`.

There are some points in this definition that are unclear to me. One is the use of character arrays. I've made the
example involving copying bytes in a loop, but I'm not sure myself it is correct. The standard talks about
value "copied as an array of character type", as something other than using `memcpy` or `memmove`, 
but it isn't clear if it means a loop like mine.

The other point is what happens if we modify a single byte of a value of non-character type. Consider the pointer `p`
from the example above; we know that, although `p` is declared as `char*`, `*p` has the effective type `int`, because
`int` was the first thing written to `*p` after it was allocated. This remains its effective type for all subsequent
accesses that do not modify the stored value. What happens when the stored value is modified using a `char*`
pointer? One can interpret the standard so that the effective type will remain `int`, or so that the effective type
will be lost completely, as if the object was released and allocated again.

Finally, the effective type is only set if the value is stored using lvalue of non-character type. Does that mean that
elements of an allocated character array do not have effective types?

Aliasing in C++
---------------

The **C++** definition (see [here](http://open-std.org/JTC1/SC22/WG21/docs/papers/2011/n3242.pdf), paragraph 3.10)
adjusts the rule for inheritance (it is possible to access an object using one of its
base classes) and removes references to effective types and to library functions. This is understandable,
because an arbitrary **C++** object can't be simply copied using `memcpy`. Someone must initialise the virtual table
pointer and call the appropriate constructor. For the same reason creating an object by means of an array copy
operation is not encouraged. However, the desire to analyse the byte structure of the objects still exists, as does
the clause on the use of a character type. Apart from this, there is nothing new in **C++** aliasing rule.

Aliasing rule and our **C++** program
-------------------------------------

Here I'm referring to the E1 de-multiplexer, which was [written in **Java**]({{ site.ART-E1 }}),
then [converted to **C++**]({{ site.ART-E1-C }}), and later [optimised using SSE]({{ site.ART-E1-C-SSE }}). 

The original solutions (described in ["{{ site.TITLE-E1-C }}"]({{ site.ART-E1-C }})) were completely correct with respect to the aliasing rules, and could be
run on any machine, including mentioned tagged architectures. The bytes were copied across one at a time, the only
influence the aliasing had on these solution was that the compiler avoided some important optimisations.
We had to perform those optimisations manually.

The more optimised solutions (described in ["{{ site.TITLE-E1-C-SSE }}"]({{ site.ART-E1-C-SSE }}) are much more tricky. Very common element of these solution is
reading or writing of 4, 8, 16 or 32 consecutive bytes to or from a character array using `int`, `long`, or SSE pointers.

If my understanding of the Standard is correct, a dynamically allocated object gets created when its value is
written for the first time. This is when it acquires its type -- it is the type of the l-value used to write the
object. Our source data is a `byte` (`char`) array. Assuming it was filled in
byte by byte (as is the case in our test code), using `char*` pointers, each element of this array (a character) is an object, and it
can't be read using an `int*` pointer. Or, for that matter, using a `__m128i*` pointer.

The last part is especially important, because almost any SSE program that works on integer data loads this data
into the SSE registers using `_mm_load_si128`, which takes parameter of type `const __m128i *`, so it has to convert
whatever pointer to the source data it has to that type. As a result, any such program violates the **C** standard.
This sounds scary, but in reality it is not. Programming with SSE is nonportable by definition, so, instead of
following the Standard strictly, we should rather try to satisfy the limitations of our target compilers.
In this case our target compiler is GNU C, and it allows reading and writing from aliased memory without any problems
(except for some performance penalties because you have to switch off certain optimisations).

It looks odd, but it seems that the Standard is much more tolerant to writing big objects to character
arrays than to reading. When we are reading an `int` from a `char` array, we're violating the standard. But if we are
writing an `int` to a freshly allocated `char` array, we are not violating it -- we a creating a new object, of type `int`.
As a result, our output array, being declared as `char *`, in reality consists of objects of type `int` (`long`, or `__m128i`),
which is perfectly legal.

If the customer of our demodulation function wants to read the output arrays byte by byte, he is free to do so,
since the Standard allows reading complex objects byte by byte. He is also free to read the data using the same types
we used to fill the arrays. But if he tries reading the data using other types, he'll be violating the Standard. 

It does sound like a paradox -- that we can write output in funny ways but are not allowed to read input similarly --
but it seems to be in accordance of the Standard.

Using `restrict` keyword
------------------------

Another interesting topic of the Standard is the `restrict` keyword. We've tried to use `__restrict__` qualifiers
before, and found out two unpleasant things. One was that in order to mark two pointers as unaliased, both must
be declared as `__restrict__`, while the definition seemed to require only one such declaration.
The definition in question is the following:

  <p style="padding-left:50px;">
  Declaring a pointer as `__restrict__` indicates that within current
  block the object pointed to by this pointer can only be accessed via this pointer or derived one.
  </p>

The other problem was that there seemed to be no way to eliminate aliasing of a `char**` pointer with its own
targets (`char*` pointers).

The good news is that since 1999 the `restrict` keyword (without the underscores) became part of the C99 standard,
so we can check the exact definition there. Please note that this keyword is only part of the **C** standard, not
of **C++**.

Here is the definition:

  <p style="font-family:Times;padding-left:50px;">
  <b>6.7.3.1 Formal definition of <code>restrict</code></b>
  <br>
  <br>
  1 Let <code><b>D</b></code> be a declaration of an ordinary identifier that provides a means of designating an
  object <code><b>P</b></code> as a restrict-qualified pointer to type <code><b>T</b></code>.
  <br>
  <br>
  2 If <code><b>D</b></code> appears inside a block and does not have storage class <code><b>extern</b></code>, let <code><b>B</b></code> denote the
  block. If <code><b>D</b></code> appears in the list of parameter declarations of a function definition, let <code><b>B</b></code>
  denote the associated block. Otherwise, let <code><b>B</b></code> denote the block of <code><b>main</b></code> (or the block of
  whatever function is called at program startup in a freestanding environment).
  <br>
  <br>
  3 In what follows, a pointer expression <code><b>E</b></code> is said to be _based_  on object <code><b>P</b></code> if (at some
  sequence point in the execution of <code><b>B</b></code> prior to the evaluation of <code><b>E</b></code>) modifying <code><b>P</b></code> to point to
  a copy of the array object into which it formerly pointed would change the value of <code><b>E</b></code>.<sup>119</sup>)
  Note that '_based_' is defined only for expressions with pointer types.
  <br>
  <br>
  4 During each execution of <code><b>B</b></code>, let <code><b>L</b></code> be any lvalue that has <code><b>&L</b></code> based on <code><b>P</b></code>.
  If <code><b>L</b></code> is used to
  access the value of the object <code><b>X</b></code> that it designates, and <code><b>X</b></code> is also modified (by any means),
  then the following requirements apply: <code><b>T</b></code> shall not be const-qualified. Every other lvalue
  used to access the value of <code><b>X</b></code> shall also have its address based on <code><b>P</b></code>. Every access that
  modifies <code><b>X</b></code> shall be considered also to modify <code><b>P</b></code>, for the purposes of this subclause. If <code><b>P</b></code>
  is assigned the value of a pointer expression <code><b>E</b></code> that is based on another restricted pointer
  object <code><b>P2</b></code>, associated with block <code><b>B2</b></code>, then either the execution of <code><b>B2</b></code> shall begin before
  the execution of <code><b>B</b></code>, or the execution of <code><b>B2</b></code> shall end prior to the assignment. If these
  requirements are not met, then the behavior is undefined.
  <br>
  <br>
  5 Here an execution of <code><b>B</b></code> means that portion of the execution of the program that would
  correspond to the lifetime of an object with scalar type and automatic storage duration
  associated with <code><b>B</b></code>.
  <br>
  <br>
  <sup>119</sup>) In other words, <code><b>E</b></code> depends on the value of <code><b>P</b></code> itself rather than on the value of an object referenced
  indirectly through <code><b>P</b></code>. For example, if identifier `p` has type `(int **restrict)`, then the pointer
  expressions `p` and `p+1` are based on the restricted pointer object designated by `p`, but the pointer
  expressions `*p` and `p[1]` are not.
  </p>

This definition really looks impressive; it is a masterpiece of a standard-speak. A single passage like this
makes it worth reading the entire Standard. It takes a while to read the
whole page full of bold latin-alphabet symbols, but when you finish, you will probably see that the meaning of this definition
is exactly the same as that of the single line above. I don't say "probably" because I doubt your ability to
understand the above text -- I say it because I doubt my own ability to understand it.

The definition allows specifying one pointer to be `restrict`. This guarantees that no one can access its destination,
except for its derived pointers, so the compiler knows exactly what can and what cannot alias its destination.
The reason the compiler requires two pointers to be both declared `restricted` in order to prevent aliasing is still
unknown. All the examples in [The C99 Rationale](http://www.open-std.org/JTC1/SC22/WG14/www/C99RationaleV5.10.pdf)
(6.7.3.1. Formal definition of `restrict`) operate with pairs of `restrict` pointers, such as in this example:

{% highlight C++ %}
void f1(int n, float * restrict a1,
        const float * restrict a2)
{
    int i;
    for ( i = 0; i < n; i++ )
        a1[i] += a2[i];
}
{% endhighlight %}

Perhaps any of the readers knows why both pointers must be defined `restrict`, while it seems that just one (any one)
must be sufficient? Am I getting something wrong?

The definition in the Standard answers another question -- about aliasing `char**` pointer with its targets.
The definition immediately says that it deals with an ordinary identifier and nothing else. We can't, for instance, have
an array of `restrict` pointers. It is sad because it makes the language non-orthogonal. The `restrict` keyword isn't
a storage class or some other mark put on an identifier. It is a type qualifier, with the same syntax role as `const`.
However, `const` can appear in several places in the type definition, with valid meaning everywhere:

- `const int ** p;` -- a pointer to an array of pointers to arrays of constant `int`s (meaning that `p` can change,
  `p[i]` can change but `p[i][j]` cannot)
- `int * const * p;` -- a pointer to an array of constant pointers to arrays of `int`s (meaning that `p` can change,
  `p[i]` can't change and `p[i][j]` again can change)
- `int ** const p;` -- a constant pointer to an array of pointers to arrays of `int`s (meaning that `p` can't change,
  while `p[i]` and `p[i][j]` can)
- `const int * const * const p;` -- a constant pointer to an array of constant pointers to arrays of constant `int`s (meaning
 that nothing can change).

The `restrict` qualifier, being formally in the same class (type qualifier) demonstrates no such flexibility. The only
meaningful place for it is right at the end:

- `int ** restrict p;` -- meaning that `p` can point to an array of pointers to arrays to `int`, and inside current block
  nothing else can point to the same array.

We can't define `int * restrict * p;` (a pointer to an array of restricted pointers). It probably wasn't easy
to define the behaviour of such pointers.

This means that, unfortunately, the `restrict` qualifier has not been added in a consistent way. As a result, it has
limited power. However, this qualifier is still useful in the case of ordinary identifiers.

By the way, [The Rationale](http://www.open-std.org/JTC1/SC22/WG14/www/C99RationaleV5.10.pdf) 
says that `struct` fields can be declared as `restrict`. This is strange, for `struct` fields are not ordinary identifiers,
and the definition shown above is not applicable. The way Rationale interprets such declaration is that if a
`struct` declaration contains two `restrict` pointers, these pointers taken from the same instance of the `struct`
do not alias each other. The pointers from different instances, however, can alias.

Here is a demonstration of the limited power of the `restrict` qualifier. No matter what we do, there is no way to use
`restrict` to optimise the following function:

{% highlight C++ %}
void f (char ** p)
{
    p[0][0] = 'a';
    p[0][1] = 'b';
}
{% endhighlight %}

We want to indicate that `p` isn't pointing to the same location as `p[0]` and therefore the first line does not
modify `p[0]`. The strict aliasing rule allows aliasing (`p[0]` is a `char*` pointer, which aliases everything),
and `restrict` on `p` does not help, because we need two of them and there is nowhere to put another one. All we're
left with is the manual optimisation:

{% highlight C++ %}
void f (char ** p)
{
    char * q = p[0];
    q[0] = 'a';
    q[1] = 'b';
}
{% endhighlight %}

<br>
Conclusions
-----------

- The Standards are very interesting reading; the language they are written in is magnificent.

- The **C** standard does define aliasing rules, which GCC call strict. The definition and the exact consequences
  are slightly different, but the general meaning is very close.

- The GCC allows switching these rules off, which can be considered a language extension; in this case everything
  is supposed to alias everything, and the optimisations
  are disabled; the semantics of the program is, however, well defined. This is the safest option.

- The **C** standard (starting from C99) provides `restrict` keyword, but **C++** standard does not. GCC provides
  `__restrict__` in both (as a language extension in the case of **C++**). The semantics is the same as in the C99
  standard. For an unknown reason, to indicate that two pointers do not alias, both must be declared `restrict`. In general,
  the power of this declaration is limited -- it does not help for complex pointer structures.

- The strict aliasing rules put restrictions on programs (often too strict), but allow the compiler to do some
  optimisations -- quite limited, however. Switching them off makes programs safer but disables optimisations.
  There seems to be no reliable way to tell the compiler where there is aliasing and where not, so the only real
  way out is to perform these optimisations by hand. The strict aliasing rules may be switched off in this case.

- Many highly optimised **C** programs (including all of those using SSE and AVX) utilise reading and writing
  blocks of memory using data types other than the declared type of the data (such as using `int` to manipulate
  a `char` array). Such programs may be violating the **C** standard. In our case (the article
  ["{{ site.TITLE-E1-C-SSE }}"]({{ site.ART-E1-C-SSE }})) this kind of optimisation caused the speed increase of
  six times. It is up to the programmer to decide if such a speedup is worth breaking the standard.

- I learned something new while writing this article; I hope the reader has learned something new while reading.

- However, the wording of the Standard is so unclear that I can't be sure I understood everything correctly there.
  I welcome any corrections.

- I thank mr. Kenton Varda for the comment that initiated this article. More comments are always welcome!
