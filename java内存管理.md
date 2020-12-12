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



### java栈的学习

参考资料: https://www.artima.com/insidejvm/ed2/jvm8.html

java栈里面由栈帧组成，每一个方法的创建，都会创建一个栈帧。jvm执行引擎执行的用到的就是栈顶的那个栈帧。这个栈帧也叫当前栈帧，对应的方法叫当前方法，对应的类叫当前类。方法的创建和退出，对应着栈帧入栈和出栈的动作。

栈帧的结构：栈帧是方法运行需要用到的数据结构，用于存储数据和临时结果。主要由：局部变量表，操作数栈，方法返回地址，动态链接组成



局部变量表

用于存储方法的参数和局部变量，它的基本单位是slot，每一个slot能存32位，能存一个int，short，boolean，byte，char，reference,float类型，对于64位的long和double则需要两个连续的slot来存储。它的最大大小，存在方法的Code属性中的：max_locals的属性中。它的访问是通过索引来访问的。

比如以下代码：

```java
package cn.yishijie.jol;

public class JvmStack0 {

    public static void main(String[] args) {
        int a = 1; 
        boolean b = true;
        short c = 1;
        Object obj = new Object();
        float d = 2f;
        char e = 12;
        byte f = 1;
    }
}
```

![max-locals](D:\jeffchan\workspace\learn-note\img\max-locals.png)

上面7个局部变量，都是占一个slot的，因为默认有个this的reference类型的局部变量，所以这里的最大槽数是8.

验证long和double都需要两个槽：

```java
package cn.yishijie.jol;

public class JvmStack1 {

    public static void main(String[] args) {
        double a = 12D;
        long b = 12L;
    }
}
```

下面的图片中，最大槽数是5，因为long两个，double两个，剩下一个是reference类型的this

![twoslot](D:\jeffchan\workspace\learn-note\img\twoslot.png)

验证下槽的复用:

```java
package cn.yishijie.jol;
public class JvmStack2 {

    public static void main(String[] args) {
        {
            double a = 12D;
        }
        long b = 12L;
    }
}
```

这里的b变量会复用a变量的槽，所以这里理论上来说一共是3个槽的。

![reuseslot](D:\jeffchan\workspace\learn-note\img\reuseslot.png)

图上也是只有3个槽。



关于槽没复用，带来内存泄漏的问题。

```java
package cn.yishijie.jol;

public class JvmStack2 {

    public static void main(String[] args) {
        {
            byte[] b = new byte[1024*1024*10];
        }
        //int a = 1;
        System.gc();
    }
}
```

注释掉：

![leak](D:\jeffchan\workspace\learn-note\img\leak.png)

可以发现，这10M的内存，是没有被回收掉的，按道理说，gc触发的地方，已经超过b的作用域了，但是却没有回收掉已经没有用的对象。其实这是因为这个方法的栈帧中的局部变量表还是存着上述的对象引用，所以无法将上述的对象回收，如果这个时候，有新的变量需要使用到slot时，就会复用上述的对象占用的槽，那么上述对象的gc root就无法访问到，所以就可以回收上述的对象。

将注释的那个代码打开，可以发现打印出的日志：

![unleak](D:\jeffchan\workspace\learn-note\img\unleak.png)

发现对应的对象b，已经被回收掉了。



操作数栈

操作数栈也称作操作栈，一种先进后出的数据结构。

和局部变量表一样，操作数栈也被组织成字节数组，但是操作数栈不是通过索引来访问的，而是通过pushing和poping 值来访问。如果一个指令将一个值压入操作数栈中，过后就有有一个指令会弹出栈顶值，并且使用这个值。java虚拟机中操作数栈的元素类型和局部变量表的元素是一样的，都是int,long,float,double,reference 和returnType;在将byte,short,char压入栈之前，会将他们转成int类型。

java虚拟机是基于栈的而不是基于寄存器，因为java虚拟机获取操作数，是从栈中获取的，并不是在寄存器上获取的。它使用操作数栈作为工作空间。许多指令弹出栈顶的值，进行操作，然后压栈。比如iadd指令就是将操作数栈顶的两个元素弹出相加，然后压入栈。

比如以下指令：

iload_0: 将局部变量表中第0个索引的局部变量压入栈，

