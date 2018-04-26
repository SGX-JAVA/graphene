### Introducing JMH
JMH is a Java harness library for writing benchmarks on the JVM, and it was developed as part of the OpenJDK project.

JMH is popular for writing *microbenchmarks*, that is, benchmarks that stress a very specific piece of code.

**Creating and running a JMH project**. While JMH releases are being regularly published to Maven Central Repository, JMH development is very active and it is a great idea to make builds of JMH yourself. To do do, you need to clone the JMH Mercurial repository, and then build it with Apache Maven, as shown in Listing 6. Once this is done, you can bootstrap a new Maven-based JMH project, as shown in Listing 7.

```
$ mvn archetype:generate \
    -DinteractiveMode=false \
    -DarchetypeGroupId=org.openjdk.jmh \
    -DarchetypeArtifactId=jmh-java-benchmark-archetype \
    -DgroupId=com.mycompany \
    -DartifactId=benchmarks \
    -Dversion=1.0-SNAPSHOT
```

This creates a project in the `benchmarks` folder. A sample benchmark can be found in `src/main/java/MyBenchmark.java`. While we will dissect the same benchmark in a minute, we can already build the project with Apache Maven:
```
$ cd benchmarks/
$ mvn package
  (...)
$ java -jar target/microbenchmarks.jar
```
When you run the self-contained `microbenchmarks.jar` executable JAR file, JMH launches all the benchmarks of the project with default settings.

```java
package com.mycompany;

import org.openjdk.jmh.annotations.GenerateMicroBenchmark;

public class MyBenchmark {

  @GenerateMicroBenchmark
  public void testMethod() {
    // place your benchmarked code here
  }
}
```

**Anatomy of a JMH benchmark**. The sample benchmark that was generated looks like **Listing 10**. A JMH benchmark is simply a class in which each `@GenerateMicroBenchmark` annotated method is a benchmark. Let's transform the benchmark to measure the cost of adding two integers (see **Listing 11**).

```java
package com.mycompany;

import org.openjdk.jmh.annotations.*;

import java.util.concurrent.TimeUnit;

@State(Scope.Thread)
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@Fork(value = 3, jvmArgsAppend = {"-server", "-disablesystemassertions"})
public class MyBenchmark {

  int x = 923;
  int y = 123;

  @GenerateMicroBenchmark
  @Warmup(iterations = 10, time = 3, timeUnit = TimeUnit.SECONDS)
  public int baseline() {
    return x;
  }

  @GenerateMicroBenchmark
  @Warmup(iterations = 5, time = 5, timeUnit = TimeUnit.SECONDS)
  public int sum() {
    return x + y;
  }
}
```
The benchmark has more configuration annotations present. The `@State` annotation is useful in the context of concurrent benchmarks. In our case, we simply hint to JMH that `x` and `y` are thread-scoped.

We can also require JMH to inject a `Blackhole` object. A `Blackhole` is used when it is not convenient to return a single object from a benchmark method. This happens when the benchmark produces several values, and we want to make sure that the virtual machine will not speculate based on the observation that the benchmark code does not make use of these. The `Blackhole` class provides several `consume(...)` methods.

The class shown in **Listing 13a** and **13b** is an elaborated version of the previous benchmark with a state class, a lifecycle for the state class, and a `Blackhole`.
```java
package com.mycompany;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.logic.BlackHole;

import java.util.Random;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@Fork(value = 3, jvmArgsAppend = {"-server", "-disablesystemassertions"})
public class MyBenchmark {

  @State(Scope.Thread)
  static public class AdditionState {

    int x;
    int y;

    @Setup(Level.Iteration)
    public void prepare() {
      Random random = new Random();
      x = random.nextInt();
      y = random.nextInt();
    }

    @TearDown(Level.Iteration)
    public void shutdown() {
      x = y = 0; // useless in this benchmark...
    }
  }

  @GenerateMicroBenchmark
  @Warmup(iterations = 10, time = 3, timeUnit = TimeUnit.SECONDS)
  public int baseline(AdditionState state) {
    return state.x;
  }

  @GenerateMicroBenchmark
  @Warmup(iterations = 5, time = 5, timeUnit = TimeUnit.SECONDS)
  public int sum(AdditionState state) {
    return state.x + state.y;
  }

  @GenerateMicroBenchmark
  @Warmup(iterations = 10, time = 3, timeUnit = TimeUnit.SECONDS)
  public void baseline_blackhole(AdditionState state, BlackHole blackHole) {
    blackHole.consume(state.x);
  }

  @GenerateMicroBenchmark
  @Warmup(iterations = 5, time = 5, timeUnit = TimeUnit.SECONDS)
  public void sum_blackhole(AdditionState state, BlackHole blackHole) {
    blackHole.consume(state.x + state.y);
  }
}
```
