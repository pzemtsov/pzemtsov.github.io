---
layout: post
title:  "Macros, call by name, lambda and Jensen's device"
date:   2014-10-28 12:00:00
tags: C C++ GCC optimisation
story: callbacks
story-title: "Performance of various callback mechanisms"
---


Call by name
------------

In [the previous article]({{ site.ART-BUG2 }}) we were tracing a bug that was introduced when a function was replaced with a macro.
The parameters passed into a macro conflicted with the internal variables declared inside.

The root of the problem is in textual parameter substitution where we in fact wanted proper object passing.
This resembles the way parameters were passed ages ago in [**Algol 60**](http://en.wikipedia.org/wiki/ALGOL_60),
which was **'call by name'**. The formal parameters were substituted with the textual representations of the actual
parameters, which were re-evaluated each times these parameters were used. This mechanism is well explained
[here](http://www.cs.sfu.ca/~cameron/Teaching/383/PassByName.html). It is interesting to compare this mechanism
with those available in **C++** and check the performance.

The **Algol 60**'s parameter passing and the way macros work in **C** are not identical.
**C** macros accept any text as parameters; it doesn't even have to be syntactically correct, as long as all
necessary commas and parentheses are in place. This text, as a sequence of characters, is placed inside the macro body.
**Algol 60** worked on a higher level: the parameters must be correct expressions with identifiers resolved in the calling context.
The textual substitution of actual parameters into the function acted as if the internal variables were renamed first
(this was the way we fixed the conflict in **C** macros, too: we renamed the internal variables).
As a result, there was no chance of clash between the parameters and the internal variables,
but the passed values could conflict with each other and with external
variables. **C** macros suffer from both problems.

It's impossible to write a procedure that swaps two variables in **Algol 60**, neither can it be done using macro
in **C**. Just the danger is higher in **C**. The **Algol 60** procedure

{% highlight Pascal %}
procedure swap (a, b);
integer a, b;
begin
  integer temp;
  temp := a;
  a := b;
  b:= temp
end;
{% endhighlight %}

fails when called with parameters `i` and `x[i]`.

The **C** macro

{% highlight C++ %}
#define swap(a, b) do {\
    int temp = a;\
    a = b;\
    b = temp;\
} while (0)
{% endhighlight %}

fails in this case, too, but it also fails when called with parameters `x` and `temp`.

It is interesting that in **Algol 60** call by name was the default way of passing parameters;
one had to use additional keyword `value` to pass parameters the way we now consider the most obvious (by value).

Fortunately, in most languages (not in **Java**, though) there is still the good old call by reference
(which existed already in **FORTRAN** -- but there it was the only way), so one can write a proper procedure to
swap the values. This is the **C++** example:

{% highlight C++ %}
void swap(int &a, int &b)
{
    int temp = a;
    a = b;
    b = temp;
}
{% endhighlight %}

Using pointers or references can imitate call by reference in macros:

{% highlight C++ %}
#define swap(a, b) do {\
    int * pb = & b;
    int temp = a;\
    a = b;\
    *pb = temp;\
} while (0)
{% endhighlight %}

This code works correctly when called with `i` and `x[i]`, but adds additional internal variable that can clash
with a parameter.

Jensen's device in **Algol 60**
-------------------------------

There are cases when **Algol**'s way of parameter passing is beneficial -- when we really want to re-evaluate parameters
each time they are used inside, and not evaluate at all if never used. The article referenced above illustrated this
by the so called **Jensen's device** -- a function that sums arbitrary expressions. Here is its text in **Algol 60**:

{% highlight Pascal %}
real procedure Sum(j, lo, hi, Ej);
  value lo, hi;
  integer j, lo, hi; real Ej;
begin
  real S;
  S := 0;
  for j := lo step 1 until hi do
    S := S + Ej;
  Sum := S
end;
{% endhighlight %}

and a typical call:

{% highlight Pascal %}
  s := Sum (i, 1, 10, i * x[i]);
{% endhighlight %}


This kind of parameter passing was implemented by wrapping the parameters in special functions (__thunks__) and
passing those functions. Effectively the expression passed by name acted as a [lambda definition](http://en.wikipedia.org/wiki/Anonymous_function)
-- a construct common for functional languages and now finding its way into the imperative languages as well. It seems that these days
there is big demand for functionality of the type demonstrated in Jensen's device. Let's implement this device in **C++**
and check the performance.

Jensen's device in **C++**: using macro
---------------------------------------

I'll implement an integer version of Jensen't device (I prefer not to deal with floating point unless I have to).

Since we observed that there is similarity between call by name and **C** macros, they become our first choice
tool to do the job:

{% highlight C++ %}
#define sum_macro(lo, hi, i, f, res) do {\
        int x = 0;\
        for (i = lo; i < hi; i++)\
            x += f;\
        res = x;\
    } while (0)
{% endhighlight %}

