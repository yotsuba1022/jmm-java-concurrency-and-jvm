# The Implementation of Atomic Operations

### 前言

原子\(atom\)的本意是"無法再被進一步分割的最小粒子", 至於原子操作\(atomic operation\)則是"**不可被中斷的一個或一系列操作**". 在多處理器上實作原子操作就變得有點複雜, 這篇要紀錄的心得是Intel處理器與Java實作原子操作的方式.

### 處理器如何實作原子操作

32-bits的IA-32處理器使用了基於對快取上鎖或對匯流排\(bus\)上鎖的方式來實作多處理器之間的原子操作

* #### 處理器自動保證基本記憶體操作的原子性

  首先, 處理器會自動保證基本的記憶體操作之原子性. 處理器保證從系統記憶體當中讀取或著寫入一個byte是原子的, 即當一個處理器讀取一個byte時, 其它處理器不能存取這個byte的記憶體位置. Pentium6和最新的處理器能自動保證單處理器對同一個cache line裡進行16/32/64 bits的操作是原子的, 但對於複雜的記憶體操作來說, 處理器不能自動保證其原子性, 譬如跨bus寬度, 跨多個cache line, 跨page table的存取. 但處理器提供bus locking以及cache locking這兩種機制來保證複雜記憶體操作的原子性.

* #### 使用bus lock保證原子性

  此機制是通過對bus上鎖保證原子性: 若多個處理器同時對共享變數進行讀改寫\(i++就是讀改寫的一種例子\)操作, 那麼共享變數就會被多個處理器同時進行操作, 這樣讀改寫操作就不是原子的了, 操作完之後共享變數的值會和期望的不一致, 譬如說, 若i = 1, 且由兩個執行序進行兩次i++操作, 這時候期望得到i = 3, 但結果卻有可能是2.

  原因是可能有多個處理器同時從各自的cache中讀取變數i, 並分別進行+1的操作, 然後分別寫入系統記憶體內. 若想要保證讀改寫共享變數的操作是原子的, 就必須保證CPU1讀改寫變數i的時候, CPU2不能操作該變數i在記憶體地址的快取.

  處理器使用bus locking就是要解決這種問題的, 所謂的bus locking就是使用處理器提供的一個LOCK\#訊號, 當一個處理器在bus上輸出這個訊號時, 其它處理器的request就會被blocking, 然後當前這個處理器就可以獨佔地使用共享記憶體.

* #### 使用cache lock保證原子性

  第二個機制是通過cache locking來保證原子性. 在同一時刻我們只需要保證對某個記憶體地址的操作是原子性即可, 但bus locking把CPU與記憶體之間的通訊給鎖住了, 這就會使得在鎖定期間, 其它處理器不能操作其它記憶體地址的資料, 所以bus locking的開銷比較大就是這樣, 在近代的新型處理器中, 對於某些場合下可能就會使用cache locking替代bus locking以達到最佳化.

  頻繁使用的記憶體會快取在處理器的L1/L2/L3高速快取裡, 那麼原子操作就可以直接在處理器內部的快取中進行, 並不需要斷言\(assert\)bus locking, 在Pentium6以及近代的新處理器中可以使用cache locking的方式來實作複雜的原子操作. 所謂的cache locking就是若快取在處理器的cache line中的記憶體區域在LOCK操作期間被鎖定, 當它執行lock操作回寫\(write-back\)記憶體時, 處理器不會於bus上斷言LOCK\#訊號, 而是修改內部的記憶體位置, 並允許它的快取一致性機制\(cache coherency mechanism\)來保證操作的原子性, 因為快取一致性機制會阻止同時修改被兩個以上的處理器快取的記憶體區域內的資料, 當其它處理器回寫已被鎖定的cache line的資料時會引起cache line無效, 在前面的範例中, 當CPU1修改cache line中的i時使用cache locking, 那麼CPU2就不能同時快取i的cache line.

  要注意的是, 在兩種情況下, 處理器不會使用cache locking:

  * 當操作的資料不能被快取在處理器內部, 或操作的資料跨多個cache line的時候, 則處理器會使用bus locking.

  * 有些處理器不支持cache locking, 對這種處理器來說, 就算鎖定的記憶體區域在處理器的cache line中, 還是照樣使用bus locking.

  以上兩個機制我們可以通過Intel處理器提供的LOCK prefix指令來實作, 常見的如cmpxchg等, 被這些指令操作的記憶體區域就會上鎖, 導致其它處理器不能同時存取之.

### Java如何實作原子操作

在Java中可以透過lock與循環CAS的方式來實作原子操作

* #### 使用循環CAS實作原子操作

  JVM中的CAS正是利用了前面提到的cmpxchg指令來實作的. 自旋CAS實作的基本思路就是循環進行CAS操作直到成功為止, 以下範例程式實作了一個基於CAS且thread-safe的counter方法threadSafeCount\(\)和一個non thread-safe的counter方法nonThreadSafeCount\(\):

