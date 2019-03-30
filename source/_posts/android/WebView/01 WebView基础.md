---
title: WebView基础
tags:
  - Android控件
  - WebView
categories:
  - Android
  - WebView
abbrlink: 2109
date: 2018-02-25 21:16:17
---


### 1.WebView最简单的使用方法
- 布局
```xml
<WebView
    android:id="@+id/webView"
    android:layout_width="match_parent"
    android:layout_height="match_parent"/>
```


- 在activity中最简单的使用
```java
webview = (WebView) findViewById(R.id.webView);
//加载web资源
webview.loadUrl("http://www.baidu.com/");
//加载本地资源
webView.loadUrl("file:///android_asset/example.html");       
这个时候发现一个问题，启动应用后，自动的打开了系统内置的浏览器，解决这个问题需要为webview设置 WebViewClient，并重写方法：
webview.setWebViewClient(new WebViewClient(){
            @Override
            public boolean shouldOverrideUrlLoading(WebView view, String url) {
                view.loadUrl(url);
                return true;//true 调用Webview,false调用第三方。
            }
});
```

### 2.WebView的常用方法
#### 2.1 WebView回退等功能
```java
webView.canGoBack();//判断是否有可以回退，可以返回true
webView.goBack();//返回上一层级
webView.canGoForward();//判断是否可以前进，可以返回true
webView.goForward();//进入上一层级
webView.reload();//刷新

@Override
public boolean onKeyDown(int keyCode, KeyEvent event) {//改写物理按键——返回的逻辑 这个很重要
    if(keyCode==KeyEvent.KEYCODE_BACK) {
        if(wv.canGoBack()) {
            wv.goBack();
            return true;
        } 
    }
    return super.onKeyDown(keyCode, event);
}
```


#### 2.2 WebView的状态
```java
//激活WebView为活跃状态，能正常执行网页的响应
webView.onResume() ；

//当页面被失去焦点被切换到后台不可见状态，需要执行onPause
//通过onPause动作通知内核暂停所有的动作，比如DOM的解析、plugin的执行、JavaScript执行。
webView.onPause()；

//当应用程序(存在webview)被切换到后台时，这个方法不仅仅针对当前的webview而是全局的全应用程序的webview//它会暂停所有webview的layout，parsing，javascripttimer。降低CPU功耗。
webView.pauseTimers()
恢复pauseTimers状态
webView.resumeTimers()；

销毁Webview
/**
在关闭了Activity时，如果Webview的音乐或视频，还在播放。就必须销毁Webview
但是注意：webview调用destory时,webview仍绑定在Activity上,这是由于自定义webview构建时传入了该Activity的context对象,因此需要先从父容器中移除webview,然后再销毁webview:
**/
rootLayout.removeView(webView);
webView.destroy();
```


#### 2.3 清除缓存数据
```java
//清除网页访问留下的缓存
//由于内核缓存是全局的因此这个方法不仅仅针对webview而是针对整个应用程序.
Webview.clearCache(true);
//清除当前webview访问的历史记录,只会webview访问历史记录里的所有记录除了当前访问记录
Webview.clearHistory()；
//这个api仅仅清除自动完成填充的表单数据，并不会清除WebView存储到本地的数据
Webview.clearFormData()；
```


