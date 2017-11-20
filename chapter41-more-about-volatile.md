# More About Volatile

### 前言

前面一篇是站在JMM的角度去詮釋volatile及其背後的運作機制的, 這邊只是要做一些簡單的補充而已, 算是把我還有想到跟讀到的一些東西補進來.

在多執行緒並發的開發情境下, synchronized和volatile都扮演著重要的角色, volatile是輕量級的synchronized, 其在多處理器開發中保證了共享變數的"可見性". 透過之前的章節, 我們可以知道可見性指的就是當一個執行緒修改一個共享變數時, 另外一個執行緒可以讀到這個被修改的值.

### Volatile的官方定義

Java語言規範第三版中對volatile的定義如下: Java允許執行緒存取共享變數, 為了確保共享變數能被準確且一致的更新, 執行緒應該確保通過排他鎖單獨獲取這個變數. Java提供了volatile, 在某些情況下比lock更加方便. 若一個field被宣告成volatile, JMM就會確保所有執行緒看到這個變數的值是一致的.

### 為何要使用volatile

volatile如果使用恰當的話, 相較於synchronized, 其使用和執行成本會更低, 因為其不會引起執行緒context的切換與調度.



