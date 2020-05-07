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
        // 打印出当前目录以下的所有文件类型的文件的名字
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



## 1.2、InputStream

FileInputStream的使用

```java
 public static void main(String[] args) {
        InputStream in = null;
        try {
            // 读取文件hello.txt，转成二进制流
            in  = new FileInputStream(new File("hello.txt"));
            
            // 打印读入文件的大小，字节为单位
            System.out.println(in.available());
            
            int lent = 0;
            //  存储读入的二进制流
            byte[] b = new byte[10];
            while (in.read(b) != -1){
                System.out.println(new String(b,"utf-8"));
            }
        }catch (Exception e){

        }finally {
            if(in != null){
                try {
                    in.close();
                }catch (Exception e){

                }
            }
        }
    }
```

该类一般用于读取图片等二进制，不能用于读取文字，引入如果有中文时，可能出现乱码的情况

## 1.3、OutputStream

FileOutputStream的使用

```java
 public static void main(String[] args) {
        OutputStream out = null;
        try {
            // 输入的文件
            out  = new FileOutputStream(new File("hello1.txt"));
            // 将hello输入到hello1.txt文件中
            out.write("hello".getBytes());
        }catch (Exception e){

        }finally {
            if(out != null){
                try {
                    out.close();
                }catch (Exception e){

                }
            }
        }
    }
```

## 1.4、Reader

FileReader的使用

```JAVA
public static void main(String[] args) {
        Reader reader = null;
        try {
            // 读取文件hello1.txt，转成字符流
            reader  = new FileReader(new File("hello1.txt"));
            int lent = 0;
            char[] chars = new char[10];
            while ((lent=reader.read(chars)) != -1){
                System.out.println(new String(chars));
            }
        }catch (Exception e){

        }finally {
            if(reader != null){
                try {
                    reader.close();
                }catch (Exception e){

                }
            }
        }
    }
```

该类一般用于读取文件中的纯文本内容，不会出现乱码的情况。

### 1.41、BufferReader

```java
public static void main(String[] args) {
        BufferedReader bufferedReader = null;
        try {
            // 读取文件hello.txt，加缓存，可以一行一行读取
            bufferedReader  = new BufferedReader(new FileReader(new File("hello1.txt")));
            String lineContent = bufferedReader.readLine();
            while (lineContent != null){
                System.out.println(lineContent);
                lineContent = bufferedReader.readLine();
            }
        }catch (Exception e){

        }finally {
            if(bufferedReader != null){
                try {
                    bufferedReader.close();
                }catch (Exception e){

                }
            }
        }
    }
```

该类可以提供buffer，较少io的次数，提高效率

```java
// 默认的缓冲的大小为8192
int defaultCharBufferSize = 8192;
```

## 1.5、Writer

```java
public static void main(String[] args) {
        Writer writer = null;
        try {
           
            writer  = new FileWriter(new File("hello1.txt"));
            // 直接将字符内容输出到文件hello1.txt中
            writer.write("本来讨厌下雨的天空，直到听到有人说爱我");
        }catch (Exception e){

        }finally {
            if(writer != null){
                try {
                    writer.close();
                }catch (Exception e){

                }
            }
        }
    }
```

### 1.51、BufferWriter

```java
public static void main(String[] args) {
        BufferedWriter bufferedWriter = null;
        try {
         
            bufferedWriter  = new BufferedWriter(new FileWriter(new File("hello1.txt")));
            // 带缓存的将内容写出到hello1.txt中
            bufferedWriter.write("天空灰得像哭过，离开你之后，并没有更自由");
        }catch (Exception e){

        }finally {
            if(bufferedWriter != null){
                try {
                    // 还没有关闭之前，如果写出的内容不超出默认的大小，不会写出到文件中
                    // 除非手动调用flush方法
                    bufferedWriter.close();
                }catch (Exception e){

                }
            }
        }
    }
```

