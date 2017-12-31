# Synchronized v.s. Lock

這篇筆記主要是想記錄一下synchronized保留字與Lock的一些差異: 在Java中處理concurrency issue時, 我們常常會需要做到鎖定某個程式區塊\(critical region\)這種事情, 以避免多個執行緒之間同時在critical region裡面對資料進行修改進而產生不可預期的結果. 當談到鎖定特定程式碼區塊時, 除了synchronized保留字之外, 在java.util.concurrent package裡面還提供了Lock Framework讓我們可以做到類似的效果, 其中常見的Lock工具有:

* Lock\(這只是介面而已, 你要自己選實作\)
* ReentrantLock
* Condition
* ReadWeiteLock
* ReentrantReadWriteLock

接下來, 就是這兩個東西比較常被人們拿出來比較的差異了:

### 彈性

synchronized與Lock有相似的並發特性以及記憶體語意, 但Lock本身還多了一些比較具有彈性的部分, 諸如可設定等待鎖定的時間/中斷鎖定等特性.

首先, 我們先來定義一個情境:  
**有兩個執行緒, 執行緒A與執行緒B, 它們都想要獲得物件O的鎖, 假設執行緒A得到了物件O的鎖, 那基本上執行緒B就必須等待執行緒A釋放掉物件O的鎖之後才可以鎖定物件O了.**

基於這個情境, 我們來看一下用不同方式實作的情況會是如何:

* **使用Synchronized來實作**: 如果執行緒A一直不釋放這個鎖, 執行緒B就會一直在那邊等, 也不能被中斷.

* **使用ReentrantLock來實作**: 如果執行緒A一直不釋放這個鎖, 我們可以透過設定讓執行緒B在等待了一定時間之後, 中斷等待的動作, 去做別的事情.

這是一個我認為ReentrantLock比較有彈性的地方. 再來, 我們來看一下ReentrantLock中常見的幾個方法:

* void lock\(\): 嘗試去獲得\(acquire\)鎖, 如果這個鎖當前被別的執行緒佔用著, 那就一直等待直到獲得鎖為止.
* void lockInterruptibly\(\): 嘗試去獲得鎖直到當前執行緒被中斷. 當鎖還無法被獲得時, 當前執行緒會一直等待直到這個鎖被釋放出來. 如果獲得的話就馬上return.
* boolean tryLock\(\): 嘗試獲得鎖, 如果成功了就回傳true, 反之回傳false.
* boolean tryLock\(long time, TimeUnit unit\): 嘗試獲得鎖, 如果成功了就回傳true, 反之會根據給定的時間參數去等待鎖的釋放. 如果在限定的時間內等到了, 就獲取鎖並且回傳true, 反之回傳false\(timeout\). 
* void unlock\(\): 釋放當前獲得的鎖

### 實作層面以及實作技術上的不同

在實作上, synchronized與Lock也是不同的:

* synchronized: **主要是在JVM層級上實作的**, JVM會透過進入/退出物件的monitor來實現同步\(不管是方法還是程式區塊\), 我沒有特別去區別這兩種\(方法/程式區塊\)實作差在哪. 不過按照網路跟書籍上找的資料, 程式區塊的同步是透過**monitorenter**以及**monitorexit**指令來實作的. 關於monitorenter/monitorexit的說明, 可以參考這本筆記的Chapter 1-5.1 Synchronized的"同步的原理"那段. 除了這點之外, 我們還可以透過一些monitor tool來監控synchronized的鎖定, 而且在程式執行的過程中, 如果出現異常, JVM是可以自動釋放鎖定的.
* Lock: 這邊以ReentrantLock作為討論的出發點, ReentrantLock的實作基本上是依賴於java的 synchronizer framework的, 其使用到的類別叫做**AbstractQueuedSynchronizer \(AQS\)**, AQS會使用一個整數形態的volatile變數來維護同步的狀態. 講白了, 就是說**Lock的實作是在程式層級上實作的**, 其核心概念就是透過**volatile的記憶體語意**以及**CAS操作**來實現的. 關於這部分的詳細說明, 可以參考本筆記的Chapter 1-5 Lock的部分. 另外, 如果你想要保證在使用ReentrantLock的時候, 鎖一定會被釋放, 那你就必須將unlock\(\)方法寫在finally區塊中.

### 效能

在早期的JDK\(JDK1.6之前\)之中, 若在資源競爭不是很激烈的狀況之下, synchronized的性能基本上是比ReentrantLock要來得好的, 但在資源競爭激烈的情況下, synchronized的性能就會被ReentrantLock給海放了, 這個其實google一下"synchronized vs ReentrantLock"就可以看到很多圖表結果. 不過在JDK1.6發佈後, 這個問題基本上就慢慢被彌平了, 因為在JDK1.6之後針對synchronized加入了很多的最佳化手段. 關於這部分, 可以參考本筆記的Chapter 1-5.1 Synchronized的部分.

### 結論

其實在當前的JDK中\(我寫這篇時是2017/12/31, 當前最新的是JDK9\), synchronized與ReentrantLock都是很不錯的選擇. 對我個人而言, synchronized的可讀性是稍微優於ReentrantLock的, 因為通常的寫法就是看你要綁個方法或是某個critical region這樣而已. 然而在使用ReentrantLock的時候, 通常都會出現try/finally區塊, 分別用於tryLock/lock/unlock這些動作, 其實也沒什麼不好, 而且它還可以讓你透過指定時間參數去決定要等多久, 這點在synchronize上是看不到的.











