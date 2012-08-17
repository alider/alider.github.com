---
layout: post
title: "How fast is your CPU?"
description: ""
category: 
tags: []
---

According [the specification](http://ark.intel.com/products/47341/Intel-Core-i5-520M-Processor-(3M-Cache-2_40-GHz)) i5 CPU in my MacBookPro 2010 doesn't seem to be really fast in these days. It wasn't even the top model when i bought the laptop almost 2 years ago. Since that time new series of i5 and i7 have been released. Also, right now we have CPUs based on totally new architecture - [Sandy Bridge](http://software.intel.com/en-us/articles/sandy-bridge/). But still I'm convinced that the CPU I have is really fast, much faster what i had in past and - what can be surprising - faster then you may think when reading the spec. To prove that i will compare it to some of my old laptops:

- [MacBookPro 2010, i5 2.4GHz, 8GB RAM](http://www.everymac.com/systems/apple/macbook_pro/stats/macbook-pro-core-i5-2.4-aluminum-15-mid-2010-unibody-specs.html)
- [Sony VAIO 2009, Core 2 Duo 2.13GHz, 4GB RAM](http://www.sony.co.uk/product/vn-cw-series/vpccw1s1e-r)
- [Toshiba Satellite 2004, Pentium M 1.6GHz, 1GB RAM](http://eu.computers.toshiba-europe.com/innovation/jsp/SUPPORTSECTION/discontinuedProductPage.do?service=EU&PRODUCT_ID=97587)

From [the CPUs comparison](http://ark.intel.com/compare/36734,27584,47341) we can see that Core 2 Duo and i5 seems to be quite similar but Pentium M looks like something from the previous century. To test them i'm not going to use any available benchmarking tool but i will try to do something on my own. Of course this way won't provide anything sophisticated which would test all possible aspects and gave absolute results ready to publish and compare to others. But that's not my goal. In this post i will try to touch only one aspect of these CPUs - caches and memory access - and leave other stuff like threading for further topics.

**Tested on**

<table>
  <tbody>
    <tr><td>MacBookPro</td><td>OSX 10.6.8</td><td>64-bit</td><td>OpenJDK build 1.7.0-jdk7u4-b17-20120323</td></tr>
    <tr><td>Sony VAIO</td><td>Windows 7</td><td>64-bit</td><td>build 1.7.0_03-b05</td></tr>
    <tr><td>Toshiba Satellite</td><td>Ubuntu 11.10</td><td>32-bit</td><td>OpenJDK 1.7.0_147-icedtea (from apt repository)</td></tr>
  </tbody>
</table>

**The test**

    public class CPUTest {
      private static final int SIZE_X = 4 * 1024;
      private static final int SIZE_Y = 4 * 1024;
      private static final int ITERATIONS = 100;
      private static final int[][] array = new int[SIZE_X][SIZE_Y];

      public static void main(String[] args) throws Exception {
        // warm-up
        for (int x = 0; x &lt; SIZE_X; x++) {
          for (int y = 0; y &lt; SIZE_Y; y++) {
            array[x][y] = 1;
          }
        }

        final long startTime = System.nanoTime();

        for (int r = 0; r &lt; ITERATIONS; r++) {
          for (int x = 0; x &lt; SIZE_X; x++) {
            for (int y = 0; y &lt; SIZE_Y; y++) {
              array[x][y]++;
            }
          }
        }

        final long duration = System.nanoTime() - startTime;
        System.out.println(duration / 1000000 / ITERATIONS);
      }
    }

The test just iterates over quite big (64MB) array of ints and increments each of them. To get more accurate results we have one additional iteration at the begin called warm-up and whole measured code block is repeated 100 times.<br/>

My results:
<table>
  <tbody>
    <tr><td>1.</td><td>i5</td><td>15</td></tr>
    <tr><td>2.</td><td>Core 2 Duo</td><td>31</td></tr>
    <tr><td>3.</td><td>Pentium M</td><td>135</td></tr>
  </tbody>
</table>

The order itself isn't surprising but the numbers are quite interesting. i5 and Core 2 Duo look quite similar when reading the specifications but the test shows that there is significant improvement introduced in i5 in the way memory access and caches work in this test case. Pentium M is totally out of the competition here - 9 times slower the i5.

The next test is "almost" the same - only one change in the above code:
`array[x][y]++` to `array[y][x]++`

My results:

<table>
  <tbody>
    <tr><td>1.</td><td>Core 2 Duo</td><td>254</td></tr>
    <tr><td>2.</td><td>i5</td><td>330</td></tr>
    <tr><td>3.</td><td>Pentium M</td><td>400</td></tr>
  </tbody>
</table>

First of all, current results are much worse then previous. But what's really surprising here is the fact that newer i5 is slower the Core 2 Duo in this test. It's something what wasn't expected by me when doing the tests. To try to explain it somehow we need to get deeper into JVM, memory and CPU.

**What does exactly the test do?**

- it defines and allocates memory for 4096x4096 2-dimensional array of ints which gives 64MB
- it iterates over it and reads value from each element which gives over 16 millions of iterations and memory reads

**Java / JVM / Arrays / Memory**

When defining an array in the following way: `int[ ] array = new array[100]` we have continuous block of (100 + 3) * 4 bytes allocated in the memory on 32-bit JVM. Multi-dimensional arrays are a little different. Java/JVM implements them as "array of array references". When defining 2-dimensional array it's required to specify size only for the first dimension: `int[] [] array = new int[100][]` which btw will allocate the same number of bytes in the memory like the previous definition. That's because we defined the array for references to `int[]` class objects which have (on 32-bit JVM) the same size as int - 4 bytes.<br />

Anyway, in the test the array is defined with concrete sizes for both dimensions which means that memory for all arrays is allocated at once in the following way:

<img border="0" src="http://3.bp.blogspot.com/-424ytTIpIQc/T3nl597EFoI/AAAAAAAACOE/ZZLJP6RqsIU/s1600/2-dim-arrays.png" />

What's really important here is the fact that all arrays are allocated as continuous memory block.

**CPU overview**

Generally CPU can be described as a loop of the following steps:

- fetch a instruction and decode it
- fetch a memory data if needed
- execute instruction
- store the result back to memory

Seems to be a lot of work for single pass. Fortunately that's not how the CPUs work these days. First of all most of the steps above are made in parallel even in one single core - modern CPU has even 6+ execution units per core. The second important thing are caches. We have multi-level instruction and data caches which can help a lot if we allow them. When CPU needs some data first it looks in registers then caches L1, L2, L3 and then goes to the memory. Of course each next step in this checking has much higher access time. Because of that when some data is read from the memory will be stored in caches for some possible usage in future. And not only bytes which are needed right now but also many next bytes which are likely to be needed soon. This way in one memory read access CPU can get a lot of data and in case the pre-fetching was accurate it will save a lot of time/cycles. So that's why the first test version is so fast - it just accesses the memory in a linear way allowing caches to do it's best. It's just cache friendly. But the second version which makes a lot of jumps through the memory just fights against CPU and it's caches not only ignoring it's power but also wasting time needed for handling cache misses. Also the things getting even more complicated when considering threading but that's a case for it's own topic.

<img border="0" src="http://2.bp.blogspot.com/-FuIP_W7YF60/T3q4Dt5Dj4I/AAAAAAAACOc/aXQe7qxP2aE/s1600/cpu.png" />
CPU cache architecture overview

**Summary**

The most important observation from the tests is not the fact that caches help because it's quite obvious but the factor how much. All CPUs perform better when accessing memory in linear way but the numbers differ a lot:

- Pentium M ~ 3 times
- Core 2 Duo ~ 8 times
- i5 ~ 22 times !!!

The newer CPU the difference is bigger which means caches and pre-fetching logic getting more and more important when writing a code these days. Such small (stupid) change like the one in the test can lead to drastically different results.
