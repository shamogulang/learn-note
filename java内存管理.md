java内存管理

我这里使用的版本为：java version "1.8.0_31"

1、运行时数据区域

方法区(method area)：常量池和类的类型信息等

虚拟机栈：本地方法栈：本地变量和方法

堆：对象分配的地方

程数计数器：行号指示器



首先，这里会先用一个工具查看内存的信息：jconsole

相关的介绍：https://docs.oracle.com/javase/1.5.0/docs/guide/management/jconsole.html



装了jdk，并且配置了环境变量，可以直接在控制台中输入jconsole,就会弹出对应的界面。

这里我首先写一段代码，仅仅是sleep一段时间，这里就可以使用jconsole来查看这一段运行着

的代码。

```java
package cn.yishijie.jvm;
import java.util.concurrent.TimeUnit;
public class JDK8Memory {

    public static void main(String[] args) throws InterruptedException{

        // 睡一个小时
        TimeUnit.HOURS.sleep(1L);
    }
}
```

运行起来，然后直接在控制台输入jconsole：弹出以下界面





## 1、方法区

方法区是线程共享的内存区域，用于存储jvm加载的

类型信息，用本地内存元空间（meta space）实现。

垃圾回收会被该内存进行处理，主要是对类型的卸载和

常量池的回收。

方法区在JVM启动的时候被创建，它的大小决定了能保存

的类的数量，如果超出，会出现oom的异常。



## 2、堆内存

对象的探索

堆中的对象创建：需要划分内存区域。

有两种方式：

一种：指针碰撞

另外一种： 空闲列表

具体采用哪种方式，跟内存是否是规整有关。

内存是否规整，跟垃圾回收的策略有关。

如果采用的是复制算法，那么内存规整，采用指针碰撞的方式

如果是标记清除的方法，那么内存就不规整，采用空闲列表的方式



为了防止并发出现问题，一般会使用相关的同步策略。

一个使用cas方法

一个使用本地线程缓存的方式，就是为每一个线程分一块

TLAB,等需要分配新的缓存的时候，才需要同步。



分配完之后，会堆分配的内存进行初始化零值。

完成之后会进行对象头的设置。

对象在内存布局中分成三个部分：

对象头，实例数据，对齐填充（要求对象的大小满足8字节的整数倍）



对象头的介绍：

对象头由两部分组成：mark word 和 Klass pointer(如果是数组，还有加上数组的长度)

mark word这里会占用8个字节，class workd占用4个字节。

mark word的用处：

1、存gc分代年龄

2、存hash code

3、存锁的信息

这里由参考资料：https://stackoverflow.com/questions/26357186/what-is-in-java-object-header

因为我这里是64位的机器：只是截取其中一部分

```
//  64 bits:
//  --------
//  unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
//  JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
```

具体各个位的状态如下图:

```java
//    [JavaThread* | epoch | age | 1 | 01]       lock is biased toward given thread
//    [0           | epoch | age | 1 | 01]       lock is anonymously biased
//
//  - the two lock bits are used to describe three states: locked/unlocked and monitor.
//
//    [ptr             | 00]  locked             ptr points to real header on stack
//    [header      | 0 | 01]  unlocked           regular object header
//    [ptr             | 10]  monitor            inflated lock (header is wapped out)
//    [ptr             | 11]  marked             used by markSweep to mark an object
//                                               not valid at any other time
```



查看对象的大小，可以使用jol-core来查看：

相应的jar包，或者是直接将maven的坐标放入到pom.xml中。

```java
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.14</version>
    <scope>provided</scope>
</dependency>
```



编写下面的代码：

```java
package cn.yishijie.jol;
import org.openjdk.jol.info.ClassLayout;
public class JolTest {
    public static void main(String[] args) {
        Object obj = new Object();
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
    }
}
```

执行，打印出以下结果： -XX:-UseCompressedOops

java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 20 (11100101 00000001 00000000 00100000) (536871397)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total



可以发现一个单纯的Object对象，大小是16字节。对象头占12个字节，实例数据为0字节，还有4个字节是填充字节。Space losses 表示损失的字节，就是浪费掉的字节数。分为内部和外部，这个还不知道是什么？

