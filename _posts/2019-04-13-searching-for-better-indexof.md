---
layout: post
title:  "Searching for a better indexOf"
date:   2019-04-13 12:00:00
tags: Java optimisation search
---

Today we'll be searching for strings inside other strings. We all know that this topic
has been studied comprehensively, and virtually all possible algorithms have already been developed
and published. We'll deliberately not look there. Our objective is not to google the best algorithm
but to see what ideas naturally come to a programmer, implement them in **Java**, and measure performance.

The journey in front of us is rather long. We're going to look at twenty-two different solutions.
For the impatient ones and the scared ones I suggest jumping straight
to the [`LastByteMatcher`](#LastByteMatcher), [`SuffixMatcher`](#SuffixMatcher)
and [`LightestChainSuffixMatcher`](#LightestChainSuffixMatcher).
The summary of all results is [here](#Summary). The other results of 
interest are [Big random patterns](#BigRandom) and
[Big random patterns, huge data](#HugeRandom).

The problem
-----------

In my **Java** coding practice I once had to search for certain byte sequences inside bigger byte arrays. If these were character strings, I would have used
`String.indexOf` straight away, and there would have been no topic for this study. I had to implement
`indexOf` for byte arrays, and, as far as I know, there is no library function for that.

There are three factors that affect search algorithms.

The first one is the length of a pattern. There are fast ways to search for a single byte,
and there are (as we'll see later) fast ways to search for a megabyte-long pattern. Different algorithms
prefer different pattern lengths. The patterns I had to search were 10 -- 20 bytes long.

The second one is the nature of data. On one end of the spectrum lies searching for random byte
sequences inside other random byte sequences. On the other end we have regular repeating patterns, such as
searching for `aaaabaa` in `aaaaaacaaaaaabaaaa`. Somewhere in between lies the case where there are some patterns
of much less horrible structure, such as in natural language text. In my case, the pattern was such a text, while
the array contained both natural text and binary data. Natural text search is a good enough approximation of this problem.

The third factor is possibility of pre-processing the data. This makes three types of search problems:

1) searching for the same pattern in multiple strings (pattern pre-compilation)

2) searching for multiple patterns in the same string (indexing)

3) searching for a single ad-hoc pattern in a single ad-hoc string (both indexing and pre-compilation
can work if they are fast, but pre-compilation seems to have better chance).

We'll concentrate on the first class, although the simplest solutions cover all the classes.

Conventions
-----------

Here are some naming and formatting conventions used in this article:

- The string to search in we'll call **the text**;
- The string to search for we'll call **the pattern**;
- Our algorithms will compare the pattern with various portions of the text; the substring we're
currently busy comparing with we'll call **the current portion**;
- The terms **prefix** and **suffix** will have their usual meaning: a possibly empty substring at the beginning or
at the end of a given string; the string itself is a prefix and a suffix for itself;
- Prefixes and suffixes that are shorter than the string are called **proper prefixes** and
**proper suffixes**;
- The longest suffix of the current portion than matches the text will be called **the good suffix**;
- The first byte that follows the good suffix to the left, if exists, will be called **the bad byte**
(**the bad character** if describing the character search).

The snapshots of search algorithms running will be shown in the following format:

{% highlight text %}
We have almost found our pattern in the text
                        pattern
{% endhighlight %}

Here the top line contains the current portion and some of its neighbourhood, while the second one
shows our pattern, placed right under the current portion. We'll avoid patterns starting or ending
with spaces for these demonstrations.

The test
--------

As a text we need something with a non-trivial distribution
of characters, repeating patterns and good chances to find short strings in more than one place.
Some piece of poetry will do, so we'll use _The Tragedy of Hamlet, Prince of Denmark_
(file [`text`]({{ site.REPO-INDEXOF }}/blob/master/text) in the root directory).
We'll convert it into lowercase, remove punctuation
and replace line breaks with spaces. Our text is 150128 characters long.
We'll use one arbitrarily chosen verse as the base of our patterns:

{% highlight text %}
doubt thou the stars are fire
doubt that the sun doth move
doubt truth to be a liar
but never doubt i love
{% endhighlight %}

We'll use substrings of this pattern base of various lengths as our patterns.

Since there are very good ways to search for short patterns (such as one or two bytes),
and complex algorithms are unlikely to perform well there, we'll limit the study to substrings of length
of at least four.

Our pattern base is 106 bytes long. To save space, we'll only publish the exponential results (for patterns
of lengths of 4, 8, 16, 32, 64 and 106). For each of those we'll use all possible substrings of this length and
average the results. This makes the full 106-long pattern special, so the results may be a bit odd for that one.
To get averaged results for long patterns, we'll add length 96 as well.

The test will count how many times each pattern is encountered in the entire text (which must be
at least one).
We'll measure the time spent, in nanoseconds, per byte of the text. Since the text is short and is iterated more than
once, the caching effects are ignored for now. We will, however, try very long data sets later.

Our matchers will extend the abstract class
[`Matcher`]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/Matcher.java)
with the following method:

{% highlight Java %}
public abstract int indexOf (byte [] text, int fromIdx);
{% endhighlight %}

Each class will be accompanied by a corresponding factory class -- some implementations will use different classes
for different patterns. The classes will contain debug code, which collects appropriate statistics; I'll omit this
code in included code snippets.

The main class of the test is [`byteIndexof.Indexof`]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/Indexof.java).

We'll run our tests on **Java** 1.8.142 on Xeon&reg; CPU E5-2620 v3 @ 2.40GHz, using Linux. The full source code
is available in the [repository]({{ site.REPO-INDEXOF }}). Now we're ready to go.

A Simple Matcher
----------------

Repository reference: [SimpleMatcher.java]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/SimpleMatcher.java).

This is the simplest of all implementations. We'll use it as the reference point: everything else will be tested against this.

{% highlight Java %}
public int indexOf (byte[] text, int fromIdx)
{
    int pattern_len = pattern.length;
    int text_len = text.length;
    for (int i = fromIdx; i + pattern_len <= text_len; i++) {
        if (compare (text, i, pattern, pattern_len)) return i;
    }
    return -1;
}
{% endhighlight %}

where `compare()` is defined in the base `Matcher` class:

{% highlight Java %}
public static boolean compare (byte [] text, int offset,
                               byte [] pattern, int pattern_len)
{
    for (int i = 0; i < pattern_len; i++) {
        if (text [offset + i] != pattern [i]) return false;
    }
    return true;
}
{% endhighlight %}

Here are the times for our exponential set (nanoseconds per byte of the text):

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">Simple</td><td>  2.90</td><td>  2.89</td><td>  3.30</td><td>  3.28</td><td>  3.29</td><td>  3.21</td><td>  2.68</td></tr>
</table>

We promised some statistics. For this matcher, we'll calculate the average inequality index, that is, the average
value of `i` at the exit of `compare()`:

<table class="numeric">
<tr><th> Value </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">Inequality index </td><td> 0.10 </td><td> 0.10 </td><td> 0.10 </td><td> 0.10 </td><td> 0.10 </td><td> 0.10 </td><td> 0.04 </td></tr>
</table>

The reason this index is so much smaller in the case of length 106 is that in this case the first
character of the pattern is fixed (`d`), and this letter occurs rather infrequently (3.22%),
while smaller pattern lengths involve variety of first characters.

This index is bigger than the value of 1/26 = 0.038, which is the mean predicted value for the
alphabet of 27 characters. The reason is non-uniform distribution of characters, both in the text
and in the pattern.

First Byte Matcher
------------------

Repository reference: [FirstByteMatcher.java]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/FirstByteMatcher.java).

Since 90% of compare failures happen at the first byte, it looks attractive to treat this byte specially:

{% highlight Java %}
public int indexOf (byte [] text, int fromIdx)
{
    int pattern_len = pattern.length;
    int text_len = text.length;
    byte a = pattern [0];

    for (int i = fromIdx; i <= text_len - pattern_len; i++) {
        if (buf [i] == a && compare (buf, i + 1, pattern, 1, pattern_len - 1))
            return i;
    }
    return -1;
}
{% endhighlight %}

This is indeed faster than the simple matcher:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">Simple</td><td>  2.90</td><td>  2.89</td><td>  3.30</td><td>  3.28</td><td>  3.29</td><td>  3.21</td><td>  2.68</td></tr>
<tr><td class="ttext">FirstByte</td><td>  1.47</td><td>  1.51</td><td>  1.52</td><td>  1.52</td><td>  1.49</td><td>  1.46</td><td>  0.92</td></tr>
</table>

We even broke the one-nanosecond barrier, but we already know why this happened: the first character
of the pattern is a rare letter `d`.

The statistic we collect this time is the rate `compare()` is invoked, or, in other words,
the frequency of the first byte not matching:

<table class="numeric">
<tr><th> Value </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">Compare() rate </td><td> 8.4% </td><td> 8.2% </td><td> 8.4% </td><td> 8.3% </td><td> 8.2% </td><td> 7.6% </td><td> 3.2% </td></tr>
</table>

First Bytes Matcher
-------------------

Repository reference: [FirstBytesMatcher.java]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/FirstBytesMatcher.java).

The success of special treatment of the first byte makes us think of extending this approach to
multiple first bytes. This would definitely help if we wrote in **C**, where 
reading (at least, on Intel architecture) a multi-byte word is as cheap as reading one byte.
In **Java**, we aren't so sure, so it needs testing. We'll use the first four bytes:

{% highlight Java %}
public int indexOf (byte[] text, int fromIdx)
{
    for (int i = fromIdx; i <= text.length - pattern.length; i++) {
        if (text[i] == a && text[i+1] == b && text[i+2] == c && text[i+3] == d
            && compare (text, i + 4, pattern, 4, pattern.length - 4))
        {
            return i;
        }
    }
    return -1;
}
{% endhighlight %}

Here are the results:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">FirstByte </td><td>  1.47</td><td>  1.51</td><td>  1.52</td><td>  1.52</td><td>  1.49</td><td>  1.46</td><td>  0.92</td></tr>
<tr><td class="ttext">FirstBytes</td><td>  1.18</td><td>  1.35</td><td>  1.36</td><td>  1.37</td><td>  1.38</td><td>  1.32</td><td>  0.86</td></tr>
</table>

There is some improvement, although it isn't very big. We didn't expect much anyway,
because the change only addresses 8% of all cases.

We collect the `compare()` rate again, and it is much lower than before:

<table class="numeric">
<tr><th> Value </th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">compare() rate </td><td>0.09%</td><td>0.09%</td><td>0.10%</td><td>0.15%</td><td>0.16%</td><td>0.01%</td></tr>
</table>

First Bytes Matcher improved
----------------------------

Repository reference: [FirstBytesMatcher2.java]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/FirstBytesMatcher2.java).

We can save a bit on reading `buf [i+1]`, `buf [i+2]` and `buf [i+3]` in the following way:

{% highlight Java %}
private int indexOf (byte[] text, int fromIdx, byte [] pattern)
{
    int text_len = text.length;
    int pattern_len = pattern.length;
    if (text_len < pattern_len) {
        return -1;
    }
    byte a = pattern [0];
    byte b = pattern [1];
    byte c = pattern [2];
    byte d = pattern [3];
    byte A = text [fromIdx];
    byte B = text [fromIdx+1];
    byte C = text [fromIdx+2];

    for (int i = fromIdx; i <= text_len - pattern_len; i++) {
        byte D = text [i+3];
        if (A == a && B == b && C == c && D == d &&
            compare (text, i+4, pattern, 4, pattern_len - 4))
        {
            return i;
        }
        A = B;
        B = C;
        C = D;
    }
    return -1;
}
{% endhighlight %}

Generally, I don't like such manual improvements, because I believe that a good compiler should be
clever enough to perform them for us. However, it did help a bit:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">FirstBytes</td><td>  1.18</td><td>  1.35</td><td>  1.36</td><td>  1.37</td><td>  1.38</td><td>  1.32</td><td>  0.86</td></tr>
<tr><td class="ttext">FirstBytes2</td><td>  1.21</td><td>  1.18</td><td>  1.19</td><td>  1.19</td><td>  1.20</td><td>  1.16</td><td>  0.83</td></tr>
</table>

Compiled matcher
----------------

Repository reference: [CompileMatcher.java]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/CompileMatcher.java).

Let's try a different approach and make use of the fact that we're writing in **Java**.
What if we create a new class for every pattern we search for, compile it and load? All of that must
happen in the factory class. It takes time, but this time isn't affecting our test results. There are three
ways to generate a class:

- create the byte code directly;
- generate **Java** text and invoke the compiler from `tools.jar`; that will require knowing the
path to that jar;
- generate **Java** text and invoke **javac**, assuming it's in the `${PATH}`.

We'll go the last route as it's the simplest one; the generator is in the
[repository]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/CompileMatcher.java).  Here
is an example of the code for some small pattern length (pattern `doubt th`):

{% highlight Java %}
    public int indexOf (byte[] buf, int fromIdx)
    {
        for (int i = fromIdx; i < buf.length - 7; i++) {
            if (buf [i+0] != 100) continue;
            if (buf [i+1] != 111) continue;
            if (buf [i+2] != 117) continue;
            if (buf [i+3] != 98) continue;
            if (buf [i+4] != 116) continue;
            if (buf [i+5] != 32) continue;
            if (buf [i+6] != 116) continue;
            if (buf [i+7] != 104) continue;
            return i;
        }
        return -1;
    }
{% endhighlight %}

Results are, however, not better than before, so we can discard this method:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">FirstBytes2</td><td>  1.21</td><td>  1.18</td><td>  1.19</td><td>  1.19</td><td>  1.20</td><td>  1.16</td><td>  0.83</td></tr>
<tr><td class="ttext">Compile</td><td>  1.18</td><td>  1.38</td><td>  1.38</td><td>  1.39</td><td>  1.34</td><td>  1.27</td><td>  0.80</td></tr>
</table>

This is actually quite strange. Perhaps, it's a topic of a new study.

Hash matcher
------------

Repository reference: [HashMatcher.java]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/HashMatcher.java).

For the first time we'll try not to improve little things here and there, but rather use some
sort of non-trivial algorithm. We'll start with a hash matcher.