### 3.WebView常用类的介绍
#### 3.1 WebSettings类
- 主要的作用对WebView进行配置和管理
```java
//声明WebSettings子类
WebSettings webSettings = webView.getSettings();
//注意：这个很重要   如果访问的页面中要与Javascript交互，则webview必须设置支持Javascript
webSettings.setJavaScriptEnabled(true); 
//支持插件
webSettings.setPluginsEnabled(true);
//设置自适应屏幕，两者合用
webSettings.setUseWideViewPort(true);           //将图片调整到适合webview的大小
webSettings.setLoadWithOverviewMode(true);      //缩放至屏幕的大小
//缩放操作
webSettings.setSupportZoom(true);               //支持缩放，默认为true。是下面那个的前提。
webSettings.setBuiltInZoomControls(true);       //设置内置的缩放控件。若为false，则该WebView不可缩放
webSettings.setDisplayZoomControls(false);      //隐藏原生的缩放控件
//其他细节操作
webSettings.setCacheMode(WebSettings.LOAD_CACHE_ELSE_NETWORK); //关闭webview中缓存
webSettings.setAllowFileAccess(true);           //设置可以访问文件
webSettings.setJavaScriptCanOpenWindowsAutomatically(true); //支持通过JS打开新窗口
webSettings.setLoadsImagesAutomatically(true);  //支持自动加载图片
webSettings.setDefaultTextEncodingName("utf-8");//设置编码格式

/**
设置WebView缓存
当加载 html 页面时，WebView会在/data/data/包名目录下生成 database 与 cache 两个文件夹
请求的 URL记录保存在 WebViewCache.db，而 URL的内容是保存在 WebViewCache 文件夹下
是否启用缓存：
**/
//优先使用缓存:
WebView.getSettings().setCacheMode(WebSettings.LOAD_CACHE_ELSE_NETWORK);
//缓存模式如下：
//LOAD_CACHE_ONLY: 不使用网络，只读取本地缓存数据
//LOAD_DEFAULT: （默认）根据cache-control决定是否从网络上取数据。
//LOAD_NO_CACHE: 不使用缓存，只从网络获取数据.
//LOAD_CACHE_ELSE_NETWORK，只要本地有，无论是否过期，或者no-cache，都使用缓存中的数据。

//不使用缓存:
WebView.getSettings().setCacheMode(WebSettings.LOAD_NO_CACHE);
webSettings.setDomStorageEnabled(true); // 开启 DOM storage API 功能
webSettings.setDatabaseEnabled(true);  //开启 database storage API 功能
webSettings.setAppCacheEnabled(true);//开启 Application Caches 功能
String cacheDirPath = getFilesDir().getAbsolutePath() + APP_CACAHE_DIRNAME;
webSettings.setAppCachePath(cacheDirPath); //设置  Application Caches 缓存目录
```


#### 3.2 WebViewClient类
- 主要的作用是处理各种通知和请求事件
- shouldOverrideUrlLoading()方法
```java
shouldOverrideUrlLoading()方法，使得打开网页时不调用系统浏览器， 而是在本WebView中显示
webView.setWebViewClient(new WebViewClient(){
    @Override
    public boolean shouldOverrideUrlLoading(WebView view, String url) {
        view.loadUrl(url);
        return true;
    }
});
```

- onPageStarted()
```java
//作用：开始载入页面调用的，我们可以设定一个loading的页面，告诉用户程序在等待网络响应。
webView.setWebViewClient(new WebViewClient(){
    @Override
    public void  onPageStarted(WebView view, String url, Bitmap favicon) {
        //开始加载网页
    }
});
```

- onPageFinished()
```java
//作用：在页面加载结束时调用。我们可以关闭loading 条，切换程序动作。
webView.setWebViewClient(new WebViewClient(){
    @Override
    public void onPageFinished(WebView view, String url) {
        //网页加载完成
    }
});
```


- onLoadResource()
```java
//作用：在加载页面资源时会调用，每一个资源（比如图片）的加载都会调用一次。
webView.setWebViewClient(new WebViewClient(){
    @Override
    public boolean onLoadResource(WebView view, String url) {
        //设定加载资源的操作
    }
});
```


- onReceivedError() //加载出错的时候
```java
作用：加载页面的服务器出现错误时（如404）调用。
App里面使用webview控件的时候遇到了诸如404这类的错误的时候，若也显示浏览器里面的那种错误提示页面就显得很丑陋了，那么这个时候我们的app就需要加载一个本地的错误提示页面，即webview如何加载一个本地的页面

//步骤1：写一个html文件（error_handle.html），用于出错时展示给用户看的提示页面
//步骤2：将该html文件放置到代码根目录的assets文件夹下
//步骤3：复写WebViewClient的onRecievedError方法
//该方法传回了错误码，根据错误类型可以进行不同的错误分类处理
webView.setWebViewClient(new WebViewClient(){
    @Override
    public void onReceivedError(WebView view, int errorCode, String description, String failingUrl){
        switch(errorCode){
            case HttpStatus.SC_NOT_FOUND:
                view.loadUrl("file:///android_assets/error_handle.html");
                break;
        }
    }
});
```


- onReceivedSslError()   https处理
```java
作用：处理https请求
webView默认是不处理https请求的，页面显示空白，需要进行如下设置：

webView.setWebViewClient(new WebViewClient() {   
    @Override   
    public void onReceivedSslError(WebView view, SslErrorHandler handler, SslError error) {   
        handler.proceed();    //表示等待证书响应
        // handler.cancel();      //表示挂起连接，为默认方式
        // handler.handleMessage(null);    //可做其他处理
    }   
});   
```

