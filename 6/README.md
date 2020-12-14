# Configuring Memory Used by GraalVM Native Images

<img src="../images/noun_Stopwatch_14262.png"
     style="display: inline; height: 2.5em;">
<strong style="margin: 0;
  position: absolute;
  top: 50%;
  -ms-transform: translateY(-60%);
  transform: translateY(-60%);">
  Estimated time: 5 minutes
</strong>

## Overview

In the previous part we built a small microservice that can respond to the HTTP traffic.
Let's continue using it and try to explore some other deployment options.

## Memory Management in Our App

Locate the sample project we created:

![User Input](../images/noun_Computer_3477192_100.png)
![User Input](../images/noun_SH_File_272740_100.png)
```SH
cd primes-web
```

If you haven't done it yet, build the app:

![User Input](../images/noun_Computer_3477192_100.png)
![User Input](../images/noun_SH_File_272740_100.png)
```SH
./gradlew build
```

If necessary, build the native image of it again:

![User Input](../images/noun_Computer_3477192_100.png)
![User Input](../images/noun_SH_File_272740_100.png)
```SH
./gradlew nativeImage
```

And locate the binary of the app at `./build/native-image/application`.

The native executable uses managed memory just as your application would expect. This means that the allocation/GC of the 
objects are handled by a GC implementation that is contained within the native image, within the SubstrateVM.

## Configuring the Heap

You can configure the heap configuration options similar to running Java application on HotSpot.

Use the `-Xmx` and `-Xmn` options to configure the heap size and the young generation size for your application.

Explore various options to see how low can you go without crashing? or with still reasonably decent startup time 
(<100ms)? Be aware though, everything is a trde off. A native image runs using much less than a Java process would, but 
if you squeeze the memory available to it, beyond certain limits, you will effictively be trading performance / startup time
for memory usage.


---
![User Input](../images/noun_bulb_1912576_100.png)
### In Which Certain Naive Assumptions About Memory Usage are Shown to Require More Subtelty

Another point to consider : Although Native Images use less memory than a Java process requires, if your app loads a lot
of data into memory, and holds it in memory, then these savings may be dwarfed by the amount of memory your application 
needs! 

Also, if you don't constrain the native image of an application when it runs, it will **potentially** consume upto 80% of the available
memory, this **does not** mean that this will happen, just that it could - here native image is behaving in a similar way to the JVM. So you should always try to constrain your process, if you care about memory consumption. Put it into a container and constrain the continer (though be careful to switching off swap, as this will slow donw your app!)

**When running native images in containers, constrain the container to some reasonable size.**

[Memory Management at Imag Run Time](https://www.graalvm.org/reference-manual/native-image/MemoryManagement/)

> If no maximum Java heap size is specified, a native image that uses the Serial GC will set its maximum Java heap size 
> to 80% of the physical memory size. For example, on a machine with 4GB of RAM, the maximum Java heap size will be set 
> to 3.2GB. If the same image is executed on a machine that has 32GB of RAM, the maximum Java heap size will be set to 
> 25.6GB. Note that this is just the maximum value. Depending on the application, the amount of actually used Java heap 
> memory can be much lower. 
---

For example run with the max heap size of 32M:

![User Input](../images/noun_Computer_3477192_100.png)
![User Input](../images/noun_SH_File_272740_100.png)
```SH
./build/native-image/application -Xmx32m
```

The default values for these options are: 8M for the young generation, unlimited (limited by the hw resources) for the max heap size. You can control it with the `-XX:MaximumHeapSizePercent` option.

Enable the logging for GC using the following flags - this may help if you want to understand what is happening in the
heap:

![User Input](../images/noun_Computer_3477192_100.png)
![User Input](../images/noun_SH_File_272740_100.png)
```SH
-XX:+PrintGC - print basic information for every garbage collection
-XX:+VerboseGC - can be added to print further garbage collection details
```

Try using the application for a bit to trigger the GC, observe the logs trying to make sense of them.

![User Input](../images/noun_Computer_3477192_100.png)
![User Input](../images/noun_SH_File_272740_100.png)
```SH
./build/native-image/application -XX:+PrintGC -XX:+VerboseGC -Xmx32m
```

Next, we'll try to explore various options for configuring the runtime for native images.

---
<a href="../7/README.md">
    <img src="../images/noun_Next_511450_100.png"
        style="display: inline; height: 6em;" />
</a>