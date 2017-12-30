# GC - Memory Allocation Demo

在Java技術體系中所提倡的自動記憶體管理最終可以歸納為自動化地解決了下面這兩個問題:

* 給物件分配記憶體
* 回收分配給物件的記憶體

在前面的筆記裡面已經把很多基本的概念都帶過了, 這邊就要來實際操作看看前面提過的各種東西.

關於物件的記憶體分配, 粗略一點說, 就是要在Java Heap上分配, 物件主要分配在新生代的Eden區上面, 若啟動了前面提到過的TLAB, 那就會在TLAB上先分配. 當然也有少數情況是會直接分到老年代裡面的, 由於分配的規則並不是完全固定的, 這些都必須要取決於你當下用了哪種GC組合還有各種帶有不同效果的vm options.

我在這邊是按照"深入理解Java虛擬機"這本書提到的各種情境去做練習的, 但前面有提到過, 由於JDK版本不match的關係, 所以結果可能不會那麼的精準, 就當做個參考看看吧. 要注意的是, 我在範例中使用的GC組合是: Serial/SerialOld.

### 物件優先在Eden分配

我們知道, 新物件基本上都會先進Eden區, 然後Eden區沒有足夠空間的時候, JVM就會發動Minor GC. [範例程式](https://github.com/yotsuba1022/java-concurrency/commit/75ed76a1711acfcd41c3d0b845d3683f686c630a)如下:

```java
package idv.java.jvm.gc.memoryallocate.eden;

/**
 * @author Carl Lu
 * VM args: -Xloggc:gclog-MinorGCDemo.log -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8 -XX:+UseSerialGC -XX:-UseCompressedClassPointers -XX:-UseCompressedOops
 */
public class MinorGCDemo {

    private static final int _1MB = 1024 * 1024;

    public static void main(String args[]) {
        byte[] allocation1, allocation2, allocation3, allocation4;
        allocation1 = new byte[2 * _1MB];
        allocation2 = new byte[2 * _1MB];
        allocation3 = new byte[2 * _1MB];
        allocation4 = new byte[4 * _1MB]; //這裡會觸發一次Minor GC
    }

}
```

GC log如下:

```log
Java HotSpot(TM) 64-Bit Server VM (25.152-b16) for bsd-amd64 JRE (1.8.0_152-b16), built on Sep 14 2017 02:31:13 by "java_re" with gcc 4.2.1 (Based on Apple Inc. build 5658) (LLVM build 2336.11.00)
Memory: 4k page, physical 8388608k(189488k free)

/proc/meminfo:

CommandLine flags: -XX:InitialHeapSize=20971520 -XX:MaxHeapSize=20971520 -XX:MaxNewSize=10485760 -XX:NewSize=10485760 -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:SurvivorRatio=8 -XX:-UseCompressedClassPointers -XX:-UseCompressedOops -XX:+UseSerialGC 
0.191: [GC (Allocation Failure) 0.191: [DefNew: 6443K->679K(9216K), 0.0067661 secs] 6443K->4775K(19456K), 0.0069140 secs] [Times: user=0.00 sys=0.01, real=0.01 secs] 
Heap
 def new generation   total 9216K, used 7145K [0x0000000103c00000, 0x0000000104600000, 0x0000000104600000)
  eden space 8192K,  78% used [0x0000000103c00000, 0x0000000104250748, 0x0000000104400000)
  from space 1024K,  66% used [0x0000000104500000, 0x00000001045a9f78, 0x0000000104600000)
  to   space 1024K,   0% used [0x0000000104400000, 0x0000000104400000, 0x0000000104500000)
 tenured generation   total 10240K, used 4096K [0x0000000104600000, 0x0000000105000000, 0x0000000105000000)
   the space 10240K,  40% used [0x0000000104600000, 0x0000000104a00030, 0x0000000104a00200, 0x0000000105000000)
 Metaspace       used 3187K, capacity 4112K, committed 4352K, reserved 8192K
```

不只這個範例, 在接下來的範例我會有幾個設定都是固定的, 像是這幾個:

* **-Xloggc:gclog-xxx.log**: 這就只是指定log名稱而已, xxx大概就會跟class name一樣

* **-Xms20M**: 限制Java Heap的最小size為20MB

* **-Xmx20M**: 限制Java Heap的最大size為20MB

* **-Xmn10M**: 把Java Heap中的10MB分給新生代, 所以綜合這三個參數, 表示說新生代跟老年代各佔10MB

* **-XX:+PrintGCDetails**: 告訴JVM在發生GC的時候記得印出回收log

* **-XX:SurvivorRatio=8**: 定義Eden/Survivor的記憶體區塊比例為8:1, 因為前面設定新生代佔了10MB, 所以這邊的新生代**總可用**空間\(就是一個Eden + 一個Survivor\)應該為\(8 + 1\) \* 1MB = 9 MB = 9216 KB \(這邊只會算一個Survivor, 另一個備用的那個Survivor同樣是1MB, 但它不會被列入當前正在使用的空間裡, 至於原因, 可以回去前面的章節看\)

* **-XX:-UseCompressedClassPointers**: 這預設會打開, 但這邊我不想要這個所以先關掉它.

* **-XX:-UseCompressedOops**: 這預設會打開, 但這邊我不想要這個所以先關掉它.

### 大物件直接進入老年代

通常這邊講的大物件是說需要大量連續記憶體空間的Java物件, 譬如說很長的字串或是陣列. 大物件對JVM來說通常都是壞消息, 畢竟分配上比小物件麻煩, 還有一種更麻煩的是同時來了一群短命的大物件, 這種事情能避免的話還是避免吧. 大物件出現得越頻繁, 越容易提前觸發GC. [範例程式](https://github.com/yotsuba1022/java-concurrency/commit/5460380eaef861cf8a2b7f06508aaadaa256a394)如下:

```java
package idv.java.jvm.gc.memoryallocate.pretenure;

/**
 * @author Carl Lu
 * VM args: -Xloggc:gclog-PretenureDemo.log -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8 -XX:+UseSerialGC -XX:PretenureSizeThreshold=3145728 -XX:-UseCompressedClassPointers -XX:-UseCompressedOops
 * Here we use 3145728(byte) to represent 3MB since this can not be written as 3M for "PretenureSizeThreshold".
 * 3145728/1024/1024 = 3(MB)
 */
public class PretenureDemo {

    private static final int _1MB = 1024 * 1024;

    public static void main(String[] args) {
        byte[] allocatetion;
        allocatetion = new byte[4 * _1MB];
    }

}
```

GC log如下:

```
Java HotSpot(TM) 64-Bit Server VM (25.152-b16) for bsd-amd64 JRE (1.8.0_152-b16), built on Sep 14 2017 02:31:13 by "java_re" with gcc 4.2.1 (Based on Apple Inc. build 5658) (LLVM build 2336.11.00)
Memory: 4k page, physical 8388608k(211428k free)

/proc/meminfo:

CommandLine flags: -XX:InitialHeapSize=20971520 -XX:MaxHeapSize=20971520 -XX:MaxNewSize=10485760 -XX:NewSize=10485760 -XX:PretenureSizeThreshold=3145728 -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:SurvivorRatio=8 -XX:-UseCompressedClassPointers -XX:-UseCompressedOops -XX:+UseSerialGC 
Heap
 def new generation   total 9216K, used 2550K [0x0000000104800000, 0x0000000105200000, 0x0000000105200000)
  eden space 8192K,  31% used [0x0000000104800000, 0x0000000104a7daf0, 0x0000000105000000)
  from space 1024K,   0% used [0x0000000105000000, 0x0000000105000000, 0x0000000105100000)
  to   space 1024K,   0% used [0x0000000105100000, 0x0000000105100000, 0x0000000105200000)
 tenured generation   total 10240K, used 4096K [0x0000000105200000, 0x0000000105c00000, 0x0000000105c00000)
   the space 10240K,  40% used [0x0000000105200000, 0x0000000105600018, 0x0000000105600200, 0x0000000105c00000)
 Metaspace       used 3296K, capacity 4112K, committed 4352K, reserved 8192K
```

關於這部分的說明我已經寫在commit message裡了, 可以點範例程式的連結去看, 這裡我只記錄一下以下這個參數:

* **-XX:PretenureSizeThreshold=3145728**: 這個參數的意思是說, **size大於這個threshold的物件就直接送進老年代**. 原因是想要**避免在Eden跟Survivor之間發生大量的記憶體複製**\(應該還記得新生代基本上都採用Copying演算法吧?\). 然後為什麼要寫3145728這個數字呢? 因為**3145728 = 3 \* 1024 \* 1024, 也就是3MB的意思**, 原因是這個參數不能像-Xmx這類參數一樣直接寫個3MB就搞定. 最後, **這個參數只對Serial/ParNew這兩個collector有用**, Parallel Scavenge不認得這個參數. 通常若你想要用這個參數的話, 可以考慮使用ParNew + CMS的組合.

### 長期存活的物件將進入老年代

這個範例參照書上的去做已經不準確了, 因為collector的演算法有更動過了, 但還是可以透過一些奇技淫巧把它弄出來, [範例程式](https://github.com/yotsuba1022/java-concurrency/commit/254af3939eea648c35af72734e4f29945d4bb44a)如下:

```java
package idv.java.jvm.gc.memoryallocate.tenuringthreshold;

/**
 * @author Carl Lu
 * VM args:
 * -Xloggc:gclog-TenuringThresholdEqualsTo1Demo.log -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=1 -XX:+PrintTenuringDistribution -XX:-UseCompressedClassPointers -XX:-UseCompressedOops -XX:+UseSerialGC
 * -Xloggc:gclog-TenuringThresholdEqualsTo7Demo.log -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=7 -XX:+PrintTenuringDistribution -XX:-UseCompressedClassPointers -XX:-UseCompressedOops -XX:-UseAdaptiveSizePolicy -XX:+UseParNewGC
 */
public class TenuringThresholdDemo {
    private static final int _1MB = 1024 * 1024;

    public static void main(String[] args) {
        byte[] allocation1, allocation2, allocation3;

        allocation1 = new byte[_1MB / 4];
        allocation2 = new byte[4 * _1MB];
        allocation3 = new byte[4 * _1MB];
        allocation3 = null;
        allocation3 = new byte[4 * _1MB];
    }
}
```

GC log如下:

-XX:MaxTenuringThreshold=1的情況\(一次就衝進去了\):

```
Java HotSpot(TM) 64-Bit Server VM (25.152-b16) for bsd-amd64 JRE (1.8.0_152-b16), built on Sep 14 2017 02:31:13 by "java_re" with gcc 4.2.1 (Based on Apple Inc. build 5658) (LLVM build 2336.11.00)
Memory: 4k page, physical 8388608k(229412k free)

/proc/meminfo:

CommandLine flags: -XX:InitialHeapSize=20971520 -XX:InitialTenuringThreshold=1 -XX:MaxHeapSize=20971520 -XX:MaxNewSize=10485760 -XX:MaxTenuringThreshold=1 -XX:NewSize=10485760 -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintTenuringDistribution -XX:SurvivorRatio=8 -XX:-UseCompressedClassPointers -XX:-UseCompressedOops -XX:+UseSerialGC 
0.220: [GC (Allocation Failure) 0.220: [DefNew
Desired survivor size 524288 bytes, new threshold 1 (max 1)
- age   1:     972088 bytes,     972088 total
: 6902K->949K(9216K), 0.0068881 secs] 6902K->5045K(19456K), 0.0070786 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
0.228: [GC (Allocation Failure) 0.228: [DefNew
Desired survivor size 524288 bytes, new threshold 1 (max 1)
- age   1:       2992 bytes,       2992 total
: 5129K->2K(9216K), 0.0019897 secs] 9225K->5018K(19456K), 0.0020846 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
Heap
 def new generation   total 9216K, used 4236K [0x0000000117200000, 0x0000000117c00000, 0x0000000117c00000)
  eden space 8192K,  51% used [0x0000000117200000, 0x00000001176227d0, 0x0000000117a00000)
  from space 1024K,   0% used [0x0000000117a00000, 0x0000000117a00bb0, 0x0000000117b00000)
  to   space 1024K,   0% used [0x0000000117b00000, 0x0000000117b00000, 0x0000000117c00000)
 tenured generation   total 10240K, used 5015K [0x0000000117c00000, 0x0000000118600000, 0x0000000118600000)
   the space 10240K,  48% used [0x0000000117c00000, 0x00000001180e5fe8, 0x00000001180e6000, 0x0000000118600000)
 Metaspace       used 3288K, capacity 4112K, committed 4352K, reserved 8192K
```

-XX:MaxTenuringThreshold=7的情況\(還沒到門檻的七次就衝進去了\):

```
Java HotSpot(TM) 64-Bit Server VM (25.152-b16) for bsd-amd64 JRE (1.8.0_152-b16), built on Sep 14 2017 02:31:13 by "java_re" with gcc 4.2.1 (Based on Apple Inc. build 5658) (LLVM build 2336.11.00)
Memory: 4k page, physical 8388608k(169340k free)

/proc/meminfo:

CommandLine flags: -XX:InitialHeapSize=20971520 -XX:MaxHeapSize=20971520 -XX:MaxNewSize=10485760 -XX:MaxTenuringThreshold=7 -XX:NewSize=10485760 -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintTenuringDistribution -XX:SurvivorRatio=8 -XX:-UseAdaptiveSizePolicy -XX:-UseCompressedClassPointers -XX:-UseCompressedOops -XX:+UseParNewGC 
0.216: [GC (Allocation Failure) 0.216: [ParNew
Desired survivor size 524288 bytes, new threshold 1 (max 7)
- age   1:     971880 bytes,     971880 total
: 6902K->974K(9216K), 0.0045596 secs] 6902K->5070K(19456K), 0.0047542 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
0.222: [GC (Allocation Failure) 0.222: [ParNew
Desired survivor size 524288 bytes, new threshold 7 (max 7)
- age   1:        536 bytes,        536 total
: 5154K->295K(9216K), 0.0008993 secs] 9250K->5338K(19456K), 0.0010252 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 par new generation   total 9216K, used 4529K [0x0000000102000000, 0x0000000102a00000, 0x0000000102a00000)
  eden space 8192K,  51% used [0x0000000102000000, 0x0000000102422828, 0x0000000102800000)
  from space 1024K,  28% used [0x0000000102800000, 0x0000000102849c80, 0x0000000102900000)
  to   space 1024K,   0% used [0x0000000102900000, 0x0000000102900000, 0x0000000102a00000)
 tenured generation   total 10240K, used 5043K [0x0000000102a00000, 0x0000000103400000, 0x0000000103400000)
   the space 10240K,  49% used [0x0000000102a00000, 0x0000000102eecc00, 0x0000000102eecc00, 0x0000000103400000)
 Metaspace       used 3289K, capacity 4112K, committed 4352K, reserved 8192K
```

這邊要討論的參數是這個:

* **-XX:MaxTenuringThreshold**:  基本上, JVM會給每個物件定義年齡, **若物件在Eden出生且能熬過第一次Minor GC, 然後又能被Survivor容納的話, 就可以進入Survivor, 且此時其年齡為1.** 之後每在Survivor中熬過一次Minor GC, 就變老一歲, 而在早期的預設值中, 活到15歲的物件就可以進入老年代. 而這個參數就是用來定義這個年齡門檻的, 但...我發現這個在JDK8裡面的行為已經不太一樣了, 有興趣的請自行參考範例的commit history, 裡面有寫原因.

### 動態物件年齡判定

這個範例可以算是對前面一個範例出現的問題所產生的一個解釋, [範例程式](https://github.com/yotsuba1022/java-concurrency/commit/9151ee276286009d26591578a3a7a903d3966b53)如下:

```java
package idv.java.jvm.gc.memoryallocate.tenuringthreshold;

/**
 * @author Carl Lu
 */
public class TenuringThresholdDemo2 {
    private static final int _1MB = 1024 * 1024;

    public static void main(String[] args) {
        byte[] allocation1, allocation2, allocation3, allocation4;

        allocation1 = new byte[_1MB / 4];
        allocation2 = new byte[_1MB / 4];
        allocation3 = new byte[4 * _1MB];
        allocation4 = new byte[4 * _1MB];
        allocation4 = null;
        allocation4 = new byte[4 * _1MB];
    }
}
```

GC log如下:

有發生promotion的情況:

```
Java HotSpot(TM) 64-Bit Server VM (25.152-b16) for bsd-amd64 JRE (1.8.0_152-b16), built on Sep 14 2017 02:31:13 by "java_re" with gcc 4.2.1 (Based on Apple Inc. build 5658) (LLVM build 2336.11.00)
Memory: 4k page, physical 8388608k(206472k free)

/proc/meminfo:

CommandLine flags: -XX:InitialHeapSize=20971520 -XX:MaxHeapSize=20971520 -XX:MaxNewSize=10485760 -XX:MaxTenuringThreshold=15 -XX:NewSize=10485760 -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintTenuringDistribution -XX:SurvivorRatio=8 -XX:-UseCompressedClassPointers -XX:-UseCompressedOops -XX:+UseSerialGC 
0.281: [GC (Allocation Failure) 0.281: [DefNew
Desired survivor size 524288 bytes, new threshold 1 (max 15)
- age   1:    1048576 bytes,    1048576 total
: 7158K->1024K(9216K), 0.0073013 secs] 7158K->5326K(19456K), 0.0074506 secs] [Times: user=0.01 sys=0.01, real=0.00 secs] 
0.290: [GC (Allocation Failure) 0.290: [DefNew
Desired survivor size 524288 bytes, new threshold 15 (max 15)
: 5120K->0K(9216K), 0.0034517 secs] 9422K->5326K(19456K), 0.0035425 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
Heap
 def new generation   total 9216K, used 4178K [0x0000000110800000, 0x0000000111200000, 0x0000000111200000)
  eden space 8192K,  51% used [0x0000000110800000, 0x0000000110c14970, 0x0000000111000000)
  from space 1024K,   0% used [0x0000000111000000, 0x0000000111000000, 0x0000000111100000)
  to   space 1024K,   0% used [0x0000000111100000, 0x0000000111100000, 0x0000000111200000)
 tenured generation   total 10240K, used 5326K [0x0000000111200000, 0x0000000111c00000, 0x0000000111c00000)
   the space 10240K,  52% used [0x0000000111200000, 0x00000001117339c8, 0x0000000111733a00, 0x0000000111c00000)
 Metaspace       used 3303K, capacity 4112K, committed 4352K, reserved 8192K
```

沒發生promotion的情況\(把程式的13跟14行註解掉就可以看到這個結果了\):

```
Java HotSpot(TM) 64-Bit Server VM (25.152-b16) for bsd-amd64 JRE (1.8.0_152-b16), built on Sep 14 2017 02:31:13 by "java_re" with gcc 4.2.1 (Based on Apple Inc. build 5658) (LLVM build 2336.11.00)
Memory: 4k page, physical 8388608k(232432k free)

/proc/meminfo:

CommandLine flags: -XX:InitialHeapSize=20971520 -XX:MaxHeapSize=20971520 -XX:MaxNewSize=10485760 -XX:MaxTenuringThreshold=15 -XX:NewSize=10485760 -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintTenuringDistribution -XX:SurvivorRatio=8 -XX:-UseCompressedClassPointers -XX:-UseCompressedOops -XX:+UseSerialGC 
0.305: [GC (Allocation Failure) 0.305: [DefNew
Desired survivor size 524288 bytes, new threshold 1 (max 15)
- age   1:     976104 bytes,     976104 total
: 6902K->953K(9216K), 0.0034486 secs] 6902K->953K(19456K), 0.0036413 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
Heap
 def new generation   total 9216K, used 5215K [0x000000011b400000, 0x000000011be00000, 0x000000011be00000)
  eden space 8192K,  52% used [0x000000011b400000, 0x000000011b8299b0, 0x000000011bc00000)
  from space 1024K,  93% used [0x000000011bd00000, 0x000000011bdee4e8, 0x000000011be00000)
  to   space 1024K,   0% used [0x000000011bc00000, 0x000000011bc00000, 0x000000011bd00000)
 tenured generation   total 10240K, used 0K [0x000000011be00000, 0x000000011c800000, 0x000000011c800000)
   the space 10240K,   0% used [0x000000011be00000, 0x000000011be00000, 0x000000011be00200, 0x000000011c800000)
 Metaspace       used 3298K, capacity 4112K, committed 4352K, reserved 8192K
```

關於動態物件年齡的判定: JVM並不是永遠地要求物件的年齡必須達到MaxTenuringThreshold才可以升級到tenured generation, 若在survivor空間中, **相同年齡的所有物件大小之總和大於survivor空間的一半**\(在上面的範例, 就是1MB/2 = 512KB\), 年齡大於或等於該年齡的物件就可以直接進入tenured generation, 不需要等到MaxTenuringThreshold所要求的年齡.

### 空間分配擔保

這邊要講的東西比較複雜, 先貼[範例程式](https://github.com/yotsuba1022/java-concurrency/commit/f3b9db23c8b904b520bcefebaf4ede37b8eb67de):

```java
package idv.java.jvm.gc.memoryallocate.forcepromotion;

/**
 * @author Carl Lu
 * VM args: -Xloggc:gclog-ForcePromotionDemo.log -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8 -XX:+PrintTenuringDistribution -XX:-UseCompressedClassPointers -XX:-UseCompressedOops -XX:+UseSerialGC
 */
public class ForcePromotionDemo {
    private static final int _1MB = 1024 * 1024;

    public static void main(String[] args) {
        byte[] allocation1, allocation2, allocation3, allocation4, allocation5, allocation6, allocation7;
        allocation1 = new byte[2 * _1MB];
        allocation2 = new byte[2 * _1MB];
        allocation3 = new byte[2 * _1MB];
        allocation1 = null;
        allocation4 = new byte[2 * _1MB];
        allocation5 = new byte[2 * _1MB];
        allocation6 = new byte[2 * _1MB];
        allocation4 = null;
        allocation5 = null;
        allocation6 = null;
        allocation7 = new byte[2 * _1MB];
    }
}
```

GC log如下:

把-XX:+PrintTenuringDistribution打開的log:

```
Java HotSpot(TM) 64-Bit Server VM (25.152-b16) for bsd-amd64 JRE (1.8.0_152-b16), built on Sep 14 2017 02:31:13 by "java_re" with gcc 4.2.1 (Based on Apple Inc. build 5658) (LLVM build 2336.11.00)
Memory: 4k page, physical 8388608k(223884k free)

/proc/meminfo:

CommandLine flags: -XX:InitialHeapSize=20971520 -XX:MaxHeapSize=20971520 -XX:MaxNewSize=10485760 -XX:NewSize=10485760 -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintTenuringDistribution -XX:SurvivorRatio=8 -XX:-UseCompressedClassPointers -XX:-UseCompressedOops -XX:+UseSerialGC 
0.249: [GC (Allocation Failure) 0.249: [DefNew
Desired survivor size 524288 bytes, new threshold 1 (max 15)
- age   1:     709448 bytes,     709448 total
: 6646K->692K(9216K), 0.0066271 secs] 6646K->4788K(19456K), 0.0068361 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
0.257: [GC (Allocation Failure) 0.258: [DefNew (promotion failed) : 7076K->6383K(9216K), 0.0059956 secs]0.264: [Tenured: 8855K->8854K(10240K), 0.0055091 secs] 11172K->8854K(19456K), [Metaspace: 3279K->3279K(8192K)], 0.0116887 secs] [Times: user=0.01 sys=0.01, real=0.01 secs] 
Heap
 def new generation   total 9216K, used 4480K [0x0000000111e00000, 0x0000000112800000, 0x0000000112800000)
  eden space 8192K,  54% used [0x0000000111e00000, 0x00000001122600f8, 0x0000000112600000)
  from space 1024K,   0% used [0x0000000112600000, 0x0000000112600000, 0x0000000112700000)
  to   space 1024K,   0% used [0x0000000112700000, 0x0000000112700000, 0x0000000112800000)
 tenured generation   total 10240K, used 8854K [0x0000000112800000, 0x0000000113200000, 0x0000000113200000)
   the space 10240K,  86% used [0x0000000112800000, 0x00000001130a5ae0, 0x00000001130a5c00, 0x0000000113200000)
 Metaspace       used 3285K, capacity 4112K, committed 4352K, reserved 8192K
```

把-XX:+PrintTenuringDistribution關掉的log\(我覺得看起來比較乾淨\):

```
Java HotSpot(TM) 64-Bit Server VM (25.152-b16) for bsd-amd64 JRE (1.8.0_152-b16), built on Sep 14 2017 02:31:13 by "java_re" with gcc 4.2.1 (Based on Apple Inc. build 5658) (LLVM build 2336.11.00)
Memory: 4k page, physical 8388608k(190140k free)

/proc/meminfo:

CommandLine flags: -XX:InitialHeapSize=20971520 -XX:MaxHeapSize=20971520 -XX:MaxNewSize=10485760 -XX:NewSize=10485760 -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:SurvivorRatio=8 -XX:-UseCompressedClassPointers -XX:-UseCompressedOops -XX:+UseSerialGC 
0.279: [GC (Allocation Failure) 0.279: [DefNew: 6482K->718K(9216K), 0.0070587 secs] 6482K->4814K(19456K), 0.0072504 secs] [Times: user=0.01 sys=0.01, real=0.01 secs] 
0.288: [GC (Allocation Failure) 0.288: [DefNew (promotion failed) : 7019K->6301K(9216K), 0.0057992 secs]0.294: [Tenured: 8881K->8880K(10240K), 0.0057257 secs] 11115K->8880K(19456K), [Metaspace: 3291K->3291K(8192K)], 0.0117102 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
Heap
 def new generation   total 9216K, used 4563K [0x0000000110a00000, 0x0000000111400000, 0x0000000111400000)
  eden space 8192K,  55% used [0x0000000110a00000, 0x0000000110e74f70, 0x0000000111200000)
  from space 1024K,   0% used [0x0000000111200000, 0x0000000111200000, 0x0000000111300000)
  to   space 1024K,   0% used [0x0000000111300000, 0x0000000111300000, 0x0000000111400000)
 tenured generation   total 10240K, used 8880K [0x0000000111400000, 0x0000000111e00000, 0x0000000111e00000)
   the space 10240K,  86% used [0x0000000111400000, 0x0000000111cac000, 0x0000000111cac000, 0x0000000111e00000)
 Metaspace       used 3297K, capacity 4112K, committed 4352K, reserved 8192K
```

所謂的空間分配擔保機制是這樣的: **在發生Minor GC之前, JVM會先檢查老年代最大可用的連續記憶體空間是否大於新生代所有物件的總記憶體空間**, 若這個條件成立, 那麼Minor GC就可以確保是安全的. 反之, JVM就會去看**HandlePromotionFailure**的設定值**\(-XX:-HandlePromotionFailure是允許擔保失敗, 即不去handle promotion failure; -XX:+HandlePromotionFailure是不允許擔保失敗, 即要去handle promotion failure\)**, 看其是否允許擔保失敗\(**Promotion Failure**\). 如果允許, 那麼會繼續檢查老年代最大可用的連續記憶體空間是否大於**歷次晉升至老年代的物件之平均大小**, **若有大於, 就嘗試進行一次Minor GC**, 儘管這個Minor GC可能隱含著風險\(這個風險的定義等等下面會詳細說明\); **若是小於, 或著HandlePromotionFailure設置不準冒這種風險, 那這時就改成進行一次Full GC**.

那風險是什麼? 我們知道新生代用的是copying演算法, 但為了記憶體的利用率, 我們只會使用兩個Survivor區塊的其中一區來作為備份, 所以**如果出現了大量物件在Minor GC後還是活著的情況下, 講極端點, 全員生還\(100%\), 這樣就要老年代來擔保, 把Survivor無法容納的物件都送進老年代去.** 這跟你去銀行貸款很像, 你的擔保人\(老年代\)要幫你擔保, 他本身口袋也要夠深\(記憶體空間要夠\), 然而, **實際會有多少物件會活下來在實際完成GC之前是不知道的, 所以只好取之前每一次回收時, 晉升到老年代的物件之容量平均大小來作為一個評估值, 並拿此評估值來與老年代的剩餘空間作比較, 看是否要發動Full GC來讓老年代釋放出更多空間.**

其實這種取平均值的方式本身就是一種仰賴動態機率的手段, 這背後隱含了一個問題: **如果某次Minor GC存活後的物件暴增了, 且遠遠高於平均值, 那還是會出現Handle Promotion Failure. 如果是這樣, 就只好於失敗後發動Full GC了**. 儘管如此, 通常還是會把HandlePromotionFailure這個開關打開, 畢竟這樣還是有很高的機率可以避免頻繁的出現Full GC.

好, 寫了這麼多, 但我覺得接下來的才是最重要的: 因為上面講的這一串都**只適用於JDK6u24之前**, 在JDK6u24之後, **HandlePromotionFailure這個參數基本上已經不會再被使用了**\(~~不要打我~~\), 雖然據說你在JDK的原始碼中還是看得到它, 但就是沒有被用到. 在新的JDK實作中, 上述的規則已經變更成: **只要老年代的連續記憶體空間大於新生代物件總大小或著歷次晉升的平均大小, 就會發動Minor GC, 反之則進行Full GC.**

