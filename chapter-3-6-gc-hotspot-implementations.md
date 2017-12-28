# GC - HotSpot Implementations

前面簡單的介紹了物件存活判定以及垃圾回收的演算法, 再來這邊會簡介一下Hotspot在實作這些演算法時的做法, 但我功力不足, 這邊寫得不是很好, 等之後讀通了會再回來修改, 所以看看就好.

### 窮舉根節點\(GC Roots Enumeration\)

在reachability analysis中, 以從GC Roots節點找到reference chain這個操作為例子, 可作為GC Roots的節點主要在global的reference\(如**常數**或是**類別的static fields**\)與執行context\(譬如說stack frame中的**Local Variable table**\)中, 如果要逐一檢查這裡面的參照, 是一定會消耗相當可觀的時間的.

另外, reachability對執行時間的敏感也體現在GC停頓上, 因為**這個分析作業必須在一個能確保一致性的snapshot中進行**, 這邊講的一致性就是指說在整個分析期間, **整個執行系統看起來就像是被'凍結"在某個時間點上一般, 不可以出現分析過程中物件參照關係還一直在不斷變化的情況, 否則分析結果的準確性就無法得到保證了**. 這點就是導致GC運作時**必須停頓所有Java執行緒**\(這個動作又稱為**Stop The World**, a.k.a. **STW**\)的其中一個重要原因, 即使是在號稱幾乎不會發生停頓的CMS collector中, 窮舉根節點的時候也是必須要停頓的.

其實, 目前的主流JVM用的都是準確式GC, 故當執行系統停頓下來後, 並不需要一個不漏地檢查完所有執行context和global的參照位置, JVM應該是要有辦法直接得知哪些地方存放著物件參照的. 這部分在HotSpot中, 是使用一組稱為**OopMap**的資料結構來達到這個目的的, 在class加載完畢後, HotSpot就把物件內什麼offset上是什麼類型的資料給計算出來, 在JIT編譯過程中, 也會在**特定的位置\(即接下來會提到的安全點 --- Safepoint\)**記錄下stack/registor中哪些位置是參照. 這樣, GC在掃描時就可以直接知道這些資訊了.

上面這樣講OopMap感覺有講跟沒講一樣, 所以這邊就再寫多一點, 但這只是我粗淺的讀了點東西後的想法, 可能也不是完全正確的.

所謂OopMap, 就是要拿來紀錄stack上Local Variable Table裡的變數到heap上的物件的參照關係. **它的作用是當GC發生時, collector thread會對stack上的記憶體進行掃描, 看看哪些地方儲存了reference type. 若某些位置儲存了reference type, 就表示這些位置所參照的物件不可以在這次的回收作業中被回收**. 但問題就在這: **stack上的Local Variable Table裡只有一部分資料是reference type\(這就是我們要的\), 但那些非reference type的資料, 對我們又沒什麼用處, 但我們還是不得不把整個stack掃一次, 這就很浪費\(時間跟資源\)**.

所以, 另一個想法就是, 能不能乾脆用空間來換時間呢? 就是說**在某個時間點把stack上代表reference的位置都記下來, 這樣到真正GC的時候, 就直接讀取這些被記下來的位置就好了, 然後就不用一直在那邊掃描了**. 這就是HotSpot裡的OopMap在做的事情.

到這邊為止, 我們可以知道這樣的關係: **一個執行緒代表一個stack -&gt; 一個stack由多個stack frame組成 -&gt; 一個stack frame對應著一個方法 -&gt; 一個方法裡可能會有多個Safepoint**. GC發生時, 程式就會先找到最近的一個Safepoint停下來, 然後更新自己的OopMap, 記錄一下stack上有哪些位置代表reference type. 這樣一來, 在窮舉GC Roots的時候, 只要遞迴掃過每個stack frame的OopMap, 通過stack中紀錄的被參照物件的記憶體位置, 就可以找出這些GC Roots節點了.

### 安全點\(Safepoint\)

好了, 所以在OopMap的協助下, HotSpot就可以用比較有效率的方式準確地完成窮舉GC Roots的動作, 但接下來又出現了另一個問題 --- 可能導致參照關係發生變化, 或是換句話說, OopMap內容變化的指令非常多, 如果為每一條指令都升成對應的OopMap, 那就會需要大量的額外空間, 這樣GC的空間成本會變很高.

不過實際上HotSpot也沒有真的為每一條指令都升成OopMap, 前面有提到只會在"**特定的位置**"記錄這些資訊, 這些位置就叫做安全點\(Safepoint\), 即程式運作時並非在所有地方都能停頓下來開始GC, 只有在到達安全點的時候才可以暫停. 選Safepoint是一門學問, 因為選太少可能會讓GC等待時間太長; 選太多又會過份增加運行時的負擔. 所以其選定的基本方式是以程式"是否具有讓程式長時間執行的特徵"為標準來進行選定的. 因為每條指令執行的時間都很短, 程式不太可能因為指令流的長度太長這個原因就過長時間運作, 長時間運作的最明顯特徵就是指令序列的重用, 譬如說method invocation, loop跳轉, exception跳轉等, 所以具有這些功能的指令才會產生Safepoint.

下一個要考慮的問題是, 如何在GC發生時讓所有執行緒\(不包含執行JNI invocation的執行緒\)都"跑"到最近的Safepoint上再停下來. 這裡基本上有兩種方案:

* **搶先式中斷\(Preemptive Suspension\)**: 不需要執行緒的執行程式主動去配合, 在GC發生時, 首先把所有執行緒全都中斷, 若發現有執行緒中斷的地方不是在Safepoint上, 就恢復該執行緒, 讓其跑到Safepoint上.

* **主動式中斷\(Voluntary Suspension\)**: 當GC需要中斷執行緒的時候, 不直接對執行緒操作, 僅僅是簡單地設置一個flag, 各個執行緒執行時主動去poll\(輪詢\)這個flag, 發現中斷flag為trye的時候就自己中斷並且hang up. 這些pool flag和Safepoint是重合的, 另外再加上建立物件需要分配記憶體的地方. 目前HotSpot採用的就是這種方式.

### 安全區域