The idea is to apply some function to both the pattern and the current portion to obtain
their hash value. This function must be additive, i.e. allow for quick addition and removal
of bytes, so that we could slide the current portion along the text.
We'll only perform full comparison when the hash values match. To make things simple, we'll use addition as a hash function:

{% highlight Java %}
private static int hashCode (byte[] array, int fromIdx, int length)
{
    int hash = 0;
    for (int i = fromIdx; i < fromIdx + length; i++)
        hash += array[i];
    return hash;
}

public int indexOf (byte[] text, int fromIdx)
{
    byte [] pattern = this.pattern;
    int pattern_len = pattern.length;
    int text_len = text.length;

    if (fromIdx + pattern_len > text_len) return -1;
    int pattern_hash = this.hash;
    
    int hash = hashCode (text, fromIdx, pattern_len);

    for (int i = fromIdx; ; i++) {
        if (hash == pattern_hash && compare (text, i, pattern, pattern_len))
            return i;
        if (i + pattern_len >= text_len) return -1;
        hash -= text [i];
        hash += text [i + pattern_len];
    }
}
{% endhighlight %}

Here are the times:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">FirstBytes2</td><td>  1.21</td><td>  1.18</td><td>  1.19</td><td>  1.19</td><td>  1.20</td><td>  1.16</td><td>  0.83</td></tr>
<tr><td class="ttext">Hash</td><td>  2.59</td><td>  2.47</td><td>  2.43</td><td>  2.42</td><td>  2.40</td><td>  2.39</td><td>  2.40</td></tr>
</table>

Rather poor performance of this algorithm is surprising. Let's look at the `compare()` rate statistics:

<table class="numeric">
<tr><th> Value </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">Compare() rate </td><td>1.6%</td><td>1.2%</td><td>1.0%</td><td>0.8%</td><td>0.7%</td><td>0.6%</td><td>0.6%</td></tr>
</table>

The compare rate isn't very low. It is  much higher than for `FirstBytesMatcher`, and some operations (two
hash updates and one comparison) happen on each iteration. Some better algorithm is required, one
that helps reduce the number of iterations.

<div id="LastByteMatcher"></div>

Last Byte Matcher
-----------------

Repository reference: [LastByteMatcher.java]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/LastByteMatcher.java).

The idea of the last byte matcher is that when our pattern does not match the current portion,
in most cases we can do better than shift the pattern by one. We can look at the last
byte of the current portion, find it in the pattern and shift accordingly. This is how it works:

Let's assume we are searching for the first line of our pattern: `doubt thou the stars are fire`,
and let's try matching it against an arbitrary portion of our text:

{% highlight text %}
to be or not to be that is the question whether tis nobler in the mind to suff
doubt thou the stars are fire
{% endhighlight %}

The strings clearly don't match, and the last character of the pattern (`e`) falls against an `h`.
Instead of shifting the pattern by 1, which would put this `h` against an `r`, we can search
our pattern right to left and find the rightmost `h` in it, which is the one in `the` (16 positions
from the end). We can then shift our pattern by 16 to put `h` under `h`:

{% highlight text %}
to be or not to be that is the question whether tis nobler in the mind to suff
                doubt thou the stars are fire
{% endhighlight %}

Completely coincidentally, the last `e` fell on `h` once more, so we can shift by 16 again:

{% highlight text %}
ot to be that is the question whether tis nobler in the mind to suffer the sli
                      doubt thou the stars are fire
{% endhighlight %}

Here we got very lucky: `e` falls on `n`, which does not occur in our pattern. We can shift by
the entire pattern length (29):

{% highlight text %}
whether tis nobler in the mind to suffer the slings and arrows of outrageous fo
                     doubt thou the stars are fire
{% endhighlight %}

Note that shift by 1 can still occur, such as in this case:

{% highlight text %}
to be or not to be that is the question whether tis nobler
                  doubt thou the stars are fire
{% endhighlight %}

What if the last byte matches, but not the rest?

{% highlight text %}
to be or not to be that is the question whether tis nobler
 doubt thou the stars are fire
{% endhighlight %}

In this case we must find the second rightmost occurrence of the last character (in this case it's
`e` in `are`) and shift there (by 5):

{% highlight text %}
to be or not to be that is the question whether tis nobler
      doubt thou the stars are fire
{% endhighlight %}

So here is the plan: we search for all possible bytes in our pattern and
record how far from the end of the pattern they occur, for the last byte in the pattern
ignoring that last occurrence. For the bytes that never occur, we record the pattern length.
Then we'll take the last byte of the current portion, look up the recorded value for that byte and
shift our pattern by this number. This is how we calculate
the shifts:

{% highlight Java %}
private final int [] shifts = new int [256];

public MatcherImpl (byte [] pattern)
{
    this.pattern = pattern;

    int len = pattern.length;
    for (int i = 0; i < 256; i++) {
        shifts [i] = len;
    }
    for (int i = 0; i < len-1; i++) {
        shifts [pattern[i] & 0xFF] = len - i - 1;
    }
}
{% endhighlight %}

And this is how we search:

{% highlight Java %}
public int indexOf (byte[] text, int fromIndex)
{
    byte [] pattern = this.pattern;
    int pattern_len = pattern.length;
    byte last = pattern [pattern_len-1];
    int [] shifts = this.shifts;
    
    for (int pos = fromIndex; pos < text.length - pattern_len + 1;) {
        byte b = text [pos + pattern_len - 1];
        if (b == last && compare (text, pos, pattern, pattern_len - 1)) {
            return pos;
        }
        pos += shifts [b & 0xFF];
    }
    return -1;
}
{% endhighlight %}

Except for very short patterns, it really runs faster than everything we tried so far:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">FirstBytes2</td><td>  1.21</td><td>  1.18</td><td>  1.19</td><td>  1.19</td><td>  1.20</td><td>  1.16</td><td>  0.83</td></tr>
<tr><td class="ttext">LastByte</td><td>  1.54</td><td>  0.90</td><td>  0.56</td><td>  0.40</td><td>  0.30</td><td>  0.26</td><td>  0.22</td></tr>
</table>

The one-nanosecond barrier has now been broken convincingly; it doesn't depend on the specific character
of the pattern.

The statistics of interest for this matcher are the average shift and the compare rate:

<table class="numeric">
<tr><th> Value </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">Avg shift      </td><td> 3.5</td><td> 6.0</td><td> 9.6</td><td>13.9</td><td>19.2</td><td>22.2</td><td>24.8</td></tr>
<tr><td class="ttext">Compare() rate </td><td> 8.9%</td><td> 9.2%</td><td> 9.1%</td><td> 9.0%</td><td> 8.9%</td><td> 7.8%</td><td>10.1%</td></tr>
</table>

The compare rate is pretty much the same as for the first byte matcher (except for the last pattern;
probably, because it ends with a very frequent letter `e`).

The average shift is quite a bit bigger than one (that's why this matcher is so fast), but it still
disappoints: sometimes it is as much as four times smaller than the pattern length. This is to be
expected, though, since our alphabet is so small. The chances for a given character to be found inside
a long pattern are quite high. If the text and the pattern were random byte sequences,
we could be much better off with this matcher (or, rather, the same problem would require longer patterns to occur),
but for our string example we need to increase the shift.

For medium-sized patterns (such as of length 16), however, the shift value is satisfactory,
the speed is good, and the
simplicity of this matcher makes it very attractive. I ended up using this one in my production
code.

Multi-byte matcher
------------------

Repository reference: [MultiByteMatcher.java]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/MultiByteMatcher.java).

One way to increase the average shift value is to work with more than one byte.
Let's look at the example we used when introducing the `LastByteMatcher`:

{% highlight text %}
to be or not to be that is the question whether tis nobler in the mind
doubt thou the stars are fire
{% endhighlight %}

We looked at the last character (`h`), and found it 16 positions from the end of the pattern,
so we could shift the pattern by this number. The next character in the current portion (`t`)
doesn't work so well, since it occurs in `stars` and only provides the shift value of 11. The next one
(space) is even worse, as it occurs just two positions to the left. The next one (`s`) is no better: it
gives the shift value of 6. Finally, `i` becomes the champion: it never occurs in the pattern
before this point, so the shift is 24. There is no point looking further -- the shift can only
get smaller.

The last byte has the potential to produce the biggest shift (the shift can never
be bigger than the character's position from the left), that's why if we can only use one byte, the
last one is the best choice. What if we use more than one? We'll start from the right-hand side
and go left while there is a chance to improve the shift.

One downside of this approach is that instead of one array of size 256 we'll have to keep a whole
collection of those -- one for each position in the pattern:

{% highlight Java %}
private final int [][] shifts;

public MatcherImpl (byte [] pattern)
{
    this.pattern = pattern;

    int len = pattern.length;
    shifts = new int [pattern.length][256];
    for (int pos = len-1; pos >= 0; pos --) {
        for (int i = 0; i < 256; i++) {
            shifts [pos][i] = pos+1;
        }
        for (int i = 0; i < pos; i++) {
            shifts [pos][pattern[i] & 0xFF] = pos - i;
        }
    }
}
{% endhighlight %}

This is what matching looks like:

{% highlight Java %}
public int indexOf (byte[] text, int fromIndex)
{
    byte [] pattern = this.pattern;
    int pattern_len = pattern.length;
    int [][] shifts = this.shifts;
    
    for (int pos = fromIndex; pos < text.length - pattern_len + 1;) {
        if (compare (text, pos, pattern, pattern_len)) {
            return pos;
        }
        
        int shift = 0;
        for (int i = pattern_len - 1; i >= 0; i--) {
            int sh = shifts [i] [text [pos + i] & 0xFF];
            if (sh > shift) shift = sh;
            if (shift > i) {
                break;
            }
        }
        pos += shift;
    }
    return -1;
}
{% endhighlight %}

We collect two statistics for this matcher: the average shift and the average number of iterations
this shift was obtained at:

<table class="numeric">
<tr><th> Value </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">Avg shift  </td><td> 3.7</td><td> 7.3</td><td>14.6</td><td>29.1</td><td>57.6</td><td>87.2</td><td>97.3</td></tr>
<tr><td class="ttext">Iterations </td><td>1.3</td><td>1.6</td><td>2.4</td><td>3.9</td><td>7.4</td><td>9.8</td><td>9.7</td></tr>
</table>

The shift looks very good (very close to the pattern length), but the iteration count is quite high,
which can undermine the entire effort. 

This seems indeed to be happening: the results aren't very good.

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">LastByte</td><td>  1.54</td><td>  0.90</td><td>  0.56</td><td>  0.40</td><td>  0.30</td><td>  0.26</td><td>  0.22</td></tr>
<tr><td class="ttext">MultiByte</td><td>  2.59</td><td>  1.90</td><td>  1.32</td><td>  0.95</td><td>  0.69</td><td>  0.54</td><td>  0.45</td></tr>
</table>

Limited Multi-byte matcher
--------------------------

Repository reference: [LimitedMultiByteMatcher.java]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/LimitedMultiByteMatcher.java).

We can limit the number of iterations in `MultiByteMatcher` to some small value (two or three). This
will reduce both the average shift and the operation cost, and the overall performance can improve.
We just copy the code for `MultiByteMatcher` and introduce the constructor parameter `n` for
the factory and the matcher.  The main loop will run to `pattern_len - n` instead of `0`:

{% highlight Java %}
for (i = pattern_len - 1; i >= pattern_len-n; i--) {
    int sh = shifts [i] [text [pos + i] & 0xFF];
    if (sh > shift) shift = sh;
    if (shift > i) {
        break;
    }
}
{% endhighlight %}

First, some statistics. Let's start with the average shift:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">LastByte      </td><td> 3.5</td><td> 6.0</td><td> 9.6</td><td>13.9</td><td>19.2</td><td>22.2</td><td>24.8</td></tr>
<tr><td class="ttext">LMultiByte (2)</td><td> 3.7</td><td> 7.1</td><td>12.7</td><td>20.4</td><td>30.3</td><td>36.6</td><td>39.8</td></tr>
<tr><td class="ttext">LMultiByte (3)</td><td> 3.7</td><td> 7.3</td><td>13.9</td><td>24.1</td><td>37.6</td><td>47.0</td><td>50.5</td></tr>
<tr><td class="ttext">LMultiByte (4)</td><td> 3.7</td><td> 7.3</td><td>14.4</td><td>26.1</td><td>42.6</td><td>54.9</td><td>59.6</td></tr>
<tr><td class="ttext">MultiByte     </td><td> 3.7</td><td> 7.3</td><td>14.6</td><td>29.1</td><td>57.6</td><td>87.2</td><td>97.3</td></tr>
</table>

As expected, the shift gradually increases with the maximal iteration count.

Here are the actual iteration counts used:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">LMultiByte (2)</td><td> 1.2</td><td> 1.4</td><td> 1.6</td><td> 1.8</td><td> 1.9</td><td> 1.9</td><td> 1.9</td></tr>
<tr><td class="ttext">LMultiByte (3)</td><td> 1.3</td><td> 1.6</td><td> 2.0</td><td> 2.4</td><td> 2.7</td><td> 2.7</td><td> 2.7</td></tr>
<tr><td class="ttext">LMultiByte (4)</td><td> 1.3</td><td> 1.6</td><td> 2.2</td><td> 2.8</td><td> 3.4</td><td> 3.5</td><td> 3.5</td></tr>
<tr><td class="ttext">MultiByte     </td><td> 1.3</td><td> 1.6</td><td> 2.4</td><td> 3.9</td><td> 7.4</td><td> 9.8</td><td> 9.7</td></tr>
</table>

