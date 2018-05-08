在Android 中默认使用xml书写布局,这带来了下面几个问题:

	1. 类型不安全
	2. 没有空安全
	3. 迫使每个布局书写差不多相同的代码
	4. 解析xml浪费设备太多的时间和性能
	5. 代码不能复用

非DSL书写布局
	
	val act = this
	val layout = LinearLayout(act)
	layout.orientation = LinearLayout.VERTICAL
	val name = EditText(act)
	val button = Button(act)
	button.text = "Say Hello"
	button.setOnClickListener {
    	Toast.makeText(act, "Hello, ${name.text}!", Toast.LENGTH_SHORT).show()
	}
	layout.addView(name)
	layout.addView(button)
	
anko DSL方式:

	verticalLayout {
    val name = editText()
    button("Say Hello") {
        onClick { toast("Hello, ${name.text}!") }
    	}
	}
	
由上面可以看出,DSL书写布局方便阅读,书写简单,而且更为重要的是没有运行时候的开销