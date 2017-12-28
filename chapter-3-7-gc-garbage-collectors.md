# GC - Garbage Collectors

在Chapter3-5中已經簡單地介紹了GC演算法, 其就相當於是記憶體回收的方法論, 而這篇要介紹的各種Garbage Colletor就是記憶體回收的具體實作了. JVM規格中並沒有去規定要怎麼實作Garbage Colletor, 所以各個廠商所提供的collector可能就會有很大的差別, 且一般都會提供參數以供使用者根據自己的應用程式特性與需求去組合出適用於各年代的collector. 這篇會介紹的collector基本上都是基於JDK1.7u14之後的HotSpot VM, 其所包含的所有collector如下圖所示:

### Serial Collector \(Copying Algorithm\)

這是最基本且歷史最悠久的collector, 從它的名字可以得知其是一個**單執行緒的collector**, 其"單執行緒"的意義並不僅止於說明其**只會用一個CPU或是一條執行緒去收垃圾**, 更重要的是它在收垃圾的時候, **必須暫停其它所有正在正常工作的執行緒, 直到它收完垃圾為止**. 這就是前面提到的"Stop The World". STW這項工作基本上是由JVM在後台自動啟動與完成的 --- 在使用者不可見的情況下把使用者正常工作的執行緒全都停掉. 聽起來很討厭, 但你可以這樣想: **你媽在打掃房間的時候應該也會叫你在旁邊別亂動然後別同時繼續給她製造垃圾, 或是乾脆叫你滾出房間, 不然你媽一邊打掃你一邊在亂, 你媽可能會先打掃你而不是打掃房間了**. 這聽起來就合理多了, 而且GC這件事比你媽打掃房間還複雜就是了.

可能你會覺得這個collector很廢, 但到目前為止, 其仍然是JVM在client mode下預設的新生代collector. 畢竟相較於其他比較新的collector, Serial Collector有一個優點: **簡單且高效能\(當然是說跟其它collector的單執行緒相比\)**, 對於限定單CPU的環境來說, 此collector因為沒有執行緒互動的開銷, 所以專心做收垃圾這件事情自然可以獲得最高的單執行緒收集效率. 這就是為什麼對於運作在client mode下的JVM來說, Serial Collector仍然是一個不錯的選擇的原因.

### ParNew Collector \(Copying Algorithm\)

ParNew基本上就是**Serial Collector的多執行緒版本**, 除了使用多條執行緒進行收集之外, 其餘行為包含Serial Collector可用的所有控制參數\(-XX:SurvivorRatio, -XX:PretenureSizeThreshold, -XX:HandlePromotionFailure等\), 收集演算法, STW, 物件分配規則, 回收策略等都與Serial Collector完全一樣. 這聽起來好像沒有什麼特別之處, 但ParNew卻是許多運作在server mode下的JVM中首選的新生代collector, 另外還有一個與性能無關但很重要的一點: **除了Serial Collector之外, ParNew是唯一一個能與CMS合作的新生代collector**. 通常當你在VM options中指定了"**-XX:+UseConcMarkSweepGC**"後, 預設的新生代collector就會變成ParNew, 當然也可以使用"**-XX:+UseParNewGC**"來指定.

ParNew在單CPU的環境中, 基本上不會有比Serial Collector更優秀的表現, 甚至由於存在執行緒互動的開銷, 儘管在Hyper-Threading的CPU環境下也不能保證能夠超越Serial Collector. 當然, 隨著可用CPU的數量之遞增, 其對於GC時系統資源的有效利用還是有好處的. 其默認啟動的收集執行緒數量與CPU的數量相等, 若你的CPU很多, 可以用"**-XX:ParallelGCThreads**"來限制用於GC的執行緒數量.

### Parallel Scavenge Collector \(Copying Algorithm\)

這是一個用於新生代的collector, 且還是**平行**的**多執行緒**collector, 這感覺上跟ParNew很像, 但是其關注點與別的collector是不同的, Parallel Scavenge Collector最在意的是要可以達到一個可控制的吞吐量\(Throughput\). 這邊的吞吐量指的是說CPU用於執行client code的時間與CPU總消耗時間的比值, 寫成算式就是這樣:

**Throughput = \(Time for running client code\) / \(Time for running client code + Time for GC\)**

假設JVM運作了100分鐘, 其中收垃圾花了1分鐘, 那吞吐量就是: 99%

高吞吐量可以高效率地利用CPU時間, 盡快完成程式的運算任務, 主要適合在**後台運算且不需要太多互動的任務**.

此collector提供了兩個用於精確控制吞吐量的參數:

* **-XX:MaxGCPauseMillis**: **控制最大GC停頓時間**, 此參數允許的值是一個大於0的毫秒數, collector會盡可能地保證記憶體回收所花費的時間不超過設定值. 但你也**不要以為把這個參數設定得很小就可以讓系統的GC速度變快**, 因為**GC停頓時間的縮短基本上都是用犧牲吞吐量以及新生代空間換來的**: 你把新生代調小一點, 收200MB的新生代一定比收集800MB的新生代快呀, 這其實也會間接導致GC發動的頻率變高, 可能原來是10sec收一次, 每次停100ms, 後來變成5sec收一次, 每次停80ms. 這感覺好像也不值得, 因為吞吐量掉下來了.

* **-XX:GCTimeRatio**: **直接設定吞吐量大小**, 此參數應是一個大於0且小於100的整數, 也就是**GC時間占總時間的比率, 相當於是吞吐量的倒數**. 譬如說你設定成24, 那允許的最大GC時間就占總時間的:\(1/\(1 + 24\)\) = 4%. 這個參數的預設值是99, 就是允許最大1%的GC時間\(1/\(1 + 99\)\).

因為吞吐量關係密切, 這個collector也被稱為Throughput-First Collector, 而除了上述兩個參數之外, 還有一個很重要的參數:

* **-XX:+UseAdaptiveSizePolicy**: 這個參數是一個開關\(+/-\), 打開後你就不用手工指定新生代的大小\(-Xmn\), Eden與Survivor區的比例\(-XX:SurvivorRatio\), 以及晉升至老年代物件之大小\(-XX:PretenureSizeThreshold\)等參數了,** JVM會根據當前系統的運作情況收集性能監控資訊, 動態調整這些參數以提供最適合的停頓時間或是最大吞吐量, 這種自適應的方式又稱為GC Ergonomics**. 所以如果你很懶或是沒有把握能夠自己tune得很好, 就可以打開這個開關讓JVM去幫你搞定, 這時候你需要設定的東西就只剩下一些比較基本的參數像是"-Xmx"\(最大Java Heap size\), "-XX:MaxGCPauseMillis"或是"-XX:GCTimeRatio", 藉此提供JVM一個最佳化的方向. 單就adaptability這一點來說, 就是跟ParNew之間的一個重要區別了.

### Serial Old Collector \(Mark-Compact Algorithm\)

這個是Serial Collector的老年代版本, 一樣是單執行緒collector. 其主要意義也是在於給client mode下的JVM使用. 但若是在server mode下, 還有兩個用途:

* 在JDK1.5及之前的版本中與Parallel Scavenge搭配使用
* **作為CMS Collector的backup solution**

關於這兩點, 之後會再慢慢提到.

### Parallel Old Collector \(Mark-Compact Algorithm\)

Parallel Scavenge Collector的老年代版本, 使用**多執行緒**. 此collector是JDK1.6才開始出現的, 在此之前, 新生代的Parallel Scavenge其實一直都很尷尬, 原因是在JDK1.6之前, Parallel Scavenge\(新生代\)只能跟Serial Old\(老年代\)搭配\(因為Parallel Scavenge不能跟CMS搭配\), 這就會出現一個問題: **Serial Old在server端的性能可能會拖累整體GC效能\(因為單執行緒的老年代收集無法充分利用server多CPU的優勢\), 導致Parallel Scavenge未必能在整體上達到吞吐量最大化之效果**. 且這種尷尬的組合可能還會輸給ParNew + CMS.

不過在Parallel Old出現後, 就可以使用吞吐量優先的應用組合了, 在特別要求吞吐量或是CPU資源很敏感的情境中, 就可以優先考慮Parallel Scavenge + Parallel Old.

### CMS Collector \(Mark-Sweep Algorithm\)

CMS\(Concurrent Mark Sweep\)是一種**以獲取最短回收停頓時間為主要關注點**的collector. 對於目前很多集中在internet website或是B/S系統的Java application來說, 服務的回應速度是很重要的, 所以也希望停頓時間可以越短越好, 以帶來更好的UX. 這時候CMS就是很好的選擇.

不過, CMS跟前面幾種collector比起來, 其運作方式又更複雜了一點, 基本上分為以下四個步驟:

1. **初始標記 \(CMS initial mark, STW required\)**: 就只是標記一下GC Roots能直接關連到的物件而已, 這步動作還算快.
2. **並發標記 \(CMS concurrent mark\)**: 並發地進行GC Roots Tracing, 最慢的大概就是這步了, 但是可以同時跟client code一起運作.
3. **重新標記 \(CMS remark, STW required\)**: 這裡是為了修正並發標記期間因為client code繼續運作而導致標記產生變動的那一部分物件的標記紀錄, 此階段會比初始標記稍微久一點, 但遠比並發標記要短.
4. **並發清除 \(CMS concurrent sweep\)**: 就是並發的清垃圾, 可以跟client code一起運作.

