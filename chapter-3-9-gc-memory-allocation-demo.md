# GC - Memory Allocation Demo

在Java技術體系中所提倡的自動記憶體管理最終可以歸納為自動化地解決了下面這兩個問題:

* 給物件分配記憶體
* 回收分配給物件的記憶體

在前面的筆記裡面已經把很多基本的概念都帶過了, 這邊就要來實際操作看看前面提過的各種東西.

關於物件的記憶體分配, 粗略一點說, 就是要在Java Heap上分配, 物件主要分配在新生代的Eden區上面, 若啟動了前面提到過的TLAB, 那就會在TLAB上先分配. 當然也有少數情況是會直接分到老年袋裡面的, 由於分配的規則並不是完全固定的, 這些都必須要取決於你當下用了哪種GC組合還有各種帶有不同效果的vm options.

我在這邊是按照"深入理解Java虛擬機"這本書提到的各種情境去做練習的, 但前面有提到過, 由於JDK版本不match的關係, 所以結果可能不會那麼的精準, 就當做個參考看看吧. 要注意的是, 我在範例中使用的GC組合是: Serial/SerialOld.

### 物件優先在Eden分配

我們知道, 新物件基本上都會先進Eden區, 然後Eden區沒有足夠空間的時候, JVM就會發動Minor GC. 範例程式如下:

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

123

### 長期存活的物件將進入老年代

123

### 動態物件年齡判定

123

### 空間分配擔保

123



