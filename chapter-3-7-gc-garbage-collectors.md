# GC - Garbage Collectors

在Chapter3-5中已經簡單地介紹了GC演算法, 其就相當於是記憶體回收的方法論, 而這篇要介紹的各種Garbage Colletor就是記憶體回收的具體實作了. JVM規格中並沒有去規定要怎麼實作Garbage Colletor, 所以各個廠商所提供的collector可能就會有很大的差別, 且一般都會提供參數以供使用者根據自己的應用程式特性與需求去組合出適用於各年代的collector. 這篇會介紹的collector基本上都是基於JDK1.7u14之後的HotSpot VM, 其所包含的所有collector如下圖所示:

### Serial Collector

padding

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

