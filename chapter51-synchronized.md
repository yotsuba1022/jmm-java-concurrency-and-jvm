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

JVM規格規定JVM基於進入與退出monitor物件來實現方法同步以及程式區塊同步, 但兩者的實作細節並不相同. 程式區塊同步是使用monitorenter以及monitorexit指令實現, 而方法同步是使用另外一種方式實現的, 細節在JVM規格裡面並沒有詳細說明, 但是方法的同步同樣可以使用這兩個指令來實現. monitorenter指令是在編譯後插入到同步程式區塊的開始位置, 而monitorexit是插入到方法結束處與異常處, JVM要保證每個monitorenter必須有對應的monitorexit與之配對. 任何物件都有一個monitor與之關聯, 並且當一個monitor被持有後, 其將處於鎖定狀態. 當執行緒執行到monitorenter指令時, 將會嘗試獲取物件所對應的monitor之所有權, 即嘗試獲得該物件的鎖.

* #### Java Object Header
* #### Lock Upgrade
* #### Biased Locking
* #### Lightweight Locking

### 鎖的優缺點對比

### 參考資料

* [Java Virtual Machine Online Instruction Reference - monitorenter](https://cs.au.dk/~mis/dOvs/jvmspec/ref--44.html)
* 

