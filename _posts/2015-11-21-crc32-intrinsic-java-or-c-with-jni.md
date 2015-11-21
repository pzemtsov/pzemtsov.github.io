---
layout: post
title:  "CRC32: intrinsic, Java or C with JNI?"
date:   2015-11-21 12:00:00
tags: Java optimisation hashmap life
story: life
story-title: "Conway's Game of Life"
---

Today we are going to try to optimise a hash-function based on CRC32 calculation. Here is some background.

In ["{{ site.TITLE-HASH }}"]({{ site.ART-HASH }}) we implemented [Conway's game of Life](http://en.wikipedia.org/wiki/Conway's_Game_of_Life) using **Java**'s `HashMap`.
We discovered that performance depends on a choice of a hash function and identified some of them that work well (spread hash values evenly). Those included a function based
on calculating a remainder and another one that used [CRC32](https://en.wikipedia.org/wiki/Cyclic_redundancy_check).
Then we measured the speed of the hash functions and tried optimising
the remainder-based one (in ["{{ site.TITLE-DIVISION-HASH }}"]({{ site.ART-DIVISION-HASH }}) and ["{{ site.TITLE-DIVISION-HASH-MORE }}"]({{ site.ART-DIVISION-HASH-MORE }})).
These attempts failed, because the optimisation performed by the HotSpot compiler was better than anything we could do by hand. However, I don't regret that effort --
we've learned something interesting in the process. Among other things, we discovered (see
["{{ site.TITLE-MICROBENCHMARK }}"]({{ site.ART-MICROBENCHMARK }})) that the results of micro-benchmark tests do not always correlate with the results
of full program tests, so micro-benchmarks must be used with care.

Nevertheless, today we are going to run micro-benchmarks again, this time on a CRC32-based hash function. Having learned the lesson, we'll later verify the results
on the full program test. Obviously, the same may happen that happened with the remainder-based function: the compiler may already be very good in compiling it.
We can only hope that even in this case we'll learn something interesting. Let's go.

The method in question
----------------------

The full code of the project is [here]({{ site.REPO-LIFE }}/tree/0161781215b7e713e7222a21f628c29b6764b778). This is the function we are talking about -- `LongPoint7.hashCode()`:

{% highlight Java %}
    public int hashCode ()
    {
        CRC32 crc = new CRC32 ();
        crc.update ((int)(v>>> 0) & 0xFF);
        crc.update ((int)(v >>> 8) & 0xFF);
        crc.update ((int)(v >>> 16) & 0xFF);
        crc.update ((int)(v >>> 24) & 0xFF);
        crc.update ((int)(v >>> 32) & 0xFF);
        crc.update ((int)(v >>> 40) & 0xFF);
        crc.update ((int)(v >>> 48) & 0xFF);
        crc.update ((int)(v >>> 56) & 0xFF);
        return (int) crc.getValue ();
    }
{% endhighlight %}

Here are the execution times, in milliseconds for 100M iterations (for comparison, I added times for one other hash function and for the null function, which does nothing):

<table class="numeric">
<tr><th>Class name</th>                <th>Comment</th>                           <th>Time, <b>Java 7</b></th><th>Time, <b>Java 8</b></th></tr>
<tr><td class="label">LongPoint5</td>  <td class="ttext">Multiply by one big prime         </td><td>  547</td><td>  486</td></tr>
<tr><td class="label">LongPoint7</td>  <td class="ttext"><code>java.util.zip.CRC32</code>  </td><td>10109</td><td> 1882</td></tr>
<tr><td class="label">NullPoint</td>   <td class="ttext">Null function (does nothing)      </td><td>  532</td><td>  455</td></tr>
</table>

Our hash function is much slower than the others, especially on **Java 7**. Let's see what can be done here.

The first ideas
---------------

The `CRC32` class we are using comes from `java.util.zip`. Let's look inside this class.

This is the code of `update()` in both **Java 7** and **Java 8**:

{% highlight Java %}
    public void update(int b) {
        crc = update(crc, b);
    }

    private native static int update(int crc, int b);
{% endhighlight %}

A native call is made for every input byte, and we know that native calls are expensive.
This suggests our first idea for improvement: to make just one native call for the entire `long`.
The `CRC32` class contains a method we can use for that (at a cost of an array allocation):

