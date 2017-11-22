# Synchronized

### 前言

在多執行緒並發的環境中, synchronized是很重要的角色, 也有人稱其為重量級鎖, 但隨著JDK6對synchronized進行了各種最佳化後, 有些情況下它並沒有那麼重了, 這篇會介紹JDK6中為了減少獲得鎖和釋放鎖所帶來的性能消耗而引入的偏向鎖\(Biased Locking\)和輕量級鎖\(Lightweight Locking\), 以及鎖的儲存結構與升級過程.

### 同步的基礎

* Java中的每個物件都可以做為鎖

  * 對於同步方法, 鎖是當前的實例物件
  * 對於靜態同步方法, 鎖是當前物件的class物件
  * 對於同步方法區塊\(critical region\), 鎖是synchronized括號裡配置的物件



* 當一個執行緒嘗試存取critical region時, 它必須先得到鎖, 而退出critical region或是拋出異常時必須釋放鎖. 再來會談到鎖儲存的位置以及其中儲存的訊息.

### 同步的原理

* #### Java Object Header
* #### Lock Upgrade
* #### Biased Locking
* #### Lightweight Locking

### 鎖的優缺點對比

### 參考資料



