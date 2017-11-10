# Volatile

### volatile的特性

* 當我們宣告共享變數為volatile後, 對這個變數的讀/寫就會變得比較特別. 理解volatile特性的一個好方法是: 把對volatile變數的單個讀/寫, 看成是使用同一個monitor lock對這些單個讀/寫操作做了同步.以下通過具體的範例來說明, 首先是範例程式:
* 假設有多個執行緒分別呼叫上面範例的三個方法, 那這個程式在語意上和下面這個範例程式是等價的:
* 如上面範例程式所示, 對一個volatile變數的單個讀/寫操作, 與對一個普通變數的讀/寫操作使用同一個monitor lock來同步, 其之間的執行效果是相同的.
* Monitor lock的happens-before規則保證釋放monitor和獲取monitor的兩個執行緒之間的記憶體可見性, 這意味著對一個volatile變數的讀, 總是能看到\(任意執行緒\)對這個volatile變數最後的寫入.
* Monitor lock的語意決定了臨界區\(critical region\)的執行具有原子性. 這意味著即使是64-bit的long/double變數, 只要其為volatile變數, 對該變數的讀寫就將具有原子性. 倘若是多個volatile操作或類似於volatile++這種複合操作, 這些操作整體上就不具有原子性了.
* 簡單來說, volatile變數本身具有下列特性:
  * 可見性\(visibility\): 對一個volatile變數的讀, 總是能看到\(任意執行緒\)對這個volatile變數最後的寫入.
  * 原子性\(atomicity\): 對任意單個volatile變數的讀/寫具有原子性, 但對於volatile++這種複合操作就不具備原子性.
* 副作用
  * 在Java裡, volatile關鍵字有一個副作用: 刷新快取\(flush the cache\), 以便所有其它地方看到資料的最新版本, 這在大多數情況下其實是很嚴格的, 但這種副作用, 在某些時候也可以用來保障可見性, 這種情況常被稱為"Piggyback", 在下一章節\(Lock\)中, 就可以看到應用此副作用的地方\(CAS\).

### volatile寫入-讀取建立的happens-before關係

* 上面提到的是volatile變數自身的特性, 對開發者來說, volatile對執行緒的記憶體可見性的影響比volatile自身的特性更為重要, 也更需要我們去關注. 從JSR-133開始, volatile變數的寫-讀可以實現執行緒之間的通信.
* 從記憶體語意的角度來說 我們可以觀察到以下對應關係:
  * volatile與monitor lock有相同的效果
  * volatile寫與monitor lock的釋放有相同的記憶體語意
  * volatile讀與monitor lock的獲取有相同的記憶體語意
* 以下程式片段是使用volatile變數的範例程式:

  * 假設執行緒A執行write method之後, 執行緒B才執行read method. 根據happens before規則, 這個過程建立的happens before關係可以分為兩類:

    1. 根據程式順序規則, 1 happens before 2; 3 happens before 4.
    2. 根據volatile規則, 2 happens before 3.  
    3. 根據上述兩條happens before規則與遞移律, 1 happens before 4.

* 上述happens before關係的圖形化表現形式如下:  
  在上圖中, 每一個箭頭所連接的兩個節點, 都代表了一個happens before關係. 紫色箭頭表示程式順序規則; 橙色箭頭表示volatile規則; 湖水綠色箭頭表示組合這些規則後提供的happens before保證.

* 這裡執行緒A寫一個volatile變數後, 執行緒B讀同一個volatile變數. 執行緒A在寫volatile變數之前的所有可見的共享變數, 在B執行緒讀同一個volatile變數後, 就會立即變得對執行緒B可見.

### volatile寫入-讀取的記憶體語意