A few comments. First, the semantics of `lo` and `hi` parameters has changed: inclusive lower boundary and exclusive
upper one are customary in **C**, while **Algol 60** preferred both boundaries inclusive. And second, a **C** macro
can't return a value, unless it is a simple one-liner. Return via output parameter was needed in this case.

This is how we are going to call it:

{% highlight C++ %}
static const int SRC_SIZE = 10000;
static int src[SRC_SIZE];

class Macro : public Test
{
public:
    int test() const
    {
        int i, res;
        sum_macro(0, SRC_SIZE, i, i*src[i], res);
        return res;
    }
};
{% endhighlight %}

(a test was wrapped in a class to simplify measurement framework).

Using a functional pointer
--------------------------

Next step is to imitate the **Algol-60** thunks by using functional pointers:

{% highlight C++ %}
int sum_func(int lo, int hi, int f(int))
{
    int x = 0;
    for (int i = lo; i < hi; i++)
        x += f(i);
    return x;
}
{% endhighlight %}

In **C** we would have to create a separate function

{% highlight C++ %}
int f(int i)
{
    return i * x[i];
}
{% endhighlight %}

and then pass it as a parameter:

{% highlight C++ %}
    return sum_func(0, SRC_SIZE, f);
{% endhighlight %}

Since **C++** standard version 11, the language contains definitions of inline functions (lambdas),
so the call may now look much neater:

{% highlight C++ %}
class Lambda : public Test
{
public:
    int test() const
    {
        return sum_func(0, SRC_SIZE, [](int i) {return i*src[i]; });
    }
};
{% endhighlight %}

Lambda with capture: templates
------------------------------

The last version suffers from a serious deficiency: it requires the `src` array to be visible inside a function,
and in **C** this can only be achieved by declaring it on the top level (as a global or static object). The macro
version was free from this limitation -- it worked on whatever object called `src` was visible where the macro was expanded.

**C++**'s lambdas allow useful extension: a capture. When declaring a capture, one can specify the variables that must
be visible inside. The syntax is quite flexible: one can capture specific variables or all of them, capture can
be done by value or by reference. If we want to give a lambda function access to current class fields, we can
define lambda as capturing `this`:

{% highlight C++ %}
class Lambda_Capture : public Test
{
    const int* x;
public:
    Lambda_Capture(const int* x) : x(x)
    {}

public:
    int test() const
    {
        return sum_func(0, SRC_SIZE, [this](int i) {return i*x[i]; });
    }
};
{% endhighlight %}

This, however, does not compile:

    # c++ -std=c++0x -o lambda lambda.cpp
    lambda.cpp: In member function "virtual int Lambda_Capture::test() const":
    lambda.cpp:88:69: error: cannot convert "Lambda_Capture::test() const::<lambda
    (int)>" to "int (*)(int)" for argument "3" to "int sum_func(int, int, int (*)
    (int))"

The reason it does not compile is simple. There is no miracles in **C++**. A function is a function. Lambda or
no lambda, it has a parameter list and only sees the objects that are either global or passed as parameters. If a function is an
instance method, it can see the object fields as well -- but only because an extra parameter `this` is passed to it.
Our interface does not provide any additional parameters, that's why we can't pass a lambda function with a capture into our
`sum_func`.

How do then lambdas with capture work at all? The reason they work is because _they are not functions_. They are classes
that define a function call (`operator()`). Such classes are called **functors**, and they can't be passed where functions are expected.

There is, however, an easy way to write a function that can accept both functions and functors: the function must be a template:

{% highlight C++ %}
template<typename T> int sum_template(int lo, int hi, T f)
{
    int x = 0;
    for (int i = lo; i < hi; i++)
        x += f(i);
    return x;
}
{% endhighlight %}

The function body is identical to `sum_func`, and it can accept parameters of any type `T`, for which
expression `f(i)` is defined. We can call it with lambdas with or without captures, it will work for both.
We'll add two more classes (`Lambda_Template` and `Lambda_Template_Capture`) to our test set.

Lambda with capture: `std::function`
------------------------------------

Although templates work well for various types of lambdas, the recommended way in **C++** is different. It involves
using a type from the standard library, called `std::function`, which is templated by the function pointer type,
and which can be constructed from both functions and functors. This class is a functor itself, which allows not to
change anything in the function body:

{% highlight C++ %}
int sum_std_function(int lo, int hi, std::function<int (int)> f)
{
    int x = 0;
    for (int i = lo; i < hi; i++)
        x += f(i);
    return x;
}
{% endhighlight %}

This declaration has additional advantage: templates must be declared in header files while functions can be placed
wherever appropriate, provided that the prototype is visible. This is definitely a nicer way to write such functions --
provided, of course, that performance is not compromised. This is what we are going to measure.

Object-oriented approach
------------------------

Just for completeness, I'd like to add two more versions using traditional **C++** ways to implement callbacks.
The first one involves interfaces (abstract classes without any fields):

{% highlight C++ %}
class Func
{
public:
    virtual int f(int) const = 0;
};

int sum_interface(int lo, int hi, const Func * f)
{
    int x = 0;
    for (int i = lo; i < hi; i++)
        x += f->f(i);
    return x;
}

class Interface : public Func, public Test
{
    const int* x;
public:
    Interface(const int * x) : x(x)
    {}

    int f (int i) const
    {
        return i*x[i];
    }

    int test() const
    {
        return sum_interface(0, SRC_SIZE, this);
    }
};
{% endhighlight %}

The second one involves abstract classes (a function is defined inside a class and calls a virtual method from
derived class):

{% highlight C++ %}
class Adder
{
protected:
    virtual int f(int) const = 0;

public:
    inline int sum(int lo, int hi) const
    {
        int x = 0;
        for (int i = lo; i < hi; i++)
            x += f(i);
        return x;
    }
};

class Abstract_Class : public Adder, public Test
{
    const int* x;
public:
    Abstract_Class(const int * x) : x(x)
    {}

    int f(int i) const
    {
        return i*x[i];
    }

    int test() const
    {
        return sum(0, SRC_SIZE);
    }
};
{% endhighlight %}

Note that both approaches provide capturing automatically, since the method has access to `this` pointer by design.

The performance
----------------------

To measure performance, we'll run each of the tests one million times:

{% highlight C++ %}
void measure(const Test & test)
{
    uint64_t t0 = currentTimeMillis();
    long sum = 0;
    for (int i = 0; i < ITERATIONS; i++) {
        sum += (long) test.test ();
    }
    uint64_t t = currentTimeMillis() - t0;
    std::cout << typeid (test).name() << ": " << sum << ": " << t << std::endl;
}

class Test {
public:
    virtual int test() const = 0;
};
{% endhighlight %}

We add all the results together and print the sum in order to prevent compiler from optimising the code away completely.

The full text can be found [in the repository]({{ site.REPO-JENSEN-DEVICE }}), file
[`lambda.cpp`]({{ site.REPO-JENSEN-DEVICE }}/blob/master/lambda.cpp).

This is the result (I've run the `Test_Macro` version twice to get rid of whatever "warm-up" effects there might be,
such as delay in setting CPU clock to full speed):

    # c++ -std=c++0x -O3 -msse4.2 -falign-functions=32 -falign-loops=32 -funroll-loops -o lambda lambda.cpp -lrt
    # ./lambda
    10Test_Macro: -1724114088000000: 1878
    10Test_Macro: -1724114088000000: 1870
    11Test_Lambda: -1724114088000000: 1869
    20Test_Lambda_Template: -1724114088000000: 1870
    28Test_Lambda_Template_Capture: -1724114088000000: 18032
    15Test_Lambda_Std: -1724114088000000: 15701
    23Test_Lambda_Std_Capture: -1724114088000000: 18121
    14Test_Interface: -1724114088000000: 14142
    19Test_Abstract_Class: -1724114088000000: 14137

We see three fast versions (`Test_Macro`, `Test_Lambda` and `Test_Lambda_Template`) and five slow versions. Equal speed of `Lambda`
and `Lambda_Template` is no surprise - after template is resolved, we get exactly the same code as when using a functional pointer.
But the fact that ordinary lambda is as fast as a macro is a surprise. Also surprising is a big difference in speed
between slow and fast versions: the slowdown factor varies between 7.6 and 9.7 times.

Looking at the code
-------------------

Why is there such a difference in speed? Let's first look at the `Test_Macro`'s code:

{% highlight c-objdump %}
_ZNK10Test_Macro4testEv:
.LFB1483:
        .cfi_startproc
        pxor    %xmm0, %xmm0
        movl    $_ZL3src, %eax
        movdqa  .LC1(%rip), %xmm1
        movdqa  .LC0(%rip), %xmm2
        .p2align 5