Now let's see if smaller iteration counts compensate for smaller shifts:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">LastByte</td><td>  1.54</td><td>  0.90</td><td>  0.56</td><td>  0.40</td><td>  0.30</td><td>  0.26</td><td>  0.22</td></tr>
<tr><td class="ttext">LMultiByte (2)</td><td>  2.63</td><td>  1.90</td><td>  1.20</td><td>  0.71</td><td>  0.45</td><td>  0.37</td><td>  0.22</td></tr>
<tr><td class="ttext">LMultiByte (3)</td><td>  2.66</td><td>  1.96</td><td>  1.29</td><td>  0.80</td><td>  0.51</td><td>  0.41</td><td>  0.28</td></tr>
<tr><td class="ttext">LMultiByte (4)</td><td>  2.66</td><td>  1.98</td><td>  1.35</td><td>  0.86</td><td>  0.55</td><td>  0.43</td><td>  0.30</td></tr>
<tr><td class="ttext">MultiByte</td><td>  2.59</td><td>  1.90</td><td>  1.32</td><td>  0.95</td><td>  0.69</td><td>  0.54</td><td>  0.45</td></tr>
</table>

It doesn't. The limited multi-byte matcher is still slower than the ordinary last-byte matcher.

Unrolled Limited Multi-byte matcher
----------------------------------

Repository reference: [UnrolledLimitedMultiByteMatcher.java]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/UnrolledLimitedMultiByteMatcher.java).

We don't give up so easily. When the number of iterations of the limited multi-byte version
is known and small, we can unroll the loop and, possibly, increase performance.

This is what the unrolled routine looks like for the number of iterations of three:

{% highlight Java %}
public int indexOf (byte[] text, int fromIndex)
{
    byte [] pattern = this.pattern;
    int pattern_len = pattern.length;
    int [] shifts1 = shifts[len-1];
    int [] shifts2 = shifts[len-2];
    int [] shifts3 = shifts[len-3];
    byte last = pattern [len-1];
    
    for (int pos = fromIndex; pos < text.length - pattern_len + 1;) {
        byte b1 = text [pos + len-1];
        if (b1 == last && compare (text, pos, pattern, pattern_len-1)) {
            return pos;
        }
        
        int shift = shifts1 [b1 & 0xFF];
        if (shift < pattern_len) {
            byte b2 = text [pos + pattern_len-2];
            int sh = shifts2 [b2 & 0xFF];
            if (sh > shift) shift = sh;
            if (shift < pattern_len) {
                byte b3 = text [pos + pattern_len-3];
                int sh3 = shifts3 [b3 & 0xFF];
                if (sh3 > shift) shift = sh3;
            }
        }
        pos += shift;
    }
    return -1;
}
{% endhighlight %}

Here are the times:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">LastByte</td><td>  1.54</td><td>  0.90</td><td>  0.56</td><td>  0.40</td><td>  0.30</td><td>  0.26</td><td>  0.22</td></tr>
<tr><td class="ttext">UnrolledLMultiByte (2)</td><td>  2.14</td><td>  1.38</td><td>  0.79</td><td>  0.48</td><td>  0.33</td><td>  0.27</td><td>  0.16</td></tr>
<tr><td class="ttext">UnrolledLMultiByte (3)</td><td>  2.19</td><td>  1.49</td><td>  0.90</td><td>  0.54</td><td>  0.36</td><td>  0.29</td><td>  0.15</td></tr>
<tr><td class="ttext">UnrolledLMultiByte (4)</td><td>  2.31</td><td>  1.57</td><td>  0.97</td><td>  0.59</td><td>  0.38</td><td>  0.30</td><td>  0.11</td></tr>
</table>

Unrolling improves times, but not enough to outperform the `LastByteMatcher`, except for 
our pattern of 106, which is non-representative by its nature.

Search String Suffixes
----------------------

Until now we looked for occurrences of single bytes in our pattern. When more than one
byte was used, they were searched independently of each other. It may be more productive
to search for groups of bytes, e.g. for entire suffixes of the current portion.

Let's look at some example: we'll search for the second
line of our pattern base (which is 28 characters long) close to its presence in the text:

{% highlight text %}
doubt thou the stars are fire doubt that the sun doth move doubt truth to be a
    doubt that the sun doth move
{% endhighlight %}

If we record positions of single characters only (as in `LastByteMatcher`), we get the
shift value of 2, as this is how far the last `o` is found from the end of the pattern.

If we work with suffixes of length 2, we'll search for `do` and find it 7 characters from the
end, which is better but still far from perfect.

Three characters (`" do"`) take us to the same place (the shift value of 7).

Finally, four characters (`e do`) are not found in the pattern at all. We can shift by 25,
so that the next current portion starts at `" do"`:

{% highlight text %}
doubt thou the stars are fire doubt that the sun doth move doubt truth to be a
                             doubt that the sun doth move
{% endhighlight %}

We can do better than this if for all suffix lengths we record whether these suffixes occur in the
beginning of the pattern. We know the pattern does not start with `" do"`, so we can increase
the shift to 26. Since the pattern starts with `do`, we can't increase it more, and we can see
that 26 is indeed the right value -- it takes us straight to the match:

{% highlight text %}
doubt thou the stars are fire doubt that the sun doth move doubt truth to be a
                              doubt that the sun doth move
{% endhighlight %}

What if no suffix is found in the beginning of the pattern? Then we can shift by the entire
pattern length, as in the example from the `LastByteMatcher` section:

{% highlight text %}
to be or not to be that is the question whether tis nobler in the mind to suff
doubt thou the stars are fire
{% endhighlight %}

Here `h`, `th`, `" th"` give the same shift (16), and neither of them is found in the beginning.
The suffix of length 4 (`s th`) isn't found at all, which allows us to shift the pattern
by its entire length of 29.

What if more than one suffix is found in the beginning of the pattern? We must be conservative
and use the longest one, which will result in the smallest shift. This case seems impossible
at first, but it is in fact quite real:

{% highlight text %}
none my lord but that the worlds grown honest
       t that the sun d
{% endhighlight %}

Three suffixes are found in the beginning: `t`, `t t`, `t that t`, the longest one
being eight characters long. This means that if we get as far as the suffix `ut that t`,
which does not occur in the pattern, we must shift by the pattern length minus 8, which is 8:

{% highlight text %}
None my lord but that the worlds grown honest
               t that the sun d
{% endhighlight %}

In short, here are the simple rules we can follow (assuming _P_ is the pattern length):

- if some current portion suffix is found in the pattern at the smallest distance of _N_ from the right end,
use _N_ as the shift value;
- if some current portion suffix of size _L_ is not found in the pattern at all, and we have no information
on the occurrence in the beginning, use _P_ &minus; _L_ + _1_ as the shift value;
- if some current portion suffix is not found in the pattern at all, and _L_ is the length of the
longest suffix of this suffix that occurs in the beginning (zero if none), use _P_ &minus; _L_ as the shift.

Last Byte Suffix Matcher
------------------------

Repository reference: [LastByteSuffixMatcher.java]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/LastByteSuffixMatcher.java).

Let's start with a simplistic implementation of this idea, which only deals with the suffixes
of the pattern.

Let's go back to the `LastByteMatcher`. There we used the last byte of the current portion
to calculate the shift, no matter if it matched the last byte of the pattern or not. Let's modify
this a bit. If the last byte doesn't match, we do exactly as before.
If it does match, we look how many more bytes match (find the **good suffix**), and apply the
entire procedure described above to that suffix. The good part of it is that the suffix is also a
suffix of the pattern and is therefore entirely defined by its length. We can pre-calculate whatever
we want for these suffixes. Here is what we do to initialise the matcher:

{% highlight Java %}
private final byte [] pattern;
private final int [] shifts;
private final int [] suffix_shifts; // index is the suffix length

private int find (byte [] pattern, int suffix_len)
{
    int len = pattern.length;
    
    for (int pos = len - suffix_len - 1; pos >= 0; pos --) {
        if (compare (pattern, pos, pattern, len - suffix_len, suffix_len)) {
            return pos;
        }
    }
    return -1;
}

public MatcherImpl (byte [] pattern)
{
    this.pattern = pattern;

    int len = pattern.length;
    shifts = new int [256];
    for (int i = 0; i < 256; i++) {
        shifts [i] = len;
    }
    for (int i = 0; i < len-1; i++) {
        shifts [pattern[i] & 0xFF] = len - i - 1;
    }
    
    suffix_shifts = new int [pattern.length];
    suffix_shifts [0] = shifts [pattern [len-1] & 0xFF];
    int atzero_len = 0;
    
    for (int suffix_len = 1; suffix_len < pattern.length; suffix_len ++) {
        int pos = find (pattern, suffix_len);
        int suffix_shift;
        
        if (pos < 0) {
            suffix_shift = len - atzero_len;
        } else {
            suffix_shift = len - pos - suffix_len;
        }
        suffix_shifts [suffix_len] = suffix_shift;

        if (compare (pattern, len - suffix_len, pattern, 0, suffix_len)) {
            atzero_len = suffix_len;
        }
    }
}
{% endhighlight %}

And here is the matcher:

{% highlight Java %}
public int indexOf (byte[] text, int fromIndex)
{
    byte [] pattern = this.pattern;
    int len = pattern.length;
    byte last = pattern [len-1];
    int [] shifts = this.shifts;
    
    for (int pos = fromIndex; pos <= text.length - len;) {
        byte b = text [pos + len - 1];
        int shift;
        if (b != last) {
            shift = shifts [b & 0xFF];
        } else {
            int i = len-2;
            while (true) {
                if (text [pos + i] != pattern [i]) {
                    break;
                }
                if (i == 0) {
                    return pos;
                }
                -- i;
            }
            int suffix_len = len - i - 1;
            shift = suffix_shifts [suffix_len];
        }
        pos += shift;
    }
    return -1;
}
{% endhighlight %}

Here is an example of operation of this matcher:

{% highlight text %}
had he been put on to have proved most royally
                ver doubt i love
{% endhighlight %}

Here the pattern length is 16 and the good suffix is of size 3 (`ove`), and it never occurs in the pattern.
However, it doesn't mean we can shift by 16: a shorter suffix (`ve`) matches the beginning of
the pattern, so the correct shift is 14:

{% highlight text %}
had he been put on to have proved most royally
                              ver doubt i love
{% endhighlight %}

The times, however, aren't great:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">LastByte</td><td>  1.54</td><td>  0.90</td><td>  0.56</td><td>  0.40</td><td>  0.30</td><td>  0.26</td><td>  0.22</td></tr>
<tr><td class="ttext">LastByteSuffix</td><td>  1.56</td><td>  0.91</td><td>  0.57</td><td>  0.40</td><td>  0.30</td><td>  0.25</td><td>  0.20</td></tr>
</table>

It is easy to see why the speed hasn't improved. Remember, in the `LastByteMatcher` the compare rate
was 9 -- 10%. This is the fraction of invocations when the last byte didn't match. And this is the
only case when this new matcher tries to make a difference. Even if we made very high shift values
in these 10% of cases, we wouldn't have made a big difference, but we didn't do even that: when the
good suffix length is exactly one, we're back at the `LastByteMatcher` territory, where shift
values aren't very big, as any single character is quite likely to be found in a long enough pattern.

To check this, let's look at statistics: the average shift counts in general, the average shift counts
for the suffixes of length one, and the average shift counts for the suffixes of length more than one
(together with the same statistic from the `LastByteMatcher`):

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>

<tr><td class="ttext">Avg shift, LastByte </td><td> 3.5</td><td> 6.0</td><td>  9.6</td><td> 13.9</td><td> 19.2</td><td> 22.2</td><td> 24.8</td></tr>
<tr><td class="ttext">Avg shift           </td><td> 3.5</td><td> 6.1</td><td>  9.7</td><td> 14.2</td><td> 19.6</td><td> 22.7</td><td> 24.9</td></tr>
<tr><td class="ttext">Avg shift, size = 1 </td><td> 3.8</td><td> 5.7</td><td>  7.9</td><td>  9.7</td><td> 10.3</td><td> 10.7</td><td> 14.0</td></tr>
<tr><td class="ttext">Avg shift, size > 1 </td><td> 3.9</td><td> 7.5</td><td> 13.7</td><td> 22.9</td><td> 37.2</td><td> 43.4</td><td> 29.9</td></tr>
</table>

We see that increasing the suffix size does increase the shift, but not enough; besides, the situation
when it happens is rather rare.

Next Byte Suffix Matcher
------------------------

Repository reference: [NextByteSuffixMatcher.java]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/NextByteSuffixMatcher.java).

There are ways to improve these solutions. Let's look at one of them.

We can combine the `LastByteSuffixMatcher` with the `MultiByteMatcher`. For that, we'll look
at both the good suffix (just like in the previous case), and several of
the bytes that follow, choosing the bigger one of two shifts. The simplest is to look at exactly
one such byte -- the bad byte. For that, we need a full collection of 256-element shift arrays, one for each
position where that bad byte occurs.

I'll skip the initialisation code. The matcher code looks like this:

{% highlight Java %}
public int indexOf (byte[] text, int fromIndex)
{
    byte [] pattern = this.pattern;
    int len = pattern.length;
    byte last = pattern [len-1];
    int [][] shifts = this.shifts;
    int [] last_shifts = shifts [len-1];
    
    for (int pos = fromIndex; pos <= text.length - len;) {
        byte b = text [pos + len - 1];
        int shift;
        if (b != last) {
            shift = last_shifts [b & 0xFF];
        } else {
            int i = len-2;
            while (true) {
                b = text [pos + i];
                if (b != pattern [i]) {
                    break;
                }
                if (i == 0) {
                    return pos;
                }
                -- i;
            }
            int suffix_len = len - i - 1;
            shift = Math.max (shifts [i][b & 0xFF],
                              suffix_shifts [suffix_len]);
        }
        pos += shift;
    }
    return -1;
}
{% endhighlight %}

The shift count improved a bit:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">Avg shift, LastByteSuffix </td><td> 3.5</td><td> 6.0</td><td>  9.6</td><td> 13.9</td><td> 19.2</td><td> 22.2</td><td> 24.8</td></tr>
<tr><td class="ttext">Avg shift, NextByteSuffix </td><td> 3.5</td><td> 6.1</td><td> 10.0</td><td> 14.8</td><td> 20.9</td><td> 24.3</td><td> 27.2</td></tr>
</table>