iload_1:将局部变量表中第1个索引的局部变量压入栈。

iadd：一次弹出栈顶两个元素，然后相机，再压入栈中。

istore_2: 弹出栈顶元素，然后存入到局部表第二个索引的局部变量中。

![stack](D:\jeffchan\workspace\learn-note\img\stack.png)

如下代码：

```java
package cn.yishijie.jol;

public class JvmStack3 {
    public static void main(String[] args) {
        int a = 10;
        int b = 100;
        int c = a + b;
    }
}
```

![bytecodes](D:\jeffchan\workspace\learn-note\img\bytecodes.png)



点入到杂项 验证下：

![bytecode1](D:\jeffchan\workspace\learn-note\img\bytecode1.png)

局部变量槽是4个（max_locals），字节码长度是11，栈的最大深度（max_stacks）是2.

查看字节码指令的含义：

https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.bipush

```java
 0 bipush 10  // 将10压入栈，这里占用两个字节，一个是指令占用一个字节，操作数占用一个字节
 2 istore_1   // 将栈顶元素弹出，存入到索引位置为1的局部变量中
 3 bipush 100 // 将100压入栈顶
 5 istore_2  // 将栈顶元素弹出，存入到索引位置为2的局部变量中
 6 iload_1   // 将局部变量1的值压入栈中
 7 iload_2   // 将局部变量2 的值压入栈中
 8 iadd      // 将栈顶两个元素弹出，相加，然后压入栈中
 9 istore_3  // 将栈顶元素弹出压入到局部变量3中。
10 return // 返回
     
// 整个压栈的过程，最大的深度，将两个局部变量得值压入到栈中。为2
```



动态链接：

当需要调用其他方法的时候，需要将这些方法的符号引用转成直接引用。当每次运行时，

都需要转成直接引用的话，这种转换叫动态链接。如果仅仅时在类加载，或者第一次使用

时才需要转换的话，叫静态解析。

直接引用，是能在内存定位到的地址。可以直接定位到要执行的方法。

符号引用，就是常量池中的一些类型。比如Methodref类型的数据。

```java
package cn.yishijie.jol;

public class JvmDynamicLink {

    public static void main(String[] args) {
        JvmDynamicLink jvmDynamicLink = new JvmDynamicLink();
        jvmDynamicLink.link();
    }

    public void link(){
    }
}
```

通过javap -verbose JvmDynamicLink.class查看:

```java
public class cn.yishijie.jol.JvmDynamicLink
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #5.#21         // java/lang/Object."<init>":()V
   #2 = Class              #22            // cn/yishijie/jol/JvmDynamicLink
   #3 = Methodref          #2.#21         // cn/yishijie/jol/JvmDynamicLink."<init>":()V
   #4 = Methodref          #2.#23         // cn/yishijie/jol/JvmDynamicLink.link:()V
   #5 = Class              #24            // java/lang/Object
   #6 = Utf8               <init>
   #7 = Utf8               ()V
   #8 = Utf8               Code
   #9 = Utf8               LineNumberTable
  #10 = Utf8               LocalVariableTable
  #11 = Utf8               this
  #12 = Utf8               Lcn/yishijie/jol/JvmDynamicLink;
  #13 = Utf8               main
  #14 = Utf8               ([Ljava/lang/String;)V
  #15 = Utf8               args
  #16 = Utf8               [Ljava/lang/String;
  #17 = Utf8               jvmDynamicLink
  #18 = Utf8               link
  #19 = Utf8               SourceFile
  #20 = Utf8               JvmDynamicLink.java
  #21 = NameAndType        #6:#7          // "<init>":()V
  #22 = Utf8               cn/yishijie/jol/JvmDynamicLink
  #23 = NameAndType        #18:#7         // link:()V
  #24 = Utf8               java/lang/Object
{
  public cn.yishijie.jol.JvmDynamicLink();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcn/yishijie/jol/JvmDynamicLink;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=1
         0: new           #2                  // class cn/yishijie/jol/JvmDynamicLink
         3: dup
         4: invokespecial #3                  // Method "<init>":()V
         7: astore_1
         8: aload_1
         9: invokevirtual #4                  // Method link:()V
        12: return
      LineNumberTable:
        line 6: 0
        line 7: 8
        line 8: 12
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      13     0  args   [Ljava/lang/String;
            8       5     1 jvmDynamicLink   Lcn/yishijie/jol/JvmDynamicLink;

  public void link();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=0, locals=1, args_size=1
         0: return
      LineNumberTable:
        line 12: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       1     0  this   Lcn/yishijie/jol/JvmDynamicLink;
}
SourceFile: "JvmDynamicLink.java"
```

