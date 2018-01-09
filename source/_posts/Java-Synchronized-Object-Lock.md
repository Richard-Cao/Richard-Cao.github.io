title: 使用Synchronized对象锁的一个优化细节
date: 2018-01-09 14:09:30
categories: 踩坑总结
tags: Java
---

## 背景

大家在使用synchronized关键字的时候，可能经常会这么写：

```java
synchronized (this) {
  ...
}
```

它的作用域是当前对象，锁的就是当前对象，谁拿到这个锁谁就可以运行它所控制的代码。当有一个明确的对象作为锁时，就可以这么写，但是当没有一个明确的对象作为锁，只想让一段代码同步时，可以创建一个特殊的变量（对象）来充当锁：

```java
public class Demo {
    private final Object lock = new Object();

    public void methonA() {
      synchronized (lock) {
        ...
      }
    }
}
```

这样写没问题。但是用new Object()作为锁对象是否是一个最佳选择呢？于是我好奇的搜索了一下，发现了这么一篇文章：[object-vs-byte0-as-lock](https://stackoverflow.com/questions/2120437/object-vs-byte0-as-lock)，大意就是用new byte[0]作为锁对象更好，会减少字节码操作的次数。由于这篇文章已经比较老了，为了确定到底如何，我还是决定手动验证一下。

## 准备工作

首先，我写了两个非常简单的java类作为Demo使用：

```java
// Demo.java
public class Demo {
    private final Object lock = new Object();
}
```

```java
// Demo_2.java
public class Demo_2 {
    private final byte[] lock = new byte[0];
}
```
我电脑上java的版本是：

```
java version "1.8.0_91"
Java(TM) SE Runtime Environment (build 1.8.0_91-b14)
Java HotSpot(TM) 64-Bit Server VM (build 25.91-b14, mixed mode)
```

## JVM

首先，要验证一下编译成JVM字节码是不是真的如上文所述。于是分别编译出两个java文件的class文件：

```
javac Demo.java
javac Demo_2.java
```

然后使用javap命令查看Demo的字节码：

```
▶ javap -c Demo
Compiled from "Demo.java"
public class Demo {
  public Demo();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: aload_0
       5: new           #2                  // class java/lang/Object
       8: dup
       9: invokespecial #1                  // Method java/lang/Object."<init>":()V
      12: putfield      #3                  // Field lock:Ljava/lang/Object;
      15: return
}
```

再查看Demo_2的字节码：

```
▶ javap -c Demo_2
Compiled from "Demo_2.java"
public class Demo_2 {
  public Demo_2();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: aload_0
       5: iconst_0
       6: newarray       byte
       8: putfield      #2                  // Field lock:[B
      11: return
}
```

可以看出，new byte[0]确实比new Object()少4条字节码操作。再计算一下内存占用，参考[JAVA 对象大小](https://www.liaohuqiu.net/cn/posts/caculate-object-size-in-java/)一文可以计算出：在64位jvm默认开启UseCompressedOops的情况下（Java 1.6.0_23版本开始就默认开启了），一个空对象，不包含任何成员变量，大小16字节，一个byte[0]数组，大小也是16字节，是相等的。

于是验证了上面的结论：用new byte[0]作为锁对象是优于new Object()的。在日常的java开发中，可以注意到这个细节的点来优化代码。

But，我是个Android开发，Android的虚拟机不是jvm，而是dvm。dvm相比jvm做了很多优化，那么在dvm上，结论还是一样的吗？带着这个问题，我又进一步做了验证。

## DVM

要想得到dvm的字节码，就需要使用Android SDK自带的dx工具将class文件转换为dex格式并dump出dex文件的内容。我使用了build-tools/26.0.2目录中的dx工具。

```
▶ ~/developer/android-sdk-macosx/build-tools/26.0.2/dx --dex --verbose --dump-to=Demo.dex.txt --verbose-dump Demo.class
processing Demo.class...
```

同理，Demo_2也是这条命令，只是将Demo换成Demo2而已。这样就得到了两个class文件对应的dvm字节码。先来看看Demo的：

```
...
0000ee: 2200 0100     |  0003: new-instance v0, java.lang.Object // type@0001
0000f2: 7010 0100 0000|  0005: invoke-direct {v0}, java.lang.Object.<init>:()V 
                      |        // method@0001
0000f8: 5b10 0000     |  0008: iput-object v0, v1, Demo.lock:Ljava/lang/Object;
                      |         // field@0000
0000fc: 0e00          |  000a: return-void
...
```

再来看看Demo_2的：

```
...
0000f6: 1200          |  0003: const/4 v0, #int 0 // #0
0000f8: 2300 0300     |  0004: new-array v0, v0, byte[] // type@0003
0000fc: 5b10 0000     |  0006: iput-object v0, v1, Demo_2.lock:[B // field@0000
000100: 0e00          |  0008: return-void
...
```

对比一下发现，指令都是4条。Dalvik字节码是以16位为单元（双字节码），java字节码以1字节为单元（单字节码）。可以看出，Demo这部分字节码一共8个单元即16字节，而Demo_2这部分字节码一共6个单元即12字节，new Object()比new byte[0]多了2个单元，意味着只是多分配了2个虚拟寄存器而已。内存占用方面同上。那么可以得出结论了：在Android编程中，使用new Object()或者new byte[0]作为对象锁差别不大。

## 总结

虽然最后发现经过dvm的优化，用new Object()还是new byte[0]作为锁对象差别不大，但是总归追究了这个问题得出了结论，并且也验证了在jvm下用new byte[0]作为锁对象是更好的选择，也了解了java对象的内存占用，dvm和jvm的区别以及dx工具的使用，以后再看到dx工具dump出的dex文件内容就不会感到陌生了。