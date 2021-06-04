# OkHttp系列(1)——使用

参考：

[Okhttp使用详解](https://blog.csdn.net/iispring/article/details/51661195)

## 四大对象

### OkHttpClient

OkHttpClient表示了HTTP请求的客户端类，在绝大多数的App中，我们只应该执行一次new OkHttpClient()，将其作为**全局的实例**进行保存，从而在App的各处都只使用这一个实例对象，这样所有的HTTP请求都可以共用Response缓存、共用线程池以及共用连接池。

```kotlin
private val client = OkHttpClient.Builder()
        .connectTimeout(10, TimeUnit.SECONDS)//超时
        .build()
```

### Request

Request类封装了请求报文信息：请求的**Url地址**、请求的**方法**（如GET、POST等）、各种**请求头**（如Content-Type、Cookie）以及可选的**请求体**（POST请求需要）。一般通过内部类`Request.Builder`的链式调用生成Request对象。

```kotlin
val request = Request.Builder()
.url("https://www.baidu.com")//url
.addHeader()//请求头(add可以重复添加相同key的header)
.header()
.post(requestBody)//请求体（POST请求需要设置）
.build()

//请求头：如application/json格式
val requestBody =
"""{"content": "🚀","phone": "13000000000","username": "2"}""".toRequestBody("application/json".toMediaType())
```

### Call

Call代表了一个实际的HTTP请求，它是连接Request和Response的桥梁，通过Request对象的`newCall()`方法可以得到一个Call对象。Call对象既支持同步获取数据，也可以异步获取数据。

```kotlin
val call = client.newCall(retquest)

call.execute()//发起同步请求
call.enqueue(CallBack)//发起异步请求
```

### Response

Response类封装了响应报文信息：**状态码**（200、404等）、**响应头**（Content-Type、Server等）以及可选的**响应体**。

```kotlin
call.enqueue(object : Callback {
      override fun onFailure(call: Call, e: IOException) {
      }

      override fun onResponse(call: Call, response: Response) {
          if(response.code==200){
              response.headers//响应头
              response.body//响应体
              
              //1.将响应体全部转换为ByteArray返回（注意OOM）
              response.body().bytes()
              //2.将响应体全部转换为ByteString返回（注意OOM）
              response.body().byteString()
              //3.将响应体全部转换为String返回（注意OOM）
              response.body().string()
              
              //3.返回响应体的InputStream
              response.body().byteStream()
              //4.返回响应体的Reader
              response.body().charStream()
          }
      }
})
```

## GET

### 请求文本

```kotlin
val request = Request.Builder().url("https://www.baidu.com").build()
val call = client.newCall(request)

call.enqueue(object : Callback {
    override fun onFailure(call: Call, e: IOException) {
    }

    override fun onResponse(call: Call, response: Response) {
        val code=response.code//200
        val msg=response.message//OK
        response.headers.forEach {
            CmdUtil.v(it.toString())//headers
        }
        CmdUtil.i(response.body?.string())//body:字符串
    }
})
```

### 请求图片

```kotlin
val request = Request.Builder()
    .url("https://cdn.pixabay.com/photo/2021/05/11/05/57/men-6245003_960_720.jpg")
    .build()
val call = client.newCall(request)

thread {
    val response = call.execute()
    if (response.code == 200) {
        //body：输入流
        val `is` = response.body?.byteStream()
        val bitmap = BitmapFactory.decodeStream(`is`)//这里简写了，还需要根据imageView尺寸进行采样压缩
        (requireContext() as Activity).runOnUiThread {
            get_pic.setImageBitmap(bitmap)
        }
    }
}
```

### 小结

GET请求不需要构建请求体，拿到Response后从body中取数据。

## POST

### 提交JSON

```kotlin
//1.构建出一个RequestBody，MediaType = "application/json"
val requestBody =
    """{"content": "🚀","phone": "13000000000","username": "2"}""".trimIndent()
        .toRequestBody("application/json".toMediaType())
val request = Request.Builder()
    .url("http://xxxxxxx")
    //2.提交RequestBody
    .post(requestBody)
    .build()

val call = client.newCall(request)
call.enqueue(object : Callback {
    override fun onFailure(call: Call, e: IOException) {
    }

    override fun onResponse(call: Call, response: Response) {
        if (response.code == 200) {
        }
    }
})
```

### 提交流

```kotlin
//1.bitmap
val bitmap = (get_pic.drawable as BitmapDrawable).bitmap

//2.自定义构建一个RequestBody，MediaType = "image/jpeg"
val imaregeBody = object : RequestBody() {
    override fun contentType() = "image/jpeg".toMediaType()

    override fun writeTo(sink: BufferedSink) {
        val os = sink.outputStream()
        //3.将bitmap怼向输出流
        bitmap.compress(Bitmap.CompressFormat.JPEG, 50, os)
    }
}
//4.构建MultipartBody，MediaType = "multipart/form-data"
val requestBody = MultipartBody.Builder()
    //5.将RequestBody添加到MultipartBody
    .addFormDataPart("file", "TestPic.jpg", imageBody)
    .setType(MultipartBody.FORM)//注意 type = "multipart/form-data"
    .build()
val request = Request.Builder()
    .url("http://xxxx")
    //6.提交MultipartBody
    .post(requestBody)
    .build()
val call = client.newCall(request)
call.enqueue(object : Callback {
    override fun onFailure(call: Call, e: IOException) {
    }

    override fun onResponse(call: Call, response: Response) {
        if (response.code == 200) {
        }
    }
})
```

### 提交文件

```kotlin
val bitmap = (get_pic.drawable as BitmapDrawable).bitmap

val picPath = requireContext().externalCacheDir?.path + File.separator + "haha.jpg"
val picFile = File(picPath)
if (!picFile.exists()) {
    picFile.createNewFile()
}
bitmap.compress(Bitmap.CompressFormat.JPEG, 50, picFile.outputStream())

//1.以File构建一个RequestBody，MediaType = "image/jpeg"
val imageBody = picFile.asRequestBody("image/jpeg".toMediaType())

val requestBody = MultipartBody.Builder()
    .addFormDataPart("file", "TestPicFile.jpg", imageBody)
    .setType(MultipartBody.FORM)
    .build()
val request = Request.Builder()
    .url("http://xxxx")
    .post(requestBody)
    .build()
val call = client.newCall(request)
call.enqueue(object : Callback {
    override fun onFailure(call: Call, e: IOException) {
    }

    override fun onResponse(call: Call, response: Response) {
        if (response.code == 200) {
            val res = response.body.toString()
            CmdUtil.v(res)
            CmdUtil.i("请求:" + response.request.toString())
        }
    }
})
```

### 小结

1. post请求需要额外提交**请求体**(RequestBody)。该请求体必须指定MediaType（MIME）
2. 如果提交文件，则需要将contentType设置为"multipart/form-data"，对应okhttp的MultipartBody。之后组合拼接RequestBody最终提交服务端。