This wasn't, however, enough to achieve better performance. It stayed the same or even dropped a bit:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">LastByteSuffix</td><td>  1.56</td><td>  0.91</td><td>  0.57</td><td>  0.40</td><td>  0.30</td><td>  0.25</td><td>  0.20</td></tr>
<tr><td class="ttext">NextByteSuffix</td><td>  1.70</td><td>  0.98</td><td>  0.60</td><td>  0.42</td><td>  0.31</td><td>  0.26</td><td>  0.21</td></tr>
</table>

Possible improvement: Next Suffix Matcher
-----------------------------------------

Another possible way to improve this solution is to extend the set of the indexed suffixes.
Currently, we only work with the direct suffixes of the pattern.
We can extend it to work with the strings like "a suffix of the pattern plus one arbitrary character".
This will require allocating an array of size 256 for every position in the pattern (just like what
we did in the `MultiByteMatcher`). The shifts are quite likely to improve a lot. The overall big
performance improvement is, however, highly unlikely, because these big shifts will only be produced in 10% of
all the cases -- when the last bytes match. The shifts in 90% of the cases will be exactly the
same as for `LastByteMatcher`. That's why we'll skip this solution.

If we want to get good shifts, we must really
go wild and index  more suffixes. What if we index everything?

<div id="SuffixMatcher"></div>

Suffix Matcher
--------------

Repository reference: [SuffixMatcher.java]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/SuffixMatcher.java).

The procedure of searching suffixes described above was rather complex; this was due to limitations
on the kind of strings we indexed, both in their length and their nature.

If we can afford to index suffixes of any length, things suddenly become much simpler. There is no such thing
anymore as "a suffix matches some string in the middle of the pattern". If it does, expand it by one
byte. Eventually, we'll either find a mismatch, or reach the beginning of the pattern.

The description of our algorithm becomes very easy and straightforward:

- let _P_ be the length of the pattern
- let _N_ be the length of the longest suffix of the current portion that is also a prefix of the
pattern (zero if none)
- if _P_ = _N_, we've found a match
- otherwise, shift the current portion right by _P_ &minus; _N_.

If we look at the previously considered example:

{% highlight text %}
None my lord but that the worlds grown honest
       t that the sun d
{% endhighlight %}

There are three suffixes of the current portion that are also prefixes of the pattern:
the strings `t`, `t t`, `t that t` of lengths 1, 3 and 8. Since the pattern length is
16, we must shift by 8:

{% highlight text %}
None my lord but that the worlds grown honest
               t that the sun d
{% endhighlight %}

We don't need to index all the substrings of our pattern; we only need the prefixes, and
there are only _P_ of them. We could use a hash table for that, but I feel that a search tree
is more appropriate. This is what we do:

{% highlight text %}
private static final class Node
{
    Node [] nodes = null;
    boolean boundary = false;
}

private final Node root;

public MatcherImpl (byte [] pattern)
{
    int pattern_len = pattern.length;
    root = new Node ();

    for (int prefix_len = 1; prefix_len <= pattern_len; prefix_len++) {
        Node node = root;
        for (int i = prefix_len - 1; i >= 0; --i) {
            int b = pattern [i] & 0xFF;
            if (node.nodes == null) {
                node.nodes = new Node [256];
            }
            if (node.nodes [b] == null) {
                node.nodes [b] = new Node ();
            }
            node = node.nodes [b];
        }
        node.boundary = true;
    }
}
{% endhighlight %}

This is much simpler than initialisation of the `LastByteSuffixMatcher`: we don't need to
search anything, we just index what we have. We make use of a relatively small branching factor (256).

Here are the conventions:

- each `node` represents a string (some substring of the pattern); the `root` node represents an
empty string;
- if `node.boundary` is true, this string is a prefix of the pattern;
- if `node.nodes == null`, this node represents a maximal string (a string that can't be expanded
left to another substring of the pattern).
- if `b` is a byte and `node.nodes[b]` is `null`, this node's string, expanded left by
this byte, does not occur in the pattern.

Note that `node.nodes == null` can only happen if `node.boundary` flag is set. The opposite is not true:
a substring that is a prefix of the pattern can be expanded left to another prefix of the pattern.
We've seen the examples of this before.

This is what the search procedure now looks like:

{% highlight text %}
public int indexOf (byte[] text, int fromIndex)
{
    byte [] pattern = this.pattern;
    int len = pattern.length;
    
    for (int pos = fromIndex; pos < text.length - len + 1;) {
        Node node = root;
        int shift = len;

        for (int i = len - 1; ; --i) {
            node = node.nodes [text [pos + i] & 0xFF];
            if (node == null) {
                break;
            }
            if (node.boundary) {
                shift = i;
                if (node.nodes == null) {
                    break;
                }
            }
        }
        if (shift == 0) return pos;
        pos += shift;
    }
    return -1;
}
{% endhighlight %}

Both initialisation and search procedures are very simple and easy to read -- much simpler,
for instance, than the `LastByteSuffixMatcher`.

Here are some comments on this code:

- iterations traverse the search tree instead of comparing the current portion to the pattern;
the iteration count is not affected by equality or inequality of encountered bytes to those of the
pattern;

- there is no direct comparison of the pattern to the current portion at all; the match is found
if the tree is fully traversed. This means that the `pattern` variable is unnecessary. It is kept
for debug purposes only;

- we start with an unsafe value of the shift (`len`), and decrease it as we proceed, when we encounter
nodes marked `boundary`. This means that we have to go as deep as possible into the tree, and are
not allowed to abort iterations prematurely;

- all previous versions could be modified by using smaller number of bits than eight in each byte
to access the arrays; the arrays could thus be allocated smaller. This would decrease the average
shift but could be handy when units of text were not bytes but, say, Unicode characters.
This version is different: since the tree traversal replaces string comparison, the tree must be exact.
An explicit comparison of the pattern to the current portion will be required if smaller arrays
are used.

Here are the stats:

<table class="numeric">
<tr><th> Statistic </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">Avg shift </td><td> 3.9</td><td> 7.9</td><td> 15.9</td><td>31.9</td><td>63.9</td><td>95.8</td><td>105.9</td></tr>
<tr><td class="ttext">Avg count </td><td> 1.3</td><td> 1.5</td><td> 1.8</td><td> 2.1</td><td> 2.4</td><td> 2.6</td><td> 2.7</td></tr>
</table>

Both statistics look very good. We manage to shift the pattern by very close to its full length,
and to establish this shift in under three iterations.

How is the speed?

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">LastByte</td><td>  1.54</td><td>  0.90</td><td>  0.56</td><td>  0.40</td><td>  0.30</td><td>  0.26</td><td>  0.22</td></tr>
<tr><td class="ttext">Suffix</td><td>  1.89</td><td>  1.33</td><td>  0.74</td><td>  0.41</td><td>  0.27</td><td>  0.22</td><td>  0.11</td></tr>
</table>

The suffix matcher is slower on short patterns and faster on long ones. The speed achieved for
patterns of length 106 is very good; it is 24 times higher than that of the Simple matcher and
9 times higher than the `FirstByteMatcher`. The nice thing here is that the speed was achieved
by better algorithm only -- no intricate optimisations or dirty tricks.

Compiled versions
-----------------

The last result was quite good. Can we possibly improve on that by employing the technique
of generating **Java** code again? After all, the entire search tree is known when we create a matcher,
so why now hard-code the entire tree traversal in **Java**?

I'll be brief here. We tried three approaches:

1) [`CompileSuffixMatcher`]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/CompileSuffixMatcher.java):
we generate a method for every tree node; it calls other methods as it goes deeper. We rely
on JVM to optimise these methods and perform all necessary inlining. Here is an example of one
of the methods (all the methods return calculated shift):

{% highlight java %}
private int match_74 (byte [] text, int pos)
{
    switch (text [pos + 6]) {
        case 32: return match_74_20 (text, pos);
        case 98: return match_74_62 (text, pos);
    }
    return 8;
}
{% endhighlight %}

2) [`CompileSuffixMatcher2`]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/CompileSuffixMatcher2.java):
put the entire search into one function, with corresponding depth of nesting `if`s and `switch`es.
It slows down at length 64 and fails completely at 106: the code of the method becomes too big for the **Java** compiler.

3) [`CompileSuffixMatcher3`]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/CompileSuffixMatcher3.java):
we've seen that the average iteration count is 2.7 for the longest pattern (length 106). It seems
attractive to perform the first iterations in generated **Java** code, and the rest on a tree, as before.
Here is an example of generated code (pattern `doub`):

{% highlight java %}
private static int match (byte [] text, int pos)
{
    switch (text [pos + 3]) {
    case 98: 
        if (text [pos + 2] == 117) return match (text, pos + 1, node_62_75);
        return 4;
    case 100: return 3;
    case 111: 
        if (text [pos + 2] == 100) return 2;
        return 4;
    case 117: 
        if (text [pos + 2] == 111) return match (text, pos + 1, node_75_6F);
        return 4;
    }
    return 4;
}
{% endhighlight %}

Here are the results:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">Suffix</td><td>  1.89</td><td>  1.33</td><td>  0.74</td><td>  0.41</td><td>  0.27</td><td>  0.22</td><td>  0.11</td></tr>
<tr><td class="ttext">CompileSuffix</td><td>  1.93</td><td>  1.43</td><td>  0.91</td><td>  0.52</td><td>  0.32</td><td>  0.24</td><td>  0.16</td></tr>
<tr><td class="ttext">CompileSuffix2</td><td>  1.76</td><td>  1.59</td><td>  0.99</td><td>  0.55</td><td>  1.31</td><td>  0.90</td><td>  fail </td></tr>
<tr><td class="ttext">CompileSuffix3(1)</td><td>  2.05</td><td>  1.70</td><td>  1.05</td><td>  0.56</td><td>  0.37</td><td>  0.29</td><td>  0.18</td></tr>
<tr><td class="ttext">CompileSuffix3(2)</td><td>  2.08</td><td>  1.60</td><td>  1.02</td><td>  0.57</td><td>  0.38</td><td>  0.29</td><td>  0.18</td></tr>
<tr><td class="ttext">CompileSuffix3(3)</td><td>  1.94</td><td>  1.40</td><td>  0.99</td><td>  0.55</td><td>  0.34</td><td>  0.28</td><td>  0.18</td></tr>
<tr><td class="ttext">CompileSuffix3(4)</td><td>  1.76</td><td>  1.56</td><td>  0.98</td><td>  0.54</td><td>  0.33</td><td>  0.26</td><td>  0.17</td></tr>
<tr><td class="ttext">CompileSuffix3(5)</td><td>  1.76</td><td>  1.57</td><td>  0.97</td><td>  0.54</td><td>  0.33</td><td>  0.25</td><td>  0.16</td></tr>
<tr><td class="ttext">CompileSuffix3(6)</td><td>  1.76</td><td>  1.57</td><td>  0.97</td><td>  0.54</td><td>  0.32</td><td>  0.25</td><td>  0.16</td></tr>
<tr><td class="ttext">CompileSuffix3(7)</td><td>  1.76</td><td>  1.56</td><td>  0.97</td><td>  0.53</td><td>  0.32</td><td>  0.25</td><td>  0.81</td></tr>
<tr><td class="ttext">CompileSuffix3(8)</td><td>  1.76</td><td>  1.55</td><td>  0.97</td><td>  0.54</td><td>  0.32</td><td>  0.91</td><td>  0.84</td></tr>
</table>

The results are not so good. The original version is the best (and, as we remember, on small patterns
the `LastByteMatcher` is even better).

Saving memory: SmallSuffixMatcher
---------------------------------

Repository reference: [SmallSuffixMatcher.java]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/SmallSuffixMatcher.java).

The `SuffixMatcher`, being fast, has one big disadvantage: it allocates a 256-element array of objects
(one kilobyte on JVM with compressed pointers) for each internal tree node, and the upper limit
for the number of these nodes is

  <div class="formula">
  <table>
  <tr>
    <td rowspan="3">N&nbsp;=&nbsp;</td>
    <td class="num">P&nbsp;(P&nbsp;&minus;&nbsp;1)</td>
    <td rowspan="2">&nbsp;+&nbsp;1</td>
  </tr>
  <tr>
    <td class="denom">2</td>
  </tr>
  </table>
  </div>

where _P_ is the pattern length. The actual values are a bit lower, but not by much: the value
for our 106-byte pattern is 5353, while the limit is 5365.

This can become a problem for longer patterns, so it is attractive to reduce this number,
even at the cost of some speed degradation. It may, however, even help improve the speed for very
long patterns due to better caching.

Let's implement tree nodes as different classes depending on their children number. There is a big
variety of options we can take: use vectors with linear search, vectors with binary search, small hash
tables and so on. For now, let's implement a very simple solution where we define three classes of nodes:

- nodes with zero children (correspond to the nodes where `nodes` array is `null`)
- nodes with exactly one child (with the obvious search procedure)
- nodes with more than one child (with a 256-long array, just like before).

We can build these nodes directly, but it felt easier to build the old-style nodes (just using maps instead of
arrays) first and then convert them into proper nodes. This is the matter of taste; practically applied
solutions can do it differently.

So we rename our previous class to `BuildNode`:

{% highlight Java %}
private static final class BuildNode
{
    boolean boundary = false;
    ByteMap<BuildNode> map = null;
    
    BuildNode add (byte b)
    {
        if (map == null) {
            map = new ByteMap<> ();
        }
        BuildNode n = map.get (b);
        if (n == null) {
            n = new BuildNode ();
            map.put (b, n);
        }
        return n;
    }
    
    Node convert ()
    {
        if (map == null) {
            return terminal;
        }
        if (map.size () == 1) {
            ByteMap.Entry<BuildNode> e = map.iterator ().next ();
            return new SingleNode (boundary, e.b, e.n.convert ());
        }
        ArrayNode n = new ArrayNode (boundary);
        for (ByteMap.Entry<BuildNode> e : map) {
            n.put (e.b, e.n.convert ());
        }
        return n;
    }
}
{% endhighlight %}

Here `ByteMap` is a home-made map from `byte` to `BuildNode`
(in [repository]({{ site.REPO-INDEXOF}}/blob/master/byteIndexof/ByteMap.java)). A normal **Java** map could be used
here; I just don't like excessive boxing.

