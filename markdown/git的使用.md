<center><h1>git的使用</h1></center>
前提：首先要注册一个github账号： https://github.com/ ，然后要安装Git的工具：https://git-scm.com/download/win 



# 1、SSH协议

这里不是介绍这个协议，而是通过这个协议来使用代码的上传和下载。如果没有生成publickey的话，那么拉取代码的时候，会出现： Permission denied

```shell
$ git clone git@github.com:shamogulang/git-learn.git
Cloning into 'git-learn'...
git@github.com: Permission denied (publickey).
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```



## 1.1、通过ssh-keygen生成publickey

1、 在生成key之前，先告诉git你的身份和邮件：

```
git config --global user.name "随便填你想填的名字"
git config --global user.email "随便填你想要填的邮箱"

这里网上资料都是说填写你的账号和邮箱，我一直认为是github的账号和邮箱，其实可以不是，随便填都可以，我填了xxx和xxx,也是可以的。
其实这里还是要填你的名字和真实的邮箱比较好，因为在开源的项目，你在项目上提交了代码的话，别人就能知道你是谁，还有邮箱是多少，别人想联系你也比较方便。
```



2、使用ssh-keygen命令生成key:

```java
ssh-keygen -t rsa -C "随便填你想填的东西，不过大部分都是填一个邮箱"

在git base中输入上述的指令，然后一直回车就行。-C后面的内容也是可以随便填的，我这里填了xxd2，后续在下面生成的publickey中的id_rsa.pub文件的内容中就有这个字符：xxd2。
这里的-t是指什么类型的意思，-C是指comment，就是对这个key的注释的意思
```

对应的截图：

![pkey](D:\jeffchan\markdown\image\pkey.png)



3、将~/.ssh目录下的publickey拷贝出来：

```java
 cat ~/.ssh/id_rsa.pub
 
 ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC3+KX6QQLtvda6npWHsJYb5fdGvSMICqX4TFUaZdMnkBNtol2qZLlK7x2Cdm+9PbRCcmtjh7tqPtq/Tm4J1qko29WAqrdE9hLIC0rNkgw8ktJkmh8qcPFFeBqPHkign7jHr30onRcfduPhQmGFTwHs/dwvHPAerDIkIHgbxCpU/K4QmENyxsdfasdfQJaKkMleOZ/nhT5fwcmAAWb5fTOPZmxg01H9PWzHtGCOAg5+domf6B2v7p+Xh4r7O8tqSW1waKlNhD4tRktryUSpsahdDBWTs5euu+JyDiGBI27OO71YqdjNCcpiZOGS3uwj0PNIj0PIB6Z455rwF33UQDwm+HQuBFAdZryeXtEh9NDMLb94kdySjyUu2FuqanCXPHeqAAC9ByKWe6K+P5ZcoPh+DeKrMgf1T0kJZWCwGIwQGzZP3mNiummCC7uerD9VCkGpdH8UDXNEtSqE+BlPCeVXtM08yF83R/qbabMiaeXU6OYI3Q/9CMukv0= xxd2
```



4、登陆github，然后将上述的key复制到github的ssh keys中即可：

右上角的settings --> SSH and GPG keys --> News SSH key添加标题加入key。然后点击 Add SSH key即可

![github](D:\jeffchan\markdown\image\github.png)



## 1.2、git区的简单介绍

在讲指令之前先讲几个区: 

本地仓库：存在.git文件夹中。里面有分支，分支里有暂存区。

远程仓库：比如github，公司内部的私服，比如gitlab。是托管代码仓库。

工作区：这个是我们开发的区域



## 1.3、简单开发模式



1、git clone xxx 这个是很常用的指令：就是将github上的代码clone到本地。



2、git status 查看修改或增加的文件

![gitstatus](D:\jeffchan\markdown\image\gitstatus.png)

​     cjh这个文件，我增加了几个字符，那么就会出现上述的提示，README.md文件是我这次新加的。



3、git diff 可以查看到修改的地方:



4、git add .    或者   git add 具体文件的路径，将工作区的代码保存到暂存区中。

![gitstatus1](D:\jeffchan\markdown\image\gitstatus1.png)

5、git commit -m "注释"， 将暂存区的代码合并到当前分支。



6、git push origin master 将代码提交到远程master分支

![gitpush](D:\jeffchan\markdown\image\gitpush.png)

​        当初的user.name=xxx就出现在这里的提交提示了,所以才说取自己的名字，不然不知道是谁提交的。first     commimt这个是git commit时的注释



 7、解决冲突。这个确实有点恶心，这个冲突是因为有多个人某个时刻，从某个分支拉取出代码，然后进行开发，同时有多个人对某个文件的行号一致的地方进行了修改，然后就会出现冲突。

本地建两个文件夹，分别为jeffchan1和jeffchan2，然后同时修改cjh的第一行。然后jeffchan1先提交。然后jeffchan2再提交，看下情况。

发现提交不了：

