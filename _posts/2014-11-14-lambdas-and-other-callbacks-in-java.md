---
layout: post
title:  "Lambdas and other callbacks in Java"
date:   2014-11-14 12:00:00
tags: Java optimisation
story: callbacks
story-title: "Performance of various callback mechanisms"
---

In [the previous article]({{ site.ART-JENSEN-DEVICE}}) we constructed a **Jensen's device** in **C++**. This device is a procedure that calculates
a sum of arbitrary expression over a range of indices. The expressions are passed to this procedure using various types
of callbacks. **Java** also has several callback mechanisms, and using these mechanisms is a regular and recommended
practice in **Java**. Let's check the performance and compare to **C++**. The code can be found in
[the repository]({{ site.REPO-JAVA-CALLBACK }}).

Direct calculation
------------------

**Java** does not have macros, so we can't perform direct substitution of code into the text of a method. We'll have to
perform this substitution by hand, or, simply speaking, write the calculation directly, without any callbacks:

{% highlight Java %}
static interface Test
{
    public abstract int test();
}

static class Test_Direct implements Test
{
    private final int[] x;
    
    public Test_Direct (int [] x)
    {
        this.x = x;
    }
    
    @Override
    public int test()
    {
        int res = 0;
        for (int i = 0; i < x.length; i++) res += i*x[i];
        return res;
    }
}
{% endhighlight %}

Interfaces
----------

We can pass a function reference using interface:

{% highlight Java %}
static interface F_Interface
{
    public final int f (int i);
}

static int sum_interface (int lo, int hi, F_Interface f)
{
    int res = 0;
    for (int i = 0; i < hi; i++) res += f.f(i);
    return res;
}

static class Test_Interface implements F_Interface, Test
{
    private final int[] x;
    
    public Test_Interface (int [] x)
    {
        this.x = x;
    }

    @Override
    public final int f (int i)
    {
        return i * x[i];
    }
    
    @Override
    public int test()
    {
        return sum_interface (0, x.length, this);
    }
}
{% endhighlight %}

Abstract classes
----------------

Abstract classes can be used instead of interfaces. This is not a recommended practice in **Java**, and we don't
expect much of performance difference, but we'll try it anyway:

{% highlight Java %}
static abstract class F_Abstract_Class
{
    public abstract int f (int i);
}

static int sum_abstract_class (int lo, int hi, F_Abstract_Class f)
{
    int res = 0;
    for (int i = 0; i < hi; i++) res += f.f(i);
    return res;
}

static class Test_Abstract_Class extends F_Abstract_Class implements Test
{
    private final int[] x;
    
    public Test_Abstract_Class (int [] x)
    {
        this.x = x;
    }

    @Override
    public final int f (int i)
    {
        return i * x[i];
    }
    
    @Override
    public int test()
    {
        return sum_abstract_class (0, x.length, this);
    }
}
{% endhighlight %}

We can see why this is not recommended in **Java**: in our example the test class was implementing `Test` interface
and not extending any class. If it did extend some class, it wouldn't be possible to make it extend `F_Abstract_Class`
as well (**Java** does not support multiple inheritance). That's why it is recommended to use interfaces in such cases,
reserving class inheritance to cases where some functionality of base class is used in derived class, or other way around.

Inheritance
-----------

Class inheritance, just like in **C++**, is another way to arrange a callback:

{% highlight Java %}
static abstract class Sum_Inherited
{
    abstract int f (int i);
    
    int sum (int lo, int hi)
    {
        int res = 0;
        for (int i = 0; i < hi; i++) res += f(i);
        return res;
    }
}

static class Test_Inherited extends Sum_Inherited implements Test
{
    private final int[] x;
    
    public Test_Inherited (int [] x)
    {
        this.x = x;
    }

    @Override
    public final int f (int i)
    {
        return i * x[i];
    }
    
    @Override
    public int test()
    {
        return sum (0, x.length);
    }
}
{% endhighlight %}

