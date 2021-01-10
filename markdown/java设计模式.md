<center><h1>java设计模式</h1></center>
前言：以下的设计模式大部分是从开源项目java-design-patterns中来，主要是学习过程中的一些笔记。

java-design-patterns的git地址： https://github.com/iluwatar/java-design-patterns.git

java-design-patterns学习资料链接： https://java-design-patterns.com/patterns/

我的学习git地址（使用gradle开发）：

## 1、策略模式

策略模式关注在于行为，通过一个执行类，包含一个算法策略的抽象类或者是接口属性，然后执行属性对应的方法，因为不同实现的算法，行为不一样，从而可以表现出不同的行为。

比如有  

执行类A(里面有个属性为算法的抽象类或者是接口B), 抽象类或者接口B有众多算法实现类：b0,b1,b2

![策略模式](D:\jeffchan\markdown\java-pattern-img\策略模式.png)

那么执行类A组合算法抽象类或者接口B,就根据绑定的具体算法实现类来体现不同的执行行为。就是在do()方法中执行算法抽象类或者接口B的算法抽象，然后根据不同的绑定，动态调用不同的具体实现。



接下来通过商品折扣的一个例子来解释策略模式，具体的类的关系图如下：

![SaleDiscount](D:\jeffchan\markdown\java-pattern-img\SaleDiscount.png)

上述中的SaleDiscount是抽象的商品折扣类（即是策略类）SaleHandler是商品打折的执行类：里面会有SaleDiscount的实例。可以根据该类型实例的不同子类来体现不同的行为。



SaleDiscount.java  // 抽象折扣策略接口

```java
package cn.yishijie.strategy;

import java.math.BigDecimal;
// 函数接口标记，后续需要的策略比较简单时，可以直接使用lambda表达式的形式，而不用添加一个策略类
@FunctionalInterface 
public interface SaleDiscount {
    // 折扣的策略，输入购买的金额，返回打折后的金额
    BigDecimal discount(BigDecimal buyAmount);
}
```



SaleAmountPointDiscount.java  // 具体打折策略类

```java
package cn.yishijie.strategy;

import java.math.BigDecimal;

public class SaleAmountPointDiscount implements SaleDiscount{
    // 可以根据传入的折扣，进行不同折扣的优惠
    private BigDecimal discountPoint;

    public SaleAmountPointDiscount(BigDecimal discountPoint) {
        this.discountPoint = discountPoint;
    }

    // 入参buyAmount为客户购买的总的金额
    @Override
    public BigDecimal discount(BigDecimal buyAmount) {
        // 对购买的总金额进行打折操作，乘于折扣discountPoint
        return buyAmount.multiply(discountPoint);
    }
}
```



SaleUpperAmountMinusAmountDiscount.java  // 具体打折策略类

```java
package cn.yishijie.strategy;

import java.math.BigDecimal;

/**
 * 满多少金额减多少金额策略类
 */
public class SaleUpperAmountMinusAmountDiscount implements SaleDiscount{
    // 设置的上限金额
    private BigDecimal crossOverAmount;
    // 设置优惠的金额
    private BigDecimal minusAmount;

    public SaleUpperAmountMinusAmountDiscount(BigDecimal crossOverAmount, BigDecimal minusAmount) {
        this.crossOverAmount = crossOverAmount;
        this.minusAmount = minusAmount;
    }

    @Override
    public BigDecimal discount(BigDecimal buyAmount) {
        // 如果客户购买的金额超过上限金额crossOverAmount，那么就要优惠minusAmount
        // 即减去金额minusAmount
        return crossOverAmount.compareTo(buyAmount) < 0 ? buyAmount.subtract(minusAmount) : buyAmount;
    }
}
```



SaleHandler.java // 具体打折的操作类

```java
package cn.yishijie.strategy;

import java.math.BigDecimal;

public class SaleHandler {

    private SaleDiscount saleDiscount;

    public SaleHandler(SaleDiscount saleDiscount) {
        this.saleDiscount = saleDiscount;
    }
    // 修改打折策略
    public void changeSaleStrategy(SaleDiscount saleDiscount){
        this.saleDiscount = saleDiscount;
    }
    
    // 打折后的金额
    public BigDecimal afterDiscount(BigDecimal buyAmount){
        // 打折操作
        return saleDiscount.discount(buyAmount);
    }
}
```



