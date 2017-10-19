## 前言
JDK Analysis系列的第一篇，就从万众瞩目的ReentrantLock开始吧，而谈到ReentrantLock，就不得不说AQS，它是AbstractQueuedSynchronizer类的简称，Doug Lea上神在JDK1.5将其引入，这才有了现在的并发包java.util.concurrent，所以要理解ReentrantLock的原理，AQS也是必须要搞懂的。

