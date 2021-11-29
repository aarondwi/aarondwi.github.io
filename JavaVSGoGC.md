# Java VS Go GC

Java does generational memory management. Creating objects do not call malloc, instead just use from already malloc large buffer. This malloc'ed buffer periodically (or when reaching mmeory limit) cleaned, compacted during the GC. No fragmented memory. Allocation is cheap and blazing fast, as every allocations are just like allocations on stack, but longer GC pause to move memory between, at least until ZGC/Shenandoah in JDK 16/17, where every steps in the GC can be done concurrently. This also why in JVM world object pooling not so popular.

Go does not use generational or any kind, but a variant of Concurrent mark-and-sweep. Non compacting, so they can't just use large buffer, cause gonna be wasteful, and instead is a malloc. So allocation is expensive, but can get smaller GC pause (less work as there is no need moving between generation, etc). Also use escape analysis to allocate on stack as most as possible. This is why object pooling is popular, beside reducing malloc cost, also reduce fragmentation.

## Few discussions

1. [Here](https://erik-engheim.medium.com/go-does-not-need-a-java-style-gc-ac99b8d26c60) and its [HN thread](https://news.ycombinator.com/item?id=29319160)
2. [This SO thread](https://stackoverflow.com/questions/40318257/why-go-can-lower-gc-pauses-to-sub-1ms-and-jvm-has-not)