* volatile write的記憶體語意如下:
  * 當寫入一個volatile變數時, JMM會把該執行緒對應的區域記憶體中的共享變數更新到主記憶體中. 以上面的範例程式VolatileExample3為例, 假設執行緒A首先執行write方法, 隨後執行緒B執行read方法,
    初始時兩個執行緒的區域記憶體中的flag和a都是初始狀態. 下圖是執行緒A執行volatile write後, 共享變數的狀態示意圖:  
    如上圖所示, 執行緒A在寫入flag變數後, 區域記憶體A中被執行緒A更新過的兩個共享變數的值被更新到主記憶體中. 此時, 區域記憶體A和主記憶體中的共享變數的值是一致的.
* volatile read的記憶體語意如下:
  * 當讀一個volatile變數時, JMM會把該執行緒對應的區域記憶體置為無效. 執行緒接下來將從主記憶體中讀取共享變數.
* 以下是執行緒B讀同一個volatile變數後, 共享變數的狀態示意圖:  
  如上圖所示, 在讀取flag變數後, 區域記憶體B已經被置為無效. 此時, 執行緒B必須從主記憶體中讀取共享變數. 執行緒B的讀取操作將導致區域記憶體B與主記憶體中的共享變數的值同步.

* 若此處把volatile寫和volatile讀這兩個步驟綜合起來看, 在執行緒B讀取一個volatile變數後, 執行緒A在寫這個volatile變數之前的所有可見的共享變數的值, 都將會立即變得對執行緒B可見.

* 下面對volatile write/read的記憶體語意做個總結:

  * 執行緒A寫一個volatile變數, 實質上是執行緒A向接下來將要讀這個volatile變數的某個執行緒發出了\(其對共享變數所在修改的\)訊息.
  * 執行緒B讀一個volatile變數, 實質上是執行緒B接收了之前某個執行緒發出的\(在寫這個volatile變數之前對共享變數所做修改的\)訊息.
  * 執行緒A寫一個volatile變數, 隨後執行緒B讀這個volatile變數, 這個過程實質上是執行緒A通過主記憶體向執行緒B發送訊息.

### volatile記憶體語意的實作

* 再來, 看看JMM如何實作volatile write/read的記憶體語意, 在前面的章節中有提到過重排序分為編譯器重排序與處理器重排序.  
  為了實現volatile記憶體語意, JMM會分別限制這兩種類型的重排序類型. 下圖是JMM針對編譯器制定的volatile重排序規則表:  
  舉例來說, 當程式順序中, 第一個操作為普通變數的讀/寫時, 若第二個操作為volatile write, 則編譯器不能重排序這兩個操作.

* 從上表我們可以看出:

  * 當第二個操作是volatile write時, 不管第一個操作是什麼, 都不能重排序. 這個規則確保volatile write之前的操作不會被編譯器重排序到volatile write之後.
  * 當第一個操作是volatile read時, 不管第二個操作是什麼, 都不能重排序. 這個規則確保volatile read之後的操作不會被編譯器重排序到volatile read之前.
  * 當第一個操作是volatile write, 第二個操作是volatile read時, 不能重排序.

* 為了實現volatile的記憶體語意, 編譯器在生成byte code時, 會在指令序列中插入記憶體屏障來禁止特定類型的處理器重排序. 對於編譯器來說, 發現一個最佳佈局來最小化插入屏障的總數幾乎不可能, 為此, JMM採取保守策略. 以下是基於保守策略的JMM記憶體屏障插入策略:

  * 在每個volatile write操作的前面插入一個StoreStore屏障

  * 在每個volatile write操作的後面插入一個StoreLoad屏障

  * 在每個volatile read操作的後面插入一個LoadLoad屏障

  * 在每個volatile read操作的後面插入一個LoadStore屏障

    上述的記憶體屏障插入策略非常保守, 但其可以保證在任意處理器平台, 任意的程式中都能得到正確的volatile記憶體語意.

