# Kotliln中常用的kt文件

## MutableCollections.kt

对MutableCollection的增删操作，其中还可以通过操作符+、-来增加或移除集合

```kotlin
public inline operator fun <T> MutableCollection<in T>.plusAssign(elements: Iterable<T>) {
    this.addAll(elements)
}
public inline operator fun <T> MutableCollection<in T>.minusAssign(element: T) {
    this.remove(element)
}

//OkHttp代码片段
val interceptors = mutableListOf<Interceptor>()
    interceptors += client.interceptors
    interceptors += RetryAndFollowUpInterceptor(client)
    interceptors += BridgeInterceptor(client.cookieJar)
    interceptors += CacheInterceptor(client.cache)
    interceptors += ConnectInterceptor
```