The new `Node` class and its subclasses look like this:

{% highlight Java %}
static final EmptyNode terminal = new EmptyNode ();

private static abstract class Node
{
    public final boolean boundary;
    
    Node (boolean boundary) {
        this.boundary = boundary;
    }
    
    abstract Node get (byte b);

    boolean terminal () { return false;}
}

private static class EmptyNode extends Node
{
    EmptyNode () {super (true);}
    @Override Node get (byte b)   {return null;}
    @Override boolean terminal () {return true;}
}

private static final class SingleNode extends Node
{
    private final int b;
    private final Node node;
    
    SingleNode (boolean boundary, int b, Node node)
    {
        super (boundary);
        this.b = b;
        this.node = node;
    }
    
    @Override
    Node get (byte b)
    {
        return b == this.b ? node : null;
    }
}

private static final class ArrayNode extends Node
{
    private final Node [] nodes = new Node [256];
    
    ArrayNode (boolean boundary)
    {
        super (boundary);
    }

    void put (byte b, Node node)
    {
        nodes [b & 0xFF] = node;
    }
    
    @Override
    Node get (byte b)
    {
        return nodes [b & 0xFF];
    }
}
{% endhighlight %}

There is no information inside a terminal node. That's why we can allocate exactly one of those and
re-use it (previously we couldn't do it, because a terminal node could become internal as we built
the tree).

The search procedure changes, too:

{% highlight Java %}
for (int pos = fromIndex; pos < text.length - len + 1;) {
    Node node = root;
    int shift = len;
    int i;

    for (i = len - 1; ; --i) {
        node = node.get (text [pos + i]);
        if (node == null) {
            break;
        }
        if (node.boundary) {
            shift = i;
            if (node.terminal ()) {
                break;
            }
        }
    }
    if (shift == 0) return pos;
    pos += shift;
}
return -1;
{% endhighlight %}

Obviously, such an amount of virtual calls should affect performance, and it did:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">LastByte</td>   <td>  1.54</td><td>  0.90</td><td>  0.56</td><td>  0.40</td><td>  0.30</td><td>  0.26</td><td>  0.22</td></tr>
<tr><td class="ttext">Suffix</td>     <td>  1.89</td><td>  1.33</td><td>  0.74</td><td>  0.41</td><td>  0.27</td><td>  0.22</td><td>  0.11</td></tr>
<tr><td class="ttext">SmallSuffix</td><td>  2.09</td><td>  1.86</td><td>  1.10</td><td>  0.61</td><td>  0.35</td><td>  0.25</td><td>  0.10</td></tr>
</table>

Results are still better than for `LastByteMatcher` for long patterns, but the boundary value is
now higher (96 rather than 32).

The number of allocated arrays improved dramatically: the pattern of length 106 produced 4335 instances
of `SingleNode` and 49 of `ArrayNode`. Of these, 33 had two children, 7 had three and 4 had four.
It is unclear if these numbers justify special treatment for nodes with very small branch factors.

Possible other options
----------------------

We know that for patterns of length 106 the average iteration count is 2.7. We could keep the first
three levels of our tree as `ArrayNode`s, and implement the rest using some space-saving technique.
We could even hard-code this in our search routine, avoiding virtual calls at these levels. This is
very speculative, as it might not work with data of other nature. And, in any case, it won't run faster
than the `SuffixMatcher`. After all, the purpose of this exercise is to make a matcher close to
`SuffixMatcher` in performance, but less memory-hungry. However, there is one option that has
a potential to be faster as well.

Chain matcher
-------------

Repository reference: [ChainSuffixMatcher.java]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/ChainSuffixMatcher.java).

If we dump a typical search tree now, we'll see that there are lots of nodes of type `SingleNode`,
whose children are again `SingleNode`s, and so on. These `SingleNode`s form a chain, and the search
engine traverses this chain one by one. How about modelling this chain explicitly as a special type
of node? This will require change in the interface of `Node`: it must match itself against a byte
sequence instead of a single byte:

{% highlight Java %}
abstract int match (byte [] text, int pos, int i, int shift);
{% endhighlight %}

Each node matches several incoming bytes to a portion of the tree and then calls the matcher of the
subsequent node. The method returns obtained shift. This way iteration is replaced by tail recursion,
so the main loop body becomes very small:

{% highlight Java %}
for (int pos = fromIndex; pos < text.length - len + 1;) {
    int shift = root.match (text, pos, len, len);
    if (shift == 0) return pos;
    pos += shift;
}
return -1;
{% endhighlight %}

We'll follow the same approach as for `SmallSuffixMatcher`: build a generic tree first, converting it
later to the specialised tree. Here are the classes of that tree:

{% highlight Java %}
private static abstract class Node
{
    public final boolean boundary;
    
    Node (boolean boundary) {
        this.boundary = boundary;
    }
    abstract int match (byte [] text, int pos, int i, int shift);
}

private static class EmptyNode extends Node
{
    EmptyNode () {super (true);}

    @Override
    int match (byte [] text, int pos, int i, int shift)
    {
        return i;
    }
}

private static final class SingleNode extends Node
{
    private final byte b;
    private final Node node;
    
    SingleNode (boolean boundary, byte b, Node node)
    {
        super (boundary);
        this.b = b;
        this.node = node == terminal ? null : node;
    }

    @Override
    int match (byte [] text, int pos, int i, int shift)
    {
        if (boundary) shift = i;
        -- i;
        if (text [pos + i] != b) return shift;
        if (node == null) return i;
        return node.match (text, pos, i, shift);
    }
}

private static final class ArrayNode extends Node
{
    private final Node [] nodes = new Node [256];
    
    ArrayNode (boolean boundary)
    {
        super (boundary);
    }

    void put (byte b, Node node)
    {
        nodes [b & 0xFF] = node;
    }
    
    @Override
    int match (byte [] text, int pos, int i, int shift)
    {
        if (boundary) shift = i;
        -- i;
        Node n = nodes [text [pos + i] & 0xFF];
        if (n == null) return shift;
        return n.match (text, pos, i, shift);
    }
}

private static final class ChainNode extends Node
{
    private final byte [] chain;
    private final Node node;
    
    ChainNode (boolean boundary, byte [] chain, Node node)
    {
        super (boundary);
        this.chain = chain;
        this.node = node;
    }

    @Override
    int match (byte [] text, int pos, int i, int shift)
    {
        if (boundary) shift = i;
        i -= chain.length;
        for (int j = 0; j < chain.length; j++) {
            if (text [pos + i + j] != chain [j]) return shift;
        }
        if (node == null) return i;
        return shift = node.match (text, pos, i, shift);
    }
}
{% endhighlight %}

Most single nodes were converted to chains: our 106-byte pattern now produces 49 arrays, 9 single
nodes and 107 chains of average length 49.5.

Unfortunately, the times didn't improve:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">Suffix</td>     <td>  1.89</td><td>  1.33</td><td>  0.74</td><td>  0.41</td><td>  0.27</td><td>  0.22</td><td>  0.11</td></tr>
<tr><td class="ttext">SmallSuffix</td><td>  2.09</td><td>  1.86</td><td>  1.10</td><td>  0.61</td><td>  0.35</td><td>  0.25</td><td>  0.10</td></tr>
<tr><td class="ttext">ChainSuffix</td><td>  2.50</td><td>  1.77</td><td>  1.11</td><td>  0.61</td><td>  0.35</td><td>  0.25</td><td>  0.15</td></tr>
</table>

There is still some value in this solution: it allocates fewer memory. Here is the amount of memory, in kilobytes,
allocated for several types of matchers:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">LastByte</td>       <td>  1</td><td>  1</td><td>   1</td><td>   1</td><td>   1</td><td>    1</td><td>    1</td></tr>
<tr><td class="ttext">MultiByte</td>      <td>  4</td><td>  8</td><td>  17</td><td>  33</td><td>  67</td><td>  100</td><td>  110</td></tr>
<tr><td class="ttext">LastByteSuffix</td> <td>  1</td><td>  1</td><td>   1</td><td>   1</td><td>   1</td><td>    1</td><td>    1</td></tr>
<tr><td class="ttext">NextByteSuffix</td> <td>  4</td><td>  8</td><td>  17</td><td>  33</td><td>  67</td><td>  100</td><td>  111</td></tr>
<tr><td class="ttext">Suffix</td>         <td>  8</td><td> 30</td><td> 117</td><td> 500</td><td>2020</td><td> 4662</td><td> 5698</td></tr>
<tr><td class="ttext">SmallSuffix</td>    <td>  2</td><td>  3</td><td>  10</td><td>  27</td><td>  77</td><td>  149</td><td>  179</td></tr>
<tr><td class="ttext">ChainSuffix</td>    <td>  2</td><td>  2</td><td>   8</td><td>  18</td><td>  37</td><td>   54</td><td>   62</td></tr>
</table>

Light chain suffix matcher
--------------------------

Repository reference: [LightChainSuffixMatcher.java]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/LightChainSuffixMatcher.java).

There are multiple ways the `ChainMatcher` can be improved. We tried two, which are not presented here
for the reason that they didn't improve speed. They are, however, published in the repository.
In one, [`ChainSuffixMatcher2`]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/ChainSuffixMatcher2.java),
tree nodes were specialised by presence of the child (`node` being `null` in `SingleNode` or `ChainNode`).
In another one, [`ChainSuffixMatcher3`]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/ChainSuffixMatcher3.java),
they were also specialised by the `boundary` flag. This
complicated the code but made no performance difference.

This doesn't mean there are no ways to improve speed. And there are definitely ways to improve
memory footprint. The table above shows very good memory use for `ChainSuffix`, but only compared
to other methods.  If the pattern length grows, so does the memory use, eventually making this method
inapplicable. We want to move this point as far right as possible on the pattern length scale.

One way to save memory looks next to trivial: all our chains are in fact substrings of our pattern,
so they can be represented as offsets and lengths, with no need to copy them into specially allocated
arrays. This is a second-degree improvement compared to what we've just done (remember, previously
each byte of a chain was represented by a node (a **Java** object), and before that - as an array
of 256 elements. Still, getting rid of this array looks attractive. All that's necessary is to
collect the positions of the original nodes in the pattern, and, obviously, provide the `ChainNode`s
with the reference of the pattern.

I'll skip the modifications to the tree building procedure (see in the repository); here is the
new `ChainNode`:

{% highlight Java %}
private static final class ChainNode extends Node
{
    private final byte [] pattern;
    private final int start;
    private final int len;
    private final Node node;
    
    ChainNode (byte [] pattern, int position, int len,
               boolean boundary, Node node)
    {
        super (boundary);
        this.pattern = pattern;
        this.start = position - len;
        this.len = len;
        this.node = node;
    }
    
    @Override
    int match (byte [] text, int pos, int i, int shift)
    {
        if (boundary) shift = i;
        i -= len;
        for (int j = 0; j < len; j++) {
            if (text [pos + i + j] != pattern [start + j]) {
                return shift;
            }
        }
        if (node == null) {
            return i;
        }
        return shift = node.match (text, pos, i, shift);
    }
}
{% endhighlight %}

Here are the times:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">ChainSuffix</td><td>  2.50</td><td>  1.77</td><td>  1.11</td><td>  0.61</td><td>  0.35</td><td>  0.25</td><td>  0.15</td></tr>
<tr><td class="ttext">LightChainSuffix</td><td>  2.51</td><td>  1.79</td><td>  1.13</td><td>  0.62</td><td>  0.35</td><td>  0.25</td><td>  0.14</td></tr>

</table>

And the memory use:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">ChainSuffix</td>         <td>  2</td><td>  2</td><td>   8</td><td>  18</td><td>  37</td><td>   54</td><td>   62</td></tr>
<tr><td class="ttext">LightChainSuffix</td>    <td>  2</td><td>  2</td><td>   8</td><td>  17</td><td>  34</td><td>   48</td><td>   56</td></tr>
</table>

The memory use isn't much lower than before, but we'll look at this again when considering longer
patterns.

There are other ways to reduce memory, e.g. to re-use some subtrees, essentially converting the tree
into a [DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph). This, however, falls outside of
the scope of this article, which already has grown too long.

Let's however, do the last effort.

<div id="LightestChainSuffixMatcher"></div>

Lightest chain suffix matcher
-----------------------------

Repository reference: [LightestChainSuffixMatcher.java]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/LightestChainSuffixMatcher.java).

The `LightChainSuffixMatcher` is good in speed and in memory, but lacks a good procedure for
its instantiation. The way the tree is created is obviously non-optimal. The final tree is well
optimised for space, but the initial tree is not, and, as we'll see later, it may require quite
an amount of memory. It is also not very quick to build. Even though our primary interest is in
application of matchers and not in their instantiation, we still prefer our solution to be
practically usable for wide range of inputs.

Let's save some time and space by creating the tree in one step. Another change is that
we'll also insert full strings into the tree rather than bytes one by one.

Here is the base class for a tree node:

{% highlight Java %}
private static abstract class Node
{
    boolean boundary;
            
    abstract Node add (byte [] pattern, int start, int len);
    abstract int match (byte [] text, int pos, int i, int shift);
}
{% endhighlight %}

A new method `add` will insert a given substring of `pattern` into this node, creating, if necessary,
more nodes and, possibly, replacing this node with a new one.

We'll sacrifice a `SingleNode` (it will be replaced with a `ChainNode` of length 1; it can be re-introduced,
but the code will become longer and less clear with uncertain benefits). There will be three subclasses
of `Node`:

- `ArrayNode`: a branch node, which has more than one child. Currently, always implemented as
a 256-long array, but, for saving on memory, can be replaced by something smaller. Null value in
the array means that corresponding byte is illegal in this state, otherwise, we can descend one
byte deeper into the tree;

- `ChainNode`: a node representing a string of bytes; it has a single child indicating the next node
to match; `null` means end of matching process.

- `EmptyNode`: a terminal node that indicates end of matching process for an `ArrayNode` (`null` has
other meaning there). This node is immutable, so exactly one instance is needed.

The `boundary` bit indicates that the path in the tree from the root to (but not including) this node
represents a valid prefix of the pattern. The `EmptyNode` always has this bit on. This bit always
applies to the whole node; if a prefix of the pattern ends in the middle of a chain, this chain is
split in two.

