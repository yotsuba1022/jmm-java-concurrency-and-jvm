# Fork/Join Framework

### 何謂fork/Join

Fork/Join是從JDK7開始提供的一個用於平行執行任務的框架, 是一個可以把母任務切割成多個子任務, 並且最終彙整各個子任務的結果並得到母任務之結果的框架.

* Fork: 即把一個母任務切割成多個子任務並且平行地執行.
* Join: 合併切割後的子任務之執行結果, 最終得到母任務之結果.

例如: 計算整數1~10000的加總, 可以切割成100個子任務, 每個子任務分別對100個整數進行加總, 最終彙整這100個子任務的結果.

Fork/Join的運作流程圖如下:

![](/assets/jmm-111.png)

### Work-Stealing演算法

Work-stealing演算法是指某個執行緒從其它queue裡"**竊取\(steal\)**"任務來執行, 其運作流程圖如下:

![](/assets/jmm-112.png)

至於為何要有這種演算法呢? 假設當前有一個比較大的母任務, 我們可以把這個母任務切割為若干個互不相依的子任務, 然而為了減少執行緒之間的競爭, 便把這些子任務分別放到不同的queue裡, 並**為每個queue建立一個單獨的執行緒來執行queue裡的任務, 執行緒與queue一一對應, 即A執行緒負責處理queue A裡的任務, **但是有的執行緒可能會先把自己的queue中的任務完成, 而其它執行緒對應的queue裡卻還有任務在等待處理中. 所以, 已經完成所有任務的執行緒與其在那邊乾等, 倒不如去幫其它執行緒完成剩下的任務, 於是這個已經沒事做的執行緒就會去其它的queue裡竊取一個任務來執行. 在這種情況下, 它們會存取同一個queue, 所以為了減少竊取任務的執行緒與被竊取任務的執行緒之間的競爭, 通常會使用雙向佇列\(**double-ended queue**, a.k.a Deque, 其發音為"deck"\) --- **被竊取任務的執行緒永遠從雙向佇列的頭部取出任務來執行, 而竊取任務的執行緒永遠從雙向佇列的尾部取出任務來執行**.

Work-stealing的優點: 充分地利用執行緒進行平行運算.

Work-stealing的缺點: 在某些情況下還是存在競爭, 譬如雙向佇列裡只有一個任務時. 除了這點之外, 另一個缺點是其消耗了更多的系統資源, 譬如建立了多個執行緒以及雙向佇列.

### Fork/Join的基本介紹

前面的部分已經清楚地說明了Fork/Join框架的需求了, 再來就可以思考一下若要設計一個Fork/Join框架的話該要如何設計.

第一步, **分割任務**: 首先需要有一個fork類別來把母任務分割成子任務, 若分割後的子任務還是太大, 就繼續分割, 直到子任務夠小為止.

第二部, **執行任務並合併結果**: 分割的子任務分別放在雙向佇列中, 然後幾個啟動的執行緒分別從雙向佇列裡取得任務並執行. 子任務執行完的結果都統一放在一個佇列裡, 啟動一個執行緒從這個佇列裡拿資料, 並且合併這些資料.

Fork/Join使用兩個類別來完成上述的兩件事情:

* **ForkJoinTask**: 要使用Fork/Join框架, 必須先建立一個ForkJoin任務. 其提供在任務中執行fork\(\)與join\(\)操作之機制, 通常情況下我們不需要直接繼承ForkJoinTask類別, 而只需要繼承其子類別, 如下:
  * **RecursiveAction: 用於沒有回傳值的任務.**
  * **RecursiveTask: 用於有回傳值的任務.**
* **ForkJoinPool**: ForkJoinTask需要通過ForkJoinPool來執行, 任務分割出的子任務會添加到當前工作執行緒所維護的雙向佇列中, 進入佇列的頭部. 當一個工作執行緒的佇列中暫時沒有任務時, 其會隨機地從其它工作執行緒的佇列之尾部獲取一個任務.

### 使用Fork/Join

這邊就用一個簡單的範例來示範怎麼使用Fork/Join, 需求為: **計算1+2+3+4+....+100的結果**.

使用Fork/Join框架首先要考慮到的是如何分割任務, 若我們希望每個子任務最多執行四個數字的相加, 那麼我們可以設置threshold為4, 由於是100個數字相加, 所以Fork/Join框架會把這個任務fork成25個子任務, 子任務一負責計算1+2+3+4, 子任務二負責5+6+7+8 ..., 以此類推, 最後再join所有子任務的結果.

因為這是個有結果\(回傳值\)的任務, 所以必須繼承**RecursiveTask**, 實作內容如下:

```java
package idv.java.ccr.jsr133.forkjoin;

import java.util.concurrent.RecursiveTask;

/**
 * @author Carl Lu
 */
public class CountTask extends RecursiveTask<Integer> {

    private static final int THRESHOLD = 4;
    private int start;
    private int end;

    public CountTask(int start, int end) {
        this.start = start;
        this.end = end;
    }

    @Override
    protected Integer compute() {
        int sum = 0;

        boolean computable = ( end - start ) <= THRESHOLD;

        if (computable) {
            for (int i = start; i <= end; i++) {
                sum += i;
            }
        } else {
            int middle = ( start + end ) / 2;
            CountTask leftSubTask = new CountTask(start, middle);
            CountTask rightSubTask = new CountTask(middle + 1, end);

            leftSubTask.fork();
            rightSubTask.fork();

            int resultOfLeft = leftSubTask.join();
            int resultOfRight = rightSubTask.join();
            sum = resultOfLeft + resultOfRight;

            if (leftSubTask.isCompletedAbnormally()) {
                System.out.println(
                        "Left sub task completed abnormally, exception message: " + leftSubTask.getException().getMessage());
            }

            if (rightSubTask.isCompletedAbnormally()) {
                System.out.println(
                        "Right sub task completed abnormally, exception message: " + leftSubTask.getException().getMessage());
            }
        }

        return sum;
    }
}
```

