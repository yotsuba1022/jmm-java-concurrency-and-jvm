# Synchronized

### 前言

在多執行緒並發的環境中, synchronized是很重要的角色, 也有人稱其為重量級鎖, 但隨著JDK6對synchronized進行了各種最佳化後, 有些情況下它並沒有那麼重了, 這篇會介紹JDK6中為了減少獲得鎖和釋放鎖所帶來的性能消耗而引入的偏向鎖\(Biased Locking\)和輕量級鎖\(Lightweight Locking\), 以及鎖的儲存結構與升級過程.

### 同步的基礎

* Java中的每個物件都可以做為鎖

  * 對於同步方法, 鎖是**當前的實例物件**
  * 對於靜態同步方法, 鎖是**當前物件的class物件**
  * 對於同步方法區塊\(critical region\), 鎖是**synchronized括號裡配置的物件**

* 當一個執行緒嘗試存取critical region時, 它必須先得到鎖, 而退出critical region或是拋出異常時必須釋放鎖. 再來會談到鎖儲存的位置以及其中儲存的訊息.

### 同步的原理

JVM規格規定JVM基於進入與退出monitor物件來實現方法同步以及程式區塊同步, 但兩者的實作細節並不相同. 程式區塊同步是使用**monitorenter**以及**monitorexit**指令實現, 而方法同步是使用另外一種方式實現的, 細節在JVM規格裡面並沒有詳細說明, 但是方法的同步同樣可以使用這兩個指令來實現. monitorenter指令是在編譯後插入到同步程式區塊的開始位置, 而monitorexit是插入到同步程式區塊結束處與異常處, JVM要保證每個monitorenter必須有對應的monitorexit與之配對. 任何物件都有一個monitor與之關聯, 並且當一個monitor被持有後, 其將處於鎖定狀態. 當執行緒執行到monitorenter指令時, 將會嘗試獲取物件所對應的monitor之所有權, 即嘗試獲得該物件的鎖.

* #### Java Object Header

  * Java object header\(以下稱物件頭\)包含了mark word跟klass pointer. 至於mark word的大小, 會隨著JVM而改變:

    * 在**32-bit**的JVM中, 一個word等於**4 byte \(32 bit\)**

    * 在**64-bit**的JVM中, 一個word等於**8 byte \(64 bit\)**

  * 鎖會存在物件頭裡. 如果物件是陣列型態, 則JVM用3個word\(mark word\)儲存物件頭, 如果物件是非陣列型態, 則用**2個word**儲存物件頭.  
    ![](/assets/jmm-89.png)

  * Java物件頭裡的mark word裡預設儲存物件的hash code, GC generation資訊和鎖標記. 32-bit JVM的mark word預設之儲存結構如下:

    ![](/assets/jmm-90.png)

  * 在運行期間mark word裡儲存的資料會隨著鎖標記的變化而作出相應的改變. Mark word可能變化為儲存以下四種資料\(指32-bit JVM中\):  
    ![](/assets/jmm-91.png)

  * 至於在64-bit的JVM下, mark word的大小是64-bit, 其儲存結構如下:  
    ![](/assets/jmm-92.png)
* #### Lock Upgrade

  JDK6為了減少獲得鎖和釋放鎖的過程所帶來的性能消耗, 引入了**biased locking**與**lightweight locking**, 所以在JDK6裡, 鎖一共有四種狀態:

  * **無鎖狀態**

  * **Biased Locking**

  * **Lightweight Locking**

  * **Heavyweight Locking**

    鎖會隨著競爭的情況加劇而逐漸升級, 但是鎖**只能升級, 不能降級**, 這意味著biased locking升級成lightweight locking後, 就沒辦法再降級回biased locking了. 這種只升不降的策略, 其目的是為了**提高獲得鎖和釋放鎖的效率**, 之後會繼續提到.  
    ![](/assets/jmm-93.png)  
    另外要注意的是, 鎖的升級是不需要等待同步程式區塊完成的, 持有鎖的執行緒在執行完同步程式區塊的當下檢查一下鎖是否有被升級, 若升級了, 就喚醒等待的其它執行緒去重新競爭鎖.