{% highlight Java %}
    public void update(byte[] b, int off, int len) {
        if (b == null) {
            throw new NullPointerException();
        }
        if (off < 0 || len < 0 || off > b.length - len) {
            throw new ArrayIndexOutOfBoundsException();
        }
        crc = updateBytes(crc, b, off, len);
    }

    private native static int updateBytes(int crc, byte[] b, int off, int len);
{% endhighlight %}

[Here is the new code]({{ site.REPO-LIFE }}/blob/2a500aaa1164f05cace692f17841c0522fca1ccb/LongPoint71.java) (class `LongPoint71`):

{% highlight Java %}
    @Override
    public int hashCode ()
    {
        byte[] b = new byte[8];
        b[0] = (byte) (v >>>  0);
        b[1] = (byte) (v >>>  8);
        b[2] = (byte) (v >>> 16);
        b[3] = (byte) (v >>> 24);
        b[4] = (byte) (v >>> 32);
        b[5] = (byte) (v >>> 40);
        b[6] = (byte) (v >>> 48);
        b[7] = (byte) (v >>> 56);
        CRC32 crc = new CRC32 ();
        crc.update (b,  0,  8);
        return (int) crc.getValue ();
    }
{% endhighlight %}

One immediate observation about this code is that we don't have to allocate a new byte array each time and can re-use an existing one. This may seem obvious that this
way it will be faster but I won't be convinced until I try. Here is the new class [`LongPoint72`]({{ site.REPO-LIFE }}/blob/2a500aaa1164f05cace692f17841c0522fca1ccb/LongPoint72.java):

{% highlight Java %}
    private byte[] b = new byte[8];
    
    @Override
    public int hashCode ()
    {
        b[0] = (byte) (v >>>  0);
        b[1] = (byte) (v >>>  8);
        b[2] = (byte) (v >>> 16);
        b[3] = (byte) (v >>> 24);
        b[4] = (byte) (v >>> 32);
        b[5] = (byte) (v >>> 40);
        b[6] = (byte) (v >>> 48);
        b[7] = (byte) (v >>> 56);
        CRC32 crc = new CRC32 ();
        crc.update (b,  0,  8);
        return (int) crc.getValue ();
    }
{% endhighlight %}

Another improvement that seems obvious is to re-use the `CRC32` object as well. We'll try this, too, but there is one important point to note: after this the `hashCode()` function
stops being thread-safe. Two threads using the same `CRC32` object may corrupt each other's calculations. Sharing the byte array does not cause this, because the functions write
the same values into the elements of this array. Our program is single-threaded, but it is preferable to write more generally applicable code. Still, we'll try this option, too
(see [`LongPoint73`]({{ site.REPO-LIFE }}/blob/2a500aaa1164f05cace692f17841c0522fca1ccb/LongPoint73.java)):

{% highlight Java %}
    private byte[] b = new byte[8];
    private CRC32 crc = new CRC32 ();
    
    @Override
    public int hashCode ()
    {
        b[0] = (byte) (v >>>  0);
        b[1] = (byte) (v >>>  8);
        b[2] = (byte) (v >>> 16);
        b[3] = (byte) (v >>> 24);
        b[4] = (byte) (v >>> 32);
        b[5] = (byte) (v >>> 40);
        b[6] = (byte) (v >>> 48);
        b[7] = (byte) (v >>> 56);
        crc.reset ();
        crc.update (b,  0,  8);
        return (int) crc.getValue ();
    }
{% endhighlight %}

Now it's time to start running the programs (the main class is `HashTime`):

Here are the results:


<table class="numeric">
<tr><th>Name</th><th>Description</th><th>Time, <b>Java&nbsp;7</b></th><th>Time, <b>Java&nbsp;8</b></th></tr>
<tr><td class="label">LongPoint7</td><td class="ttext">CRC32, 8 calls one byte each  </td><td>10109</td><td>1882</td></tr>
<tr><td class="label">LongPoint71</td><td class="ttext">CRC32, one call with an array </td><td>6730</td><td>2056</td></tr>
<tr><td class="label">LongPoint72</td><td class="ttext">CRC32, one call with re-used array </td><td>6422</td><td>1901</td></tr>
<tr><td class="label">LongPoint73</td><td class="ttext">CRC32, one call with re-used array and CRC </td><td>6566</td><td>2009</td></tr>
</table>

