---
title: "GC日志解读"
date: 2021-06-05T11:04:00+08:00
draft: false
tags:
- Java
- GC
- JVM
---

为深入学习GC（Garbage Collection，垃圾回收），本文将使用一段测试代码来测试不同的GC策略下的执行情况，并对输出的GC日志做简要分析。

## 1. 测试环境

### 1.1. 操作系统及jdk版本

![操作系统信息](/image/20210605/os-info.jpg)

```bash
➜  01jvm git:(main) ✗ java -version
java version "1.8.0_131"
Java(TM) SE Runtime Environment (build 1.8.0_131-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.131-b11, mixed mode)
```

### 1.2. 测试代码

> 测试代码来源： https://github.com/JavaCourse00/JavaCourseCodes/blob/main/01jvm/GCLogAnalysis.java

```java
import java.util.Random;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.LongAdder;
/*
演示GC日志生成与解读
*/
public class GCLogAnalysis {
    private static Random random = new Random();
    public static void main(String[] args) {
        // 当前毫秒时间戳
        long startMillis = System.currentTimeMillis();
        // 持续运行毫秒数; 可根据需要进行修改
        long timeoutMillis = TimeUnit.SECONDS.toMillis(1);
        // 结束时间戳
        long endMillis = startMillis + timeoutMillis;
        LongAdder counter = new LongAdder();
        System.out.println("正在执行...");
        // 缓存一部分对象; 进入老年代
        int cacheSize = 2000;
        Object[] cachedGarbage = new Object[cacheSize];
        // 在此时间范围内,持续循环
        while (System.currentTimeMillis() < endMillis) {
            // 生成垃圾对象
            Object garbage = generateGarbage(100*1024);
            counter.increment();
            int randomIndex = random.nextInt(2 * cacheSize);
            if (randomIndex < cacheSize) {
                cachedGarbage[randomIndex] = garbage;
            }
        }
        System.out.println("执行结束!共生成对象次数:" + counter.longValue());
    }

    // 生成对象
    private static Object generateGarbage(int max) {
        int randomSize = random.nextInt(max);
        int type = randomSize % 4;
        Object result = null;
        switch (type) {
            case 0:
                result = new int[randomSize];
                break;
            case 1:
                result = new byte[randomSize];
                break;
            case 2:
                result = new double[randomSize];
                break;
            default:
                StringBuilder builder = new StringBuilder();
                String randomString = "randomString-Anything";
                while (builder.length() < randomSize) {
                    builder.append(randomString);
                    builder.append(max);
                    builder.append(randomSize);
                }
                result = builder.toString();
                break;
        }
        return result;
    }
}
```

编译测试代码

```shell
javac GCLogAnalysis.java
```

在控制台执行如下命令，打印GC信息

```shell
java -XX:+PrintGCDetails GCLogAnalysis
```

```log
正在执行...
[GC (Allocation Failure) [PSYoungGen: 65252K->10747K(76288K)] 65252K->19386K(251392K), 0.0064988 secs] [Times: user=0.02 sys=0.02, real=0.01 secs]
[GC (Allocation Failure) [PSYoungGen: 76210K->10738K(141824K)] 84850K->40942K(316928K), 0.0119404 secs] [Times: user=0.02 sys=0.04, real=0.01 secs]
[GC (Allocation Failure) [PSYoungGen: 141810K->10742K(141824K)] 172014K->85396K(316928K), 0.0169297 secs] [Times: user=0.03 sys=0.08, real=0.02 secs]
[GC (Allocation Failure) [PSYoungGen: 141629K->10744K(272896K)] 216283K->130177K(448000K), 0.0191197 secs] [Times: user=0.03 sys=0.09, real=0.02 secs]
[Full GC (Ergonomics) [PSYoungGen: 10744K->0K(272896K)] [ParOldGen: 119433K->119191K(258560K)] 130177K->119191K(531456K), [Metaspace: 2544K->2544K(1056768K)], 0.0200658 secs] [Times: user=0.10 sys=0.00, real=0.02 secs]
[GC (Allocation Failure) [PSYoungGen: 262144K->10751K(272896K)] 381335K->193115K(531456K), 0.0451714 secs] [Times: user=0.04 sys=0.09, real=0.04 secs]
[Full GC (Ergonomics) [PSYoungGen: 10751K->0K(272896K)] [ParOldGen: 182363K->166011K(343040K)] 193115K->166011K(615936K), [Metaspace: 2544K->2544K(1056768K)], 0.0215776 secs] [Times: user=0.11 sys=0.00, real=0.02 secs]
[GC (Allocation Failure) [PSYoungGen: 262144K->84521K(546304K)] 428155K->250532K(889344K), 0.0364856 secs] [Times: user=0.05 sys=0.10, real=0.04 secs]
[GC (Allocation Failure) [PSYoungGen: 542761K->102396K(560640K)] 708772K->350584K(903680K), 0.1000140 secs] [Times: user=0.10 sys=0.22, real=0.10 secs]
[GC (Allocation Failure) [PSYoungGen: 560636K->161780K(923648K)] 808824K->443025K(1266688K), 0.1457590 secs] [Times: user=0.10 sys=0.26, real=0.15 secs]
[Full GC (Ergonomics) [PSYoungGen: 161780K->0K(923648K)] [ParOldGen: 281244K->304039K(501760K)] 443025K->304039K(1425408K), [Metaspace: 2544K->2544K(1056768K)], 0.0563595 secs] [Times: user=0.19 sys=0.03, real=0.05 secs]
执行结束!共生成对象次数:6870
Heap
 PSYoungGen      total 923648K, used 30397K [0x000000076ab00000, 0x00000007b5f00000, 0x00000007c0000000)
  eden space 761856K, 3% used [0x000000076ab00000,0x000000076c8af788,0x0000000799300000)
  from space 161792K, 0% used [0x00000007a5b00000,0x00000007a5b00000,0x00000007af900000)
  to   space 204800K, 0% used [0x0000000799300000,0x0000000799300000,0x00000007a5b00000)
 ParOldGen       total 501760K, used 304039K [0x00000006c0000000, 0x00000006dea00000, 0x000000076ab00000)
  object space 501760K, 60% used [0x00000006c0000000,0x00000006d28e9dd8,0x00000006dea00000)
 Metaspace       used 2551K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 276K, capacity 386K, committed 512K, reserved 1048576K
```

还可以使用以下命令，将GC信息输出到日志文件中，同时打印GC的详细信息和时间戳

```shell
java -Xloggc:gc.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps GCLogAnalysis
```

## 2. 演示不同GC策略

### 2.1 并行GC的演示

#### 2.1.1 默认不配置堆内存参数

```shell
# input
java -XX:+PrintGCDetails -XX:+PrintGCDateStamps GCLogAnalysis

# output
正在执行...
2020-10-25T20:09:45.532-0800: [GC (Allocation Failure) [PSYoungGen: 65536K->10730K(76288K)] 65536K->22808K(251392K), 0.0094137 secs] [Times: user=0.02 sys=0.04, real=0.01 secs]
2020-10-25T20:09:45.560-0800: [GC (Allocation Failure) [PSYoungGen: 76266K->10746K(141824K)] 88344K->41760K(316928K), 0.0207021 secs] [Times: user=0.02 sys=0.09, real=0.02 secs]
2020-10-25T20:09:45.629-0800: [GC (Allocation Failure) [PSYoungGen: 141818K->10742K(141824K)] 172832K->84807K(316928K), 0.0276267 secs] [Times: user=0.04 sys=0.14, real=0.03 secs]
2020-10-25T20:09:45.690-0800: [GC (Allocation Failure) [PSYoungGen: 141814K->10742K(272896K)] 215879K->129048K(448000K), 0.0265651 secs] [Times: user=0.03 sys=0.14, real=0.02 secs]
2020-10-25T20:09:45.717-0800: [Full GC (Ergonomics) [PSYoungGen: 10742K->0K(272896K)] [ParOldGen: 118306K->117582K(252928K)] 129048K->117582K(525824K), [Metaspace: 2689K->2689K(1056768K)], 0.0275435 secs] [Times: user=0.15 sys=0.01, real=0.03 secs]
2020-10-25T20:09:45.848-0800: [GC (Allocation Failure) [PSYoungGen: 262144K->10746K(272896K)] 379726K->196735K(525824K), 0.0408603 secs] [Times: user=0.05 sys=0.20, real=0.04 secs]
2020-10-25T20:09:45.889-0800: [Full GC (Ergonomics) [PSYoungGen: 10746K->0K(272896K)] [ParOldGen: 185988K->170435K(351232K)] 196735K->170435K(624128K), [Metaspace: 2689K->2689K(1056768K)], 0.0375020 secs] [Times: user=0.22 sys=0.01, real=0.04 secs]
2020-10-25T20:09:45.982-0800: [GC (Allocation Failure) [PSYoungGen: 262144K->84652K(532480K)] 432579K->255087K(883712K), 0.0448442 secs] [Times: user=0.05 sys=0.22, real=0.04 secs]
2020-10-25T20:09:46.169-0800: [GC (Allocation Failure) [PSYoungGen: 528044K->102900K(546304K)] 698479K->347045K(897536K), 0.0791139 secs] [Times: user=0.10 sys=0.39, real=0.08 secs]
2020-10-25T20:09:46.326-0800: [GC (Allocation Failure) [PSYoungGen: 546292K->157688K(867328K)] 790437K->437489K(1218560K), 0.0928147 secs] [Times: user=0.10 sys=0.49, real=0.09 secs]
2020-10-25T20:09:46.419-0800: [Full GC (Ergonomics) [PSYoungGen: 157688K->0K(867328K)] [ParOldGen: 279800K->293211K(494080K)] 437489K->293211K(1361408K), [Metaspace: 2689K->2689K(1056768K)], 0.0528697 secs] [Times: user=0.29 sys=0.04, real=0.06 secs]
执行结束!共生成对象次数:7171
Heap
 PSYoungGen      total 867328K, used 86222K [0x000000076ab00000, 0x00000007b1f00000, 0x00000007c0000000)
  eden space 709632K, 12% used [0x000000076ab00000,0x000000076ff33908,0x0000000796000000)
  from space 157696K, 0% used [0x00000007a2400000,0x00000007a2400000,0x00000007abe00000)
  to   space 200704K, 0% used [0x0000000796000000,0x0000000796000000,0x00000007a2400000)
 ParOldGen       total 494080K, used 293211K [0x00000006c0000000, 0x00000006de280000, 0x000000076ab00000)
  object space 494080K, 59% used [0x00000006c0000000,0x00000006d1e56fa8,0x00000006de280000)
 Metaspace       used 2695K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 295K, capacity 386K, committed 512K, reserved 1048576K
```

不指定垃圾收集器 默认使用的是并行GC

以上GC日志显示，程序执行期间共发生11次GC 其中包含3次Full GC

第一次Young GC，[GC (Allocation Failure) [PSYoungGen: 65536K->10730K(76288K)] 65536K->22808K(251392K), 0.0094137 secs] [Times: user=0.02 sys=0.04, real=0.01 secs]

Young区堆内存使用 65536K->10730K(76288K)，减少54806K。

整个堆内存使用 65536K->22808K(251392K)，减少42728K。

初始的堆内存使用量相同，是因为一次GC也没有发生时，Old区尚未使用。

Young区的减少 大于 整个堆区的减少，因为有可能有约十几m的对象进入了Old区。Young GC发生时，一部分对象被晋升到了Old区，一部分对象被复制到了未使用的Survivor区。

经过4次Young GC之后，堆内存的使用已经较大，程序继续执行就触发了一次Full GC

[Full GC (Ergonomics) [PSYoungGen: 10742K->0K(272896K)] [ParOldGen: 118306K->117582K(252928K)] 129048K->117582K(525824K), [Metaspace: 2689K->2689K(1056768K)], 0.0275435 secs] [Times: user=0.15 sys=0.01, real=0.03 secs]

Full GC发生时

Young区10742K->0K(272896K)

Old区118306K->117582K(252928K) ，Old区变化较少

整个堆129048K->117582K(525824K)

Metaspace 无变化

在不配置堆内存参数的情况下 默认使用的堆内存大小是物理内存的1/4，本机配置为8核16G 所以此处默认的堆内存大小为4G，查看GC日志之后发现执行的命令如下：

CommandLine flags: -XX:InitialHeapSize=268435456 -XX:MaxHeapSize=4294967296 -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC

只是Xmx的配置是4G Xms的配置为256m


#### 2.1.2 配置512m堆内存

