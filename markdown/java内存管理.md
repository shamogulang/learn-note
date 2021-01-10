java内存管理

我这里使用的版本为：java version "1.8.0_31"，64位的机器



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

![jdk8](D:\jeffchan\markdown\image\jdk8.png)

点击箭头的哪个进程，然后选择不安全连接然后就可以进去查看这个类运行的具体情况了。

直接进入到内存的那个菜单进入：



![jdk82](D:\jeffchan\markdown\image\jdk82.png)



点开A区域可以看到内存的划分情况；点B可以执行一次GC；C可以查看对应的内存的使用情况；

D可以图示的看到内存情况，鼠标移动到该区域停住可以弹出该区域的名称，点击可以选中对应的区域

并展示该区域的信息。

这里分为：堆和非堆。

```yml
堆: 
  PS Old Gen: 老年代
  PS Eden Space: 新生代eden区
  PS Survivor Space: 新生代幸存区
  
非堆:
  Meta Data: 元空间
  Code Cache: 代码缓存
  Compressed Class Space: 压缩类空间
```

启动的时候，增加VM参数：-XX:+PrintCommandLineFlags

```java
然后可以看到：

-XX:InitialHeapSize=133094272 -XX:MaxHeapSize=2129508352 -XX:+PrintCommandLineFlags -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:-UseLargePagesIndividualAllocation -XX:+UseParallelGC 
```

可以发现这里使用的垃圾回收器： ParallelGC ，它会和Parallel Old搭配使用。其实上面的PS就是表示

这个垃圾收集器组合。





## 1、方法区

方法区是线程共享的内存区域，用于存储jvm加载的

类型信息，用本地内存元空间（meta space）实现。

垃圾回收会被该内存进行处理，主要是对类型的卸载和

常量池的回收。

方法区在JVM启动的时候被创建，它的大小决定了能保存

的类的数量，如果超出，会出现oom的异常。



方式区使用上述的Meta Data来实现，使用的不是jvm的内存，而是物理机器的内存，

所以它的上限是物理机器的内存。

启动的时候加入虚拟机参数：-XX:+PrintFlagsFinal，在一堆输出中找到下面元空间的初始值和最大值

MetaspaceSize   = 21807104   =  20.8M

MaxMetaspaceSize    = 4294901760   = 4G

当元空间的大小使用量到达初始值大小就会触发一次full gc，如果还不够的话，就会扩展大小，但是最大不能

超过MaxMetaspaceSize的大小。

这里验证下full gc的产生，回看到原来的那个图，然后选中Meta Data

![jdk83](D:\jeffchan\markdown\image\jdk83.png)

可以看到，我这个程序运行起来，Meta Data基本就8M多，所以我们可以通过：

```java
-XX:MetaspaceSize=6M  6M内存
-XX:MaxMetaspaceSize=200M  200M内存
```

来设置大小，从而触发full gc

![jdk84](D:\jeffchan\markdown\image\jdk84.png)

这里设置了元空间的初始大小为6M，同时也发生了元空间也发生了Full GC。这里是需要为什么要jconsole连接上才会Full Gc。

后来我试了设置成4M，发现确实可以一下就打印出来。看来应该是程序本身加载的类并没有达到6M，应该是连接jconsole的时候，还会加载类，占用了一部分空间，达到8M就触发了。



这个我也做个实验了(vm参数中加入:-verbose:class即可)，确实是这样子。这里我只是截取出打印的一部分记录：

```java
[Loaded sun.rmi.transport.SequenceEntry from D:\software\program-tool\javaSE1.8\jdk1.8\jre\lib\rt.jar]
[Loaded java.util.concurrent.locks.LockSupport from D:\software\program-tool\javaSE1.8\jdk1.8\jre\lib\rt.jar]
[Loaded sun.rmi.server.MarshalOutputStream$1 from D:\software\program-tool\javaSE1.8\jdk1.8\jre\lib\rt.jar]
```



当元空间经过Full gc也不够的时候，就会抛出异常。

-XX:MaxMetaspaceSize=4M 设置最大大小为4M,那么对于这个程序来说肯定是不够内存的。运行程序，打印结果如下图:

![jdk85](D:\jeffchan\markdown\image\jdk85.png)

可以发现，已经抛出异常。

