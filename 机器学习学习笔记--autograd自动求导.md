用途:

计算梯度,用于梯度下降中

    `import mxnet.ndarray as nd`
	`import mxnet.autograd as ag`

# 为变量附上梯度 #

如:对函数 f=2*x² 求关于 x 的导数。我们先创建变量x，并赋初值。

	`x=nd.array([[1,2],[3,4])`
-->
	
存放x的导数,attach_grad()方法

	`x.attach_grad()`
-->


定义f 使用record函数

	with ag.record():
    	y = x * 2
    	z = y * x

-->

使用backward()进行求导。如果z不是一个标量（只有大小没有方向），那么z.backward（）等价于nd.sum(z).backward()

	z.backward()

-->

结果

	x.grad

总结:
1,输入.attach_grad 申请缓存区
2,ag.record  记录函数操作
3,f.backward() 求导
4,a.grad 获取结果

# NDArray 部分api讲解 #
## sum ##
mxnet.ndarray.sum(data=None, axis=_Null, keepdims=_Null, exclude=_Null, out=None, name=None, **kwargs)

求总和

Example:

	data = [[[1,2],[2,3],[1,3]],
        [[1,4],[4,3],[5,2]],
        [[7,1],[7,2],[7,3]]]

	sum(data, axis=1)
	[[  4.   8.]
 	[ 10.   9.]
 	[ 21.   6.]]

	sum(data, axis=[1,2])
	[ 12.  19.  27.]

## norm ##

norm则表示范数，首先需要注意的是范数是对向量（或者矩阵）的度量，是一个标量（scalar）：

[http://blog.csdn.net/lanchunhui/article/details/51004387](http://blog.csdn.net/lanchunhui/article/details/51004387)

## asscalar()  ##
返回这个数组的标量

# 对控制流求导 #