这个类用于测试上述的设计是否起作用：

```java
package cn.yishijie.strategy;

import org.junit.Test;

import java.math.BigDecimal;

public class SaleHandlerTest {

    @Test
    public void afterDiscount() {
        // 打7折
        SaleHandler saleHandler = new 
            SaleHandler(new SaleAmountPointDiscount(BigDecimal.valueOf(0.7)));
        // 也可以通过lambda表达式
        //SaleHandler saleHandler = new 
        //   SaleHandler(buyAmount -> buyAmount.mutiply(BigDecimal.valueOf(0.7)));
       
        // assert 断言，当运行结果和预期不一致的时候，会抛出异常
        assert saleHandler.afterDiscount(BigDecimal.valueOf(100))
            .compareTo(BigDecimal.valueOf(70)) == 0;

        // 换策略 满两百减20
        saleHandler.changeSaleStrategy
            (new SaleUpperAmountMinusAmountDiscount(
                BigDecimal.valueOf(200), BigDecimal.valueOf(20)
        ));
        assert saleHandler.afterDiscount(BigDecimal.valueOf(300))
            .compareTo(BigDecimal.valueOf(280)) == 0;
        assert saleHandler.afterDiscount(BigDecimal.valueOf(180))
            .compareTo(BigDecimal.valueOf(180)) == 0;
    }
```



其实Runnable接口就是使用了策略模式：

PrintThread

```java
package cn.yishijie.run;
/**
 * Runnable 为策略抽象接口
 * PrintThread是具体的策略类
 */
public class PrintThread implements Runnable {

    @Override
    public void run() {
        for(int i = 0; i < 30; i++){
            System.out.println(i);
        }
    }
}
```



测试类：

```java
package cn.yishijie.run;

public class Client {

    public static void main(String[] args) {
        // Thread相当于SaleHandler
        Thread thread = new Thread(new PrintThread());
        // start方法，相当于SaleHandler的afterDiscount(xxx)方法
        thread.start();

        // 也可以使用lambda
        new Thread(() -> {
            for (int i =0; i< 30; i++){
                System.out.println(i);
            }
        }).start();
    }
}
```

所以平时我们的一个一个任务，其实就是一种一种run的策略，它们的抽象接口为Runnale。



## 2、单例模式



饿汉式：

```java
package cn.yishijie.singleton;

public class EagerModeSingleton {

    // 私有化构造器，防止别人创建新的实例对象
    private EagerModeSingleton(){}
    // 类加载时就会实列化
    private static EagerModeSingleton instance = new EagerModeSingleton();
    // 也可以通过这种静态代码块的方式，这个跟上述的方式
    // 是一样的，同样加载类的时候
    // private static EagerlySingleton instance;
    // static {
    //   instance = new EagerlySingleton();
    // }
 
    public static EagerModeSingleton getInstance(){
        return instance;
    }
}
```

这种模式，线程安全，但无法实现懒加载（就是调用该方法的时候才实列化）调用这个方法的时候，肯定会触发这个类的对静态代码块的初始化，此时代码就会为instance变量分配内存，如果使用到，那就好，但是也有可能是其他动作触发的，但是触发完后，可能没使用到这个实例，那么就造成内存泄漏了。这种模式是直接创建对象的形式，当单列类被加载初始化的时候，就会创建对象，分配内存，不管是否已经在使用这个对象。

ps:  内存泄漏是指一些没用对象占用了一部分内存，导致这部分内存不能使用；

​       还有一个比较常见的内存溢出，这个是指对象占用的内存太多， 导致内存不够用



懒汉式：