Reflection
----------

**Java** allows getting hold of methods using their names and invoking them later. This can be done using
reflection mechanism. We have a choice between using static methods (in which case the methods won't have access
to the object's fields), and instance methods (in which case they will). In the latter case an object reference
will be required to invoke the method. We'll try both options. This is the static version:

{% highlight Java %}
static int sum_static_reflection (int lo, int hi, Method f)
{
    try {
        int res = 0;
        for (int i = 0; i < hi; i++) res += (Integer) f.invoke (i);
        return res;
    } catch (Exception e) {
        throw new AssertionError ();
    }
    
}

static class Test_Static_Reflection implements Test
{
    private final Method m;

    public Test_Static_Reflection ()
    {
        try {
            m = this.getClass ().getMethod ("f", int.class);
        } catch (NoSuchMethodException e) {
            throw new AssertionError ();
        }
    }
    
    public static int f (int i)
    {
        return i * src[i];
    }
    
    @Override
    public int test()
    {
        return sum_static_reflection (0, src.length, m);
    }
}
{% endhighlight %}

And this is the non-static version:

{% highlight Java %}
static int sum_reflection (int lo, int hi, Object obj, Method f)
{
    try {
        int res = 0;
        for (int i = 0; i < hi; i++) res += (Integer) f.invoke (obj, i);
        return res;
    } catch (Exception e) {
        throw new AssertionError ();
    }
}

static class Test_Reflection implements Test
{
    private final int[] x;
    private final Method m;

    public Test_Reflection (int [] x)
    {
        this.x = x;
        try {
            m = this.getClass ().getMethod ("f", int.class);
        } catch (NoSuchMethodException e) {
            throw new AssertionError ();
        }
    }
    
    public final int f (int i)
    {
        return i * x[i];
    }
    
    @Override
    public int test()
    {
        return sum_reflection (0, x.length, this, m);
    }
}
{% endhighlight %}

Note that reflection doesn't operate on parameters and results of primitive types. Everything we pass to methods
and everything methods return is objects. That's why parameters are boxed and unboxed for each call, and so is the result.
This will inevitably reduce the performance.

Method pointers
---------------

**Java** does not have function pointers in the same sense as **C**. But since version 1.7 it has method handles,
which behave in a way similar to `Method` objects from reflection API, but are based on the new `invokedynamic`
JVM instruction. I'll make only one `MethodHandle` test, based on a static method:

{% highlight Java %}
static int sum_handle (int lo, int hi, MethodHandle f)
{
    try {
        int res = 0;
        for (int i = 0; i < hi; i++) res += (Integer) f.invoke (i);
        return res;
    } catch (Throwable e) {
        throw new AssertionError ();
    }
}

static class Test_MethodHandle implements Test
{
    private final MethodHandle m;

    public Test_MethodHandle ()
    {
        try {
            MethodHandles.Lookup lookup = MethodHandles.lookup();
            MethodType t = MethodType.methodType (int.class, int.class);
            m = lookup.findStatic (this.getClass (), "f", t);
        } catch (Exception e) {
            e.printStackTrace ();
            throw new AssertionError ();
        }
    }
    
    public static int f (int i)
    {
        return i * src[i];
    }
    
    @Override
    public int test()
    {
        return sum_handle (0, src.length, m);
    }
}
{% endhighlight %}

This solution suffers from the boxing and unboxing overhead just as reflection-based solution does.

Dynamic proxy
-------------

Sometimes we don't want our class to implement the callback interface -- for instance, when that interface isn't
available at compile time. In this case we can use a _dynamic proxy_ mechanism, which creates the class implementing
given interfaces at run time. The test involving the dynamic proxy looks like this:

{% highlight Java %}
static class Test_DynamicProxy implements Test, InvocationHandler
{
    private final int[] x;
    private final F_Interface i;

    public Test_DynamicProxy (int[] x)
    {
        this.x = x;
        i = (F_Interface) Proxy.newProxyInstance (
                              this.getClass ().getClassLoader (),
                              new Class<?> [] {F_Interface.class},
                              this);
    }
    
    private int f (int i)
    {
        return i * x[i];
    }

    @Override
    public Object invoke (Object proxy, Method method, Object[] args)
                         throws Throwable
    {
        return f ((Integer) args[0]);
    }
    
    @Override
    public int test()
    {
        return sum_interface (0, x.length, i);
    }
}
{% endhighlight %}

I don't expect high performance from this test, but it is still interesting to see what the speed is.

Lambda expressions
------------------

Finally, we got to the most exciting part of today's story: the lambda expressions. They have been introduced in
**Java 8**, so we'll need this version to compile the `Lambda` tests. A lambda expression is an anonymous function
that can be used where a functional interface reference is expected. A functional interface is an ordinary **Java**
interface, containing exactly one method declaration, such as our `F_Interface`. This is the test rewritten using
lambda expression:

{% highlight Java %}
static class Test_Lambda implements Test
{
    @Override
    public int test()
    {
        return sum_interface (0, src.length, i -> i * src[i]);
    }
}
{% endhighlight %}

Since `F_Interface` is a functional interface, the `sum_interface` can be used as is,
no changes is necessary. The `i -> i * src[i]` is a lambda expression. It defines an anonymous function
that implements the method from our functional interface. The code looks very neat, perhaps better than the original code from **Algol 60**:

{% highlight Pascal %}
  s := Sum (i, 1, 10, i * x[i]);
{% endhighlight %}

Officially lambda definition is just a syntax decoration, an easy way to create an anonymous inner class. The lambda
call shown above is defined to be completely equivalent to this:

{% highlight Java %}
public int test()
{
    return sum_interface (0, src.length, new F_Interface () {
        public int f (int i)
        {
            return i * src[i];
        }
    });
}
{% endhighlight %}

However, these two samples produce completely different byte code. The version with an anonymous inner class
produces additional class file, called `Lambda$Test_Lambda$1.class`, while the lambda version does not produce
any extra classes. The class is created on the fly. If we replace `src[i]` with `src[i-1]` in the lambda call,
we get the following when running the code:

    Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException: -1
            at Lambda$Test_Lambda.lambda$test$0(Lambda.java:310)
            at Lambda$Test_Lambda$$Lambda$1/41359092.f(Unknown Source)
            at Lambda.sum_interface(Lambda.java:51)
            at Lambda$Test_Lambda.test(Lambda.java:310)
            at Lambda.measure(Lambda.java:337)
            at Lambda.main(Lambda.java:358)

Here `Lambda$Test_Lambda.lambda$test$0` refers to the synthetic function `lambda$test$0` in our class `Test_Lambda`,
which contains the actual text of the lambda definition. This function is present in the class file. The line `Lambda$Test_Lambda$$Lambda$1/41359092.f`
refers to a synthetic class implementing `F_Interface` and its method `f()`, which calls our lambda function. This class
is generated at run time.

Our lambda function does not capture any variables. It operates on the static variable `src`. I'll create two more
tests involving capturing lambdas. The first one will capture an instance field:

{% highlight Java %}
static class Test_Lambda_Capture implements Test
{
    private final int[] x;

    public Test_Lambda_Capture (int [] x)
    {
        this.x = x;
    }

    @Override
    public int test()
    {
        return sum_interface (0, x.length, i -> i * x[i]);
    }
}
{% endhighlight %}

The second one will capture a local variable:

{% highlight Java %}
static class Test_Lambda_Capture_2 implements Test
{
    @Override
    public int test()
    {
        int[] y = src;
        return sum_interface (0, y.length, i -> i * y[i]);
    }
}
{% endhighlight %}

Running the tests
-----------------

Here is the program that runs all the mentioned tests: [`Lambda.java`]({{ site.REPO-JAVA-CALLBACK }}/blob/master/Lambda.java).

Until now I ran all my **Java** tests on **Java 1.7.0_45**. Let's try this again. Obviously, we'll have to comment out
lambda tests, since **Java 1.7** does not support lambdas. The size of the source array will be `10,000`, the number
of iterations `100,000`.

    # java -server  Lambda
    class Lambda$Test_Direct:  562 518 518; sum = -862057044000000
    class Lambda$Test_Interface:  527 518 518; sum = -862057044000000
    class Lambda$Test_Abstract_Class:  524 517 518; sum = -862057044000000
    class Lambda$Test_Inherited:  525 519 518; sum = -862057044000000
    class Lambda$Test_DynamicProxy:  8521 7907 7968; sum = -862057044000000
    class Lambda$Test_Reflection: ^C

The reflection test ran too long, I had to abort it. Let's reduce the iteration count to `1000`:

    class Lambda$Test_Static_Reflection:  2626 2517 2514; sum = -8620570440000
    class Lambda$Test_Reflection:  2278 2263 2256; sum = -8620570440000
    class Lambda$Test_MethodHandle:  40850 40345 40411; sum = -8620570440000

We see that direct computation and all types of simple callbacks are reasonably fast, and, very important, there is
no difference in speed between tests with callbacks and tests without them (`Test_Direct`). Reflection is slow, and method
handles are incredibly slow.

Now let's try **Java** 1.8:

    class Lambda$Test_Direct:  580 517 517; sum = -862057044000000
    class Lambda$Test_Interface:  531 518 518; sum = -862057044000000
    class Lambda$Test_Abstract_Class:  532 518 518; sum = -862057044000000
    class Lambda$Test_Inherited:  529 518 517; sum = -862057044000000
    class Lambda$Test_DynamicProxy:  8933 8399 8378; sum = -862057044000000
    class Lambda$Test_Static_Reflection:  5085 5012 5028; sum = -862057044000000
    class Lambda$Test_Reflection:  20290 20381 20368; sum = -862057044000000
    class Lambda$Test_MethodHandle:  7852 7825 7816; sum = -862057044000000
    class Lambda$Test_Lambda:  3363 3344 3345; sum = -862057044000000
    class Lambda$Test_Lambda_Capture:  3715 3711 3710; sum = -862057044000000
    class Lambda$Test_Lambda_Capture_2:  3369 3350 3351; sum = -862057044000000

We see that the speed of method handles and reflection went up dramatically; lambdas are relatively slow. However,
this is not a final result. Let's comment out all the sophisticated tests, except for `Static_Reflection`:

    class Lambda$Test_Direct:  576 517 517; sum = -862057044000000
    class Lambda$Test_Static_Reflection:  6383 6057 6064; sum = -862057044000000
    class Lambda$Test_Lambda:  533 473 473; sum = -862057044000000
    class Lambda$Test_Lambda_Capture:  590 517 518; sum = -862057044000000
    class Lambda$Test_Lambda_Capture_2:  3352 3350 3350; sum = -862057044000000

Suddenly two of the lambda versions run fast, and the ordinary lambda even runs faster than a direct loop test!
`Static_Reflection` is somehow important. If we comment it out, we get

    # java  Lambda
    class Lambda$Test_Direct:  577 517 517; sum = -862057044000000
    class Lambda$Test_Lambda:  647 587 587; sum = -862057044000000
    class Lambda$Test_Lambda_Capture:  587 516 516; sum = -862057044000000
    class Lambda$Test_Lambda_Capture_2:  3357 3359 3353; sum = -862057044000000

The plain lambda test still runs fast, but not faster than the direct test.

Finally, let's change the order of tests and repeat them twice:

    # java  Lambda
    class Lambda$Test_Direct:  595 517 518; sum = -862057044000000
    class Lambda$Test_Lambda_Capture_2:  642 586 585; sum = -862057044000000
    class Lambda$Test_Lambda:  517 474 474; sum = -862057044000000
    class Lambda$Test_Lambda_Capture:  3721 3716 3708; sum = -862057044000000
    class Lambda$Test_Lambda_Capture_2:  474 474 474; sum = -862057044000000
    class Lambda$Test_Lambda:  475 474 474; sum = -862057044000000
    class Lambda$Test_Lambda_Capture:  3708 3705 3708; sum = -862057044000000

We see that the test that was slow (`Test_Lambda_Capture_2`) is now fast, so the test in this case depends
on the test's position in run sequence rather than on the type of used lambda.

This instability is very worrying and requires further investigation. It falls, however, outside today's topic,
so we'll make a note and return to this issue later.

Let's collect all the results in one table, adding those from **C++** (see previous article). There is no direct
translation of **C++** tests into **Java** tests, so we'll have to make some voluntary decisions. For instance, I wouldn't map
**C** function pointers to **Java**'s method handles, since they are implemented using totally different mechanisms.
I'll use the fastest lambda times for **C++** and **Java**. We mustn't forget that **C++** tests were run with the
iteration count of `1,000,000`, while `100,000` was used in **Java**. I'll calibrate the results for `1,000,000`.


<table class="numeric">
<tr><th>     Test                         </th><th> Time, C++</th><th> Time, Java 7</th><th> Time, Java 8 </th></tr>
<tr><td class="ttext">Direct              </td><td>  1,125 </td>    <td>      5,180  </td><td>      5,170 </td></tr>
<tr><td class="ttext">Interface           </td><td>        </td>    <td>      5,180  </td><td>      5,180 </td></tr>
<tr><td class="ttext">Abstract class      </td><td> 14,428 </td>    <td>      5,180  </td><td>      5,180 </td></tr>
<tr><td class="ttext">Inheritance         </td><td> 14,438 </td>    <td>      5,180  </td><td>      5,170 </td></tr>
<tr><td class="ttext">Lambda              </td><td>  1,124 </td>    <td>             </td><td>      4,740 </td></tr>
<tr><td class="ttext">Lambda with capture </td><td> 17,999 </td>    <td>             </td><td>      4,740 </td></tr>
<tr><td class="ttext">Dynamic proxy       </td><td>        </td>    <td>     79,680  </td><td>     83,780 </td></tr>
<tr><td class="ttext">Static reflection   </td><td>        </td>    <td>    251,400  </td><td>     50,280 </td></tr>
<tr><td class="ttext">Reflection          </td><td>        </td>    <td>    225,600  </td><td>    203,680 </td></tr>
<tr><td class="ttext">Method handle       </td><td>        </td>    <td>  4,041,100  </td><td>     78,160 </td></tr>
</table>

The method handle performance difference is shocking. It became fifty times faster in **Java 8**, which means that
the implementation in **Java 7** was a prototype rather than a production-quality job. The performance of static reflection calls has also improved a lot.
We can see that peak performance (when everything is inlined) is much higher in **C++**, but this has a simple
explanation: the **C++** compiler performed vectorisation and calculated the result in SSE registers. We wouldn't
expect the same from just-in-time compiler in **Java** VM. At the same time, we see that **Java** is much better
in inlining than **C++**: it has successfully inlined all callbacks based on object orientation and was capable
to inline both types of lambdas -- with and without capture. Sometimes lambda tests were even faster than
direct loops -- sounds like a miracle. Unfortunately, not all lambda tests were that fast, some took as long as 37100 ms
to run.

Let's modify the test to eliminate the effect of SSE vectorisation. We know that **C++** does not perform it for floats,
unless we switch on unsafe mathematics. Let's rewrite the **Java** example to floats and run it on **Java 8**
(see [`FLambda.java`]({{ site.REPO-JAVA-CALLBACK }}/blob/master/FLambda.java)).

    # java -server FLambda
    class FLambda$Test_Direct:  1076 1026 1026; sum = 1.66690911E17
    class FLambda$Test_Interface:  1154 1139 1139; sum = 1.66690911E17
    class FLambda$Test_Abstract_Class:  1153 1139 1139; sum = 1.66690911E17
    class FLambda$Test_Inherited:  1151 1140 1140; sum = 1.66690911E17
    class FLambda$Test_DynamicProxy:  4878 4107 4107; sum = 1.66690911E17
    class FLambda$Test_Static_Reflection:  5355 5263 5264; sum = 1.66690911E17
    class FLambda$Test_Reflection:  20430 20439 20451; sum = 1.66690911E17
    class FLambda$Test_MethodHandle:  7309 7282 7289; sum = 1.66690911E17
    class FLambda$Test_Lambda:  3364 3344 3344; sum = 1.66690911E17
    class FLambda$Test_Lambda_Capture:  3627 3588 3588; sum = 1.66690911E17
    class FLambda$Test_Lambda_Capture_2:  3393 3360 3359; sum = 1.66690911E17

If we comment out everything except `Test_Direct` and all the lambdas, we get:

    # java -server FLambda
    class FLambda$Test_Direct:  1080 1025 1025; sum = 1.66690911E17
    class FLambda$Test_Lambda:  1429 1365 1365; sum = 1.66690911E17
    class FLambda$Test_Lambda_Capture:  1840 1778 1778; sum = 1.66690911E17
    class FLambda$Test_Lambda_Capture_2:  3368 3381 3360; sum = 1.66690911E17

In a table form (again, adjusted for `1,000,000` iterations):

<table class="numeric">
<tr><th>     Test                         </th><th> Time, C++</th><th> Time, Java 8 </th></tr>
<tr><td class="ttext">Direct              </td><td>  9,100 </td>    <td>   10,260 </td></tr>
<tr><td class="ttext">Interface           </td><td>        </td>    <td>   11,390 </td></tr>
<tr><td class="ttext">Abstract class      </td><td> 28,110 </td>    <td>   11,390 </td></tr>
<tr><td class="ttext">Inheritance         </td><td> 28,110 </td>    <td>   11,400 </td></tr>
<tr><td class="ttext">Lambda              </td><td>  9,100 </td>    <td>   13,650 </td></tr>
<tr><td class="ttext">Lambda with capture </td><td> 27,370 </td>    <td>   17,780 </td></tr>
<tr><td class="ttext">Dynamic proxy       </td><td>        </td>    <td>   41,070 </td></tr>
<tr><td class="ttext">Static reflection   </td><td>        </td>    <td>   52,640 </td></tr>
<tr><td class="ttext">Reflection          </td><td>        </td>    <td>  204,510 </td></tr>
<tr><td class="ttext">Method handle       </td><td>        </td>    <td>   72,890 </td></tr>
</table>

A direct test runs virtually at the same speed in **Java** as in **C++**. Inlining of virtual calls is still
better in **Java**. Lambdas are a bit slower than direct loops this time, but lambdas with capture are faster
than in **C++** (however, the unexpected behaviour where lambdas are slow still occurs). What is surprising is
the faster execution of dynamic proxy tests.

Conclusions
-----------

- Lambda performance is unstable in **Java 8**. It can be as fast as a direct loop, or even faster, but it can also be
  quite slow.

- Unlike in **C++**, adding capture to a lambda does not slow down execution

- **Java** is very good in inlining hot procedure calls and resolving virtual methods

- **C++** code can be much faster than **Java** code if a compiler manages to use SSE instructions; otherwise
  the execution speed is likely to be very similar

- In general, performance of lambdas in **Java** is satisfactory

- Exotic forms of callback, such as reflection, dynamic proxy and method handles, are relatively slow. They
  are appropriate for the cases where flexibility is more important than performance.
