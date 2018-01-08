#管理conda

conda --version 查看当前版本

conda update conda 升级到当前最新版本

#管理环境

##创建环境

conda creat --name hehe python=2 

比如我原来有Python3的环境，现在新创建一个新环境，叫hehe，里面安装python2。

##环境操作

Linux, OS X: source activate hehe
Windows: activate hehe

如果要更换环境，继续用上面的命令，加另外的环境名就好。
退出则使用deactivate hehe或 source deactivate hehe，退回到默认root环境中。

查看电脑中都有哪些环境：conda info --envs或conda env list，带*号的是当前所在的环境。

复制一个环境：conda create --name lala --clone hehe，复制hehe。

删除环境：conda remove --name lala --all，删除lala。

共享环境：把你的环境导出来（包括安装的包们）给别人使用，这样他可以很快搞出一个和你一样的环境，而不用一步一步的安装了。

进入到要共享的环境下activate peppermint或source activate peppermint，执行conda env export > environment.yml。如果当前环境下已经有一个.yml的文件，它会被重写的。
把这个environment.yml文件拷贝给其他人。（默认出现在你安装anaconda的文件夹下）
然后用conda env create -f environment.yml在其他电脑上创建新环境。

#管理Python和package



- 升级Python版本：

conda update python，升级到当前分支的最新版本，如果3.5.2会升级到3.5.3。
conda isntall python=3.6，这样会直接升级到3.6的最新版本。

- 查看已安装的包：conda list（当前环境下的），conda list -name hehe（hehe环境下的）

- 搜索某个特定的包：conda search beautifulsoup4
- 安装包：conda isntall --name envName beautifulsoup4，用--name指定安装在哪个环境下。
  也可以用pip install packageName，但是不能指定环境，也不能升级Python。

- 升级包：conda update conda, conda update python, conda update beautifulsoup4。几乎啥都可以升级。

- 删除包：conda remove --name envName beautifulsoup4



参考:

[https://www.jianshu.com/p/d2e15200ee9b](https://www.jianshu.com/p/d2e15200ee9b)

[http://blog.csdn.net/menc15/article/details/71477949](http://blog.csdn.net/menc15/article/details/71477949)