```java
package cn.yishijie.singleton;

public class LazyModeSingleton {

    // 当使用getInstance2方法的时候，这里要加入volatile
    // 关键字，防止指令的重排，出现设置了对象应用，却还没
    // 初始化完的情况，这样可能会出现异常
    private static LazyModeSingleton lazyModeSingleton;
    // 私有化构造器，防止创建多个对象
    private LazyModeSingleton() {
    }

    // 可以实现懒加载，就是要使用的时候，调用获取单列的
    // 方法，然后才给属性分配内存。但是这种方式，会出现
    // 线程安全问题，
    public static LazyModeSingleton getInstance(){
        if(lazyModeSingleton == null){
            // 这里加入sleep的代码，然后开启多个线程调用
            // getInstance方法，会很明显出现多个对象的情况
            lazyModeSingleton = new LazyModeSingleton();
        }
        return lazyModeSingleton;
    }

    // 线程安全，同时也是懒加载，当需要使用实例时，才会分配内存
    // 但是synchronized这个锁会影响性能，只能一个一个排队获取
    // 对应的实例，当其中一个线程进行获取的时候，其他线程要阻塞
    // 排队等候
    public static synchronized LazyModeSingleton getInstance1(){
        if(lazyModeSingleton == null){
            lazyModeSingleton = new LazyModeSingleton();
        }
        return lazyModeSingleton;
    }

    // 线程安全，同时也是懒加载，当需要使用实例时，才会分配内存
    // double check的方式性能较getInstance1有提升，因为在锁住
    // 整个方法，当线程抛入到第一层的时候，那么已经进入的对象
    // 会在这里进行阻塞，当第一个线程已经完成实例化之后，后续来
    // 调用这个方法的线程，都会在第一个判断中检验不通过，直接
    // 返回实例对象，不像getInstance1会一直拿锁的情况。这里要
    // 注意把属性lazyModeSingleton 加上volatile
    public static LazyModeSingleton getInstance2(){
        if(lazyModeSingleton == null){
            // 第一层
            synchronized (LazyModeSingleton.class){
                // 第二层
                if(lazyModeSingleton == null){
                    lazyModeSingleton = new LazyModeSingleton();
                }
            }
        }
        return lazyModeSingleton;
    }

    private static class LazyMode{
        private static final LazyModeSingleton INSTANCE = new LazyModeSingleton();
    }

    // 线程安全，同时效率很高，也是懒加载，因为要主动去
    // 调用方法时，才会去拿静态内部类的属性，从而触发
    // 初始化操作，才会分配内存
    public static LazyModeSingleton getInstance3(){

        return LazyMode.INSTANCE;
    }
}
```

getInstance()可以实现懒加载，但不是线程安全的方式，就是多线程调用这个方法的时候，可能会产生多个对象，那么就不是单例了。尤其是在判空和new对象之前，加入sleep的动作，这个现象会更加明显。当可能出现并发的时候，不推荐使用这种方式。



getInstance1()线程安全的方式，但是直接用同步方法，性能会差一点，因为当一个线程来访问的时候，会拿锁，其他线程就必须等待这个线程完成后，才能进入这个方法，所以其他线程会阻塞，性能比较差。



getInstance2()这个是double check的方式，就是在外层会先判空，如果为空，那么进入到判空里面，那么此时进入同步块的线程只能一个一个进入，当一个线程进入后，其他线程必须等待，当这个线程完成对象的创建后，已经分配了内存地址和空间，那么后续还有其他线程来调用这个方法的时候，会直接获取到同一个对象返回；而此时进入到判空里面等待的线程，也会一个一个进入到同步块，判空不通过后，直接返回同一个对象。这里锁的粒度更小了，性能比较好。这里要注意给实例属性加volatile，防止出现指令重排，导致一个对象分配了引用地址，却还没来得及初始化，就被其他线程拿到，出现异常的情况。



getInstance3()这个是通过静态内部类的方式来实现懒加载，还有多线程加载类的时候是线程安全的，虚拟机在多线程的环境下会保证只有一个线程调用初始化方法，其他线程必须阻塞等待。所以这种方式也是线程安全的。



枚举:

```java
package cn.yishijie.singleton;

public enum  EnumModeSingleton {

    INSTANCE;
    // effected java 作者推荐使用的方式
    // 效率高，并且还可以防止反射获取，可是
    // 就是不太清楚有啥用？怎么用？
    public EnumModeSingleton getInstance(){

        return EnumModeSingleton.INSTANCE;
    }
}
```

effected java 作者推荐使用的方式,效率高，并且还可以防止反射获取，可是就是不太清楚有啥用？怎么用？



## 3、责任链模式

责任链模式可以通过一条链式的方式，将各个处理类串联起来，当满足某一个条件的时候，就进行处理。每一个处理类只负责自己需要做的事情，如果不是自己的事情，那么就交给下一个处理类处理。当处理类处理后，就直接返回，后续的处理器就不必循环，如果走到末尾还是没有处理，应该要有一个合理的缺省的处理，防止出现异常。

![责任链](D:\jeffchan\markdown\java-pattern-img\责任链.png)

如上图，这里只是给出了责任链处理类之间的关系图。经过客户端或者你可以有一个HandlerHelper的类，帮你把上面的具体实现，串成一条链，在处理方法中根据条件，选择是否要处理。



假设有这个一个场景，需要处理各种媒体类型的数据，比如文本，音频，视频，不同类型的媒体，由不同类型的处理类进行处理，给出下面设计：

![Handler](D:\jeffchan\markdown\java-pattern-img\Handler.png)

这里Handler类不知为什么画不出聚合自己的一条线，注意Handler抽象类，会有一个Handler类型的属性。



Handler.java

```java
package cn.yishijie.chain;

public abstract class Handler {

    private  Handler next;

    // 用于设置下一个处理器
    public Handler(Handler next) {
        this.next = next;
    }

    public void handleRequest(Request req) {
        if (next != null) {
            next.handleRequest(req);
        }
    }
}
```



RequestType.java // 请求类型

```java
package cn.yishijie.chain;

public enum  RequestType {

    TEXT, AUDIO, VIDEO
}
```



Request.java // 请求信息

```java
package cn.yishijie.chain;

public class Request {
    // 请求类型
    private RequestType requestType;

    public RequestType getRequestType() {
        return requestType;
    }

    public void setRequestType(RequestType requestType) {
        this.requestType = requestType;
    }
}
```



三个实现类：

```java
package cn.yishijie.chain;

public class TextHandler extends Handler {

    public TextHandler(Handler next) {
        super(next);
    }

    @Override
    public void handleRequest(Request req) {
        if (req.getRequestType() == RequestType.TEXT) {
            System.out.println("text...");
        } else {
             // 调用下一个处理器的处理方法
            super.handleRequest(req);
        }
    }
}


package cn.yishijie.chain;

public class AudioHandler extends Handler {

    public AudioHandler(Handler next) {
        super(next);
    }

    @Override
    public void handleRequest(Request req) {
        if(req.getRequestType() == RequestType.AUDIO){
            System.out.println("audio...");
        }else{
            // 调用下一个处理器的处理方法
            super.handleRequest(req);
        }
    }
}


package cn.yishijie.chain;

public class VideoHandler extends Handler {

    public VideoHandler(Handler next) {
        super(next);
    }

    @Override
    public void handleRequest(Request req) {
        if(req.getRequestType() == RequestType.VIDEO){
            System.out.println("video xxx");
        }else{
             // 调用下一个处理器的处理方法
            super.handleRequest(req);
        }
    }
}
```



我这里给一个工具类，将这三个实现类串起来：

```java
package cn.yishijie.chain;

public class HandlerHelper {

   public Handler buildHandler(){
       // 这里没有设置成环的情况，最后一个的下一个处理器为null
       // 当走到最后也没有处理器处理的时候，根据抽象类Handler的处理方法
       // handleRequest会进行非空判断，那么就直接退出，不会产生异常
       return new TextHandler(new AudioHandler(new VideoHandler(null)));
   }
}
```



测试用例：

