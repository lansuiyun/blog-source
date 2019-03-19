---
title: 微基准测试JMH
date: 2018-10-19 14:39:41
tags: test
categories: java tool
---
JMH 是 Java Microbenchmark Harness（微基准测试）,其特色优势在于它是由 Oracle 实现 JIT 的相同人员开发的。 
### 为什么使用JMH
在开发过程中，为了确定一个方法的性能，很多时候都是写一个循环多次调用方法通过计算时间来查看性能是否达到预期。但这种简单的方式得出的结论往往是不准确的。现代JVM不断优化，随着代码执行次数的增加，JVM会对其进行编译优化（JIT），最终结果与简单测试得出的结果可能差距很大。此外，测试往往需要做很多额外工作，比如：控制读写线程比例，控制线程同时启动，设置多线程测试中的状态对象，统计测试结果等等，自己实现这些功能既繁琐又不能完全保证正确。而JMH就是恰好能满足上述性能测试需求的工具。

### hello wrold
增加JMH依赖
```
        <dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-core</artifactId>
            <version>1.21</version>
        </dependency>
        <dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-generator-annprocess</artifactId>
            <version>1.18</version>
        </dependency>
```
第一个例子：
```
/**
 * hello JMH
 *
 * @author fei
 */
@BenchmarkMode(Mode.Throughput)
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
public class HelloJHM {

    @Benchmark
    public void wellHelloThere() {
        // this method was intentionally left blank.
    }
}  
```
最终结果：
```
Benchmark                 Mode  Cnt           Score          Error  Units
HelloJHM.wellHelloThere  thrpt    5  1173003521.014 ± 72815462.460  ops/s
```
<!--more-->
### 基本概念
`@BenchmarkMode(Mode.Throughput)`用来设置测试模式，本来中为`吞吐量`模式，即一个时间单位内操作数量。

`@Warmup`用来设置预热相关参数。`iterations`代表预热要迭代几次，`time = 1, timeUnit = TimeUnit.SECONDS`两个参数表明预热每次迭代的时间。

`@Measurement`设置正式测试参数。参数与`@Warmup`意义相同。

`@Benchmark`标记要测试的方法。

运行测试有两种方式：
1. 通过命令行
```
 $ mvn clean install
 $ java -jar target/benchmarks.jar HelloJHM
```
`$ java -jar target/benchmarks.jar -h` 可以查看JMH的命令
2. IDE中运行
在类中加入如下的`main`方法，直接运行。
```
    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(HelloJHM.class.getSimpleName())
                .forks(1)
                .build();

        new Runner(opt).run();
    }
```

### 测试模式
JMH提供的测试模式有：
- `Throughput` 吞吐量 即每时间单位进行的操作数
- `AverageTime` 平均耗时 
- `SampleTime` 随机取样，最后输出取样结果的分布，例如“99%的调用在xxx毫秒以内，99.99%的调用在xxx毫秒以内”
- `SingleShotTime` 执行一次时间。可以用来测试`冷测试模式`（不进行预热）
- `ALL` 以上所有模式

`@BenchmarkMode()` 可以标记在方法上指定方法的测试模式，或者标记在类上提供默认模式。

### 状态对象
大多数的测试都需要维护一些状态，在JMH中在类上加上`@State`注解，类就变成了一个状态对象。
`@State`接受一个 `Scope` 参数用来表示该状态的共享范围。状态对象一般以依赖注入的方式作为参数注入到`Benchmark methods`，或者注入到其它状态对象`Setup and TearDown methods`中做初始化工作。状态对象可以被继承，父类加了`@State`注解，则子类也可以被当做状态对象。
`Scope`有以下三种：
- `Benchmark` 所有线程共享对象，`Setup`、`TearDown`方法只会被其中的一个线程执行一次
- `Group` 现场组共享
- `Thread` 线程独享

也可以给`benchmark`所在类加`@State`注解，`benchmark`方法可以直接使用类属性。