因为jvm存储是按照大端的存储方式，那么高字节在前，将上述写出我们习惯的顺序。

那么mark word的实际结构

```java
unused:25 | hash:31 | unused:1 |  age:4  |  biased_lock:1 | lock:2 (normal object)
00000000 00000000 00000000 0 | 0000000 00000000 00000000 00000000 | 0 | 0000 | 0 | 01
    
```

最后的三位：0（是否偏向）01（无锁）表示无锁且不偏向。

这里没有打印出hashcode是因为，hashcode的写入，调用了系统的hashcode的方法后，写入的，自己重写的

hashcode方法对这个值不起作用。

这里调用下hashcode的方法。改写上述代码：

```java
package cn.yishijie.jol;

import org.openjdk.jol.info.ClassLayout;

public class JolTest {

    public static void main(String[] args) {
        Object obj = new Object();
        System.out.println(obj.hashCode());
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
    }
}

```

这里仅仅是调用了一下hashcode方法

```ajva
999966131
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 b3 45 9a (00000001 10110011 01000101 10011010) (-1706708223)
      4     4        (object header)                           3b 00 00 00 (00111011 00000000 00000000 00000000) (59)
      8     4        (object header)                           e5 01 00 20 (11100101 00000001 00000000 00100000) (536871397)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

对mark word部分进行整理：

```java
unused:25 | hash:31 | unused:1 |  age:4  |  biased_lock:1 | lock:2 (normal object)

00000000 00000000 00000000 0 | 0111011 10011010 01000101 10110011 | 0| 0000| 0 | 01
```

999966131 =  11101110011010 0100010110110011   跟hash那部分是一样的。



修改下原来的代码如下：

```java
package cn.yishijie.jol;

import org.openjdk.jol.info.ClassLayout;
public class JolTest {
    public static void main(String[] args) {
        Object obj = new Object();
        System.out.println(obj.hashCode());
        synchronized (obj){
            System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        }
    }
}
```

输出结果如下：

```java
999966131
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           28 f3 38 02 (00101000 11110011 00111000 00000010) (37286696)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 20 (11100101 00000001 00000000 00100000) (536871397)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

对mark word部分进行整理：指向栈中的锁记录的指针+00标记，轻量级锁

```java
00000000 00000000  00000000 00000000 00000010 00111000 11110011 001010 | 00
```

发现此时hashcode并没有写进mark word



注意上述的hashcode都是 identity hash code，用户重写的hashcode是不会写入到该对象头里去的。还有计算了

 identity hash code的对象，是无法进入偏向状态的，就算对象已经进入了偏向状态，如果你 identity hash code计算了，那么会撤销偏向锁，膨胀为重量级锁。



不知道为什么，我去掉了 identity hash code，还是无法进入偏向锁的状态。

```java
package cn.yishijie.jol;

import org.openjdk.jol.info.ClassLayout;

public class JolTest {
    public static void main(String[] args) {
        final Object obj = new Object();
        ClassLayout classLayout = ClassLayout.parseInstance(obj);
        synchronized (obj){
            System.out.println(classLayout.toPrintable());
        }
    }
}
```

```java
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           40 f3 e8 02 (01000000 11110011 11101000 00000010) (48821056)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 20 (11100101 00000001 00000000 00100000) (536871397)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

发现结果还是处于轻量级锁状态。不知道为什么？



代码样式：

```java
package cn.yishijie.jol;

import org.openjdk.jol.info.ClassLayout;

public class JolTest {
    public static void main(String[] args) throws Exception{
        Object obj = new Object();
        ClassLayout classLayout = ClassLayout.parseInstance(obj);
        new Thread(() -> {
            synchronized (obj){
                try {
                    Thread.sleep(1000L);
                }catch (Exception e){

                }
            }
        }).start();
        synchronized (obj){
            System.out.println(classLayout.toPrintable());
        }
    }
}

```

打印结果：

```java
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           1a bd 01 03 (00011010 10111101 00000001 00000011) (50445594)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 20 (11100101 00000001 00000000 00100000) (536871397)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