```java
package cn.yishijie.chain;

public class Client {

    public static void main(String[] args) {
        HandlerHelper handlerHelper = new HandlerHelper();
        Handler handler = handlerHelper.buildHandler();
        Request request = new Request();
        // 请求类型换不同的类型，可以使用不同的处理器处理
        request.setRequestType(RequestType.VIDEO);
        handler.handleRequest(request);
    }
}
```

通过上述的Client的代码可以直到，客户端不需要知道具体的处理类是谁，直接将请求给handler处理，那么总会由相应的处理类进行业务的处理，当然如果类型不存在，那么缺省状态就是退出调用链，跟没事发生过一样。这样客户端和处理类的耦合就降低了，不许具体到实际的处理类。那么后续比如我要加其他类型，那么直接扩展一个处理类和在工具类HandlerHelper中将该处理类加入到调用链中即可。

不过不好的地方是：如果调用链太过长，那么就会对性产生一点影响（我个人觉得基本没影响，就是多判断几次而已），有个比较麻烦的地方就是，查找代码的时候会比较麻烦，因为我现在所在的公司就经常用到这个设计模式，找对应的实现类时，会由一点麻烦。（你可以通过对类型的调用来找到具体的实现类，我工作上就是这么做的）



还有一种责任链方式，就是不区分类型的，比如我现在公司的一个采购业务，要对订单进行校验。此时会对输入参数，用户的权限，订单记录进行校验，但是这里没有说那种类型，而是要全部都通过后，才会进行订单的保持操作。



### 4、装饰者模式

装饰者模式其实就是添加附加功能的一种方式。就是说一个类已经有了基本功能，那么添加额外的功能要怎么添加才比较好。一般情况的话，你可以在该类中添加一个方法，然后在调用原来的方法之前或者之后调用，那么就相当于进行相应的装饰。这种方式不太好的方面是就是违反开闭原则。还有进行不够灵活，比如要各种装饰的搭配也不好解决。装饰模式可以通过多态的方式来解决这种问题，而且可以自由搭配。

注意装饰器不可以单独使用，它必须指定一个装饰器或者一个实际的被装饰的对象。



![装饰器](D:\jeffchan\markdown\java-pattern-img\装饰器.png)

比如这里SwimDecorator和FlyDecorator都是装饰器，都有一个非空的构造器，必须指定装饰器或者是实际的被装饰对象（ConreteA也需要实现Decorator）。最终的闭环是实际的被装饰对象，而不是装饰类。



Decorator:

```java
package cn.yishijie;

public interface Decorator {

    void decorator();
}
```



ConcreteA:

```java
package cn.yishijie;

public class ConcreteA implements Decorator {

    @Override
    public void decorator() {
        System.out.println("concreteA ...");
    }
}
```



SwimDecorator:

```java
package cn.yishijie;

public class FlyDecorator implements Decorator {

    private Decorator decorator;
    public FlyDecorator(Decorator decorator) {
        this.decorator = decorator;
    }

    @Override
    public void decorator() {
        decorator.decorator();
        System.out.println("add fly function...");
    }
}
```



FlyDocorator:

```java
package cn.yishijie;

public class SwimDecorator implements Decorator {

    private Decorator decorator;

    public SwimDecorator(Decorator decorator) {
        this.decorator = decorator;
    }

    @Override
    public void decorator() {
        decorator.decorator();
        System.out.println("add swim function...");
    }
}
```



Client:

```java
package cn.yishijie;

public class Client  {

    public static void main(String[] args) {
        ConcreteA concreteA = new ConcreteA();
        concreteA.decorator();
        FlyDecorator flyDecorator = new FlyDecorator(concreteA);
        flyDecorator.decorator();
        SwimDecorator swimDecorator = new SwimDecorator(flyDecorator);
        swimDecorator.decorator();
    }
}
```

可以发现，这个装饰的作用可以随意搭配，就是各种装饰对象都是相互当作被装饰的对象。比如ConreateA需要相应的额外功能就可以添加一个装饰类，而且这个装饰类还可以复用，组成不同的组合。比如这里SwimDecorator和FlyDecorator都可以和ConcreteA搭配使用。



### 5、组合设计模式

