---
layout: post
title:  "On the benefits of stream buffering in Java"
date:   2015-01-19 12:00:00
tags: Java optimisation performance streams socket buffering
REPO-BUFFER: https://github.com/pzemtsov/article-buffering-streams
---

Today's topic may seem trivial. Everybody knows what a `BufferedInputStream` is and why it is useful. Reading one
byte at a time from a file or other stream can be incredibly slow, and buffering is the solution. However, few
people know exactly how slow the "incredibly slow" is, so the effect is worth measuring. Besides, today's story
originates from real life, which means that even some production code can sometimes be improved by adding buffering.

The problem
-----------

Let's consider a simple network protocol. A client connects to a server using TCP. The server sends back a sequence of
messages. Each message consists of a type (4 bytes), length (4 bytes big-endian integer) and a body (sequence of
bytes of the specified length).

Here is the **Java** client code (the full code for both client and server is in
[repository]({{ page.REPO-BUFFER }}/commit/80cb2ae3067f8d69443e2241355402f99c1e321f)):

{% highlight Java %}
public class Client
{
    public static void main (String [] args) throws Exception
    {
        String hostName = args [0];
        Socket socket = new Socket (hostName, 22222);
        InputStream in = socket.getInputStream ();
        while (true) {
            long t0 = System.currentTimeMillis ();
            long sum = 0;
            int N = 1000000;
            for (int i = 0; i < N; i++) {
                byte [] type = readBytes (in, 4);
                int len = readInt (in);
                byte [] msg = readBytes (in, len);
                processMessage (type, msg);
                sum += msg.length + 8;
            }
            long t1 = System.currentTimeMillis ();
            long t = t1 - t0;
            System.out.printf ("Time for %d msg: %d; speed: %d msg/s; %.1f MB/s\n",
                               N, t, N * 1000L / t, sum * 0.001 / t);
        }
    }
    
    private static byte [] readBytes (InputStream in, int expectedSize) throws IOException
    {
        final byte[] buffer = new byte[expectedSize];
        int totalReadSize = 0;
        while (totalReadSize < expectedSize) {
            int readSize = in.read(buffer, totalReadSize, expectedSize - totalReadSize);
            if (readSize < 0) throw new EOFException ();
            totalReadSize += readSize;
        }
        return buffer;
    }
    
    private static final int readInt (InputStream in) throws IOException
    {
        int ch1 = in.read();
        int ch2 = in.read();
        int ch3 = in.read();
        int ch4 = in.read();
        if ((ch1 | ch2 | ch3 | ch4) < 0)
            throw new EOFException();
        return ((ch1 << 24) + (ch2 << 16) + (ch3 << 8) + (ch4 << 0));
    }
    
    private static void processMessage (byte [] type, byte [] msg)
    {
    }
}
{% endhighlight %}


The test server sends an endless stream of messages of size 100, as fast as possible.
The client receives messages and silently discards them.
The server and the client will run on two Linux servers connected to `1 Gbit` Ethernet network.

Running on `1 Gbit`
-----------------

When we run it, we get the following (I cut some arbitrary four lines from the output; in the future I'll quote
just one line, the most representative):

    Time for 1000000 msg: 3630; speed: 275482 msg/s; 29.7 MB/s
    Time for 1000000 msg: 3587; speed: 278784 msg/s; 30.1 MB/s
    Time for 1000000 msg: 3728; speed: 268240 msg/s; 29.0 MB/s
    Time for 1000000 msg: 3458; speed: 289184 msg/s; 31.2 MB/s

This looks very slow. The program isn't doing anything with data, it must be able to receive more than `30 MB/sec`.
After all, this is about `80` CPU cycles for each incoming byte. We should be able to do better than that.

One trivial line of code changes things dramatically:

{% highlight Java %}
        in = new BufferedInputStream (in);
{% endhighlight %}