一般来说，元空间垃圾回收的机会比较小，因为卸载类的信息是非常严格的，所以你可以试下在jconsole中，那个执行gc的按钮，进行gc的操作，但是你会发现，基本没有回收元空间的内存，一般会触发这里gc的原因，都是那个自定义加载器加载的那些类。比如cglib技术加载的类等。



1.1、Class文件介绍                                                                                                   

class文件中只有两种数据类型，一种是无符号数，一种是表，无符号数是基本数据类型

以u1、u2、u4、u8分别表示1，2，4，8字节的无符号数。无符号可以用来描述数字，

索引引用，数量值或者UTF-8表面的字符串。

表是由多个无符号数或者其他表组成的复合数据类型。以_info结尾。

```java
package cn.yishijie.jvm;
public class MethodArea {
    private String a;
    public static void main(String[] args) {
        String b = "jeff.chan";
    }
}

```

对应的二进制文件内容：

```java
CA FE BA BE 00 00 00 34 00 19 0A 00 04 00 15 08 00 16 07 00 17 07 00 18 01 00 01 61 01 00 12 4C 6A 61 76 61 2F 6C 61 6E 67 2F 53 74 72 69 6E 67 3B 01 00 06 3C 69 6E 69 74 3E 01 00 03 28 29 56 01 00 04 43 6F 64 65 01 00 0F 4C 69 6E 65 4E 75 6D 62 65 72 54 61 62 6C 65 01 00 12 4C 6F 63 61 6C 56 61 72 69 61 62 6C 65 54 61 62 6C 65 01 00 04 74 68 69 73 01 00 1C 4C 63 6E 2F 79 69 73 68 69 6A 69 65 2F 6A 76 6D 2F 4D 65 74 68 6F 64 41 72 65 61 3B 01 00 04 6D 61 69 6E 01 00 16 28 5B 4C 6A 61 76 61 2F 6C 61 6E 67 2F 53 74 72 69 6E 67 3B 29 56 01 00 04 61 72 67 73 01 00 13 5B 4C 6A 61 76 61 2F 6C 61 6E 67 2F 53 74 72 69 6E 67 3B 01 00 01 62 01 00 0A 53 6F 75 72 63 65 46 69 6C 65 01 00 0F 4D 65 74 68 6F 64 41 72 65 61 2E 6A 61 76 61 0C 00 07 00 08 01 00 09 6A 65 66 66 2E 63 68 61 6E 01 00 1A 63 6E 2F 79 69 73 68 69 6A 69 65 2F 6A 76 6D 2F 4D 65 74 68 6F 64 41 72 65 61 01 00 10 6A 61 76 61 2F 6C 61 6E 67 2F 4F 62 6A 65 63 74 00 21 00 03 00 04 00 00 00 01 00 02 00 05 00 06 00 00 00 02 00 01 00 07 00 08 00 01 00 09 00 00 00 2F 00 01 00 01 00 00 00 05 2A B7 00 01 B1 00 00 00 02 00 0A 00 00 00 06 00 01 00 00 00 03 00 0B 00 00 00 0C 00 01 00 00 00 05 00 0C 00 0D 00 00 00 09 00 0E 00 0F 00 01 00 09 00 00 00 3C 00 01 00 02 00 00 00 04 12 02 4C B1 00 00 00 02 00 0A 00 00 00 0A 00 02 00 00 00 08 00 03 00 09 00 0B 00 00 00 16 00 02 00 00 00 04 00 10 00 11 00 00 00 03 00 01 00 12 00 06 00 01 00 01 00 13 00 00 00 02 00 14
```

一个字节有8个bit,上面的内容中，两个十六进制数为一组，表示一个字节。

class文件的头4个字节CA FE BA BE被称为魔数，用于标识它能否被虚拟机接受。

后两个字节是次版本号，再后两个字节是主版本号。0x34 = 52 表示java的主版本号为java8

接着就是常量池入口，前两个字节代表常量池里的常量个数。0x19=25,第0位有特殊含义，所以这里表示

有24个常量。常量池里数值代表的意义：

```java
Constant Type	Value
CONSTANT_Class	7
CONSTANT_Fieldref	9
CONSTANT_Methodref	10
CONSTANT_InterfaceMethodref	11
CONSTANT_String	8
CONSTANT_Integer	3
CONSTANT_Float	4
CONSTANT_Long	5
CONSTANT_Double	6
CONSTANT_NameAndType	12
CONSTANT_Utf8	1
CONSTANT_MethodHandle	15
CONSTANT_MethodType	16
CONSTANT_InvokeDynamic	18
```