#### 4.3 WebChromeClient类
- 主要的作用是辅助 WebView 处理 Javascript 的对话框,网站图标,网站标题等等
- onProgressChanged()
- onReceivedTitle()
```java
作用：获得网页的加载进度并显示
webview.setWebChromeClient(new WebChromeClient(){
    @Override
    public void onProgressChanged(WebView view, int newProgress) {
        if (newProgress < 100) {
            String progress = newProgress + "%";
            progress.setText(progress);
        }
    }
    /**
    作用：获取Web页中的标题
	每个网页的页面都有一个标题
    **/
     @Override
    public void onReceivedTitle(WebView view, String title) {
        titleview.setText(title)；
    }
});
```




### 5.WebView注意事项
#### 5.1 动态创建WebView使用上下文为全局上下文
```java
LinearLayout.LayoutParams params = new LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT);
mWebView = new WebView(getApplicationContext());
mWebView.setLayoutParams(params);
mLayout.addView(mWebView);
```


#### 5.2 Activity销毁的时候，销毁WebView
```java
//在 Activity 销毁（ WebView ）的时候，先让 WebView 加载null内容，然后移除 WebView，再销毁 WebView，最后置空。
@Override
protected void onDestroy() {
    if (mWebView != null) {
        mWebView.loadDataWithBaseURL(null, "", "text/html", "utf-8", null);
        mWebView.clearHistory();

        ((ViewGroup) mWebView.getParent()).removeView(mWebView);
        mWebView.destroy();
        mWebView = null;
    }
    super.onDestroy();
}
```


#### 5.3 WebView页面中播放了音频,退出Activity后音频仍然在播放，需要在Activity的onDestory()中调用
```java
@Override
protected void onDestroy() {
    if (mWebView != null) {
        mWebView.loadDataWithBaseURL(null, "", "text/html", "utf-8", null);
        mWebView.clearHistory();

        ViewGroup parent = (ViewGroup) mWebView.getParent();
        if (parent != null) {
            parent.removeView(mWebView);
        }
        mWebView.removeAllViews();
        mWebView.destroy();
        mWebView = null;
    }
    super.onDestroy();
}
```


### 6.WebView滑动监听
#### 6.1 WebView滑动监听事件

```java
public class NewWebView extends WebView{
    private OnScrollChangeListener mOnScrollChangeListener;
    public NewWebView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }
    /**
    以屏幕的左上角为（0,0）点，l表示滑动后的x值，oldl表示滑动前的x位置,t表示滑动后的y值，oldt表示滑动前的y位置。
    **/
    @Override 
    protected void onScrollChanged(int l, int t, int oldl, int oldt) {
        super.onScrollChanged(l, t, oldl, oldt);
/**
getScrollY()方法返回的是当前可见区域的顶端距整个页面顶端的距离,也就是当前内容滚动的距离。getHeight()或者getBottom()方法都返回当前WebView 这个容器的高度,getContentHeight 返回的是整个html 的高度,但并不等同于当前整个页面的高度,因为WebView 有缩放功能, 所以当前整个页面的高度实际上应该是原始html 的高度再乘上缩放比例getScale(). 因此,更正后的结果,准确的判断方法应该是：
**/        
        // webview的高度
        float webcontent = getContentHeight() * getScale();
        // 当前webview的高度
        float webnow = getHeight() + getScrollY();
        if (Math.abs(webcontent - webnow) < 1) {
            //处于底端 
            mOnScrollChangeListener.onPageEnd(l, t, oldl, oldt);
        } else if (getScrollY() == 0) {
            //处于顶端
            mOnScrollChangeListener.onPageTop(l, t, oldl, oldt);
        } else { 
            mOnScrollChangeListener.onScrollChanged(l, t, oldl, oldt); 
        } 
    }
    
    public void setOnScrollChangeListener(OnScrollChangeListener listener) {
        this.mOnScrollChangeListener = listener; 
    }
    
    public interface OnScrollChangeListener {
        
        public void onPageEnd(int l, int t, int oldl, int oldt);
        
        public void onPageTop(int l, int t, int oldl, int oldt);
        
        public void onScrollChanged(int l, int t, int oldl, int oldt); 
    
    }
    
}
```

### 7.WebView其他使用
#### 7.1 如何调试WebView加载的页面？
- 在Android 4.4版本以后，可以使用Chrome开发者工具调试WebView内容5。调试需要在代码里设置打开调试开关。
```java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
    WebView.setWebContentsDebuggingEnabled(true);
}
```