Here are our nodes:

{% highlight Java %}
private static class EmptyNode extends Node
{
    EmptyNode () {boundary = true;}
    
    @Override
    Node add (byte [] pattern, int start, int len)
    {
        Node n = new ChainNode (pattern, start, len, null);
        n.boundary = true;
        return n;
    }

    @Override
    int match (byte [] text, int pos, int i, int shift)
    {
        return i;
    }
}

private static final class ArrayNode extends Node
{
    private final Node [] nodes = new Node [256];
    
    @Override
    Node add (byte [] pattern, int start, int len)
    {
        byte b = pattern [start +-- len];
        Node n = nodes [b & 0xFF];
        if (n != null) {
            if (len == 0) {
                n.boundary = true;
            } else {
                nodes [b & 0xFF] = n.add (pattern, start, len);
            }
        } else {
            nodes [b & 0x0FF] = len == 0 ? terminal :
                                new ChainNode (pattern, start, len, null);
        }
        return this;
    }
    
    void put (byte b, Node node)
    {
        if (node == null) node = terminal;
        nodes [b & 0xFF] = node;
    }

    @Override
    int match (byte [] text, int pos, int i, int shift)
    {
        if (boundary) shift = i;
        -- i;
        Node n = nodes [text [pos + i] & 0xFF];
        if (n == null) return shift;
        return n.match (text, pos, i, shift);
    }
}

private static final class ChainNode extends Node
{
    private final byte [] pattern;
    private int start;
    private int len;
    private Node node;
    
    ChainNode (byte [] pattern, int start, int len, Node node)
    {
        this.pattern = pattern;
        this.start = start;
        this.len = len;
        this.node = node;
    }

    @Override
    Node add (byte [] pattern, int start, int len)
    {
        Node result = this;
        int end = start + len - 1;
        int this_end = this.start + this.len - 1;
        
        int min_len = Math.min (len,  this.len);
        int i;
        for (i = 0; i < min_len; i++) {
            if (pattern [end - i] != pattern [this_end - i]) {
                break;
            }
        }
        if (i == this.len) {
            if (this.len == len) {
                if (node != null) {
                    node.boundary = true;
                }
            } else {
                if (node != null) {
                    node = node.add (pattern, start, len - i);
                } else {
                    node = new ChainNode (pattern, start, len - i, null);
                    node.boundary = true;
                }
            }
        } else if (i == len) {
            node = new ChainNode (pattern, this.start, this.len - len, node);
            node.boundary = true;
            this.start = this.start + this.len - i;
            this.len = i;
        } else {
            byte chain_byte = pattern [this_end - i];
            ArrayNode array = new ArrayNode ();
            if (i == 0) {
                array.boundary = boundary;
                boundary = false;
                -- this.len;
                array.put (chain_byte, this.len == 0 ? node : this);
                result = array;
            } else if (i == this.len - 1) {
                ++ this.start;
                -- this.len;
                array.put (chain_byte, node);
                node = array;
            } else {
                Node new_chain = new ChainNode (pattern, this.start,
                                                this.len-i-1, node);
                this.start = this.start + this.len - i;
                this.len = i;
                array.put (chain_byte, new_chain);
                node = array;
            }
            array.add (pattern, start, len - i);
        }
        return result;
    }
    
    @Override
    int match (byte [] text, int pos, int i, int shift)
    {
        if (boundary) shift = i;
        i -= len;
        for (int j = 0; j < len; j++) {
            if (text [pos + i + j] != pattern [start + j])
                return shift;
        }
        if (node == null) return i;
        return shift = node.match (text, pos, i, shift);
    }
}
{% endhighlight %}

The `ChainNode.add` looks complicated but actually is not. It must cater for four major cases:

- the added string is a proper suffix of the chain: split the chain at that point to put a `boundary`
flag on the second chain;
- the chain is a proper suffix of the added string: add the remaining string to the chain's child, if it exists,
otherwise make the remaining string such a child with the `boundary` flag;
- the added string is equal to the chain: if there is a next node, set the `boundary` flag there;
- the string and the chain differ somewhere: create an `ArrayNode` and insert it into an
appropriate place. There are three sub-cases: the array node may replace the first byte of the chain, or the last one,
or some middle one, splitting the chain in two.

The tree-building procedure now looks very neat:

{% highlight Java %}
private Node build_root ()
{
    Node root = terminal;
    for (int prefix_len = pattern.length; prefix_len >= 1; prefix_len--) {
        root = root.add (pattern, 0,  prefix_len);
    }
    return root;
}
{% endhighlight %}

The matching procedure looks nice, too:

{% highlight Java %}
public int indexOf (byte[] text, int fromIndex)
{
    int len = pattern.length;
    for (int pos = fromIndex; pos < text.length - len + 1;) {
        int shift = root.match (text, pos, len, len);
        if (shift == 0) return pos;
        pos += shift;
    }
    return -1;
}
{% endhighlight %}

The times are very close to those of the `LightChainMatcher` -- removal of the `SingleNode` hasn't affected them:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">LightChainSuffix</td><td>  2.51</td><td>  1.79</td><td>  1.13</td><td>  0.62</td><td>  0.35</td><td>  0.25</td><td>  0.14</td></tr>
<tr><td class="ttext">LightestChainSuffix</td><td>  2.43</td><td>  1.71</td><td>  1.11</td><td>  0.61</td><td>  0.35</td><td>  0.25</td><td>  0.14</td></tr>
</table>

This was the last of the algorithms we planned to look at today. Now it's time to look at other aspects of string search.

String matcher
--------------

The primary reason why we went on this journey was absence of `indexOf` method for byte arrays,
which was needed for my real-world problem. However, there are such methods for **Java** Strings,
and we can compare them with our solutions converted to work with strings.

We'll convert some of our solutions (too much work to do it for everything; besides, it feels unnecessary),
namely `Simple`, `FirstBytes`, `LastByte` and `Suffix`. To simplify things, we'll ignore Unicode 
and limit our character set to 256 characters.

Besides mentioned matchers, we'll add three string-specific ones:

- `StandardMatcher`: the same as `SimpleMatcher`, except it calls `String.regionMatch()` method instead
of our `compare()`. The methods are nearly identical, except the former one is implemented
right inside the `String` class and has direct access to its internals. This is the way to check if
it gives any advantage.

- `IndexofMatcher`: makes direct use of `String.indexof()`, which is (as of **Java 8**) essentially
the same as our `FirstByteMatcher`

- `RegexMatcher`: we compile the pattern
as a regular expression and then use this pre-compiled pattern. Note that an
arbitrary string isn't yet a valid regex pattern: it may contain special characters that have
their own semantics and must be properly escaped. We'll ignore this for now, as our artificial
example doesn't contain such characters.

The code is [here]({{ site.REPO-INDEXOF }}/blob/master/stringIndexof), and the main class is
[here]({{ site.REPO-INDEXOF }}/blob/master/stringIndexof/Indexof.java).

Here are the times:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">Standard</td>  <td class="yellow"> 1.43</td><td>                 1.37</td><td>                1.41</td><td>                 1.41</td><td>                 1.43</td><td>                 1.35</td><td>                0.72</td></tr>
<tr><td class="ttext">Simple</td>    <td>                2.99</td><td>                 3.42</td><td>                3.45</td><td>                 3.45</td><td>                 3.49</td><td>                 3.38</td><td>                2.58</td></tr>
<tr><td class="ttext">Indexof</td>   <td class="red">    1.53</td><td>                 1.51</td><td>                1.54</td><td>                 1.53</td><td>                 1.56</td><td>                 1.47</td><td>                0.80</td></tr>
<tr><td class="ttext">FirstBytes</td><td class="green">  1.16</td><td>                 2.14</td><td>                2.16</td><td>                 2.16</td><td>                 2.18</td><td>                 2.10</td><td>                1.56</td></tr>
<tr><td class="ttext">Regex</td>     <td>                2.25</td><td class="yellow">  1.26</td><td class="red">    0.79</td><td class="red">     0.56</td><td class="red">     0.41</td><td class="red">     0.35</td><td class="red">    0.31</td></tr>
<tr><td class="ttext">LastByte</td>  <td>                2.03</td><td class="green">   0.90</td><td class="green">  0.58</td><td class="yellow">  0.43</td><td class="yellow">  0.33</td><td class="yellow">  0.28</td><td class="yellow"> 0.23</td></tr>
<tr><td class="ttext">Suffix</td>    <td>                1.93</td><td class="red">     1.32</td><td class="yellow"> 0.73</td><td class="green">   0.42</td><td class="green">   0.27</td><td class="green">   0.22</td><td class="green">  0.13</td></tr>
</table>

The first place in each column is marked green, the second is yellow and the third is red.

The observations:

- `Standard` is roughly 10% faster than `IndexOf`, which is surprising;

- the `Simple` matcher does the same as the `Standard` one, just calls `String.charAt()` instead of
char array access. This makes hell of a difference: it works at half the speed;

- `FirstBytes` is still much faster than `Simple`, but slower than the matchers based on the
standard library;

- the compiled regular expression matcher is really fast, quite a bit faster than all na&iuml;ve matchers;

- we still managed to outperform it using both `LastByte` and `Suffix` matchers;

- of these two matchers, the `LastByte` is faster for shorter patterns, while `Suffix` is faster for
longer ones (beginning at 32).

The Regex pattern matcher
-------------------------

Repository reference: [RegexByteMatcher.java]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/RegexByteMatcher.java).

The pattern matching engine of **Java** is designed to handle much more complex cases than ours:
it works with potentially very sophisticated regular expressions. Taking this into account, even though
we managed to overtake it, this engine performed very well in our test. How was this achieved?

The pattern search routine, starting at `java.util.regex.Matcher.find ()`, quickly ends up at
the `java.util.regex.Matcher.BnM.match ()` (`BnM` standing for
[Boyer-Moore search algorithm](https://en.wikipedia.org/wiki/Boyer%E2%80%93Moore_string-search_algorithm), 1977):

{% highlight Java %}
boolean match(Matcher matcher, int i, CharSequence seq) {
    int[] src = buffer;
    int patternLength = src.length;
    int last = matcher.to - patternLength;

NEXT:
     while (i <= last) {
         for (int j = patternLength - 1; j >= 0; j--) {
             int ch = seq.charAt(i+j);
             if (ch != src[j]) {
                 i += Math.max(j + 1 - lastOcc[ch&0x7F], optoSft[j]);
                 continue NEXT;
             }
         }
         .....
     }
     .....
}
{% endhighlight %}
 
There is more code there, related to matching more complex patterns than just a simple string,
which we can ignore for now. This is, however, the main execution path in our case.
Here are some immediate observations on this code:

- This code looks very neat.

- Labels may look odd for someone grown up with the
["Goto statement considered harmful"](https://homepages.cwi.nl/~storm/teaching/reader/Dijkstra68.pdf)
statement, but can be used quite successfully by **Java** developers; after all, **Java** labels
are not quite **goto**.

- The **Java** library developers also prefer to pre-load fields into local variables, just like we did.
It's not clear if it makes any difference to the code generated, but let's not break the tradition.

- This code doesn't make use of the string internals: the pattern is converted into an
`int` array, while the text is addressed as a `CharSequence` using a virtual
call to `charAt()`. So it's possible to make a fast matcher even under these limitations.

- You can have your name given even to such a simple algorithm if you are the first
to invent it.

We didn't use this exact algorithm anywhere above; the closest to it
was the `NextByteSuffixMatcher`. This is what the Regex code does:

1) compare the current portion to the pattern right to left and identify the good suffix
(we remind that this is the longest suffix that matches, and we know the most common case is the suffix of length zero);

2) if this suffix spans the entire pattern, we have a match;

2) otherwise, look up the shift for this suffix in a prepared table, just like we did;

3) then look at the bad character (the first character that didn't match), and get a shift for this character.
The greater of the two shifts is used.

The last step differs from our implementation: we had a
separate shift table for each position; they use just one table, calculated for the last position
(exactly the same as our table for `LastByteMatcher`), adjusting the shift by the bad character's position.
The shift obtained this way, while safe, must be, generally, worse than the shift we got.
Let's see by how much:

<table class="numeric">
<tr><th> Statistic </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>

<tr><td class="ttext">Avg shift, LastByteSuffix </td><td> 3.5</td><td> 6.0</td><td>  9.6</td><td> 13.9</td><td> 19.2</td><td> 22.2</td><td> 24.8</td></tr>
<tr><td class="ttext">Avg shift, Regex          </td><td> 3.5</td><td> 6.1</td><td> 10.0</td><td> 14.8</td><td> 20.9</td><td> 24.3</td><td> 27.2</td></tr>
<tr><td class="ttext">Avg shift, NextByteSuffix </td><td> 3.5</td><td> 6.1</td><td> 10.0</td><td> 14.8</td><td> 20.9</td><td> 24.3</td><td> 27.2</td></tr>
</table>

Here comes a surprise: the average shift counts for the `Regex` are exactly the same as for
`NextByte`. Is it simply a rounding problem, the results being similar but not equal? We can print
the raw data used to calculate averages. They are exactly the same. For instance, for the pattern length
of 16 both algorithms report the shift sum of  1514837185 for 151637812 operations.

This can't be a coincidence, and, indeed, it's not. The shift generated by multiple tables (`NextByte`)
can differ from the shift from a single table (`Regex`) only when the latter is negative, that is,
if the bad character occurs in the good suffix.

Anything that can only be found to the left of the bad character, 
gets the same shifts from both algorithms:

{% highlight text %}
 beetles oer his base into the sea and there ass
                doubt thou the s
{% endhighlight %}

Here the bad character is `o`, which gets the shift of one from the `NextByteMatcher` array,
and the shift of 7 found in `Regex` table, adjusted by its position 6, also produces 1.

When the `Regex` shift of the bad character is negative, the final shift is determined by the good
suffix shift. It is easy to see that it is never smaller than the shift of `NextByteMatcher` for
the bad character.

Let the pattern length be _l_ and the good suffix length be _n_. We have two cases:

- the good suffix is never found in the pattern left from itself. Let the longest suffix of the good suffix
that is also a prefix of the pattern have length _m_ (possibly zero). In this case this suffix produces shift
_l_ &minus; _m_, which is
greater than _l_ &minus; _n_, which is the maximal possible shift for the bad character (obtained when it doesn't
occur in the pattern left of itself).

Here is the example demonstrating this case, where _m_ = 3:

{% highlight text %}
which his melancholy sits on brood and i do doubt the hatch and
               doubt that the sun doth move dou
{% endhighlight %}

The bad character is `o`, and its shift is &minus;3, so the shift of 29 is produced by the good suffix
`" dou"`, of which the suffix `dou` if a prefix of the pattern (_l_&nbsp;=&nbsp;32, _n_&nbsp;=&nbsp;4, _m_&nbsp;=&nbsp;3).

- the good suffix is found in the pattern _k_ &ge; _n_ positions left from itself, which means that it produces the shift of _k_.
In this case the bad character, being part of this suffix, is found in the pattern between positions _k_
and _k_ + _n_ &minus; 1 from the right. This makes its shift _k_ &minus; _n_ to _k_ &minus; 1, which is less than _k_.

Here is the example demonstrating this case:

{% highlight text %}
for every thing is seald and done
               sun doth move do
{% endhighlight %}

Here the suffix is `" do"` of length 3, and its shift _k_&nbsp;=&nbsp;10. The bad character is `d` (occurs in this suffix),
and it is found 8 positions left from itself. The `NextByte` implementation will give 8, the `Regex` will produce &minus;2,
and the total shift will be determined by the suffix shift (10).

So it is proven that the algorithm with one table produces exactly the same results as the
algorithm with a table at every character position. This is remarkable, and it wasn't obvious immediately. This
shows that there is a reason algorithms get inventors' names.

It's worth noticing, however, that the `Regex` algorithm, even being very smart, is still a variation of
our `LastByteMatcher`. In 90% cases (when the last character does not match) it produces the same shift.
The reason it performs so well as a string matcher (although it lost to `LastByte`, and lost big time to `Suffix` on
long patterns) is its simplicity.

The RegexByteMatcher
--------------------

Repository reference: [RegexByteMatcher.java]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/RegexByteMatcher.java).

A detour into string matchers was quite productive: we've learned another good algorithm,
which is close, but not identical to those considered before. Let's return to the byte array matching
territory and try implementing this algorithm. The new matcher will get its name based on its origin
(`RegexByteMatcher`):

{% highlight Java %}
public int indexOf (byte[] text, int fromIndex)
{
    byte [] pattern = this.pattern;
    int len = pattern.length;
    int [] shifts = this.shifts;
    int [] suffix_shifts = this.suffix_shifts;
    
    int pos = fromIndex;
NEXT:
    while (pos <= text.length - len) {
        for (int i = len-1; i >= 0; i--) {
            byte b = text [pos + i];
            if (b != pattern [i]) {
                pos += Math.max (shifts [b & 0xFF] - (len - 1 - i),
                                 suffix_shifts [i]);
                continue NEXT;
            }
        }
        return pos;
    }
    return -1;
}
{% endhighlight %}

The shift statistics is exactly the same as expected, which means we've implemented it right. The times, however, are
not so good:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">NextByteSuffix</td><td>  1.70</td><td>  0.98</td><td>  0.60</td><td>  0.42</td><td>  0.31</td><td>  0.26</td><td>  0.21</td></tr>
<tr><td class="ttext">Regex</td><td>  2.16</td><td>  1.24</td><td>  0.76</td><td>  0.52</td><td>  0.38</td><td>  0.32</td><td>  0.27</td></tr>
</table>

What could have caused such a difference? The `NextByteSuffix` matcher, apart from using multiple tables, also employs
a special treatment of the last byte. This is a classic case of a partial loop unroll: if there are big chances that
the loop is executed exactly once (which is our case, the probability is 90%), and the loop body in this case is simpler
than the general one, move it out of the loop and adjust the iteration count. Sometimes, a compiler can perform this
transformation automatically, but often it does not have enough information to do so. Let's help it. We saw that
previously it helped in `FirstByteMatcher`; let's try again, we'll call it `RegexMatcher2`
(repository reference: [RegexByteMatcher2.java]({{ site.REPO-INDEXOF }}/blob/master/byteIndexof/RegexByteMatcher2.java)).

{% highlight Java %}
public int indexOf (byte[] text, int fromIndex)
{
    byte [] pattern = this.pattern;
    int len = pattern.length;
    int [] shifts = this.shifts;
    int [] suffix_shifts = this.suffix_shifts;
    byte last = pattern [len-1];
    
    int pos = fromIndex;
NEXT:
    while (pos <= text.length - len) {
        byte b = text [pos + len-1];
        if (b != last) {
            pos += shifts [b & 0xFF];
            continue NEXT;
        }
        for (int i = len-2; i >= 0; i--) {
            b = text [pos + i];
            if (b != pattern [i]) {
                pos += Math.max (shifts [b & 0xFF] - (len - 1 - i),
                                 suffix_shifts [i]);
                continue NEXT;
            }
        }
        return pos;
    }
    return -1;
}
{% endhighlight %}

Two factors may contribute to performance of this code: the `last` variable loaded once (hopefully, stored in a
register), and the `suffix_shifts` array not being accessed when the suffix length is zero (the shift array contains zero
in this case, but it would be too much to expect the compiler to make use of that). So, this is the speed:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">LastByte</td><td>  1.54</td><td>  0.90</td><td>  0.56</td><td>  0.40</td><td>  0.30</td><td>  0.26</td><td>  0.22</td></tr>
<tr><td class="ttext">NextByteSuffix</td><td>  1.70</td><td>  0.98</td><td>  0.60</td><td>  0.42</td><td>  0.31</td><td>  0.26</td><td>  0.21</td></tr>
<tr><td class="ttext">Regex</td><td>  2.16</td><td>  1.24</td><td>  0.76</td><td>  0.52</td><td>  0.38</td><td>  0.32</td><td>  0.27</td></tr>
<tr><td class="ttext">Regex2</td><td>  1.74</td><td>  1.00</td><td>  0.62</td><td>  0.43</td><td>  0.31</td><td>  0.26</td><td>  0.21</td></tr>
</table>

Performance has improved quite a bit and is now pretty much the same as that of `NextByteSuffixMatcher`, but the code
is simpler. It is, however, still more complex than the simple `LastByteMatcher`, and is not faster than that.

<div id="Summary"></div>

Results summary
---------------

We tried many options:

- a `Simple` matcher that uses plain string comparison: runs rather slowly;
- several of `FirstByte` matchers, optimised for special treatment of first bytes; definite
speed improvement;
- a `Compile` version that creates **Java** class on the fly: marginally faster;
- a `Hash` matcher: the first attempt to use non-trivial algorithm; failed miserably;
- a `LastByte` matcher: another non-trivial algorithm and the first to produce shift values
bigger than one; a big success;
- several `MultiByte` versions that improve shift values at the cost of more complex code
and fail to improve speed;
- `LastByteSuffix`: the first attempt to operate on entire suffixes rather than single characters;
failed due to limited subset of cases it addressed;
- `NextByteSuffix`: a combination of the suffixes approach and the single character approach: worked
well but not faster than the `LastByte`;
- `Regex`: a classic algorithm that has a name and a [Wiki page](https://en.wikipedia.org/wiki/Boyer%E2%80%93Moore_string-search_algorithm); is a variation of the `NextByteSuffix`,
and, being optimised to `Regex2`, performs as fast as that one;
- `Suffix`: a generalised algorithm operating on suffixes and addressing all the cases; a good success
on longer patterns but suffers there from high memory demand;
- several compiled versions of suffix matchers, failed to improve speed (only one is included in the table to keep
it shorter);
- `SmallSuffix`: a way to reduce memory requirements of `Suffix` at the cost of some performance
loss;
- `ChainSuffix`: further reduction of memory requirements; the same or slightly worse performance as
that of the `SmallSuffix`;
- `LightChainSuffix`: an improvement of memory use of `ChainSuffix`; pretty much the same performance
on the tested pattern lengths;
- `LightestChainSuffix`: identical matching algorithm to the previous one, but better procedure for
constructing the matcher.

Here are all the results:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>96</th><th>106</th></tr>
<tr><td class="ttext">Simple</td><td>  2.90</td><td>  2.89</td><td>  3.30</td><td>  3.28</td><td>  3.29</td><td>  3.21</td><td>  2.68</td></tr>
<tr><td class="ttext">FirstByte</td><td class="red">  1.47</td><td>  1.51</td><td>  1.52</td><td>  1.52</td><td>  1.49</td><td>  1.46</td><td>  0.92</td></tr>
<tr><td class="ttext">FirstBytes</td><td class="green">  1.18</td><td>  1.35</td><td>  1.36</td><td>  1.37</td><td>  1.38</td><td>  1.32</td><td>  0.86</td></tr>
<tr><td class="ttext">FirstBytes2</td><td class="yellow">  1.21</td><td>  1.18</td><td>  1.19</td><td>  1.19</td><td>  1.20</td><td>  1.16</td><td>  0.83</td></tr>
<tr><td class="ttext">Compile</td><td>  1.18</td><td>  1.38</td><td>  1.38</td><td>  1.39</td><td>  1.34</td><td>  1.27</td><td>  0.80</td></tr>
<tr><td class="ttext">Hash</td><td>  2.59</td><td>  2.47</td><td>  2.43</td><td>  2.42</td><td>  2.40</td><td>  2.39</td><td>  2.40</td></tr>
<tr><td class="ttext">LastByte</td><td>  1.54</td><td class="green">  0.90</td><td class="green">  0.56</td><td class="green">  0.40</td><td class="yellow">  0.30</td><td class="red">  0.26</td><td>  0.22</td></tr>
<tr><td class="ttext">MultiByte</td><td>  2.59</td><td>  1.90</td><td>  1.32</td><td>  0.95</td><td>  0.69</td><td>  0.54</td><td>  0.45</td></tr>
<tr><td class="ttext">MultiByte2</td><td>  2.39</td><td>  1.73</td><td>  1.22</td><td>  0.88</td><td>  0.63</td><td>  0.48</td><td>  0.41</td></tr>
<tr><td class="ttext">LMultiByte (2)</td><td>  2.63</td><td>  1.90</td><td>  1.20</td><td>  0.71</td><td>  0.45</td><td>  0.37</td><td>  0.22</td></tr>
<tr><td class="ttext">LMultiByte (3)</td><td>  2.66</td><td>  1.96</td><td>  1.29</td><td>  0.80</td><td>  0.51</td><td>  0.41</td><td>  0.28</td></tr>
<tr><td class="ttext">LMultiByte (4)</td><td>  2.66</td><td>  1.98</td><td>  1.35</td><td>  0.86</td><td>  0.55</td><td>  0.43</td><td>  0.30</td></tr>
<tr><td class="ttext">LMultiByte2</td><td>  2.06</td><td>  1.32</td><td>  0.77</td><td>  0.48</td><td>  0.34</td><td>  0.28</td><td>  0.18</td></tr>
<tr><td class="ttext">UnrolledLMultiByte (2)</td><td>  2.14</td><td>  1.38</td><td>  0.79</td><td>  0.48</td><td>  0.33</td><td>  0.27</td><td>  0.16</td></tr>
<tr><td class="ttext">UnrolledLMultiByte (3)</td><td>  2.19</td><td>  1.49</td><td>  0.90</td><td>  0.54</td><td>  0.36</td><td>  0.29</td><td class="red">  0.15</td></tr>
<tr><td class="ttext">UnrolledLMultiByte (4)</td><td>  2.31</td><td>  1.57</td><td>  0.97</td><td>  0.59</td><td>  0.38</td><td>  0.30</td><td class="yellow">  0.11</td></tr>
<tr><td class="ttext">NextByteSuffix</td><td>  1.70</td><td class="red">  0.98</td><td class="red">  0.60</td><td class="red">  0.42</td><td class="red">  0.31</td><td class="red">  0.26</td><td>  0.21</td></tr>
<tr><td class="ttext">Regex</td><td>  2.16</td><td>  1.24</td><td>  0.76</td><td>  0.52</td><td>  0.38</td><td>  0.32</td><td>  0.27</td></tr>
<tr><td class="ttext">Regex2</td><td>  1.74</td><td>  1.00</td><td>  0.62</td><td>  0.43</td><td class="red">  0.31</td><td class="red">  0.26</td><td>  0.21</td></tr>
<tr><td class="ttext">LastByteSuffix</td><td>  1.56</td><td class="yellow">  0.91</td><td class="yellow">  0.57</td><td class="green">  0.40</td><td class="yellow">  0.30</td><td class="yellow">  0.25</td><td>  0.20</td></tr>
<tr><td class="ttext">Suffix</td><td>  1.89</td><td>  1.33</td><td>  0.74</td><td class="yellow">  0.41</td><td class="green">  0.27</td><td class="green">  0.22</td><td class="yellow">  0.11</td></tr>
<tr><td class="ttext">CompileSuffix    </td><td>  1.93</td><td>  1.43</td><td>  0.91</td><td>  0.52</td><td>  0.32</td><td>  0.24</td><td>  0.16</td></tr>
<tr><td class="ttext">SmallSuffix</td><td>  2.09</td><td>  1.86</td><td>  1.10</td><td>  0.61</td><td>  0.35</td><td class="yellow">  0.25</td><td class="green">  0.10</td></tr>
<tr><td class="ttext">ChainSuffix</td><td>  2.50</td><td>  1.77</td><td>  1.11</td><td>  0.61</td><td>  0.35</td><td class="yellow">  0.25</td><td>  0.15</td></tr>
<tr><td class="ttext">LightChainSuffix</td><td>  2.51</td><td>  1.79</td><td>  1.13</td><td>  0.62</td><td>  0.35</td><td class="yellow">  0.25</td><td class="red">  0.14</td></tr>
<tr><td class="ttext">LightestChainSuffix</td><td>  2.43</td><td>  1.71</td><td>  1.11</td><td>  0.61</td><td>  0.35</td><td class="yellow">  0.25</td><td class="red">  0.14</td></tr>
</table>

The colour scheme is the same as before (green -- yellow -- red for the first, second and third place).
The compiled versions are excluded from the competition as very exotic. `LastByteMatcher` and its
variants are clearly winning on shorter patterns, while the family of the `SuffixMatcher` is taking over
on the longer ones.

It is remarkable how the biggest improvements were obtained by algorithmic changes rather than
low-level optimisations. It is also worth noticing that the generic `Suffix` matcher was both neater
and faster than the specialised `LastByteSuffix` matcher.

Random data
-----------

Until now we've been working with natural texts with significant simplifications (lowercase, no
punctuation, no line breaks, the total alphabet size of 27). How will the results change if
our text and our pattern becomes totally random sequences of bytes?

