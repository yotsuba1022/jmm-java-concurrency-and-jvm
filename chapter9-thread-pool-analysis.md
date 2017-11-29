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

    * **corePoolSize**  
      Thread pool的基本大小, 當提交一個task到thread pool時, thread pool會建立一個執行緒來執行任務, 即使其它空閑的基本執行緒能夠執行新的task, thread pool還是會建立執行緒, 等到需要執行的task數量大於thread pool的基本大小時就不會再建立了. 若呼叫了thread pool的prestartAllCoreThreads方法, thread pool會提前建立並啟動所有基本執行緒:

      ![](/assets/jmm-98.png)

    * **maximumPoolSize**  
      Thread pool最大的size, 即其允許建立的最大執行緒數目. 若queue滿了, 並且已經建立的執行緒數目小於最大執行緒數目, 則thread pool會再建立新的執行緒來執行task. 要注意的是若使用了unbounded task queue, 這個參數就沒什麼效果了.

    * **keepAliveTime**  
      Thread pool的工作執行緒空閑後, 保持存活的時間. 若task很多, 並且每個task的執行時間都不長, 就可以調大這個時間, 提高執行緒的利用頻率.

    * **unit**  
      執行緒活動保持時間的單位, 可選的時間單位如下

      * DAYS

      * HOURS

      * MINUTES

      * MILLISECONDS

      * MICROSECONDS

      * NANOSECONDS

    * **workQueue**  
      用以保存等待執行的task之blocking queue, 大致上有以下幾種選擇:

      * **ArrayBlockingQueue**: 是一種基於陣列結構的bounded blocking queue\(有界阻塞佇列\), 此queue按FIFO原則對元素進行排序.

      * **LinkedBlockingQueue**: 是一種基於連結串列結構的optionally-bounded blocking queue\(即可以自己指定界限大小的佇列\), 此queue也是以FIFO排序元素, 吞吐量通常要高於ArrayBlockingQueue. 靜態工廠方法Executors.newFixedThreadPool\(\)就是使用這個queue:

        ![](/assets/jmm-99.png)

      * **SynchronousQueue**: 一個不儲存元素的blocking queue, 每個插入操作必須等到另一個執行緒呼叫移除操作, 否則插入操作會一直處於blocking狀態, 吞吐量通常要高於LinkedBlockingQueue, 靜態工廠方法Executors.newCachedThreadPool就是用這個queue:

        ![](/assets/jmm-100.png)

      * **PriorityBlockingQueue**: 一個具有優先級別的unbounded blocking queue\(無界阻塞佇列\).

    * **threadFactory**  
      用於設置建立執行緒的工廠, 可以通過執行緒工廠給每個建立出來的執行緒設定更有意義的名字

    * **handler**  
      RejectedExecutionHandler\(飽和策略\), 即當queue跟thread pool都滿了, 表示thread pool處於飽和狀態, 那麼必須採取一種策略去處理提交的新task. 這個策略預設是AbortPolicy, 表示無法處理新task時拋出exception. 下面是JDK5提供的四種策略:

      * **ThreadPoolExecutor.AbortPolicy**: 直接拋出runtime exception \(**RejectedExecutionException**\).

      * **ThreadPoolExecutor.CallerRunsPolicy**: 只有呼叫execute\(\)的那個執行緒來執行task, 這其實提供了一種回饋控制機制\(feedback control mechanism\), 讓提交新task的速率得以減緩.

      * **ThreadPoolExecutor.DiscardPolicy**: 就不處理了, 直接丟掉\(dropped\).

      * **ThreadPoolExecutor.DiscardOldestPolicy**: 若executor當下沒有被shut down, 位於work queue的**head**之task就會被丟掉\(因為它是最老的, oldest\).
* #### 向thread pool提交task

  我們可以使用execute提交task, 但是execute方法是沒有回傳值的, 所以無法判斷task是否被thread pool執行成功. 通過以下原始碼\(java.util.concurrent.Executor, 所有executor service的根介面\)可以知道execute方法輸入的task是一個Runnable的instance:

  ![](/assets/jmm-101.png)  
  當然我們也可以使用submit方法來提交task, 其會回傳一個future, 然後我們就可以通過這個傳回來的future物件來判斷task有沒有執行成功, 通過future的get\(\)來獲得回傳值, get\(\)會block, 直到task完成為止, 而使用get\(long timeout, TimeUnit unit\)則可以指定block一定時間後就回傳, 這時候task就有可能沒有執行完, 以下只截取**java.util.concurrent.ExecutorService**的其中一個submit方法, 其實submit除了**Callable**, 也是可以接收**Runnable**的\(自己翻, 我就不貼在這了\):

  ![](/assets/jmm-102.png)

* #### Thread pool的關閉

  我們可以通過呼叫thread pool的**shutdown\(\)/shutdownNow\(\)**來關閉thread pool, 其原理是迭代thread pool中的worker thread, 然後逐一呼叫執行緒的interrupt\(\)來中斷執行緒, 所以無法回應/中斷的task可能就永遠無法終止. 然而, 這兩種關閉的方式其實存在著一定的區別, **shutdownNow\(\)**首先將thread pool的狀態設成**STOP**, 然後嘗試停止所有正在執行或著暫停task的執行緒, 並**回傳等待執行task的list**, 而**shutdown\(\)**只是將thread pool的狀態設成**SHUTDOWN**狀態, 然後中斷所有閒置的執行緒\(呼叫interruptIdleWorkers\(\)\).

  只要呼叫了這兩個關閉方法的任一個, isShutdown\(\)就會回傳true. 當所有的task都已經關閉後, 才意味著thread pool關閉成功, 這時呼叫isTerminated\(\)會回傳true. 至於應該呼叫哪一種方法來關閉thread pool, 應該由提交到thread pool的task之特性來決定, 一般來說都會呼叫shutdown來關閉, 若task不一定要執行完, 則可以呼叫shutdownNow\(\).

  ![](/assets/jmm-103.png)

### Thread Pool分析

#### 流程分析

Thread pool的主要工作流程如下圖:  
![](/assets/jmm-104.png)

從上圖可以看出, 在提交一個新的task至thread pool時, thread pool的處理流程如下:

1. 首先, thread pool判斷基本的thread pool是否已滿, 若沒有則建立一個worker thread來執行任務, 反之則進入下一道流程.
2. 其次, thread pool判斷work queue是否已滿, 若沒有就將新提交的task儲存在work queue裡, 滿了則進入下一道流程.
3. 最後thread pool會判斷整個pool是否滿了, 若沒滿就建立一個新的worker thread來執行task, 反之交給飽和策略來處理此task.

#### 原始碼分析

以上的流程分析已經很直觀的闡述了thread pool的工作原理, 再來稍微看一下原始碼是怎麼實作的:

![](/assets/jmm-105.png)  
這部分是擷取自java.util.concurrent.ThreadPoolExecutor.java

#### Worker Thread

Thread pool在建立執行緒時, 會將執行緒封裝成worker thread \(Worker\), Worker在執行完task後, 還會無限迭代獲取workQueue裡的task來執行. 我們可以從Worker的run方法看到這點:

![](/assets/jmm-106.png)  
這部分是擷取自java.util.concurrent.ThreadPoolExecutor.Worker.

### 合理的組態Thread Pool

要想合理的組態你的thread pool, 可以先從task的特性分析著手, 這邊有幾個常見的切入點可以作為分析的依據:

#### Task的性質: CPU密集型/IO密集型或著是混合型的task

* 這種性質的task可以用不同規模的thread pool分開處理.

  * **CPU密集型**: 盡可能配少量一點的執行緒, 譬如配置**cpu數量+1**個執行緒的thread pool.

  * **IO密集型**: 由於這種task的執行緒並不是一直在執行, 則可以盡可能配制多一點執行緒, 如**cpu數量的兩倍**.

  * **混合型**: 若可以拆分, 則將其拆分成一個CPU密集型task和一個IO密集型task, 只要這兩個task執行的時間相差不是太大, 那麼分解後執行的吞吐量應該可以高於串連執行的吞吐量; 倘若執行時間相差過大, 就沒必要分解了. 我們可以通過Runtime.getRuntime\(\).availableProcessors\(\)方法得到當前設備上的CPU數量.

#### Task的優先程度: 由高到低

* 這種類型可以使用**PriorityBlockingQueue**來處理: 其可以讓優先度高的task先執行, 但要注意的是若一直都有高優先度的task被提交到queue裡, 那麼低優先度的task可能永遠都執行不到.

#### Task的執行時間: 由長到短

* 這種有時間差異的task可以交給不同規模的thread pool來處理, 或是使用PriorityBlockingQueue也可以, 讓執行時間短的任務先執行.

#### Task的相依性: 是否相依於其它系統資源, 如資料庫

* 相依於DB connection pool的task, 因為執行緒提交SQL後還要等DB回傳結果, 若等待的時間越長, CPU的空閑時間就越長, 那麼執行緒數量應該要設置得越大, 這樣才能更好地利用CPU.

除了上述的手段, 還有一個建議就是: **使用bounded queue, 因為這種queue能增加系統的穩定性與預警能力**, 可以根據需求調大一點, 譬如幾千. 試想以下情境: Thread pool跟workQueue都滿了, 不斷地拋出拋棄task的exception\(RejectedExecutionException\). 這時若發現是DB出了問題, 導致執行SQL變得異常緩慢, 因為thread pool裡的task全都是需要向DB進行query或是insert/update資料的, 所以導致thread pool裡的worker都塞住了, 因而讓task都積在thread pool中. **若這時候的workQueue是unbounded queue, thread pool的workQueue就會越長越大, 進而可能撐爆memory**, 導致系統不能用, 而不再只是task出問題而已了. 當然, 儘管系統所有的task是用單獨的server去做部署的, 且針對不同類型的task使用不同規模的thread pool, 但出現了這種問題時, 還是有可能影響到其它有相依性的task.

### Thread Pool的監控

### 參考資料



