# Object in Java Heap of HotSpot

目前最常見的JVM大概就是HotSpot了, 這邊就簡單記錄一下HotSpot在Java Heap中的物件分配, 佈局與存取的過程.

### 物件的建立

我們都知道在Java裡面要建立一個物件只要用new關鍵字搭配類別/介面名稱即可, 但這是在語言層面上的解釋, 若是在JVM層級呢? \(這邊只討論普通的Java物件, 不包含陣列以及Class物件等\).

在JVM層級, 對於物件建立的過程大概可以分成以下四個步驟:

1. **檢查/初始化**: 當JVM遇到一條new指令, 它會先去檢查這個指令的參數是否能在constant pool中定位到一個class的symbolic reference, 然後檢查這個symbolic reference代表的class是否已經被加載/解析/初始化. 若沒有, 就要執行相應的class loading過程.

2. **為新生物件分配記憶體**: 前面的class load檢查過程通過後, 物件所需的記憶體大小基本上就完全確定了. 為物件分配空間的任務等同於把一塊確定大小的記憶體從Java Heap中劃分出來, 這邊的劃分又有兩種方式:

   **a. Bump the Pointer**: 假設Java Heap是很整齊的, 所有用過的記憶體都放在一邊, 空閑的放另一邊, 中間有一個指標作為分界點, 那分配記憶體就只是要把這個指標往空閒的空間那邊挪動一段與物件大小相同的距離而已.

   **b. Free List**: 假設Java Heap不是整齊的, 使用過/空閒的空間互相交錯, 那就不能靠一個指標移動來搞定了, 這時候JVM就要維護一個list, 其中紀錄了哪些記憶體區塊是可以用的, 在分配的時候就從list中找到一塊夠大的空間分給物件就好, 然後更新這個list.

   選擇哪種記憶體劃分方式由Java Heap是否整齊而決定, 而Java Heap是否整齊, 又取決於所採用的Garbage Collector是否帶有compact\(壓縮/整理\)功能. 像之後會提到的Serial/ParNew等具備compact功能的收集器, 就是用Bump the Pointer; 而CMS這種基於Mark-Sweep演算法的收集器, 基本上就採用Free List.

   除了劃分空間之外, 這邊還有一個問題, 因為物件的建立在JVM中是很頻繁的行為, 僅僅只是修改一個指標的位置, 在並發\(concurrent\)的情況下也不是thread-safe的, 這部分主要有兩種解決方案:

   **a. 對分配記憶體空間的動作進行同步處理**: 事實上, JVM採用CAS配上retry的機制保證更新操作的原子性

   **b. 把記憶體分配的動作按照執行緒劃分在不同的空間中進行**: 這種作法就是說, 每個執行緒都預先在Java Heap中分配一小塊的記憶體, 就是前一節提到的TLAB. 哪個執行緒想要記憶體, 就在自己的TLAB上分配, 只有TLAB用完並分配新的TLAB時, 才需要進行同步鎖定. 要不要使用TLAB, 可透過參數**-XX:+/-UseTLAB**來設定.

3. **將分配到的記憶體空間都初始化為預設值**: 這個動作不包含物件頭\(Object Header, 之後會提到\), 若啟用TLAB, 也可在TLAB分配時就進行. 這個步驟主要是要保證物件的instance field在Java code中可以不assign init value就能直接使用, 程式可以存取到這些field的資料類型所對應的預設值.

4. **對物件進行必要的設置**: 譬如說這個物件是哪個類別的instance, 怎麼找到class的meta information, object hash code, object的GC

### 物件的記憶體佈局

12

### 物件的存取定位

12

