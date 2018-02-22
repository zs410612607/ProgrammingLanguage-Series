﻿# Garbage Collection

![](https://blog-assets.risingstack.com/2016/11/ancient-garbage-collector-in-action.jpg)

* **程序吞吐量**：回收算法会在多大程度上拖慢程序？有时候，这个是通过回收占用的 CPU 时间与其它 CPU 时间的百分比来描述的。
* **GC 吞吐量**：在给定的 CPU 时间内，回收器可以回收多少垃圾？
    堆内存开销：回收器最少需要多少额外的内存开销？如果回收器在回收垃圾时需要分配临时的内存，对于程序的内存使用是否会有严重影响？
* **停顿时间**：回收器会造成多长时间的停顿？
* **停顿频率**：回收器造成的停顿频率是怎样的？
* **停顿分布**：停顿有时候很短暂，有时候很长？还是选择长一点但保持一致的停顿时间？
* **分配性能**：新内存的分配是快、慢还是无法预测？
* **压缩**：当堆内存里还有小块碎片化的内存可用时，回收器是否仍然抛出内存不足（OOM）的错误？如果不是，那么你是否发现程序越来越慢，并最终死掉，尽管仍然还有足够的内存可用？
* **并发**：回收器是如何利用多核机器的？
* **伸缩**：当堆内存变大时，回收器该如何工作？
* **调优**：回收器的默认使用或在进行调优时，它的配置有多复杂？
* **预热时间**：回收算法是否会根据已发生的行为进行自我调节？如果是，需要多长时间？
* **页释放**：回收算法会把未使用的内存释放回给操作系统吗？如果会，会在什么时候发生？
* **可移植性**：回收器是否能够在提供了较弱内存一致性保证的 CPU 架构上运行？
* **兼容性**：回收器可以跟哪些编程语言和编译器一起工作？它可以跟那些并非为 GC 设计的编程语言一起工作吗，比如 C++？它要求对编译器作出改动吗？如果是，那么是否意味着改变 GC 算法就需要对程序和依赖项进行重新编译？

## Reference

* [Memory management in various languages](http://www.memorymanagement.org/mmref/lang.html)

* [visualizing-garbage-collection-algorithms](https://spin.atomicobject.com/2014/09/03/visualizing-garbage-collection-algorithms/)

* [mark-and-sweep-garbage-collection-algorithm](http://www.geeksforgeeks.org/mark-and-sweep-garbage-collection-algorithm/)

* [Modern Garbage Collection](https://medium.com/@octskyward/modern-garbage-collection-911ef4f8bd8e#.e8fq0wq0r)

# Reference Counting Collector:引用计数

# Mark Sweep:标记清除

第一批垃圾回收算法是为单核机器和小内存程序而设计的。那个时候，CPU 和内存价格昂贵，而且用户没有太多的要求，即使有明显的停顿也没有关系。这个时期的算法设计更注重最小化回收器对 CPU 和堆内存的开销。也就是说，除非内存不足，否则 GC 什么事也不做。而当内存不足时，程序会被暂停，堆空间会被标记并清除，部分内存会被尽快释放出来。这类回收器很古老，不过它们也有一些优势——它们很简单，而且在空闲时不会拖慢你的程序，也不会造成额外的内存开销。像 Boehm GC 这种守旧的回收器，它甚至不要求你对编译器和编程语言做任何改动！这种回收器很适合用在桌面应用里，因为桌面应用的堆内存一般不会很大。比如虚幻游戏引擎，它会在内存里存放数据文件，但不会被扫描到。标记并清除算法存在的最大问题是它的伸缩性很差。在增加 CPU 核数并加大堆空间之后，这种算法几乎无法工作。不过有时候你的堆空间不会很大，而且停顿时间可以接受。那么在这种情况下，你或许可以继续使用这种算法，毕竟它不会造成额外的开销。反过来说，或许你的一台机器就有数百 G 的堆空间和几十核的 CPU，这些服务器可能被用来处理金融市场的交易，或者运行搜索引擎，停顿时间对于你来说很敏感。在这种情况下，你或许希望使用一种能够在后台进行垃圾回收，并带来更短停顿时间的算法，尽管它会拖慢你的程序。不过事情并不会像看上去的那么简单！在这种配置高端的服务器上可能运行着大批量的作业，因为它们是非交互性的，所以停顿时间对于你来说无关紧要，你只关心它们总的运行时间。在这种情况下，你最好使用一种可以最大化吞吐量的算法，尽量提高有效工作时间和回收时间之间的比率。问题是，根本不存在十全十美的算法。没有任何一个语言运行时能够知道你的程序到底是一个批处理作业系统还是一个对延迟敏感的交互型应用。这也就是为什么会存在“GC 调优”——并不是我们的运行时工程师无所作为。这也反映了我们在计算机科学领域的能力是有限的。

# Generational Garbage Collection:分代垃圾回收

从 1984 年以来，人们就已知道大部分的内存对象“朝生夕灭”，它们在分配到内存不久之后就被作为垃圾回收。这就是分代理论假说的基础，它是整个软件产品线领域最贴合实际的发现。数十年来，在软件行业，这个现象在各种编程语言上表现出惊人的一致性，不管是函数式编程语言、命令式编程语言、没有值类型的编程语言，还是有值类型的编程语言。

这个现象的发现是很有意义的，我们可以基于这个现象改进 GC 算法的设计。新的分代回收器比旧的标记并清除回收器有很多改进：

* GC 吞吐量：它们可以更快地回收更多的垃圾。
* 分配内存的性能：分配新内存时不再需要从堆里搜索可用空间，因此内存分配变得很自由。
* 程序的吞吐量：分配的内存空间几乎相互邻接，对缓存的利用有显著的改进。分代回收器要求程序在运行时要做一些额外的工作，不过这点开销完全可以被缓存的改进所带来的好处抵消掉。
* 停顿时间：大多数时候（不是所有）停顿时间变得更短。

不过分代回收器也引入了一些缺点：

* 兼容性：分代回收器需要在内存里移动对象，在某些情况下，当程序对指针进行写入时还需要做一些额外的工作。也就是说，GC 必须跟编译器紧紧地绑定在一起，这也就是为什么 C++里没有分代回收器。
* 堆内存开销：分代回收器通过在内存空间里移动对象实现垃圾回收。这个要求有额外的空间用来拷贝对象，所以这些回收器会带来一些堆内存开销。另外，它们需要维护指针映射表，从而带来更大的开销。
* 停顿时间分布：尽管大部分 GC 停顿时间都很短，不过有一些仍然要求在整个堆内进行彻底的标记并清除操作。
* 调优：分代回收器引入了“年轻代”，或者叫“eden 空间”，程序性能对这块区域的大小非常敏感。
* 预热时间：为了解决上述的调优问题，有一些回收器根据程序的运行情况来决定年轻代的大小，而如果是这样的话，那么 GC 的停顿时间就取决于程序的运行时间长短。在实际当中，只要不是作为基准，这个算不上什么大问题。

因为分代算法的优点，现代垃圾回收器基本上都是基于分代算法。如果你能承受得起，就会想用它们，而一般来说你很可能会这样。分代回收器可以加入其它各种特性，一个现代回收器将会集并发、并行、压缩和分代于一身。

现代计算机使用的默认算法是吞吐量回收算法。这种算法是为批处理作业而设计的，默认情况下不对停顿时间做任何限制（不过可以通过命令行指定）。跟这种默认行为相比较，人们会觉得 Java 的 GC 简直有点糟糕了：默认情况下，Java 试图让你的程序运行尽可能的快，使用尽可能少的内存，但停顿时间却很长。

如果你很在意停顿时间，或许可以使用并发标记并清除回收器（CMS）。这种回收器跟 Go 的回收器最为接近。不过 CMS 也是分代回收器，所以它的停顿时间仍然会比 Go 的要长一些：在年轻代被压缩时，程序会被暂停，因为回收器需要移动对象。CMS 里有两种停顿，第一种是快速的停顿，可能会持续 2 到 5 毫秒，第二种可能会超过 20 毫秒。CMS 是自适应的，因为它的并发性，它需要预测何时需要启动垃圾回收（类似 Go）。你需要对 Go 的堆内存开销进行配置，而 CMS 会在运行时自适应调整，避免发生并发模式故障。不过 CMS 仍然使用标记并清除策略，堆内存仍然会出现碎片化，所以还是会出现问题，程序还是会被拖慢。

最新的 Java 回收器叫作“G1”（Garbage First）。在 Java 8 和 Java 9 里，它都是默认的回收器。它是一种“一刀切”的回收算法，它会尽量满足我们的各种需求。它几乎集并发、分代和堆空间压缩于一身。它在很大程度上可以自我调节，不过因为它无法知道你的真正需求（这个跟其它所有的回收器一样），所以你仍然可以对它做出折衷：告诉它可用的最大内存和预期的停顿时间（以毫秒为单位），它会通过自我调节来达到预期的停顿时间。

默认的预期停顿时间是 100 毫秒，只有指定了更低的预期停顿时间才能看到更好的效果：G1 会优先考虑程序的运行速度，而不是停顿时间。不过停顿时间并非一直保持一致，大部分情况下都非常短（少于 1 毫秒），在压缩堆空间时停顿会长一些（超过 50 毫秒）。G1 具有良好的伸缩性，有人在 TB 级别的堆上使用过 G1。G1 还有一些很有意思的特性，比如堆内字符串去重。

# Java:Concurrent Mark Sweep

>

* [Garbage Collection in Java (3) - Concurrent Mark Sweep](http://insightfullogic.com/2013/May/07/garbage-collection-java-3/)

# Go:Tri-Color Incremental GC

>

* [golangs-real-time-gc-in-theory-and-practice](https://blog.pusher.com/golangs-real-time-gc-in-theory-and-practice/?utm_source=reddit&utm_campaign=blog&utm_medium=social&utm_content=go-sub)
  >
* [why-white-gray-black-in-gc](http://stackoverflow.com/questions/9285741/why-white-gray-black-in-gc)

Go 的新回收器是一种并发的、三基色的、标记清除回收器，它的设计想法是由 Dijkstra 在 1978 年提出的。它有别于现今大多数“企业”级垃圾回收器，而且我们相信它跟现代硬件的属性和现代软件的低延迟需求非常匹配。