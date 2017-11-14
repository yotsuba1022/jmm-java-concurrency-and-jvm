# Lock

### Lock的釋放\(release\)-獲取\(acquire\)建立的happens before關係

* Lock是Java concurrency中最重要的同步機制. Lock除了讓critical region互斥執行之外, 還可以讓釋放lock的執行緒向獲取同一個lock的執行緒發送訊息.
* 以下是lock release-acquire的範例程式:
* 假設執行緒A執行write方法, 隨後執行緒B執行read方法: 根據happens-before規則, 這個過程包含的happens-before關係可以分為兩類:
  * 根據程式順序規則, 1 happens before 2, 2 happens before 3; 4 happens before 5, 5 happens before 6.
  * 根據monitor lock規則, 3 happens before 4.
  * 根據happens before的傳遞性, 2 happens before 5.
* 上述happens before關係的圖形化表現形式如下:  
  在上圖中, 每一個箭頭連接的兩個節點, 代表了一個happens before關係. 紫色箭頭表示程式順序規則; 橙色箭頭表示monitor lock規則; 湖水綠色箭頭表示組合這些規則後提供的happens before保證.

* 上圖表示在執行緒A釋放了lock之後, 隨後執行緒B獲取同一個lock. 在圖中, 2 happens before 5. 因此, 執行緒A在釋放lock之前所有可見的共享變數, 在執行緒B獲取同一個lock之後, 將立刻變得對B執行緒可見.

### Lock釋放和獲取的記憶體語意

* 當執行緒釋放lock時, JMM會把該執行緒對應的區域記憶體中的共享變數更新到主記憶體中. 以上面的MonitorExample為例, A執行緒釋放lock後, 共享資料的狀態示意圖如下:
* 當執行緒獲取lock時, JMM會把該執行緒對應的區域記憶體置為無效. 從而使得被monitor保護的critical region code必須要從主記憶體中去讀取共享變數. 下圖是獲取lock的狀態示意圖:
* 對比lock釋放-獲取的記憶體語意與volatile寫-讀的記憶體語意, 可以得出以下兩點結論:
  * lock釋放與volatile寫具有相同的記憶體語意.
  * lock獲取與volatile讀具有相同的記憶體語意.
* 以下對lock釋放與lock獲取的記憶體語意做個總結:
  * 執行緒A釋放一個lock, 實質上是執行緒A向接下來將要獲取這個lock的某個執行緒發出了\(執行緒A對共享變數所做修改的\)訊息.
  * 執行緒B獲取一個lock, 實質上是執行緒B接收了之前某個執行緒發出的\(在釋放這個lock之前對共享變數所做修改的\)訊息.
  * 執行緒A釋放lock, 隨後執行緒B獲取這個lock, 這個過程實質上是執行緒A通過主記憶體向執行緒B發送訊息.

### Lock記憶體語意的實現

### Concurrent Package的實作



