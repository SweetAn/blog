---
title: WebView和Js交互方式
tags:
  - Android控件
  - WebView
categories:
  - Android
  - WebView
abbrlink: 27514
date: 2018-02-26 11:16:17
---

### 1.WebView和Js交互方式
#### 1.1 Android去调用JS的代码方式
- 对于android调用JS代码的方法有2种：
    - 第一种方式已经不推荐使用了，第二种方式不仅更方便，也提供了结果的回调，但仅支持API 19以后的系统。
    ```java
    WebView.loadUrl("javascript:" + javascript);
    WebView.evaluateJavascript(javascript, callbacck);
    ```

#### 1.2 JS去调用Android的代码方式
- 对于JS调用Android代码的方法有3种：
    - 第一种：通过WebView的addJavascriptInterface（）进行对象映射

    - 第二种：通过 WebViewClient 的shouldOverrideUrlLoading ()方法回调拦截 url

    - 第三种：通过 WebChromeClient 的onJsAlert()、onJsConfirm()、onJsPrompt（）方法回调拦截JS对话框alert()、confirm()、prompt()消息

      

##### 第一种 addJavascriptInterface

    根据安卓官方的api说明，addJavaScriptInterface方法存在安全漏洞。因为JS可以通过反射访问注入对象。官方在4.2及以后的系统中修复了该问题，要求注入的远程方法必须使用注解@JavascriptInterface

```java
    //允许运行js代码
mWebView.getSettings().setJavaScriptEnabled(true); 

class JsObject {  
   @JavascriptInterface  
   public String toString() { return "injectedObject"; }  
}  
webView.addJavascriptInterface(new JsObject(), "injectedObject");  

//JS调用注入对象示例【java代码】
webView.loadUrl("javascript:alert(injectedObject.toString())");  
```
​    

#####  第二种  shouldOverrideUrlLoading回调拦截

    WebViewClient.shouldOverrideUrlLoading()

``` java
                    @Override
                    public boolean shouldOverrideUrlLoading(WebView view, String url) {
                        if ( TextUtils.isEmpty(url) ) {
                            return false;
                        }
                        // 通用url跳转规则
                        if ( TbUrlBridge.overrideUrl(TbWebViewActivity.this, url) ) {
                            return true;
                        } else {
                            // 非通用url规则，则用当前webview直接打开
                            try {
                                if(url.startsWith("weixin://") //微信
                                        || url.startsWith("alipays://") //支付宝
                                        || url.startsWith("mailto://") //邮件
                                        || url.startsWith("tel://")//电话
                                        || url.startsWith("baidumap://")//大众点评
                                    //其他自定义的scheme
                                        ) {
    //                                Intent intent = new Intent(Intent.ACTION_VIEW, Uri.parse(url));
    //                                startActivity(intent);
                                    return true;
                                }
                            } catch (Exception e) { //防止crash (如果手机上没有安装处理某个scheme开头的url的APP, 会导致crash)
                                return true;//没有安装该app时，返回true，表示拦截自定义链接，但不跳转，避免弹出上面的错误页面
                            }
                            mUrl = url;
                            refresh();
                        }
                        return super.shouldOverrideUrlLoading(view, url);
                    }
    
```



##### 第三种  onJsAlert，onJsConfirm，onJsPrompt

```java

    /**
     * Tell the client to display a javascript alert dialog.  If the client
     * returns true, WebView will assume that the client will handle the
     * dialog.  If the client returns false, it will continue execution.
     * @param view The WebView that initiated the callback.
     * @param url The url of the page requesting the dialog.
     * @param message Message to be displayed in the window.
     * @param result A JsResult to confirm that the user hit enter.
     * @return boolean Whether the client will handle the alert dialog.
     */
    public boolean onJsAlert(WebView view, String url, String message,
            JsResult result) {
        return false;
    }


    /**
     * Tell the client to display a confirm dialog to the user. If the client
     * returns true, WebView will assume that the client will handle the
     * confirm dialog and call the appropriate JsResult method. If the
     * client returns false, a default value of false will be returned to
     * javascript. The default behavior is to return false.
     * @param view The WebView that initiated the callback.
     * @param url The url of the page requesting the dialog.
     * @param message Message to be displayed in the window.
     * @param result A JsResult used to send the user's response to
     *               javascript.
     * @return boolean Whether the client will handle the confirm dialog.
     */
    public boolean onJsConfirm(WebView view, String url, String message,
            JsResult result) {
        return false;
    }


    /**
     * Tell the client to display a prompt dialog to the user. If the client
     * returns true, WebView will assume that the client will handle the
     * prompt dialog and call the appropriate JsPromptResult method. If the
     * client returns false, a default value of false will be returned to to
     * javascript. The default behavior is to return false.
     * @param view The WebView that initiated the callback.
     * @param url The url of the page requesting the dialog.
     * @param message Message to be displayed in the window.
     * @param defaultValue The default value displayed in the prompt dialog.
     * @param result A JsPromptResult used to send the user's reponse to
     *               javascript.
     * @return boolean Whether the client will handle the prompt dialog.
     */
    public boolean onJsPrompt(WebView view, String url, String message,
            String defaultValue, JsPromptResult result) {
        return false;
    }

//控制台log
    /**
     * Report a JavaScript console message to the host application. The ChromeClient
     * should override this to process the log message as they see fit.
     * @param consoleMessage Object containing details of the console message.
     * @return true if the message is handled by the client.
     */
WebChromeClient.onConsoleMessage()
```


https://www.jianshu.com/p/7d820c00642a



### 2.Android调用JS脚本

- 对于Android调用JS代码的方法有2种：
    - 通过WebView的loadUrl（）
    - 通过WebView的evaluateJavascript（）