* 下圖是保守策略下, volatile write插入記憶體屏障後生成的指令順序示意圖:  
  上圖中的StoreStore屏障可以保證在volatile write之前, 其前面的所有normal write操作已經對任意處理器可見了. 這是因為StoreStore屏障將保障上面所有的normal write在volatile write之前更新到主記憶體.

  這裡的一個小亮點是volatile write後面的StoreLoad屏障. 這個屏障的作用是避免volatile write與後面可能有的volatile read/write操作重排序. 因為編譯器常常無法準確判斷在一個volatile write的後面, 是否需要插入一個StoreLoad屏障\(譬如, 一個volatile write之後方法立刻return\). 為了保證能正確實現volatile的記憶體語意, JMM在這裡採取了保守策略: 在每個volatile write的後面或在每個volatile read的前面插入一個StoreLoad屏障. 從整體執行效率的角度考慮, JMM選擇了在每個volatile write的後面插入一個StoreLoad屏障.

  因為volatile write-read記憶體語意的常見使用模式是: 一個執行緒寫volatile變數, 多個執行緒讀取這個volatile變數. 當讀取的執行緒數量大大超過寫入的執行緒時, 選擇在volatile write之後插入StoreLoad屏障將帶來可觀的執行效率之提升. 從這裡我們可以看到JMM在實現上的一個特點: 首先確保正確性, 然後才去追求執行效率.

* 下圖是在保守策略下, volatile read插入記憶體屏障後生成的指令順序示意圖:  
  上圖中的LoadLoad屏障用來禁止處理器把上面的volatile read與下面的normal read重排序. LoadStore屏障用來禁止處理器把上面的volatile read與下面的normal write重排序.

* 上述volatile write和volatile read的記憶體屏障插入策略非常保守. 在實際執行時, 只要不改變volatile write-read的記憶體語意, 編譯器就可以根據具體情況省略不必要的屏障. 以下通過具體的範例程式來說明:  
  針對readAndWrite method, 編譯器在生成byte code的時候可以做如下的最佳化:  
  注意, 最後的StoreLoad屏障不能省略. 因為第二個volatile write之後, 方法立刻return. 此時編譯器可能無法準確斷定後面是否會有volatile read/write, 為了安全起見, 編譯器常常會在這裡插入一個StoreLoad屏障.

* 上面的最佳化是針對任意處理器平台, 由於不同的處理器有不同"鬆緊度"的處理器記憶體模型, 記憶體屏障的插入還可以根據具體的處理器記憶體模型繼續最佳化. 以x86處理器為例, 上圖中除了最後的StoreLoad屏障之外, 其它的屏障都會被省略. 故前面保守策略下的volatile read/write, 在x86處理器平台可以最佳化成:

* 前面的章節提到過, x86處理器僅會對write-read操作進行重排序. 其不會對read-read, read-write與write-write進行重排序, 因此在x86處理器中會省略掉這三種操作類型對應的記憶體屏障. 在x86中, JMM僅需在volatile write後面插入一個StoreLoad屏障即可正確實現volatile write-read的記憶體語意. 這意味著在x86處理器中, volatile write的開銷比volatile read的開銷會大很多\(因為執行StoreLoad屏障開銷會較大\).

### JSR-133為何要增強volatile的記憶體語意

* 在JSR-133之前的舊JMM中, 雖然不允許volatile變數之間的重排序, 但舊的JMM允許volatile變數與普通變數之間重排序. 在舊的記憶體模型中, VolatileExample3範例程式可能被重排序成下列順序來執行:
  在舊的記憶體模型中, 當1和2之間沒有資料相依性時, 1和2之間就可能被重排序\(3與4類似\). 其結果就是: 執行緒B執行4時, 不一定能看到執行緒A在執行1時對共享變數的修改.
  因此在舊的記憶體模型中, volatile的write-read沒有monitor lock的release-acquire所具有的記憶體語意. 為了提供一種比monitor lock更輕量級的機制以利執行緒之間的通信, JSR-133決定增強volatile的記憶體語意: 嚴格限制編譯器和處理器對volatile變數與普通變數的重排序, 確保volatile的write-read和監視器的release-acquire一樣, 具有相同的記憶體語意. 從編譯器重排序規則和處理器記憶體屏障插入策略來看, 只要volatile變數與普通變數之間的重排序可能會破壞volatile的記憶體語意, 這種重排序就會被編譯器重排序規則和處理器記憶體屏障插入策略禁止.
  由於volatile僅僅保證對單個volatile變數的read/write具有原子性, 而monitor lock的互斥執行之特性可以確保對整個臨界區程式的執行具有原子性. 在功能上, monitor lock比volatile更強大; 在可伸縮性與執行性能上, volatile更有優勢. 若要在程式中使用volatile, 請務必謹慎.