- 开启后，使用USB连接电脑，加载URL时，打开Chrome开发者工具，在浏览器输入
```java
chrome://inspect/#devices
```
- 可以看到当前正在浏览的页面，点击inspect即可看到WebView加载的内容。


#### 7.2 WebView中部分Settings方法介绍
```java
setSupportZoom(boolean support)                          //是否支持缩放，配合方法setBuiltInZoomControls使用，默认true
setMediaPlaybackRequiresUserGesture(boolean require)     //是否需要用户手势来播放Media，默认true
setDisplayZoomControls(boolean enabled)                  //是否显示窗口悬浮的缩放控制，默认true
setAllowFileAccess(boolean allow)                        //是否允许访问WebView内部文件，默认true
setAllowContentAccess(boolean allow)                     //是否允许获取WebView的内容URL 
setLoadWithOverviewMode(boolean overview)                //是否启动概述模式浏览界面，当页面宽度超过WebView显示宽度时，缩小页面适应WebView。默认false
setSaveFormData(boolean save)                            //是否保存表单数据，默认false
setTextZoom(int textZoom)                                //设置页面文字缩放百分比，默认100%
setUseWideViewPort(boolean use)                          //是否支持ViewPort的meta tag属性
setSupportMultipleWindows(boolean support)               //是否支持多窗口
setLayoutAlgorithm(LayoutAlgorithm l)                    //指定WebView的页面布局显示形式，调用该方法会引起页面重绘。默认LayoutAlgorithm#NARROW_COLUMNS
setStandardFontFamily(String font)                       //设置标准的字体族，默认”sans-serif”。font-family 规定元素的字体系列。
setFixedFontFamily(String font)                          //设置混合字体族。默认”monospace”
setSansSerifFontFamily(String font)                      //设置SansSerif字体族。默认”sans-serif”
setSerifFontFamily(String font)                          //设置SerifFont字体族，默认”sans-serif”
setCursiveFontFamily(String font)                        //设置CursiveFont字体族，默认”cursive”
setFantasyFontFamily(String font)                        //设置FantasyFont字体族，默认”fantasy”
setMinimumFontSize(int size)                             //设置最小字体，默认8. 取值区间[1-72]，超过范围，使用其上限值。
setMinimumLogicalFontSize(int size)                      //设置最小逻辑字体，默认8. 取值区间[1-72]，超过范围，使用其上限值。
setDefaultFontSize(int size)                             //设置默认字体大小，默认16，取值区间[1-72]，超过范围，使用其上限值。
setDefaultFixedFontSize(int size)                        //设置默认填充字体大小，默认16，取值区间[1-72]，超过范围，使用其上限值。
setLoadsImagesAutomatically(boolean flag)                //设置是否加载图片资源
setBlockNetworkImage(boolean flag)                       //是否加载网络图片资源。
setBlockNetworkLoads(boolean flag)                       //设置是否加载网络资源
setJavaScriptEnabled(boolean flag)                       //设置是否允许执行JS
setAllowUniversalAccessFromFileURLs(boolean flag)        //是否允许Js访问任何来源的内容
setAllowFileAccessFromFileURLs(boolean flag)             //是否允许Js访问其他file scheme的URLs。
setGeolocationDatabasePath(String databasePath)          //设置存储定位数据库的位置
setAppCacheEnabled(boolean flag)                         //是否允许Cache，默认false。
setAppCachePath(String appCachePath)                     //设置Cache API缓存路径。
setDatabaseEnabled(boolean flag)                         //是否允许数据库存储。默认false。
setDomStorageEnabled(boolean flag)                       //是否存储页面DOM结构，默认false
setGeolocationEnabled(boolean flag)                      //是否允许定位，默认true。
setJavaScriptCanOpenWindowsAutomatically(boolean flag)   //是否允许JS自动打开窗口。默认false
setDefaultTextEncodingName(String encoding)              //设置页面的编码格式，默认UTF-8
setUserAgentString(String ua)                            //设置WebView代理，默认使用默认值
setNeedInitialFocus(boolean flag)                        //通知WebView是否需要设置一个节点获取焦点当
setCacheMode(int mode)                                   //基于WebView导航的类型使用缓存
setMixedContentMode(int mode)                            //设置加载不安全资源的WebView加载行为
```




#### 7.3 加载证书错误
- webview加载一些别人的url时候，有时候会发生证书认证错误的情况，这时候我们希望能够正常的呈现页面给用户，我们需要忽略证书错误，需要调用WebViewClient类的onReceivedSslError方法，调用handler.proceed()来忽略该证书错误。
