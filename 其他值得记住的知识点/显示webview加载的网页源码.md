1. 为webview添加一个js接口(定义一个Handler)
2. 为webview设置WebViewClient，并重写**onPageFinished**(这一步很重要)方法

代码如下：




    mWebView.addJavascriptInterface(new Handler(), "handler");


    class Handler {
    @JavascriptInterface
    public void show(String data) {//获取网页源码
    Logger.d(data);
    }

    mWebView.setWebViewClient(new WebViewClient() {
    @Override
    public void onPageFinished(WebView view, String url) {
    view.loadUrl("javascript:window.handler.show(document.body.innerHTML);");
    super.onPageFinished(view, url);
    }
    }