```java
package idv.java.ccr.jsr133.forkjoin;

import idv.java.ccr.util.ThreadColor;

import java.util.concurrent.ExecutionException;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.Future;

/**
 * @author Carl Lu
 */
public class ForkJoinDemo {

    public static void main(String[] args) {

        ForkJoinPool forkJoinPool = new ForkJoinPool();
        CountTask task = new CountTask(1, 100);
        Future<Integer> result = forkJoinPool.submit(task);
        try {
            System.out.println(ThreadColor.ANSI_BRIGHT_GREEN + "Sum result of 1 to 100: " + result.get());
        } catch (InterruptedException | ExecutionException ignore) {
        }

    }

}
```

執行結果:

![](/assets/jmm-121.png)

範例程式碼commit紀錄[點我](https://github.com/yotsuba1022/java-concurrency/commit/17bc4fa36cd5e9c758eb545f4bcad5fa82b42dd6).

通過這個範例我們可以再看更深一點, 關於ForkJoinTask, 其與一般任務之主要區別在於其需要實作**compute**方法, 在這個方法中, 首先需要判斷任務是否足夠小, 若夠小就直接執行任務; 反之就必須進行任務分割. 每個子任務在呼叫fork方法時, 又會進入compute方法, 看看當前的子任務是否需要繼續往下分割成更多的子任務, 若不需要, 就執行當前子任務並且回傳結果. 使用**join**方法則會等待子任務執行完成並且得到其結果.

### Fork/Join中的Exception Handling

ForkJoinTask在執行的時候可能會拋出exception, 但是我們沒辦法於main thread中直接catch這些exception, 故ForkJoinTask提供了**isCompletedAbnormally\(\)**方法來檢查任務是否已經拋出exception或是已經被取消了, 並且可以通過ForkJoinTask的**getException\(\)**方法取得exception. 使用上大概長得像這樣\(擷取自前面的範例程式碼\):  
![](/assets/jmm-122.png)  
getException\(\)回傳Throwable物件, 若任務被取消了則回傳CancellationException. 若任務沒有完成若著沒有拋出exception則回傳null.

### Fork/Join的實作原理

ForkJoinPool由ForkJoinTask陣列與ForkJoinWorkerThread陣列組成, ForkJoinTask陣列負責存放程式提交給ForkJoinPool的任物, 而ForkJoinWorkerThread陣列則負責執行這些任務.

#### ForkJoinTask的fork方法實作原理:

當我們呼叫ForkJoinTask的fork方法時, 程式會呼叫ForkJoinWorkerThread的workQueue\(ForkJoinPool.WorkQueue\)的push方法非同步地執行這個任務, 然後立刻回傳結果, 原始碼如下:

![](/assets/jmm-113.png)

push方法把當前的任務存放在ForkJoinTask陣列queue裡, 然後再呼叫ForkJoinPool的signalWork方法喚醒\(active\)或創造一個工作執行緒來執行任務, 原始碼如下:

![](/assets/jmm-114.png)

#### ForkJoinTask的join方法實作原理:

join方法的主要作用是阻塞\(block\)當前執行緒並且等待獲得結果, 其原始碼如下:

![](/assets/jmm-115.png)

首先, 其呼叫了doJoin\(\), 通過doJoin\(\)得到當前任務的狀態來判斷回傳什麼結果, 任務狀態有四種: **NORMAL**\(已完成\), **CANCELLED**\(被取消\), **SIGNAL**\(信號\)以及**EXCEPTIONAL**\(出現異常\), 如下圖\(**這邊不把MASK當狀態來看待**\):

![](/assets/jmm-116.png)

* 若任務狀態是NORMAL, 則直接回傳任務結果.
* 若任務狀態是CANCELLED, 則直接拋出CancellationException.
* 若任務狀態是EXCEPTIONAL, 則直接拋出對應的異常.

拋出異常的部份\(reportException\)如下圖:

![](/assets/jmm-117.png)

再來, 看一下doJoin的原始碼:

![](/assets/jmm-118.png)

在doJoin中, 首先通過查看任務的狀態, 看是否已經執行完了\(**s &lt; 0, negative means NORMAL**\), 若執行完畢, 則直接回傳任務狀態; 反之, 則從任務陣列裡取出任務並且透過doExec\(\)執行任務, 其中的exec\(\)會由繼承ForkJoinTask的類別實作\(此處由RecursiveTask實作\). 若任務順利執行完了, 則把任務狀態設定為NORMAL; 反之則紀錄exception, 並把任務狀態設為EXCEPTIONAL, doExec\(\)原始碼如下:

![](/assets/jmm-119.png)

最後, join中的getRawResult方法則會交由繼承ForkJoinTask的類別實作\(此處為RecursiveTask\), 看起來就只是把計算完的結果傳回去而已:

![](/assets/jmm-120.png)

### 參考資料

* JDK原始碼\(1.8.0\_152\)



