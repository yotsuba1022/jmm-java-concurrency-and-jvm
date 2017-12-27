# DIY Exception

前面講了那麼多記憶體區塊, 各區塊上可能也會有噴錯的情況, 以下就列出我自己實作的結果, 因為我很懶, 所以這邊就直接附上commit連結而已.

* Java Heap OOM: [點我](https://github.com/yotsuba1022/java-concurrency/commit/2158f637da3f598a86c3e81028c1aca973555c79)
* Java VM Stack SOF: [點我](https://github.com/yotsuba1022/java-concurrency/commit/51ffea9fe598eb0681495ff049f38468d2a4aeed)
* Java VM Stack OOM: [點我](https://github.com/yotsuba1022/java-concurrency/commit/a5713778b4a0645c8f87c85e96796c443d7118ed)
* Runtime Constant Pool OOM: [點我](https://github.com/yotsuba1022/java-concurrency/commit/59cc5e8456834a617e329b8c8895bd3ba91b6821)
* Java Method Area OOM: [點我](https://github.com/yotsuba1022/java-concurrency/commit/c11486db65fbf16c9f29f716a86eaa0f2b3a93a4)
* Direct Memory OOM: [點我](https://github.com/yotsuba1022/java-concurrency/commit/2299793e5a86d3a0065c700f89361760e05ccbf1)

我覺得這邊的範例其實有些是有問題的, 因為我讀的書是用JDK7, 但我現在用的是JDK8, 在JDK8裡, 引入了meta space的概念, 所以很多OOM就制造不出來了, 所以看看就好...



