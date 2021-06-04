# WebView

## 配置

```kotlin
webview.settings.apply {
    //一定要设置该属性
    javaScriptEnabled = true
    //一定要设置该属性
    domStorageEnabled=true
    //允许js弹框
    javaScriptCanOpenWindowsAutomatically=true
    setAppCacheEnabled(true)
    setAppCachePath(this@WebViewAy.cacheDir.absolutePath)
}

webview.webViewClient = WebViewClient()
webview.loadUrl("file:///android_asset/dist/index.html")
```

## Android Call JS

### 1. 通过webview.evaluateJavascript

*Android*

```kotlin
//1.通过webview.evaluateJavascript（函数名）
btn_webview_rotate1.setOnClickListener {
    webview.evaluateJavascript("javascript:callJs1('90deg')"){
        //2.拿到Js方法的返回值
        Toast.makeText(requireContext(), it, Toast.LENGTH_SHORT).show()
    }
}
```

*JavaScript*

```javascript
function callJs1(deg) {
 document.getElementById("box").style.transform="rotateY("+deg+")"
 return "js：调用成功啦！"
}
```

### 2. 通过webview.loadUrl

*Android*

```kotlin
//1.通过webview.loadUrl（函数名）
btn_webview_rotate.setOnClickListener {
    webview.loadUrl("javascript:callJs('90deg')")
}
```

*JS*

```javascript
function callJs(deg) {
 document.getElementById("box").style.transform="rotateZ("+deg+")"
}
```

## JS Call Android

### 1. 通过@JavascriptInterface

Android

```kotlin
class JsInterface (val context:Context){

    //1.声明注解，标识这是Js调用的接口
    @JavascriptInterface
    fun startActivity(str: String) {
        context.startActivity(Intent(context, JsInteractionActivity::class.java).apply {
            putExtra("jsCall",str)
        })
    }
}
```

```kotlin
//2.注册接口，将JsInterface对象以androidInterface的名字映射到JS端
webview.addJavascriptInterface(JsInterface(requireContext()), "androidInterface")
```

*JS*

```javascript
var btnEle = document.getElementById("btn")
btnEle.addEventListener('click', () => {
    if (window.androidInterface) {
        //3.调用androidInterface映射的对象JsInterface
        window.androidInterface.startActivity('我通过addJavascriptInterface调起Android')
    }
})
```

### 2. 通过WebViewClient.shouldOverrideUrlLoading

*Android*

```kotlin
webview.webViewClient = object : WebViewClient() {
    override fun shouldOverrideUrlLoading(
        view: WebView?,
        request: WebResourceRequest?
    ): Boolean {
        val uri = request?.url
        //2.Android端通过解析uri来调用响应的方法
        if ("js" == uri?.scheme) {
            if (uri.authority.equals("webview")) {
                val str = uri.getQueryParameter(uri.queryParameterNames?.elementAt(0))
                JsInterface(requireContext()).startActivity(str?:"")
                return true
            }
        }
        return super.shouldOverrideUrlLoading(view, request)
    }
}
```

*JS*

```javascript
var btnEle = document.getElementById("btn1")
btnEle.addEventListener('click', () => {
    //1.设置uri
    document.location = "js://webview?message=我通过WebViewClient.shouldOverrideUrlLoading调起Android";
})
```

### 3. 通过WebChromeClient.onJsAlert、onJsConfirm等

Android

```kotlin
webview.webChromeClient = object : WebChromeClient() {
    override fun onJsAlert(
        view: WebView?,
        url: String?,
        message: String?,
        result: JsResult?
    ): Boolean {
         //2.Android端通过解析uri来调用响应的方法
        val uri = Uri.parse(message)
        if ("js" == uri?.scheme) {
            if (uri.authority.equals("webview")) {
                val str = uri.getQueryParameter(uri.queryParameterNames?.elementAt(0))
                JsInterface(requireContext()).startActivity(str?:"")
                result?.confirm()//注意要执行确认
                return true
            }
        }
        return super.onJsAlert(view, url, message, result)
    }
}
```

JS

```javascript
var btnEle = document.getElementById("btn2")
btnEle.addEventListener('click', () => {
    //1.alert uri
    alert("js://webview?message=我通过WebChromeClient.onJsAlert调起Android");
})
```

## 总结

Android和Js交互一般都选择上述第一种方法。