.L2:
        movdqa  %xmm2, %xmm10
        pmulld  (%rax), %xmm2
        paddd   %xmm2, %xmm0
        movdqa  144(%rax), %xmm3
        paddd   %xmm1, %xmm10
        movdqa  %xmm10, %xmm9
        pmulld  16(%rax), %xmm10
        paddd   %xmm10, %xmm0
        paddd   %xmm1, %xmm9
        movdqa  %xmm9, %xmm8
        pmulld  32(%rax), %xmm9
        paddd   %xmm9, %xmm0
        paddd   %xmm1, %xmm8
        movdqa  %xmm8, %xmm7
        pmulld  48(%rax), %xmm8
        paddd   %xmm8, %xmm0
        paddd   %xmm1, %xmm7
        movdqa  %xmm7, %xmm6
        pmulld  64(%rax), %xmm7
        paddd   %xmm7, %xmm0
        paddd   %xmm1, %xmm6
        movdqa  %xmm6, %xmm5
        pmulld  80(%rax), %xmm6
        paddd   %xmm6, %xmm0
        paddd   %xmm1, %xmm5
        movdqa  %xmm5, %xmm2
        pmulld  96(%rax), %xmm5
        paddd   %xmm5, %xmm0
        paddd   %xmm1, %xmm2
        movdqa  %xmm2, %xmm4
        pmulld  112(%rax), %xmm2
        paddd   %xmm2, %xmm0
        paddd   %xmm1, %xmm4
        movdqa  %xmm4, %xmm2
        pmulld  128(%rax), %xmm4
        addq    $160, %rax
        paddd   %xmm4, %xmm0
        cmpq    $_ZL3src+40000, %rax
        paddd   %xmm1, %xmm2
        pmulld  %xmm2, %xmm3
        paddd   %xmm1, %xmm2
        paddd   %xmm3, %xmm0
        jne     .L2
        movdqa  %xmm0, %xmm11
        psrldq  $8, %xmm11
        paddd   %xmm11, %xmm0
        movdqa  %xmm0, %xmm1
        psrldq  $4, %xmm1
        paddd   %xmm1, %xmm0
        pextrd  $0, %xmm0, %eax
        ret
 {% endhighlight %}

We can see that the loop is unrolled and vectorised. Each loop iteration advances the source pointer (`%rax`) by 160,
or 40 numbers. These 40 numbers are processed in 10 steps, four numbers at a time, using SSE. We load values `(0, 1, 2, 3)`
(the value of `.LC0`) into an SSE register (`%xmm2` is used initially, then the register changes), and multiply this
register component-wise by the SSE value containing four values read from the source pointer. Then we add 4 to all
the components and carry on. In the end we get four partial sums in four component of one register (`%xmm0`), all
that's left is add these four components together after the loop. One additional interesting detail is reading `144(%rax)`
early in the loop. This is a prefetch. The compiler initiates reading of a value from the memory way ahead of
current pointer, so that it is ready by the time it is required. Moreover, since the caches operate on entire
cache lines, not only this value will be fetched but the nearby values as well. Very clever trick.

In general, the code looks very good. Perhaps, it can be improved but this requires manual programming in assembly language,
which I'd like to avoid at this point. It would also be interesting to measure the effect of the prefetch.

Looking at the code of the `Test_Lambda` and `Test_Lambda_Template` versions reveals a surprising fact: the code is identical to
the code shown above. The compiler inlined the function (`sum_func` or appropriate version of `sum_template`), and then
inlined the function passed to it as a parameter. It would probably not have happened if any of these functions were
much larger. 

What about the other versions? In theory, nothing prevents the code from `std::function` to be inlined, too.
However, this does not happen. This is what the inner loop of the `Test_Lambda_Std` test look like:

{% highlight c-objdump %}

        movq    $_ZNSt17_Function_handlerIFiiEZNK15Test_Lambda_Std4testEvEUliE_E9_M_invokeERKSt9_Any_datai, 24(%rsp)
        movq    $_ZNSt14_Function_base13_Base_managerIZNK15Test_Lambda_Std4testEvEUliE_E10_M_managerERSt9_Any_dataRKS4_St18_Manager_operation, 16(%rsp)
.LEHB4:
        call    _Znwm
.LEHE4:
        movq    %rax, (%rsp)
        xorl    %ebp, %ebp
        xorl    %ebx, %ebx
.L125:
        cmpq    $0, 16(%rsp)
        je      .L149
        movl    %ebx, %esi
        movq    %rsp, %rdi
