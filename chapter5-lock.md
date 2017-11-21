# Lock

### Lock的釋放\(release\)-獲取\(acquire\)建立的happens before關係

* Lock是Java concurrency中最重要的同步機制. Lock除了讓critical region互斥執行之外, 還可以讓釋放lock的執行緒向獲取同一個lock的執行緒發送訊息.

* 以下是lock release-acquire的範例程式:  
  ![](/assets/jmm-36.png)

* 假設執行緒A執行write方法, 隨後執行緒B執行read方法: 根據happens-before規則, 這個過程包含的happens-before關係可以分為兩類:

  * 根據程式順序規則, 1 happens before 2, 2 happens before 3; 4 happens before 5, 5 happens before 6.
  * 根據**monitor lock規則**, **3 happens before 4**.
  * 根據happens before的傳遞性, 2 happens before 5.

* 上述happens before關係的圖形化表現形式如下:  
  ![](/assets/jmm-37.png)  
  在上圖中, 每一個箭頭連接的兩個節點, 代表了一個happens before關係. 紫色箭頭表示**程式順序規則**; 橙色箭頭表示**monitor lock規則**; 湖水綠色箭頭表示組合這些規則後提供的**happens before保證**.

* 上圖表示在執行緒A釋放了lock之後, 隨後執行緒B獲取同一個lock. 在圖中, 2 happens before 5. 因此, 執行緒A在釋放lock之前所有可見的共享變數, 在執行緒B獲取同一個lock之後, 將立刻變得對B執行緒可見.

### Lock釋放和獲取的記憶體語意

* **當執行緒釋放lock時, JMM會把該執行緒對應的區域記憶體中的共享變數更新到主記憶體中**. 以上面的MonitorExample為例, A執行緒釋放lock後, 共享資料的狀態示意圖如下:  
  ![](/assets/jmm-38.png)

* **當執行緒獲取lock時, JMM會把該執行緒對應的區域記憶體置為無效. 從而使得被monitor保護的critical region code必須要從主記憶體中去讀取共享變數.** 下圖是獲取lock的狀態示意圖:  
  ![](/assets/jmm-39.png)

* 對比lock釋放-獲取的記憶體語意與volatile寫-讀的記憶體語意, 可以得出以下兩點結論:

  * lock釋放與volatile寫具有相同的記憶體語意.
  * lock獲取與volatile讀具有相同的記憶體語意.

* 以下對lock釋放與lock獲取的記憶體語意做個總結:

  * 執行緒A釋放一個lock, 實質上是執行緒A向接下來將要獲取這個lock的某個執行緒發出了\(執行緒A對共享變數所做修改的\)訊息.
  * 執行緒B獲取一個lock, 實質上是執行緒B接收了之前某個執行緒發出的\(在釋放這個lock之前對共享變數所做修改的\)訊息.
  * 執行緒A釋放lock, 隨後執行緒B獲取這個lock, 這個過程實質上是執行緒A通過主記憶體向執行緒B發送訊息.

### Lock記憶體語意的實現

* 這邊會以ReentrantLock的source code來分析lock記憶體語意的具體實作機制, 首先, 請看以下範例程式:  
  ![](/assets/jmm-40.png)  
  在ReentrantLock中, 呼叫lock\(\)方法獲取lock; 呼叫unlock\(\)方法釋放lock.

* ReentrantLock的實作依賴於java synchronizer framework - **AbstractQueuedSynchronizer** \(下文簡稱AQS\). AQS使用一個整數型態的volatile變數\(其被命名為state\)來維護同步狀態, 稍後也會在JDK的原始碼中看到這個volatile變數是如何扮演實現ReentrantLock記憶體語意實作的關鍵.

* 下圖是ReentrantLock的class diagram \(僅畫出與本文相關的部分\):  
  ![](/assets/jmm-41.png)

* 額外附贈一張IntelliJ產生的精美簡圖\(~~絕非業配文~~\):  
  ![](/assets/jmm-42.png)