发现已经上升到重量级锁了。



第二天，继续探索。

参考资料：https://shipilev.net/jvm/objects-inside-out/

我发现第一天，一直没有进入偏向锁的状态，是因为偏向锁是由延迟加载的。延迟时间默认是4s

BiasedLockingStartupDelay  = 4000 

引入代码：

```java
package cn.yishijie.jol;

import org.openjdk.jol.info.ClassLayout;

import java.util.concurrent.TimeUnit;

public class MyTest {
    public static void main(String[] args) throws Exception{
        TimeUnit.SECONDS.sleep(5L); // 这里睡5s，是为了让偏向锁启动起来。
        Object o = new Object();
        ClassLayout classLayout = ClassLayout.parseInstance(o);
        System.out.println(classLayout.toPrintable());
    }
}
```

```java
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 20 (11100101 00000001 00000000 00100000) (536871397)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

发现此时的101就是处于偏向状态：此时是匿名偏向状态。

```java
//    [JavaThread* | epoch | age | 1 | 01]       lock is biased toward given thread
//    [0           | epoch | age | 1 | 01]       lock is anonymously biased
```

继续上代码：

```java
package cn.yishijie.jol;

import org.openjdk.jol.info.ClassLayout;

import java.util.concurrent.TimeUnit;

public class MyTest {
    public static void main(String[] args) throws Exception{
        TimeUnit.SECONDS.sleep(5L);
        Object o = new Object();
        synchronized (o){
            ClassLayout classLayout = ClassLayout.parseInstance(o);
            System.out.println(classLayout.toPrintable());
        }
    }
}
```

打印结果：

```java
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 58 6a 02 (00000101 01011000 01101010 00000010) (40523781)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 20 (11100101 00000001 00000000 00100000) (536871397)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

对于mark word 部分

```java
00000000 00000000 00000000 00000000  00000000 01101010 010110 | 00 | 0 | 0000| 1| 01
```

处于偏向锁状态。



上代码：

```java
package cn.yishijie.jol;

import org.openjdk.jol.info.ClassLayout;

import java.util.concurrent.TimeUnit;

public class JolTest {
    public static void main(String[] args) throws Exception{
        TimeUnit.SECONDS.sleep(5L);
        Object obj = new Object();
        ClassLayout classLayout = ClassLayout.parseInstance(obj);
        new Thread(() -> {
            synchronized (obj){
                System.out.println("jeff.chan");
                System.out.println(classLayout.toPrintable());
            }
        }).start();
        synchronized (obj){
            System.out.println("caraliu");
        }
    }
}
```

打印结果：

```java
caraliu
jeff.chan
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           18 f2 5a 1a (00011000 11110010 01011010 00011010) (442167832)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 20 (11100101 00000001 00000000 00100000) (536871397)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

发现现在处于轻量级锁，打印caraliu很快就执行完了，jeff.chan的那里等待并不会自旋太久，所以就升级到轻量级锁。

可以给caraliu的那里，睡眠一段时间，那么对于jeff.chan,就会一直自旋，达到一定次数，默认10次，就升级为重量级锁。

```java
package cn.yishijie.jol;

import org.openjdk.jol.info.ClassLayout;

import java.util.concurrent.TimeUnit;

public class JolTest {

    public static void main(String[] args) throws Exception{
        TimeUnit.SECONDS.sleep(5L);
        Object obj = new Object();
        ClassLayout classLayout = ClassLayout.parseInstance(obj);
        new Thread(() -> {
            synchronized (obj){
                System.out.println("jeff.chan");
                System.out.println(classLayout.toPrintable());
            }
        }).start();
        synchronized (obj){
            System.out.println("caraliu");
            Thread.sleep(3000L);
        }
    }
}

```

结果：

caraliu
jeff.chan
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           1a c3 30 03 (00011010 11000011 00110000 00000011) (53527322)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 20 (11100101 00000001 00000000 00100000) (536871397)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

可以看到锁的标记为10，处于重量级锁状态。

