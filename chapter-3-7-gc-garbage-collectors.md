# GC - Garbage Collectors

在Chapter3-5中已經簡單地介紹了GC演算法, 其就相當於是記憶體回收的方法論, 而這篇要介紹的各種Garbage Colletor就是記憶體回收的具體實作了. JVM規格中並沒有去規定要怎麼實作Garbage Colletor, 所以各個廠商所提供的collector可能就會有很大的差別, 且一般都會提供參數以供使用者根據自己的應用程式特性與需求去組合出適用於各年代的collector. 這篇會介紹的collector基本上都是基於JDK1.7u14之後的HotSpot VM, 其所包含的所有collector如下圖所示:

![](/assets/3-7-1.png)

在開始介紹各種collector之前, 有兩個名詞要特別解釋一下, 因為在接下來的context中, 這會直接影響到你看不看得懂這篇在寫什麼:

* **平行\(Parallel\)**: 指多條GC執行緒平行工作, 但此時**client code仍然處於等待狀態**.
* **並發\(Concurrent\)**: 指client code跟GC執行緒同時工作\(當然這不見得就是平行的了, 有可能是**交替執行**\), 譬如說client code在繼續運作, 而GC程式則是運作在另一個CPU上.

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
3. **重新標記 \(CMS final remark, STW required\)**: 這裡是為了修正並發標記期間因為client code繼續運作而導致標記產生變動的那一部分物件的標記紀錄, 此階段會比初始標記稍微久一點, 但遠比並發標記要短.
4. **並發清除 \(CMS concurrent sweep\)**: 就是並發的清垃圾, 可以跟client code一起運作.

這邊可以回想一下Serial Collector時候提到: "你媽在打掃, 你就在旁邊待著的情境". 在應用CMS的場合來說, 會變成: "**你媽在打掃的同時, 你還可以在旁邊繼續丟垃圾\(**~~**你之後會不會被你媽收掉我不知道**~~**\)**".

到這裡, 我們可以看到CMS的幾個優點: **並發收集, 低停頓,** 所以其也被稱為低停頓收集器\(Concurrent Low Pause Collector\). 不過世界上沒有任何東西是完美的, 基本上, CMS有3個明顯的缺點:

* **對CPU資源非常敏感**: 基本上, 只要扯到concurrent, 對CPU都會很敏感, 因為在並發階段, 其雖然不會導致client code停頓, 但是會因為佔用了一部分執行緒\(或是說CPU資源\)而導致應用程式變慢, 進而導致總吞吐量下降. CMS預設會啟動的回收執行緒數量是: **\(CPU數量 + 3\) / 4**, 就是說, **當CPU數量在4個以上時, 並發回收時收集垃圾的執行緒不會低於25%的CPU資源, 且會隨著CPU數量的遞增下降**. 但是**當CPU不足4個的時候\(譬如雙CPU\), CMS對client code的影響就很大了**, 試想若本來CPU loading就很大了, 還要分一半的運算能力去收垃圾, 就是說執行速度被腰斬了. 為了對付這種情況, CMS還有衍生出另一個變種 --- i-CMS\(Incremental Concurrent Mark Sweep\), 其原理是在並發標記/清除的時候讓GC執行緒跟client執行緒交替運作, 以期減少CPU被GC執行緒佔用的時間, 但這樣整個收垃圾的時間就會被拉長了, 而實驗證明, 這東西好像也沒那麼有用, 所以現在已經被標為deprecated了.

