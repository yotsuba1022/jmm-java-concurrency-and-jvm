# GC - Garbage Collectors

在Chapter3-5中已經簡單地介紹了GC演算法, 其就相當於是記憶體回收的方法論, 而這篇要介紹的各種Garbage Colletor就是記憶體回收的具體實作了. JVM規格中並沒有去規定要怎麼實作Garbage Colletor, 所以各個廠商所提供的collector可能就會有很大的差別, 且一般都會提供參數以供使用者根據自己的應用程式特性與需求去組合出適用於各年代的collector. 這篇會介紹的collector基本上都是基於JDK1.7u14之後的HotSpot VM, 其所包含的所有collector如下圖所示:

### Serial Collector

這是最基本且歷史最悠久的collector, 從它的名字可以得知其是一個**單執行緒的collector**, 其"單執行緒"的意義並不僅止於說明其**只會用一個CPU或是一條執行緒去收垃圾**, 更重要的是它在收垃圾的時候, **必須暫停其它所有正在正常工作的執行緒, 直到它收完垃圾為止**. 這就是前面提到的"Stop The World". STW這項工作基本上是由JVM在後台自動啟動與完成的 --- 在使用者不可見的情況下把使用者正常工作的執行緒全都停掉. 聽起來很討厭, 但你可以這樣想: **你媽在打掃房間的時候應該也會叫你在旁邊別亂動然後別同時繼續給她製造垃圾, 或是乾脆叫你滾出房間, 不然你媽一邊打掃你一邊在亂, 你媽可能會先打掃你而不是打掃房間了**. 這聽起來就合理多了, 而且GC這件事比你媽打掃房間還複雜就是了.

可能你會覺得這個collector很廢, 但到目前為止, 其仍然是JVM在client mode下預設的新生代collector. 畢竟相較於其他比較新的collector, serial collector有一個優點: **簡單且高效能\(當然是說跟其它collector的單執行緒相比\)**, 對於限定單CPU的環境來說, 此collector因為沒有執行緒互動的開銷, 所以專心做收垃圾這件事情自然可以獲得最高的單執行緒收集效率. 這就是為什麼對於運作在client mode下的JVM來說, serial collector仍然是一個不錯的選擇的原因.

### ParNew Collector

padding

### Parallel Scavenge Collector

padding

### Serial Old Collector

padding

### Parallel Old Collector

padding

### CMS Collector

padding

### G1 Collector

padding