#### 2.1 通过WebView的evaluateJavascript()
- 优点：该方法比第一种方法效率更高、使用更简洁。
- 1. 因为该方法的执行不会使页面刷新，而第一种方法（loadUrl ）的执行则会。
- 2. Android 4.4 后才可使用
```java
//只需要将第一种方法的loadUrl()换成下面该方法即可
    /**
     * Asynchronously evaluates JavaScript in the context of the currently displayed page.
     * If non-null, |resultCallback| will be invoked with any result returned from that
     * execution. This method must be called on the UI thread and the callback will
     * be made on the UI thread.
     * <p>
     * Compatibility note. Applications targeting {@link android.os.Build.VERSION_CODES#N} or
     * later, JavaScript state from an empty WebView is no longer persisted across navigations like
     * {@link #loadUrl(String)}. For example, global variables and functions defined before calling
     * {@link #loadUrl(String)} will not exist in the loaded page. Applications should use
     * {@link #addJavascriptInterface} instead to persist JavaScript objects across navigations.
     *
     * @param script the JavaScript to execute.
     * @param resultCallback A callback to be invoked when the script execution
     *                       completes with the result of the execution (if any).
     *                       May be null if no notification of the result is required.
     */
mWebView.evaluateJavascript（"javascript:callJS()", new ValueCallback<String>() {
    @Override
    public void onReceiveValue(String value) {
        //此处为 js 返回的结果
    }
});
```

#### 2.2 通过WebView的loadUrl()
- 直接Webview调用loadUrl方法，里面是JS的方法名，并可以传入参数，javascript：xxx()方法名需要和JS方法名相同
- contentWebView.loadUrl("javascript:javacalljs()");
- HTML代码

``` html
  function alertFun()
  {
   alert("Alert警告对话框!");
  }
```
``` java
 webView.loadUrl("javascript:alertFun()");
```


#### 2.3 使用建议
```java
两种方法混合使用，即Android 4.4以下使用方法1，Android 4.4以上方法2
//Android版本变量
final int version = Build.VERSION.SDK_INT;
//因为该方法在 Android 4.4 版本才可使用，所以使用时需进行版本判断
if (version < 18) {
    mWebView.loadUrl("javascript:callJS()");
} else {
    mWebView.evaluateJavascript（"javascript:callJS()", new ValueCallback<String>() {
        @Override
        public void onReceiveValue(String value) {
            //此处为 js 返回的结果
        }
    });
}
```



### 3.什么时候注入js探索
- **3.1 onPageFinished()或者onPageStarted()方法中注入js代码**
- 做过WebView开发，并且需要和js交互，大部分都会认为js在WebViewClient.onPageFinished()方法中注入最合适，此时dom树已经构建完成，页面已经完全展现出来。但如果做过页面加载速度的测试，会发现WebViewClient.onPageFinished()方法通常需要等待很久才会回调（首次加载通常超过3s），这是因为WebView需要加载完一个网页里主文档和所有的资源才会回调这个方法。
- 能不能在WebViewClient.onPageStarted()中注入呢？答案是不确定。经过测试，有些机型可以，有些机型不行。在WebViewClient.onPageStarted()中注入还有一个致命的问题——这个方法可能会回调多次，会造成js代码的多次注入。
- 从7.0开始，WebView加载js方式发生了一些小改变，**官方建议把js注入的时机放在页面开始加载之后**。



- 3.2 WebViewClient.onProgressChanged()方法中注入js代码**
- WebViewClient.onProgressChanged()这个方法在dom树渲染的过程中会回调多次，每次都会告诉我们当前加载的进度。
    - 在这个方法中，可以给WebView自定义进度条，类似微信加载网页时的那种进度条
    - 如果在此方法中注入js代码，则需要避免重复注入，需要增强逻辑。可以定义一个boolean值变量控制注入时机
- 那么有人会问，加载到多少才需要处理js注入逻辑呢？
    - 正是因为这个原因，页面的进度加载到80%的时候，实际上dom树已经渲染得差不多了，表明WebView已经解析了<html>标签，这时候注入一定是成功的。在WebViewClient.onProgressChanged()实现js注入有几个需要注意的地方：
    - 6.2.1 上文提到的多次注入控制，使用了boolean值变量控制
    - 6.2.2 重新加载一个URL之前，需要重置boolean值变量，让重新加载后的页面再次注入js
    - 6.2.3 如果做过本地js，css等缓存，则先判断本地是否存在，若存在则加载本地，否则加载网络js
    - 6.2.4 注入的进度阈值可以自由定制，理论上10%-100%都是合理的，不过建议使用了75%到90%之间可以。





### 7.H5页面点击图片监听图片链接地址
```java
settings.setJavaScriptEnabled(true);
wv_view.addJavascriptInterface(new ImageJs(this),"imageListener");

/**打开图片js通信接口*/
private class ImageJs {
    private final Activity activity;
    public ImageJs(Activity activity) {
        this.activity = activity;
    }
    // 下面的@SuppressLint("JavascriptInterface")最好加上。防止在某些版本中js和java的交互不支持。
    //@SuppressLint("JavascriptInterface")
    @android.webkit.JavascriptInterface
    public void openImage(String img) {
        Log.i("url地址","图片"+ img);
        //跳转页面
    }
}

/**添加图片点击事件的js代码，网上找到，就是这样写，不需要明白*/
private void addImageClickListner() {
    String jsCode="javascript:(function(){" +
            "var imgs=document.getElementsByTagName(\"img\");" +
            "for(var i=0;i<imgs.length;i++){" +
            "imgs[i].onclick=function(){" +
            "window.imageListener.openImage(this.src);" +    //imageListener自定义，openImage要与js通信接口相同
            "}}})()";
    wv_view.loadUrl(jsCode);
}
```