- The optimisation helped on **Java 7** (we've got 33% improvement), but not on **Java 8**, where the time got worse (it resembles the story with the remainder-based hash, doesn't it?)

- Re-using the array have helped a bit

- Not only is re-using of CRC object thread-unsafe, it is also slower than allocating new one each time. Object allocation isn't always slow!

From arrays to buffers
----------------------

Our next step is replacing an array with a direct byte buffer. This offers two potential advantages:

- Direct byte buffers support fast decomposition of primitive values into bytes. The entire `long` can be written into a buffer using just one CPU instruction

- JNI has built-in support for direct buffers, and passing them to the native code is faster than passing arrays.

Unfortunately, only **Java 8** contains a method that applies CRC32 to a byte buffer. Here it is:

{% highlight Java %}
    public void update(ByteBuffer buffer) {
        int pos = buffer.position();
        int limit = buffer.limit();
        assert (pos <= limit);
        int rem = limit - pos;
        if (rem <= 0)
            return;
        if (buffer instanceof DirectBuffer) {
            crc = updateByteBuffer(crc, ((DirectBuffer)buffer).address(), pos, rem);
        } else if (buffer.hasArray()) {
            crc = updateBytes(crc, buffer.array(), pos + buffer.arrayOffset(), rem);
        } else {
            byte[] b = new byte[rem];
            buffer.get(b);
            crc = updateBytes(crc, b, 0, b.length);
        }
        buffer.position(limit);
    }

    private native static int updateByteBuffer(int adler, long addr,
                                               int off, int len);
{% endhighlight %}

This method doesn't even use the JNI support for direct buffers.
Instead, it takes the address of the underlying memory and passes it to the native code as a number. This trick
promises very high performance. 

Let's write a version that uses this method and run it on **Java 8**.
[Here is the code]({{ site.REPO-LIFE }}/blob/f23fcdee9191717969294d72c8cd583278824d53/LongPoint74.java) (`LongPoint74`):

{% highlight Java %}
    static private ByteBuffer buf = ByteBuffer.allocateDirect (8)
                                              .order (ByteOrder.LITTLE_ENDIAN);
    
    @Override
    public int hashCode ()
    {
        CRC32 crc = new CRC32 ();
        buf.putLong (0,  v);
        buf.position (0);
        crc.update (buf);
        return (int) crc.getValue ();
    }
{% endhighlight %}

Note that this code went one step further in re-using the same object than `LongPoint73`: it made the buffer a static object. There is a reason for it: direct buffers are allocated
in separate (native) memory, and the minimal quantum of allocation is 4096 bytes. They are freed when their **Java** objects get garbage-collected, over which we have
no control. If **Java** heap is big enough, too many **Java** objects may be allocated before the garbage collector is called, and we may run out of native memory.
Obviously, the static buffer causes this method to be thread-unsafe.

The results, however, are not impressive:

<table class="numeric">
<tr><th>Name</th><th>Description</th><th>Time, <b>Java&nbsp;7</b></th><th>Time, <b>Java&nbsp;8</b></th></tr>
<tr><td class="label">LongPoint7</td><td class="ttext">CRC32, 8 calls one byte each  </td><td>10109</td><td>1882</td></tr>
<tr><td class="label">LongPoint72</td><td class="ttext">CRC32, one call with re-used array </td><td>6422</td><td>1901</td></tr>
<tr><td class="label">LongPoint74</td><td class="ttext">CRC32, one call with a direct byte buffer </td><td></td><td>1903</td></tr>
</table>

We won't even try re-using the CRC object.

Writing in Java
---------------

What else can we do? It seems like all our attempts have failed. No, there are other options to try. The first one is to write the entire CRC32 calculation in **Java**. It is unlikely to
be fast (if it was fast enough, the JDK designers would have used this approach), but completeness requires us to test this anyway. Fortunately, we don't need to implement
CRC32 according to its strict definition (division of polynoms). There is a fast table-based implementation
(see [`TableCRC32.java`]({{ site.REPO-LIFE }}/blob/75e4a5986c46434f4d4ccf6af36bc695f2b7fe38/TableCRC32.java)). This is what the application of CRC to a byte
and to a `long` look like:

{% highlight Java %}
     public static int update (byte b, int crc)
     {
         return (crc >>> 8) ^ crc_table[(crc ^ b) & 0xff];
     }

     public static int update (long n, int crc)
     {
         return update ((byte) (n >>> 56),
                update ((byte) (n >>> 48),
                update ((byte) (n >>> 40),
                update ((byte) (n >>> 32),
                update ((byte) (n >>> 24),
                update ((byte) (n >>> 16),
                update ((byte) (n >>>  8),
                update ((byte) (n >>>  0), crc))))))));
     }

     public static int crc32 (long n)
     {
         return update (n, -1) ^ -1;
     }
{% endhighlight %}

And this is our [new hash function]({{site.REPO-LIFE }}/blob/75e4a5986c46434f4d4ccf6af36bc695f2b7fe38/LongPoint75.java):

{% highlight Java %}
    @Override
    public int hashCode ()
    {
        return TableCRC32.crc32 (v);
    }
{% endhighlight %}

The results look much better:

<table class="numeric">
<tr><th>Name</th><th>Description</th><th>Time, <b>Java&nbsp;7</b></th><th>Time, <b>Java&nbsp;8</b></th></tr>
<tr><td class="label">LongPoint7</td><td class="ttext">CRC32, 8 calls one byte each  </td><td>10109</td><td>1882</td></tr>
<tr><td class="label">LongPoint72</td><td class="ttext">CRC32, one call with re-used array </td><td>6422</td><td>1901</td></tr>
<tr><td class="label">LongPoint75</td><td class="ttext">CRC32, written in <b>Java</b> </td><td>1505</td><td>1420</td></tr>
</table>

Our prediction was wrong -- this is definitely an improvement. The time on **Java 7** has improved dramatically,
while on **Java 8** we've improved over the original time for the first time. It is unclear why JDK does not use **Java** implementation of CRC32.

Writing in C
------------

The next thing we can try is rewrite the same code in **C**. Most probably, the original native code of **CRC32** is very similar to what we are going to write,
but one trick may give us an advantage: we are going to introduce a JNI call that operates on a `long`. No arrays or byte buffers necessary. For the purpose of this article
we'll create [a simple class with a single native method]({{ site.REPO-LIFE }}/blob/3e26e50861210b163f9e30beb89af3b6a2eb2543/NativeCRC32.java):

{% highlight Java %}
public class NativeCRC32
{
    public static native int crc32 (long n);
}
{% endhighlight %}

Obviously, the proper implementation must have many other useful methods, but we'll implement only this one, to keep things short.

We'll need a **C** file, called [`NativeCRC32.c`]({{ site.REPO-LIFE }}/blob/3e26e50861210b163f9e30beb89af3b6a2eb2543/NativeCRC32.c),
with the table-based CRC implementation and a JNI wrapper:

{% highlight C++ %}
#define CRC32(c,d) (c = (c >> 8) ^ crc_c [(c^(d)) & 0xFF] )

JNIEXPORT jint JNICALL Java_NativeCRC32_crc32 (JNIEnv *env, jclass clazz,
                                                            jlong x)
{
    uint32_t c = ~0;
    CRC32 (c, x >> 0);
    CRC32 (c, x >> 8);
    CRC32 (c, x >> 16);
    CRC32 (c, x >> 24);
    CRC32 (c, x >> 32);
    CRC32 (c, x >> 40);
    CRC32 (c, x >> 48);
    CRC32 (c, x >> 56);
    return (jint) (c ^ ~0);
}
{% endhighlight %}

To build it, we use a rather sophisticated command line:

    cc -m64 -Wall -O3 -fPIC -I${JAVA_HOME}/include -I${JAVA_HOME}/include/linux
    -shared -o libNativeCRC32.so NativeCRC32.c
    
The compiler produces library `libNativeCRC32.so`, which will be loaded when the class `NativeCRC32` is initialised.

Here are the results:

<table class="numeric">
<tr><th>Name</th><th>Description</th><th>Time, <b>Java&nbsp;7</b></th><th>Time, <b>Java&nbsp;8</b></th></tr>
<tr><td class="label">LongPoint72</td><td class="ttext">CRC32, one call with re-used array </td><td>6422</td><td>1901</td></tr>
<tr><td class="label">LongPoint75</td><td class="ttext">CRC32, written in <b>Java</b> </td><td>1505</td><td>1420</td></tr>
<tr><td class="label">LongPoint76</td><td class="ttext">CRC32, written in <b>C</b>, one JNI call </td><td>1890</td><td>1898</td></tr>
</table>