```shell
java -XX:+UseParallelGC -Xms512m -Xmx512m -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps GCLogAnalysis
正在执行...
2020-10-27T21:48:09.230-0800: 0.168: [GC (Allocation Failure) [PSYoungGen: 131584K->21496K(153088K)] 131584K->50443K(502784K), 0.0223842 secs] [Times: user=0.03 sys=0.11, real=0.02 secs]
2020-10-27T21:48:09.296-0800: 0.234: [GC (Allocation Failure) [PSYoungGen: 153080K->21500K(153088K)] 182027K->92156K(502784K), 0.0301401 secs] [Times: user=0.06 sys=0.15, real=0.03 secs]
2020-10-27T21:48:09.351-0800: 0.289: [GC (Allocation Failure) [PSYoungGen: 153084K->21492K(153088K)] 223740K->136536K(502784K), 0.0361967 secs] [Times: user=0.09 sys=0.14, real=0.03 secs]
2020-10-27T21:48:09.411-0800: 0.349: [GC (Allocation Failure) [PSYoungGen: 152653K->21493K(153088K)] 267697K->176705K(502784K), 0.0263466 secs] [Times: user=0.06 sys=0.11, real=0.02 secs]
2020-10-27T21:48:09.466-0800: 0.404: [GC (Allocation Failure) [PSYoungGen: 153077K->21490K(153088K)] 308289K->219164K(502784K), 0.0333237 secs] [Times: user=0.06 sys=0.14, real=0.04 secs]
2020-10-27T21:48:09.530-0800: 0.469: [GC (Allocation Failure) [PSYoungGen: 153074K->21494K(80384K)] 350748K->257036K(430080K), 0.0261909 secs] [Times: user=0.07 sys=0.09, real=0.02 secs]
2020-10-27T21:48:09.571-0800: 0.510: [GC (Allocation Failure) [PSYoungGen: 79983K->36297K(116736K)] 315525K->275163K(466432K), 0.0065826 secs] [Times: user=0.03 sys=0.01, real=0.00 secs]
2020-10-27T21:48:09.594-0800: 0.532: [GC (Allocation Failure) [PSYoungGen: 95177K->48333K(116736K)] 334043K->294781K(466432K), 0.0154817 secs] [Times: user=0.08 sys=0.02, real=0.02 secs]
2020-10-27T21:48:09.631-0800: 0.570: [GC (Allocation Failure) [PSYoungGen: 107179K->57284K(116736K)] 353627K->312167K(466432K), 0.0217891 secs] [Times: user=0.10 sys=0.03, real=0.02 secs]
2020-10-27T21:48:09.671-0800: 0.610: [GC (Allocation Failure) [PSYoungGen: 116164K->37958K(116736K)] 371047K->328937K(466432K), 0.0355582 secs] [Times: user=0.07 sys=0.13, real=0.03 secs]
2020-10-27T21:48:09.707-0800: 0.645: [Full GC (Ergonomics) [PSYoungGen: 37958K->0K(116736K)] [ParOldGen: 290979K->229237K(349696K)] 328937K->229237K(466432K), [Metaspace: 2689K->2689K(1056768K)], 0.0638496 secs] [Times: user=0.37 sys=0.02, real=0.07 secs]
2020-10-27T21:48:09.782-0800: 0.720: [GC (Allocation Failure) [PSYoungGen: 58880K->15657K(116736K)] 288117K->244895K(466432K), 0.0021288 secs] [Times: user=0.01 sys=0.00, real=0.00 secs]
2020-10-27T21:48:09.794-0800: 0.732: [GC (Allocation Failure) [PSYoungGen: 74537K->19089K(116736K)] 303775K->263160K(466432K), 0.0037648 secs] [Times: user=0.03 sys=0.00, real=0.00 secs]
2020-10-27T21:48:09.808-0800: 0.746: [GC (Allocation Failure) [PSYoungGen: 77863K->21565K(116736K)] 321934K->284247K(466432K), 0.0138577 secs] [Times: user=0.07 sys=0.00, real=0.02 secs]
2020-10-27T21:48:09.843-0800: 0.781: [GC (Allocation Failure) [PSYoungGen: 80028K->22805K(116736K)] 342710K->305201K(466432K), 0.0122709 secs] [Times: user=0.07 sys=0.01, real=0.01 secs]
2020-10-27T21:48:09.877-0800: 0.815: [GC (Allocation Failure) [PSYoungGen: 81685K->18378K(116736K)] 364081K->323158K(466432K), 0.0197694 secs] [Times: user=0.05 sys=0.04, real=0.02 secs]
2020-10-27T21:48:09.897-0800: 0.835: [Full GC (Ergonomics) [PSYoungGen: 18378K->0K(116736K)] [ParOldGen: 304780K->263770K(349696K)] 323158K->263770K(466432K), [Metaspace: 2689K->2689K(1056768K)], 0.0698176 secs] [Times: user=0.40 sys=0.01, real=0.07 secs]
2020-10-27T21:48:09.989-0800: 0.927: [GC (Allocation Failure) [PSYoungGen: 58880K->22461K(116736K)] 322650K->286232K(466432K), 0.0070142 secs] [Times: user=0.05 sys=0.00, real=0.01 secs]
2020-10-27T21:48:10.018-0800: 0.956: [GC (Allocation Failure) [PSYoungGen: 81323K->20438K(116736K)] 345094K->304973K(466432K), 0.0113317 secs] [Times: user=0.07 sys=0.01, real=0.02 secs]
2020-10-27T21:48:10.050-0800: 0.988: [GC (Allocation Failure) [PSYoungGen: 79260K->22582K(116736K)] 363796K->326898K(466432K), 0.0137229 secs] [Times: user=0.08 sys=0.00, real=0.01 secs]
2020-10-27T21:48:10.085-0800: 1.023: [GC (Allocation Failure) [PSYoungGen: 81462K->21499K(116736K)] 385778K->348006K(466432K), 0.0248649 secs] [Times: user=0.05 sys=0.08, real=0.03 secs]
2020-10-27T21:48:10.110-0800: 1.048: [Full GC (Ergonomics) [PSYoungGen: 21499K->0K(116736K)] [ParOldGen: 326506K->285936K(349696K)] 348006K->285936K(466432K), [Metaspace: 2689K->2689K(1056768K)], 0.0695991 secs] [Times: user=0.42 sys=0.01, real=0.06 secs]
执行结束!共生成对象次数:5942
Heap
 PSYoungGen      total 116736K, used 2476K [0x00000007b5580000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 58880K, 4% used [0x00000007b5580000,0x00000007b57eb1f8,0x00000007b8f00000)
  from space 57856K, 0% used [0x00000007b8f00000,0x00000007b8f00000,0x00000007bc780000)
  to   space 57856K, 0% used [0x00000007bc780000,0x00000007bc780000,0x00000007c0000000)
 ParOldGen       total 349696K, used 285936K [0x00000007a0000000, 0x00000007b5580000, 0x00000007b5580000)
  object space 349696K, 81% used [0x00000007a0000000,0x00000007b173c3c0,0x00000007b5580000)
 Metaspace       used 2695K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 295K, capacity 386K, committed 512K, reserved 1048576K
```

> 2020-10-27T21:48:09.230-0800: 0.168: [GC (Allocation Failure) [PSYoungGen: 131584K->21496K(153088K)] 131584K->50443K(502784K), 0.0223842 secs] [Times: user=0.03 sys=0.11, real=0.02 secs]

第一次Young GC

年轻代使用的堆内存从131584K 降低到 21496K ，减少110088K，Young区的容量为153088K

堆内存使用从151584K 降低至 50443K，减少101141K ，整个堆容量为502784K

共有8947K大小的对象从年轻代中晋升到老年代。

GC耗时约22ms

> 2020-10-27T21:48:09.707-0800: 0.645: [Full GC (Ergonomics) [PSYoungGen: 37958K->0K(116736K)] [ParOldGen: 290979K->229237K(349696K)] 328937K->229237K(466432K), [Metaspace: 2689K->2689K(1056768K)], 0.0638496 secs] [Times: user=0.37 sys=0.02, real=0.07 secs]

第一次Full GC

年轻代使用的堆内存从37958K减少到0

老年代使用的堆内存从290979K 减少到 229237K 销毁掉61742K大小的对象

整个堆内存的使用从328937K 减少到 229237K

元数据区没有变化

GC耗时约64ms 应用程序暂停70ms

#### 2.1.3 使用2g堆内存

```shell
java -XX:+UseParallelGC -Xms2g -Xmx2g -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps GCLogAnalysis
正在执行...
2020-10-27T22:03:40.498-0800: 0.375: [GC (Allocation Failure) [PSYoungGen: 524800K->87035K(611840K)] 524800K->141649K(2010112K), 0.0619776 secs] [Times: user=0.07 sys=0.33, real=0.06 secs]
2020-10-27T22:03:40.660-0800: 0.537: [GC (Allocation Failure) [PSYoungGen: 611835K->87030K(611840K)] 666449K->258132K(2010112K), 0.0911996 secs] [Times: user=0.11 sys=0.48, real=0.09 secs]
2020-10-27T22:03:40.840-0800: 0.717: [GC (Allocation Failure) [PSYoungGen: 611830K->87032K(611840K)] 782932K->373115K(2010112K), 0.0814008 secs] [Times: user=0.23 sys=0.29, real=0.08 secs]
2020-10-27T22:03:41.021-0800: 0.898: [GC (Allocation Failure) [PSYoungGen: 611832K->87032K(611840K)] 897915K->489779K(2010112K), 0.0649686 secs] [Times: user=0.15 sys=0.28, real=0.06 secs]
2020-10-27T22:03:41.175-0800: 1.052: [GC (Allocation Failure) [PSYoungGen: 611832K->87025K(611840K)] 1014579K->595189K(2010112K), 0.0598866 secs] [Times: user=0.14 sys=0.25, real=0.06 secs]
执行结束!共生成对象次数:9771
Heap
 PSYoungGen      total 611840K, used 108298K [0x0000000795580000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 524800K, 4% used [0x0000000795580000,0x0000000796a46228,0x00000007b5600000)
  from space 87040K, 99% used [0x00000007b5600000,0x00000007baafc668,0x00000007bab00000)
  to   space 87040K, 0% used [0x00000007bab00000,0x00000007bab00000,0x00000007c0000000)
 ParOldGen       total 1398272K, used 508163K [0x0000000740000000, 0x0000000795580000, 0x0000000795580000)
  object space 1398272K, 36% used [0x0000000740000000,0x000000075f040e58,0x0000000795580000)
 Metaspace       used 2695K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 295K, capacity 386K, committed 512K, reserved 1048576K
```

可以观察到未发生Full GC，GC之后老年代的使用率只有36%

创建的对象数也随着堆内存的增大而增多。

#### 2.1.4 使用128m堆内存

