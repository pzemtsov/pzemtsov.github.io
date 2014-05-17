---
layout: post
title:  GCC, optimisations and pointer aliasing
date:   2014-05-09 00:00:00
tags: C C++ aliasing optimisation
---

Today I'm going to talk about **pointer aliasing** and the way it is controlled in GCC. While most people know
the first part, not so many know the second (I didn't until recently), so hopefully there will be readers
who find this information useful. 

All the code snippets can be copied directly from the text and compiled, but for convenience I've also placed
them in one single file in a [GIT repository]({{ site.REPO-ALIASING }}), together with produced assembly code.

A little quiz
-------------

Here is a little quiz (true GCC experts will know the answer straight away).

Consider the following program (let's call it [`a.c`]({{ site.REPO-ALIASING }}/blob/master/a.c))):

{% highlight C++ %}
void a (int ** p)
{
    p[0][0] = 0;
    p[0][1] = 1;
}

void b (char ** p)
{
    p[0][0] = 0;
    p[0][1] = 1;
}
{% endhighlight %}

Let's compile it with GCC and look at the generated code (here I use 32-bin MINGW compiler; my previous tests were
in 64-bit mode but 32-bit code is a bit easier to read and I'm not going to run it anyway):

    > gcc -O3 -S a.c

We get the following (in file [`a.s`]({{ site.REPO-ALIASING }}/blob/master/a.s)), I only left relevant parts:

{% highlight text %}
_a:
        movl    4(%esp), %eax
        movl    (%eax), %eax
        movl    $0, (%eax)
        movl    $1, 4(%eax)
        ret

_b:
        movl    4(%esp), %eax
        movl    (%eax), %edx
        movb    $0, (%edx)
        movl    (%eax), %eax
        movb    $1, 1(%eax)
        ret
{% endhighlight %}

Why if the second piece of code longer than the first one? Why six instructions instead of five?

Looking at the code
-------------------

If you don't know how to read this assembler, don't worry: I didn't know either. I grew up with MASM, which looks
different from GNU assembler. It does not use `'%'` for registers and '`$`' for constants; besides, it specifies
the destinaton before the source and the operation size on the operand rather than in the instruction code. GNU
assembler does other way around. For instance, the instruction from the above listing:

{% highlight text %}
        movb    $1, 1(%eax)
{% endhighlight %}

in the familiar format of MASM looks like this, which is much more pleasant for the reader's eye:

{% highlight nasm %}
        mov byte ptr [eax+1], 1
{% endhighlight %}

So if you are also used to MASM syntax, it will take some effort reading GCC outputs, but it is still possible
to see what is going on. In `a()` we get parameter `p` into `eax`, then read `p[0]` also into `eax`, then write zero and one as
double words (`movl` instruction moves four bytes) at appropriate offsets from this `eax`. In `b()` we do exactly the
same, except we use byte store instructions (`movb`), and read `(%eax)` twice, as if we were afraid someone could have
changed `p[0]` between reads. This is indeed the case: we're scared of pointer aliasing.

What is pointer aliasing?
-------------------------

**Aliasing** is accessing of the same object through two different names. [**Pointer aliasing**](http://en.wikipedia.org/wiki/Pointer_aliasing)
is accessing the same memory through two different pointers. This situation can affect correctness of the compiled program,
so the compiler has to compile with pointer aliasing in mind.

Let's look at a simple example:

{% highlight C++ %}
void c (int * p, int * q)
{
    *p = *q + 1;
    *p = *q + 1;
}
{% endhighlight %}

If the compiler is dead sure that `p` and `q` are not pointing to the same memory, then the value of `*q` in the second
line is the same as in the first. This makes the second line redundant, and compiler may remove it. If, however,
`p` and `q` point to the same location, the first line will increment the value at that location by 1, and the second line
will do it again. As a result, the unoptimised program increments the location by 2, while the optimised program
only increments by 1, which is incorrect.

Let's see how GCC compiles this:

{% highlight text %}
_c:
        movl    8(%esp), %edx
        movl    4(%esp), %eax
        movl    (%edx), %ecx
        addl    $1, %ecx
        movl    %ecx, (%eax)
        movl    (%edx), %edx
        addl    $1, %edx
        movl    %edx, (%eax)
	ret
{% endhighlight %}

Both statements are present -- the compiler assumed the worst case and followed the safest route. As in many other cases,
safety contradicts performance: providing for the worst case (which may never occur), the compiler avoids performing
some important optimisations. In our example it didn't perform 
[common subexpression elimination](http://en.wikipedia.org/wiki/Common_subexpression_elimination). It is also
possible to construct example where it will not perform [loop-invariant code motion](http://en.wikipedia.org/wiki/Loop-invariant_code_motion):

{% highlight C++ %}
void d (int * p, int * q)
{
    int i = 0;
    for (i = 0; i < 1000; i++)
        *p = *q * 10 * i;
}
{% endhighlight %}

Here not only `*q`, but `*q * 10` are loop invariant expressions, and could be moved out of the loop, were it not for
the writing onto `*p`. The compiler is also scared:

{% highlight text %}
_d:
        pushl   %ebx
        movl    8(%esp), %ebx
        xorl    %eax, %eax
        movl    12(%esp), %ecx
L6:
        movl    (%ecx), %edx
        leal    (%edx,%edx,4), %edx
        addl    %edx, %edx
        imull   %eax, %edx
        addl    $1, %eax
        cmpl    $1000, %eax
        movl    %edx, (%ebx)
        jne     L6
        popl    %ebx
        ret
{% endhighlight %}

You can see that both reading from `*q`

{% highlight text %}
        movl    (%ecx), %edx
{% endhighlight %}

and multiplication by 10

{% highlight text %}
        leal    (%edx,%edx,4), %edx
        addl    %edx, %edx
{% endhighlight %}

are generated inside the loop and performed in each loop iteration. As a short off-topic: note the clever way the compiler multiplies
by 10. In case you are not familiar with this common abuse of `lea` instruction, this is what these instructions do:

    edx = edx + edx * 4
    edx = edx + edx

There are surprisingly many constants the compile can multiply by using some combination of `lea` instructions.    

Helping the compiler: `__restrict__`
----------------------------------

As we saw, the compiler on its own can't determine that there is no aliasing. It needs help from a human.
In GCC thare are two ways to provide such help: `__restrict__` keyword and `-fstrict-aliasing` switch.

The [**\_\_restrict\_\_**](http://en.wikipedia.org/wiki/Restrict) keyword is a type qualifier, which, applied to a pointer,
indicates that for the lifetime of this pointer its target can only be accessed via this pointer or the derived one.
By derived pointer we mean one produced by a direct assignment or pointer arithmetics (such as `p + 1`).

Or at least this is how it is defined. The reality is somewhat different.

Let's modify our `c()` procedure:

{% highlight C++ %}
void c_1 (int * p, int * __restrict__ q)
{
    *p = *q + 1;
    *p = *q + 1;
}
{% endhighlight %}

According to the above definition, this should promise the compiler that no one modifies `*q` for the lifetime of
`q`. However, it does not work. The promise did not convince the compiler, and the generated code was exactly the same.

Try again:

{% highlight C++ %}
void c_2 (int * __restrict__ p, int * q)
{
    *p = *q + 1;
    *p = *q + 1;
}
{% endhighlight %}

Same story: the same code. Let's go all the way:

{% highlight C++ %}
void c_3 (int * __restrict__ p, int * __restrict__ q)
{
    *p = *q + 1;
    *p = *q + 1;
}
{% endhighlight %}

Compile it:

{% highlight text %}
_c_3:
        movl    8(%esp), %eax
        movl    (%eax), %edx
        movl    4(%esp), %eax
        addl    $1, %edx
        movl    %edx, (%eax)
        ret
{% endhighlight %}

This is much better. The compiler dropped the second assignment completely.

The same happens with our loop example. Neither

{% highlight C++ %}
void d (int * p, int * __restrict__ q)
{% endhighlight %}

nor

{% highlight C++ %}
void d (int * __restrict__ p, int * q)
{% endhighlight %}

made any difference, while 

{% highlight C++ %}
void d_1 (int * __restrict__ p, int * __restrict__ q)
{
    int i = 0;
    for (i = 0; i < 1000; i++)
        *p = *q * 10 * i;
}
{% endhighlight %}

produced fantastic result:

{% highlight text %}
_d_1:
        .cfi_startproc
        movl    8(%esp), %eax
        movl    (%eax), %eax
        leal    (%eax,%eax,4), %edx
        movl    4(%esp), %eax
        addl    %edx, %edx
        imull   $999, %edx, %edx
        movl    %edx, (%eax)
        ret
{% endhighlight %}

Where is the loop? The loop is gone. The compiler figured out that all the loop iterations have no effect except
for the last one, and removed the entire loop. The code it generated is equivalent to

{% highlight C++ %}
    *p = *q * 10 * 999;
{% endhighlight %}

It is strange it didn't generate `*q * 9990`. 32-bit multiplication is associative even when an overflows occur. It
could be an artefact of moving invarint code out of loop first, and optimising the loop out next.

We didn't see the loop invariant code motion in its pure form. To do it the loop must be less trivial. This form
works:

{% highlight C++ %}
void d_2 (int * p, int * q)
{
    int i = 0;
    for (i = 0; i < 1000; i++)
        *p += *q * 10 * i;
}
{% endhighlight %}

I'm leaving it to the reader to check what code is generated with and without `__restrict__` (the code is in
the repository).

We have established that the official definition of the effect of `__restrict__` is not quite correct. According
to the definition, applying this qualifier to any of the parameters must resolve the aliasing issue. In reality
we need to apply it to both.

It helps to be strict. Sometimes...
-----------------------------------

Let's modify our `c()` procedure again, removing `__restrict__` and replacing `q` with `short`:

{% highlight C++ %}
void c_short (int * p, short * q)
{
    *p = *q + 1;
    *p = *q + 1;
}
{% endhighlight %}

We don't expect much to change, but we are wrong:

{% highlight text %}
_c_short:
        movl    8(%esp), %eax
        movswl  (%eax), %edx
        movl    4(%esp), %eax
        addl    $1, %edx
        movl    %edx, (%eax)
        ret
{% endhighlight %}

The code looks as if there were `__restrict__` keywords used. What happened, why replacing `int` with `short`
made such a difference?

Let's compile the program with a special switch: `-fno-strict-aliasing`. The full assembly can be found in
the repository, file [a-nostrict.s]({{ site.REPO-ALIASING }}/blob/master/a-nostrict.s).

{% highlight text %}
_c_short:
        movl    8(%esp), %edx
        movl    4(%esp), %eax
        movswl  (%edx), %ecx
        addl    $1, %ecx
        movl    %ecx, (%eax)
        movswl  (%edx), %edx
        addl    $1, %edx
        movl    %edx, (%eax)
        ret
{% endhighlight %}

The effect is gone: the code looks very close to the old `int` version.

All the `-fno-strict-aliasing` key does is that it switches off previous `-fstrict-aliasing`. We didn't invoke
`-fstrict-aliasing`, but it was switched on automatically as part of `-O3` optimisatin pack (in fact, it is
switched on by `-O2`, and `-O3` is a superset of `-O2`).

This is what GCC documentation says about the meaning of `-fstrict-aliasing`:

<table>
<tr><td><pre>-fstrict-aliasing</pre></td>
<td>
Allows the compiler to assume the strictest aliasing rules applicable to the language being compiled.
For C (and C++), this activates optimizations based on the type of expressions. In particular, an object
of one type is assumed never to reside at the same address as an object of a different type, unless the
types are almost the same. For example, an unsigned int can alias an int, but not a void* or a double.
<strong>A character type may alias any other type</strong>.
</td></tr></table>

This is why `c_short()` was optimised, while `int` version (`c()`) was not: the compiler assumed that two
`int` pointers can point to the same memory while an `int` pointer and a `short` pointer can not. The text in bold
suggests that if we replace `short` with `char`, it again won't optimise. And this is indeed the case:

{% highlight C++ %}
void c_char (int * p, char * q)
{
    *p = *q + 1;
    *p = *q + 1;
}
{% endhighlight %}

compiles as this:

{% highlight text %}
_c_char:
        movl    8(%esp), %edx
        movl    4(%esp), %eax
        movsbl  (%edx), %ecx
        addl    $1, %ecx
        movl    %ecx, (%eax)
        movsbl  (%edx), %edx
        addl    $1, %edx
        movl    %edx, (%eax)
        ret
{% endhighlight %}

The answer to the little quiz
-----------------------------

Now we know why our original two functions compiled so differently.

{% highlight C++ %}
void a (int ** p)
{
    p[0][0] = 0;
    p[0][1] = 1;
}

void b (char ** p)
{
    p[0][0] = 0;
    p[0][1] = 1;
}
{% endhighlight %}

In both cases there is a possibility of `p` aliasing with `*p`. In both cases in can only be achieved by some
nasty type cast:

{% highlight C++ %}
    int * m[1];
    m[0] = (int*) m;
    a (m);
{% endhighlight %}

When `-fstrict-aliasing` is on, the compiler assumes that in `a()` aliasing is impossible (a pointer to `int` can not
alias with a pointer to `int*`), while in `b()` aliasing is possible (a pointer to `char` is allowed to alias a
pointer to `char*`). This explains the difference in code. If you compile with `-fno-strict-aliasing`, the compiler
will generate bad code for both cases.

Why?
----

Why is there special provision for character type? Why GCC developers did not specify more logical rule: "Any two
pointers of different types are assumed unaliased", with a possible exception of a `void*`?

Obviously, I don't know the exact motivations of GCC developers. But I will try to make a guess.

First of all, it must be made clear that any unmotivated assumption the compiler makes about the program is
potentially dangerous and can cause incorrect behaviour of the compiled program. By "motivated" assumption I mean
a result of a formal proof.  For instance, if the function `c()` is declared `static` and is called from a single
place inside the program, and the call looks like this:

{% highlight C++ %}
    int x = 0;
    int y = 0;
    c (&x, &y);
{% endhighlight %}

the compiler may assume that parameters of `c()` are never aliased and compile it accordingly.
Strict aliasing, however, is different. It causes making unsafe assumption based on a single global setting,
the optimisation level.

This is very important and justifies a bold statement:

**Any C/C++ program compiled with option `-O2` or `-O3` is potentially unsafe and incorrect!**

It is common practice to compile everything with `-O3`. Everybody wants speed, and everybody knows `-O3` means speed.
This way unfamilliar, old and well forgotten, or third-party code  may end up being compiled like this, and who
knows what is inside? An author may have specifically relied on aliasing. One very common way of deliberate aliasing
is when we cast pointers to different types in order to access the same memory differently. Consider following
artificial example: we want to check how the value changes if we fill its first bytes with zeroes (for instance,
we want to establish the byte order of an `int`):

{% highlight C++ %}
int e (int * p)
{
    int a = *p;
    *(short*) p = 0;
    return *p - a;
}
{% endhighlight %}

This is how it compiles with `-O3`:

{% highlight text %}
_e:
        movl    4(%esp), %eax
        xorl    %edx, %edx
        movw    %dx, (%eax)
        xorl    %eax, %eax
        ret
{% endhighlight %}

This function always returns zero, which is exactly what you expect if you assume no aliasing: `*(short*)p` is not
aliased with `*p` as having different type. As a result, `*p` is only read once, and obviously the difference is zero.

This won't happen if we write one byte instead of two:

{% highlight C++ %}
int e_char (int * p)
{
    int a = *p;
    *(char*) p = 0;
    return *p - a;
}
{% endhighlight %}

It produces the following:

{% highlight text %}
_e_char:
        movl    4(%esp), %eax
        movl    (%eax), %edx
        movb    $0, (%eax)
        movl    (%eax), %eax
        subl    %edx, %eax
        ret
{% endhighlight %}

As you can see, the code is correct and performs exactly as intended.

Most probably, the developers of GCC considered automatic non-aliasing of char pointers too high a risk. Programmers
convert `int` pointers to `short` pointers  and vice versa relatively seldom. The same applies to `int` pointers 
being converted to pointers to pointers. But converting pointers to various types to `char*` is very common. All sorts of
memory dump, memory comparison, memory move routines are doing it. Converting in other direction is very common, too -- 
for example, when encoding and decoding protocol messages. Imagine a message, consisting of a tag (byte value),
a port (two bytes), an IP address (four bytes), and possible other fields, and let this message be transmitted in
a byte-packed form. A routine that packs that message into a byte buffer is very likely to look similar to this:

{% highlight C++ %}
void encode (Message * message, char * buffer)
{
    char * p = buffer;
    *(char*) p = message -> tag; p += 1;
    *(unsigned short*) p = message -> port; p += 2;
    *(unsigned *) p = message -> addr; p += 4;
    /* etc. */
}
{% endhighlight %}

If filling the buffer this way is interleaved with accessing its bytes, no-aliasing assumption is quite likely
to cause incorrect behaviour. And there is too much of this type of code around to take a risk.

This is, probably, why "different types are not aliased" rule is not applied to `char` pointers:
it is considered unsafe. However, we mustn't forget that strict aliasing is unsafe in general.

What to do?
-----------

Let's return to our quiz example, specifically to our `b()` function:

{% highlight C++ %}
void b (char ** p)
{
    p[0][0] = 0;
    p[0][1] = 1;
}
{% endhighlight %}

Let's assume we know there is no aliasing here, because we work with a regular pointer structure, where `p` is a pointer to
an array of pointers to arrays of `char` (a usual way to implement two-dimentions arrays), and we know no one
is using unsafe casts. We want to tell this fact to the compiler. Maybe `__restrict__` will help?

No, applying this qualifier to `p` does not help, this code

{% highlight C++ %}
void b_1 (char ** __restrict__ p)
{
    p[0][0] = 0;
    p[0][1] = 1;
}
{% endhighlight %}

produces exactly the same result. We remember that declaring just one pointer as `__restrict__` did not help. But here we
have just one pointer, `p`, which potentially aliases with `*p`. How about this?

{% highlight C++ %}
void b_2 (char * __restrict__ * __restrict__ p)
{
    p[0][0] = 0;
    p[0][1] = 1;
}
{% endhighlight %}

This does not help either. I couldn't find any way to tell a compiler that there is no aliasing in this case,
Maybe some of the readers know the way?

How about the `const` specifier in **C++**? Doesn't declaring a `const` pointer indicate that the memory pointed
by it won't change? No, it doesn't: it only means that the memory won't be changed via this pointer, and says
nothing about other pointers. It is just an extra safety feature, additional syntax check (you can't pass `const`
pointer where non-`const` is expected), and not a performance improvement mechanism.

For example, if we modify `c()` like this

{% highlight C++ %}
void c_const (int * p, const int * q)
{
    *p = *q + 1;
    *p = *q + 1;
}
{% endhighlight %}

it won't affect the code.

We are coming to the sad conclusion that there are cases where the compiler assumes aliasing and there is no way to
convince it otherwise. We have to change the code. In our case it must look like this:

{% highlight C++ %}
void a_modified (int ** p)
{
    int * p0 = p[0];
    p0[0] = 0;
    p0[1] = 1;
}

void b_modified (char ** p)
{
    char * p0 = p[0];
    p0[0] = 0;
    p0[1] = 1;
}
{% endhighlight %}

The code generated is good in both cases:

{% highlight text %}
_a_modified:
        movl	4(%esp), %eax
        movl	(%eax), %eax
        movl	$0, (%eax)
        movl	$1, 4(%eax)
        ret

_b_modified:
        movl	4(%esp), %eax
        movl	(%eax), %eax
        movb	$0, (%eax)
        movb	$1, 1(%eax)
        ret
{% endhighlight %}

The code does not depend on `-fstrict-aliasing`, so we can switch it off:

    > gcc -O3 -fno-strict-aliasing -S a.c

This will improve overall safety of our compiled program.

Other languages
---------------

**Java** claims it does not have pointers, but this is not completely true. It does not have pointer arithmetics
or unsafe pointer casting (unless you use `java.misc.Unsafe`), but its object references are in fact pointers
and as such are subject to pointer aliasing problem. Our `c()` example could be re-written in **Java** using either classes or arrays:

{% highlight java %}
class T {
    int x;
}

static void c_object (T p, T q)
{
    p.x = q.x + 1;
    p.x = q.x + 1;
}

static void c_array (int [] p, int [] q)
{
    p[0] = q[0] + 1;
    p[0] = q[0] + 1;
}
{% endhighlight %}

Here we have the same pointer aliasing problem as in **C**.

Hovever, absence of pointer arithmetics and unsafe casts helps reduce the scope of the problem. For instance,
non-aliasing of pointers to different types (achieved in GCC by `-fstrict-aliasing`) in **Java** is built into the
language. Two references of incompatible types or two arrays with different types of elements never alias,
and this allows **Java** compiler (HotSpot) to perform optimisations that **C** compiler does not do.
For example, Java version of `b()`

{% highlight java %}
void b (byte [][] p)
{
    p[0][0] = (byte)0;
    p[0][1] = (byte)1;
}
{% endhighlight %}

never suffers from aliasing problem, and access to `p[0]` can be done once.

It is worth mentioning that aliasing may happen without pointers at all. It is sufficient for a language to
have arrays. After all, the memory can be seen as a huge array, and a pointer as its index.
The opposite is also true, and two indices in the same array behave exactly like pointers and suffer from the
same aliasing problems. A **C** example may look like this:

{% highlight C++ %}
int array [1000];

void c_array (int p, int q)
{
    array [p] = array [q] + 1;
    array [p] = array [q] + 1;
}
{% endhighlight %}

The code looks like this:

{% highlight text %}
_c_array:
        movl    8(%esp), %edx
        movl    4(%esp), %eax
        movl    _array(,%edx,4), %ecx
        addl    $1, %ecx
        movl    %ecx, _array(,%eax,4)
        movl    _array(,%edx,4), %edx
        addl    $1, %edx
        movl    %edx, _array(,%eax,4)
        ret
{% endhighlight %}

As you can see, the code is generated with aliasing in mind, and in this case there is no way to configure the
compiler to do otherwise.

Conclusions
-----------

- Pointer aliasing is a serious problem that significally reduces compiler's optimising ability.

- Languages with built-in type safety, such as **Java**, suffer less from it than lower level, less safe languages,
  such as **C**.

- As a result, there may be cases where program in **Java** runs faster than the same program rewritten in **C**.

- You can use `__restrict__` keyword in **C** to control pointer aliasing but you can't rely on it because it
  does not always work.

- Strict aliasing as provided by GCC improves performance but is inherently unsafe.

- And it does not help where `char` pointers are involved.

- Since the strict aliasing is activated automatically with `-O3` option, it makes sense to switch it off explicitly
  in the command line (`-fno-strict-aliasing`), and only switch it on for the files we trust.

- Sometimes you have to help compiler by performing some of the optimisations manually.

Coming soon
-----------

How is all of this applicable to our [de-multiplexing of E1 streams in C]({{ site.ART-E1-C }})
example? Stay tuned, this will be covered soon.
.