The results are not great, in the case of **Java 8** they don't differ much from those of our version using an array. In both cases they are worse than our results
in **Java**. This disproves a popular belief that **C** is always faster than **Java** -- at least, when JNI is involved.

Writing in assembly
-------------------

It looks like we've reached a limit. What more can one do when we already tried to write everything in **C**? Only one thing: write in assembly. OK, it won't be a true assembly; **C** with
SSE intrinsics will be sufficient for our needs.

Intel processors (starting at SSE 4.2) have an instruction calculating CRC32. This, however, is a different version of CRC32, called CRC32C (Castagnolli). It differs
from our CRC32 in the polynom it uses, but the general principles are the same, and it can be calculated by the same table-based code (except the table is different).
We didn't measure hash values distribution when using CRC32C, but let's hope it is good. We'll add another function to our `NativeCRC32` class:

{% highlight Java %}
public class NativeCRC32
{
    public static native int crc32c (long n);
}
{% endhighlight %}

{% highlight C++ %}
#include <nmmintrin.h>

JNIEXPORT jint JNICALL Java_NativeCRC32_crc32c (JNIEnv *env, jclass clazz,
                                                             jlong x)
{
    return (jint) (_mm_crc32_u64(~0, x) ^ ~0);
}
{% endhighlight %}

We'll also need another class, [`LongPoint77`]({{ site.REPO-LIFE }}/blob/3e26e50861210b163f9e30beb89af3b6a2eb2543/LongPoint77.java), to use this function:

{% highlight Java %}
    @Override
    public int hashCode ()
    {
        return NativeCRC32.crc32c (v);
    }
{% endhighlight %}


Finally, we need to add `-msse4.2` to our build command, and we are ready to run:

<table class="numeric">
<tr><th>Name</th><th>Description</th><th>Time, <b>Java&nbsp;7</b></th><th>Time, <b>Java&nbsp;8</b></th></tr>
<tr><td class="label">LongPoint75</td><td class="ttext">CRC32, written in <b>Java</b> </td><td>1505</td><td>1420</td></tr>
<tr><td class="label">LongPoint76</td><td class="ttext">CRC32, written in <b>C</b>, one JNI call </td><td>1890</td><td>1898</td></tr>
<tr><td class="label">LongPoint77</td><td class="ttext">CRC32C, written in <b>C</b> using SSE, one JNI call </td><td>1257</td><td>1214</td></tr>
</table>

This is much better than anything we've seen before. On **Java 7** this version runs faster by 87% (that is, eight times faster) than the original one.
On **Java 8** the speedup is smaller, only by 35%. Unfortunately, even such a speedup can't be considered a success: other hash functions take 400-600 ms with similar
quality of hashing, and we've exhausted our list of ideas for CRC32 optimisation.

The cost of JNI
---------------