(full source is [here]({{ page.REPO-BUFFER }}/commit/259397e2f89ab5cad35f2200871000abbf4e265e))

Now the results are:

    Time for 1000000 msg: 931; speed: 1074113 msg/s; 116.0 MB/s

This is almost four times faster, which is easy to explain. Previously, we called `read()` six times for each message
(assuming that the message arrived at once): once for the type, four times for the length and once for the body.
Each `read()` causes a system call, which involves a context switch. If we can avoid a system call, we must do so.

Of course, those four times for the length could be reduced to one by reading all four bytes at once:

{% highlight Java %}
    private static final int readInt (InputStream in) throws IOException
    {
        byte [] b = readBytes (in, 4);
        return (((b[0] & 0xFF) << 24) + ((b[1] & 0xFF) << 16) + ((b[2] & 0xFF) << 8) + ((b[3] & 0xFF) << 0));
    }
{% endhighlight %}

(the code is [here]({{ page.REPO-BUFFER }}/commit/729c023f17947f92c2be2cb9edb5b3e8dc6f2130)).

Results are:

    Time for 1000000 msg: 1909; speed: 523834 msg/s; 56.6 MB/s

This change made reading almost twice as fast, but the `BufferedInputStream` gave us more. It reads everything that's
available in the socket buffer (not exceeding the length of its own buffer, which is `1024` bytes by default). In the
worst case it means one read operation per message (assuming that the server also uses some kind of buffering and writes
the entire message at once). But the socket buffer may contain more than one message. What's important, it is likely
to happen if the client is too slow to read messages; by reading everything from the socket buffer it recovers,
in other words, there is a negative feedback scheme here, which is always good, as is makes things stable.

Can we read messages faster than this by, for instance, employing both tricks together (`BufferedInputStream` and reading the entire
four bytes)? It is possible, but we can't test it in this environment. The fastest result achieved so far (`116 MB/s`) is
very close to the throughput limit of the `1 Gbit` Ethernet. We'll have to look for faster network. I was lucky to find
some hosts with `10 Gbit` Ethernet interfaces around.

Running on `10 Gbit`
------------------

These are the results I got on `10 Gbit` network (note the increased iteration count):

The original test (no buffering, four reads per integer):

    Time for 10000000 msg: 31497; speed: 317490 msg/s; 34.3 MB/s

Modified `readInt`:

    Time for 10000000 msg: 16953; speed: 589866 msg/s; 63.7 MB/s

Introduced `BufferedInputStream`:

    Time for 10000000 msg: 2712; speed: 3687315 msg/s; 398.2 MB/s

[Both modifications]({{ page.REPO-BUFFER }}/commit/791e159888a567fcc13bc7f6ff4af2b021314c74):

    Time for 10000000 msg: 1885; speed: 5305039 msg/s; 572.9 MB/s

We can see that the performance gain we measured on `1 Gbit` interface (`4 times`) was misleading -- the real speedup
factor is `16`.

It's worth noticing that both methods combined achieved combined effect, which is, however, smaller than the
sum of the effects. This is normal, as, while the old `readInt` modification eliminated very expensive calls
into the kernel, the new one only eliminates calls to the `BufferedInputStream`, which are cheap. Still,
removing these four calls improved speed, even despite one extra memory allocation.

Removing memory allocation
--------------------------

Allocating a fresh object each time it is needed is a common practice in **Java**. It is tempting to remove
these allocation and rather re-use the objects allocated once. Memory allocation is cheap in **Java**, but more
memory means more frequent garbage collection, and this can be expensive. It is easy to re-use the four-byte
array in `readInt`. To remove other memory allocations, we must modify the interface to the message consumer.
It will now receive a byte array and a length:

{% highlight Java %}
    private static void processMessage (byte [] type, byte [] msg, int len)
{% endhighlight %}

