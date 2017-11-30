# The Implementation of Atomic Operations

### 前言

原子\(atom\)的本意是"無法再被進一步分割的最小粒子", 至於原子操作\(atomic operation\)則是"**不可被中斷的一個或一系列操作**". 在多處理器上實作原子操作就變得有點複雜, 這篇要紀錄的心得是Intel處理器與Java實作原子操作的方式.

### 處理器如何實作原子操作

處理器自動保證基本記憶體操作的原子性

使用bus lock保證原子性

使用cache lock保證原子性

### Java如何實作原子操作

使用循環CAS實作原子操作

使用lock機制實作原子操作