.LEHB5:
        call    *24(%rsp)
        addl    %eax, %ebp
        addl    $1, %ebx
        cmpq    $0, 16(%rsp)
        je      .L149
        movl    %ebx, %esi
        movq    %rsp, %rdi
        call    *24(%rsp)
        addl    %eax, %ebp
        cmpq    $0, 16(%rsp)
        leal    1(%rbx), %esi
        je      .L149
        movq    %rsp, %rdi
        call    *24(%rsp)
        addl    %eax, %ebp
        cmpq    $0, 16(%rsp)
        leal    2(%rbx), %esi
        je      .L149
        movq    %rsp, %rdi
        call    *24(%rsp)
        addl    %eax, %ebp
        cmpq    $0, 16(%rsp)
        leal    3(%rbx), %esi
        je      .L149
        movq    %rsp, %rdi
        call    *24(%rsp)
        addl    %eax, %ebp
        cmpq    $0, 16(%rsp)
        leal    4(%rbx), %esi
        je      .L149
        movq    %rsp, %rdi
        call    *24(%rsp)
        addl    %eax, %ebp
        cmpq    $0, 16(%rsp)
        leal    5(%rbx), %esi
        je      .L149
        movq    %rsp, %rdi
        call    *24(%rsp)
        addl    %eax, %ebp
        cmpq    $0, 16(%rsp)
        leal    6(%rbx), %esi
        je      .L149
        movq    %rsp, %rdi
        call    *24(%rsp)
.LEHE5:
        addl    $7, %ebx
        addl    %eax, %ebp
        cmpl    $10000, %ebx
        jne     .L125
{% endhighlight %} 

The loop is also unrolled, 8 times, but each time it performs unnecessary test of `16(%rsp)` for zero and an indirect call
of `24(%rsp)`. And here is the procedure that it calls:

{% highlight c-objdump %}
_ZNSt17_Function_handlerIFiiEZNK15Test_Lambda_Std4testEvEUliE_E9_M_invokeERKSt9_Any_datai:
.LFB1673:
        .cfi_startproc
        movslq  %esi, %rax
        imull   _ZL3src(,%rax,4), %esi
        movl    %esi, %eax
        ret
{% endhighlight %} 

This, obviously, is our lambda function.

It looks like the glue code inside the implementation of `std::function` was too complex for the compiler to inline
and optimise properly. It is worth mentioning that `_Znwm` called in the beginning is in fact `new`, so the implementation
allocates dynamic memory. In short, the `std::function` presents considerable overhead, at least in GNU C 4.6.

It is less clear why traditional object-oriented implementations are also slow. There are no intermediate objects,
no constructors or destructors to call -- in short, there is very little difference between code with virtual calls
and code with function pointers from optimisation point of view. However, the methods are not de-virtualised
and inlined. This is the code for `Test_Abstract_Class::test`:

{% highlight c-objdump %}
_ZNK19Test_Abstract_Class4testEv:
        pushq   %r12
        xorl    %r12d, %r12d
        pushq   %rbp
        xorl    %ebp, %ebp
        pushq   %rbx
        movq    %rdi, %rbx
        .p2align 5
.L39:
        movq    (%rbx), %rax
        movl    %ebp, %esi
        movq    %rbx, %rdi
        call    *(%rax)
        addl    %eax, %r12d
        movq    (%rbx), %rax
        leal    1(%rbp), %esi
        movq    %rbx, %rdi
        call    *(%rax)
        addl    %eax, %r12d
        movq    (%rbx), %rax
        leal    2(%rbp), %esi
        movq    %rbx, %rdi
        call    *(%rax)
        addl    %eax, %r12d
        movq    (%rbx), %rax
        leal    3(%rbp), %esi
        movq    %rbx, %rdi
        call    *(%rax)
        addl    %eax, %r12d
        movq    (%rbx), %rax
        leal    4(%rbp), %esi
        movq    %rbx, %rdi
        call    *(%rax)
        addl    %eax, %r12d
        movq    (%rbx), %rax
        leal    5(%rbp), %esi
        movq    %rbx, %rdi
        call    *(%rax)
        addl    %eax, %r12d
        movq    (%rbx), %rax
        leal    6(%rbp), %esi
        movq    %rbx, %rdi
        call    *(%rax)
        addl    %eax, %r12d
        movq    (%rbx), %rax
        leal    7(%rbp), %esi
        addl    $8, %ebp
        movq    %rbx, %rdi
        call    *(%rax)
        addl    %eax, %r12d
        cmpl    $10000, %ebp
        jne     .L39
        popq    %rbx
        popq    %rbp
        movl    %r12d, %eax
        popq    %r12
        ret
{% endhighlight %} 

