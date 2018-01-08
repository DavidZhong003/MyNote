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
	定义f