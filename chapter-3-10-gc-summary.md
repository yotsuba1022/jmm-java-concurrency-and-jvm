# GC Summary

在GC - Basics那章, 有提到這三件事:

* **When - 什麼時候要回收?**
* **What - 哪些記憶體要被回收?**
* **How - 怎麼回收?**

在經過前面幾篇筆記的介紹後, 這邊就可以來整理一下這三件事情了, 算是對GC基礎篇的一個收尾.

### When \(什麼時候要回收?\)

關於回收的時間點, 我們可以這樣分析: GC有哪幾種, 以及這幾種會在什麼地方的什麼時候出現回收的動作:

首先, 在談GC的時候, Java Heap可以分成新生代\(Young Generation\)以及老年代\(Tenured Generation\), 這兩個地方發生的GC是不同的.

* Minor GC: 在大部分的情況下, 當新建立的物件在新生代沒辦法被分配到記憶體空間時\(這裡通常指Eden區\), 就會觸發Minor GC. 而在發生Minor GC之前, JVM會先檢查老年代最大可用的連續記憶體空間是否大於新生代所有物件的總記憶體空間, 若這個條件成立, 那麼Minor GC就是安全的, 即可以放心的執行Minor GC, 反之, 則根據HandlePromotionFailure參數決定要則發動MinorGC或是Full GC. 不過在最新的JDK中, HandlePromotionFailure已經沒用了, 所以規則就變成: **只要老年代的連續記憶體空間大於新生代物件總大小或著歷次晉升物件的平均大小, 就會發動Minor GC, 反之則進行Full GC.**
* Major GC: 這通常是被Minor GC間接觸發的, 目的是要回收老年代的物件, 但也有特例是可以只觸發Major GC的\(如Parallel Scavenge Collector\).
* Full GC: 晉升到老年代的物件之大小若超過了老年代的剩餘記憶體空間, 就會觸發Full GC. 若你用的JDK\(JDK6u24以前\)有支援HandlePromotionFailure參數, 那也有可能因為你打開了這個參數\(**-XX:+HandlePromotionFailure**\), 而出現明明老年代空間還夠, 但還是來一發Full GC的情況\(這年頭很多公司的起手式都是Java8了, 所以...\).

不過具體要在什麼時刻執行, 還是由系統來決定, 這基本上是無法預測的.

### What \(哪些記憶體要被回收?\)

123

### How \(怎麼回收?\)

123