```java
package idv.java.ccr.jsr133.cas;
import idv.java.ccr.util.ThreadColor;
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadPoolExecutor;

/**
 * @author Carl Lu
 */
public class CasDemo {

    private final static int MAX_COUNT = 10000;
    private final static int MAX_THREAD_NUM = 300;
    private final static int EXPECTED_TOTAL_COUNT = MAX_COUNT * MAX_THREAD_NUM;

    public static void main(String[] args) {
        final Counter counter = new Counter();
        ThreadPoolExecutor executorService = (ThreadPoolExecutor) Executors.newFixedThreadPool(MAX_THREAD_NUM);
        long startTime = System.currentTimeMillis();

        for (int createRound = 0; createRound < MAX_THREAD_NUM; createRound++) {
            Thread thread = new Thread(() -> {
                for (int count = 0; count < MAX_COUNT; count++) {
                    counter.nonThreadSafeCount();
                    counter.threadSafeCount();
                }
            });
            executorService.execute(thread);
        }

        /*
        * If we didn't invoke shutdown() here,
        * it might cause the incorrect output results since that there might still have some tasks are not finished yet.
        * */
        executorService.shutdown();

        long endTime = System.currentTimeMillis();
        long completedTaskCount = executorService.getCompletedTaskCount();
        int nonAtomicResult = counter.getNonAtomicInteger();
        int atomicResult = counter.getAtomicInteger().get();
        double loseRate = (double) ( EXPECTED_TOTAL_COUNT - nonAtomicResult ) / EXPECTED_TOTAL_COUNT;

        System.out.println(ThreadColor.ANSI_MAGENTA + "Non thread-safe integer result: " + nonAtomicResult);
        System.out.println(ThreadColor.ANSI_CYAN + "Thread-safe atomic integer result: " + atomicResult);
        System.out.println("Counting lose rate: " + loseRate);
        System.out.println("Total cost time: " + ( endTime - startTime ) + " msecs.");
        System.out.println(ThreadColor.ANSI_BRIGHT_GREEN + "Complete task count: " + completedTaskCount);
    }

}
```

```java
 package idv.java.ccr.jsr133.cas;

import java.util.concurrent.atomic.AtomicInteger;

/**
 * @author Carl Lu
 */
public class Counter {

    private AtomicInteger atomicInteger = new AtomicInteger(0);
    private int nonAtomicInteger = 0;

    public void threadSafeCount() {
        for (; ; ) {
            int current = atomicInteger.get();
            boolean success = atomicInteger.compareAndSet(current, ++current);
            if (success) {
                break;
            }
        }
    }

    public void nonThreadSafeCount() {
        nonAtomicInteger++;
    }

    public AtomicInteger getAtomicInteger() {
        return atomicInteger;
    }

    public int getNonAtomicInteger() {
        return nonAtomicInteger;
    }

}
```

原始碼commit history[點我](https://github.com/yotsuba1022/java-concurrency/commit/1c28b548af7283c9eb2340529317ea5500a51b58)

在Java concurrency package中有一些並發框架也使用了自旋CAS的方式來實作原子操作, 譬如說LinkedTransferQueue的xfer\(\)方法. CAS雖然可以很高效的解決原子操作, 但CAS仍然存在三大問題:

* **ABA問題**  
  因為CAS需要於操作值的時候檢查一下值有沒有發生變化, 若沒有發生變化就更新, 但是若一個值原本是A, 變成了B, 然後又變成了A, 那麼使用CAS進行檢查時會發現這個值沒有發生變化, 可是實際上是真的有變化的. ABA問題的解決思路就是使用版好. 在變數前面追加版號, 每次變數更新的時候把版號加一, 那麼A-B-A就會變成1A-2B-3A. 從JDK5開始, atomic package裡面提供了一個AtomicStampedReference來解決ABA問題. 這個class的compareAndSet\(\)方法的作用是首先檢查當前參照是否等於預期參照, 並且當前的stamp是否等於預期的stamp, 若全都相等, 就以原子方式將該參照及stamp的值設定為給定的新值. 以下是compareAndSet\(\)的原始碼截圖:

  ![](/assets/jmm-110.png)

* **循環時間長且開銷大**  
  自旋CAS若長時間不成功, 會給CPU帶來非常大的執行開銷. 若JVM能支持處理器提供的pause指令那麼效率會有一定的提升, pause指令有兩個作用, 第一是可以延遲管線執行指令\(de-pipeline\), 使CPU不會消耗過多的執行資源, 延遲的時間取決於具體實作的版本, 在一些處理器上延遲時間可能會是0. 其二是它可以避免在退出循環的時候因為有記憶體順序衝突\(memory order violation\)因而引起CPU pipeline被清空\(CPU pipeline flush\), 從而提高CPU的執行效率.

* **只能保證一個共享變數的原子操作**  
  當對一個共享變數執行操作時, 我們可以使用循環CAS的方式來保證原子操作, 但對多個共享變數操作時, 循環CAS就無法保證操作的原子性, 這時就可以用lock, 或著有一個取巧的方法, 就是把多個共享變數合併成一個共享變數來操作.從JDK5開始提供了AtomicReference類別來保證參照物件之間的原子性, 你可以把多個變數放在一個物件裡並且進行CAS操作\(這邊我還沒讀完, 等我讀完再來寫一篇\).

* #### 使用lock機制實作原子操作

  Lock機制保證了只有獲得lock的執行緒能夠操作鎖定的記憶體區域. JVM內部實作了很多種lock機制, 有biased locking, lightweight locking以及heavyweight locking\(mutex lock\). 其中, 除了biased locking, JVM實作lock的方式都用到了循環的CAS, 當一個執行緒想進入同步區塊的時候使用循環CAS的方式來獲得鎖, 當其退出同步區塊的時候使用循環CAS釋放鎖. 這部分在這本讀書筆記的chapter 5.1有簡單的介紹.



