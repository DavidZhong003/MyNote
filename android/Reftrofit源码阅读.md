Retrofit是现在最火的开源网络请求库,在平常使用中我们会
1.通过Retrofit.Builder获取Retrofit对象。

		Retrofit.Builder().baseUrl(BuildConfig.HOST)
                .client(OKHTTP_CLIENT_NO_TOKEN)
                .addConverterFactory(GsonConverterFactory.create())
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .build()
2.通过Retrofit 对象的create方法，获取定义好的接口对象（代理对象）

		retrofit.create(ApiService::class.java)

3.调用接口中的某个方法获取相应的对象


# Retrofit 对象构建 #

Retrofit和Okhttp一样，通过build模式构建对象。来看下我们上面第一步创建Retrofit对象，传入url，Okhttpclient，gson转换器，rx2调用适配器。这里主要是build方法。

