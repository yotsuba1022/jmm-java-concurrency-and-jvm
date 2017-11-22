# Synchronized

### 前言

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



