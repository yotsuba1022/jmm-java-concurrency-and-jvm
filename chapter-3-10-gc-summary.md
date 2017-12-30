# GC Summary

在GC - Basics那章, 有提到這三件事:

* **When - 什麼時候要回收?**
* **What - 哪些記憶體要被回收?**
* **How - 怎麼回收?**

在經過前面幾篇筆記的介紹後, 這邊就可以來整理一下這三件事情了, 算是對GC基礎篇的一個收尾.

### When \(什麼時候要回收?\)

關於回收的時間點, 我們可以這樣分析: **GC有哪幾種, 以及這幾種分別會在什麼區塊的什麼時候出現回收的動作?**

回想一下, 在談GC的時候, Java Heap可以分成**新生代\(Young Generation\)**以及**老年代\(Tenured Generation\)**, 這兩個地方發生的GC是不同的:

* **Minor GC**: 在大部分的情況下, 當新建立的物件在新生代沒辦法被分配到記憶體空間時\(這裡通常指Eden區\), 就會觸發Minor GC. 而在發生Minor GC之前, JVM會先檢查老年代最大可用的連續記憶體空間是否大於新生代所有物件的總記憶體空間, 若這個條件成立, 那麼Minor GC就是安全的, 即可以放心的執行Minor GC; 反之, 則根據HandlePromotionFailure參數決定要發動MinorGC或是Full GC. 不過在最新的JDK中, HandlePromotionFailure已經沒用了, 所以規則就變成: **只要老年代的連續記憶體空間大於新生代物件總大小或著歷次晉升物件的平均大小, 就會發動Minor GC, 反之則進行Full GC.**

* **Major GC**: 這通常是被Minor GC間接觸發的, 目的是要回收老年代的物件, 但也有特例是可以只觸發Major GC的\(如Parallel Scavenge Collector\).

* **Full GC**: **晉升到老年代的物件之大小若超過了老年代的剩餘記憶體空間, 就會觸發Full GC**. 若你用的JDK\(JDK6u24以前\)有支援HandlePromotionFailure參數, 那也有可能因為你打開了這個參數\(**-XX:+HandlePromotionFailure**\), 而出現明明老年代空間還夠, 但還是來一發Full GC的情況\(這年頭很多公司的起手式都是Java8了, 所以...\).

不過具體要在什麼時刻執行, 還是由系統來決定, 這基本上是無法預測的.

### What \(哪些記憶體要被回收?\)

透過**Reachiability Analysis**演算法, 從GC Roots開始搜索, 如果**發現有物件到GC Roots是沒有辦法透過任何reference chain相連**的話, 這些物件就會被進行**第一次標記**, 然後進行篩選. **所謂的篩選就是看這個物件是否有必要執行finalize\(\)方法**, 如果沒有必要執行或是已經執行過了, 那這物件就要被收掉了. 反之, 若需要執行finalize\(\)方法, 那這物件就會被放入**F-Queue**裡面等著被執行finalize\(\)方法, 而如果某個物件有辦法在finalize\(\)方法裡面自救\(**即把自己跟任意reference chain上的任一物件建立關係**\)的話, 那它就可以逃脫被回收的命運, 但這種復活的機會也只有一次而已, 因為**finalize\(\)方法只會被系統執行一次**而已. 在Finalizer thread執行完F-Queue裡的東西後, 就會看有哪些物件是被第二次標記的, 然後這些物件就被收掉了.

### How \(怎麼回收?\)

這部分我想可以從以下幾個方面來談:

* #### 回收發生在哪個年代:

  * **Young Generation**: 通常都是透過**Copying**演算法來做回收, 具體的方式, 如HotSpot的實作, 是會透過Eden/From Survivor/To Survivor這三個區塊之間的互動來完成. 運作方式大概是這樣: **每次使用Eden和其中一塊Survivor, 當回收時, 只將Eden和當前的Survivor中"還活著的物件"都一次性複製到另外一個Survivor區塊中, 最後清掉Eden和剛才用過的Survivor.** 當Survivor不夠用時, 就必須依靠其它記憶體\(通常指老年代, 即Tenured Generation\)來進行擔保\(**Handle Promotion**\). 另外, 由於Copying演算法之特性的關係\(**空間會被對切**\), 所以在老年代中通常不會採用這個演算法, 畢竟對半切的最大缺點就是你要先提供兩倍的記憶體空間.

  * **Tenured Generation**: 通常會透過以下兩種演算法來做回收

    * **Mark-Sweep**: 講白一點, 就是先標記出要被回收的物件, 然後在標記完成後統一回收掉. 這個演算法的主要缺點是效率可能不太好, 還有就是**很容易產生大量不連續的記憶體碎片, 這種不連續碎片多了, 就容易觸發另一次的GC動作**.

    * **Mark-Compact**: 先標記出要被回收的物件, 然後讓所有活著的物件向記憶體區塊的某一邊靠攏, 靠攏之後清掉邊界外面的那些記憶體\(邊緣人的命運\). 這個演算法基本上避免了Copying演算法對空間利用上不夠有效率的問題, 且也完全迴避了Mark-Sweep會產生碎片的問題. 至於缺點, 其實也滿明顯的: **標記所有存活的物件後, 還要整理所有存活物件的參照地址, 基本上從效率上來講這是比Copying演算法還要慢的.**