Note that this isn't just a change in method signature: it is also a change in its contract. Previously the
method was free to do whatever it wanted with both byte arrays. For instance, it could put them into some container,
such as a queue. Now it is prohibited -- the method must consume the arrays right there and return. Unfortunately,
such restrictions can't be expressed by **Java** syntax. They can only be written down in the **JavaDoc**
for human readers, and this is unreliable -- who reads **JavaDoc**, anyway?

In short, such optimisation isn't recommended in general case. However, there are cases when it is appropriate (provided that
it actually makes things faster). Often the message consumer is under the direct control of the developer of the socket reader,
and all it does with the message is further decoding. We'll assume this case in the following discussion.

The modified code is [in repository]({{ page.REPO-BUFFER }}/commit/8f60f92fdd9b02263acdcaa6436ae7e6546cea69).
It allocates arrays for types and  messages, as well as the temporary array for
`readInt()`. This is what the latter looks like:

{% highlight Java %}
    private static byte [] tmpBuf = new byte[4];
    
    private static final int readInt (InputStream in) throws IOException
    {
        byte[] b = tmpBuf;
        readBytes (in, b, 4);
        return (((b[0] & 0xFF) << 24) + ((b[1] & 0xFF) << 16) + ((b[2] & 0xFF) << 8) + ((b[3] & 0xFF) << 0));
    }
{% endhighlight %}

And this is the speed:

    Time for 10000000 msg: 2130; speed: 4694835 msg/s; 507.0 MB/s

This is a failure -- the code runs slower. This seems counter-intuitive and requires explanation.
I don't have definite answer, only a theory. The problem may be in array index (and possibly `null`) checks.
The old code allocated array of size `4` and immediately accessed its elements at constant indices. The compiler
new that the array wasn't `null` and that all the indices were within range. When the array reference is read from a static field,
the compiler doesn't know that; it has to generate `null` and index check code. The indirect argument in favour
of this theory is that if we replace the entire `return` statement in `readInt` with

{% highlight Java %}
        return 100;
{% endhighlight %}

the speed immediately becomes `602 MB/s`.

In short, memory allocation isn't always bad, and its removing does not always improve speed. Strange, but true.


Using channels and buffers
--------------------------

We have already improved performance by `16` times and could stop here. This, however, would go against the
general policy of these articles, which is to optimise everything to extreme. It seems that nothing more
can be done within the `InputStream` framework, so we'll have to try the `nio` approach, using `ByteBuffer`s
and `Channel`s.

The primary reason we expect some speedup there is that there is support for direct buffers by the `JNI`,
so reading data from the socket channels into direct byte buffers requires much less data copying than when
working with streams and byte arrays. Some sophisticated implementations can even arrange `DMA` transfer straight from the card
directly into the `ByteBuffer` -- although we don't expect this in our case.

There is also another reason: we can read `int` and other values directly from `ByteBuffer` without any need for bit shifting.
The `getInt()`, `getLong()` and other methods of `DirectByteBuffer` are compiled straight into the word-moving Intel
instructions; a `sun.misc.Unsafe` magic takes care of that. Direct byte buffers are highly recommended for everyone
who decodes messages containing `int`, `long` and other primitive values in their natural form.

Unfortunately, there isn't any analog of a `BufferedInputStream` for byte channels; we'll have to create one.
[This is our first buffer-based version]({{ page.REPO-BUFFER }}/commit/b2698ce063edcf9da148d964ad90ddd65eb10862):

{% highlight Java %}
public class Client3
{
    private static ByteBuffer buf = ByteBuffer.allocateDirect (1024*512);
    
