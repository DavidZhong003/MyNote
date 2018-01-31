处理数据:数据的读取,数据在内存中如何处理


NDArray 数据存储/变化工具,类同Numpy,支持CPU/GPU 异步计算,自动求导.

# 创建矩阵(数组) #

`form mxnet import ndarray as nd``

- 创建空矩阵:
    `nd.zeros((3,4))`

输出:	

[[ 0.  0.  0.  0.]
 [ 0.  0.  0.  0.]
 [ 0.  0.  0.  0.]]
<NDArray 3x4 @cpu(0)>


- 创建初始值为1的数组
	`nd.ones((3,4)`
	
- 从数组构造
	`nd.array([[1,2],[2,3]])`

- 创建随机数组
	`nd.random_normal(0,1,shape=(3,4))`
- 数组形状
	`y.shape`
- 数组大小
	`y.size`

# 操作符 #

支持数学操作符
- `x+y`
- `x*y`
- `nd.exp(y)`指数
- 矩阵转置`y.T`
# 广播 #
当二元操作符两边的形状不一样时候.系统会复制到同一个形状.
# 与NumPy 的转换#
    `import numpy as np
	x = np.ones((2,3))
	y = nd.array(x)  # numpy -> mxnet
	z = y.asnumpy()  # mxnet -> numpy
	print([z, y])`

# 替换操作 #

目的节省内存空间
可以把结果通过[:]写到之前的开辟好的数组空间
如:    
    `z = nd.zeros_like(x)
	before = id(z)
	z[:] = x + y
	id(z) == before`
# 截取 #
	`
	x = nd.arange(0,9).reshape((3,3))
	print('x: ', x)
	x[1:3]`
    
直接写入位置
	`x[1,2]=9.0`
多维截取
	`
	x = nd.arange(0,9).reshape((3,3))
	print('x: ', x)
	x[1:2,1:3]`
多维写入
	`x[1:2,1:3] = 9.0`


参考:[https://mxnet.incubator.apache.org/api/python/ndarray.html](https://mxnet.incubator.apache.org/api/python/ndarray.html "NDArray Api")