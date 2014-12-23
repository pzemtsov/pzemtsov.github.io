---
layout: post
title:  "More experiments with cache"
date:   2014-12-24 12:00:00
tags: C++ optimisation cache de-multiplexing
---

In the previous article ([{{ site.TITLE-E1-CACHE }}]({{ site.ART-E1-CACHE }})) we measured how de-multiplexing
speed is affected by input and output data location (in one of three levels of cache or not in any cache at all).
To do this, we processed more than one input buffer in a round-robin fashion and varied the total input data size.
When this size exceeds one of the cache sizes, the execution slows down.

We saw three steps of performance drop, which corresponded to three crossings of cache sizes. The increase
of execution time was roughly the same for all the algorithms, so relative slowdown was small for unoptimised versions
(25% for out-of-L3 execution) and high for optimised ones (3.7 times for the same). Performance out of L1 and L2 cache
was satisfactory, so we can concentrate on the L3 case.

The performance was affected by the total size of the input and the output, not just one of those. The reason is that
the caches are used for both reading and writing. In the reading case, the entire cache line (64 bytes on Sandy Bridge)
is read when any byte in it is read. In the writing case, the cache line is read when any portion of it is written;
this portion is modified and the entire line is marked as dirty. Subsequent write operations work on that dirty line.
Eventually (when the line has to be evicted or when the processor decides it's appropriate) the cache line is written
to memory or to the higher level of cache. When the entire cache line is to be modified (such as in our case) the
reading is unnecessary; but there is no way to tell the processor about it. It would help if the processor had 64-byte
memory access instructions, but it will only be available in AVX-512 instruction set. In the meantime, the processor
has to perform two memory accesses for each modified cache line -- read and write.

We expect writing to uncached memory to be more expensive than reading from such memory. Let's test it.

The test
--------

To test the effect of uncached reading, we will modify the cache testing program to read from sequential buffers and
write to the same memory over and over. To test the effect of uncached writing, we'll do other way around. Let's also
shorten the test. We'll test the fastest unoptimised source-first (`Src_First_1`) and destination first (`Dst_First_3a`)
version, the fastest non-SSE version (`Write8`), the fastest SSE (`Read8_Write16_SSE_Unroll`) and AVX (`Read8_Write32_AVX_Unroll`). We'll also add `Copy_AVX`,
just for interest sake.

I'll make another change. In each row I will print the first value as absolute time in milliseconds, and all the other
values as deltas relative to the first one. This way it'll be easier to see the impact of each cache miss.

I won't show the source, since it is straightforward, the entire text is in the
[repository]({{ site.REPO-E1-CACHE }}/blob/662ca08d602b0aaed3392640e9f6afb87a1d2037/e1-multi.cpp).

This is what we get when we run all the tests:

    # ./e1-multi
                                        :   4k   8k  16k  32k  64k 128k 256k 512k   1m   2m   4m   8m  16m  32m  64m 128m 256m 512m   1g   2g   4g
          11Src_First_1                 : 1077   12   28   97  159  170  187  191  190  192  192  192  215  353  356  354  354  353  353  352  353
     read-11Src_First_1                 : 1080  -13  -15  -16  -10  -10   -9   -7   -4   -5   -5   -4   -5   -3   31   31   32   32   32   32   32
    write-11Src_First_1                 : 1079  -14   25   60   70   74   90  112  113  108  113  112  113  143  244  249  250  250  250  250  250

          12Dst_First_3a                :  724    3    3   21   44   64   75   73   82   85   84   84  105  233  235  229  227  225  225  224  225
     read-12Dst_First_3a                :  723  -17  -20  -20  -21  -16  -17   -5   12   15   15   14   14   18  133  140  139  140  139  139  140
    write-12Dst_First_3a                :  723  -19  -20   -5   17   23   28   34   34   33   35   35   34   44   84   86   86   86   86   86   86
    
          6Write8                       :  550   -3    0    6   27   36   43   43   51   53   52   53   71  189  192  187  185  185  184  183  183
     read-6Write8                       :  547   -5   -4   -4    4   11   12   21   38   39   39   39   39   42  153  159  158  158  159  158  158
    write-6Write8                       :  545   -4   -6   -2    1    1    1   -1    0    0    0    0    0    4   17   18   18   18   19   18   19
    
          24Read8_Write16_SSE_Unroll    :  127    0    0    4   20   37   58   59   70   74   73   73  101  288  295  287  284  283  282  281  281
     read-24Read8_Write16_SSE_Unroll    :  127    0   -1    0    3    7    8   17   33   34   35   35   34   37  142  147  147  146  147  147  147
    write-24Read8_Write16_SSE_Unroll    :  128   -2   -2   -1   -2    1    6   15   15   13   14   15   15   40  160  171  173  174  174  174  175
    
          24Read8_Write32_AVX_Unroll    :  121   -2   -1    3   20   37   60   62   74   77   77   77  106  298  305  299  294  294  294  292  292
     read-24Read8_Write32_AVX_Unroll    :  120   -2   -2   -2    2    8    7   17   34   35   34   35   35   36  144  148  148  148  149  148  148
    write-24Read8_Write32_AVX_Unroll    :  121   -4   -4   -3    3    4   10   20   19   18   20   20   20   45  166  177  179  179  181  180  180
    
          8Copy_AVX                     :   49    0    3   30   57   72  101  107  115  118  118  118  143  311  318  311  307  305  305  304  303
     read-8Copy_AVX                     :   49    0   -1    1    8   17   17   25   35   36   35   36   35   36  113  118  117  117  118  117  118
    write-8Copy_AVX                     :   49   -1    1   12   30   37   43   66   66   63   66   66   67   93  200  209  210  211  210  211  211


(Remember, "write" remark here means "test for uncached write", and likewise for read).

As expected, both uncached read and uncached write slow down the execution. The position where it can be observed is
one step right from the position where original function slows down. For instance, `Read8_Write16_SSE_Unroll` demonstrates
performance drop at problem size 32m, while read-uncached and write-uncached versions of the same slow down at 64m,
this is what we expected.

Now let's concentrate on the biggest source array (the last number in each row). Let's collect these numbers in
a table:

<table class="numeric">
<tr><th>              Test                      </th><th> Read </th><th> Write </th><th> Read+Write</th><th> Both </th><th> Ratio </th></tr>
<tr><td class="label">Src_First_1               </td><td>   32 </td><td>   250 </td><td>       282 </td><td>  353 </td><td>  125% </td></tr>
<tr><td class="label">Dst_First_3a              </td><td>  140 </td><td>    86 </td><td>       226 </td><td>  225 </td><td>  100% </td></tr>
<tr><td class="label">Write8                    </td><td>  158 </td><td>    19 </td><td>       177 </td><td>  183 </td><td>  103% </td></tr>
<tr><td class="label">Read8_Write16_SSE_Unroll  </td><td>  147 </td><td>   175 </td><td>       322 </td><td>  281 </td><td>   87% </td></tr>
<tr><td class="label">Read8_Write32_AVX_Unroll  </td><td>  148 </td><td>   180 </td><td>       328 </td><td>  292 </td><td>   89% </td></tr>
<tr><td class="label">Copy_AVX                  </td><td>  118 </td><td>   211 </td><td>       329 </td><td>  303 </td><td>   92% </td></tr>            </tr>
</table>

Here `Read+Write` stands for the sum of effect of uncached reading (`Read`) and uncached writing (`Write`). `Both`
is the observed effect of both operations working with uncached memory. `Ratio` equals to `Both / (Read + Write)`

We can see that `Ratio` varied between `92%` and `125%`, which means that the effect of uncached reading and writing
is roughly equal to the sum of the effect of just uncached reading and that of uncached writing, It is also worth mentioning
that the ratio is below `100%` (roughly `90%`) for highly optimised versions. I'm not sure why this is so; in fact,
I expected the opposite.

The relative impact of uncached reading and uncached writing differs from test to test. As a rule, the effect of writing
is higher -- this is to be expected, since writing involves two accesses to memory, while reading only one.
However, higher impact of reading also happens - in both destination-first solutions, where we perform
scattered reading all over the place and write sequentially.

Two cases, `Src_First_1` and `Write8` are exceptional. The effect of uncached reading in `Src_First_1` and of uncached
writing in `Write8` is surprisingly small, almost zero. Such result for `Src_First_1` can be explained by the
hardware prefetch. We read sequential bytes from memory. Once the first one is loaded, the entire line is in the cache.
By the time we are finished with the line (and we have quite a bit of work to do, since we read one byte at a time),
the prefetch logic will have made some progress reading the next cache line (if not have read it completely).
Similar explanation probably works for `Write8` (although it's hard to explain why uncached penalty is so much
lower in `Write8` than in `Dst_First_2a`).

It looks attractive to make a broad statement that source-first solutions are more appropriate when the source data
isn't in cache, while the destination-first is better with uncached destination. This is, however, not true, because
source-first is slower in the absolute sense (`1077 ms` vs `724 ms` in the fully cached case, or `1110 ms` vs `809 ms`
in the cases in question). And, of course, the highly optimised versions are much faster than both (`408 ms` in the worst case).

Negative numbers
----------------

One interesting observation about the output of the test is that it contains negative numbers.
You can see them everywhere, but they are most visible in `Src_First_1` and `Dst_First_3a` cases.
In the `read-` and `write-` tests (but not in the original test!) it is faster to process two different buffers
than the same buffer twice, and the same with four, eight and more buffers - as long as the data fits into cache.
The cache where it must fit differs. The `write-Dst_First_3a` test shows this effect while data fits into L1 cache,
the `read-Dst_First_3a` extends it to the L2, and `read-Src_First_1` shows it while in the L3.

Since the effect is only visible on cached data, it falls outside of today's topic. However, it's worthwhile taking
a note and investigating it later.

Sequential _vs_ random processing
-------------------------------

In our tests the input data is read and the output data is written mostly sequentially. The individual bytes are not,
but the cache lines are. This happens within each input block, and also across blocks, since the input blocks lie
in memory sequentially. How much does such sequential access contribute to relatively high out-of-cache performance?
Let's change the access pattern. We can't change the pattern within each block (that would require complete rework
of the code), but we can process the blocks in random order. We'll just rewrite the inner loop of `measure()`:

{% highlight C++ %}
    for (unsigned j = 0; j < count; j++) {
        unsigned p = rand() & (count - 1);
        demux.demux(src + SRC_SIZE * p, SRC_SIZE, dst + NUM_TIMESLOTS * p);
    }
{% endhighlight %}

The text is [here]({{ site.REPO-E1-CACHE }}/blob/eb4190c90138c546297e18a72f273a1c4e30ac28/e1-multi.cpp).

Here are the results:

                                        :   4k   8k  16k  32k  64k 128k 256k 512k   1m   2m   4m   8m  16m  32m  64m 128m 256m 512m   1g   2g   4g
          11Src_First_1                 : 1079   33   69   95  164  171  187  198  200  201  203  202  220  367  371  367  370  365  368  368  367
    rand  11Src_First_1                 : 1126   -2   20   66  118  125  132  136  138  143  152  154  164  315  392  428  448  460  465  471  478
    
          12Dst_First_3a                :  726    0    2   23   43   45   74   82   83   86   87   85  101  235  234  236  236  230  227  226  224
    rand  12Dst_First_3a                :  737    3    8   14   32   40   62   83   95  102  112  108  114  238  313  355  379  393  401  408  423
    
          6Write8                       :  550    0    1    9   27   26   45   53   54   55   56   55   68  191  189  191  190  186  184  183  182
    rand  6Write8                       :  561    1    2    8   21   27   41   56   66   73   79   78   82  189  267  301  316  325  328  332  342
    
          24Read8_Write16_SSE_Unroll    :  128   -1   -1    4   18   19   44   66   72   72   73   72   95  291  291  294  292  286  281  279  278
    rand  24Read8_Write16_SSE_Unroll    :  136    1    0    1   11   15   34   53   68   77   85   84   94  278  372  433  463  462  460  461  470
    
          24Read8_Write32_AVX_Unroll    :  119    1    0    6   20   22   47   74   79   80   82   81  103  305  308  311  308  303  298  297  295
    rand  24Read8_Write32_AVX_Unroll    :  128    0    1    1   12   17   34   55   71   81   89   90   98  288  390  454  483  485  483  484  492
    
          8Copy_AVX                     :   47    0    7   34   56   58   76  114  119  121  124  121  140  317  318  320  319  311  307  305  304
    rand  8Copy_AVX                     :   54    1    6   18   39   49   68   91  107  117  127  126  133  290  375  441  484  496  498  503  515

We see that, except for the first test, the initial timings differ by about `10 ms`. This is a cost of calling `rand()`.
The easiest way to compare other numbers is by plotting graphs. I'll only plot two of them (the others are similar):

<img src="{{ site.url }}/images/e1-more-cache.png" width="600" height="315">
<img src="{{ site.url }}/images/e1-more-cache-2.png" width="600" height="324">

The random line stays very close to the sequential line while the problem
size is below the L3 cache size (32m), and in this area it looks a bit smoother (the time grows steadily and not
in steps). The interesting behaviour starts after the L3 point. The sequential line is horizontal, which is expected --
all the memory accesses are uncached, no matter if the working set size is 64m or 1g. In fact, the sequential
line even goes down a bit -- I can't explain this at this point. The random graph behaves differently;
it grows slowly along nice smooth curve, eventually becoming horizontal as well.

This looks like a puzzle, but the explanation is simple. The random order does not provide that each input block is
processed exactly once. Even if the original sequence of `rand()` values does not repeat itself for a long while,
this isn't true for `rand() & (count - 1)`. This means that, unlike the sequential test, the random test does not guarantee
100% cache misses. If the random number generator is good, the cache hit ratio will be equal to the ratio
of the cache size and the working set size. This explains the continuous increase in time after the L3 point and the
smooth shape of the graph before that.

The effect diminishes as the working set grows and becomes negligible after 1 GB point. The graph becomes nearly
horizontal. The distance between the two graphs is the penalty we pay for non-sequential memory access. The penalty
is quite high: it is about `200 ms` on top of `300 ms` already added for cache misses. This is significant, since the base
(cached) time for the fastest version is `119 ms`. It looks like, due to automatic prefetch or other reasons,
sequential processing of input data is really benefitial.

One other reason I can think of for sequential processing to be faster than random is the effect of the TLB cache.
I'll study it later.

Conclusions
-----------

- Both reading and writing benefit from the data residing in cache. If placing both input and output data in cache
isn't an option, it helps to put one of the two there.

- Keeping some data in the smaller (and faster) cache leaves more of the bigger cache for other data. For instance,
in our case writing all the outputs to the same place allows keeping 20 Mbytes of the input data in cache, while
normally we would be able to keep only 10 Mbytes.

- Usually the uncached writing is more expensive, but it depends very much on the algorithm used.

- The effects of uncached reading and uncached writing roughly add together when the entire memory access is uncached.

- The sequential access to memory (even when uncached) is faster than the random access. In the highly optimised
versions the random order added 50% to the execution time.

- Some unexpected dependencies between the data size and the execution speed were observed. In particular,
processing two input buffers (both in L1 cache) can be faster than processing one input buffer twice. The difference isn't big
(about `20 ms` with the total execution time of `700 ms`, or 2% -- 3%), but it still looks strange.