* **無法處理浮動垃圾\(Floating Garbage\)**: 這會在**並發清除**的階段出現, 因為並發清除的時候, client code還在繼續運作, 所以自然就有可能會有新的垃圾不斷產生, 就是剛才說的**你媽在掃地你還在旁邊繼續丟**. **這部分的垃圾是出現在標記過程之後的, 所以CMS無法在當前這次的收集中就清掉它們, 只好等下次GC再收, 這種垃圾就叫做浮動垃圾**. 其隱含了一個問題就是可能會出現"Concurrent Mode Failure"而導致另一次Full GC的產生. 由於在GC階段的時候, client還是要繼續運行, 這就表示**還需要預留有足夠的記憶體空間給client執行緒使用, 因此CMS不能像其他collector一樣等到老年代幾乎要塞爆了才開始收集, 需要先預留一部分空間提供並發收集時的程式運作使用**. 關於這部分, 可以透過調整"**-XX:CMSInitiatingOccupancyFraction**"參數來調整觸發的百分比, 在JDK1.6中, 其threshold已經提高至92%. **要是CMS運作期間, 預留的記憶體空間無法滿足程式的需求, 就會出現"Concurrent Mode Failure"**, 這時JVM就會啟動之前提到的備援方案: **臨時啟動Serial Old Collector來重新進行老年代的收垃圾作業, 這樣停頓時間就會變長了**. 所以說, "-XX:CMSInitiatingOccupancyFraction"設定的太高的話, 很容易導致大量的"Concurrent Mode Failure", 性能反而會降低.

* **基於Mark-Sweep**: 這其實就表示**會有大量空間碎片產生**. 碎片過多就表示分配記憶體給大物件的時候會有麻煩, 譬如說**老年代明明就還有很多空間, 但是都是很零散的, 單一空間不夠大的那種, 這時候就只好來一次Full GC了**. 針對這個問題, CMS提供了以下兩個參數:

  * **-XX:+UseCMSCompactAtFullCollection**: 這是一個開關參數\(預設是打開的\), 用在當CMS快要hold不住且要進行Full GC時開啟記憶體碎片的合併整理過程, 要注意的是, **記憶體整理的過程是無法並發的**, 空間碎片的問題解決了, 但停頓時間就是得加長.

  * **-XX:CMSFullGCsBeforeCompaction**: 這個參數用於設置執行多少次不壓縮的Full GC後, 跟著來一次帶壓縮的\(**預設是0, 表示每次發生Full GC都進行碎片整理**\).

### G1 Collector \(Copying/Mark-Compact\)

G1\(Garbage-First\)是目前最前沿的collector之一, 其定位是一款**server side導向的collector**. 與其他collector相比, G1有以下特點:

* **平行與並發**: G1可以充分利用多CPU/多核心環境下的硬體優勢來縮短STW所需的停頓時間, 部分其它collector原本需要停頓Java執行緒執行的GC動作, G1仍然可以通過並發的方式讓Java程式繼續運作.

* **分代收集**: 分代的概念在G1中仍然保留著. 雖然G1可以不搭配其它collector就獨自管理整個GC Heap, 但其能夠採用不同的方式各別處理新生代/老年代的物件以獲得更好的回收效果.

* **空間整合**: G1使用的演算法跟CMS不同, **其在整體的角度上來看, 使用的是Mark-Compact; 而在局部上來看, 使用的是Copying**, 這其實隱含了一件事: **G1在運作期間不會有產生記憶體碎片的問題, 收集後可以提供整齊且可用的記憶體空間**. 這種特性有利於程式長時間運作, 分配大物件時不會因為無法找到足夠的連續空間而提前觸發下一次GC.

* **可預測的停頓**: 這是G1相對於CMS的另一項優勢, 降低停頓時間是G1與CMS的共同關注點, 但G1除了追求低停頓時間之外, 還能建立**可預測的停頓時間模型**, 可讓使用者明確指定**在一個長度為M毫秒的時間片段裡, 消耗在GC上的時間不得超過N毫秒**, 這點其實已經是RTSJ GC的特徵了.

從前面介紹的各種collector, 不難發現這些collector收集的範圍都限定於整個新生代或著是老年代, 但G1就不是這樣了. G1在看待Java Heap記憶體佈局的時候與其它collector有很大的差別, 其**將整個Java Heap畫分為多個大小相等的獨立區域\(Region\)**, **儘管還是保留著新生代與老年代的概念, 但新生代與老年代已不再是物理上被隔離開的了, 它們都會是Region\(不需要連續\)的集合**.

G1之所以可以建立可預測的停頓時間模型, 是因為其可以**有計劃地避免在整個Java Heap中進行全區域的GC**. G1會追蹤各個Region裡面的垃圾堆積的價值大小\(**回收所獲得的空間大小以及回收所需時間的經驗值**\), **並以此在後台維護一個優先列表, 每次都會根據允許的收集時間, 優先回收CP值最高的Region\(這就是Garbage-First名稱的由來\)**. 這種用Region畫分記憶體空間以及有優先級的區域回收方式, 保證了G1在有限時間內可以獲取盡可能高的收集率.