The code is very similar to that of `Test_Lambda_Std`, except it does not test `16(%rsp)` for zero. This is probably
why it runs a bit faster. However, this code also contains some unnecessary instructions. Register `%rbx` contains
`this`. The first quadword pointed to by `this` is the virtual table pointer. It is loaded by `movq (%rbx), %rax`.
The first quardword inside this table is the address of `f`, it is read and used as a call target in `call *(%rax)`.
Neither `this`, nor its virtual table can change between calls. There is no need to reload the virtual table pointer
or the address of `f`. The compiler could have saved one instruction and two memory accesses per each virtual call
but it didn't. However, the best the compiler could do is to inline `f` into each call and produce code similar to
that for `Test_Macro`.

Perhaps, it is possible to convince the compiler to perform inlining by some magical combination of command line
options. I didn't manage it - maybe any of the readers knows the way?

The AVX
-------

I didn't expect GNU C to use SSE at all, so I specified `-msse4.2` just out of habit. However, we see that the compiler
uses SSE quite successfully. This suggests that it might be capable of using AVX as well. And indeed it is:

    # c++ -std=c++0x -O3 -mavx -falign-functions=32 -falign-loops=32 -funroll-loops -o lambda lambda.cpp -lrt
    # ./lambda 
    10Test_Macro: -1724114088000000: 1131
    10Test_Macro: -1724114088000000: 1125
    11Test_Lambda: -1724114088000000: 1124
    20Test_Lambda_Template: -1724114088000000: 1126
    28Test_Lambda_Template_Capture: -1724114088000000: 17999
    15Test_Lambda_Std: -1724114088000000: 15715
    23Test_Lambda_Std_Capture: -1724114088000000: 18087
    14Test_Interface: -1724114088000000: 14428
    19Test_Abstract_Class: -1724114088000000: 14438

This is the main loop of `Test_Macro::test`:

{% highlight c-objdump %}
.L2:
        vpaddd  %xmm0, %xmm1, %xmm12
        vpmulld (%rax), %xmm1, %xmm13
        vpaddd  %xmm0, %xmm12, %xmm9
        vpaddd  %xmm13, %xmm2, %xmm10
        vpmulld 16(%rax), %xmm12, %xmm11
        vpaddd  %xmm0, %xmm9, %xmm6
        vpaddd  %xmm11, %xmm10, %xmm7
        vpmulld 32(%rax), %xmm9, %xmm8
        vpaddd  %xmm0, %xmm6, %xmm4
        vpaddd  %xmm8, %xmm7, %xmm5
        vpmulld 48(%rax), %xmm6, %xmm3
        vpmulld 64(%rax), %xmm4, %xmm2
        vpaddd  %xmm3, %xmm5, %xmm1
        vpaddd  %xmm0, %xmm4, %xmm15
        vpaddd  %xmm2, %xmm1, %xmm13
        vpaddd  %xmm0, %xmm15, %xmm12
        vpmulld 80(%rax), %xmm15, %xmm14
        vpaddd  %xmm0, %xmm12, %xmm9
        vpmulld 96(%rax), %xmm12, %xmm11
        vpaddd  %xmm0, %xmm9, %xmm6
        vpmulld 112(%rax), %xmm9, %xmm8
        vpaddd  %xmm0, %xmm6, %xmm1
        vpmulld 128(%rax), %xmm6, %xmm4
        vpmulld 144(%rax), %xmm1, %xmm3
        addq    $160, %rax
        vpaddd  %xmm14, %xmm13, %xmm10
        vpaddd  %xmm0, %xmm1, %xmm1
        cmpq    $_ZL3src+40000, %rax
        vpaddd  %xmm11, %xmm10, %xmm7
        vpaddd  %xmm8, %xmm7, %xmm5
        vpaddd  %xmm4, %xmm5, %xmm2
        vpaddd  %xmm3, %xmm2, %xmm2
        jne     .L2
{% endhighlight %} 

Unfortunately, the 256-bit integer arithmetics is only available in AVX2, but the compiler made use of
ternary instructions, which helped remove unnecessary register-to-register moves and caused a significant speedup
(from 1870 ms to 1125 ms).

Pathological case vs regular case
---------------------------------

We see that in our case the solutions involving indirect calls are way too slow, at least on GCC 4.6. The performance
loss is quite big, between 7 and 10 times. The only exception is plain lambda without capture and without use of
`std::function`. The compiler was clever enough to optimise that code.