```shell
java -XX:+UseParallelGC -Xms128m -Xmx128m -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps GCLogAnalysis
正在执行...
2020-10-27T22:09:29.946-0800: 0.112: [GC (Allocation Failure) [PSYoungGen: 33280K->5098K(38400K)] 33280K->13540K(125952K), 0.0055672 secs] [Times: user=0.01 sys=0.02, real=0.01 secs]
2020-10-27T22:09:29.960-0800: 0.126: [GC (Allocation Failure) [PSYoungGen: 37794K->5110K(38400K)] 46236K->25958K(125952K), 0.0095484 secs] [Times: user=0.02 sys=0.05, real=0.00 secs]
2020-10-27T22:09:29.979-0800: 0.145: [GC (Allocation Failure) [PSYoungGen: 38390K->5103K(38400K)] 59238K->36124K(125952K), 0.0071565 secs] [Times: user=0.02 sys=0.02, real=0.01 secs]
2020-10-27T22:09:29.997-0800: 0.162: [GC (Allocation Failure) [PSYoungGen: 38140K->5117K(38400K)] 69161K->48383K(125952K), 0.0092709 secs] [Times: user=0.02 sys=0.04, real=0.01 secs]
2020-10-27T22:09:30.018-0800: 0.184: [GC (Allocation Failure) [PSYoungGen: 38319K->5119K(38400K)] 81586K->61396K(125952K), 0.0091358 secs] [Times: user=0.02 sys=0.03, real=0.01 secs]
2020-10-27T22:09:30.036-0800: 0.202: [GC (Allocation Failure) [PSYoungGen: 38399K->5119K(19968K)] 94676K->74085K(107520K), 0.0100919 secs] [Times: user=0.02 sys=0.04, real=0.01 secs]
2020-10-27T22:09:30.050-0800: 0.216: [GC (Allocation Failure) [PSYoungGen: 19851K->7473K(29184K)] 88816K->79274K(116736K), 0.0030000 secs] [Times: user=0.01 sys=0.01, real=0.00 secs]
2020-10-27T22:09:30.053-0800: 0.219: [Full GC (Ergonomics) [PSYoungGen: 7473K->0K(29184K)] [ParOldGen: 71800K->70817K(87552K)] 79274K->70817K(116736K), [Metaspace: 2689K->2689K(1056768K)], 0.0220339 secs] [Times: user=0.11 sys=0.00, real=0.02 secs]
2020-10-27T22:09:30.082-0800: 0.248: [GC (Allocation Failure) [PSYoungGen: 14823K->4802K(29184K)] 85640K->75619K(116736K), 0.0015467 secs] [Times: user=0.01 sys=0.00, real=0.00 secs]
2020-10-27T22:09:30.086-0800: 0.252: [GC (Allocation Failure) [PSYoungGen: 19650K->9309K(29184K)] 90467K->80126K(116736K), 0.0018068 secs] [Times: user=0.01 sys=0.00, real=0.00 secs]
2020-10-27T22:09:30.092-0800: 0.258: [GC (Allocation Failure) [PSYoungGen: 24001K->8277K(29184K)] 94818K->83744K(116736K), 0.0029993 secs] [Times: user=0.02 sys=0.01, real=0.00 secs]
2020-10-27T22:09:30.095-0800: 0.261: [Full GC (Ergonomics) [PSYoungGen: 8277K->0K(29184K)] [ParOldGen: 75467K->79210K(87552K)] 83744K->79210K(116736K), [Metaspace: 2689K->2689K(1056768K)], 0.0224154 secs] [Times: user=0.12 sys=0.02, real=0.02 secs]
2020-10-27T22:09:30.125-0800: 0.291: [Full GC (Ergonomics) [PSYoungGen: 14731K->0K(29184K)] [ParOldGen: 79210K->83484K(87552K)] 93941K->83484K(116736K), [Metaspace: 2689K->2689K(1056768K)], 0.0188675 secs] [Times: user=0.11 sys=0.01, real=0.02 secs]
2020-10-27T22:09:30.146-0800: 0.312: [Full GC (Ergonomics) [PSYoungGen: 14367K->1235K(29184K)] [ParOldGen: 83484K->86606K(87552K)] 97851K->87842K(116736K), [Metaspace: 2689K->2689K(1056768K)], 0.0255021 secs] [Times: user=0.16 sys=0.01, real=0.03 secs]
2020-10-27T22:09:30.174-0800: 0.340: [Full GC (Ergonomics) [PSYoungGen: 14821K->5046K(29184K)] [ParOldGen: 86606K->87242K(87552K)] 101427K->92288K(116736K), [Metaspace: 2689K->2689K(1056768K)], 0.0185236 secs] [Times: user=0.12 sys=0.01, real=0.02 secs]
2020-10-27T22:09:30.196-0800: 0.362: [Full GC (Ergonomics) [PSYoungGen: 14463K->8485K(29184K)] [ParOldGen: 87242K->86870K(87552K)] 101706K->95355K(116736K), [Metaspace: 2689K->2689K(1056768K)], 0.0251259 secs] [Times: user=0.16 sys=0.00, real=0.03 secs]
2020-10-27T22:09:30.223-0800: 0.389: [Full GC (Ergonomics) [PSYoungGen: 14779K->9297K(29184K)] [ParOldGen: 86870K->87484K(87552K)] 101649K->96781K(116736K), [Metaspace: 2689K->2689K(1056768K)], 0.0225548 secs] [Times: user=0.15 sys=0.01, real=0.02 secs]
2020-10-27T22:09:30.248-0800: 0.414: [Full GC (Ergonomics) [PSYoungGen: 14518K->10348K(29184K)] [ParOldGen: 87484K->87310K(87552K)] 102002K->97658K(116736K), [Metaspace: 2689K->2689K(1056768K)], 0.0076190 secs] [Times: user=0.04 sys=0.00, real=0.01 secs]
2020-10-27T22:09:30.257-0800: 0.423: [Full GC (Ergonomics) [PSYoungGen: 14816K->12611K(29184K)] [ParOldGen: 87310K->87190K(87552K)] 102127K->99801K(116736K), [Metaspace: 2689K->2689K(1056768K)], 0.0289935 secs] [Times: user=0.20 sys=0.00, real=0.03 secs]
2020-10-27T22:09:30.287-0800: 0.453: [Full GC (Ergonomics) [PSYoungGen: 14813K->13881K(29184K)] [ParOldGen: 87190K->87130K(87552K)] 102003K->101012K(116736K), [Metaspace: 2689K->2689K(1056768K)], 0.0038787 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
2020-10-27T22:09:30.291-0800: 0.457: [Full GC (Ergonomics) [PSYoungGen: 14831K->14006K(29184K)] [ParOldGen: 87130K->87130K(87552K)] 101961K->101136K(116736K), [Metaspace: 2689K->2689K(1056768K)], 0.0018450 secs] [Times: user=0.01 sys=0.00, real=0.00 secs]
2020-10-27T22:09:30.293-0800: 0.459: [Full GC (Ergonomics) [PSYoungGen: 14311K->14200K(29184K)] [ParOldGen: 87130K->87130K(87552K)] 101441K->101330K(116736K), [Metaspace: 2689K->2689K(1056768K)], 0.0016810 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-27T22:09:30.295-0800: 0.461: [Full GC (Allocation Failure) [PSYoungGen: 14200K->14200K(29184K)] [ParOldGen: 87130K->87111K(87552K)] 101330K->101311K(116736K), [Metaspace: 2689K->2689K(1056768K)], 0.0293173 secs] [Times: user=0.17 sys=0.01, real=0.03 secs]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at GCLogAnalysis.generateGarbage(GCLogAnalysis.java:48)
	at GCLogAnalysis.main(GCLogAnalysis.java:25)
Heap
 PSYoungGen      total 29184K, used 14551K [0x00000007bd580000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 14848K, 98% used [0x00000007bd580000,0x00000007be3b5ed0,0x00000007be400000)
  from space 14336K, 0% used [0x00000007bf200000,0x00000007bf200000,0x00000007c0000000)
  to   space 14336K, 0% used [0x00000007be400000,0x00000007be400000,0x00000007bf200000)
 ParOldGen       total 87552K, used 87111K [0x00000007b8000000, 0x00000007bd580000, 0x00000007bd580000)
  object space 87552K, 99% used [0x00000007b8000000,0x00000007bd511c70,0x00000007bd580000)
 Metaspace       used 2719K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 297K, capacity 386K, committed 512K, reserved 1048576K
```

当堆内存降低至128m时，系统不仅频频发生Full GC，而且产生了`java.lang.OutOfMemoryError`

而且在发生Full GC的时候，整个老年代及堆区被回收的对象很少，GC之后新生代Eden区的使用率达到98% 老年代使用率达到99%


#### 2.1.5 并行GC总结

1. 使用`-XX:+UseParallelGC`指定GC为并行GC
2. 年轻代使用“标记-复制”算法
3. 老年代使用“标记-清除-整理”算法
4. 使用`-XX:ParallelGCThreads=N`来指定 GC 线程数，默认是机器CPU的核数
5. 适用于多核的服务器，主要的目标是增加吞吐量
    * GC期间，所有的CPU内核都在并行清理垃圾，总的暂停时间更短
    * GC间隔，没有GC线程运行，不会消耗系统资源
6. 默认情况下会启用并行压缩，需要关闭的话则要指定参数`-XX:+UseParallelOldGC`

### 2.2 串行GC的演示

#### 2.2.1 使用512m堆内存

```shell
java -XX:+UseSerialGC -Xms512m -Xmx512m -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps GCLogAnalysis
正在执行...
2020-10-26T22:16:34.202-0800: 0.169: [GC (Allocation Failure) 2020-10-26T22:16:34.202-0800: 0.169: [DefNew: 139450K->17472K(157248K), 0.0360407 secs] 139450K->54983K(506816K), 0.0361185 secs] [Times: user=0.02 sys=0.01, real=0.03 secs]
2020-10-26T22:16:34.278-0800: 0.246: [GC (Allocation Failure) 2020-10-26T22:16:34.279-0800: 0.246: [DefNew: 157201K->17469K(157248K), 0.0431319 secs] 194713K->103739K(506816K), 0.0431980 secs] [Times: user=0.04 sys=0.02, real=0.05 secs]
2020-10-26T22:16:34.351-0800: 0.318: [GC (Allocation Failure) 2020-10-26T22:16:34.351-0800: 0.318: [DefNew: 157245K->17469K(157248K), 0.0403336 secs] 243515K->153361K(506816K), 0.0403988 secs] [Times: user=0.02 sys=0.02, real=0.04 secs]
2020-10-26T22:16:34.414-0800: 0.382: [GC (Allocation Failure) 2020-10-26T22:16:34.414-0800: 0.382: [DefNew: 157245K->17470K(157248K), 0.0364800 secs] 293137K->193188K(506816K), 0.0365467 secs] [Times: user=0.02 sys=0.01, real=0.04 secs]
2020-10-26T22:16:34.477-0800: 0.444: [GC (Allocation Failure) 2020-10-26T22:16:34.477-0800: 0.444: [DefNew: 156908K->17471K(157248K), 0.0449685 secs] 332626K->242779K(506816K), 0.0451249 secs] [Times: user=0.02 sys=0.02, real=0.05 secs]
2020-10-26T22:16:34.550-0800: 0.518: [GC (Allocation Failure) 2020-10-26T22:16:34.550-0800: 0.518: [DefNew: 157247K->17471K(157248K), 0.0389673 secs] 382555K->288695K(506816K), 0.0390317 secs] [Times: user=0.03 sys=0.02, real=0.03 secs]
2020-10-26T22:16:34.617-0800: 0.584: [GC (Allocation Failure) 2020-10-26T22:16:34.617-0800: 0.585: [DefNew: 157247K->17471K(157248K), 0.0354948 secs] 428471K->332159K(506816K), 0.0355636 secs] [Times: user=0.03 sys=0.01, real=0.04 secs]
2020-10-26T22:16:34.676-0800: 0.644: [GC (Allocation Failure) 2020-10-26T22:16:34.676-0800: 0.644: [DefNew: 157247K->157247K(157248K), 0.0000357 secs]2020-10-26T22:16:34.676-0800: 0.644: [Tenured: 314687K->277678K(349568K), 0.0734037 secs] 471935K->277678K(506816K), [Metaspace: 2689K->2689K(1056768K)], 0.0735272 secs] [Times: user=0.07 sys=0.00, real=0.08 secs]
2020-10-26T22:16:34.788-0800: 0.755: [GC (Allocation Failure) 2020-10-26T22:16:34.788-0800: 0.755: [DefNew: 139776K->17471K(157248K), 0.0133521 secs] 417454K->325093K(506816K), 0.0134581 secs] [Times: user=0.01 sys=0.00, real=0.02 secs]
2020-10-26T22:16:34.833-0800: 0.800: [GC (Allocation Failure) 2020-10-26T22:16:34.833-0800: 0.800: [DefNew: 157247K->157247K(157248K), 0.0000314 secs]2020-10-26T22:16:34.833-0800: 0.800: [Tenured: 307621K->307464K(349568K), 0.0653442 secs] 464869K->307464K(506816K), [Metaspace: 2689K->2689K(1056768K)], 0.0654628 secs] [Times: user=0.06 sys=0.00, real=0.06 secs]
2020-10-26T22:16:34.928-0800: 0.896: [GC (Allocation Failure) 2020-10-26T22:16:34.928-0800: 0.896: [DefNew: 139776K->139776K(157248K), 0.0000328 secs]2020-10-26T22:16:34.928-0800: 0.896: [Tenured: 307464K->322195K(349568K), 0.0645899 secs] 447240K->322195K(506816K), [Metaspace: 2689K->2689K(1056768K)], 0.0647116 secs] [Times: user=0.06 sys=0.00, real=0.07 secs]
2020-10-26T22:16:35.022-0800: 0.989: [GC (Allocation Failure) 2020-10-26T22:16:35.022-0800: 0.989: [DefNew: 139386K->139386K(157248K), 0.0000488 secs]2020-10-26T22:16:35.022-0800: 0.990: [Tenured: 322195K->314828K(349568K), 0.0637727 secs] 461582K->314828K(506816K), [Metaspace: 2689K->2689K(1056768K)], 0.0639394 secs] [Times: user=0.07 sys=0.00, real=0.06 secs]
2020-10-26T22:16:35.109-0800: 1.076: [GC (Allocation Failure) 2020-10-26T22:16:35.109-0800: 1.076: [DefNew: 139776K->139776K(157248K), 0.0000494 secs]2020-10-26T22:16:35.109-0800: 1.076: [Tenured: 314828K->343459K(349568K), 0.0646202 secs] 454604K->343459K(506816K), [Metaspace: 2689K->2689K(1056768K)], 0.0647652 secs] [Times: user=0.06 sys=0.01, real=0.07 secs]
执行结束!共生成对象次数:6812
Heap
 def new generation   total 157248K, used 5654K [0x00000007a0000000, 0x00000007aaaa0000, 0x00000007aaaa0000)
  eden space 139776K,   4% used [0x00000007a0000000, 0x00000007a0585830, 0x00000007a8880000)
  from space 17472K,   0% used [0x00000007a8880000, 0x00000007a8880000, 0x00000007a9990000)
  to   space 17472K,   0% used [0x00000007a9990000, 0x00000007a9990000, 0x00000007aaaa0000)
 tenured generation   total 349568K, used 343459K [0x00000007aaaa0000, 0x00000007c0000000, 0x00000007c0000000)
   the space 349568K,  98% used [0x00000007aaaa0000, 0x00000007bfa08f30, 0x00000007bfa09000, 0x00000007c0000000)
 Metaspace       used 2695K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 295K, capacity 386K, committed 512K, reserved 1048576K
```