這邊可以回想一下Serial Collector時候提到: "你媽在打掃, 你就在旁邊待著的情境". 在應用CMS的場合來說, 會變成: "**你媽在打掃的同時, 你還可以在旁邊繼續丟垃圾\(**~~**你之後會不會被你媽收掉我不知道**~~**\)**".

到這裡, 我們可以看到CMS的幾個優點: **並發收集, 低停頓,** 所以其也被稱為低停頓收集器\(Concurrent Low Pause Collector\). 不過世界上沒有任何東西是完美的, 基本上, CMS有3個明顯的缺點:

* **對CPU資源非常敏感**: 基本上, 只要扯到concurrent, 對CPU都會很敏感, 因為在並發階段, 其雖然不會導致client code停頓, 但是會因為佔用了一部分執行緒\(或是說CPU資源\)而導致應用程式變慢, 進而導致總吞吐量下降. CMS預設會啟動的回收執行緒數量是: **\(CPU數量 + 3\) / 4**, 就是說, **當CPU數量在4個以上時, 並發回收時收集垃圾的執行緒不會低於25%的CPU資源, 且會隨著CPU數量的遞增下降**. 但是**當CPU不足4個的時候\(譬如雙CPU\), CMS對client code的影響就很大了**, 試想若本來CPU loading就很大了, 還要分一半的運算能力去收垃圾, 就是說執行速度被腰斬了. 為了對付這種情況, CMS還有衍生出另一個變種 --- i-CMS\(Incremental Concurrent Mark Sweep\), 其原理是在並發標記/清除的時候讓GC執行緒跟client執行緒交替運作, 以期減少CPU被GC執行緒佔用的時間, 但這樣整個收垃圾的時間就會被拉長了, 而實驗證明, 這東西好像也沒那麼有用, 所以現在已經被標為deprecated了.

* **無法處理浮動垃圾\(Floating Garbage\)**: 這會在**並發清除**的階段出現, 因為並發清除的時候, client code還在繼續運作, 所以自然就有可能會有新的垃圾不斷產生, 就是剛才說的**你媽在掃地你還在旁邊繼續丟**. **這部分的垃圾是出現在標記過程之後的, 所以CMS無法在當前這次的收集中就清掉它們, 只好等下次GC再收, 這種垃圾就叫做浮動垃圾**. 其隱含了一個問題就是可能會出現"Concurrent Mode Failure"而導致另一次Full GC的產生. 由於在GC階段的時候, client還是要繼續運行, 這就表示**還需要預留有足夠的記憶體空間給client執行緒使用, 因此CMS不能像其他collector一樣等到老年代幾乎要塞爆了才開始收集, 需要先預留一部分空間提供並發收集時的程式運作使用**. 關於這部分, 可以透過調整"**-XX:CMSInitiatingOccupancyFraction**"參數來調整觸發的百分比, 在JDK1.6中, 其threshold已經提高至92%. **要是CMS運作期間, 預留的記憶體空間無法滿足程式的需求, 就會出現"Concurrent Mode Failure"**, 這時JVM就會啟動之前提到的備援方案: **臨時啟動Serial Old Collector來重新進行老年代的收垃圾作業, 這樣停頓時間就會變長了**. 所以說, "-XX:CMSInitiatingOccupancyFraction"設定的太高的話, 很容易導致大量的"Concurrent Mode Failure", 性能反而會降低.

* **基於Mark-Sweep**: 這其實就表示**會有大量空間碎片產生**. 碎片過多就表示分配記憶體給大物件的時候會有麻煩, 譬如說**老年代明明就還有很多空間, 但是都是很零散的, 單一空間不夠大的那種, 這時候就只好來一次Full GC了**. 針對這個問題, CMS提供了以下兩個參數:

  * **-XX:+UseCMSCompactAtFullCollection**: 這是一個開關參數\(預設是打開的\), 用在當CMS快要hold不住且要進行Full GC時開啟記憶體碎片的合併整理過程, 要注意的是, **記憶體整理的過程是無法並發的**, 空間碎片的問題解決了, 但停頓時間就是得加長.

  * **-XX:CMSFullGCsBeforeCompaction**: 這個參數用於設置執行多少次不壓縮的Full GC後, 跟著來一次帶壓縮的\(**預設是0, 表示每次發生Full GC都進行碎片整理**\).

### G1 Collector

G1\(Garbage-First\)是目前最前沿的collector之一, 其定位是一款server side導向的collector. 與其他collector相比, G1有以下特點:

* 平行與並發
* 分代收集
* 空間整合
* 可預測的停頓











