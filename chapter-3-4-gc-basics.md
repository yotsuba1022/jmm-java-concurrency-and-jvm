# GC - Basics

### 前言

談到GC, 通常我們要思考三件事情:

* **When - 什麼時候要回收?**
* **What - 哪些記憶體要被回收?**
* **How - 怎麼回收?**

在前面的Chapter 3-1裡, 我們提到了JVM裡各個記憶體區塊及其特性, 其中, Program Counter Register/JVM Stack/Native Method Stack這三塊的生命週期基本上是跟執行緒一樣的, 且stack中要分配多少記憶體也是在class結構確定下來後就已經可以計算出來的\(大部分情況下啦\). 因此這幾個區塊的記憶體分配/回收都具有確定性, 因此不太需要過多去考慮回收的問題. 然而, Java Heap/Method Area就沒有這麼好了, 譬如說一個介面的多個實作, 其各別所需要的記憶體空間可能就差很多, 一個方法中的多個分支\(if...else這類\)所導致的記憶體需求也會差很多, 這些都只能在runtime的時候才能知道要建立哪些物件, 故這兩個區塊的記憶體分配/回收都是動態的, 而這就是GC所關注的部分. 在之後的文章裡, 探討的記憶體分配/回收基本上就會專注在這兩個區塊上.

### 物件死了沒

GC在對heap中進行回收之前, 第一件事就是要確定這些物件之中哪些還活著, 哪些已經死了. 關於這部分, 常見的有以下兩種作法:

* **Reference Counting**: 這不是一個完善的做法, 但還是在這邊紀錄一下, 其主要概念是說, 在物件中添加一個counter, 每當有一個地方參照這個物件時, counter就+1; 當參照失效時, counter就-1; 任何時刻當counter變0的話, 就表示這個物件不可能再被使用了. 這在客觀上來說好像很正確, 大部分情況下可能都沒問題, 但就是有例外. 試想一下, 如果物件之間出現了互相循環參照的情況下, 而外界對這些物件之間的參照又斷掉了的情況下, 這些物件之間的counter就不會全都是0了. 這感覺就像是一群人落難飄到荒島上, 外界以為這些人已經死了, 但他們卻還在世界上的某個角落活著, 且彼此之間有關連, 可能想合作逃出荒島之類的. 不管怎樣, 在主流的JVM的實作中, 基本上不會選擇這種作法.  
  我在[這邊](https://github.com/yotsuba1022/java-concurrency/commit/709f24f474bf3b82c3215c998b6638151a6ca8e0)有寫一段程式並且附上GC log, 示範了這種情況, 有興趣的可以去看看, 可以發現JVM不是按照這種方式去做回收的.

* **Reachability Analysis**: 在大部分的商用解決方案裡, 幾乎都是透過這種方式去做判定的. 這個演算法的核心概念是通過一系列稱為"GC Roots"的物件作為起點, 從這些節點開始向下搜尋, 搜尋的路徑稱為**reference chain**, 當一個物件到GC Roots沒有任何reference chain相連的話, 就表示這個物件是不可用的了. 如下圖所示:  
  ![](/assets/3-4-1.png)  
  物件5~7看似有關聯, 但他們跟GC Roots之間是不可達的, 所以可以被回收.  
  而在Java中, 可作為GC Roots的物件大致上有以下幾種:

  1. JVM Stack\(這裡指Local Variable Table\)中參照的物件  
  2. Method Area中靜態屬性參照的物件  
  3. Method Area中常數參照的物件  
  4. Native Method Stack中JNI\(就是Native method\)參照的物件

* **About Reference**: 不管用上面提到的哪種方式去判斷, 判斷物件是否存活都跟參照有關, 在JDK1.2之前 參照的定義很單純 --- "若reference type的資料中儲存的資料代表的是另外一塊記憶體的起始位置, 就稱這塊記憶體代表著一個reference". 這很純粹, 但也有點不夠, 試想若有一種物件是"當記憶體空間還足夠時, 就留它一命; 但若記憶體在進行GC後還是很吃緊, 就回收掉這些它", 這在某些系統的cache功能都很適合這種情境. 所以, 在JDK1.2之後, Java對reference的概念進行了擴充, 把Reference分成以下四種, 按強度由強至低排列如下:

  1. **Strong Reference**:  這種就是你常看到的"Object obj = new Object\(\);", 只要strong reference還存在, GC就不會收走它們.

  2. **Soft Reference**: 這是指一些還有用但並非必須的物件. 在系統將要發生OOM之前, 會把這些物件列入回收名單中進行第二次回收, 若回收後還是沒有足夠記憶體, 才拋OOM. 關於這部分, 有SoftReference class可以用\(我是沒用過啦\).

  3. **Weak Reference**: 一樣是描述非必要物件, 但強度更低, 被這種reference關聯到的物件只能活到下一次GC發生之前. GC在工作時, 無論當前記憶體是否足夠, 都會回收掉只被weak reference關聯的物件. 關於這部分, 有WeakReference class可用\(我也沒用過\).

  4. **Phantom Reference**: 這是最弱的reference, 為一個物件設置這種reference的唯一目的就是能夠在這個物件被GC時收到一個系統通知. 關於這部分, 有PhantomReference class可以用\(對, 我還是沒用過\).

* **Dead or Alive**: 即便是在reachability analysis中不可達的物件, 也並非是絕對會死的, 這時候這些物件基本上是處在緩刑的階段, 要真正宣告一個物件的死亡, **至少要經歷過兩次標記過程** --- 若物件在進行reachability analysis後發現沒有跟GC Roots相連, 它會被第一次標記並且進行一次篩選, 這個篩選就是**此物件是否有必要執行finalize\(\) method**. 當物件沒有override finalize\(\), 或是finalize\(\)已經被JVM呼叫過了, 那就是沒必要執行了. 倘若這個物件有必要執行finalize\(\), 那這個物件就會被放到一個叫做**F-Queue**的佇列裡面, 並在稍後由一個由JVM自動建立且優先順序比較低的**Finalizer thread**去執行它. 但這邊的執行僅僅只是說JVM會去觸發這個方法, 但並沒有承諾會等這個方法跑完, 原因很簡單: 若一個物件在finalize\(\)中執行得很慢, 甚至出現了dead lock, 這樣F-Queue裡的其它物件難不成要一直等然後導致GC crash嗎? 而在finalize\(\)中, 物件可以透過重新將自己跟reference chain上的任一物件建立關聯來讓自己逃離被回收的命運, 譬如把this關鍵字assign給某個變數或是物件的instance variable. 那在GC第二次標記時, 這個物件就會被移出"即將回收"的集合; 反之, 就真的被回收了. [這邊](https://github.com/yotsuba1022/java-concurrency/commit/d897defefebade66596de8fb2653731712f9a67f)實作了一個範例, 示範了自救的過程, 但物件第二次自救還是會失敗, 原因是因為任何一個物件的finalize\(\)都只會被系統呼叫一次, 若物件面對下一次回收, finalize\(\)是不會被執行的. 其實這裡只是想示範finalize\(\)只會被呼叫一次, 不是說要學怎麼靠這方法自救, 因為我覺得這並不是finalize\(\)的真正用意, 所以看看就好了.

* **Method Area的回收**: 前面的章節有提到過, 在Method Area做回收的CP值很低, 舉個例子, 通常在Java Heap的新生代中, 一次的GC大概可以收掉7~9成的垃圾, 而Method Area\(或是說HotSpot的永久代\)卻遠低於這個數字. 由於此區的主要回收大概只有兩部分內容 --- 廢棄的constant跟沒用的class. 回收廢棄的constant還算簡單, 但是判定一個class是不是沒用的class就比較麻煩了, 其要同時滿足以下三個條件才可以成立:  
  
  1. 該class所有的instance都已經被回收, 就是說Java Heap裡面已經不存在該class的任何instance了  
  
  2. 加載該class的class loader已經被回收了  
  
  3. 該class對應的java.lang.Class物件沒有於任何地方被參照, 無法在任何地方透過reflection存取該class的方法  
  
  從這邊大概可以想到一件事, 在大量使用reflection/dynamic proxy/CGLib等byte code的framework\(如Spring\), 或是動態生成JS以及OSGi這類頻繁自定義ClassLoader的情境下都需要JVM具備class unload的功能, 以保證永久代不會overflow \(這在JDK8已經有改善了\).



