# Synchronized v.s. Lock

在Java中處理concurrency issue時, 我們常常會需要做到鎖定某個critical region這種事情, 以避免多執行緒之間同時在critical region裡面對資料進行修改進而產生不可預期的結果. 當談到鎖定特定程式碼區塊時, 除了synchronized保留字之外, 在java.util.concurrent package裡面提供了Lock Framework, 其中常見的Lock工具有:

* Lock\(這只是介面而已, 你要自己選實作\)
* ReentrantLock
* Condition
* ReadWeiteLock
* ReentrantReadWriteLock

這篇筆記主要是想記錄一下synchronized保留字與Lock的一些差異, 畢竟這兩個東西有相似的並發特性以及記憶體語意, 而Lock本身還多了一些比較具有彈性的特性, 諸如可設定等待鎖定的時間/中斷鎖定等特性.

首先, 我們先來定義一個情境:   
**有兩個執行緒, 執行緒A與執行緒B, 它們都想要獲得物件O的鎖, 假設執行緒A得到了物件O的鎖, 那基本上執行緒B就必須等待執行緒A釋放掉物件O的鎖之後才可以鎖定物件O了.**

基於這個情境, 我們來看一下用不同方式實作的情況會是如何:

* **使用Synchronized來實作**: 如果執行緒A一直不釋放這個鎖, 執行緒B就會一直在那邊等, 也不能被中斷.

* **使用ReentrantLock來實作**: 如果執行緒A一直不釋放這個鎖, 我們可以透過設定讓執行緒B在等待了一定時間之後, 中斷等待的動作, 去做別的事情.





