Then we need a correctness check method:

{% highlight java %}
    static void check (Demux demux)
    {
        byte[] src = generate ();
        byte[][] dst0 = allocate_dst ();
        byte[][] dst = allocate_dst ();
        new Reference ().demux (src, dst0);
        demux.demux (src, dst);
        for (int i = 0; i < NUM_TIMESLOTS; i++) {
            if (! Arrays.equals (dst0[i], dst[i])) {
                throw new java.lang.RuntimeException ("Results not equal");
            }
        }
    }
{% endhighlight %}

This method generates input data and feeds it to both reference solution and the solution in question. Then it checks that the results are identical.
Obviously, there isn't much sense in calling it with `Reference`, because it will compare results of two calls to the same method (although for more
complex algorithms it could have been a useful test - identical results for identical inputs isn't something that can always be taken for granted).
However, very soon I'll create other implementations, and the `check()` method will be used to test them.
