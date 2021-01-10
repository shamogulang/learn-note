# <center>1、基本IO</center>

## 1.1、File API操作

```java
package cn.yishijie.io;

import java.io.File;

public class FileTest {

    public static void main(String[] args) throws Exception {
        // 通过绝对路径创建文件
        File file = new File("D:\\jeffchan\\workspace\\thread\\123.jpg");

        // 通过父路径和子路径来创建文件
        file = new File("D:\\jeffchan\\workspace\\thread","123.jpg");

        // 获取文件的大小
        System.out.println(file.length()); //  5699

        // 获取绝对路径
        System.out.println(file.getAbsolutePath()); // D:\jeffchan\workspace\thread\123.jpg

        // 获取文件名称
        System.out.println(file.getName()); // 123.jpg

        // 当前环境的相对路径
        System.out.println(System.getProperty("user.dir")); // D:\jeffchan\workspace\thread

        // 通过相对路径创建文件
        file = new File("123.jpg");
        System.out.println(file.getAbsolutePath()); // D:\jeffchan\workspace\thread\123.jpg

        // 路径分割符号
        System.out.println(File.separator);// \

        // 文件是否存在
        System.out.println(file.exists());

        // 文件是否是目录
        System.out.println(file.isDirectory());
        // 文件是否是文件
        System.out.println(file.isFile());

        // 创建新文件，不存在创建，返回true，存在返回false
        File tempFile = new File("caraliu");
        System.out.println(tempFile.createNewFile());
        // 删除文件
        System.out.println(tempFile.delete());

        File tempFile1 = new File("cara/liu");
        // mkdir只能在上一次目录存在时，才会创建目录
        System.out.println(tempFile1.mkdir());
        // 上一级目录不存在，先创建上一级目录，然后创建下一级目录
        System.out.println(tempFile1.mkdirs());

        File tempFile2 = new File("D:\\jeffchan\\workspace\\thread");
        // 列出当前目录下所有文件或者目录的名称
        String[] names = tempFile2.list();
        for (String name : names) {
            System.out.println(name);
        }
        // 列出当前目录下的所有文件实例
        File[] files = tempFile2.listFiles();
        for (File temp : files) {
            System.out.println(temp.getName());
        }
    }
}
```

打印当前目录下所有的文件类型的名称：

```java
    public static void main(String[] args) throws Exception {
        // 打印出当前目录以下的所有文件类型的名字
        File file = new File("D:\\jeffchan\\workspace\\thread");
        printFileName(file);
    }

    public static void printFileName(File file){
        if(file.isFile){
            return;
        }
        File[] files  = file.listFiles();
        for (File temp : files) {
            if(temp.isFile()){
                System.out.println(temp.getName());
            }else if(temp.isDirectory()){
               printFileName(temp);
            }
        }
    }
```

## 2、非阻塞io

客户端代码：

```java
 public static void main(String[] args) throws Exception{
        // 初始化channel
        SocketChannel channel = SocketChannel.open(new InetSocketAddress("127.0.0.1",8888));

        // 切换成非阻塞模式
        channel.configureBlocking(false);

        // 分配指定大小的缓存区
        ByteBuffer allocate = ByteBuffer.allocate(1024);
        // 将数据写入缓存区
        allocate.put(LocalDateTime.now().toString().getBytes());
        // 切换成读取模式
        allocate.flip();
        // 将数据写出到通道中
        channel.write(allocate);
        // 将缓冲区清空
        allocate.clear();
        // 关闭通道
        channel.close();
    }
```



服务端代码：

```java
  public static void main(String[] args) throws Exception{
        // 初始化ServerSocketChannel
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();

        // 切换为非阻塞模式
        serverSocketChannel.configureBlocking(false);

        // 绑定端口号
        serverSocketChannel.bind(new InetSocketAddress(8888));

        // 获取选择器
        Selector selector = Selector.open();
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        // 如果有事件来临
        while (selector.select() > 0){
            // 获取所有选择器事件
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();

            while (iterator.hasNext()){
                SelectionKey next = iterator.next();
                // 如果是可以接受状态的
                if(next.isAcceptable()){
                    // 获取通道
                    SocketChannel accept = serverSocketChannel.accept();
                    // 切换成非阻塞
                    accept.configureBlocking(false);
                    accept.register(selector,SelectionKey.OP_READ);
                }else if(next.isReadable()){
                    // 获取读数据通道
                    SocketChannel channel = (SocketChannel)next.channel();

                    ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

                    byteBuffer.put("caraliu".getBytes());

                    int lent = 0;
                    while ((lent = channel.read(byteBuffer)) != -1){
                        byteBuffer.flip();
                        System.out.println(new String(byteBuffer.array(),0,lent));
                        byteBuffer.clear();
                    }
                }
                // 取笑key
                iterator.remove();
            }
        }
    }
```

