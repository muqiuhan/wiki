#jvm #gc

> https://mill-build.org/blog/6-garbage-collector-perf.html

## TL;DR

从理论上来说，GC的性能主要集中在两个方面，一是程序用于收集垃圾的时间百分比，而不是实际工作，这个值越低越好。二是程序在收集垃圾时完全暂停的最长时间，这个值也是越低越好。

分开这两个指标的原因是：

- 有些程序只关心自身的运行，例如，如果一个程序只关心完成大批量分析需要多长时间，而不关心 GC 是否会导致它中途暂停。
- 其他程序只关心暂停时间，例如，玩电子游戏的人并不关心它是否能跑得比他们的眼睛能感知的更快，但他们关心它不会在爽玩时暂停明显的时间。

从这些有限描述中可以对简单垃圾回收器的性能做出一些理论上的推断：
- **暂停时间应与 live-set 的大小成正比**。这是因为集合涉及跟踪、复制和更新 live-set 中的引用。
- 暂停时间不取决于要收集的垃圾量。收集器根本没有花时间查看或扫描垃圾对象，它们所在的堆在垃圾收集后会被直接擦除。
- 集合之间的间隔与可用内存成反比。只需要在分配的垃圾填满了程序在存储 live-set 所需的 “额外” 堆内存时运行垃圾收集。
- **GC 开销是暂停时间除以间隔，或与额外内存成正比，与实时集大小和堆大小成反比**

换言之：

- `allocation_cost = O(1)`
- `gc_pause_time = O(live-set)`
- `gc_interval = O(heap-size - live-set)`
- `gc_overhead = gc_pause_time / gc_interval`
- `gc_overhead = O(live-set / (heap-size - live-set))`

从这个结论中，可以看到一些不直观的结果：

- **更多的内存不会减少暂停时间**: `gc_pause_time = O（live-set）`，因此暂停时间不取决于你有多少`堆大小`。
- **提供更多内存不会改善 GC 开销**:  `gc_overhead = O(live-set / (heap-size - live-set))` ，因此提供越大的堆大小才能拥有更少的 GC 开销（将更大百分比的程序时间花在有用的工作上）。
- **相反，提供与程序完全相同的内存是最坏的情况！** `gc_overhead = O(live-set / (heap-size - live-set))` 什么时候 `heap-size = live-set` 表示 `gc_interval = 0` 且 `gc_overhead = 无穷大`：程序将不断需要运行开销极大的收集器，并且没有时间执行实际工作。因此，垃圾回收器需要额外的内存才能使用，此外，还需要分配程序中的所有对象所需的内存。

---

Garbage collectors are a core part of many programming languages. While they generally work well, on occasion when they go wrong they can fail in very unintuitive ways. This article will discuss the fundamental design of how garbage collectors work, and tie it to real benchmarks of how GCs perform on the Java Virtual Machine. You should come away with a deeper understanding of how the JVM garbage collector works and concrete ways you can work to improve its performance in your own real-world projects.

## A Theoretical Garbage Collector

To understand how real-world JVM garbage collectors works, it is best to start by looking at a simple example garbage collector. This will both give an intuition for how things work in general, and also help you notice when things diverge from this idealized example.

### Process Memory
At its core, a garbage collector helps manage the free memory of a program, often called the _heap_. The memory of a program can be modelled as a linear sequence of storage locations, e.g. below where we have 16 slots in memory:

![[Pasted image 20250111190234.png]]
These storage locations can contain objects (below named `foo`, `bar`, `qux`, `baz`) that take up memory and may reference other objects (solid arrows). Furthermore, the values may be referenced from outside the heap (dashed lines), e.g. from the "stack" which represents local variables in methods that are currently being run (shown below) or from static global variables (not shown). We keep a `free-memory` pointer to the first empty slot on the right.

If we want to allocate a new object `new1`, we can simply put it at the location of the `free-memory` pointer (green below), and bump the pointer 1 slot to the right:

![[Pasted image 20250111190404.png]]
Similarly, objects may stop being referenced, e.g. `bar` below no longer has a reference pointing at it from the stack. This may happen because a local variable on the stack is set to `null`, or because a method call returned and the local variables associated with it are no longer necessary:
![[Pasted image 20250111190441.png]]
For the purposes of this example, we show all objects on the heap taking up 1 slot, but in real programs the size of each object may vary depending on the fields it has or if it’s a variable-length array.

### A Simple Garbage Collector

The simplest kind of garbage collector splits the 16-slot heap we saw earlier into two 8-slot halves. If we want to allocate 4 more objects (`new2`, to `new5`), but there are only 3 slots left in that half of the heap, we will need to do a collection:
![[Pasted image 20250111190820.png]]
To do a collection, the GC first starts from all non-heap references (e.g. the `STACK` references above) often called "GC roots". It then traces the graph of references, highlighted red below:
![[Pasted image 20250111190859.png]]
Here, we can see that `foo` is not referenced ("garbage"), `qux` and `new1` are referenced directly from the `STACK`, and `baz` is referenced indirectly from `qux`. `bar` is referenced by `foo`, but because `foo` is itself garbage we can count `bar` as garbage as well.

We then copy all objects we traced (often called the _live-set_) from `HALF1` to `HALF2`, adjust all the references appropriately. Now `HALF2` is the half of the heap in use, and `HALF1` can be reset to empty:

![[Pasted image 20250111191028.png]]
This collection has freed up 5 slots, so we now have space to allocate the 4 `new2` to `new5` objects we wanted (green) starting from our `free-memory` pointer:
![[Pasted image 20250111191058.png]]

You may notice that the objects `foo` and `bar` disappeared. This is because `foo` and `bar` were not referenced directly or indirectly by any GC roots: they were unreachable, and thus considered "garbage". These garbage objects were not explicitly deleted, but simply did not get copied over from `HALF1` to `HALF2` during collection, and thus were wiped out when `HALF1` was cleared.

As your program executes, the methods actively running may change, and thus the references (both from stack to heap and between entries on your heap) may change. For example, we may stop referencing `qux`, which also means that `baz` is now unreachable:

![[Pasted image 20250111191142.png]]

Although `qux` and `baz` are now "garbage", they still take up space in the heap. Thus, if we want to allocate two new objects (e.g. `new6` and `new7`), and there is only one slot left on the heap (above), we need to repeat the garbage collection process: tracing the objects transitively reachable (`new1`, `new2`, `new3`, `new4`, `new5`), copying them from `HALF2` to `HALF1`, adjusting any references to now use `HALF1` as the new heap, and clearing anything that was left behind in `HALF2`. This then gives us enough space to allocate `new6` and `new7` (below in green):

![[Pasted image 20250111191255.png]]

This process can repeat as many times as necessary: as long as there are _some_ objects that are unreachable, you can run a collection and copy the "live" objects to the other half of the heap, freeing up some space to allocate new objects. The only reason this may fail is that if you run a collection and there _still_ isn’t enough space to allocate the objects you want; that means your program has run out of memory, and will fail with an `OutOfMemoryError` or similar.

Even this simplistic GC has a lot of interesting properties, and you may have heard these terms or labels that can apply to it:

- **semi-space** garbage collector, because of the way it splits the heap into two halves
- **copying** garbage collector, because it needs to copy the heap objects back and forth between `HALF1` and `HALF2`
- **tracing** garbage collector, because of the way it traverses the graph of heap references in order to decide what to copy.
- **stop the world** garbage collector, because while this whole trace-copy-update-references workflow is happening, we have to stop the program to avoid race conditions between the garbage collector and the program code.
- **compacting** garbage collector, because every time we run a GC, we copy everything to the left-most memory, avoiding the memory fragmentation that occurs with other memory management techniques such as [Reference Counting](https://en.wikipedia.org/wiki/Reference_counting).

Most modern GCs are considerably more complicated than this: e.g. they may have optimizations to avoid wasting half the heap by leaving it empty, or they may have *optimizations for handling short-lived objects*, but at their heart this is still what they do. And understanding the performance characteristics of this simple, naive GC can help give you an intuition in how GCs compare to other memory management strategies, and how modern GCs behave in terms of performance.

## Compared to Reference Counting

[Reference Counting](https://en.wikipedia.org/wiki/Reference_counting) is another popular memory management strategy that Garbage Collection is often compared to. Reference counting works by keeping track of how many incoming references each object has, and when that number reaches zero the object can be collected. This approach has a few major differences from that of a tracing GC. We discuss a few of them below:

### Reference counting does not compact the heap

Program that use reference counting tend to find their heap getting more and more fragmented over time We can see this in the heap diagrams: the tracing garbage collector heaps above always had a single block of empty space to the right, and had the `new` objects allocated in ascending order from left-to-right:
![[Pasted image 20250111191658.png]]
In contrast, reference counted heaps (e.g. below) tend to get fragmented, with free space scattered about, and the allocated objects jumbled up in no particular order:

![[Pasted image 20250111191743.png]]

There are two main ways this affect performance:
- With garbage collection all the free memory is always on the right in one contiguous block, so an allocation just involves putting the object at the `free-pointer` location and moving `free-pointer` one slot to the right. Furthermore, newly allocated objects (which tend to be used together) are placed next to each other, making them more cache-friendly and improving access performance.
- With reference counting objects are usually freed in-place, meaning that the free space is scattered throughout the heap, and you may need to scan the entire heap from left-to-right in order to find a spot to allocate something. There are data structures and algorithms that can make allocation faster than a linear scan, but they will never be as fast as the single pointer lookup necessary with a GC.

### Reference counting cannot collect cycles
Objects that reference each other cyclically can thus cause memory leaks when their objects never get collected, resulting in the program running out of memory even though much of the heap could be cleaned up by a tracing garbage collector.

For example, consider the following heap, identical to the one we started with, but with an additional edge from `bar` to `foo` (green), and with the edge from the stack to `bar` removed:

![[Pasted image 20250111191936.png]]
With reference counting, even though `foo` and `bar` cannot be reached by any external reference - they are "garbage" - each one still has a reference pointing at it from the other. Thus they will never get collected.

But with a tracing garbage collector, a collection can traverse the reference graph (red), and copy `qux` and `baz` to the other half of the heap, leaving `foo` and `bar` behind as garbage, despite the reference cycle between them.

_Garbage Collection_ and _Reference Counting_ have very different characteristics, and neither is strictly superior to the other in all scenarios. Many programming languages (e.g. Python) that use reference counting also have a backup tracing garbage collector that runs once in a while to clean up unreachable reference cycles and compact the heap, and most modern GCs (e.g. ZGC discussed below) use some reference-counting techniques as part of their implementation.

## Theoretical GC Performance

Typically, GC performance focuses on two main aspects:

- **Overhead**: what % of the time your program is spent collecting garbage, rather than real work. Lower is better.
- **Pause Times**: what is the longest time your program is completely paused while collecting garbage. Lower is better.

These two metrics are separate:

- **Some programs only care about throughput**, e.g. if you only care about how long a big batch analysis takes to complete, and don’t care if it pauses in the middle to GC: you just want it to finish as soon as possible.
- **Other programs only care about pause times**, e.g. someone playing a videogame doesn’t care if it can run faster than their eye can perceive, but they do care that it does not freeze or pause for noticeable amounts of time while you are playing it.

Even from the limited description above, we can already make some interesting inferences about how the performance of a simple garbage collector will be like.

1. Allocations in garbage collectors are cheap: when the heap is not yet full, we can just allocate things on the first empty slots on the right side of the heap and bump free-pointer, without having to scan the heap to find empty slots.
2. **Pause times should be proportional to the size of the live-set**. That is because a collection involves tracing, copying, then updating the references within the live-set.
3. **Pause times would _not_ depend on the amount of garbage to be collected**. The collection we looked at above spend no time at all looking at or scanning for garbage objects, they simply all disappeared when their half of the heap was wiped out following a collection.
4. **Interval between collections is inversely proportional to free memory**. We only need to run a collection when the garbage we allocate fills up the "extra" heap memory our program has on top of what is necessary to store the live-set.
5. GC overhead is the pause time divided by the interval, or proportional to the extra memory and inversely proportional to the live-set size and heap size.

In other words:

- `allocation_cost = O(1)`
- `gc_pause_time = O(live-set)`
- `gc_interval = O(heap-size - live-set)`
- `gc_overhead = gc_pause_time / gc_interval`
- `gc_overhead = O(live-set / (heap-size - live-set))`

Even from this small conclusions, we can already see some unintuitive results:

- **More memory does _not_ reduce pause times!** `gc_pause_time = O(live-set)`, and so pause times do not depend on how much `heap-size` you have.
- **There is no point at which providing more memory does not improve GC overhead!** `gc_overhead = O(live-set / (heap-size - live-set))`, so providing larger and larger `heap-size`s means less and less GC overhead, meaning a larger % of your program time is spent on useful work.
- **Conversely, providing exactly as much memory as the program is the worst case possible!** `gc_overhead = O(live-set / (heap-size - live-set))` when `heap-size = live-set` means `gc_interval = 0` and `gc_overhead = infinity`: the program will constantly need to run an expensive collections and have no time left to do actual work. Garbage collectors therefore _need_ excess memory to work with, on top of the memory you would expect to need to allocate all the objects in your program.

Even from this theoretical analysis, we have already found a number of surprising results in how GCs perform over time. Let’s now see how this applies to some real-world garbage collectors included with the Java Virtual Machine.

## Benchmarking JVM Garbage Collectors

Now that we have run through a theoretical introduction and analysis of how GCs work and how we would expect them to perform, let’s look at some small Java programs and monitor how garbage collection happens when using them. For this benchmark, we’ll be using the following Java program:

```java
public class GC {
    public static void main(String[] args) throws Exception{
        final long liveSetByteSize = Integer.parseInt(args[0]) * 1000000L;
        final int benchMillis = Integer.parseInt(args[1]);
        final int benchCount = Integer.parseInt(args[2]);
        // 0-490 array entries per object, * 4-bytes per entry,
        // + 20 byte array header = average 1000 bytes per entry
        final int maxObjectSize = 490;
        final int averageObjectSize = (maxObjectSize / 2) * 4 + 20;

        final int liveSetSize = (int)(liveSetByteSize / averageObjectSize);

        long maxPauseTotal = 0;
        long throughputTotal = 0;

        for(int i = 0; i < benchCount + 1; i++) {
            int chunkSize = 256;
            Object[] liveSet = new Object[liveSetSize];
            for(int j = 0; j < liveSetSize; j++) liveSet[j] = new int[j % maxObjectSize];
            System.gc();
            long maxPause = 0;
            long startTime = System.currentTimeMillis();

            long loopCount = 0;
            java.util.Random random = new java.util.Random(1337);
            int liveSetIndex = 0;

            while (startTime + benchMillis > System.currentTimeMillis()) {
                if (loopCount % liveSetSize == 0) Thread.sleep(1);
                long loopStartTime = System.currentTimeMillis();
                liveSetIndex = random.nextInt(liveSetSize);
                liveSet[liveSetIndex] = new int[liveSetIndex % maxObjectSize];
                long loopTime = System.currentTimeMillis() - loopStartTime;
                if (loopTime > maxPause) maxPause = loopTime;
                loopCount++;
            }
            if (i != 0) {
                long benchEndTime = System.currentTimeMillis();
                long bytesPerLoop = maxObjectSize / 2 * 4 + 20;
                throughputTotal += (long) (1.0 * loopCount * bytesPerLoop / 1000000 / (benchEndTime - startTime) * averageObjectSize);
                maxPauseTotal += maxPause;
            }

            System.out.println(liveSet[random.nextInt(liveSet.length)]);
        }

        long maxPause = maxPauseTotal / benchCount;
        long throughput = throughputTotal / benchCount;

        System.out.println("longest-gc: " + maxPause + " ms, throughput: " + throughput + " mb/s");
    }
}
```

This is a small Java program designed to do a rough benchmark of Java garbage collection performance. For each benchmark, it:

- Starts off allocating a bunch of `int[]` arrays of varying size in liveSet, on average taking up 1000 bytes each.
- Loops continuously to allocate more `int[]`s and over-writes the references to older ones.
- Tracks how long each allocation takes to run: ideally it should be almost instant, but if that allocation triggers a GC it may take some time.

Lastly, we print out the two numbers we care about in a GC: the `maxPause` time in milliseconds, and the `throughput` it is able to handle in megabytes per second (`throughput` being the opposite of `overhead` we mentioned earlier).

To be clear, this benchmark is _rough_. Performance will vary between runs, and on what hardware and software you run it (I ran it on a M1 Macbook Pro running Java 23). But the results should be clear even if the exact numbers will differ between runs.

You can run this program via:

```
> java -Xmx1g GC.java 800 10000 5 # Default is -XX:+UseG1GC
> java -Xmx1g -XX:+UseParallelGC GC.java 800 10000 5
> java -Xmx1g -XX:+UseZGC GC.java 800 10000 5
```

Above, `-Xmx1g` sets the heap size, the -XX: flags set the garbage collector, 800 sets the liveSet size (in megabytes), and 10000 and 5 set the duration and number of iterations to run the benchmark (here 10 seconds, 5 iterations). The measured pause times and allocation rate are averaged over those 5 iterations.

I used the following Java program to run the benchmark for a range of inputs to collect the numbers shown below:

```java
package mill.main.client;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

public class GCBenchmark {
    public static void main(String[] args) throws Exception {
        String[][] javaGcCombinations = { {"23", "G1"}, {"23", "Z"} };

        for (String[] combination : javaGcCombinations) {
            String javaVersion = combination[0];
            String gc = combination[1];

            System.out.println("Benchmarking javaVersion=" + javaVersion + " gc=" + gc);

            int[] liveSets = {400, 800, 1600, 3200, 6400};
            int[] heapSizes = {800, 1600, 3200, 6400, 12800};

            List<List<String[]>> lists = new ArrayList<>();

            for (int liveSet : liveSets) {
                List<String[]> innerList = new ArrayList<>();

                for (int heapSize : heapSizes) {
                    if (liveSet >= heapSize) innerList.add(new String[]{"", ""});
                    else innerList.add(runBench(liveSet, heapSize, javaVersion, gc));
                }

                lists.add(innerList);
            }

            renderTable(liveSets, heapSizes, lists, 0);
            renderTable(liveSets, heapSizes, lists, 1);
        }
    }

    static String[] runBench(int liveSet, int heapSize, String javaVersion, String gc) throws Exception {
        System.out.println("Benchmarking liveSet=" + liveSet + " heapSize=" + heapSize);

        String javaBin = "/Users/lihaoyi/Downloads/amazon-corretto-" + javaVersion + ".jdk/Contents/Home/bin/java";

        ProcessBuilder processBuilder = new ProcessBuilder(
            javaBin, "-Xmx" + heapSize + "m", "-XX:+Use" + gc + "GC", "GC.java", "" + liveSet, "10000", "5"
        );

        Process process = processBuilder.start();
        process.waitFor();


        List<String> outputLines =
            new String(process.getInputStream().readAllBytes()).lines().toList();

        Optional<String[]> result = outputLines.stream()
            .filter(line -> line.startsWith("longest-gc: "))
            .map(line -> {
                String[] parts = line.split(", throughput: ");
                return new String[]{
                    parts[0].split(": ")[1].trim(),
                    parts[1].trim()
                };
            })
            .findFirst();

        return result.orElse(new String[]{"error", "error"});
    }

    static void renderTable(int[] liveSets, int[] heapSizes, List<List<String[]>> lists, int columnIndex) {
        StringBuilder header = new StringBuilder("| live-set\\heap-size | ");
        for (int heapSize : heapSizes) header.append(heapSize).append(" mb | ");
        System.out.println(header);
        for (int i = 0; i < liveSets.length; i++) {
            StringBuilder row = new StringBuilder("| ").append(liveSets[i]).append(" mb | ");
            for (String[] pair : lists.get(i)) row.append(pair[columnIndex]).append(" | ");
            System.out.println(row);
        }
    }
}
```

### G1 Garbage Collector Benchmarks

Running this on the default GC (G1), we get the followings numbers:

Pause Times:

![[Pasted image 20250111193311.png]]

**Throughput**:

![[Pasted image 20250111193326.png]]

Some things worth noting with ZGC:

- In the lower heap-size benchmarks - with heap-size twice live-set - ZGC has worse pause times than the default G1GC (10s to 100s of milliseconds) but and worse throughput (2300-2600 mb/s rather than the 2800-3100 mb/s of G1GC).
- For larger `heap-size`s - 4 times the `live-set` and above - ZGC’s pause times drop to single-digit milliseconds (1-10 ms), much lower than those of G1GC.

As mentioned in the discussion on Theoretical GC Performance, for most garbage collectors pause times are proportional to the live set, and increasing the heap size does not help at all (and according to our G1 Garbage Collector Benchmarks, may even make things worse!). This can be problematic, because there are many use cases that cannot tolerate long GC pause times, but at the same time may require a significant amount of live data to be kept in memory, so shrinking the live-set is not possible.

ZGC provides an option here, where if you are willing to provide _significantly_ more memory than the default G1GC requires, perhaps twice as much, you can get your pause times from 10-100s of milliseconds down to 1-2 milliseconds. These pause times remain low for a wide range of heap sizes and live set sizes, and can be beneficial for a lot of applications that cannot afford to just randomly stop for 100ms at a time. But the extra memory requirement means it’s not a strict improvement, and it really depends on your use case whether the tradeoff is worth it.

## GC Performance Takeaways GC

Now that we’ve studied garbage collections in theory, and looked at some concrete numbers, there are some interesting conclusions. First, the unintuitive things:

- **Adding more memory does _not improve_ GC pause times**. It may even make things worse! This is perhaps the most unintuitive thing about garbage collectors: it seems so obvious that problems with memory management would be solved by adding more memory, but we can see from our theoretical analysis above why that is not the case, and we verified that empirically in benchmarks.

- **Caching data _in-process_ can make garbage collection pause times _worse_!** If you have problems with GC pause times then caching things in-memory will increase the size of your _live-set_ and therefore make your pause times even worse! "LRU" caches in particular are the worst case for garbage collectors, which are typically optimized for collecting recently-allocated short-lived objects. In contrast, caching things _out of process_ does not have this problem. Caching can be worthwhile to reduce redundant computation, but it is not a solution to garbage collection problems.

- **There will never be an _exact_ amount of memory that a garbage-collected application needs.** You can _always_ reduce-overhead/increase-throughput by providing more memory, to make GCs less and less frequent, leaving more time to do useful work. And you can usually provide less memory, at the cost of more and more frequent GCs. Exactly how much memory to provide is thus something you tweak and tune rather than something you can calculate exactly.

- **Fewer larger processes can have worse GC performance than more smaller processes!** There are many ways in which consolidating smaller processes into larger ones can improve efficiency: less per-process overhead, eliminating [inter-process communication](https://en.wikipedia.org/wiki/Inter-process_communication) cost, etc. But GC pause times scale with _total live set size_, so combining two smaller processes into one large one can make pause times _worse_ than they were before. Even if the large process does the same thing as the smaller processes, it can suffer from worse GC pause times.

- **You can reduce pause times by reducing the _live-set_**. If you have very large in-process data structures, moving them somewhere else (e.g. into [SQLite](https://www.sqlite.org/), [Redis](https://github.com/redis/redis), or [Memcached](https://memcached.org/)) would reduce the amount of objects the GC needs to trace and copy every collection, and reduce the pause times

- **Shorter-lived objects are faster to collect**, due to most GCs being _generational_. This also ties into (1) above: caches tend to keep lots of long-lived objects in memory, which apart from slowing down collections due to the size of the live-set, _also_ slows them down by missing out on the GC’s optimizations for short-lived objects.

- **Switch to the Z garbage collector lets you trade off memory for pause times.** JVM programs are by default already very memory hungry compared to other languages (Go, Rust, etc.) and ZGC requires perhaps another 2x as much memory to work. But if you are willing to pay the cost, ZGC can bring pause times down from 50-500ms down to 1-5ms, which may make a big different for latency-sensitive applications.

The Java benchmarks above were run on one particular set of hardware on one version of the JVM, and the exact numbers will differ when run on other hardware or JVM versions. Nevertheless, the overall trends that you can see would remain the same, as would the take-aways of what you need to know to understand garbage collector performance.