到這裡, 我們可以看出來G1的思路是想要把記憶體"**化整為零**", 這理解起來很容易, 但G1從開始發想到開發出商用版本, 也是花了整整10年的時間的\(從Sun於2004年發表第一篇相關論文開始算, 沒找錯的話, 應該是[這篇](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.63.6386&rep=rep1&type=pdf)\). 這其中其實有很多很多非常困難的問題要解決, 舉個例子: **把Java Heap分為多個Region後, GC真的就能以Region為單位進行收集了嗎?** 其實不然, 因為**Region不可能是孤立的**. **一個物件被分配在某個Region中, 其並非只能被同一Region中的其它物件參照, 而是可以與整個Java Heap中的任一物件發生參照關係**. 按照這個想法, 在做reachability analysis的時候豈不是就要掃整個Java Heap才能確定有哪些物件是否存活了嗎? 這個問題其實不是只有在G1才有, 只是說, 這個問題在G1中更為突出而已. 在之前談過的分代收集裡, 新生代的規模通常來說都會比老年代要小得多, 且新生代的回收次數也比老年代要來得頻繁, 那回收新生代物件時也會面臨相同的問題, **若回收新生代時也不得不同時掃描老年代的話, 那麼Minor GC的效率可能會下降很多**.

在G1裡, Region之間的物件參照以及其它collector中的新生代與老年代之間的物件參照, 在JVM中都是使用一個叫做**Remembered Set**的東西來避免對整個Heap做掃描的.** G1中的每個Region裡都會有一個與之對應的Rememvered Set, JVM發現程式在對Reference Type的資料進行寫入操作時, 會先產生一個Write Barrier暫時中斷這個寫入操作, 檢查Reference參照的物件是否處於不同的Region之中\(這個在分代的一個例子就是: 檢查老年代中的物件是否參照了新生代的物件\), 若是, 則通過一個叫做CardTable的東西把相關參照訊息記錄到被參照物件所屬的Region的Remembered Set中\(這邊的例子就是在新生代那邊的Region中的Rememvered Set中記下來\). 當進行記憶體回收時, 在窮舉GC Roots的範圍中加入Remembered Set就可以保證不對全Heap進行掃描也不會有遺漏了**\(繼續套用前面的例子, 就是說**新生代的GC Roots + Remembered Set儲存的內容 = 新生代收集時真正的GC Roots**\). 基本上講到這裡, 我們可以知道Remembered Set基本上跟前面提到過的OopMap一樣, 都是**拿空間換時間**的解決方案.

以上是對G1初步的介紹, 再來要談的是G1的運作步驟, 如果不看維護Remembered Set的操作的話, 大概可以分成以下四個步驟:

1. **初始標記 \(Initial Marking, STW required\)**: 標記一下GC Roots能直接關連到的物件, 並且修改TAMS\(Next Top at Mark Start\)的值, 讓下一階段的client code並發執行時, 可以在正確可用的Region中建立新物件, 這個階段一樣要STW, 但時間很短.

2. **並發標記 \(Concurrent Marking\)**: 從GC Roots開始對Java Heap中的物件進行reachability analysis, 找出還活著的物件, 比較耗時, 但是可以跟client code並發執行.

3. **最終標記 \(Final Marking, STW required\)**: 為了修正在並發標記期間**你媽在打掃然後你又亂丟垃圾的關係**, 導致標記產生變化的那一部分標記紀錄, JVM會把這段時間物件的變化記錄在執行緒的Remembered Set Logs裡面, 然後還要把Remembered Set Logs的資料合併到Remembered Set裡面, 這裡雖然需要STW, 但是可以平行\(parallel\)執行.

4. **篩選回收 \(Live Data Counting and Evacuation, STW optional\)**: 此階段首先對各個Region的回收價值跟成本進行排序, 根據使用者所期望的GC停頓時間來制定回收計劃, 通常這階段都會STW\(因為這樣可以大幅提高收集效率\).