注：**`benchmark`方法中只能使用状态对象或者局部变量，否则会提示编译错误。**
```
public class JMHSample_03_States {
    @State(Scope.Benchmark)
    public static class BenchmarkState {
        volatile double x = Math.PI;
    }

    @State(Scope.Thread)
    public static class ThreadState {
        volatile double x = Math.PI;
    }

    /*
     * Benchmark methods can reference the states, and JMH will inject the
     * appropriate states while calling these methods. You can have no states at
     * all, or have only one state, or have multiple states referenced. This
     * makes building multi-threaded benchmark a breeze.
     *
     * For this exercise, we have two methods.
     */

    @Benchmark
    public void measureUnshared(ThreadState state) {
        // All benchmark threads will call in this method.
        //
        // However, since ThreadState is the Scope.Thread, each thread
        // will have it's own copy of the state, and this benchmark
        // will measure unshared case.
        state.x++;
    }

    @Benchmark
    public void measureShared(BenchmarkState state) {
        // All benchmark threads will call in this method.
        //
        // Since BenchmarkState is the Scope.Benchmark, all threads
        // will share the state instance, and we will end up measuring
        // shared case.
        state.x++;
    }
```


`@Setup`,`@TearDown` 用来标记状态对象的设置与清理方法（JMH中称为`fixture method`），概念类似于JUnit中`@Before`、`@After`。`setup/teardown`方法的数量是任意的。这些方法不会影响测试时间。`@Setup`,`@TearDown`通过设置`Level`参数来指定方法被何时调用，有以下三种：
- `Level.Trial` 一轮benchmark测试前/后调用，默认Level。`fork(number)`即为`number`轮。
- `Level.Iteration` 一次迭代前之后调用
- `Level.Invocation` 每个方法调用前/后调用。**慎用**

除了将单独的类标记@State，也可以将`benchmark`类使用`@State`标记，然后在`benchmark`方法中直接使用类属性。
```
@State(Scope.Thread)
public class JMHSample_06_FixtureLevel {

    double x;
    
    @TearDown(Level.Trial)
    public void check() {
        assert x > Math.PI : "Nothing changed?";
    }

    @Benchmark
    public void measureRight() {
        x++;
    }

    @Benchmark
    public void measureWrong() {
        double x = 0;
        x++;
    }
}
```

### 死代码消除
很多测试失败都是由于死代码消除(Dead-Code Elimination (DCE))引起的。编译器会优化去掉无用的代码。要想限制死代码消除，方法必须返回计算结果，不能返回 void。但如果方法有多个返回值时，部分代码可能会被优化掉
```
    /*
     * While the Math.log(x2) computation is intact, Math.log(x1)
     * is redundant and optimized out.
     */

    @Benchmark
    public double measureWrong() {
        Math.log(x1);
        return Math.log(x2);
    }
```
这时，可以使用`Blackhole`处理返回值
```
    /*
     * This demonstrates Option B:
     *
     * Use explicit Blackhole objects, and sink the values there.
     * (Background: Blackhole is just another @State object, bundled with JMH).
     */

    @Benchmark
    public void measureRight_2(Blackhole bh) {
        bh.consume(Math.log(x1));
        bh.consume(Math.log(x2));
    }
```
死代码消除的另一个方面是常量处理。如果计算结果是可预见的并且不依赖于状态对象，它可能被JIT优化。
```

    // IDEs will say "Oh, you can convert this field to local variable". Don't. Trust. Them.
    // (While this is normally fine advice, it does not work in the context of measuring correctly.)
    private double x = Math.PI;

    // IDEs will probably also say "Look, it could be final". Don't. Trust. Them. Either.
    // (While this is normally fine advice, it does not work in the context of measuring correctly.)
    private final double wrongX = Math.PI;

    @Benchmark
    public double baseline() {
        // simply return the value, this is a baseline
        return Math.PI;
    }

    @Benchmark
    public double measureWrong_1() {
        // This is wrong: the source is predictable, and computation is foldable.
        return Math.log(Math.PI);
    }

    @Benchmark
    public double measureWrong_2() {
        // This is wrong: the source is predictable, and computation is foldable.
        return Math.log(wrongX);
    }

    @Benchmark
    public double measureRight() {
        // This is correct: the source is not predictable.
        return Math.log(x);
    }
```
为防止常量处理被优化掉，需要遵守规则：永远从状态对象的`non-final`字段读取测试输入并返回计算的结果