首先观察日志最后Heap部分数据

程序执行完成后，堆内存中

1. 年轻代，总容量157248K，使用5654K，
2. 老年代，总容量349568K，使用343459
3. 元数据区，总计使用了2695K 容量是4486K JVM保证可用的大小是4864K 保留空间1G左右


**Minor GC分析**

Minor GC事件的内容：

> 2020-10-26T22:16:34.202-0800: 0.169: [GC (Allocation Failure) 2020-10-26T22:16:34.202-0800: 0.169: [DefNew: 139450K->17472K(157248K), 0.0360407 secs] 139450K->54983K(506816K), 0.0361185 secs] [Times: user=0.02 sys=0.01, real=0.03 secs]

发生在JVM启动后的0.169秒，GC的原因是 `Allocation Failure` 分配内存失败，年轻代中没有空间存放新生成的对象。

DefNew 这个名字代表的是: *单线程(single-threaded), 采用标记复制(mark-copy)算法的, 使整个JVM暂停运行(stop-the-world)的年轻代(Young generation) 垃圾收集器(garbage collector)*

139450K->17472K 表示垃圾收集前后的年轻代使用量，157248K 表示年轻代的总空间大小，GC使年轻代的使用率从88% 降至 11%，减少 121978K

0.0360407 secs 表示GC的持续时间

139450K->54983K 表示垃圾收集前后堆内存的使用情况，506816K表示 堆内存的总可用空间大小，GC使整个堆的使用率从 27% 降至 11%，减少 84467K

0.0360407 secs 表示GC的持续时间，可以发现 GC完整耗时与GC在年轻代的耗时相同，说明JVM启动后首次GC耗时均发生在年轻代上

也可以比较垃圾收集前年轻代的使用量和整个堆的使用量，发现两者相同，说明在GC之前老年代中并没有对象。

GC前后堆内存的使用量比年轻代的使用量减少的要小，可以得出从年轻代提升至老年代的对象约为 121978K - 84467K = 37511K

Times: user=0.02 sys=0.01, real=0.03 secs 表示此次GC事件的持续时间，30ms

user表示所有GC线程消耗的CPU时间

sys表示系统调用和系统等待事件消耗的时间

real表示应用程序暂停的时间

在本次GC中，因为使用的是串行GC，且只有单线程，所以在这里 real = user + sys

**Full GC分析**

Full GC事件内容：

> 2020-10-26T22:16:34.676-0800: 0.644: [GC (Allocation Failure) 2020-10-26T22:16:34.676-0800: 0.644: [DefNew: 157247K->157247K(157248K), 0.0000357 secs]2020-10-26T22:16:34.676-0800: 0.644: [Tenured: 314687K->277678K(349568K), 0.0734037 secs] 471935K->277678K(506816K), [Metaspace: 2689K->2689K(1056768K)], 0.0735272 secs] [Times: user=0.07 sys=0.00, real=0.08 secs]

2020-10-26T22:16:34.676-0800: 0.644: [DefNew: 157247K->157247K(157248K), 0.0000357 secs]

内存分配失败导致了一次年轻代GC，但是此次年轻代GC堆内存的使用量没有变化，使用量达到99.99%，年轻代几乎用满，且耗时几乎为0

2020-10-26T22:16:34.676-0800: 0.644: [Tenured: 314687K->277678K(349568K), 0.0734037 secs]

Tenured 代表清理老年代使用的垃圾收集器名称，单线程的STW垃圾收集器，使用的算法为 标记‐清除‐整理(mark‐sweep‐ compact ) 。

GC前后老年的使用量从314687K降至277678K，总的老年代空间大小为349568K

GC之后，老年代的堆内存使用量下降37009K 使用率从90% 降至 79%

471935K->277678K(506816K) 表示堆内存在GC前后的使用量，使用量下降 194257 使用率从93% 下降至 55%

这次Full GC之后老年代的使用量和整个堆的使用量相同，那么是否是年轻代的使用量在Tenured之后变为了0，只是日志体现的是Tenured之前的年轻代GC，所以在日志中没有体现？？？

[Metaspace: 2689K->2689K(1056768K)]

元数据库内存使用没有变化

[Times: user=0.07 sys=0.00, real=0.08 secs]

此次GC合计耗时80ms

#### 2.2.2 使用128m堆内存

```shell
java -XX:+UseSerialGC -Xms128m -Xmx128m -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps GCLogAnalysis
正在执行...
2020-10-27T11:12:39.633-0800: 0.104: [GC (Allocation Failure) 2020-10-27T11:12:39.633-0800: 0.105: [DefNew: 34944K->4351K(39296K), 0.0091493 secs] 34944K->12494K(126720K), 0.0092266 secs] [Times: user=0.01 sys=0.01, real=0.01 secs]
2020-10-27T11:12:39.658-0800: 0.130: [GC (Allocation Failure) 2020-10-27T11:12:39.658-0800: 0.130: [DefNew: 39284K->4349K(39296K), 0.0170365 secs] 47427K->25816K(126720K), 0.0171250 secs] [Times: user=0.03 sys=0.01, real=0.02 secs]
2020-10-27T11:12:39.690-0800: 0.162: [GC (Allocation Failure) 2020-10-27T11:12:39.690-0800: 0.162: [DefNew: 39277K->4351K(39296K), 0.0128654 secs] 60744K->39193K(126720K), 0.0129536 secs] [Times: user=0.01 sys=0.01, real=0.01 secs]
2020-10-27T11:12:39.718-0800: 0.189: [GC (Allocation Failure) 2020-10-27T11:12:39.718-0800: 0.189: [DefNew: 39295K->4334K(39296K), 0.0135349 secs] 74137K->51634K(126720K), 0.0136381 secs] [Times: user=0.02 sys=0.00, real=0.02 secs]
2020-10-27T11:12:39.738-0800: 0.210: [GC (Allocation Failure) 2020-10-27T11:12:39.738-0800: 0.210: [DefNew: 39278K->4350K(39296K), 0.0100082 secs] 86578K->65081K(126720K), 0.0100772 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
2020-10-27T11:12:39.763-0800: 0.235: [GC (Allocation Failure) 2020-10-27T11:12:39.763-0800: 0.235: [DefNew: 39279K->4346K(39296K), 0.0146205 secs] 100010K->77928K(126720K), 0.0147487 secs] [Times: user=0.02 sys=0.01, real=0.01 secs]
2020-10-27T11:12:39.784-0800: 0.256: [GC (Allocation Failure) 2020-10-27T11:12:39.784-0800: 0.256: [DefNew: 39290K->39290K(39296K), 0.0000325 secs]2020-10-27T11:12:39.784-0800: 0.256: [Tenured: 73582K->83357K(87424K), 0.0211825 secs] 112872K->83357K(126720K), [Metaspace: 2689K->2689K(1056768K)], 0.0213299 secs] [Times: user=0.01 sys=0.00, real=0.02 secs]
2020-10-27T11:12:39.816-0800: 0.288: [GC (Allocation Failure) 2020-10-27T11:12:39.816-0800: 0.288: [DefNew: 34229K->34229K(39296K), 0.0000539 secs]2020-10-27T11:12:39.816-0800: 0.288: [Tenured: 83357K->87278K(87424K), 0.0296567 secs] 117586K->92405K(126720K), [Metaspace: 2689K->2689K(1056768K)], 0.0298485 secs] [Times: user=0.02 sys=0.00, real=0.03 secs]
2020-10-27T11:12:39.859-0800: 0.331: [Full GC (Allocation Failure) 2020-10-27T11:12:39.859-0800: 0.331: [Tenured: 87278K->87113K(87424K), 0.0341419 secs] 126121K->101878K(126720K), [Metaspace: 2689K->2689K(1056768K)], 0.0342425 secs] [Times: user=0.03 sys=0.00, real=0.04 secs]
2020-10-27T11:12:39.899-0800: 0.371: [Full GC (Allocation Failure) 2020-10-27T11:12:39.899-0800: 0.371: [Tenured: 87113K->87406K(87424K), 0.0270719 secs] 126021K->106166K(126720K), [Metaspace: 2689K->2689K(1056768K)], 0.0272392 secs] [Times: user=0.02 sys=0.01, real=0.03 secs]
2020-10-27T11:12:39.930-0800: 0.402: [Full GC (Allocation Failure) 2020-10-27T11:12:39.930-0800: 0.402: [Tenured: 87406K->87406K(87424K), 0.0034222 secs] 126628K->111789K(126720K), [Metaspace: 2689K->2689K(1056768K)], 0.0034929 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-27T11:12:39.935-0800: 0.407: [Full GC (Allocation Failure) 2020-10-27T11:12:39.935-0800: 0.407: [Tenured: 87406K->87406K(87424K), 0.0048305 secs] 126554K->115996K(126720K), [Metaspace: 2689K->2689K(1056768K)], 0.0049248 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
2020-10-27T11:12:39.942-0800: 0.414: [Full GC (Allocation Failure) 2020-10-27T11:12:39.942-0800: 0.414: [Tenured: 87406K->87245K(87424K), 0.0093468 secs] 126700K->120097K(126720K), [Metaspace: 2689K->2689K(1056768K)], 0.0094152 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
2020-10-27T11:12:39.954-0800: 0.425: [Full GC (Allocation Failure) 2020-10-27T11:12:39.954-0800: 0.425: [Tenured: 87421K->87301K(87424K), 0.0229400 secs] 126708K->117971K(126720K), [Metaspace: 2689K->2689K(1056768K)], 0.0230235 secs] [Times: user=0.02 sys=0.00, real=0.02 secs]
2020-10-27T11:12:39.978-0800: 0.450: [Full GC (Allocation Failure) 2020-10-27T11:12:39.978-0800: 0.450: [Tenured: 87301K->87301K(87424K), 0.0073974 secs] 126286K->119722K(126720K), [Metaspace: 2689K->2689K(1056768K)], 0.0074798 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
2020-10-27T11:12:39.987-0800: 0.459: [Full GC (Allocation Failure) 2020-10-27T11:12:39.987-0800: 0.459: [Tenured: 87301K->87301K(87424K), 0.0048841 secs] 126309K->121072K(126720K), [Metaspace: 2689K->2689K(1056768K)], 0.0049778 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
2020-10-27T11:12:39.993-0800: 0.465: [Full GC (Allocation Failure) 2020-10-27T11:12:39.993-0800: 0.465: [Tenured: 87301K->87301K(87424K), 0.0048639 secs] 126484K->122840K(126720K), [Metaspace: 2689K->2689K(1056768K)], 0.0049178 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-27T11:12:39.998-0800: 0.470: [Full GC (Allocation Failure) 2020-10-27T11:12:39.999-0800: 0.470: [Tenured: 87301K->86827K(87424K), 0.0204955 secs] 126581K->123367K(126720K), [Metaspace: 2689K->2689K(1056768K)], 0.0206002 secs] [Times: user=0.02 sys=0.00, real=0.02 secs]
2020-10-27T11:12:40.021-0800: 0.493: [Full GC (Allocation Failure) 2020-10-27T11:12:40.021-0800: 0.493: [Tenured: 87280K->87280K(87424K), 0.0027976 secs] 126481K->124145K(126720K), [Metaspace: 2689K->2689K(1056768K)], 0.0028881 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-27T11:12:40.025-0800: 0.497: [Full GC (Allocation Failure) 2020-10-27T11:12:40.025-0800: 0.497: [Tenured: 87280K->87280K(87424K), 0.0024275 secs] 126321K->125053K(126720K), [Metaspace: 2689K->2689K(1056768K)], 0.0025057 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-27T11:12:40.028-0800: 0.499: [Full GC (Allocation Failure) 2020-10-27T11:12:40.028-0800: 0.499: [Tenured: 87280K->87280K(87424K), 0.0013018 secs] 126417K->125676K(126720K), [Metaspace: 2689K->2689K(1056768K)], 0.0013524 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-27T11:12:40.029-0800: 0.501: [Full GC (Allocation Failure) 2020-10-27T11:12:40.029-0800: 0.501: [Tenured: 87280K->86936K(87424K), 0.0148080 secs] 126435K->125631K(126720K), [Metaspace: 2689K->2689K(1056768K)], 0.0148632 secs] [Times: user=0.02 sys=0.00, real=0.02 secs]
2020-10-27T11:12:40.044-0800: 0.516: [Full GC (Allocation Failure) 2020-10-27T11:12:40.044-0800: 0.516: [Tenured: 87354K->87354K(87424K), 0.0016582 secs] 126597K->126044K(126720K), [Metaspace: 2689K->2689K(1056768K)], 0.0017080 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-27T11:12:40.046-0800: 0.518: [Full GC (Allocation Failure) 2020-10-27T11:12:40.046-0800: 0.518: [Tenured: 87354K->87354K(87424K), 0.0012578 secs] 126199K->126044K(126720K), [Metaspace: 2689K->2689K(1056768K)], 0.0013884 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-27T11:12:40.048-0800: 0.520: [Full GC (Allocation Failure) 2020-10-27T11:12:40.048-0800: 0.520: [Tenured: 87354K->87221K(87424K), 0.0224445 secs] 126044K->125911K(126720K), [Metaspace: 2689K->2689K(1056768K)], 0.0225360 secs] [Times: user=0.02 sys=0.00, real=0.03 secs]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at GCLogAnalysis.generateGarbage(GCLogAnalysis.java:48)
	at GCLogAnalysis.main(GCLogAnalysis.java:25)
Heap
 def new generation   total 39296K, used 38914K [0x00000007b8000000, 0x00000007baaa0000, 0x00000007baaa0000)
  eden space 34944K, 100% used [0x00000007b8000000, 0x00000007ba220000, 0x00000007ba220000)
  from space 4352K,  91% used [0x00000007ba220000, 0x00000007ba600b78, 0x00000007ba660000)
  to   space 4352K,   0% used [0x00000007ba660000, 0x00000007ba660000, 0x00000007baaa0000)
 tenured generation   total 87424K, used 87221K [0x00000007baaa0000, 0x00000007c0000000, 0x00000007c0000000)
   the space 87424K,  99% used [0x00000007baaa0000, 0x00000007bffcd648, 0x00000007bffcd800, 0x00000007c0000000)
 Metaspace       used 2719K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 297K, capacity 386K, committed 512K, reserved 1048576K
```

