
# JNI 是什么 #

JNI , java native interface 的缩写。

作用

1. java 可调用 Native 语言书写的函数，如 C++/C 的函数
2. Native 程序可调用java层函数。

# Jni开发流程 #

1. 编写申明了native方法的java类
2. 源代码(.java)编译成class字节码文件
3. 用javah -jni 命令生成.h 头文件
4. 用本地代码实现.h文件中的函数
5. 把本地代码编译成动态库(windows: dll ; Linux: so , mac : jnilib)
6. 拷贝动态库到程序指定目录下运行java程序

# JVM 查找native方法 #

主要有两种方式进行native方法查找

1. 按照JNI 命名规范

	Java_类全路径_方法名

eg：
	`JNIEXPORT jstring JNICALL Java_com_doive_jnitest_HelloJni_setName(JNIEnv *, jclass, jstring);`

其中 夹在 JNIEXPORT 和 JNICALL 宏中间的 jstring 是返回值类型。
方法参数：
	JNIEnv* 是指向JVM函数表的指针
	
	jclass 调用java中native方法的实例对象或者 class对象，如果是实例方法则是jobject

	jstring,java中对应JNI的数据类型

2. 调用JNI 的RegisterNative 函数,把本地函数注册到JVM中。

[http://wiki.jikexueyuan.com/project/jni-ndk-developer-guide/function2.html](http://wiki.jikexueyuan.com/project/jni-ndk-developer-guide/function2.html)

# CMake #

在AS 2.2 以上使用ndk和CMake去编译c/c++ 代码为本地库。