Perhaps, one can consider our case pathological -- the compiler didn't just inline the function and unroll the loop,
it also vectorised it using SSE. One can't expect this to happen in arbitrary case (although I didn't expect this to happen in
our case either). Let's try to make an example without SSE. The easiest example is a floating point
version of the same code. This is the code generated for `Test_Macro` if we replace integers with floats
and compile for AVX (file [`lambda-float.cpp`]({{ site.REPO-JENSEN-DEVICE }}/blob/master/lambda-float.cpp)):

{% highlight c-objdump %}
        vxorps  %xmm0, %xmm0, %xmm0
        xorl    %eax, %eax
        .p2align 5
.L2:
        vcvtsi2ss       %eax, %xmm6, %xmm6
        leaq    1(%rax), %r10
        leaq    2(%rax), %r9
        leaq    3(%rax), %r8
        leaq    4(%rax), %rdi
        vcvtsi2ss       %r10d, %xmm4, %xmm4
        leaq    5(%rax), %rsi
        vcvtsi2ss       %r9d, %xmm1, %xmm1
        leaq    6(%rax), %rcx
        vcvtsi2ss       %r8d, %xmm14, %xmm14
        leaq    7(%rax), %rdx
        vcvtsi2ss       %edi, %xmm11, %xmm11
        vcvtsi2ss       %esi, %xmm8, %xmm8
        vmulss  _ZL3src(,%rax,4), %xmm6, %xmm5
        addq    $8, %rax
        cmpq    $10000, %rax
        vmulss  _ZL3src(,%r10,4), %xmm4, %xmm3
        vmulss  _ZL3src(,%r8,4), %xmm14, %xmm13
        vmulss  _ZL3src(,%rdi,4), %xmm11, %xmm10
        vaddss  %xmm5, %xmm0, %xmm2
        vmulss  _ZL3src(,%r9,4), %xmm1, %xmm0
        vcvtsi2ss       %ecx, %xmm5, %xmm5
        vmulss  _ZL3src(,%rsi,4), %xmm8, %xmm7
        vaddss  %xmm3, %xmm2, %xmm15
        vcvtsi2ss       %edx, %xmm2, %xmm2
        vaddss  %xmm0, %xmm15, %xmm12
        vaddss  %xmm13, %xmm12, %xmm9
        vmulss  _ZL3src(,%rcx,4), %xmm5, %xmm4
        vmulss  _ZL3src(,%rdx,4), %xmm2, %xmm1
        vaddss  %xmm10, %xmm9, %xmm6
        vaddss  %xmm7, %xmm6, %xmm3
        vaddss  %xmm4, %xmm3, %xmm0
        vaddss  %xmm1, %xmm0, %xmm0
        jne     .L2
        ret
{% endhighlight %} 

There are SSE instructions in the code, but these are scalar instructions. Everything that ends with `ss`, such as
`vmulss` operates on the lowest components of its arguments. What we see is un-vectorised loop unrolled 8 times.

And this is the result (we reduced the iteration count to 100000):

    #c++ -std=c++0x -O3 -mavx -falign-functions=32 -falign-loops=32 -funroll-loops -o lambda-float lambda-float.cpp -lrt
    # ./lambda-float
    10Test_Macro: 3.32996e+16: 938
    10Test_Macro: 3.32996e+16: 910
    11Test_Lambda: 3.32996e+16: 911
    20Test_Lambda_Template: 3.32996e+16: 910
    28Test_Lambda_Template_Capture: 3.32996e+16: 2737
    15Test_Lambda_Std: 3.32996e+16: 2740
    23Test_Lambda_Std_Capture: 3.32996e+16: 2737
    14Test_Interface: 3.32996e+16: 2811
    19Test_Abstract_Class: 3.32996e+16: 2811

The tests with interfaces, abstract classes, captured lambdas and `std::function` are still quite slow. The speed difference
is not as bigh as before (now it is only 3 times), but it is still significant.