可以看出程序执行导致了堆内存溢出 触发了 `java.lang.OutOfMemoryError`

1. 年轻代 总容量39296K 使用38914K Eden区使用率达到100%
2. 老年代 使用率99%
3. 堆内存几乎用满，无法再分配对象尝试Full GC失败 导致OOM

首先触发了6次Minor GC，清理年轻代

随后发生了两次 GC （未标明是Full GC）,既清理了年轻代，也清理了老年代和元数据区，只是在几乎未对年轻代清理的情况下 老年代使用量竟让上涨了。


#### 2.2.3 使用1g堆内存

```shell
java -XX:+UseSerialGC -Xms1g -Xmx1g -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps GCLogAnalysis
正在执行...
2020-10-27T21:11:18.890-0800: 0.250: [GC (Allocation Failure) 2020-10-27T21:11:18.890-0800: 0.250: [DefNew: 279616K->34944K(314560K), 0.0496792 secs] 279616K->82661K(1013632K), 0.0497561 secs] [Times: user=0.03 sys=0.02, real=0.04 secs]
2020-10-27T21:11:18.993-0800: 0.354: [GC (Allocation Failure) 2020-10-27T21:11:18.993-0800: 0.354: [DefNew: 314560K->34943K(314560K), 0.0700488 secs] 362277K->160948K(1013632K), 0.0701214 secs] [Times: user=0.04 sys=0.03, real=0.07 secs]
2020-10-27T21:11:19.111-0800: 0.472: [GC (Allocation Failure) 2020-10-27T21:11:19.112-0800: 0.472: [DefNew: 314559K->34943K(314560K), 0.0596861 secs] 440564K->242355K(1013632K), 0.0597577 secs] [Times: user=0.03 sys=0.02, real=0.06 secs]
2020-10-27T21:11:19.220-0800: 0.580: [GC (Allocation Failure) 2020-10-27T21:11:19.220-0800: 0.580: [DefNew: 314310K->34943K(314560K), 0.0541097 secs] 521721K->318969K(1013632K), 0.0541829 secs] [Times: user=0.04 sys=0.02, real=0.05 secs]
2020-10-27T21:11:19.321-0800: 0.682: [GC (Allocation Failure) 2020-10-27T21:11:19.321-0800: 0.682: [DefNew: 314262K->34942K(314560K), 0.0684594 secs] 598287K->406393K(1013632K), 0.0685505 secs] [Times: user=0.04 sys=0.03, real=0.07 secs]
2020-10-27T21:11:19.444-0800: 0.805: [GC (Allocation Failure) 2020-10-27T21:11:19.444-0800: 0.805: [DefNew: 314558K->34943K(314560K), 0.0710469 secs] 686009K->491481K(1013632K), 0.0711163 secs] [Times: user=0.05 sys=0.03, real=0.07 secs]
2020-10-27T21:11:19.566-0800: 0.927: [GC (Allocation Failure) 2020-10-27T21:11:19.566-0800: 0.927: [DefNew: 314559K->34943K(314560K), 0.0546956 secs] 771097K->562797K(1013632K), 0.0547539 secs] [Times: user=0.03 sys=0.02, real=0.06 secs]
2020-10-27T21:11:19.669-0800: 1.029: [GC (Allocation Failure) 2020-10-27T21:11:19.669-0800: 1.030: [DefNew: 314559K->34943K(314560K), 0.0573799 secs] 842413K->644270K(1013632K), 0.0574553 secs] [Times: user=0.04 sys=0.02, real=0.06 secs]
执行结束!共生成对象次数:8765
Heap
 def new generation   total 314560K, used 85051K [0x0000000780000000, 0x0000000795550000, 0x0000000795550000)
  eden space 279616K,  17% used [0x0000000780000000, 0x00000007830eefc8, 0x0000000791110000)
  from space 34944K,  99% used [0x0000000791110000, 0x000000079332fff0, 0x0000000793330000)
  to   space 34944K,   0% used [0x0000000793330000, 0x0000000793330000, 0x0000000795550000)
 tenured generation   total 699072K, used 609326K [0x0000000795550000, 0x00000007c0000000, 0x00000007c0000000)
   the space 699072K,  87% used [0x0000000795550000, 0x00000007ba85b978, 0x00000007ba85ba00, 0x00000007c0000000)
 Metaspace       used 2695K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 295K, capacity 386K, committed 512K, reserved 1048576K
```

使用1g堆内存后，共发生8次Minor GC，未发生老年代GC，GC时间从50ms到70ms不等。

随着堆内存的配置越来越大，程序产生的对象也随之增多。

#### 2.2.4 使用2g堆内存

```shell
java -XX:+UseSerialGC -Xms2g -Xmx2g -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps GCLogAnalysis
正在执行...
2020-10-27T21:23:03.643-0800: 0.387: [GC (Allocation Failure) 2020-10-27T21:23:03.643-0800: 0.387: [DefNew: 559232K->69888K(629120K), 0.0887940 secs] 559232K->150907K(2027264K), 0.0888694 secs] [Times: user=0.06 sys=0.04, real=0.09 secs]
2020-10-27T21:23:03.840-0800: 0.584: [GC (Allocation Failure) 2020-10-27T21:23:03.840-0800: 0.584: [DefNew: 629120K->69887K(629120K), 0.1088363 secs] 710139K->272051K(2027264K), 0.1089175 secs] [Times: user=0.06 sys=0.05, real=0.10 secs]
2020-10-27T21:23:04.054-0800: 0.799: [GC (Allocation Failure) 2020-10-27T21:23:04.054-0800: 0.799: [DefNew: 629119K->69888K(629120K), 0.0948227 secs] 831283K->400122K(2027264K), 0.0948869 secs] [Times: user=0.06 sys=0.03, real=0.09 secs]
2020-10-27T21:23:04.243-0800: 0.988: [GC (Allocation Failure) 2020-10-27T21:23:04.243-0800: 0.988: [DefNew: 629120K->69887K(629120K), 0.0842347 secs] 959354K->522406K(2027264K), 0.0843166 secs] [Times: user=0.05 sys=0.03, real=0.08 secs]
执行结束!共生成对象次数:8619
Heap
 def new generation   total 629120K, used 137193K [0x0000000740000000, 0x000000076aaa0000, 0x000000076aaa0000)
  eden space 559232K,  12% used [0x0000000740000000, 0x00000007441ba4c8, 0x0000000762220000)
  from space 69888K,  99% used [0x0000000762220000, 0x000000076665fff0, 0x0000000766660000)
  to   space 69888K,   0% used [0x0000000766660000, 0x0000000766660000, 0x000000076aaa0000)
 tenured generation   total 1398144K, used 452518K [0x000000076aaa0000, 0x00000007c0000000, 0x00000007c0000000)
   the space 1398144K,  32% used [0x000000076aaa0000, 0x0000000786489ad0, 0x0000000786489c00, 0x00000007c0000000)
 Metaspace       used 2695K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 295K, capacity 386K, committed 512K, reserved 1048576K
```

这次只发生了4次Minor GC 耗时在80到100ms之间，可见耗时比堆内存为1g时要多

程序生成的对象却没有明显地增多

#### 2.2.5 使用4g堆内存

```shell
java -XX:+UseSerialGC -Xms4g -Xmx4g -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps GCLogAnalysis
正在执行...
2020-10-27T21:42:05.545-0800: 0.673: [GC (Allocation Failure) 2020-10-27T21:42:05.545-0800: 0.673: [DefNew: 1118528K->139775K(1258304K), 0.1543860 secs] 1118528K->243637K(4054528K), 0.1544662 secs] [Times: user=0.09 sys=0.07, real=0.15 secs]
2020-10-27T21:42:05.906-0800: 1.035: [GC (Allocation Failure) 2020-10-27T21:42:05.906-0800: 1.035: [DefNew: 1258303K->139775K(1258304K), 0.1673658 secs] 1362165K->399506K(4054528K), 0.1674364 secs] [Times: user=0.09 sys=0.07, real=0.17 secs]
执行结束!共生成对象次数:8370
Heap
 def new generation   total 1258304K, used 184640K [0x00000006c0000000, 0x0000000715550000, 0x0000000715550000)
  eden space 1118528K,   4% used [0x00000006c0000000, 0x00000006c2bd0210, 0x0000000704450000)
  from space 139776K,  99% used [0x0000000704450000, 0x000000070cccfff8, 0x000000070ccd0000)
  to   space 139776K,   0% used [0x000000070ccd0000, 0x000000070ccd0000, 0x0000000715550000)
 tenured generation   total 2796224K, used 259730K [0x0000000715550000, 0x00000007c0000000, 0x00000007c0000000)
   the space 2796224K,   9% used [0x0000000715550000, 0x00000007252f4b58, 0x00000007252f4c00, 0x00000007c0000000)
 Metaspace       used 2695K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 295K, capacity 386K, committed 512K, reserved 1048576K
```

GC执行时间边长，但是生成的对象并没有明显增长

#### 2.2.6 串行GC总结

1. 使用`-XX:+UseSerialGC`指定GC算法
2. 对年轻代使用 “标记-复制”算法
3. 对老年代使用“标记-清除-整理”算法
4. 单线程，所以对年轻代和老年代的处理都会"stop the world"，停止应用线程进行GC
5. 不能使用多核CPU，暂停时间也更长
6. 优点：CPU的利用率较高

### 2.3 CMS GC的演示

#### 2.3.1 使用512m堆内存配置

