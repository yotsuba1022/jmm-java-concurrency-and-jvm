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

4. **對物件進行必要的設置**: 譬如說這個物件是哪個類別的instance, 怎麼找到class的meta information, object hash code, object的Generational GC Age\(分代年齡\)的資訊. 這些資訊基本上都放在物件的物件頭\(Object Header\)中. 根據JVM當前運行狀態的不同, 如是否啟用biased locking等\(參閱Chapter 1-5.1 Synchronized\), 物件頭會有不同的設定方式.

在以上步驟都結束之後, 一個新的物件基本上就產生了, 但從Java程式的角度來看, 物件的建立才剛要開始, 因為init方法還沒執行, 所有的field都還沒有根據開發者的意願進行初始化, 等這部分也完成, 一個真正可用的物件才算是完全產生出來.

### 物件的記憶體佈局

在HotSpot中, 物件在記憶體中儲存的佈局可以分成以下三塊:

* **Object Header**: 物件頭基本上分成兩部分, 如下所述.  
  
  a. **Mark Word**: 儲存物件自身的runtime data, 像是hash code, Generational GC Age, lock status flag, 執行緒持有的lock, biased thread id等等, 這部分的資料長度在32/64位元的JVM中分別為32bit/64bit. 物件頭資訊是與物件自身定義的資料無關的額外儲存成本, 考慮到JVM的空間效率, mark word被設計成一個非固定的資料結構以便在極小的空間內儲存盡可能多的資訊, 其會根據物件的狀態重用自己的儲存空間, 這部分可參閱Chapter 1-5.1 Synchronized.

  b. **Klass Pointer**: 此即物件指向其class metadata的指標, JVM通過這個指標來確定這個物件是哪個class的instance. 不過並不是所有的JVM實作都要靠這個東西, 換句話說, 找metadata不一定要經過物件本身, 這個後面會提到. 這邊要額外提的是, 若物件是一個Java陣列, 那在物件頭中還必須有一塊用來記錄陣列長度的資料, 因為從物件的metadata可以確定Java物件的大小, 但是從陣列的metadata不能確定陣列的大小.

* **Instance Data**:

* **Padding**:

### 物件的存取定位

12

