---
layout: post
title:  "Abnormal string hashing"
date:   2019-10-07 12:00:00
tags: Java optimisation hashmap
---

Today we'll have a little **Java** quiz. Most readers probably know the answer.

The question
-------------

Let's create a hash map in **Java** and fill it with some big number of strings:

{% highlight Java %}
import java.util.HashMap;

public class HashTest
{
    static HashMap<String, Integer> map = new HashMap<> ();
    
    static void build (int N)
    {
        for (int i = 0; i < N; i++) {
            map.put ("String#" + i, i);
        }
    }
}
{% endhighlight %}

We'll search various strings in this table and report time:

{% highlight Java %}
    static void test (String str)
    {
        int N = 10000000;
        int sum = 0;
        
        long t1 = System.currentTimeMillis ();
        for (int i = 0; i < N; i++) {
            Integer n = map.get (str);
            if (n != null) sum += n;
        }
        long t2 = System.currentTimeMillis ();
        
        System.out.println (str + ": time " + (t2-t1) + "; sum " + sum);
    }
{% endhighlight %}

Let's run it for some arbitrary strings:

{% highlight Java %}
    public static void main (String [] args)
    {
        build (1000000);

        test ("mouse");
        test ("String#532");
        test ("a quick brown fox jumps over the lazy dog");
        test ("aardvark polycyclic bitmap");
        test ("public static void main (String [] args)");
    }
{% endhighlight %}

Here is the output:

    mouse: time 17; sum 0
    String#532: time 93; sum 1025032704
    a quick brown fox jumps over the lazy dog: time 23; sum 0
    aardvark polycyclic bitmap: time 196; sum 0
    public static void main (String [] args): time 24; sum 0

Most of the times are very short. One (for `"String#532"`) is slightly longer than the others due
to the fact that this string is in fact present in the map. One, however (for the `"aardvark polycyclic bitmap"`)
is exceptionally long - eight times longer that the times for similar-sized strings.

What makes this aardvark string so slow, and are there other such strings?

The answer
----------

First of all, other such strings definitely do exists, for instance, `"aaron bends nonconformity"`
or `"maximal java reconstructs"`.

When searching objects in a hash map, the first thing that happens is that their hash code is calculated:

{% highlight Java %}
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
{% endhighlight %}

The `String`'s hash calculation looks like this:

{% highlight Java %}
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;

        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}
{% endhighlight %}

The hash is calculated by adding all the characters multiplied by various powers of 31. To avoid
performing this calculation every time the hash is needed, and also to avoid performing it
when it is not needed at all, the value is calculated in a lazy way and cached. It is calculated the
first time `hashCode()` is called and stored in the `hash` field.

This scheme, however, fails when the hash value is 0. This value will always present itself as
uncalculated, causing the hash code to be re-calculated each time. And this is exactly what our
strings are: the strings with the hash code of zero.

The background
--------------

**Java**'s designers could have provided an extra flag to indicate the hash value being present,
or used the highest bit of the hash value for that, or, perhaps, made the hash one if it was calculated as
zero. They, probably, considered all of this unnecessary, as the probability is very low: one
out of four billion strings.

Most of these strings look very artificial and very few make grammatical sense.
Still, meaningful ones occur, too. I started this study after reading a Russian article on the issue, where they
came up with a very beautiful example: `"лжеотожлествление электровиолончели"`
(false identification of an electric cello). So I took a list of 58K English words and tried combining them.

No single English word produced zero hash code (perhaps, some super-long hyphenated ones could, but
I didn't try them), and no two words with a space in between could do it either.

Three words, however, happened to be very productive, producing 46K strings. The shortest ones
are 13 characters long (`"civic sear ha"`, `"pleas so fold"`, `"venal us burs"`); the longest one
is 48 (`"superconducting instrumentals intercommunication"`).

Here are some that make some sense:

- `covenant selfdestructs nondrinkers`
- `cannabis auctioneers uncorrupted`
- `america envisages inflate`
- `colourful Obama aftereffect`
- `reduce whores sinfulness`
- `shivery rhinoceroses chaser`
- `ostensible communists speech`
- `flowers vendors conversation`

And here is the best one:

- `antipathies included computing`

The [full list of found phrases]({{ site.REPO-ABNORMAL-STRINGS }}/blob/master/output.txt),
together with [the program]({{ site.REPO-ABNORMAL-STRINGS }}/blob/master/Generate.java) that calculated it,
is in the [repository]({{ site.REPO-ABNORMAL-STRINGS }}). The program ran for 13 minutes -- perhaps, someone can speed it up enough
to search four and more words?

Conclusion
----------

Hash map is a very powerful tool to achieve good performance. Still, abnormal cases of unusually
bad performance do occur, so be careful.
