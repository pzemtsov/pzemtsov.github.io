Now is a good time to give a real reason behind introducing `Demux` interface and wrapping the `demux()` function into a class. What I try to achieve this
way is to avoid the inlining of this function into the measuring loop. We could have placed all the code from `measure ()` straight into `main ()` and called
the `demux ()` method directly (let's assume it is static and called `reference_demux ()`, and ignore `check ()` for now):

{% highlight java %}
    public static void main (String [] args) 
    {
        byte[] src = generate ();
        byte[][] dst = allocate_dst ();

        for (int loop = 0; loop < REPETITIONS; loop ++) {
            long t0 = System.currentTimeMillis ();
            for (int i = 0; i < ITERATIONS; i++) {
                reference_demux (src, dst);
            }
            long t = System.currentTimeMillis () - t0;
            System.out.println (t);
        }
    }
{% endhighlight %}

There is a chance that a compiler will detect that `reference_demux` is small and is only called once, and that it will put its code straight in place of its
call in `main ()`. Usually this is good as it improves the program execution speed. However, it is bad for performance measurement, because If we later replace
`reference_demux ()` with some other method, a compiler may make another decision and not inline it. The time measurements then will not be obtained in equal
conditions. Moreover, really sophisticated compilers may detect that all the iterations do the same thing and do not modify the input - and replace the
inner loop with just one call to `reference_demux ()`. Some compilers can even detect that the entire call has no visible side effect and remove all the code
altogether.

A virtual call of the de-multiplexing method makes such behaviour much less likely, although it still does not eliminate the possibility completely.
A compiler can detect that there is only one class that implements the `Demux` interface, or it can inline `measure ()` into `main ()` and then determine the
actual type of the parameter of `measure ()`, replace the virtual call with a direct call and then perform inlining. If we ever find that this is happening,
we must create even more complex structure for calling methods being tested. It wasn't necessary in this example.