    public static void main (String [] args) throws Exception
    {
        String hostName = args [0];
        final SocketAddress socketAddr = new InetSocketAddress (hostName, 22222);
        SocketChannel chan = SocketChannel.open ();
        chan.connect (socketAddr);

        byte[] type = new byte [4];
        byte[] msg = new byte [1024];
        buf.limit (0);
        
        while (true) {
            long t0 = System.currentTimeMillis ();
            long sum = 0;
            int N = 10000000;
            for (int i = 0; i < N; i++) {
                ensure (4, chan);
                buf.get (type);
                ensure (4, chan);
                int len = buf.getInt ();
                ensure (len, chan);
                buf.get (msg, 0, len);
                processMessage (type, msg, len);
                sum += len + 8;
            }
            long t1 = System.currentTimeMillis ();
            long t = t1 - t0;
            System.out.printf ("Time for %d msg: %d; speed: %d msg/s; %.1f MB/s\n",
                               N, t, N * 1000L / t, sum * 0.001 / t);
        }
    }

    private static void ensure (int len, ByteChannel chan) throws IOException
    {
        if (buf.position() > buf.capacity () - len) {
            buf.compact ();
            buf.flip ();
        }
        while (buf.remaining () < len) {
            int oldpos = buf.position ();
            buf.position (buf.limit ());
            buf.limit (buf.capacity ());
            chan.read (buf);
            buf.limit (buf.position ());
            buf.position (oldpos);
        }
    }
}
{% endhighlight %}

The `ensure()` method looks like (and is) a spaghetti of buffer manipulation calls. It requires some explanation.
What we do is allocate a big buffer (half a megabyte in our case) and read some portion of data from the socket channel into it.
Then we consume that portion until what's left there is shorter than the next value we need. In such a case we read from
the socket channel again, appending to the remaining data. This way the current position in the buffer keep moving forward,
and finally it reaches the buffer's end. Then we copy the remaining data to the beginning of the buffer (`buf.compact()` takes care
of that) and start all over. The `limit()` and `position()` manipulations toggle the buffer between the "read from socket"
mode (`position` is where the data from the socket are written to; `limit` is the end of the available space, which is the same
as `capacity`) and "read from buffer" mode (data between `position` and `limit` has been read from the socket and is available for
the application to consume).

Note that we are still using the no-allocation scheme for the messages, but not suffering from the array index checks this time
(`int` values are read straight from the buffer).

This is the speed:

    Time for 10000000 msg: 923; speed: 10834236 msg/s; 1170.1 MB/s

This is a significant improvement over the stream-based version (almost twice the speed), which suggests that the channels and buffers were
added to **Java** for a reason. We are now transferring ten million messages per second and use about `2.5` CPU cycles
per byte - what can be better than that?

Running on localhost adapter
----------------------------

Can we do even better? Looking at the numbers, we see that we're back in familiar situation: we've saturated the network adapter.
Our `10 Gbit` card can hardly transmit more than `1170 MB/s`, and I don't have anything faster available.
The only faster option is a local (within the host) network. This is, by the way, still a valid use case --
it is not uncommon for a client and a server to run on the same host.

Running our tests in the the local connection scenario, we get the following:

For original unbuffered code:

    Time for 10000000 msg: 32803; speed: 304850 msg/s; 32.9 MB/s

For optimised version using streams:

    Time for 10000000 msg: 2076; speed: 4816955 msg/s; 520.2 MB/s

(surprisingly a bit slower than on the Ethernet adapter).

For the new `ByteBuffer`-based code:

    Time for 10000000 msg: 706; speed: 14164305 msg/s; 1529.7 MB/s

So the real speedup from the use of buffers was higher than the value that we saw. Buffers work almost three times faster than streams, not two.

Simplifying the buffer operation
--------------------------------

In the `ensure()` code above we kept appending the new data to the remaining piece of old data until we hit the end of the buffer.
There is a simpler approach: we copy the leftover to the beginning of the buffer and start there each time we are short of data. The new `ensure()` method
looks like [this]({{ page.REPO-BUFFER }}/commit/a450d5246931a351769d2e70bbdfdb991adc80b7):