直接去到main方法的Code中，发现调用方法的时候：执行指令：

9: invokevirtual #4    这个4代表的就是常量池里第四个常量的位置：

 #4 ->   Methodref 类型： #2.#23     #23 ->  #18:#7     #18  ->   link   #7 ->  ()V

最终指向： cn/yishijie/jol/JvmDynamicLink.link:()V  这个就是符号引用。



方法返回

方法返回有两种，一种是正常返回，恢复调用方法的栈帧信息和pc值就可以回到方法的调用处。

```java
package cn.yishijie.jol;

public class JvmReturn {

    public static void main(String[] args) {
        noReturn();
        int i = returnInt();
        long l = returnLong();
        throwEx();
    }

    public static void noReturn(){
    }

    public static int throwEx(){
        throw new RuntimeException();
    }
    public static int returnInt(){
        return 10;
    }

    public static long returnLong(){
        return 10L;
    }
}
```

javap -verbose JvmReturn.class

太长了，我这里只截取部分:

```java
 public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=4, args_size=1
         0: invokestatic  #2                  // Method noReturn:()V
         3: invokestatic  #3                  // Method returnInt:()I
         6: istore_1
         7: invokestatic  #4                  // Method returnLong:()J
        10: lstore_2
        11: return
      LineNumberTable:
        line 6: 0
        line 7: 3
        line 8: 7
        line 9: 11
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      12     0  args   [Ljava/lang/String;
            7       5     1     i   I
           11       1     2     l   J

  public static void noReturn();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=0, locals=0, args_size=0
         0: return
      LineNumberTable:
        line 12: 0

  public static int returnInt();
    descriptor: ()I
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=0, args_size=0
         0: bipush        10
         2: ireturn
      LineNumberTable:
        line 15: 0

  public static long returnLong();
    descriptor: ()J
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=0, args_size=0
         0: ldc2_w        #5                  // long 10l
         3: lreturn
      LineNumberTable:
        line 19: 0
                                     
    public static int throwEx();
    descriptor: ()I
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=0, args_size=0
         0: new           #6                  // class java/lang/RuntimeException
         3: dup
         4: invokespecial #7                  // Method java/lang/RuntimeException."<init>":()
V
         7: athrow
      LineNumberTable:
        line 16: 0
}
```



可以看到正常返回的指令，return返回void  ireturn返回int型 lreturn返回long型。如果是正常返回的话，方法退出后，调用者的栈帧即为当前栈帧，pc下一条指令，就是返回的地址。这里如果是返回void,栈帧就不保存返回结果。如果有返回值的，返回后，会将值压入栈中。

         0: invokestatic  #2                  // Method noReturn:()V
         3: invokestatic  #3                  // Method returnInt:()I
         6: istore_1
         7: invokestatic  #4                  // Method returnLong:()J
        10: lstore_2
        11: return
第一个空返回的直接就返回了。有返回值的，比如invokestatic  #3，它返回的是int类型的值，

那么会紧接着istore_1将栈顶元素（int类型）弹出，存入到第一个局部变量中。



直接抛出异常： athrow，这个是显示抛出异常，将异常交给调用者处理。



其他信息：

还有一些信息，比如：<LineNumberTable> 用于调试用的，记录源码的行号信息和  <LocalVariableTable>记录源码方法的参数信息。



异常表的结构：

```java
package cn.yishijie.jol;

public class TestFinal {
    public static void main(String[] args) {
        int a = a(10);
        System.out.println(a);
    }

    public static  int a(int a){
        try {
            return a;
        }catch (Exception e){
            return -1;
        }finally {
           a = a+100;
        }
    }
}
```

直接copy Code属性的部分：