```shell
java -XX:+UseConcMarkSweepGC -Xms512m -Xmx512m -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps GCLogAnalysis
正在执行...
2020-10-27T22:39:39.458-0800: 0.169: [GC (Allocation Failure) 2020-10-27T22:39:39.458-0800: 0.169: [ParNew: 139776K->17470K(157248K), 0.0213819 secs] 139776K->48287K(506816K), 0.0214863 secs] [Times: user=0.04 sys=0.10, real=0.03 secs]
2020-10-27T22:39:39.508-0800: 0.219: [GC (Allocation Failure) 2020-10-27T22:39:39.508-0800: 0.219: [ParNew: 157246K->17472K(157248K), 0.0333864 secs] 188063K->94437K(506816K), 0.0334829 secs] [Times: user=0.08 sys=0.14, real=0.04 secs]
2020-10-27T22:39:39.575-0800: 0.286: [GC (Allocation Failure) 2020-10-27T22:39:39.575-0800: 0.286: [ParNew: 157248K->17472K(157248K), 0.0423986 secs] 234213K->145799K(506816K), 0.0424847 secs] [Times: user=0.29 sys=0.02, real=0.04 secs]
2020-10-27T22:39:39.639-0800: 0.350: [GC (Allocation Failure) 2020-10-27T22:39:39.640-0800: 0.351: [ParNew: 156619K->17470K(157248K), 0.0376706 secs] 284947K->183380K(506816K), 0.0377727 secs] [Times: user=0.25 sys=0.03, real=0.04 secs]
2020-10-27T22:39:39.709-0800: 0.420: [GC (Allocation Failure) 2020-10-27T22:39:39.709-0800: 0.420: [ParNew: 157246K->17468K(157248K), 0.0369231 secs] 323156K->226601K(506816K), 0.0370328 secs] [Times: user=0.26 sys=0.02, real=0.04 secs]
2020-10-27T22:39:39.746-0800: 0.457: [GC (CMS Initial Mark) [1 CMS-initial-mark: 209132K(349568K)] 226891K(506816K), 0.0004101 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-27T22:39:39.746-0800: 0.457: [CMS-concurrent-mark-start]
2020-10-27T22:39:39.748-0800: 0.459: [CMS-concurrent-mark: 0.002/0.002 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-27T22:39:39.748-0800: 0.459: [CMS-concurrent-preclean-start]
2020-10-27T22:39:39.749-0800: 0.460: [CMS-concurrent-preclean: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-27T22:39:39.749-0800: 0.460: [CMS-concurrent-abortable-preclean-start]
2020-10-27T22:39:39.768-0800: 0.479: [GC (Allocation Failure) 2020-10-27T22:39:39.768-0800: 0.479: [ParNew: 157244K->17470K(157248K), 0.0398063 secs] 366377K->268186K(506816K), 0.0398829 secs] [Times: user=0.26 sys=0.03, real=0.04 secs]
2020-10-27T22:39:39.840-0800: 0.551: [GC (Allocation Failure) 2020-10-27T22:39:39.840-0800: 0.551: [ParNew: 157228K->17471K(157248K), 0.0392140 secs] 407944K->312914K(506816K), 0.0393009 secs] [Times: user=0.27 sys=0.03, real=0.03 secs]
2020-10-27T22:39:39.900-0800: 0.611: [GC (Allocation Failure) 2020-10-27T22:39:39.900-0800: 0.611: [ParNew: 157247K->17471K(157248K), 0.0450912 secs] 452690K->363166K(506816K), 0.0451946 secs] [Times: user=0.31 sys=0.03, real=0.04 secs]
2020-10-27T22:39:39.945-0800: 0.656: [CMS-concurrent-abortable-preclean: 0.003/0.197 secs] [Times: user=0.93 sys=0.09, real=0.20 secs]
2020-10-27T22:39:39.946-0800: 0.657: [GC (CMS Final Remark) [YG occupancy: 20360 K (157248 K)]2020-10-27T22:39:39.946-0800: 0.657: [Rescan (parallel) , 0.0004848 secs]2020-10-27T22:39:39.946-0800: 0.657: [weak refs processing, 0.0000389 secs]2020-10-27T22:39:39.946-0800: 0.657: [class unloading, 0.0003198 secs]2020-10-27T22:39:39.946-0800: 0.657: [scrub symbol table, 0.0004675 secs]2020-10-27T22:39:39.947-0800: 0.658: [scrub string table, 0.0001561 secs][1 CMS-remark: 345694K(349568K)] 366054K(506816K), 0.0015994 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-27T22:39:39.947-0800: 0.658: [CMS-concurrent-sweep-start]
2020-10-27T22:39:39.948-0800: 0.659: [CMS-concurrent-sweep: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-27T22:39:39.948-0800: 0.659: [CMS-concurrent-reset-start]
2020-10-27T22:39:39.949-0800: 0.660: [CMS-concurrent-reset: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-27T22:39:39.980-0800: 0.691: [GC (Allocation Failure) 2020-10-27T22:39:39.980-0800: 0.691: [ParNew: 156971K->156971K(157248K), 0.0000456 secs]2020-10-27T22:39:39.980-0800: 0.691: [CMS: 295961K->257240K(349568K), 0.0866539 secs] 452933K->257240K(506816K), [Metaspace: 2689K->2689K(1056768K)], 0.0868169 secs] [Times: user=0.08 sys=0.00, real=0.08 secs]
2020-10-27T22:39:40.067-0800: 0.778: [GC (CMS Initial Mark) [1 CMS-initial-mark: 257240K(349568K)] 257575K(506816K), 0.0002127 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-27T22:39:40.067-0800: 0.778: [CMS-concurrent-mark-start]
2020-10-27T22:39:40.068-0800: 0.779: [CMS-concurrent-mark: 0.001/0.001 secs] [Times: user=0.01 sys=0.00, real=0.00 secs]
2020-10-27T22:39:40.068-0800: 0.779: [CMS-concurrent-preclean-start]
2020-10-27T22:39:40.069-0800: 0.780: [CMS-concurrent-preclean: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-27T22:39:40.069-0800: 0.780: [CMS-concurrent-abortable-preclean-start]
2020-10-27T22:39:40.099-0800: 0.810: [GC (Allocation Failure) 2020-10-27T22:39:40.099-0800: 0.810: [ParNew: 139308K->17471K(157248K), 0.0086479 secs] 396548K->302259K(506816K), 0.0087297 secs] [Times: user=0.05 sys=0.00, real=0.01 secs]
2020-10-27T22:39:40.146-0800: 0.857: [GC (Allocation Failure) 2020-10-27T22:39:40.146-0800: 0.857: [ParNew: 157247K->17465K(157248K), 0.0211116 secs] 442035K->341391K(506816K), 0.0212073 secs] [Times: user=0.15 sys=0.01, real=0.02 secs]
2020-10-27T22:39:40.202-0800: 0.913: [GC (Allocation Failure) 2020-10-27T22:39:40.202-0800: 0.913: [ParNew: 156731K->156731K(157248K), 0.0000506 secs]2020-10-27T22:39:40.202-0800: 0.913: [CMS2020-10-27T22:39:40.202-0800: 0.913: [CMS-concurrent-abortable-preclean: 0.003/0.133 secs] [Times: user=0.31 sys=0.01, real=0.14 secs]
 (concurrent mode failure): 323925K->294545K(349568K), 0.0699941 secs] 480657K->294545K(506816K), [Metaspace: 2689K->2689K(1056768K)], 0.0701788 secs] [Times: user=0.07 sys=0.00, real=0.07 secs]
2020-10-27T22:39:40.294-0800: 1.005: [GC (Allocation Failure) 2020-10-27T22:39:40.294-0800: 1.005: [ParNew: 139776K->17468K(157248K), 0.0181598 secs] 434321K->336614K(506816K), 0.0182725 secs] [Times: user=0.13 sys=0.00, real=0.02 secs]
2020-10-27T22:39:40.313-0800: 1.024: [GC (CMS Initial Mark) [1 CMS-initial-mark: 319146K(349568K)] 336902K(506816K), 0.0002534 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-27T22:39:40.313-0800: 1.024: [CMS-concurrent-mark-start]
2020-10-27T22:39:40.314-0800: 1.025: [CMS-concurrent-mark: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-27T22:39:40.315-0800: 1.026: [CMS-concurrent-preclean-start]
2020-10-27T22:39:40.315-0800: 1.026: [CMS-concurrent-preclean: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-27T22:39:40.315-0800: 1.026: [CMS-concurrent-abortable-preclean-start]
2020-10-27T22:39:40.346-0800: 1.058: [GC (Allocation Failure) 2020-10-27T22:39:40.347-0800: 1.058: [ParNew: 157244K->157244K(157248K), 0.0000761 secs]2020-10-27T22:39:40.347-0800: 1.058: [CMS2020-10-27T22:39:40.347-0800: 1.058: [CMS-concurrent-abortable-preclean: 0.001/0.032 secs] [Times: user=0.03 sys=0.00, real=0.03 secs]
 (concurrent mode failure): 319146K->312120K(349568K), 0.0707347 secs] 476390K->312120K(506816K), [Metaspace: 2689K->2689K(1056768K)], 0.0709816 secs] [Times: user=0.07 sys=0.00, real=0.07 secs]
执行结束!共生成对象次数:7363
Heap
 par new generation   total 157248K, used 5677K [0x00000007a0000000, 0x00000007aaaa0000, 0x00000007aaaa0000)
  eden space 139776K,   4% used [0x00000007a0000000, 0x00000007a058b560, 0x00000007a8880000)
  from space 17472K,   0% used [0x00000007a9990000, 0x00000007a9990000, 0x00000007aaaa0000)
  to   space 17472K,   0% used [0x00000007a8880000, 0x00000007a8880000, 0x00000007a9990000)
 concurrent mark-sweep generation total 349568K, used 312120K [0x00000007aaaa0000, 0x00000007c0000000, 0x00000007c0000000)
 Metaspace       used 2695K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 295K, capacity 386K, committed 512K, reserved 1048576K
```

> 2020-10-27T22:39:39.458-0800: 0.169: [GC (Allocation Failure) 2020-10-27T22:39:39.458-0800: 0.169: [ParNew: 139776K->17470K(157248K), 0.0213819 secs] 139776K->48287K(506816K), 0.0214863 secs] [Times: user=0.04 sys=0.10, real=0.03 secs]

第一次年轻代GC

使用的GC算法是 ParNew算法