* #### 用什麼collector來回收:

  * ##### **用於新生代的collector**

    * **Serial Collector**: 採用Copying演算法, 只使用單一CPU/執行緒去收垃圾, STW required. 簡單而高效\(僅限於單執行緒context下的比較\). 適合運作在client mode下的JVM.

    * **ParNew Collector**: 採用Copying演算法, Serial的多執行緒版本, STW required. 也是**除了Serial之外唯一能跟CMS搭配的collector**. **在單CPU的環境下效能可能不是Serial的對手.**

    * **Parallel Scavenge Collector**: 採用Copying演算法, 且是平行的多執行緒collector. 其關注點是**達到一個可控制的吞吐量\(Throughput\), 以便高效地運用CPU**. 另外也具有自適應機制\(GC Ergonomics\)來協助使用者達到獲取最佳吞吐量的vm option可供使用\(**-XX:+UserAdaptiveSizePolicy**\).
  * ##### **用於老年代的collector**

    * **Serial Old Collector**: 採用Mark-Compact演算法, 是Serial的老年代版本, 屬於單執行緒收集器, 一樣適用於client mode的JVM. 若是在JDK1.5之前, 可跟Parallel Scavenge搭配\(~~扯人家後腿用的, 因為其本質是單執行緒~~\). **此collector也是CMS發生Concurrent Mode Failure的backup solution\(因為只剩這傢伙可以救CMS了\).**

    * **Parallel Old Collector**: 採用Mark-Compact演算法, Parallel Scavenge的老年代版本, 屬於多執行緒收集器. **在JDK6之後出現, 可以讓Parallel Scavenge免於被Serial Old扯後腿**. 在注重吞吐量以及CPU資源敏感的場合就可以用Parallel Scavenge + Parallel Old的組合.

    * **CMS Collector**: 採用Mark-Sweep演算法, **可多執行緒並發收集/低停頓**, 主要關注點是**希望獲得最短回收停頓時間**, 就是前面提過的"**你媽在打掃的同時, 你還可以在旁邊繼續丟垃圾**". 其作業模式主要分為四步: 初始標記\(STW required\)/並發標記\(Concurrent\)/重新標記\(STW required\)/並發清除\(會產生floating garbage\). 缺點主要有: 對CPU資源敏感\(**但CPU越多的話, 影響就越小**\)/**無法處理浮動垃圾**\(想想你媽打掃你又在那邊亂的樣子\), 可能導致Concurrent Mode Failure, 就要請Serial Old來幫忙坦一下/基於Mark-Sweep的天生缺點就是**容易產生大量記憶體空間碎片**, 但可透過參數設定發生Full GC時一併整理記憶體\(compact\).
  * ##### **老少通吃的collector**

    * **Garbage First Collector \(G1\)**: 採用Mark-Compact演算法, **可平行也可並發**, 採用**分代收集**且不必跟其它collector配合\(老少通吃\)而且還能夠建立**可預測的停頓時間模型**. 其對記憶體空間的佈局概念也很特別: **把整個Java Heap區分為多個大小相等的Region來進行處理, 而各Region中卻還是保有新生代/老年代的概念.** 回收時, **從回收CP值最高的Region開始回收以獲取盡可能高的收集效率.** 其化整為零的概念會碰到不同Region間的新生代與老年代物件之間存有相依性的問題, 故這部分是透過**Remembered Set**來解決的\(其它collector其實也有用到這個東西\). 其作業模式主要分四步: 初始標記\(STW required\)/並發標記\(Concurrent\)/最終標記\(STW required\)/篩選回收. 缺點可參照前面提到的Mark-Compact演算法的部分, 但我自己在公司用G1覺得還滿耐操的就是了.