### fork
jvm在`profile-guided optimizations`方面表现出色，但这对测试来说是不利的，已经完成的测试可能会对其后运行的测试产生影响。而为每个测试fork一个新的java进程可以避免这样的问题。
默认JMH为每个测试fork一个新的java进程。

`@Fork`注解用来设置forking参数，可以被标记在方法（影响单个方法）或者类（影响所有的`Benchmark`方法）上。`@Fork`可以设置fork进程的数量，预热次数，jvm参数等。

`@Fork(0)`表示在同样的进程中运行测试。

### 线程组
使用jmh可以进行非统一测试，即绑定几个测试方法为一组，每个测试方法分配不同个数的线程进行测试。
- `@Group(name)` 为同一个组中的所有测试设置相同的名称.
- `@GroupThreads(threadsNumber)` 为测试方法指定测试线程数量
```
    private AtomicInteger counter;

    @Setup
    public void up() {
        counter = new AtomicInteger();
    }

    @Benchmark
    @Group("g")
    @GroupThreads(3)
    public int inc() {
        return counter.incrementAndGet();
    }

    @Benchmark
    @Group("g")
    @GroupThreads(1)
    public int get() {
        return counter.get();
    }
```
上述代码中：
1. 定义了执行组`g`，共有4个线程，其中3个线程执行`inc()`，一个线程执行`get()`
2. 如果使用4个线程运行测试，则有一个阵型组。使用4*N 个线程运行测试则创建N个执行组
3. 每个执行组共享一个`@State`(counter)对象.

### 编译控制
使用`@CompilerControl(Mode)`可以告诉编译器应该怎么编译特定的方法。有以下几种`Mode`:
- `EXCLUDE` 禁止编译，解释执行
- `INLINE` 强制内联
- `DONT_INLINE` 强制不内联
- `COMPILE_ONLY` 编译
- `PRINT` 打印方法的一些信息
- `BREAK` Insert the breakpoint into the generated compiled code.


### 批量测试
有些方法的性能会随着调用会发生显著的变化（例如：将原始插入到`LinkedList`中间的位置），如果我们需要评估这样一个状态不稳定的操作，使用时间测试测量并求平均值并不是一个好的方法。这时候可以使用`batch`+`SingleShotTime`模式测试。
```
    List<String> list = new LinkedList<>();

    @Benchmark
    @Warmup(iterations = 5, time = 1)
    @Measurement(iterations = 5, time = 1)
    @BenchmarkMode(Mode.AverageTime)
    public List<String> measureWrong_1() {
        list.add(list.size() / 2, "something");
        return list;
    }

    @Benchmark
    @Warmup(iterations = 5, time = 5)
    @Measurement(iterations = 5, time = 5)
    @BenchmarkMode(Mode.AverageTime)
    public List<String> measureWrong_5() {
        list.add(list.size() / 2, "something");
        return list;
    }

    /*
     * This is what you do with JMH.
     */
    @Benchmark
    @Warmup(iterations = 5, batchSize = 5000)
    @Measurement(iterations = 5, batchSize = 5000)
    @BenchmarkMode(Mode.SingleShotTime)
    public List<String> measureRight() {
        list.add(list.size() / 2, "something");
        return list;
    }

    @Setup(Level.Iteration)
    public void setup(){
        list.clear();
    }
```
`@Measurement`的`batch`参数描述了执行一次需要调用测试方法的次数。