* ReentrantLock分為**公平鎖\(fair\)**與**非公平鎖\(non-fair\)**, 首先分析公平鎖:

  * 使用公平鎖時, 上鎖的方法lock\(\)之invoke trace如下:  
    1. ReentrantLock: lock\(\)  
    2. FairSync: lock\(\)  
    3. AbstractQueuedSynchronizer: acquire\(int arg\)  
    4. ReentrantLock: tryAcquire\(int acquires\)

    在第4步真正開始上鎖, 以下是該方法的原始碼:  
    ![](/assets/jmm-43.png)  
    從上面的原始碼可以看出, 上鎖之前首先要讀取volatile變數state\(int c\), 如果你去點圖中的那個getState\(\)方法的話, 會發現這個是AQS裡面的方法, 然後會回傳AQS的**state**\(int type\):  
    ![](/assets/jmm-88.png)

  * 使用公平鎖時, 解鎖的方法unlock\(\)之invoke trace如下:  
    1. ReentrantLock: unlock\(\)  
    2. AbstractQueuedSynchronizer: release\(int arg\)  
    3. ReentrantLock.Sync: tryRelease\(int releases\)

    在第3步真正開始釋放鎖, 以下是該方法的原始碼:  
    ![](/assets/jmm-44.png)  
    從上面的原始碼可以看出, 解鎖的最後要寫volatile變數state.

  * 公平鎖在釋放鎖的最後寫volatile變數state; 在獲取鎖時首先讀這個volatile變數. 根據volatile的happens-before規則, **釋放鎖的執行緒在寫volatile變數之前可見的共享變數, 在獲取鎖的執行緒讀取同一個volatile變數後, 將立刻變得對獲取鎖的執行緒可見**.

