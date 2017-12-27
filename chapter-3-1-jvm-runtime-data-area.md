# JVM Runtime Data Area

在JVM中, 會把記憶體劃分為若干個不同的資料區塊, 其都有各自的用途以及建立/銷毀的時間, 有些區塊是隨著JVM的啟動而存在; 有些則是依賴於用戶執行緒的啟動/結束來建立與銷毀, 以下就簡單記錄一下各個區塊及其概述.

* **Program Counter Register**: 這塊空間是比較小的記憶體空間, 可看作是當前執行緒所執行的byte code的行號指示器. Byte code直譯器運作時, 就是通過改變這個counter的值來選取下一條要執行的byte code指令. 在任何一個時間點, 一個處理器\(或是多和處理器的一個core\)都只能執行一條執行緒中的指令. 因此為了執行緒切換後能恢復到正確的執行位置, 每條執行緒都需要有一個獨立的program counter, 各執行緒之間互不影響, 獨立儲存, 故此資料區塊是執行緒私有的. 此記憶體區域是唯一一個在JVM規格中沒有規定任何OutOfMemoryError\(後文中皆稱OOM\)情況的區域.

* **JVM Stack**: 跟program counter一樣, 是一種執行緒私有的資料區塊, 其生命週期與執行緒相同. 此區塊描述的是Java method執行的記憶體模型: 每個method在執行的同時都會建立一個stack frame用於儲存局部變數表\(Local Variable Table\), 運算元堆疊\(Operand Stack\), 動態連接\(Dynamic Linking\)...等等訊息. 每個方法從呼叫直至執行完成的過程, 都對應著一個stack frame在此區塊中入/出stack的過程\(push/pop\). 你可能會常常看到一些書中把Java記憶體分成Heap/Stack, 這是比較粗糙的分法, 因為實際上Java記憶體區塊的劃分比這複雜得多. 不過這種分成Heap/Stack的方式也正好說明了大多數的開發人員最關注的, 與物件記憶體分配關係最密切的記憶體區域就是這兩塊了, 其中的Stack就是指JVM Stack中的Local Variable Table這個部分. Local Variable Table儲存了編譯時期可知的各種primitive type, reference type\(不等於物件本身, 可能是一個指向物件起始記憶體位置的指標\)以及returnAddress type\(指向了一條byte code的地址\). Local Variable Table所需的記憶體空間在編譯期間就會完成分配, 當進入一個method時, 這個method需要在frame中分配多大的local variable space是完全確定的, 在method執行期間不會改變Local Variable Table的大小. 關於這個區域, JVM規格中定義了兩種異常狀況:

  * StackOverFlow: 若執行緒請求的stack深度大於VM所允許的深度, 就會拋出此異常.
  * OutOfMemory: 若此區可以自動擴增, 但擴增時要不到足夠的記憶體, 就會拋出此異常.

* **Native Method Stack**: 這個區塊跟JVM Stack很相似, 差異在於JVM Stack主要執行Java method\(byte code\), 但此區塊主要執行native method. 由於JVM規格並沒有對這區做太多的規範, 所以有的VM, 如HotSpot, 就乾脆直接把這區塊跟JVM Stack~~摻在一起做撒尿牛丸了~~. 關於此區塊, JVM規格中也定義了StackOverFlow/OutOfMemory兩種異常.

* **Java Heap**: 對大多數應用程式來說, 這個區塊是JVM管理的記憶體區塊中最大的一塊. 此區塊是被所有的執行緒共享的一塊區域, 在VM啟動後建立. 此區塊的唯一目的就是儲存物件的實例\(object instance\), 基本上所有的物件實例都會在這裡分配記憶體. 在JVM規格中對於這點的描述是: "The heap is the runtime area from which memoty for all class instances and arrays is allocated.", 當然, 也是有例外的. 此區塊也是GC管理的主要區域, 所以又稱為Garbage Collected Heap\(不是垃圾堆...\). 從記憶體回收的角度來看, 由於現在的collector基本上都採用分代收集演算法, 所以此區還可以細分為新生代\(Young Generation\)與老年代\(Tenured Generation\), 新生代一般來說又可分為Eden/From Survivor/To Survivor三個區塊. 從記憶體分配的角度來看, 此區還可能劃分出多個執行緒私有的分配緩衝區\(Thread Local Allocation Buffer, TLAB\). 根據JVM規格, 此區塊可以處於物理上不連續的記憶體空間中, 只要邏輯上是連續的即可, 就如同磁碟空間一般. 此區的大小可透過參數-Xmx/-Xms控制, 若此區中沒有足夠的記憶體完成instance分配,  且也無法再擴增時, 就會拋出OOM.

* **Method Area**: 此區同Java Heap, 是各執行緒共享的記憶體區域, 用於儲存已經被JVM加載的class information/constant/static constant等資料. 其又有一個別名叫做Non-Heap, 目的應該是想跟Java Heap做個區隔. GC行為在這區是比較少出現的, 因為在這區回收的CP值並不是那麼的高, 譬如說對於class的unloading\(其條件很嚴苛\). 此區無法滿足記憶體分配需求時, 會拋出OOM.





* **Runtime Constant Pool**:
* **Direct Memory**:

以上就是JVM中常見的各個資料區塊

留底

