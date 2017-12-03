# Fork/Join Framework

### 何謂fork/Join

Fork/Join是從JDK7開始提供的一個用於平行執行任務的框架, 是一個可以把母任務切割成多個子任務, 並且最終彙整各個子任務的結果並得到母任務之結果的框架.

* Fork: 即把一個母任務切割成多個子任務並且平行地執行.
* Join: 合併切割後的子任務之執行結果, 最終得到母任務之結果.

例如: 計算整數1~10000的加總, 可以切割成100個子任務, 每個子任務分別對100個整數進行加總, 最終彙整這100個子任務的結果.

Fork/Join的運作流程圖如下:  
  
![](/assets/jmm-111.png)

### Work-Stealing演算法

### Fork/Join的基本介紹

### 使用Fork/Join

### Fork/Join中的Exception Handling

### Fork/Join的實作原理

### 參考資料



