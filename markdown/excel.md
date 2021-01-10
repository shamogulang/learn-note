<center><h1>excel</h1></center>

### 1、excel的基本操作

首先本次学习是通过pandas这个模块来操作excel文件。在使用之前，要先安装好pandas。我这里使用的python的版本是3.8.5，所以pip是自带的。进入到pip的指令目录，直接pip install pandas即可。然后就可以在python的文件中导入pandas模块了。

直接下载会有点慢：可以使用镜像下载(这里使用清华的镜像)：

> pip install -i https://pypi.tuna.tsinghua.edu.cn/simple   pandas



#### 1.1、 创建一个简单的excel文件    

```python
import os
import pandas as pd 

# 需要导入到excel文件去的内容
content = {"ID":[1,2,3], "name":['caraliu','jeffchan','mama']}

df = pd.DataFrame(content) # 定义一个excel文件

df = df.set_index('ID') # index的修改

# 将excel文件导出
df.to_excel(os.getcwd() + "/out.xlsx")  # os.getcwd()获取当前文件目录

print("done !!!")
```

我这里第一次执行的时候，发现报错说openpyxl模块找不到，那么就

> pip install -i https://pypi.tuna.tsinghua.edu.cn/simple openpyxl

继续执行，得到文件out.xlsx，打开得到一下内容

![1](D:\jeffchan\markdown\excel-img\1.png)

里面的0，1，2那一列是pandas模块创建excel的时候，主动加入的索引列

如果想要去掉，那么直接显示指定索引的列即可 df.set_index("ID")  这里好像不起作用



#### 1.2、读取excel内容

```python
import os
import pandas as pd 

# content = {'id': [1,2,3], 'name': ['caraliu','jeffchan','mama']}

# df = pd.DataFrame(content)

# df = pd.read_excel(os.getcwd() + "/out.xlsx")

df = pd.read_excel(os.getcwd() + "/out.xlsx", header=None)
df.columns = ['MYID',"MYNAME"]
# 获取多少行，多少列
print(df.shape)
# 获取所有的列名信息
print(df.columns)
df = df.set_index("MYID")
print(df)

# 打印前面3行数据
# print(df.head(3))
# 打印后面3行数据
# print(df.tail(3))
df.to_excel(os.getcwd()+"/out1.xlsx")
# df.to_excel(os.getcwd() + "/out.xlsx")
print("done !!!")
```