We'll generate a text a bit bigger than before (4M) and use some position inside
as a source for our patterns. Since the data are random, we don't need to use very many patterns
to get an average value: we'll just use four.
We won't run all our algorithms now, just the most significant ones:

<table class="numeric">
<tr><th> Matcher </th><th>4</th><th>8</th><th>16</th><th>32</th><th>64</th><th>128</th><th>256</th></tr>
<tr><td class="ttext">Simple</td><td>  1.92</td><td>  1.92</td><td>  1.92</td><td>  1.92</td><td>  2.55</td><td>  2.55</td><td>  2.54</td></tr>
<tr><td class="ttext">FirstBytes</td><td>  0.46</td><td>  1.01</td><td>  1.01</td><td>  1.01</td><td>  1.01</td><td>  1.01</td><td>  1.01</td></tr>
<tr><td class="ttext">Hash</td><td>  2.38</td><td>  2.38</td><td>  2.36</td><td>  2.37</td><td>  2.36</td><td>  2.36</td><td>  2.36</td></tr>
<tr><td class="ttext">LastByte</td><td>  1.26</td><td>  0.64</td><td>  0.31</td><td>  0.16</td><td>  0.10</td><td>  0.07</td><td>  0.07</td></tr>
<tr><td class="ttext">NextByteSuffix</td><td>  1.20</td><td>  0.61</td><td>  0.31</td><td>  0.16</td><td>  0.10</td><td>  0.06</td><td>  0.06</td></tr>
<tr><td class="ttext">Regex2</td><td>  1.20</td><td>  0.60</td><td>  0.30</td><td>  0.16</td><td>  0.10</td><td>  0.06</td><td>  0.06</td></tr>
<tr><td class="ttext">Suffix</td><td>  0.52</td><td>  0.28</td><td>  0.16</td><td>  0.10</td><td>  0.08</td><td>  0.06</td><td>  0.04</td></tr>
<tr><td class="ttext">SmallSuffix</td><td>  0.63</td><td>  0.35</td><td>  0.20</td><td>  0.13</td><td>  0.11</td><td>  0.08</td><td>  0.06</td></tr>
<tr><td class="ttext">ChainSuffix</td><td>  0.72</td><td>  0.40</td><td>  0.23</td><td>  0.15</td><td>  0.11</td><td>  0.09</td><td>  0.07</td></tr>
<tr><td class="ttext">LightChainSuffix</td><td>  0.73</td><td>  0.40</td><td>  0.23</td><td>  0.15</td><td>  0.11</td><td>  0.09</td><td>  0.07</td></tr>
<tr><td class="ttext">LightestChainSuffix2</td><td>  0.64</td><td>  0.36</td><td>  0.23</td><td>  0.15</td><td>  0.11</td><td>  0.09</td><td>  0.07</td></tr>
</table>

Surprisingly, the top performing solutions haven't changed. `Suffix` is the best, although its
advantage over `LastByte` is small.

All simple solutions fall way behind, so better algorithms are beneficial for random data as well.

Let's try longer patterns:
<table class="numeric">
<tr><th> Matcher </th><th>512</th><th>1024</th><th>2048</th><th>4096</th><th>8192</th></tr>
<tr><td class="ttext">LastByte</td><td>  0.057</td><td>  0.051</td><td>  0.054</td><td>  0.054</td><td>  0.051</td></tr>
<tr><td class="ttext">Suffix</td><td>  0.025</td><td>  0.023</td><td>  0.030</td><td>  fail</td><td>  fail</td></tr>
<tr><td class="ttext">SmallSuffix</td><td>  0.025</td><td>  0.013</td><td>  0.010</td><td>  0.012</td><td>  0.018</td></tr>
<tr><td class="ttext">ChainSuffix</td><td>  0.025</td><td>  0.012</td><td>  0.008</td><td>  0.007</td><td>  0.010</td></tr>
<tr><td class="ttext">LightChainSuffix</td><td>  0.024</td><td>  0.010</td><td>  0.006</td><td>  0.005</td><td>  0.004</td></tr>
<tr><td class="ttext">LightestChainSuffix</td><td>  0.033</td><td>  0.020</td><td>  0.014</td><td>  0.009</td><td>  0.006</td></tr>

</table>

The `SuffixMatcher` failed on long patterns due to shortage of RAM, so it was good that we bothered
to implement more compact `SmallSuffix` and `ChainSuffix`. They started slowing down due to shortage
of L3 cache, and will eventually also fail at even longer patterns. However, on current patterns they
perform quite well. Here are the amounts of allocated memory, in megabytes:

<table class="numeric">
<tr><th> Matcher </th><th>512</th><th>1024</th><th>2048</th><th>4096</th><th>8192</th></tr>
<tr><td class="ttext">LastByte</td><td>  0</td><td>  0</td><td>  0</td><td>  0</td><td>  0</td></tr>
<tr><td class="ttext">Suffix</td><td>  139</td><td>  556</td><td>  2228</td><td>  fail</td><td>  fail</td></tr>
<tr><td class="ttext">SmallSuffix</td><td>  3.3</td><td>  12.8</td><td>  51</td><td>  202</td><td>  806</td></tr>
<tr><td class="ttext">ChainSuffix</td><td>  0.3</td><td>  0.8</td><td>  2.5</td><td>  9.0</td><td>  35</td></tr>
<tr><td class="ttext">LightChainSuffix</td><td>  0.2</td><td>  0.3</td><td>  0.4</td><td>  0.5</td><td>  1.1</td></tr>
<tr><td class="ttext">LightestChainSuffix</td><td>  0.2</td><td>  0.3</td><td>  0.4</td><td>  0.5</td><td>  1.1</td></tr>
</table>

<div id="BigRandom"></div>

Big random patterns
-------------------

How about even longer patterns? The `SmallSuffix` is already about to fail.
The chain suffix matchers look much better, but two of them suffer from the same problem.
Both `ChainBuffer` and `LightChainBuffer` first build a classic search tree, which takes a lot of
space. Here is the amount of memory this tree takes, in megabytes:

<table class="numeric">
<tr><th> Matcher </th><th>512</th><th>1024</th><th>2048</th><th>4096</th><th>8192</th></tr>
<tr><td class="ttext">ChainSuffix</td><td>  14</td><td>  54</td><td>  219</td><td>  872</td><td>  3489</td></tr>
</table>

This is where our effort to make `LightestChainSuffixMatcher` pays out: 
of the tree-based matchers, it is the only one that can survive doubling
of the pattern size. Obviously, the less efficient but less memory-hungry ones also can. Let's try to grow
our pattern to one megabyte. First of all, let's check if use of `LightestChain` is feasible at all.
We know it is building a big tree: how long does it take?

<table class="numeric">
<tr><th> Value </th><th>16K</th><th>32K</th><th>64K</th><th>128K</th><th>256K</th><th>512K</th><th>1M</th></tr>
<tr><td class="ttext">Time, ms</td><td>  2</td><td>  8</td><td>  10</td><td>  25</td><td>  60</td><td>  100</td><td>  310</td></tr>
</table>

The time of 310 ms isn't insignificant, but still appropriate for practical use. Now, let's check the
matching times:

<table class="numeric">
<tr><th> Matcher </th><th>16K</th><th>32K</th><th>64K</th><th>128K</th><th>256K</th><th>512K</th><th>1M</th></tr>
<tr><td class="ttext">LastByte</td><td>  0.053</td><td>  0.055</td><td>  0.063</td><td>  0.073</td><td>  0.095</td><td>  0.140</td><td>  0.227</td></tr>
<tr><td class="ttext">Regex2</td><td>  0.052</td><td>  0.052</td><td>  0.060</td><td>  0.067</td><td>  0.082</td><td>  0.116</td><td>  0.178</td></tr>
<tr><td class="ttext">LightestChain</td><td>  0.006</td><td>  0.010</td><td>  0.023</td><td>  0.046</td><td>  0.090</td><td>  0.177</td><td>  0.353</td></tr>
</table>

The results are somewhat disappointing: starting impressively at nearly 10 times the speed of
`Regex`, the `LightestChain` matcher eventually (at 256K bytes-long patterns) loses its advantage,
and it's easy to see why. Let's look at memory consumption of the matchers:

<table class="numeric">
<tr><th> Matcher </th><th>16K</th><th>32K</th><th>64K</th><th>128K</th><th>256K</th><th>512K</th><th>1M</th></tr>
<tr><td class="ttext">Regex2</td><td>  0.067</td><td>  0.132</td><td>  0.263</td><td>  0.525</td><td>  1.050</td><td>  2.098</td><td>  4.195</td></tr>
<tr><td class="ttext">LightestChain</td><td>  2.7</td><td>  7.7</td><td>  21</td><td>  46</td><td>  74</td><td>  95</td><td>  137</td></tr>
</table>

(all `LastByte` matchers use exactly 1064 bytes).

We ran out of L3 cache, which on our machine is only 15 MB per chip. Our text (4 MB) fits in cache,
even together with the data for simple matchers. That's why, even with higher number of iterations
they outperform the chain matcher.

<div id="HugeRandom"></div>

Big random patterns, huge data
------------------------------

Finally, let's run the ultimate test: we'll search for long patterns (16K to 1M) in a sixteen gigabyte
text. Technically this text is arranged as 16 arrays of one gigabyte each, one of which contains
the pattern. The search happens exactly once, to prevent caching.

<table class="numeric">
<tr><th> Matcher </th><th>16K</th><th>32K</th><th>64K</th><th>128K</th><th>256K</th><th>512K</th><th>1M</th></tr>
<tr><td class="ttext">LastByte</td><td>  0.37&nbsp;  </td><td>  0.34&nbsp;  </td><td>  0.35&nbsp;&nbsp;  </td><td>  0.34&nbsp;&nbsp;  </td><td>  0.35&nbsp;&nbsp;  </td><td>  0.35&nbsp;&nbsp;  </td><td>  0.35&nbsp;&nbsp;  </td></tr>
<tr><td class="ttext">Regex2  </td><td>  0.37&nbsp;  </td><td>  0.35&nbsp;  </td><td>  0.35&nbsp;&nbsp;  </td><td>  0.34&nbsp;&nbsp;  </td><td>  0.34&nbsp;&nbsp;  </td><td>  0.34&nbsp;&nbsp;  </td><td>  0.34&nbsp;&nbsp;  </td></tr>
<tr><td class="ttext">Lightest</td><td>  0.022</td><td>  0.016</td><td>  0.0086</td><td>  0.0045</td><td>  0.0023</td><td>  0.0011</td><td>  0.0009</td></tr>
</table>

It looks strange at first that the lightest chain suffix matcher is so much faster now than when the
text was cached. Most probably, it is due to much less success rate (previously the pattern was found once in
every four megabytes of scanned text), and it is a match that is slow.

The performance advantage of this matcher on the uncached data is impressive. When the pattern does not match
the current portion, we shift the pattern by the value that is very close to the length of the pattern,
avoiding a lot of uncached reads. Instrumenting the matcher code with a counter shows that matching
each gigabyte of the text involves reading about 3K bytes there. The total number of bytes read from
all 16 gigabytes is 2150497, of which 2100280 were read when processing the part that contained a match.

This makes it a good solution to find in files, too. So if one has a multi-terabyte file and needs
to find a one-megabyte fragment inside, this is the way to go. This, however, doesn't seem like
a realistic practical problem.


Obviously, the `LightestChainSuffixMatcher` has weak points, too. The tree for a one-megabyte pattern
occupies 137 Mbytes, so searching for much longer patterns requires radical space reduction.
The time to build a tree is small, but not completely insignificant: for the same tree it is 310 ms.
Some improvement may be needed there, too.

Conclusions
-----------

- We've implemented a version of `String.indexof` from scratch for byte arrays;

- Various optimisation techniques can make the implementation quite a bit faster (two to three times);

- However, the algorithmic improvements often outperform code optimisations;

- When working with strings, **Java** has a competitive advantage (access to the
String internals);

- However, it is still possible to perform better, at least in some cases (long patterns);

- **Java** regular expression library is very good and performs very well; however, we managed
to win over that library;

- I used the `LastByte` matcher in the production code; it offered the best "value for money"
(performance to complexity) ratio in my particular case. Other cases may put other matchers
in front; there is never a definite winner;

- The suffix-based family of matchers works very well on a wide range of pattern lengths, but
hits the memory limit on very long ones, where some additional work is required to save memory;

- We tested two major cases: data with small alphabet size and some repeating patterns (simplified
natural language), and completely random data. We didn't cover very exotic regular patterns.
Some other algorithms may perform better for these cases.

- Although generation of class files on the fly seems like the way to go in **Java**, it never
worked: the performance was consistently worse than that achieved by a normal **Java** code.

- Our starting point was searching for small to medium-sized patterns in not-so-big byte arrays.
In the process we accidentally came up with an algorithm suitable for searching for big patterns
in huge arrays or even in files, which works 300+ times faster than any alternatives.
There doesn't seem to be a demand for that, though.

Things to do
------------

I want to run similar tests in **C++**. It presents multiple challenges. On one hand, it also has
a pattern-matching library. On another one, it allows easy access to the CPU instructions, which
provide both fast byte search (`SCAS`) and wide comparisons (64 bit in normal mode and up to 512 in
SSE/AVX case).