It still looks strange that we've achieved such a low speed for a hash function consisting of just one CPU instruction (`CRC32`). Maybe, this is a very slow instruction?
No, it has the latency of 3 cycles (according to [Agner Fog](http://www.agner.org/optimize/instruction_tables.pdf)). The only possible explanation can be the cost of the JNI
call. Let's check this. We'll add another function to our native code:


{% highlight C++ %}
JNIEXPORT jint JNICALL Java_NativeCRC32_crc0 (JNIEnv *env, jclass clazz,
                                                           jlong x)
{
    return 0;
}
{% endhighlight %}

Here are the results:

<table class="numeric">
<tr><th>Name</th><th>Description</th><th>Time, <b>Java&nbsp;7</b></th><th>Time, <b>Java&nbsp;8</b></th></tr>
<tr><td class="label">LongPoint75</td><td class="ttext">CRC32, written in <b>Java</b> </td><td>1505</td><td>1420</td></tr>
<tr><td class="label">LongPoint76</td><td class="ttext">CRC32, written in <b>C</b>, one JNI call </td><td>1890</td><td>1898</td></tr>
<tr><td class="label">LongPoint77</td><td class="ttext">CRC32C, written in <b>C</b> using SSE, one JNI call </td><td>1257</td><td>1214</td></tr>
<tr><td class="label">LongPoint78</td><td class="ttext">JNI call to an empty function</td><td>1256</td><td>1214</td></tr>
</table>

This looks shocking at first: the results look as if the CRC calculation didn't require any CPU cycles. This can be explained by the parallel structure of our processor:
while CRC32 is busy executing and as long as its result isn't used, the rest of the program can carry on. The JNI exit sequence may take longer than 3 cycles, so the CRC32
instruction can totally disappear from our timing.

Our measurements also show that JNI isn't cheap. Let's look at **Java 8**. The test with empty JNI call (`LongPoint78`) executed for 1214 ms. The test with the empty function
(`NullPoint`) took 455 ms (these are the expenses of the virtual call plus the test overheads). This means that the JNI call to the static method taking a `long` as a parameter
and returning an `int`, executed 100 million times, took 759 ms, or about 7.6 ns per execution. On this processor it corresponds to 20 CPU cycles. This is relatively small overhead for
methods that take thousands of cycles to execute, but it is too big for short methods. In the case of CRC32C the overheads of JNI were much bigger than the cost of the method
itself, while in the case of the CRC32 written in C they were roughly the same (the calculation took about 18 cycles). In short, nothing based on JNI is a good solution for a
hash function -- unless we need to hash objects of kilobyte or megabyte sizes.

We can now see why the original version took so long in **Java 7**. Eight JNI calls alone take 5800 ms. Why is then **Java 8**'s version faster than that?

The mystery of fast CRC32 on Java 8
-----------------------------------

To answer this question, we'll have to look at the assembly listings of `LongPoint7.hashCode`. In **Java 7** the code of this method contains eight calls to `CRC32.update`:

{% highlight c-objdump %}
  0x00007f99ad0736cf: mov    r10,QWORD PTR [rsi+0x10]
  0x00007f99ad0736d3: mov    edx,r10d
  0x00007f99ad0736d6: movzx  edx,dl             ;*iand
                                                ; - LongPoint7::hashCode@19 (line 23)
  0x00007f99ad0736d9: xor    esi,esi
  0x00007f99ad0736db: call   0x00007f99ad037f60  ; OopMap{rbp=Oop off=64}
                                                ;*invokestatic update
                                                ; - java.util.zip.CRC32::update@6 (line 52)
                                                ; - LongPoint7::hashCode@20 (line 23)
                                                ;   {static_call}
  0x00007f99ad0736e0: mov    DWORD PTR [rsp],eax  ;*synchronization entry
                                                ; - java.util.zip.CRC32::update@-1 (line 52)
                                                ; - LongPoint7::hashCode@36 (line 24)
  0x00007f99ad0736e3: mov    r10,QWORD PTR [rbp+0x10]
  0x00007f99ad0736e7: shr    r10,0x8
  0x00007f99ad0736eb: mov    edx,r10d
  0x00007f99ad0736ee: movzx  edx,dl             ;*iand
                                                ; - LongPoint7::hashCode@35 (line 24)
  0x00007f99ad0736f1: mov    esi,eax
  0x00007f99ad0736f3: call   0x00007f99ad037f60  ; OopMap{rbp=Oop off=88}
                                                ;*invokestatic update
                                                ; - java.util.zip.CRC32::update@6 (line 52)
                                                ; - LongPoint7::hashCode@36 (line 24)
                                                ;   {static_call}
{% endhighlight %}

This fragment shows only two calls to `update()`. The entire code is [here]({{ site.REPO-LIFE }}/blob/a44e2a418d046cef6f21cac4da52260962875dd4/java7.LongPoint7.hashCode.asm).
 
The code for `CRC32.update` contains calls to the runtime, which probably represent JNI call.

The code on **Java 8** looks completely different. It is much longer and more complex.
The entire code is [here]({{ site.REPO-LIFE }}/blob/a44e2a418d046cef6f21cac4da52260962875dd4/java8.LongPoint7.hashCode.asm), I'll show the **Java** reconstruction:

{% highlight Java %}
public int hashCode()
{
    int r10d = (v >>> 56) & 0xFF;
    int rsi  = ~ (v & 0xFF) & 0xFF;
    int ebx  = (v >>> 48) & 0xFF;
    int r11d = (v >>> 32) & 0xFF;
    int edi  = (v >>> 40) & 0xFF;
    int r9d  = (v >>> 24) & 0xFF;
    int ebp  = (v >>> 16) & 0xFF;
    int ecx  = (v >>> 8) & 0xFF;
    int eax  = table [rsi];
    ecx  ^= eax;
    eax  ^= 0xFFFFFF;
    ecx  ^= 0xFFFFFF;
    eax  >>>= 8;
    eax  ^= table [ecx];
    ebp  ^= eax;
    eax  >>>= 8;
    eax  ^= table [ebp & 0xFF];
    r9d  ^= eax;
    eax  >>>= 8;
    eax  ^= table [r9d & 0xFF];
    r11d ^= eax;
    eax  >>>= 8;
    eax  ^= table [r11d & 0xFF];
    edi  ^= eax;
    eax  >>>= 8;
    eax  ^= table [edi & 0xFF];
    ebx  ^= eax;
    eax  >>>= 8;
    eax  ^= table [ebx & 0xFF];
    r10d ^= eax;
    eax  >>>= 8;
    eax  ^= table [r10d & 0xFF];
    eax  = ~eax;
    return eax;
}
{% endhighlight %}

Here `table` is some statically defined array of type `int[]`. This code looks odd at first, but if we simplify it, it will look very familiar:

{% highlight Java %}
public int hashCode()
{
    crc  = 0xFFFFFF ^ table [~ v & 0xFF];
    crc = (crc >>> 8) ^ table [((v >>> 8)  ^ crc) & 0xFF];
    crc = (crc >>> 8) ^ table [((v >>> 16) ^ crc) & 0xFF];  
    crc = (crc >>> 8) ^ table [((v >>> 24) ^ crc) & 0xFF];  
    crc = (crc >>> 8) ^ table [((v >>> 32) ^ crc) & 0xFF];  
    crc = (crc >>> 8) ^ table [((v >>> 40) ^ crc) & 0xFF];  
    crc = (crc >>> 8) ^ table [((v >>> 48) ^ crc) & 0xFF];  
    crc = (crc >>> 8) ^ table [((v >>> 56) ^ crc) & 0xFF];  
    crc = ~ crc;
    return crc;
}
{% endhighlight %}

This is exactly the code of `crc32 (long)` in `TableCRC32`. It starts at `crc = -1` and applies the function

{% highlight Java %}
     public static int update (byte b, int crc)
     {
         return (crc >>> 8) ^ crc_table[(crc ^ b) & 0xff];
     }

{% endhighlight %}

to all eight bytes of `v`.

What we have just found out if that in **Java 8** the method `CRC32.update (int)` is intrinsic. It is defined as `native` but is not compiled as a native call. Instead,
its code is placed directly at the call site. All the usual optimisations are performed after such substitution. For instance, in our case the constant propagation has happened --
the first `crc >>> 8` was replaced with `0xFFFFFF`. Instruction re-ordering has also been performed -- extraction of bytes from `v` happens before the table access.

No wonder we haven't achieved any speed improvements when calling versions of `update` with arrays or byte buffers -- we eventually executed the same code, but carried the
additional costs of manipulating those arrays or buffers.

What is more surprising is that the pure **Java** version is performing faster than the one using intrinsic code (1420 ms instead of 1802). The detailed study of this phenomenon
falls outside the scope of this article, but a brief examination shows that the `Java` version is in fact compiled better.
[Here is the code](https://github.com/pzemtsov/article-life/blob/a44e2a418d046cef6f21cac4da52260962875dd4/java8.LongPoint75.hashCode.asm). It is shorter: 71 instruction instead
of 87. For instance, a single table access takes four instructions:

{% highlight c-objdump %}
  0x00007fbf51311aab: movzx  ecx,cl
  0x00007fbf51311aae: xor    eax,DWORD PTR [r8+rcx*4+0x10]
  0x00007fbf51311ab3: xor    ebp,eax
  0x00007fbf51311ab5: shr    eax,0x8
{% endhighlight %}

while in the intrinsic version it took six:

{% highlight c-objdump %}
  0x00007f9a0131ba5e: movzx  ecx,cl
  0x00007f9a0131ba61: shl    ecx,0x2
  0x00007f9a0131ba64: movsxd r8,ecx
  0x00007f9a0131ba67: xor    eax,DWORD PTR [rdx+r8*1]  ;*invokestatic update
  0x00007f9a0131ba6b: xor    ebp,eax
  0x00007f9a0131ba6d: shr    eax,0x8
{% endhighlight %}

Interestingly, the access to the array in both cases is done using the static address rather than indirectly using a reference. While in the intrinsic case thia may be
explained by the code not being written in **Java**, in **Java** case the compiler clearly made use of the fact that the
array is `static final`, so its address is known at the time of compilation. The same applies to its size. The compiler knew the array size was 256 and it was accessed
using byte-sized indices, which allowed to optimise out the index checking. This resulted in a nearly optimal code.

The full test
-------------

Let's now run the full test (a Life simulation). The entire code for the test is [here]({{ site.REPO-LIFE }}/tree/3e26e50861210b163f9e30beb89af3b6a2eb2543).
This table shows both micro-benchmark and full-test times for **Java 7** and **Java 8**. Two other hash functions added
for comparison:

<table class="numeric">
<tr><th rowspan="2">Class name</th><th rowspan="2">Comment</th><th colspan="2">Time, micro-benchmark</th><th colspan="2">Time, full test</th></tr>
<tr>                                                           <th>Java 7</th><th>Java 8</th><th><b>Java 7</b></th><th><b>Java 8</b></th></tr>
<tr></tr>
<tr><td class="label">LongPoint5</td> <td class="ttext">Multiply by one big prime                           </td><td>  547</td><td> 486</td><td> 1979</td><td>1553</td></tr>
<tr><td class="label">LongPoint6</td> <td class="ttext">Modulo big prime                                    </td><td>  578</td><td>1655</td><td> 1890</td><td>1550</td></tr>
<tr><td class="label">LongPoint7</td> <td class="ttext"><code>java.util.zip.CRC32</code>                    </td><td>10109</td><td>1882</td><td>10115</td><td>3206</td></tr>
<tr><td class="label">LongPoint71</td><td class="ttext">CRC32, one call with an array                       </td><td> 6730</td><td>2056</td><td> 7416</td><td>3515</td></tr>
<tr><td class="label">LongPoint72</td><td class="ttext">CRC32, one call with re-used array                  </td><td> 6422</td><td>1901</td><td> 7196</td><td>3235</td></tr>
<tr><td class="label">LongPoint73</td><td class="ttext">CRC32, one call with re-used array and CRC          </td><td> 6566</td><td>2009</td><td> 7771</td><td>3603</td></tr>
<tr><td class="label">LongPoint74</td><td class="ttext">CRC32, one call with a direct byte buffer           </td><td>     </td><td>1903</td><td>     </td><td>3310</td></tr>
<tr><td class="label">LongPoint75</td><td class="ttext">CRC32, written in <b>Java</b>                       </td><td> 1505</td><td>1420</td><td> 3522</td><td>2664</td></tr>
<tr><td class="label">LongPoint76</td><td class="ttext">CRC32, written in <b>C</b>, one JNI call            </td><td> 1890</td><td>1898</td><td> 3584</td><td>2915</td></tr>
<tr><td class="label">LongPoint77</td><td class="ttext">CRC32C, written in <b>C</b> using SSE, one JNI call </td><td> 1257</td><td>1214</td><td> 2409</td><td>2045</td></tr>
<tr><td class="label">LongPoint78</td><td class="ttext">JNI call to an empty function                       </td><td> 1256</td><td>1214</td><td>     </td><td>    </td></tr>
<tr><td class="label">NullPoint  </td><td class="ttext">Null function (does nothing)                        </td><td>  532</td><td> 455</td><td>     </td><td>    </td></tr>
</table>

This time the results are predictable: what is faster in the micro-benchmark, is also faster in the full test.

Conclusions
-----------

- We managed to optimise CRC32 calculation quite a bit, especially on **Java 7**

- `java.util.zip.CRC32.update` is an intrinsic on **Java 8**, so there is no point reducing the JNI call overheads

- Surprisingly, the version in pure **Java** works faster than that intrinsic. It is also faster than the version in **C** called via JNI

- The only way to perform calculations faster in **C** is by using the new CRC32 instruction available via SSE intrinsic set. One must remember that this is a different version
of CRC, although it is also well suitable as a hash function.

- With all our optimisations, the hash function based on CRC32 still works slower than one based on a single multiplication or division. While CRC may be useful for hashing
longer byte sequences, or for data distributed in some special way, it isn't the best hash function for our problem. We'll use the division-based function from now on.

Coming soon
-----------

We've paid enough attention to the hash functions. Now it's time to optimise the algorithm itself. That's what we'll be doing in the next article.
