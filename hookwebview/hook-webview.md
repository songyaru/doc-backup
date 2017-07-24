## hook sdk 中的 webview 页面过程

### 背景
接入第三方广告的 sdk 广告平台，广告的展示是通过 webview 请求的页面。需求是需要控制这个 webview 的前端页面，如改变里面的内容，自动提交之类的需求。

### 技术点
* [1.利用 ActivityLifecycleCallbacks 找到相应的 activity](#activitylifecyclecallbacks)
* [2.traverseWebView 扫描 activity 的 layout 结构找到相应的 webview](#traversewebview)
* [3.利用动态代理 hook WebViewClient](#hookwebviewclient)  



<span id="activitylifecyclecallbacks"></span>
#### 1.利用 ActivityLifecycleCallbacks 找到相应的 activity
由于广告的 SDK 内部启动 Activity 来显示广告页面，因此利用 ActivityLifecycleCallbacks 这个回调来获取当前窗口的 Activity 实例。

通过在 Application的onCreate()方法 中调用 Application.registerActivityLifecycleCallbacks()方法，并实现ActivityLifecycleCallbacks接口。

``` java

ActivityLifecycleCallbacks activityLifecycleCallbacks = new ActivityLifecycleCallbacks() {
    
    @Override
    public void onActivityResumed(Activity activity) {
        if (activity instanceof PresageActivity) {
            try {
                SplashScreenManager.getInstance().traverseWebView(activity); // 见下面的 traverseWebView 方法说明
            } catch (Exception e) {
               e.printStackTrace();
            }
        }
    }
 }
```

<span id="traversewebview"></span>
### 2.traverseWebView 扫描 activity 的 layout 结构找到相应的 webview

LayoutTraverse 直接上代码，见注释：
``` java

public class LayoutTraverse {
    private int index = 0;

    public interface Processor {
        void process(View view);

        void traverseEnd(ViewGroup root);
    }

    private final Processor processor;

    private LayoutTraverse(Processor processor) {
        this.processor = processor;
    }

    public static LayoutTraverse build(Processor processor) {
        return new LayoutTraverse(processor);
    }

    public void traverse(ViewGroup root) {
        final int childCount = root.getChildCount();
        index++;
        for (int i = 0; i < childCount; ++i) {
            final View child = root.getChildAt(i);
            processor.process(child); // 节点处理的回调

            if (child instanceof ViewGroup) { //发现是 ViewGroup 就递归扫描
                traverse((ViewGroup) child);
            }
        }
        index--;
        if (index == 0) {
            processor.traverseEnd(root); //遍历结束回调
        }
    }

}

```

``` java

public void traverseWebView(final Activity activity) {
    final ViewGroup root = (ViewGroup) activity.getWindow().getDecorView().getRootView();
    final ArrayList<WebView> webViews = new ArrayList<>();
    LayoutTraverse.build(new LayoutTraverse.Processor() {
        @Override
        public void process(View view) {
            if (view instanceof WebView) {
                // 找到了 activity 里面的 webview 
                // 可能有多个 webview , process 回调可以将找到的 webview 先保存起来，等全部扫描完了再做处理
                final WebView wv = (WebView) view;
                webViews.add(wv);
            } 
        }

        @Override
        public void traverseEnd(final ViewGroup root) {
            // 可以根据 webView.getUrl() 来区分，此处 demo 略去业务中的代码 
            hookWebView(webViews.get(0)); // 见下面的 hookWebView 方法说明
        }
    }).traverse(root);
}
```

<span id="hookwebviewclient"></span>
### 3.利用动态代理 hook WebViewClient (WebChromeClient 类似)

``` java

public static void hookWebView(WebView webView) {
    if (BuildConfig.DEBUG && Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
        WebView.setWebContentsDebuggingEnabled(true);
    }
    try {
        Class clsWebView = XXXApplication.getInstance().getClassLoader().loadClass("android.webkit.WebView");
        Method method = clsWebView.getMethod("setWebViewClient", WebViewClient.class);
        method.invoke(webView, new Object[] {new HookWebViewClient()}); // 见下面的 HookWebViewClient 代码说明
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

``` java
// xxx 是广告 sdk 中的一个 extends 了 WebViewClient 的类
// 通过 hookWebView 方法把新扩展的 hook 类动态的插入来满足需求
// 其中 StartJSCode 和 finishJsCode 的 js 代码也可以通过 CMS 系统根据需求动态的下发。
public class HookWebViewClient extends xxx {
    @Override
    public void onPageStarted(WebView view, String url, Bitmap favicon) {
        super.onPageStarted(view, url, favicon);
        view.loadUrl("javascript:(function(){" + StartJSCode + "})();"); // 页面打开时想做的事情
        
    }

    @Override
    public void onPageFinished(WebView view, String url) {
        super.onPageFinished(view, url);
        view.loadUrl("javascript:(function(){" + finishJsCode + "})();"); // 页面加载完成后想做的事情
    }
	
}

```