```java
 Code:
      stack=2, locals=4, args_size=1
         0: iload_0  // 将第0个局部变量的值10入栈
         1: istore_1 // 将栈顶值10存入到第一个局部变量中 
             
+++++++++++++finally开始0++++++++++++++++++++
         2: iload_0 // 将第0个局部变量10的值入栈
         3: bipush        100 // 将100的值入栈
         5: iadd // 弹出栈顶两个元素，相加入栈 110
         6: istore_0 // 将栈顶的值110存入到第0个变量中
===============finally结束0=================
             
         7: iload_1 // 将第一个变量10的值入栈
         8: ireturn // 返回栈顶的元素10。
         9: astore_1 
        10: iconst_m1
        11: istore_2
        
+++++++++++++finally开始1++++++++++++++++++++
        12: iload_0
        13: bipush        100
        15: iadd
        16: istore_0
===============finally结束1===============
            
        17: iload_2
        18: ireturn
        19: astore_3
 +++++++++++++finally开始2++++++++++++++++++++
        20: iload_0
        21: bipush        100
        23: iadd
        24: istore_0
 ===============finally结束2===============
        25: aload_3
        26: athrow
      Exception table:
         from    to  target type
             0     2     9   Class java/lang/Exception
             0     2    19   any
             9    12    19   any
```



引用分类:

引用可以分成四类，分别为：强引用，软引用，弱引用和虚引用。

对于强引用，是我们最常见，比如直接创建一个对象：Obeject obj = new Object();那么obj就是一个强引用。在当前栈帧有效的作用域内，是永远不会被回收的。



软引用是指被SoftReference类实现的引用。它的特征是当系统有足够的内存的时候，它能够存活，当系统内存不足，垃圾回收的动作到来时，它会被回收释放内存。

```java
package cn.yishijie.jol;

import java.lang.ref.SoftReference;
import java.lang.ref.WeakReference;

public class SoftTest {

    public static void main(String[] args) {
        SoftReference<String> softStr = new SoftReference<>(new String("jeff.chan"));

        System.out.println("before gc -> " + softStr.get());

        System.gc();

        System.out.println("after gc -> " + softStr.get());
    }
}
```

执行的结果：加上虚拟机参数:  -XX:+PrintGCDetails 

```java
before gc -> jeff.chan
[GC (System.gc()) [PSYoungGen: 5350K->1047K(38400K)] 5350K->1055K(125952K), 0.0007776 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [PSYoungGen: 1047K->0K(38400K)] [ParOldGen: 8K->937K(87552K)] 1055K->937K(125952K), [Metaspace: 2988K->2988K(1056768K)], 0.0035376 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
after gc -> jeff.chan
```

发现没有被回收。这是因为内存足够：



```java
package cn.yishijie.jol;

import java.lang.ref.SoftReference;
import java.util.ArrayList;
import java.util.List;

public class SoftTest {

    public static void main(String[] args) {
        SoftReference<byte[]> softStr = new SoftReference<>(new byte[1024*1024]);

        System.out.println("before gc -> " + softStr.get());
        List<byte[]> list = new ArrayList<>();
        int i = 0;
        while (true){
            list.add(new byte[1024*1024]);
            System.out.println(++i+"allow memory -> " + softStr.get());
        }
    }
}
```

加入jvm的参数: -XX:+PrintGCDetails -Xmx10m -Xms10m 执行上述的代码

返回结果：

```java
before gc -> [B@3b9a45b3
[1]allow memory -> [B@3b9a45b3
[2]allow memory -> [B@3b9a45b3
[3]allow memory -> [B@3b9a45b3
[4]allow memory -> [B@3b9a45b3
[5]allow memory -> [B@3b9a45b3
[GC (Allocation Failure) [PSYoungGen: 1667K->487K(2560K)] 8203K->7169K(9728K), 0.0006804 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[6]allow memory -> [B@3b9a45b3
[GC (Allocation Failure) --[PSYoungGen: 1551K->1551K(2560K)] 8233K->8249K(9728K), 0.0004440 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1551K->1497K(2560K)] [ParOldGen: 6698K->6650K(7168K)] 8249K->8147K(9728K), [Metaspace: 3249K->3249K(1056768K)], 0.0037707 secs] [Times: user=0.08 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) --[PSYoungGen: 1497K->1497K(2560K)] 8147K->8147K(9728K), 0.0005219 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Allocation Failure) [PSYoungGen: 1497K->0K(2560K)] [ParOldGen: 6650K->7094K(7168K)] 8147K->7094K(9728K), [Metaspace: 3249K->3249K(1056768K)], 0.0039966 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[7]allow memory -> null
[Full GC (Ergonomics) [PSYoungGen: 1064K->1024K(2560K)] [ParOldGen: 7094K->7062K(7168K)] 8159K->8086K(9728K), [Metaspace: 3250K->3250K(1056768K)], 0.0054518 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (Allocation Failure) [PSYoungGen: 1024K->1024K(2560K)] [ParOldGen: 7062K->7062K(7168K)] 8086K->8086K(9728K), [Metaspace: 3250K->3250K(1056768K)], 0.0025587 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at cn.yishijie.jol.SoftTest.main(SoftTest.java:16)
```