Why isn't this code vectorised? Vectorisation changes the order of calculation, which, in the case of floating point,
affects the result. That's why such re-ordering is not allowed by **C** and **C++** language standards.
We can relax the standard compliance by specifying `-funsafe-math-optimizations`,
this will allow vectorisation:

    # c++ -std=c++0x -O3 -mavx -falign-functions=32 -falign-loops=32 -funroll-loops -funsafe-math-optimizations -o lambda-float lambda-float.cpp -lrt
    # ./lambda-float
    10Test_Macro: 3.32996e+16: 249
    10Test_Macro: 3.32996e+16: 228
    11Test_Lambda: 3.32996e+16: 227
    20Test_Lambda_Template: 3.32996e+16: 228
    28Test_Lambda_Template_Capture: 3.32996e+16: 2736
    15Test_Lambda_Std: 3.32996e+16: 2739
    23Test_Lambda_Std_Capture: 3.32996e+16: 2736
    14Test_Interface: 3.32996e+16: 2812
    19Test_Abstract_Class: 3.32996e+16: 2811

Let's run one more test. We'll introduce some really slow calculation into our function, such as square root:

{% highlight C++ %}
    return sum_func(0, SRC_SIZE, [](int i) {return sqrtf((float)i*src[i]); });
{% endhighlight %}

The new code is in file [`lambda-float-2.cpp`]({{ site.REPO-JENSEN-DEVICE }}/blob/master/lambda-float-2.cpp)). This is the result:

    # ./lambda-float-2 
    10Test_Macro: 4.99939e+12: 4396
    10Test_Macro: 4.99939e+12: 4374
    11Test_Lambda: 4.99939e+12: 4374
    20Test_Lambda_Template: 4.99939e+12: 4375
    28Test_Lambda_Template_Capture: 4.99939e+12: 6989
    15Test_Lambda_Std: 4.99939e+12: 7059
    23Test_Lambda_Std_Capture: 4.99939e+12: 7044
    14Test_Interface: 4.99939e+12: 7124
    19Test_Abstract_Class: 4.99939e+12: 7125

The typical slowdown is 60%, which is still not insignificant. We need really slow function (at least ten times slower
than the current one) to make the difference really negligible.

Another way to reduce the difference is to disable loop unrolling. Removing `-funroll-loops` causes following result for
`lambda-float-2`:

    # ./lambda-float-2
    10Test_Macro: 4.99939e+12: 7629
    10Test_Macro: 4.99939e+12: 7614
    11Test_Lambda: 4.99939e+12: 7613
    20Test_Lambda_Template: 4.99939e+12: 7613
    28Test_Lambda_Template_Capture: 4.99939e+12: 7595
    15Test_Lambda_Std: 4.99939e+12: 7802
    23Test_Lambda_Std_Capture: 4.99939e+12: 7596
    14Test_Interface: 4.99939e+12: 7748
    19Test_Abstract_Class: 4.99939e+12: 7750

This may seem cheating (surely slowing down a fast solution to reduce the difference between it and the slow one isn't
the recommended optimisation practice), but it shows that there might be cases where loop unrolling isn't applicable
and the performance loss due to indirect calls isn't so big.

A few words about MSVC
----------------------

MSVC demonstrated similar behaviour, with one important difference. It managed to optimise traditional object-oriented
solutions (`Test_Interface` and `Test_Abstract_Class`), but not the others. Here are the results for `lambda-float`:

    >lambda.exe
    class Test_Macro: 3.32996e+016: 1354
    class Test_Macro: 3.32996e+016: 1315
    class Test_Lambda: 3.32996e+016: 1431
    class Test_Lambda_Template: 3.32996e+016: 1431
    class Test_Lambda_Template_Capture: 3.32996e+016: 5394
    class Test_Lambda_Std: 3.32996e+016: 5485
    class Test_Lambda_Std_Capture: 3.32996e+016: 5388
    class Test_Interface: 3.32996e+016: 1361
    class Test_Abstract_Class: 3.32996e+016: 1325

Conclusions
-----------

- Lambdas and callbacks may be convenient programming tools, but they have a price tag, sometimes high

- This is especially applicable to lambdas with capture and to the use of `std::function` type

- There are cases when callbacks are appropriate. This includes cases where callback function takes a lot of time
  to calculate, cases where host function is large and irregular, and cases where the whole calculation isn't
  performance-critical

- Sometimes the compiler can optimise out the callbacks; however, this is never guarranteed. The optimisations may
  improve with new versions of the compiler, so the best is to check the output of your specific compiler
  
- In general, I would suggest not to use callbacks for performance-critical code; definitely not for Jensen's device

- Nothing can beat a good old macro for performance-critical code

- Call by name must have been equally inefficient in **Algol 60** as well. No wonder this mechanism didn't become
popular, and programmers of the time preferred **FORTRAN** (and many still do).

Coming soon
-----------

**Java** also has various forms of callbacks -- interfaces, abstract classes, reflection. Since version 8 it also has
lambda-definitions. We'll compare the performance of these mechanisms.