### Params
如果需要根据配置对比不同参数情况下的性能，这时候可以使用`@Param`
```
    @Param({"1", "31", "65", "101", "103"})
    public int arg;

    @Param({"0", "1", "2", "4", "8", "16", "32"})
    public int certainty;

    @Benchmark
    public boolean bench() {
        return BigInteger.valueOf(arg).isProbablePrime(certainty);
    }
```

### 安全的使用循环
有以下两种方式可以安全的使用循环:
1. 在循环中将要测试方法的结果传递给`Blackhole(x)`
2. 在循环中将要测试方法的结果传递给禁止内联的空方法
```
    static final int BASE = 42;

    static int work(int x) {
        return BASE + x;
    }

    /*
     * Every benchmark requires control. We do a trivial control for our benchmarks
     * by checking the benchmark costs are growing linearly with increased task size.
     * If it doesn't, then something wrong is happening.
     */

    @Param({"1", "10", "100", "1000"})
    int size;

    int[] xs;

    @Setup
    public void setup() {
        xs = new int[size];
        for (int c = 0; c < size; c++) {
            xs[c] = c;
        }
    }

    /*
     * First, the obviously wrong way: "saving" the result into a local variable would not
     * work. A sufficiently smart compiler will inline work(), and figure out only the last
     * work() call needs to be evaluated. Indeed, if you run it with varying $size, the score
     * will stay the same!
     */

    /**
     * 编译器会内联work()，并发现只有最后一次work()需要被调用。改变xs的长度，你会发现耗时并没有改变
     */
    @Benchmark
    public int measureWrong_1() {
        int acc = 0;
        for (int x : xs) {
            acc = work(x);
        }
        return acc;
    }

    /**
     * Second, another wrong way: "accumulating" the result into a local variable. While
     * it would force the computation of each work() method, there are software pipelining
     * effects in action, that can merge the operations between two otherwise distinct work()
     * bodies. This will obliterate the benchmark setup.
     * In this example, HotSpot does the unrolled loop, merges the $BASE operands into a single
     * addition to $acc, and then does a bunch of very tight stores of $x-s. The final performance
     * depends on how much of the loop unrolling happened *and* how much data is available to make
     * the large strides.
     */
    @Benchmark
    public int measureWrong_2() {
        int acc = 0;
        for (int x : xs) {
            acc += work(x);
        }
        return acc;
    }

    /*
     * Now, let's see how to measure these things properly. A very straight-forward way to
     * break the merging is to sink each result to Blackhole. This will force runtime to compute
     * every work() call in full. (We would normally like to care about several concurrent work()
     * computations at once, but the memory effects from Blackhole.consume() prevent those optimization
     * on most runtimes).
     */
    @Benchmark
    public void measureRight_1(Blackhole bh) {
        for (int x : xs) {
            bh.consume(work(x));
        }
    }

    /*
     * DANGEROUS AREA, PLEASE READ THE DESCRIPTION BELOW.
     *
     * Sometimes, the cost of sinking the value into a Blackhole is dominating the nano-benchmark score.
     * In these cases, one may try to do a make-shift "sinker" with non-inlineable method. This trick is
     * *very* VM-specific, and can only be used if you are verifying the generated code (that's a good
     * strategy when dealing with nano-benchmarks anyway).
     *
     * You SHOULD NOT use this trick in most cases. Apply only where needed.
     */

    /**
     * 如果测试的是nano级别的方法，Blackhole.consume(x)将会占据大部分的消耗。这时候可以将方法结果作为参数传递给一个
     * “non-inlineable”的空方法。
     * 注意：绝大多数情况都不应该使用这个技巧。
     */
    @Benchmark
    public void measureRight_2() {
        for (int x : xs) {
            sink(work(x));
        }
    }

    @CompilerControl(CompilerControl.Mode.DONT_INLINE)
    public static void sink(int v) {
        // IT IS VERY IMPORTANT TO MATCH THE SIGNATURE TO AVOID AUTOBOXING.
        // The method intentionally does nothing.
    }
```