{% highlight Java %}
    private static void ensure (int len, ByteChannel chan) throws IOException
    {
        if (buf.remaining () < len) {
            buf.compact ();
            buf.flip ();
            do {
                buf.position (buf.limit ());
                buf.limit (buf.capacity ());
                chan.read (buf);
                buf.flip ();
            } while (buf.remaining () < len);
        }
    }
{% endhighlight %}

I first wrote this version for simplicity -- the code looks easier to read than the original one. I expected lower performance due to more copying of
the leftovers. To my surprise, I got the following performance data:

    Time for 10000000 msg: 625; speed: 16000000 msg/s; 1728.0 MB/s

The code is actually **faster** than our previous code! And I have an explanation for that, too (any true scientist must have some explanation for whatever result
he gets). The latest solution is more memory-compact and, therefore, cache-efficient than the previous one. Instead of moving our working area forward in the
buffer, we are re-using the same space over and over again, hitting lower (and faster) levels of cache.

This is really remarkable and rare case, when the simpler and more readable code is also faster than the more complex code. I wish it was always so in computer
programming.


Byte buffers vs Byte arrays
----------------------

In our latest examples the interface to the message consumer is providing the message in a byte array. The message reader copies bytes from the byte buffer,
just to pass the array to the consumer. This is a reasonable approach if we expect that the message consumer stores the array somewhere. But in our case it
is not allowed to do so, since we give it the same array again and again. Why not give it a message in a byte buffer instead? We can do it without copying
memory, since the same memory can be pointed to by more than one byte buffer (this is what `ByteBuffer.duplicate()` method is for). Providing messages
in byte buffers is ideal for clients that don't just accept byte blob as a message, but parse it further, extracting fields. Byte buffers are well-suited
for such operations. This is what the new inner loop of our test looks like:

{% highlight Java %}
        ByteBuffer msgBuf = buf.duplicate ();
{% endhighlight %}
        
{% highlight Java %}
                ensure (len, chan);
                msgBuf.limit (buf.position () + len);
                msgBuf.position (buf.position ());
                buf.position (buf.position () + len);
                processMessage (type, msgBuf);
{% endhighlight %}

(the full code is [here]({{ page.REPO-BUFFER }}/commit/ece24bd35a5166de53fb83463aa9c191d1dd8585)).

This is the time we get when we run it on localhost:

    Time for 10000000 msg: 503; speed: 19880715 msg/s; 2147.1 MB/s

We process more than two gigabytes and almost twenty million messages per second. This is remarkable (remember, we started at `30Mb/sec` and `278K` messages).

Note 1
------

This performance was achieved when the server was sending data continuously in big blocks. In real life it may not be the case. Then the benefit from buffered operation won't
be so high.

Note 2
------

If processing of each message is time-consuming, at some point it becomes irrelevant how fast the message receiving loop works. It must just be fast enough.
The solution using `ByteArrayInputStream` can be just right for the purpose (although I still prefer using byte buffers due to easier operation).

Conclusions
-----------

- We managed to speed things up by 70 times, which is quite remarkable.

- The biggest enemy of performance in Linux is a system call. Anything that can eliminate some of them helps improve speed.

- `BufferedInputStream` can help a lot here.

- **Java** channels and byte buffers can help a lot as well.

- Byte buffers are also very efficient when you need to extract primitive data values from a byte stream.

- Another way byte buffers can help you is by eliminating unnecessary memory copying. However, improvements
  like this don't help until the extra system calls are removed.

- Eliminating allocation of new objects may be a good idea in general, but often costs are higher than benefits. Besides, the costs can be incurred in the least expected
places, so no statement about benefits of allocating or not allocating memory must be made without testing.

- The byte buffers and `BufferedInputStream` class are **Java**-specific. However, the issue of avoiding system calls is not. Whatever language you are writing in,
  proper arranging of your I/O (which includes buffering of network streams) is essential.

Coming soon
-----------

While optimising the network client, we came across the issue of dynamic memory allocation. It seems that there is
no definite answer if the memory allocation is good or bad, so the issue requires deeper investigation.