```java
caraliu@DESKTOP-I77F64F MINGW64 ~/Desktop/jeffchan2/git-learn (master)
$ git push origin master
To github.com:shamogulang/git-learn.git
 ! [rejected]        master -> master (fetch first)
error: failed to push some refs to 'git@github.com:shamogulang/git-learn.git'
hint: Updates were rejected because the remote contains work that you do
hint: not have locally. This is usually caused by another repository pushing
hint: to the same ref. You may want to first integrate the remote changes
hint: (e.g., 'git pull ...') before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```

这里的意思是要先拉取下来，在jeffchan2中先进行：git pull ，发现：

![merge](D:\jeffchan\markdown\image\merge.png)

然后查看冲突为文件：

```java
$ git status
On branch master
Your branch and 'origin/master' have diverged,
and have 1 and 1 different commits each, respectively.
  (use "git pull" to merge the remote branch into yours)

You have unmerged paths.
  (fix conflicts and run "git commit")
  (use "git merge --abort" to abort the merge)

Unmerged paths:
  (use "git add <file>..." to mark resolution)
        both modified:   cjh

no changes added to commit (use "git add" and/or "git commit -a")
```

可以看到both modified，就是说多个人同时修改了cjh这个文件。我们打开它：

![confilict](D:\jeffchan\markdown\image\confilict.png)

上面的那部分是jeffchan2那个目录里提交的，下面的那部分是jeffchan1提交的，同时jeffchan1先提交，所以就出现了冲突，对于jeffchan2来说，就是要解决冲突，但是不能影响jeffchan1的功能。

这里假设都要保留的话，就可以将保留的部分留下来

```java
hello,my name is jeffchan，这里是jeffchan2
hello,my name is jeffchan，这里是jeffchan1.
checkout branch
```

然后就git add .  然后git commit -m "xx",  最后提交就可以了。

这个冲突告诫我们，在开发一个需求前，一定要先拉最新的代码减少冲突，虽然也会出现冲突，但是会相对好一点。

 

## 1.3、分支开发

1、git branch  xxx 创建分支，这个一般是在开发某个需求的时候，需要在分支上开发，然后上线前，再合并到主分支。一般这个取名字也是有讲究的，比如你开发一个民生热线需求，组里经常用gdhotline这个词，并且打算20号上线。就可以取名字：gdhotline-20201220。  git branch gdhotline-20201220



2、git checkout gdhotline-20201220 切换分支开发

![gitcheckout](D:\jeffchan\markdown\image\gitcheckout.png)

​      可以看到已经切换分支了。那么此时在本地这个分支进行需求的开发。在切换分支的时候，有要求的：

​      就是本地工作区和暂存区都不能有代码：如果有代码没有提交到分支的话

      ```java
$ git checkout master
error: Your local changes to the following files would be overwritten by checkout:
        cjh
Please commit your changes or stash them before you switch branches.
Aborting
      ```

​      会提示你提交这部分代码，或者是stash 藏起你的代码。

2.1、git stash 藏起工作区或者暂存区的代码

2.2、git stash list 查看你藏起来的代码,藏到了 stash@{0}中

2.3、git stash pop stash@{0}  从stash@{0}中弹出代码当前分支工作区

![gitstash](D:\jeffchan\markdown\image\gitstash.png)

3、后续的还是简单开发模式里面的那些指令,最后不同的是push代码我们要push到远程的分支gdhotline-20201220上，而不是远程master分支上。

git push origin gdhotline-20201220 会直接把代码提交到远程的gdhotline-20201220分支上，如果远程没有这个分支，会主动创建，然后提交。

![gitbranchpush](D:\jeffchan\markdown\image\gitbranchpush.png)

发现确实多了一个分支，同时增加的内容也增加了。



4、好了，现在我们开发了，测试也测试好了，那么就把代码合并到远程master分支上，同时让测试做一次回归测试。首先先： git checkout master分支， 然后在master分支上，执行指令： git merge gdhotline-20201220，然后可以发现代码已经合并到主分支了。此时就可以git commit -m "合并代码"，然后再执行git push origin master进行提交了。



# 2、http协议

拉取![http](D:\jeffchan\markdown\image\http.png)代码的时候，拉取https那部分链接：

```java
$ git clone https://github.com/shamogulang/git-learn.git
Cloning into 'git-learn'...
remote: Enumerating objects: 32, done.
remote: Counting objects: 100% (32/32), done.
remote: Compressing objects: 100% (19/19), done.
remote: Total 32 (delta 3), reused 30 (delta 1), pack-reused 0
Receiving objects: 100% (32/32), 2.45 KiB | 167.00 KiB/s, done.
Resolving deltas: 100% (3/3), done.
```

这里就直接拉取下下来了。此时的各种操作都是和ssh协议的一样的：

这个方式会有点烦人，就是每次都要输入

git config --system --unset credential.helper



第一次会要求你输入账号和密码，后续都不会叫你输入了，因为它被缓存起来了。