### 其它注解
- `@OperationsPerInvocation` 标记在类或者`benchmark`方法上。用来告诉jmh调用一次方法相当于多少次`operation`。如果测试方法内部有循环，但想测试一次操作的性能，可以用此注解。
- `@OutputTimeUnit` 指定输出时间单位
- `@AuxCounters`

### 工具类
- `Control` :`cnt.stopMeasurement` 见[JMHSample_18_Control](http://hg.openjdk.java.net/code-tools/jmh/file/66fb723292d4/jmh-samples/src/main/java/org/openjdk/jmh/samples/JMHSample_18_Control.java)
- `OptionsBuilder.syncIterations(true)`
- `Blackhole`
    - `consume(o)` 消耗对象，避免JIT优化导致操作被消除
    - `consumeCPU(0)` 消耗CPU时钟
    - 可以直接作为`@Setup`、`@TearDown`方法的参数（`Blackhole`本身是一个状态对象）
- `BenchmarkParams` 获取测试设置
- `IterationParams` 获取当前进行中的迭代的设置
- `ThreadParams` 获取线程的细节信息


### Options & OptionsBuilder
`Options`是JMH的配置的配置类。可以通过`OptionsBuilder`构建，`OptionsBuilder`中一些方法能能达到上述很多注解相同的效果。
- `Options build()` 产生最终的`Options`对象
- `parent(Options other)` 使用other的配置作为默认配置
- `include(regexp)` `exclude(regexp)` 添加/排除需要测试的类。注意：参数为一个正则表达式。可以调用多次。
- `output(filename)` 设置记录运行日志的文件
- `result(filename)` 设置记录运行结果的文件
- `shouldDoGC(boolean value)` 测试迭代间隔是否需要GC
- `shouldFailOnError(boolean value)` 测试过程中出现异常是否意味着测试失败。
    - 设置为`true`，JMH会立即停止，抛出异常，并且已经正确测试过的结果也不会输出。
    - 设置为`false` 会停止测试有异常的方法，其它测试正常进行，最后给出正确测试的结果。 
- `threads(int count)` 设置运行测试的线程数，默认为`1`
- `syncIterations(boolean value)` 是否同步进行测试。详见[JMHSample_17_SyncIterations](http://hg.openjdk.java.net/code-tools/jmh/file/66fb723292d4/jmh-samples/src/main/java/org/openjdk/jmh/samples/JMHSample_17_SyncIterations.java)
    - 设置为`true`，测试会确保在所有线程运行后开始。
    - 如果设置`false`，则不能保证所有线程能同时运行，会对测试结果照常偏差，测试结果会显得更好。默认为`true`。
- `warmupIterations(int value)` 设置预热次数，默认为`5`
- `warmupBatchSize(int value)` 
- `warmupTime(TimeValue value)` 预热每次迭代运行时长，默认`10秒`
- `warmupMode(WarmupMode)` 预热模式，可以单独预热`INDI`或者全部预热`BULK`
- `measurementIterations(int count)` 测试迭代次数，默认为`5`
- `measurementTime(TimeValue value)` 测试热每次迭代运行时长，默认`10秒`
- `mode(Mode mode)` 测试模式，默认`Throughput`
- `timeUnit(TimeUnit tu)` 结果中展示用的时间单位，默认`秒`
- `forks(int value)` fork 数量，默认`5`
- `timeout(TimeValue value)` 每轮等待时长，默认`10分钟`
- `addProfiler(profiler)` 添加分析器


### Profilers
JMH内置了一些有用的Profilers
- `StackProfiler` 基础的堆栈分析器
- `GCProfiler`
- `ClassloaderProfiler`
- `CompilerProfiler` 编译解析器。在调整测试方面非常有用，通过它可以准确的设置warmup 迭代次数，以确保正式测试时JIT已经编译了代码
- `Hotspot profilers`
    - `HotspotClassloadingProfiler`
    - `HotspotCompilationProfiler`
    - `HotspotMemoryProfiler`
    - `HotspotRuntimeProfiler`
    - `HotspotThreadProfiler`

### 指令
```
λ java -jar target/benchmarks.jar -h
Usage: java -jar ... [regexp*] [options]
 [opt] means optional argument.
 <opt> means required argument.
 "+" means comma-separated list of values.
 "time" arguments accept time suffixes, like "100ms".

  [arguments]                 Benchmarks to run (regexp+).
  -bm <mode>                  Benchmark mode. Available modes are: [Throughput/thrpt,
                              AverageTime/avgt, SampleTime/sample, SingleShotTime/ss,
                              All/all]
  -bs <int>                   Batch size: number of benchmark method calls per
                              operation. (some benchmark modes can ignore this
                              setting)
  -e <regexp+>                Benchmarks to exclude from the run.
  -f [int]                    How many times to forks a single benchmark. Use 0 to
                              disable forking altogether (WARNING: disabling
                              forking may have detrimental impact on benchmark
                              and infrastructure reliability, you might want
                              to use different warmup mode instead).
  -foe [bool]                 Should JMH fail immediately if any benchmark had
                              experienced the unrecoverable error?
  -gc [bool]                  Should JMH force GC between iterations?
  -h                          Display help.
  -i <int>                    Number of measurement iterations to do.
  -jvm <string>               Custom JVM to use when forking (path to JVM executable).

  -jvmArgs <string>           Custom JVM args to use when forking.
  -jvmArgsAppend <string>     Custom JVM args to use when forking (append these)

  -jvmArgsPrepend <string>    Custom JVM args to use when forking (prepend these)

  -l                          List matching benchmarks and exit.
  -lprof                      List profilers.
  -lrf                        List result formats.
  -o <filename>               Redirect human-readable output to file.
  -opi <int>                  Operations per invocation.
  -p <param={v,}*>            Benchmark parameters. This option is expected to
                              be used once per parameter. Parameter name and parameter
                              values should be separated with equals sign. Parameter
                              values should be separated with commas.
  -prof <profiler+>           Use profilers to collect additional data. See the
                              list of available profilers first.
  -r <time>                   Time to spend at each measurement iteration.
  -rf <type>                  Result format type. See the list of available result
                              formats first.
  -rff <filename>             Write results to given file.
  -si [bool]                  Synchronize iterations?
  -t <int>                    Number of worker threads to run with.
  -tg <int+>                  Override thread group distribution for asymmetric
                              benchmarks.
  -tu <TU>                    Output time unit. Available time units are: [m, s,
                              ms, us, ns].
  -v <mode>                   Verbosity mode. Available modes are: [SILENT, NORMAL,
                              EXTRA]
  -w <time>                   Time to spend at each warmup iteration.
  -wbs <int>                  Warmup batch size: number of benchmark method calls
                              per operation. (some benchmark modes can ignore
                              this setting)
  -wf <int>                   How many warmup forks to make for a single benchmark.
                              0 to disable warmup forks.
  -wi <int>                   Number of warmup iterations to do.
  -wm <mode>                  Warmup mode for warming up selected benchmarks.
                              Warmup modes are: [INDI, BULK, BULK_INDI].
  -wmb <regexp+>              Warmup benchmarks to include in the run in addition
                              to already selected. JMH will not measure these benchmarks,
                              but only use them for the warmup.
```

### 相关网址
- [jmh](http://openjdk.java.net/projects/code-tools/jmh/)
- [jmh samples](http://hg.openjdk.java.net/code-tools/jmh/file/tip/jmh-samples/src/main/java/org/openjdk/jmh/samples/)
- [Java Performance Tuning Guide](http://java-performance.info/introduction-jmh-profilers/)