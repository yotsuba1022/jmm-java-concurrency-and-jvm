# More About Volatile

### 前言

前面一篇是站在JMM的角度去詮釋volatile及其背後的運作機制的, 這邊只是要做一些簡單的補充而已, 算是把我還有想到跟讀到的一些東西補進來.

在多執行緒並發的開發情境下, synchronized和volatile都扮演著重要的角色, volatile是輕量級的synchronized, 其在多處理器開發中保證了共享變數的"可見性". 透過之前的章節, 我們可以知道可見性指的就是當一個執行緒修改一個共享變數時, 另外一個執行緒可以讀到這個被修改的值.

### Volatile的官方定義

* Java語言規範第三版中對volatile的定義如下: 
  Java允許執行緒存取共享變數, 為了確保共享變數能被準確且一致的更新, 執行緒應該確保通過排他鎖單獨獲取這個變數. Java提供了volatile, 在某些情況下比lock更加方便. 若一個field被宣告成volatile, JMM就會確保所有執行緒看到這個變數的值是一致的.

### 為何要使用volatile

* volatile如果使用恰當的話, 相較於synchronized, 其使用和執行成本會更低, 因為其不會引起執行緒context的切換與調度.

### Volatile的實作原理

* 那麼volatile是如何來保證可見性的呢? 在x86處理器下通過工具獲取JIT編譯器生成的組合語言指令來看看volatile進行寫入操作時, CPU會做哪些事情:  
  ![](/assets/jmm-85.png)

  有volatile變數修飾的共享變數進行寫入操作時會多出第二行的組合語言代碼, 通過查詢IA-32架構軟體開發者手冊可知, lock prefix的指令在多核心處理器下會引發兩件事情:

  * **將當前處理器快取塊的資料寫回至系統記憶體**

  * **這個寫回記憶體的操作會引起在其它CPU裡快取了該記憶體地址的資料無效化**

* 處理器為了提高處理速度, 不直接和記憶體進行通訊,而是先將系統記憶體的資料讀到內部快取\(L1, L2或其它\)後再進行操作, 但操作完後不確定何時會寫入記憶體, 若對宣告為volatile的變數進行寫入操作, JVM就會向處理器發送一條Lock prefix的指令, 將這個變數所在的cache line之資料寫回至系統記憶體. 但是即使寫回了記憶體, 若其它處理器的快取裡的值還是舊的, 再執行計算操作就會出現問題, 所以在多處理器情境下, 為了確保各個處理器的快取是一致的, 就會實作**快取一致性機制\(cache coherency mechanism\)**,**每個處理器通過偵測在bus上傳播的資料來檢查自己快取的值是否已經過期, 當處理器發現自己的cache line所對應的記憶體位置已經被修改後, 就會將當前處理器的cache line設置成無效狀態, 當處理器要對這個資料進行修改操作時, 會強制重新從系統記憶體裡把資料讀到處理器的快取中**.

  這兩件事在IA-32 Architecture Software Developer Manuals的第三冊的多處理器管理章節\(MULTIPLE-PROCESSOR MANAGEMENT\)中有詳細描述.

* **Lock prefix指令會引起處理器快取回寫\(write-back\)至記憶體.** Lock prefix指令導致在執行指令期間, 斷言\(assert, 這邊我不確定中文怎麼翻比較傳神\)處理器的LOCK\#信號. **在多處理器環境中, LOCK\#信號確保在斷言該信號的期間, 處理器可以獨佔使用任何共享記憶體.\(因為其會鎖住bus, 導致其他CPU不能存取bus, 不能存取bus就意味著不能存取系統記憶體\), 但是在最近的處理器裡, LOCK\#信號一般不會鎖住bus, 而是鎖快取, 畢竟鎖bus的開銷比較大**. 在8.1.4章節有詳細說明鎖定操作對處理器快取的影響, 對於Intel486和Pentium處理器, 在鎖操作時, 總是在bus上斷言LOCK\#信號. 但在P6和最近的處理器中, 若存取的記憶體區域已經快取在處理器內部, 則不會斷言LOCK\#信號. 相反地, 其會鎖定這塊記憶體區塊的快取並且回寫致記憶體, 並使用快取一致性機制來確保修改的原子性, 此操作被稱為"cache locking", **快取一致性機制會阻止同時修改被兩個以上處理器快取的記憶體區塊資料**. \(硬體的部分我不太懂, 所以這邊從簡體中文翻譯過來後可能沒那麼精準, 關於這一段, 英文原文文件的截圖如下:\)  
  ![](/assets/jmm-86.png)  
  **一個處理器的快取回寫至記憶體會導致其它處理器的快取無效**. IA-32處理器和Intel64處理器使用**MESI\(Modified, Exclusive, Shared, Invalid\)**協議去維護內部快取和其他處理器快取的一致性. 在多核心處理器系統中進行操作的時候, IA-32與Intel64處理器嗅探其它處理器存取系統記憶體和它們的內部快取. 它們使用嗅探技術保證它的內部快取, 系統記憶體和其它處理器的快取的資料在bus上保持一致. 例如在Pentium和P6 family處理器中, 若通過嗅探一個處理器來檢測其它處理器打算寫記憶體位置, 而這個位置當前正在處理共享狀態, 那麼正在嗅探的處理器將無效其cache line, 在下次存取相同記憶體位置時, 強制執行cache line fill. 在原文\(MULTIPLE-PROCESSOR MANAGEMENT\)章節的一開始就有對這部分做簡單的描述:  
  ![](/assets/jmm-87.png)

### 參考資料

* [Intel® 64 and IA-32 Architectures Software Developer Manuals](https://software.intel.com/en-us/articles/intel-sdm) \(上面提到的第三冊是指vol-3a那本\)
* [MESI protocol](https://en.wikipedia.org/wiki/MESI_protocol)