* #### Biased Locking

  Hotspot的作者經過以往的研究後發現在大多數的情況下, 鎖不僅不存在多執行緒競爭, 而且總是由同一個執行緒多次獲得, 為了讓執行緒獲得鎖的代價更低而引入了biased locking. **當一個執行緒\(T1\)存取同步程式區塊並且獲得鎖的時候, JVM會把鎖的物件頭中的鎖標記更改為"01"\(即biased locking mode\), 同時使用CAS操作把獲取到這個鎖的執行緒\(T1\)之ID記錄在物件的mark word之中. 之後該執行緒在進入/退出同步程式區塊時就不需要花費CAS操作來加鎖/解鎖或是更換mark word, 而只需要簡單的測試一下物件頭的mark word裡是否儲存著指向當前執行緒的biased locking**, 若測試成功, 表示執行緒已經獲得了鎖, 反之, 則需要再測試一下mark word中biased locking的標記是否設置成"1" \(表示當前是biased locking\), 若沒有設置, 則用CAS競爭鎖, 反之, 則嘗試用CAS將物件頭的biased locking指向當前執行緒.

  Biased locking的撤銷\(**Revoke Bias**\): Biased locking使用了一種**等到競爭出現才會釋放鎖的機制**, 所以當其它執行緒嘗試競爭biased locking的時候, 持有biased locking的執行緒才會釋放鎖, 但這不表示可以隨時撤銷biased lock: biased locking的撤銷, 需要等待一個global的安全點\(**safepoint, 即在當前時間點上沒有任何byte code正在執行, 這裡可以想成是還未進入同步程式區塊或著是已經離開同步程式區塊了**\), 其首先會暫停擁有鎖的執行緒, 然後檢查持有biased locking的執行緒是否活著, 若執行緒不處於活動狀態, 則將物件頭設置成無鎖狀態, 若執行緒仍然活著, 擁有biased locking的stack會被執行, 迭代biased object的鎖紀錄, stack中的鎖紀錄和物件頭的mark word若不是重新偏向其它執行緒, 就是要恢復到無鎖或著標記物件不適合作為biased locking, 最後喚醒暫停的執行緒. 換個說法, 就是當revoke bias動作執行後, 若物件沒有被鎖定, 則鎖標記恢復至01, 且biased locking標記為0的狀態; 但若物件已經被鎖定, 則當下獲得biased locking的執行緒會被hang up, 然後將鎖物件升級成lightweight locking, 且將其mark word中的lock record指向剛才持有biased locking的執行緒的lock record, 最後被block在safepoint的執行緒被釋放, 進入到lightweight locking的執行路徑中. 下圖中的執行緒1展示了biased locking初始化的流程, 執行緒2展示了biased lock撤銷的流程:

  ![](/assets/jmm-94.png)  
  關閉biased locking: biased locking在JDK6/7裡是預設啟用的, 但它在應用程式啟動幾秒鐘之後才會進入activated狀態, 若有必要的話, 可以透過JVM參數來關閉延遲\(**-XX:BiasedLockingStartupDelay=0**\). 若你確定自己的應用程式裡所有的鎖通常情況下都會處在競爭的狀態, 那可以通過JVM參數關閉biased locking\(**-XX:UseBiasedLocking=false**\), 這時候就會預設進入lightweight locking mode.

* #### Lightweight Locking

  Lightweight Locking並不是要用來取代heavyweight locking的, 其本意是在沒有多執行緒競爭的前提下, 減少傳統的重量級鎖使用作業系統的mutex lock所產生之性能消耗.

  Lightweight locking的上鎖: 執行緒在執行同步程式區塊之前, **JVM會先在當前執行緒的stack frame中創造用於儲存鎖紀錄的空間\(Lock Record\), 並將物件頭中的mark word複製到鎖紀錄當中**, 官方稱此行為做"**Displaced Mark Word**". 之後, 執行緒嘗試使用CAS將物件頭中的mark word替換為指向鎖紀錄的指標. 若替換成功, 當前執行緒獲得鎖, 反之, 表示有其它執行緒在競爭, 當前執行緒就會嘗試使用**自旋\(spin\)**來獲取鎖.

  Lightweight locking的解鎖: 此時會使用原子的CAS操作來將displaced mark word替換回到物件頭, 若成功, 則表示沒有競爭發生. 若失敗, 表示當前鎖存在競爭, 鎖就會**膨脹\(inflate\)**成重量級鎖\(Heavyweight Locking\). 下圖是兩個執行緒同時競爭鎖, 導致鎖膨脹的流程圖:

  ![](/assets/jmm-95.png)

  因為spin會消耗CPU, 為了避免無意義的spin\(譬如獲得鎖的執行緒被block了\), 一但鎖升級成heavyweight locking, 就不會再恢復到lightweight locking. 當鎖處於這個狀態下, 其它執行緒試圖獲取鎖時, 都會被block, 當持有鎖的執行緒釋放鎖之後會喚醒這些執行緒, 被喚醒的執行緒就會嘗試使用spin來獲取鎖.

  Lightweight locking程式可以提昇程式同步性能的依據是"**對於絕大部分的鎖, 在整個同步週期內都是不存在競爭的\(就是說後面那些在spin的執行緒在超過spin limit之前, 當前拿到鎖的執行緒就可以完成同步程式區塊內的工作並且讓出鎖, 然後讓後面spin的那些執行緒順利取得鎖, 不會發生spin到timeout的情況\)**". 倘若沒有競爭, lightweight locking使用CAS操作避免了使用mutex的開銷, 但若存在競爭, 除了mutex外, 還額外產生了CAS操作, 因此在有競爭的情況下, lightweight locking會比傳統的heavyweight locking更慢, 所以才要膨脹.

### 鎖的優缺點對比![](/assets/jmm-96.png)

### 參考資料

* [Java Virtual Machine Online Instruction Reference - monitorenter](https://cs.au.dk/~mis/dOvs/jvmspec/ref--44.html)
* [OpenJDK wiki - Synchronization and Object Locking](https://wiki.openjdk.java.net/display/HotSpot/Synchronization)  
* [深入理解JVM: JVM進階特性與最佳實踐\(第二版\)](https://www.tenlong.com.tw/products/9787111421900)



