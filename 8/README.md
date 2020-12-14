# Profile Guided Optimizations for GraalVM Native Image

<img src="../images/noun_Stopwatch_14262.png"
     style="display: inline; height: 2.5em;">
<strong style="margin: 0;
  position: absolute;
  top: 50%;
  -ms-transform: translateY(-60%);
  transform: translateY(-60%);">
  Estimated time: 15 minutes
</strong>

## Overview

One of the great benefits of the JVM is the JIT (Just-In-Time) compiler. This allows a JVM to profile the code that is
being run, adapt to the data flowing through an application and to improve and tweak the performance. 
We shall continue to experiment on the example project we created before and look at how we can bring htis profiling 
to Native Images

## References
<img src="../images/noun_Book_3652476_100.png"
     style="display: inline; height: 2.5em;">
<strong style="margin: 0;
  position: absolute;
  top: 50%;
  -ms-transform: translateY(-60%);
  transform: translateY(-60%);">
References:
</strong>


- [Native Image Profile Guided Optimisations](https://www.graalvm.org/reference-manual/native-image/PGO/)

## Our Application
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

## Profile Guide Optimisations 

One very interesting feature of the GraalVM native images is the ability to use profile guided optimizations to improve 
the performance of the native executables produced with native image.

In a similar fashion to the JIT compiler gathering the profile, we can build an instrumented image, apply the desired 
load to it (your load tests, benchmark suite, or a slice of real workload), collect the profile, and use it to build a 
more optimized executable for the particular workloads.

Here's how you do it:

1. Instrument your native image application so that it profiles the running code. These profiles are saved to the file system
2. Run a number of representative workloads against the running native image of the application. This will help to generate all of the profiling data
3. Use the saved profile data to build your next native image of the application

## Step 1 : Instrument Our Native Image of the Application

We need to supply the `--pgo-instrument` option to the native image build, then we build native image using the `--pgo` option.

For simplicity and consistency we'll edit the `build.gradle` manually as before, but the separated nature of the build (2 stages) nicely maps into the separate jobs in CI or scripting.

Edit the `build.gradle` to include the following:

![User Input](../images/noun_Computer_3477192_100.png)
![User Input](../images/noun_SH_File_272740_100.png)
```SH
nativeImage {
  args("--pgo-instrument")
  args("--gc=G1")
}
```

Build the image:

![User Input](../images/noun_Computer_3477192_100.png)
![User Input](../images/noun_SH_File_272740_100.png)
```SH
./gradlew nativeImage
```
## Step 2 : Profile

Now run it and apply the load:

![User Input](../images/noun_Computer_3477192_100.png)
![User Input](../images/noun_SH_File_272740_100.png)
```SH
build/native-image/application
```

The load can be shorter because we just need the profile:

![User Input](../images/noun_Computer_3477192_100.png)
![User Input](../images/noun_SH_File_272740_100.png)
```SH
hey -z 30s http://localhost:8080/primes/random/100
```

Stop the application with `Ctrl+C`, look at the profile file:

![User Input](../images/noun_Computer_3477192_100.png)
![User Input](../images/noun_SH_File_272740_100.png)
```SH
ls -la default.iprof
```

## Step 3 : Build the PGO Native Image of the Application

We'll use it now to build the final image with the profile guided optimizations:

Edit the `build.gradle` (note you can have several profile files comma separated there):

Also note the `../..` it's because the building happens in the `build/native-image` directory.

![User Input](../images/noun_Computer_3477192_100.png)
![User Input](../images/noun_SH_File_272740_100.png)
```SH
nativeImage {
  args("--pgo=../../default.iprof")
  args("--gc=G1")
}
```

Build the image:

![User Input](../images/noun_Computer_3477192_100.png)
![User Input](../images/noun_SH_File_272740_100.png)
```SH
./gradlew nativeImage
```

Test it works:

![User Input](../images/noun_Computer_3477192_100.png)
![User Input](../images/noun_SH_File_272740_100.png)
```SH
./build/native-image/application
```

Package it into the docker image. We can use the same `Dockerfile.slim` to build the image (the app on the host has 
changed):

![User Input](../images/noun_Computer_3477192_100.png)
![User Input](../images/noun_SH_File_272740_100.png)
```SH
docker build -f Dockerfile.slim -t primes-web:pgo .
```

Run this new docker image:

![User Input](../images/noun_Computer_3477192_100.png)
![User Input](../images/noun_SH_File_272740_100.png)
```SH
docker run --rm -p 8080:8080 --memory="256m" --memory-swap="256m" --cpus=1 primes-web:pgo
```

And apply the load as before:  

![User Input](../images/noun_Computer_3477192_100.png)
![User Input](../images/noun_SH_File_272740_100.png)
```SH
hey -z 60s http://localhost:8080/primes/random/100
```

Explore the output.

Next, we'll try to explore how to improve peak performance of application even further.

---