# GC - How to Read GC Logs?

GC的log內容會根據你當下選擇的collector而有所差異, 但整體上來講, 格式都是差不多的, 以下是一個範例:

```
33.125: [GC [DefNew: 3324K->152K(3712K), 0.0025925 secs] 3324K->152K(11904K), 0.0031680 secs]

100.667: [Full GC [Tenured: 0K->210K(10240K), 0.0149142 secs] 4603K->210K(19456K), [Perm: 2999K->2999K(21248K)], 0.0150007 secs] [Times: user=0.01 sys=0.00, real=0.02 secs]
```

這裡大概可以分成幾個部分:

* **GC發生的時間**: 即最前面的"**33.125**:"以及"**100.667**:", 這個數字的含意是**從JVM啟動以來所經過的秒數**.

* **GC的停頓類型**: 即log開頭的"\[GC"/"\[Full GC", 這並不是用來區分新生代GC或是老年代GC的. 如果有"Full"出現, 表示該次GC有發生Stop-The-World. 再舉一個例子, 以下這段log是新生代的ParNew Collector中出現的log:

  ```
  [Full GC 283.736: [ParNew: 261599K->261599K(261599K), 0.0000288 secs] ...
  ```

  這邊也可以看到"\[Full GC", 原因可能是因為出現了promotion failure之類的問題, 所以才會引發STW.  
  如果是透過呼叫"System.gc\(\)"的話, 會顯示"\[Full GC \(System.gc\(\)\)", 像這樣:  
  `0.380: [Full GC (System.gc())`

* **GC發生的區域**: 這裡顯示的區域名稱跟你用的collector是哪種有很大的關係, 以Serial Collector來說, 其新生代的名稱就是"Default New Generation"/"def new generation", 其在log中的縮寫通常是"DefNew", 以下就是一段Serial Collector的log[範例](https://github.com/yotsuba1022/java-concurrency/blob/master/src/main/java/idv/java/jvm/gc/refcounting/gclog-UseSerialGC.log):  
  ![](/assets/3-8-1.png)如果是ParNew, 新生代名稱就會變成"\[ParNew", 意思是"Parallel New Generation"/"par new generation", [範例](https://github.com/yotsuba1022/java-concurrency/blob/master/src/main/java/idv/java/jvm/gc/refcounting/gclog-UseParNewGC.log)如下:  
  ![](/assets/3-8-2.png)  
  若是Parallel Scavenge, 新生代就叫"PSYoungGen", [範例](https://github.com/yotsuba1022/java-concurrency/blob/master/src/main/java/idv/java/jvm/gc/refcounting/gclog-UseParallelGC.log)如下:  
  ![](/assets/3-8-3.png)  
  以上, 老年代跟永久代/meta space同理, 名稱都可能會隨著collector而異.

* **GC發生前/後的記憶體區域使用量**: 回到第一個範例, 其中的"3324K-&gt;152K\(3712K\)", 指的是"**GC前該記憶體區域已使用的容量 -&gt; GC後該記憶體區域已使用的容量\(該記憶體區塊的總容量\)**"

* **GC發生前/後的Java Heap已使用容量**: 一樣看到第一個範例, 其中的"3324K-&gt;152K\(11904K\)", 指的是"**GC前Java Heap已使用容量 -&gt; GC後Java Heap已使用容量 \(Java Heap總容量\)**"

* **記憶體區塊GC所佔用的時間**: 還是第一個範例, 其中的"0.0025925 secs"表示該記憶體區塊GC所佔用的時間. 當然, 有些collector可能會給出更詳細的時間資訊, 譬如在第一個範例的最後面有一段"**\[Times: user=0.01 sys=0.00, real=0.02 secs\]**", 這裡面的user/sys/real其實就跟你在Linux裡下"time"指令的含義是一樣的, 分別代表使用者態消耗的CPU時間/內核態消耗的CPU事件/操作從開始到結束所經過的Wall Clock Time. 這邊要注意的是, CPU時間跟Wall Clock Time的區別為: 後者包括了各種非運算的等待耗時, 但前者沒有. 而當系統有多CPU或著多核心的話, **多執行緒操作會疊加這些CPU時間, 所以你如果發現user或是sys的時間超過real是正常的**.



