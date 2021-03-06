# Appendix

下表是關於文章中的一些術語的整理:

| **Term** | **English** | **Description** |
| :--- | :--- | :--- |
| 共享變數 | Shared Variable | 在多個執行緒之間能夠被共享的變數. 其包含了所有的實例變數, 靜態變數和陣列元素. 它們都被存放在heap記憶體中, volatile只會作用於共享變數. |
| 記憶體屏障 | Memory Barriers | 是一組處理器指令, 用於實作對記憶體操作的順序限制. |
| 快取行\(塊\) | Cache Line/Cache Block | 快取中可以分配的最小儲存單位. 處理器填寫cache block時會加載整個cache line, 需要使用多個主記憶體的讀取週期. |
| 原子操作 | Atomic Operation | 不可中斷的一個或一系列操作. |
| 快取行\(塊\)填充 | Cache Line/Cache Block Fill | 當處理器識別到從記憶體中讀取的運算元是可以被快取的, 處理器讀取整個cache block到適當的快取\(L1, L2, L3的或所有\). |
| 快取命中 | Cache Hit | 若進行高速cache block填充操作的記憶體位置仍然是下次處理器存取的地址時, 處理器從快取中讀取運算元, 而不是從記憶體. |
| 寫入命中 | Write Hit | 當處理器將運算元寫回一個記憶體快取區域時, 其首先會檢查此快取的記憶體地址是否在cache block中, 若存在一個有效的cache block, 則處理器將這個運算元寫回至快取, 而不是寫回記憶體. |
| 寫缺失 | Write Misses the Cache | 一個有效的cache block被寫至不存在的記憶體區域. |
| CAS | Compare and Swap | 比較並且設值. 用於硬體層面上提供原子性操作. CAS操作基本上需要輸入兩個數值, 一個舊值\(期望操作之前的值\), 一個新值, 在操作期間先比較舊值是否發生變化, 若沒有, 才交換成新值, 反之則不交換. 在Intel處理器中, CAS通過cmpxchg指令實作. |
| 執行緒安全 | Thread Safe | 當多個執行緒存取同一個物件時, 若不用考慮這些執行緒在運作時環境下的調度與交替執行, 也無需額外的同步, 或著在呼叫方進行任何其它的協調操作, 這個物件的行為都可以獲得正確的結果, 那這個物件就是執行緒安全的. 這其實隱喻了一件事: 程式本身封裝了所有必要的正確性保障手段\(如互斥同步等\). |
| 自旋 | Spin \(spin-waiting\) | 所謂的自旋, 就是讓執行緒執行一個busy loop, 這樣這個執行緒就會透過自旋來等待某個其想要獲取的鎖, 而當前持有鎖的執行緒也不用放棄CPU的執行時間. 自旋本身雖然避免了執行緒切換的開銷, 但其是要佔用處理器時間的\(有執行緒自旋的那個處理器\). |
| 鎖的粗化 | Lock Coarsening | 若JVM探測到有一串零碎的操作都對同一個物件上鎖, 就可能把上鎖同步的範圍擴展\(粗化, coarsening\)到整個操作序列的外部. 常見的情境如String物件的連續append\(by StringBuilder\). |
| CPU管線 | CPU Pipeline | CPU pipeline的運作方式就像工廠裡的裝配流水線, 在CPU中由多個不同功能的電路單元組成一條指令處理管線, 然後將一條x86指令分成多個步驟後再由這些電路單元分別執行, 如此即可實現在一個CPU clock cycle內完成一條指令, 藉此提高CPU的運算速度. 換個角度想, 這有點像是用面積在換效能的感覺, 即邏輯電路擺多一點, 就有可能同時完成由多個指令組合而成的複雜指令. |
| 記憶體順序衝突 | Memory Order Violation | 這一般是指由false-sharing所引起的, 所謂的false-sharing是指多個CPU\(或是想成執行緒\)同時修改一個cache line的不同部分而引起其中一個CPU\(執行緒\)的操作無效, 當出現這種情況時, CPU必須清空pipeline. |