* 現在再來看看非公平鎖的記憶體語意之實作

  * 非公平鎖的釋放和公平鎖是一樣的, 故此處僅分析非公平鎖的獲取. 使用非公平鎖時, 加鎖方法lock\(\)之invoke trace如下:  
    1. ReentrantLock: lock\(\)  
    2. NonfairSync: lock\(\)  
    3. AbstractQueuedSynchronizer: compareAndSetState\(int expect, int update\)

    在第3步真正開始上鎖, 以下是該方法的原始碼:  
    ![](/assets/jmm-45.png)  
    該方法以原子操作的方式更新state變數, 本文把java的compareAndSet\(\)方法簡稱為**CAS**. 根據JDK對該方法的說明: **如果當前狀態值等於預期值, 則以原子方式將同步狀態設置為給定的更新值**. 此操作具有volatile讀/寫的記憶體語意.

  * 這裡我們分別從編譯器和處理器的角度來分析, CAS如何同時具有volatile讀與volatile寫的記憶體語意.

  * 在之前的章節有提到過以下兩點:

    * 編譯器不會對volatile讀與volatile讀後面的任意記憶體操作進行重排序

    * 編譯器不會對volatile寫與volatile寫前面的任意記憶體操作進行重排序

    * 組合以上兩個條件, 意味著為了同時實現volatile讀/寫的記憶體語意, 編譯器不能對CAS與CAS前面與後面的任意記憶體操作進行重排序.

  * 下面再來看看在常見的intel x86處理器裡, CAS是如何同時具有volatile讀/寫的記憶體語意的

  * 以下是sum.misc.Unsafe class的compareAndSwapInt\(\)方法的原始碼:  
    ![](/assets/jmm-46.png)  
    可以看到這是個native method invocation. 這個native method在openjdk依次呼叫的c++程式為:  
    1. unsafe.cpp  
    2. atomic.cpp  
    3. atomicwindowsx86.inline.hpp

    這個native method的最終實現如下:  
    \[[part 1](https://github.com/JetBrains/jdk8u_hotspot/blob/master/src/os_cpu/windows_x86/vm/atomic_windows_x86.inline.hpp#L62-L69)\]  
    \[[part 2](https://github.com/JetBrains/jdk8u_hotspot/blob/master/src/os_cpu/windows_x86/vm/atomic_windows_x86.inline.hpp#L216-L226)\]  
    如上面的原始碼所示, 程式會根據當前處理器的類型來決定是否為**cmpxchg**指令添加lock prefix. 如果程式是在多處理器上運行, 就為cmpxchg指令加上lock prefix \(lock cmpxchg\). 反之, 若程式是在單處理器上運行, 就省略lock prefix \(單處理器本身會維護單處理器內的順序一致性, 不需要lock prefix提供的記憶體屏障效果\).

  * Intel的手冊對lock prefix的說明如下:

    * 確保對記憶體的讀-改-寫操作是原子執行的. **在Pentium及Pentium之前的處理器中, 帶有lock prefix的指令在執行期間會鎖住bus, 使得其他處理器暫時無法通過bus存取記憶體**. 很顯然地, 這會帶來高昂的開銷. 從Pentium 4, Intel Xeon及P6處理器開始, Intel在原有bus lock的基礎上做了一個很有意義的最佳化: **若要存取的記憶體區域\(area of memory\)在lock prefix執行期間已經在處理器內部的快取中被鎖定\(即包含該記憶體區域的cache line當前處於獨佔或已修改的狀態\), 並且該記憶體區域被完全包含在單個cache line中, 那麼處理器將會直接執行該指令.**由於在指令執行期間該cache line會一直被鎖定, 其他處理器無法讀/寫該指令要存取的記憶體區域, 因此能保證指令執行的原子性. 這個操作過程叫做快取鎖定\(cache locking\), 快取鎖定將大大降低lock prefix指令的執行開銷, 但是當多處理器之間的競爭程度很高或著指令存取的記憶體地址未對齊時, 仍然會鎖住bus.

    * 禁止該指令與之前和之後的讀/寫指令進行重排序.

    * 把write buffer中的所有資料更新到記憶體中

  * 上面的第二點跟第三點所具有的記憶體屏障效果, 足以同時實現volatile read/write的記憶體語意. 經過上面的這些分析, 就可以了解為什麼JDK文件說CAS同時具有volatile read/write的記憶體語意了

  * 再來, 對公平鎖和非公平鎖的記憶體語意做個總結:

    * 公平鎖和非公平鎖釋放時, 最後都要寫一個volatile變數state.

    * 公平鎖獲取時, 首先會去讀這個volatile變數.

    * 非公平鎖獲取時, 首先會用CAS更新這個volatile變數, 這個操作同時具有volatile read/write的記憶體語意.

  * 在本文中對ReentrantLock的分析可以看出, lock的釋放-獲取之記憶體語意的實作至少有以下兩種方式:

    * 利用volatile變數的寫-讀所具有的記憶體語意.

    * 利用CAS所附帶的volatile read/write的記憶體語意.

### Concurrent Package的實作

* 由於java的CAS同時具有volatile read/write的記憶體語意, 因此Java執行緒之間的通信現在有了以下四種方式:

  * A執行緒寫volatile變數, 隨後B執行緒讀取此volatile變數.
  * A執行緒寫volatile變數, 隨後B執行緒用CAS更新這個volatile變數.
  * A執行緒用CAS更新一個volatile變數, 隨後B執行緒用CAS更新這個volatile變數.
  * A執行緒用CAS更新一個volatile變數, 隨後B執行緒讀取這個volatile變數.

* Java的CAS會使用現代處理器上提供的高效機器級別原子指令, 這些原子指令以原子方式對記憶體執行讀-改-寫操作, 這是在多處理器中實現同步的關鍵\(從本質上來講, 能夠支持原子性讀-改-寫指令的計算機, 是順序計算圖靈機的非同步等價機器, 因此任何現代的多處理器都會去支持某種能對記憶體執行原子性讀-改-寫操作的原子指令\). 同時, volatile變數的read/write和CAS可以實現執行緒之間的通信. 把這些特性整合在一起, 就形成了整個concurrent package得以實現的基石. 如果仔細分析concurrent package的原始碼實作, 會發現一個通用化的實作模式:  
  1. 首先, 聲明共享變數為volatile  
  2. 然後, 使用CAS的原子條件更新來實現執行緒之間的同步  
  3. 同時, 配合以volatile的read/write和CAS所具有的volatile read/write記憶體語意來實現執行緒之間的通信

* AQS, non-blocking data structure和atomic變數類別\(java.util.concurrent.atomic package中的class\), 這些concurrent package中的基礎類別都是使用這種模式來實作的, 而concurrent package中的高階類別又是依賴於這些基礎來實作的. 從整體來看, concurrent package的實作示意圖如下:  
  ![](/assets/jmm-47.png)