### 結論

* Happens-before rule和volatile記憶體語意的實作不能混淆
  * Happens-before是JMM向我們提供的記憶體可見性保證
  * volatile記憶體語意的實作\(包含volatile的編譯器重排序規則和volatile的記憶體屏障插入策略\), 可以理解為JMM如何去實現這些happens-before
* 重排序是有分別的
  * 上文提到的JMM針對編譯器制定的重排序規則表, 是針對編譯器的
  * StoreLoad等記憶體屏障則是針對處理器的
* 除了可見性之外, 把物件的參照\(reference\)聲明為volatile還有什麼作用呢?
  * 有, 像是任意執行緒都可以看到這個物件參照的最新值, 以及對這個物件參照的寫-讀可以實現執行緒之間的通信. 但要注意的是, 若物件的狀態在發布後將發生改變, 那就需要額外的同步了, 原因很簡單, 雖然volatile物件參照可以保證物件的安全發布, 但是無法保證物件安全發布後, 某個執行緒對這個物件的狀態\(指物件的member fields\)的更改, 能夠被其他執行緒看到.
* 關於文中提到的: "當讀一個volatile變數時, JMM會把該執行緒對應的區域記憶體置為無效. 執行緒接下來將從主記憶體中讀取共享變數."
  * 在&lt;Java Concurrency in Practice&gt;的3.1.4節, "Volatile Variables"中, 對volatile有如下描述: **Volatile variables are not cached in registers or in caches where they are hidden from other processors, so a read of a volatile variable always returns the most recent write by any thread.**
    這段話的意思是說: volatile變數不會被"快取"在暫存器或是"快取"在對其它處理器不可見的地方, 因此當前執行緒對一個volatile變數的讀, 總是能讀取到任意執行緒對這個volatile變數最後的寫入.
* 在&lt;JSR-133: Java Memory Model and Thread Specification&gt;的"3. Informal Semantics" 以及&lt;The Java Language Specification Third Edition&gt;的"17.4.5 Happens-before Order"中, 都定義了以下的volatile規則:

  * **A write to a volatile field happens-before every subsequent read of that volatile.**  
    意思是說: 對一個volatile field的寫入, happens-before於任意後續對這個volatile field的讀取. 要注意的是, volatile規則需要一個前提: \(一個執行緒\)寫一個volatile變數之後, \(任意執行緒\)讀取這個volatile變數.

* Volatile對任意單個volatile變數的讀/寫具有原子性:

  * 在&lt;The Java Language Specification Java SE 7 Edition&gt;的17.7章, 有以下描述: **a single write to a non-volatile long or double value is treated as two separate writes. Writes and reads of volatile long and double values are always atomic.**  
    其實這也就是說, Java語言規範不保證對long/double的寫入具有原子性, 但當我們把long/double宣告為volatile後, 對這個變數的寫入就會具有原子性了.

* Volatile變數的讀取happens-before volatile變數的寫入, 這是否也是正確的呢?

  * 不正確, JMM沒有保證這種行為. 具體的原因, 可以參考volatile的編譯器重排序規則表與volatile的記憶體屏障插入策略. volatile的編譯器重排序規則表和volatile的記憶體屏障插入策略都是針對volatile變數的寫入happens-before volatile變數的讀取來設計的. JMM將其設計成這樣, 就是為了讓volatile的寫-讀可以實現執行緒之間的記憶體可見性通信.