> 2020-10-27T22:39:39.746-0800: 0.457: [GC (CMS Initial Mark) [1 CMS-initial-mark: 209132K(349568K)] 226891K(506816K), 0.0004101 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
>
> 2020-10-27T22:39:39.746-0800: 0.457: [CMS-concurrent-mark-start]
>
> 2020-10-27T22:39:39.748-0800: 0.459: [CMS-concurrent-mark: 0.002/0.002 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
>
> 2020-10-27T22:39:39.748-0800: 0.459: [CMS-concurrent-preclean-start]
>
> 2020-10-27T22:39:39.749-0800: 0.460: [CMS-concurrent-preclean: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
>
> 2020-10-27T22:39:39.749-0800: 0.460: [CMS-concurrent-abortable-preclean-start]
>
> 2020-10-27T22:39:39.945-0800: 0.656: [CMS-concurrent-abortable-preclean: 0.003/0.197 secs] [Times: user=0.93 sys=0.09, real=0.20 secs]
>
> 2020-10-27T22:39:39.946-0800: 0.657: [GC (CMS Final Remark) [YG occupancy: 20360 K (157248 K)]2020-10-27T22:39:39.946-0800: 0.657: [Rescan (parallel) , 0.0004848 secs]2020-10-27T22:39:39.946-0800: 0.657: [weak refs processing, 0.0000389 secs]2020-10-27T22:39:39.946-0800: 0.657: [class unloading, 0.0003198 secs]2020-10-27T22:39:39.946-0800: 0.657: [scrub symbol table, 0.0004675 secs]2020-10-27T22:39:39.947-0800: 0.658: [scrub string table, 0.0001561 secs][1 CMS-remark: 345694K(349568K)] 366054K(506816K), 0.0015994 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
>
> 2020-10-27T22:39:39.947-0800: 0.658: [CMS-concurrent-sweep-start]
>
> 2020-10-27T22:39:39.948-0800: 0.659: [CMS-concurrent-sweep: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
>
> 2020-10-27T22:39:39.948-0800: 0.659: [CMS-concurrent-reset-start]
>
> 2020-10-27T22:39:39.949-0800: 0.660: [CMS-concurrent-reset: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]

第一次 CMS GC，可以看出CMS GC共有6个步骤

1. CMS Initial Mark 初始标记
2. CMS-concurrent-mark 并发标记
3. CMS-concurrent-preclean 并发预清理
4. CMS Final Remark 最终标记
5. CMS-concurrent-sweep 并发清除
6. CMS-concurrent-reset 并发重置

其中初始标记和最终标记阶段没有concurrent，表示在此期间将会发生STW，停止业务线程

其余阶段均可以与业务线程并发执行，此时默认的执行GC的线程数为机器CPU核心数的1/4。


#### 2.3.2 使用1g堆内存

```shell
java -XX:+UseConcMarkSweepGC -Xms1g -Xmx1g -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps GCLogAnalysis
正在执行...
2020-10-28T21:29:26.158-0800: 0.280: [GC (Allocation Failure) 2020-10-28T21:29:26.158-0800: 0.281: [ParNew: 279616K->34944K(314560K), 0.0431802 secs] 279616K->90159K(1013632K), 0.0432705 secs] [Times: user=0.07 sys=0.20, real=0.05 secs]
2020-10-28T21:29:26.261-0800: 0.383: [GC (Allocation Failure) 2020-10-28T21:29:26.261-0800: 0.383: [ParNew: 314560K->34942K(314560K), 0.0436206 secs] 369775K->165049K(1013632K), 0.0436886 secs] [Times: user=0.09 sys=0.20, real=0.04 secs]
2020-10-28T21:29:26.360-0800: 0.482: [GC (Allocation Failure) 2020-10-28T21:29:26.360-0800: 0.482: [ParNew: 314558K->34944K(314560K), 0.0678416 secs] 444665K->250958K(1013632K), 0.0679174 secs] [Times: user=0.47 sys=0.04, real=0.06 secs]
2020-10-28T21:29:26.479-0800: 0.601: [GC (Allocation Failure) 2020-10-28T21:29:26.479-0800: 0.601: [ParNew: 314560K->34944K(314560K), 0.0656349 secs] 530574K->333860K(1013632K), 0.0657094 secs] [Times: user=0.45 sys=0.05, real=0.07 secs]
2020-10-28T21:29:26.595-0800: 0.717: [GC (Allocation Failure) 2020-10-28T21:29:26.595-0800: 0.717: [ParNew: 314560K->34942K(314560K), 0.0616112 secs] 613476K->405206K(1013632K), 0.0616833 secs] [Times: user=0.43 sys=0.04, real=0.06 secs]
2020-10-28T21:29:26.657-0800: 0.779: [GC (CMS Initial Mark) [1 CMS-initial-mark: 370263K(699072K)] 410753K(1013632K), 0.0003411 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-28T21:29:26.657-0800: 0.780: [CMS-concurrent-mark-start]
2020-10-28T21:29:26.661-0800: 0.783: [CMS-concurrent-mark: 0.003/0.003 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
2020-10-28T21:29:26.661-0800: 0.783: [CMS-concurrent-preclean-start]
2020-10-28T21:29:26.662-0800: 0.784: [CMS-concurrent-preclean: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-28T21:29:26.662-0800: 0.784: [CMS-concurrent-abortable-preclean-start]
2020-10-28T21:29:26.718-0800: 0.840: [GC (Allocation Failure) 2020-10-28T21:29:26.718-0800: 0.840: [ParNew2020-10-28T21:29:26.767-0800: 0.889: [CMS-concurrent-abortable-preclean: 0.001/0.105 secs] [Times: user=0.39 sys=0.03, real=0.10 secs]
: 314558K->34944K(314560K), 0.0699358 secs] 684822K->482118K(1013632K), 0.0700242 secs] [Times: user=0.47 sys=0.05, real=0.07 secs]
2020-10-28T21:29:26.788-0800: 0.911: [GC (CMS Final Remark) [YG occupancy: 35424 K (314560 K)]2020-10-28T21:29:26.788-0800: 0.911: [Rescan (parallel) , 0.0005293 secs]2020-10-28T21:29:26.789-0800: 0.911: [weak refs processing, 0.0000367 secs]2020-10-28T21:29:26.789-0800: 0.911: [class unloading, 0.0002643 secs]2020-10-28T21:29:26.789-0800: 0.911: [scrub symbol table, 0.0004242 secs]2020-10-28T21:29:26.790-0800: 0.912: [scrub string table, 0.0002150 secs][1 CMS-remark: 447174K(699072K)] 482598K(1013632K), 0.0016199 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
2020-10-28T21:29:26.790-0800: 0.912: [CMS-concurrent-sweep-start]
2020-10-28T21:29:26.791-0800: 0.913: [CMS-concurrent-sweep: 0.001/0.001 secs] [Times: user=0.01 sys=0.00, real=0.00 secs]
2020-10-28T21:29:26.791-0800: 0.913: [CMS-concurrent-reset-start]
2020-10-28T21:29:26.795-0800: 0.917: [CMS-concurrent-reset: 0.004/0.004 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-28T21:29:26.853-0800: 0.975: [GC (Allocation Failure) 2020-10-28T21:29:26.853-0800: 0.975: [ParNew: 314560K->34943K(314560K), 0.0353203 secs] 633375K->437839K(1013632K), 0.0354226 secs] [Times: user=0.25 sys=0.01, real=0.03 secs]
2020-10-28T21:29:26.944-0800: 1.066: [GC (Allocation Failure) 2020-10-28T21:29:26.944-0800: 1.066: [ParNew: 314559K->34943K(314560K), 0.0477693 secs] 717455K->519710K(1013632K), 0.0478575 secs] [Times: user=0.33 sys=0.02, real=0.05 secs]
2020-10-28T21:29:26.992-0800: 1.114: [GC (CMS Initial Mark) [1 CMS-initial-mark: 484766K(699072K)] 519782K(1013632K), 0.0002269 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2020-10-28T21:29:26.992-0800: 1.114: [CMS-concurrent-mark-start]
执行结束!共生成对象次数:8324
Heap
 par new generation   total 314560K, used 46300K [0x0000000780000000, 0x0000000795550000, 0x0000000795550000)
  eden space 279616K,   4% used [0x0000000780000000, 0x0000000780b17080, 0x0000000791110000)
  from space 34944K,  99% used [0x0000000791110000, 0x000000079332ffb8, 0x0000000793330000)
  to   space 34944K,   0% used [0x0000000793330000, 0x0000000793330000, 0x0000000795550000)
 concurrent mark-sweep generation total 699072K, used 484766K [0x0000000795550000, 0x00000007c0000000, 0x00000007c0000000)
 Metaspace       used 2695K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 295K, capacity 386K, committed 512K, reserved 1048576K
```

年轻代GC次数减少

当堆内存配置增大到2G时，未发生Full GC

#### 2.3.3 CMS GC总结

CMS GC对年轻代采用并行STW方式的标记-复制算法

对老年代主要采用并发-清除算法

CMS的设计目标是避免在老年代垃圾收集时出现长时间的卡顿，主要使用了以下手段

* 不对老年代进行整理，使用空闲列表来管理内存空间的回收
* 在标记-清除阶段的大部分工作是和应用线程一起并发执行的，并没有明显的应用线程暂停，只是仍然会有默认约1/4的线程数被CMS使用，无法处理业务


### 2.4 G1 GC的演示

#### 2.4.1 使用512m堆内存

```shell
java -XX:+UseG1GC -Xms512m -Xmx512m -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps GCLogAnalysis
正在执行...
2020-10-28T21:57:25.232-0800: 0.113: [GC pause (G1 Evacuation Pause) (young) 30M->12M(512M), 0.0064043 secs]
2020-10-28T21:57:25.251-0800: 0.132: [GC pause (G1 Evacuation Pause) (young) 42M->25M(512M), 0.0041272 secs]
2020-10-28T21:57:25.281-0800: 0.163: [GC pause (G1 Evacuation Pause) (young) 76M->40M(512M), 0.0078441 secs]
2020-10-28T21:57:25.371-0800: 0.253: [GC pause (G1 Evacuation Pause) (young) 207M->96M(512M), 0.0223950 secs]
2020-10-28T21:57:25.411-0800: 0.292: [GC pause (G1 Evacuation Pause) (young) 166M->118M(512M), 0.0102596 secs]
2020-10-28T21:57:25.467-0800: 0.349: [GC pause (G1 Evacuation Pause) (young) 240M->155M(512M), 0.0113187 secs]
2020-10-28T21:57:25.538-0800: 0.419: [GC pause (G1 Evacuation Pause) (young) 332M->210M(512M), 0.0238433 secs]
2020-10-28T21:57:25.575-0800: 0.456: [GC pause (G1 Humongous Allocation) (young) (initial-mark) 263M->226M(512M), 0.0055120 secs]
2020-10-28T21:57:25.580-0800: 0.461: [GC concurrent-root-region-scan-start]
2020-10-28T21:57:25.580-0800: 0.462: [GC concurrent-root-region-scan-end, 0.0002146 secs]
2020-10-28T21:57:25.580-0800: 0.462: [GC concurrent-mark-start]
2020-10-28T21:57:25.583-0800: 0.464: [GC concurrent-mark-end, 0.0020412 secs]
2020-10-28T21:57:25.583-0800: 0.464: [GC remark, 0.0016615 secs]
2020-10-28T21:57:25.585-0800: 0.466: [GC cleanup 238M->238M(512M), 0.0007807 secs]
2020-10-28T21:57:25.643-0800: 0.524: [GC pause (G1 Evacuation Pause) (young)-- 412M->322M(512M), 0.0225409 secs]
2020-10-28T21:57:25.667-0800: 0.548: [GC pause (G1 Evacuation Pause) (mixed) 326M->302M(512M), 0.0047500 secs]
2020-10-28T21:57:25.672-0800: 0.553: [GC pause (G1 Humongous Allocation) (young) (initial-mark) 305M->304M(512M), 0.0010250 secs]
2020-10-28T21:57:25.673-0800: 0.554: [GC concurrent-root-region-scan-start]
2020-10-28T21:57:25.673-0800: 0.554: [GC concurrent-root-region-scan-end, 0.0003432 secs]
2020-10-28T21:57:25.673-0800: 0.554: [GC concurrent-mark-start]
2020-10-28T21:57:25.676-0800: 0.557: [GC concurrent-mark-end, 0.0023245 secs]
2020-10-28T21:57:25.676-0800: 0.557: [GC remark, 0.0016252 secs]
2020-10-28T21:57:25.677-0800: 0.559: [GC cleanup 311M->311M(512M), 0.0007358 secs]
2020-10-28T21:57:25.702-0800: 0.583: [GC pause (G1 Evacuation Pause) (young) 413M->335M(512M), 0.0047817 secs]
2020-10-28T21:57:25.713-0800: 0.595: [GC pause (G1 Evacuation Pause) (mixed) 354M->296M(512M), 0.0066535 secs]
2020-10-28T21:57:25.727-0800: 0.608: [GC pause (G1 Evacuation Pause) (mixed) 321M->281M(512M), 0.0056230 secs]
2020-10-28T21:57:25.733-0800: 0.615: [GC pause (G1 Humongous Allocation) (young) (initial-mark) 285M->282M(512M), 0.0012278 secs]
2020-10-28T21:57:25.735-0800: 0.616: [GC concurrent-root-region-scan-start]
2020-10-28T21:57:25.735-0800: 0.616: [GC concurrent-root-region-scan-end, 0.0001595 secs]
2020-10-28T21:57:25.735-0800: 0.616: [GC concurrent-mark-start]
2020-10-28T21:57:25.736-0800: 0.617: [GC concurrent-mark-end, 0.0010456 secs]
2020-10-28T21:57:25.736-0800: 0.617: [GC remark, 0.0013495 secs]
2020-10-28T21:57:25.737-0800: 0.619: [GC cleanup 290M->290M(512M), 0.0006517 secs]
2020-10-28T21:57:25.773-0800: 0.654: [GC pause (G1 Evacuation Pause) (young) 397M->309M(512M), 0.0088930 secs]
2020-10-28T21:57:25.786-0800: 0.667: [GC pause (G1 Evacuation Pause) (mixed) 326M->285M(512M), 0.0071521 secs]
2020-10-28T21:57:25.794-0800: 0.675: [GC pause (G1 Humongous Allocation) (young) (initial-mark) 287M->285M(512M), 0.0020329 secs]
2020-10-28T21:57:25.796-0800: 0.677: [GC concurrent-root-region-scan-start]
2020-10-28T21:57:25.797-0800: 0.678: [GC concurrent-root-region-scan-end, 0.0002620 secs]
2020-10-28T21:57:25.797-0800: 0.678: [GC concurrent-mark-start]
2020-10-28T21:57:25.798-0800: 0.680: [GC concurrent-mark-end, 0.0018111 secs]
2020-10-28T21:57:25.799-0800: 0.680: [GC remark, 0.0014714 secs]
2020-10-28T21:57:25.800-0800: 0.681: [GC cleanup 290M->290M(512M), 0.0007712 secs]
2020-10-28T21:57:25.843-0800: 0.724: [GC pause (G1 Evacuation Pause) (young) 410M->318M(512M), 0.0105067 secs]
2020-10-28T21:57:25.857-0800: 0.738: [GC pause (G1 Evacuation Pause) (mixed) 332M->302M(512M), 0.0068699 secs]
2020-10-28T21:57:25.864-0800: 0.745: [GC pause (G1 Humongous Allocation) (young) (initial-mark) 303M->301M(512M), 0.0009557 secs]
2020-10-28T21:57:25.865-0800: 0.746: [GC concurrent-root-region-scan-start]
2020-10-28T21:57:25.865-0800: 0.746: [GC concurrent-root-region-scan-end, 0.0001180 secs]
2020-10-28T21:57:25.865-0800: 0.746: [GC concurrent-mark-start]
2020-10-28T21:57:25.866-0800: 0.747: [GC concurrent-mark-end, 0.0010210 secs]
2020-10-28T21:57:25.866-0800: 0.748: [GC remark, 0.0012343 secs]
2020-10-28T21:57:25.868-0800: 0.749: [GC cleanup 309M->309M(512M), 0.0006014 secs]
2020-10-28T21:57:25.910-0800: 0.791: [GC pause (G1 Evacuation Pause) (young)-- 431M->373M(512M), 0.0093748 secs]
2020-10-28T21:57:25.924-0800: 0.805: [GC pause (G1 Evacuation Pause) (mixed) 385M->352M(512M), 0.0129968 secs]
2020-10-28T21:57:25.937-0800: 0.818: [GC pause (G1 Humongous Allocation) (young) (initial-mark) 353M->352M(512M), 0.0016033 secs]
2020-10-28T21:57:25.939-0800: 0.820: [GC concurrent-root-region-scan-start]
2020-10-28T21:57:25.939-0800: 0.820: [GC concurrent-root-region-scan-end, 0.0002198 secs]
2020-10-28T21:57:25.939-0800: 0.820: [GC concurrent-mark-start]
2020-10-28T21:57:25.941-0800: 0.822: [GC concurrent-mark-end, 0.0024792 secs]
2020-10-28T21:57:25.941-0800: 0.823: [GC remark, 0.0014409 secs]
2020-10-28T21:57:25.943-0800: 0.824: [GC cleanup 358M->357M(512M), 0.0008914 secs]
2020-10-28T21:57:25.944-0800: 0.825: [GC concurrent-cleanup-start]
2020-10-28T21:57:25.944-0800: 0.825: [GC concurrent-cleanup-end, 0.0000463 secs]
2020-10-28T21:57:25.967-0800: 0.848: [GC pause (G1 Evacuation Pause) (young) 420M->365M(512M), 0.0081586 secs]
2020-10-28T21:57:25.982-0800: 0.863: [GC pause (G1 Evacuation Pause) (mixed) 389M->332M(512M), 0.0077074 secs]
2020-10-28T21:57:25.999-0800: 0.880: [GC pause (G1 Evacuation Pause) (mixed) 356M->321M(512M), 0.0088095 secs]
2020-10-28T21:57:26.008-0800: 0.889: [GC pause (G1 Humongous Allocation) (young) (initial-mark) 322M->321M(512M), 0.0019910 secs]
2020-10-28T21:57:26.010-0800: 0.891: [GC concurrent-root-region-scan-start]
2020-10-28T21:57:26.010-0800: 0.892: [GC concurrent-root-region-scan-end, 0.0003347 secs]
2020-10-28T21:57:26.010-0800: 0.892: [GC concurrent-mark-start]
2020-10-28T21:57:26.012-0800: 0.893: [GC concurrent-mark-end, 0.0018440 secs]
2020-10-28T21:57:26.012-0800: 0.894: [GC remark, 0.0015711 secs]
2020-10-28T21:57:26.014-0800: 0.895: [GC cleanup 328M->328M(512M), 0.0006491 secs]
2020-10-28T21:57:26.047-0800: 0.929: [GC pause (G1 Evacuation Pause) (young) 421M->348M(512M), 0.0080509 secs]
2020-10-28T21:57:26.059-0800: 0.941: [GC pause (G1 Evacuation Pause) (mixed) 364M->334M(512M), 0.0071440 secs]
2020-10-28T21:57:26.067-0800: 0.948: [GC pause (G1 Humongous Allocation) (young) (initial-mark) 335M->334M(512M), 0.0013653 secs]
2020-10-28T21:57:26.068-0800: 0.950: [GC concurrent-root-region-scan-start]
2020-10-28T21:57:26.069-0800: 0.950: [GC concurrent-root-region-scan-end, 0.0001877 secs]
2020-10-28T21:57:26.069-0800: 0.950: [GC concurrent-mark-start]
2020-10-28T21:57:26.070-0800: 0.951: [GC concurrent-mark-end, 0.0014384 secs]
2020-10-28T21:57:26.070-0800: 0.951: [GC remark, 0.0015030 secs]
2020-10-28T21:57:26.072-0800: 0.953: [GC cleanup 341M->341M(512M), 0.0006701 secs]
2020-10-28T21:57:26.101-0800: 0.982: [GC pause (G1 Evacuation Pause) (young) 410M->356M(512M), 0.0057115 secs]
2020-10-28T21:57:26.112-0800: 0.993: [GC pause (G1 Evacuation Pause) (mixed) 380M->337M(512M), 0.0064511 secs]
2020-10-28T21:57:26.126-0800: 1.007: [GC pause (G1 Evacuation Pause) (mixed) 364M->339M(512M), 0.0054688 secs]
2020-10-28T21:57:26.132-0800: 1.014: [GC pause (G1 Humongous Allocation) (young) (initial-mark) 343M->342M(512M), 0.0015743 secs]
2020-10-28T21:57:26.134-0800: 1.015: [GC concurrent-root-region-scan-start]
2020-10-28T21:57:26.134-0800: 1.015: [GC concurrent-root-region-scan-end, 0.0001497 secs]
2020-10-28T21:57:26.134-0800: 1.015: [GC concurrent-mark-start]
2020-10-28T21:57:26.136-0800: 1.017: [GC concurrent-mark-end, 0.0015144 secs]
2020-10-28T21:57:26.136-0800: 1.017: [GC remark, 0.0021530 secs]
2020-10-28T21:57:26.138-0800: 1.020: [GC cleanup 349M->349M(512M), 0.0007141 secs]
2020-10-28T21:57:26.151-0800: 1.032: [GC pause (G1 Evacuation Pause) (young) 410M->361M(512M), 0.0037391 secs]
2020-10-28T21:57:26.163-0800: 1.044: [GC pause (G1 Evacuation Pause) (mixed) 383M->342M(512M), 0.0105066 secs]
2020-10-28T21:57:26.179-0800: 1.060: [GC pause (G1 Evacuation Pause) (mixed) 368M->349M(512M), 0.0027737 secs]
2020-10-28T21:57:26.182-0800: 1.063: [GC pause (G1 Humongous Allocation) (young) (initial-mark) 351M->350M(512M), 0.0011975 secs]
2020-10-28T21:57:26.183-0800: 1.064: [GC concurrent-root-region-scan-start]
2020-10-28T21:57:26.183-0800: 1.064: [GC concurrent-root-region-scan-end, 0.0000716 secs]
2020-10-28T21:57:26.183-0800: 1.064: [GC concurrent-mark-start]
2020-10-28T21:57:26.185-0800: 1.066: [GC concurrent-mark-end, 0.0015372 secs]
2020-10-28T21:57:26.185-0800: 1.066: [GC remark, 0.0015067 secs]
2020-10-28T21:57:26.187-0800: 1.068: [GC cleanup 358M->358M(512M), 0.0007934 secs]
2020-10-28T21:57:26.204-0800: 1.085: [GC pause (G1 Evacuation Pause) (young) 407M->370M(512M), 0.0032264 secs]
执行结束!共生成对象次数:7336
```

1. 年轻代模式转移暂停
2. 并发标记
    * Initial Mark 初始标记
    * Root Region Scan Root区扫描
    * Concurrent mark 并发标记
    * Remark 再次标记
    * Cleanup 清理
3. 转移暂停：混合模式


#### 2.4.2 使用1g堆内存

```shell
java -XX:+UseG1GC -Xms1g -Xmx1g -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps GCLogAnalysis
正在执行...
2020-10-28T22:13:42.636-0800: 0.142: [GC pause (G1 Evacuation Pause) (young) 63M->22M(1024M), 0.0066787 secs]
2020-10-28T22:13:42.657-0800: 0.163: [GC pause (G1 Evacuation Pause) (young) 74M->38M(1024M), 0.0072367 secs]
2020-10-28T22:13:42.682-0800: 0.188: [GC pause (G1 Evacuation Pause) (young) 94M->59M(1024M), 0.0103774 secs]
2020-10-28T22:13:42.729-0800: 0.235: [GC pause (G1 Evacuation Pause) (young) 139M->87M(1024M), 0.0179104 secs]
2020-10-28T22:13:42.788-0800: 0.294: [GC pause (G1 Evacuation Pause) (young) 181M->113M(1024M), 0.0136790 secs]
2020-10-28T22:13:43.179-0800: 0.685: [GC pause (G1 Evacuation Pause) (young)-- 879M->620M(1024M), 0.0193717 secs]
2020-10-28T22:13:43.199-0800: 0.705: [GC pause (G1 Humongous Allocation) (young) (initial-mark) 621M->619M(1024M), 0.0054998 secs]
2020-10-28T22:13:43.204-0800: 0.710: [GC concurrent-root-region-scan-start]
2020-10-28T22:13:43.205-0800: 0.711: [GC concurrent-root-region-scan-end, 0.0000656 secs]
2020-10-28T22:13:43.205-0800: 0.711: [GC concurrent-mark-start]
2020-10-28T22:13:43.210-0800: 0.716: [GC concurrent-mark-end, 0.0055988 secs]
2020-10-28T22:13:43.210-0800: 0.716: [GC remark, 0.0016491 secs]
2020-10-28T22:13:43.212-0800: 0.718: [GC cleanup 643M->637M(1024M), 0.0016012 secs]
2020-10-28T22:13:43.214-0800: 0.720: [GC concurrent-cleanup-start]
2020-10-28T22:13:43.214-0800: 0.720: [GC concurrent-cleanup-end, 0.0000607 secs]
2020-10-28T22:13:43.318-0800: 0.824: [GC pause (G1 Evacuation Pause) (young)-- 917M->804M(1024M), 0.0068808 secs]
2020-10-28T22:13:43.331-0800: 0.837: [GC pause (G1 Evacuation Pause) (mixed) 836M->711M(1024M), 0.0053273 secs]
2020-10-28T22:13:43.347-0800: 0.853: [GC pause (G1 Evacuation Pause) (mixed) 768M->636M(1024M), 0.0055688 secs]
2020-10-28T22:13:43.367-0800: 0.873: [GC pause (G1 Evacuation Pause) (mixed) 693M->568M(1024M), 0.0098698 secs]
2020-10-28T22:13:43.398-0800: 0.904: [GC pause (G1 Evacuation Pause) (mixed) 622M->517M(1024M), 0.0103969 secs]
2020-10-28T22:13:43.409-0800: 0.915: [GC pause (G1 Humongous Allocation) (young) (initial-mark) 519M->519M(1024M), 0.0030924 secs]
2020-10-28T22:13:43.413-0800: 0.919: [GC concurrent-root-region-scan-start]
2020-10-28T22:13:43.413-0800: 0.919: [GC concurrent-root-region-scan-end, 0.0002625 secs]
2020-10-28T22:13:43.413-0800: 0.919: [GC concurrent-mark-start]
2020-10-28T22:13:43.416-0800: 0.922: [GC concurrent-mark-end, 0.0033186 secs]
2020-10-28T22:13:43.417-0800: 0.923: [GC remark, 0.0017304 secs]
2020-10-28T22:13:43.419-0800: 0.925: [GC cleanup 529M->508M(1024M), 0.0010501 secs]
2020-10-28T22:13:43.420-0800: 0.926: [GC concurrent-cleanup-start]
2020-10-28T22:13:43.420-0800: 0.926: [GC concurrent-cleanup-end, 0.0000595 secs]
2020-10-28T22:13:43.538-0800: 1.044: [GC pause (G1 Evacuation Pause) (young) 831M->572M(1024M), 0.0121038 secs]
2020-10-28T22:13:43.554-0800: 1.060: [GC pause (G1 Evacuation Pause) (mixed) 592M->480M(1024M), 0.0088458 secs]
2020-10-28T22:13:43.583-0800: 1.089: [GC pause (G1 Evacuation Pause) (mixed) 541M->431M(1024M), 0.0138395 secs]
执行结束!共生成对象次数:7381
```
