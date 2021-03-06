# 01 : Building a Simple GraalVM Native Image

<div class="inline-container">
<img src="../images/noun_Stopwatch_14262_100.png">
<strong>
  Estimated time: 15 minutes
</strong>
</div>

<div class="inline-container">
<img src="../images/noun_Book_3652476_100.png">
<strong>References:</strong>
</div>

- [Native Image : Build a Native Image](https://www.graalvm.org/reference-manual/native-image/#build-a-native-image)


## Overview
GraalVM native image can process your application compiling it ahead of time into a standalone executable.

Some of the benefits you get from it are:

* **Small** standalone distribution, not requiring a JDK
* **Instant Startup**
* **Lower memory** footprint

Let's take a look...

## Creating a Micronaut Application

We'll create a Micronaut application to compute sequences of prime numbers & then we will see how we can easily create 
a fast starting Native Image form this Java application.

First, let's make sure that we have (and they are the latest versions) the tools we need:

- Micronaut ([Install](https://micronaut.io/download.html) Micronaut using SDKMan), version 2.3.0
- GraalVM EE, 21.0.0

![User Input](../images/noun_Computer_3477192_100.png)
![Shell Script](../images/noun_SH_File_272740_100.png)
```bash
# Check Micronaut version
mn --version
Micronaut Version: 2.3.0
```

![User Input](../images/noun_Computer_3477192_100.png)
![Shell Script](../images/noun_SH_File_272740_100.png)
```bash
# Check Java version
java --version
java version "11.0.10" 2021-01-19 LTS
Java(TM) SE Runtime Environment GraalVM EE 21.0.0 (build 11.0.10+8-LTS-jvmci-21.0-b06)
Java HotSpot(TM) 64-Bit Server VM GraalVM EE 21.0.0 (build 11.0.10+8-LTS-jvmci-21.0-b06, mixed mode, sharing)
```

![User Input](../images/noun_Computer_3477192_100.png)
![Shell Script](../images/noun_SH_File_272740_100.png)
```bash
# Check that we have Native Image installed as well
native-image --version
GraalVM Version 21.0.0 EE (Java Version 11.0.10+8-LTS-jvmci-21.0-b06)
```

Now, create the application using Micronaut!

![User Input](../images/noun_Computer_3477192_100.png)
![Shell Script](../images/noun_SH_File_272740_100.png)
```bash
mn create-cli-app primes; cd primes
```

## Adding Functionality to Our App

Create and edit the `src/main/java/primes/PrimesComputer.java` file:

![User Input](../images/noun_Computer_3477192_100.png)
![Java](../images/noun_java_825609_100.png)
```bash
package primes;

import javax.inject.Singleton;
import java.util.stream.*;
import java.util.*;

@Singleton
public class PrimesComputer {
    private Random r = new Random(41);

    public List<Long> random(int upperbound) {
        int to = 2 + r.nextInt(upperbound - 2);
        int from = 1 + r.nextInt(to - 1);
        return primeSequence(from, to);
    }

    public static List<Long> primeSequence(long min, long max) {
        return LongStream.range(min, max)
            .filter(PrimesComputer::isPrime)
            .boxed()
            .collect(Collectors.toList());
    }
    
    /**
     * If n is not a prime, then n = a * b (some a & b)
     * Both a and b can't be larger than sqrt(n), or the product of them would be greater than n
     * One of the factors has to be smaller than sqrt(n).
     * So, if we don't find any factors by the time we get to sqrt(n), then n must be prime
     */
    public static boolean isPrime(long n) {
        return LongStream.rangeClosed(2, (long) Math.sqrt(n))
                .allMatch(i -> n % i != 0);
    }
}
```

Edit the `src/main/java/primes/PrimesCommand.java` file:

![User Input](../images/noun_Computer_3477192_100.png)
![Java](../images/noun_java_825609_100.png)
```java
package primes;

import io.micronaut.configuration.picocli.PicocliRunner;
import io.micronaut.context.ApplicationContext;
import picocli.CommandLine;
import picocli.CommandLine.Command;
import picocli.CommandLine.Option;
import picocli.CommandLine.Parameters;
import javax.inject.*;
import java.util.*;

@Command(name = "primes", description = "...",
        mixinStandardHelpOptions = true)
public class PrimesCommand implements Runnable {
    @Option(names = {"-n", "--n-iterations"}, description = "How many iterations to run")
    int n;

    @Option(names = {"-l", "--limit"}, description = "Upper limit for the sequence")
    int l;

    @Inject
    PrimesComputer primesComputer;

    public static void main(String[] args) throws Exception {
        PicocliRunner.run(PrimesCommand.class, args);
    }

    public void run() {
        for(int i =0; i < n; i++) {
            List<Long> result = primesComputer.random(l);
            System.out.println(result);
        }
    }
}
```

Remove the tests because we changed the functionality of the main command:

![User Input](../images/noun_Computer_3477192_100.png)
![Shell Script](../images/noun_SH_File_272740_100.png)
```bash
rm src/test/java/primes/PrimesCommandTest.java
```

## Build the Java Application

Now we can build this Micronaut project to get the jar file with our functionality:

![User Input](../images/noun_Computer_3477192_100.png)
![Shell Script](../images/noun_SH_File_272740_100.png)
```bash
./gradlew build
```

Test the application that it prints the prime numbers:

![User Input](../images/noun_Computer_3477192_100.png)
![Shell Script](../images/noun_SH_File_272740_100.png)
```bash
java -jar build/libs/primes-0.1-all.jar -n 1 -l 100
[53, 59, 61, 67, 71, 73]
```

## Build a Native Image

Now you can build the native image too:

![User Input](../images/noun_Computer_3477192_100.png)
![Shell Script](../images/noun_SH_File_272740_100.png)
```bash
./gradlew nativeImage
```

Micronaut includes a Gradle plugin to invoke the `native-image` utility and configure its execution.

You can find the resulting executable in `build/native-image/application`.

### Let's take a look at the Native App

Inspect it with the `ldd` utility and check that it's linked to the OS libraries.

![User Input](../images/noun_Computer_3477192_100.png)
![Shell Script](../images/noun_SH_File_272740_100.png)
```bash
# On linux
ldd build/native-image/application
```
If you are on a mac, then you will need to use the `otool`, as below:

![User Input](../images/noun_Computer_3477192_100.png)
![Shell Script](../images/noun_SH_File_272740_100.png)
```bash
# On OsX
otool -L build/native-image/application
```
Take a look at its file type:

![User Input](../images/noun_Computer_3477192_100.png)
![Shell Script](../images/noun_SH_File_272740_100.png)
```bash
file build/native-image/application
```

## Comparing the Startup Time & Performance of JIT & Native Image

If you have GNU time utility (`brew install gnu-time` on Macos), you can time the execution and the memory usage of the process. Compare the following:

![User Input](../images/noun_Computer_3477192_100.png)
![Shell Script](../images/noun_SH_File_272740_100.png)
```bash
/usr/bin/time -v java -jar build/libs/primes-0.1-all.jar -n 1 -l 100
```
vs.

![User Input](../images/noun_Computer_3477192_100.png)
![Shell Script](../images/noun_SH_File_272740_100.png)
```bash
/usr/bin/time -v build/native-image/application -n 1 -l 100
```

Next, we'll try to explore some more options how to configure the build process for native images.

---
<a href="../2/">
    <img src="../images/noun_Next_511450_100.png"
        style="display: inline; height: 6em;" />
</a>