可以发现第七次的时候； [7]allow memory -> null，已经把上面的软引用的对象给回收掉了。紧接着后续就抛出OOM异常了，因为内存不够了。所以软引用是在内存告急时，才会被回收释放。

使用场景：在Hibernate中就有用到这个软引用，就是当作缓存，当内存足够的时候，数据存在缓存中，可以直接提交查询的性能，当内存不够时，又可以释放内存，防止OOM的发生。举个例子：比如以前我做过一个城市列表的需求，第一次的时候，通过查询数据库放入到软引用的内存中，然后后续需要用到这个城市列表的时候，直接从缓存中拿，而不用穿透到数据库中去，提供了性能。但是当内存告急的时候，那么就会释放这部分内存。缓解内存的压力。内存中没有这些城市的数据，不会引起服务异常，只不过是，下一次需要数据的时候，就需要穿透数据库拿数据而已。



弱引用是指被WeakReference实现的引用。它只能存活到下一次垃圾回收发生之前。如果进行垃圾回收，那么一定会被回收。

```java
package cn.yishijie.jol;

import java.lang.ref.WeakReference;
public class WeakTest {

    public static void main(String[] args) {
        WeakReference<String> weakStr = new WeakReference<>(new String("jeff.chan"));

        System.out.println("before gc -> "+weakStr.get());

        System.gc();

        System.out.println("after gc -> " + weakStr.get());
    }
}
```

返回结果：加上虚拟机参数:  -XX:+PrintGCDetails 

```java
before gc -> jeff.chan
[GC (System.gc()) [PSYoungGen: 5350K->1015K(38400K)] 5350K->1023K(125952K), 0.0165296 secs] [Times: user=0.00 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [PSYoungGen: 1015K->0K(38400K)] [ParOldGen: 8K->944K(87552K)] 1023K->944K(125952K), [Metaspace: 3102K->3102K(1056768K)], 0.0037297 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
after gc -> null
```

可以发现，确实在gc之后，就拿不到对应的那个对象了。

使用场景：

1、ThreadLocal里面就是使用了弱引用减少内存泄漏的发生。但是我觉得这个使用场景，并不怎么好。我提出了以下的疑问？

```java
因为ThreadLocal我看到很多人使用把它当作一个全局的静态变量，一般来说类被加载进来就会有静态变量。而静态变量要被回收的话，条件是很苛刻，基本是不被回收的，那么就是说，下一次垃圾回收到来之前，根本就无法回收掉被当作弱引用创建出来的ThreadLocal，因为它有一个强引用引用着。按照这种场景的话，弱引用感觉什么用都没有。

就算有使用场景，ThreadLocal的强引用会被回收，那么对于存活的线程来说，只不过是ThreaLocalMap里面key引用着ThreadLocal这个实例，况且这个对象并不怎么占内存。而且也是推荐使用完ThreadLocal后，要手动去remove掉。那么感觉弱引用也没啥用。

ThreadLocal就算弱引用被回收了。那么这个ThreadLocal不使用的话，对应的value，也是被存活的线程引用的，也并不会回收掉。也会造成内存泄漏，最终也是要手动去remove掉，或者是再次调用set或者get的方法。

难道就为了弱引用被垃圾回收后（看使用场景，经常是通过全局的静态变量，回收的概率又不太高），当再使用set或者get的方法能够主动清除value这个功能，就引入了这个设计？
```

