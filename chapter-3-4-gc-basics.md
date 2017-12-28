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

* **Reachability Analysis**: 在大部分的商用解決方案裡, 幾乎都是透過這種方式去做判定的. 這個演算法的核心概念是通過一系列稱為"GC Roots"的物件作為起點, 從這些節點開始向下搜尋, 搜尋的路徑稱為reference chain, 當一個物件到GC Roots沒有任何reference chain相連的話, 就表示這個物件是不可用的了. 如下圖所示, 物件5~7看似有關聯, 但他們跟GC Roots之間是不可達的, 所以可以被回收.  
  而在Java中, 可作為GC Roots的物件大致上有以下幾種:

  1. JVM Stack\(這裡指Local Variable Table\)中參照的物件  
  2. Method Area中靜態屬性參照的物件  
  3. Method Area中常數參照的物件  
  4. Native Method Stack中JNI\(就是Native method\)參照的物件
  

* **About Reference**: 不管用上面提到的哪種方式去判斷, 判斷物件是否存活都跟參照有關, 在JDK1.2之前 參照的定義很單純 --- "若reference type的資料中儲存的資料代表的是另外一塊記憶體的起始位置, 就稱這塊記憶體代表著一個reference". 這很純粹, 但也有點不夠, 試想若有一種物件是"當記憶體空間還足夠時, 就留它一命; 但若記憶體在進行GC後還是很吃緊, 就回收掉這些它", 這在某些系統的cache功能都很適合這種情境. 所以, 在JDK1.2之後, Java對reference的概念進行了擴充, 把Reference分成以下四種, 按強度由強至低排列如下:

  1. Strong Reference:  這種就是你常看到的"Object obj = new Object\(\);", 只要strong reference還存在, GC就不會收走它們.

  2. Soft Reference: 這是指一些還有用但並非必須的物件. 在系統將要發生OOM之前, 會把這些物件列入回收名單中進行第二次回收, 若回收後還是沒有足夠記憶體, 才拋OOM. 關於這部分, 有SoftReference class可以用\(我是沒用過啦\).

  3. Weak Reference: 一樣是描述非必要物件, 但強度更低, 被這種reference關聯到的物件只能活到下一次GC發生之前. GC在工作時, 無論當前記憶體是否足夠, 都會回收掉只被weak reference關聯的物件. 關於這部分, 有WeakReference class可用\(我也沒用過\).

  4. Phantom Reference: 這是最弱的reference, 為一個物件設置這種reference的唯一目的就是能夠在這個物件被GC時收到一個系統通知. 關於這部分, 有PhantomReference class可以用\(對, 我還是沒用過\).

* **Dead or Alive**:



