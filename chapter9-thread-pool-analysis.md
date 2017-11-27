# Thread Pool Analysis

### 前言

合理地利用thread pool可以帶來以下好處:

1. **降低資源的消耗**: 通過重複利用已經創造好的執行緒降低執行緒的創造/銷毀所造成的消耗
2. **提高回應速度**: 當task到達後, task不用等待執行緒創造就能立即執行
3. **提高執行緒的可管理性**: 執行緒是很珍貴的資源, 若無限制的創造, 不止消耗系統資源, 同時也會降低系統的穩定性, 使用thread pool可以進行統一的分配, 最佳化與監控. 但是要做到合理的利用thread pool, 必須先對其原理有一定程度的瞭解.

### Thread Pool的使用

* #### Thread pool的建立

  * 我們可以通過ThreadPoolExecutor來建立一個thread pool:

    ![](/assets/jmm-97.png)

  * 建立一個thread pool需要以下參數:

    * corePoolSize: Thread pool的基本大小, 當提交一個task到thread pool時, thread pool會建立一個執行緒來執行任務, 即使其它空閑的基本執行緒能夠執行新的task, thread pool還是會建立執行緒, 等到需要執行的task數量大於thread pool的基本大小時就不會再建立了. 若呼叫了thread pool的prestartAllCoreThreads方法, thread pool會提前建立並啟動所有基本執行緒:

      ![](/assets/jmm-98.png)

    * maximumPoolSize: 用以保存等待執行的task之blocking queue, 大致上有以下幾種選擇:

      * **ArrayBlockingQueue**: 是一種基於陣列結構的bounded blocking queue\(有界阻塞佇列\), 此queue按FIFO原則對元素進行排序.

      * **LinkedBlockingQueue**: 是一種基於連結串列結構的optionally-bounded blocking queue\(即可以自己指定界限大小的佇列\), 此queue也是以FIFO排序元素, 吞吐量通常要高於ArrayBlockingQueue. 靜態工廠方法Executors.newFixedThreadPool\(\)就是使用這個queue:  
        ![](/assets/jmm-99.png)

      * **SynchronousQueue**: 一個不儲存元素的blocking queue, 每個插入操作必須等到另一個執行緒呼叫移除操作, 否則插入操作會一直處於blocking狀態, 吞吐量通常要高於LinkedBlockingQueue, 靜態工廠方法Executors.newCachedThreadPool就是用這個queue.  
        ![](/assets/jmm-100.png)

      * **PriorityBlockingQueue**: 一個具有優先級別的unbounded blocking queue\(無界阻塞佇列\).

    * keepAliveTime

    * unit

    * workQueue

    * threadFactory

    * handler
* #### 向thread pool提交task
* #### Thread pool的關閉

### Thread Pool分析

### 合理的組態Thread Pool

### Thread Pool的監控

### 參考資料



