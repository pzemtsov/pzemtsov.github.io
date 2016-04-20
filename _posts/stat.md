---
layout: post
title:  "Stats"
date:   2015-04-24 12:00:00
tags: Java optimisation
story: life
story-title: "Stats"
---


Unlike many other storied in my notes, this one originated from real life.

The statistics comes in two forms: counters and gauges.

A counter is a variable, that can be incremented by a known value (most commonly, one). 

Heare are the examples:
- a number of operations executed
- a number of bytes processed
- a number of packets received












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