2、当作缓存使用，在下一次gc之前可以拿到，gc之后就从数据库中拿数据。但是感觉SoftReference用作缓存更适合些。



虚引用时指被PhantomReference类实现的引用，无法通过虚引用来获取到一个对象实例。它被用来跟踪对象引用被加入到队列的时刻。所以它的使用是需要和队列一起使用的。其实上述的软引用和弱引用也是可以搭配队列使用的。但是虚引用必须搭配队列使用。

```java
package cn.yishijie.jol;

import cn.yishijie.config.Person;

import java.lang.ref.PhantomReference;
import java.lang.ref.Reference;
import java.lang.ref.ReferenceQueue;

public class PhantomReferenceTest {
    public static void main(String[] args) throws Exception{
        ReferenceQueue<Person> queue = new ReferenceQueue<>();
        PhantomReference<Person> reference = new PhantomReference<>(new Person(23,"jeff.chan"),queue);
        System.out.println("before gc->"+reference.get());
        System.out.println("before gc->"+queue.poll());
        System.gc();
        System.out.println("after gc->"+reference.get());
        System.out.println("after gc->"+queue.poll());
    }
}
```



返回结果：

```java
before gc->null
before gc->null
[GC (System.gc()) [PSYoungGen: 5350K->1031K(38400K)] 5350K->1039K(125952K), 0.0013433 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [PSYoungGen: 1031K->0K(38400K)] [ParOldGen: 8K->979K(87552K)] 1039K->979K(125952K), [Metaspace: 3249K->3249K(1056768K)], 0.0036022 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
after gc->null
after gc->java.lang.ref.PhantomReference@3b9a45b3
```



## 3、垃圾回收算法

基于不同的区域会有不同的垃圾回收算法，因为不同区域对象的存活概率不同，对象存活多的，复制成本高，效率低；直接清除掉废弃的，效率就会比较高，成本低。相反，如果对象存活低的，复制成本就比较低，效率高；此时如果采用标记清除，成本就会比较高，效率低下了。

### 3.1、标记清除

标记清除，顾名思义，就是先标记哪些对象需要被清除，然后对打上清除标记的对象进行释放内存即可。对于hotspot虚拟机，对象头中，有两位锁标记，当值为11时，就说明它是需要被清除的对象。这种算法，一般用在对象存活比较多的场景。老年代里的对象就满足这种特征，因为gc执行后，基本都会存活，回收的对象并不多。比如老年代有100个对象。gc执行后，发现只有2个对象是GC Roots不能到达的，那么就会标记这两个对象，同时释放掉这两个对象的内存。这种算法有个缺点，就是会产生空间碎片，比如我老年代有10M的内存剩余，但是却分布在5个区域，都是2M分布的，那么此时我需要分配一个3M内存的对象，发现无法分配。



### 3.2、复制算法

复制算法，就是复制存活的对象，然后将存活的对象存起来，对扫描的区域进行一整块清除。这种算法，需要一块空闲的空间来存储存活的对象。年轻代里面的对象就满足这种特征，同时还将空间分成3块，一块eden区，两个Survivor区，比例默认分别为8 : 1 : 1，所以这里使用10%空间的代价来存储存活的对象，然后对另外两个内存按照整块区域清除。比如一般大部分对象都是用过就不再使用的，存活率并不高。比如我请求一个网站，对应到服务器接收请求，假设整个请求的过程，会创建100个对象，一般来说对象分配优先是分配在年轻代的（有其他分配场景可能不是分配在年轻代），当请求完成后，就只有一个对象是存活的，那么此时如果垃圾回收的动作到来，只需要这个对象复制到用来存储存活对象的Survivor区，然后将伊甸园和另外一块survivor区整块清除掉，效率就比较好。



### 3.3、标记整理算法

这种算法是将存活的对象往一端移动，然后清除掉存活对象边界外的内存空间。这个算法也不会产生空间碎片。不过这种方法，如果每次回收都有大量回收区域，那么修改对象引用就会变得比较耗时间，那么服务停顿的时间就更长了。



空间分配担保：

这个分配担保是在新生代空间，即使是进行了minor gc之后还是不够的状态111





