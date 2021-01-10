Exception in thread "main" java.lang.NoClassDefFoundError

编译通过了，但是运行时，类加载器找不到对应的类

编译运行一次，发现没问题，

然后去编译产生class的目录，

将对应的.class文件删除掉，那么就解决问题了。