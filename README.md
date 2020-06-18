
掘金：https://juejin.im/post/5c0e1e2ae51d451d971743a1

![WKWebView的使用](https://upload-images.jianshu.io/upload_images/1708447-eb27b4d6eab2cc75.gif?imageMogr2/auto-orient/strip)

### 前言
>最近项目中的UIWebView被替换为了WKWebView，因此来总结一下WKWebView的使用。 示例Demo：[WKWebView的使用](https://github.com/wslcmk/WKWebView)
本文将从以下几方面介绍WKWebView： 
> * 1、WKWebView涉及的一些类 
> * 2、WKWebView涉及的代理方法 
> * 3、网页内容加载进度条的实现
> * 4、JS和OC的交互
> * 5、本地HTML文件的实现
> * 6、WKWebView+UITableView混排 和 WKWebView离线缓存功能 在 https://github.com/wsl2ls/iOS_Tips


##  一、WKWebView涉及的一些类

* WKWebView：网页的渲染与展示

```
    注意： #import <WebKit/WebKit.h>
     //初始化
       _webView = [[WKWebView alloc] initWithFrame:CGRectMake(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT) configuration:config];
        // UI代理
        _webView.UIDelegate = self;
        // 导航代理
        _webView.navigationDelegate = self;
        // 是否允许手势左滑返回上一级, 类似导航控制的左滑返回
        _webView.allowsBackForwardNavigationGestures = YES;
        //可返回的页面列表, 存储已打开过的网页 
       WKBackForwardList * backForwardList = [_webView backForwardList];

        //        NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:@"http://www.chinadaily.com.cn"]];
        //        [request addValue:[self readCurrentCookieWithDomain:@"http://www.chinadaily.com.cn"] forHTTPHeaderField:@"Cookie"];
        //        [_webView loadRequest:request];
        //页面后退
        [_webView goBack];
        //页面前进
         [_webView goForward];
        //刷新当前页面
        [_webView reload];
        
        NSString *path = [[NSBundle mainBundle] pathForResource:@"JStoOC.html" ofType:nil];
        NSString *htmlString = [[NSString alloc]initWithContentsOfFile:path encoding:NSUTF8StringEncoding error:nil];
      //加载本地html文件
        [_webView loadHTMLString:htmlString baseURL:[NSURL fileURLWithPath:[[NSBundle mainBundle] bundlePath]]];
        

```

* WKWebViewConfiguration：为添加WKWebView配置信息

```
       //创建网页配置对象
        WKWebViewConfiguration *config = [[WKWebViewConfiguration alloc] init];
        
        // 创建设置对象
        WKPreferences *preference = [[WKPreferences alloc]init];
        //最小字体大小 当将javaScriptEnabled属性设置为NO时，可以看到明显的效果
        preference.minimumFontSize = 0;
        //设置是否支持javaScript 默认是支持的
        preference.javaScriptEnabled = YES;
        // 在iOS上默认为NO，表示是否允许不经过用户交互由javaScript自动打开窗口
        preference.javaScriptCanOpenWindowsAutomatically = YES;
        config.preferences = preference;
        
        // 是使用h5的视频播放器在线播放, 还是使用原生播放器全屏播放
        config.allowsInlineMediaPlayback = YES;
        //设置视频是否需要用户手动播放  设置为NO则会允许自动播放
        config.requiresUserActionForMediaPlayback = YES;
        //设置是否允许画中画技术 在特定设备上有效
        config.allowsPictureInPictureMediaPlayback = YES;
        //设置请求的User-Agent信息中应用程序名称 iOS9后可用
        config.applicationNameForUserAgent = @"ChinaDailyForiPad";
         //自定义的WKScriptMessageHandler 是为了解决内存不释放的问题
        WeakWebViewScriptMessageDelegate *weakScriptMessageDelegate = [[WeakWebViewScriptMessageDelegate alloc] initWithDelegate:self];
        //这个类主要用来做native与JavaScript的交互管理
        WKUserContentController * wkUController = [[WKUserContentController alloc] init];
        //注册一个name为jsToOcNoPrams的js方法
        [wkUController addScriptMessageHandler:weakScriptMessageDelegate  name:@"jsToOcNoPrams"];
        [wkUController addScriptMessageHandler:weakScriptMessageDelegate  name:@"jsToOcWithPrams"]; 
       config.userContentController = wkUController;
        
```

* WKUserScript：用于进行JavaScript注入

```
    //以下代码适配文本大小，由UIWebView换为WKWebView后，会发现字体小了很多，这应该是WKWebView与html的兼容问题，解决办法是修改原网页，要么我们手动注入JS
        NSString *jSString = @"var meta = document.createElement('meta'); meta.setAttribute('name', 'viewport'); meta.setAttribute('content', 'width=device-width'); document.getElementsByTagName('head')[0].appendChild(meta);";
        //用于进行JavaScript注入
        WKUserScript *wkUScript = [[WKUserScript alloc] initWithSource:jSString injectionTime:WKUserScriptInjectionTimeAtDocumentEnd forMainFrameOnly:YES];
        [config.userContentController addUserScript:wkUScript];

```

* WKUserContentController：这个类主要用来做native与JavaScript的交互管理

```

      //这个类主要用来做native与JavaScript的交互管理
        WKUserContentController * wkUController = [[WKUserContentController alloc] init];
        //注册一个name为jsToOcNoPrams的js方法，设置处理接收JS方法的代理
        [wkUController addScriptMessageHandler:self  name:@"jsToOcNoPrams"];
        [wkUController addScriptMessageHandler:self  name:@"jsToOcWithPrams"];
        config.userContentController = wkUController;
       //用完记得移除
       //移除注册的js方法
        [[_webView configuration].userContentController removeScriptMessageHandlerForName:@"jsToOcNoPrams"];
       [[_webView configuration].userContentController removeScriptMessageHandlerForName:@"jsToOcWithPrams"];
```

* WKScriptMessageHandler：这个协议类专门用来处理监听JavaScript方法从而调用原生OC方法，和WKUserContentController搭配使用。

```
注意：遵守WKScriptMessageHandler协议，代理是由WKUserContentControl设置

   //通过接收JS传出消息的name进行捕捉的回调方法
- (void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(WKScriptMessage *)message{
    NSLog(@"name:%@\\\\n body:%@\\\\n frameInfo:%@\\\\n",message.name,message.body,message.frameInfo);
    //用message.body获得JS传出的参数体
    NSDictionary * parameter = message.body;
    //JS调用OC
    if([message.name isEqualToString:@"jsToOcNoPrams"]){
        UIAlertController *alertController = [UIAlertController alertControllerWithTitle:@"js调用到了oc" message:@"不带参数" preferredStyle:UIAlertControllerStyleAlert];
        [alertController addAction:([UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
        }])];
        [self presentViewController:alertController animated:YES completion:nil];
        
    }else if([message.name isEqualToString:@"jsToOcWithPrams"]){
        UIAlertController *alertController = [UIAlertController alertControllerWithTitle:@"js调用到了oc" message:parameter[@"params"] preferredStyle:UIAlertControllerStyleAlert];
        [alertController addAction:([UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
        }])];
        [self presentViewController:alertController animated:YES completion:nil];
    }
}

```


##  二、WKWebView涉及的代理方法

*  WKNavigationDelegate  ：主要处理一些跳转、加载处理操作

```

    // 页面开始加载时调用
- (void)webView:(WKWebView *)webView didStartProvisionalNavigation:(WKNavigation *)navigation {
}
    // 页面加载失败时调用
- (void)webView:(WKWebView *)webView didFailProvisionalNavigation:(null_unspecified WKNavigation *)navigation withError:(NSError *)error {
    [self.progressView setProgress:0.0f animated:NO];
} 
    // 当内容开始返回时调用
- (void)webView:(WKWebView *)webView didCommitNavigation:(WKNavigation *)navigation {
}
    // 页面加载完成之后调用
- (void)webView:(WKWebView *)webView didFinishNavigation:(WKNavigation *)navigation {
    [self getCookie];
}
    //提交发生错误时调用
- (void)webView:(WKWebView *)webView didFailNavigation:(WKNavigation *)navigation withError:(NSError *)error {
    [self.progressView setProgress:0.0f animated:NO];
}  
   // 接收到服务器跳转请求即服务重定向时之后调用
- (void)webView:(WKWebView *)webView didReceiveServerRedirectForProvisionalNavigation:(WKNavigation *)navigation {
}
    // 根据WebView对于即将跳转的HTTP请求头信息和相关信息来决定是否跳转
- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler {
    
    NSString * urlStr = navigationAction.request.URL.absoluteString;
    NSLog(@"发送跳转请求：%@",urlStr);
    //自己定义的协议头
    NSString *htmlHeadString = @"github://";
    if([urlStr hasPrefix:htmlHeadString]){
        UIAlertController *alertController = [UIAlertController alertControllerWithTitle:@"通过截取URL调用OC" message:@"你想前往我的Github主页?" preferredStyle:UIAlertControllerStyleAlert];
        [alertController addAction:([UIAlertAction actionWithTitle:@"取消" style:UIAlertActionStyleCancel handler:^(UIAlertAction * _Nonnull action) {   
        }])];
        [alertController addAction:([UIAlertAction actionWithTitle:@"打开" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
            NSURL * url = [NSURL URLWithString:[urlStr stringByReplacingOccurrencesOfString:@"github://callName_?" withString:@""]];
            [[UIApplication sharedApplication] openURL:url];
        }])];
        [self presentViewController:alertController animated:YES completion:nil];
        decisionHandler(WKNavigationActionPolicyCancel);
    }else{
        decisionHandler(WKNavigationActionPolicyAllow);
    }
}
    
    // 根据客户端受到的服务器响应头以及response相关信息来决定是否可以跳转
- (void)webView:(WKWebView *)webView decidePolicyForNavigationResponse:(WKNavigationResponse *)navigationResponse decisionHandler:(void (^)(WKNavigationResponsePolicy))decisionHandler{
    NSString * urlStr = navigationResponse.response.URL.absoluteString;
    NSLog(@"当前跳转地址：%@",urlStr);
    //允许跳转
    decisionHandler(WKNavigationResponsePolicyAllow);
    //不允许跳转
    //decisionHandler(WKNavigationResponsePolicyCancel);
} 
    //需要响应身份验证时调用 同样在block中需要传入用户身份凭证
- (void)webView:(WKWebView *)webView didReceiveAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential * _Nullable credential))completionHandler{
    //用户身份信息
    NSURLCredential * newCred = [[NSURLCredential alloc] initWithUser:@"user123" password:@"123" persistence:NSURLCredentialPersistenceNone];
    //为 challenge 的发送方提供 credential
    [challenge.sender useCredential:newCred forAuthenticationChallenge:challenge];
    completionHandler(NSURLSessionAuthChallengeUseCredential,newCred);
}
    //进程被终止时调用
- (void)webViewWebContentProcessDidTerminate:(WKWebView *)webView{
}

```

* WKUIDelegate ：主要处理JS脚本，确认框，警告框等

```

 /**
     *  web界面中有弹出警告框时调用
     *
     *  @param webView           实现该代理的webview
     *  @param message           警告框中的内容
     *  @param completionHandler 警告框消失调用
     */
- (void)webView:(WKWebView *)webView runJavaScriptAlertPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(void))completionHandler {
    UIAlertController *alertController = [UIAlertController alertControllerWithTitle:@"HTML的弹出框" message:message?:@"" preferredStyle:UIAlertControllerStyleAlert];
    [alertController addAction:([UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
        completionHandler();
    }])];
    [self presentViewController:alertController animated:YES completion:nil];
}
    // 确认框
    //JavaScript调用confirm方法后回调的方法 confirm是js中的确定框，需要在block中把用户选择的情况传递进去
- (void)webView:(WKWebView *)webView runJavaScriptConfirmPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(BOOL))completionHandler{
    UIAlertController *alertController = [UIAlertController alertControllerWithTitle:@"" message:message?:@"" preferredStyle:UIAlertControllerStyleAlert];
    [alertController addAction:([UIAlertAction actionWithTitle:@"Cancel" style:UIAlertActionStyleCancel handler:^(UIAlertAction * _Nonnull action) {
        completionHandler(NO);
    }])];
    [alertController addAction:([UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
        completionHandler(YES);
    }])];
    [self presentViewController:alertController animated:YES completion:nil];
}
    // 输入框
    //JavaScript调用prompt方法后回调的方法 prompt是js中的输入框 需要在block中把用户输入的信息传入
- (void)webView:(WKWebView *)webView runJavaScriptTextInputPanelWithPrompt:(NSString *)prompt defaultText:(NSString *)defaultText initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(NSString * _Nullable))completionHandler{
    UIAlertController *alertController = [UIAlertController alertControllerWithTitle:prompt message:@"" preferredStyle:UIAlertControllerStyleAlert];
    [alertController addTextFieldWithConfigurationHandler:^(UITextField * _Nonnull textField) {
        textField.text = defaultText;
    }];
    [alertController addAction:([UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
        completionHandler(alertController.textFields[0].text?:@"");
    }])];
    [self presentViewController:alertController animated:YES completion:nil];
}
    // 页面是弹出窗口 _blank 处理
- (WKWebView *)webView:(WKWebView *)webView createWebViewWithConfiguration:(WKWebViewConfiguration *)configuration forNavigationAction:(WKNavigationAction *)navigationAction windowFeatures:(WKWindowFeatures *)windowFeatures {
    if (!navigationAction.targetFrame.isMainFrame) {
        [webView loadRequest:navigationAction.request];
    }
    return nil;
}

```

## 三、网页内容加载进度条的实现

```
   //添加监测网页加载进度的观察者
    [self.webView addObserver:self
                   forKeyPath:NSStringFromSelector(@"estimatedProgress")
                      options:0
                      context:nil];
   //添加监测网页标题title的观察者
    [self.webView addObserver:self
                   forKeyPath:@"title"
                      options:NSKeyValueObservingOptionNew
                      context:nil];

   //kvo 监听进度 必须实现此方法
-(void)observeValueForKeyPath:(NSString *)keyPath
                     ofObject:(id)object
                       change:(NSDictionary<NSKeyValueChangeKey,id> *)change
                      context:(void *)context{
    if ([keyPath isEqualToString:NSStringFromSelector(@selector(estimatedProgress))]
        && object == _webView) {
       NSLog(@"网页加载进度 = %f",_webView.estimatedProgress);
        self.progressView.progress = _webView.estimatedProgress;
        if (_webView.estimatedProgress >= 1.0f) {
            dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.3 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
                self.progressView.progress = 0;
            });
        } 
    }else if([keyPath isEqualToString:@"title"]
             && object == _webView){
        self.navigationItem.title = _webView.title;
    }else{
        [super observeValueForKeyPath:keyPath
                             ofObject:object
                               change:change
                              context:context];
    }
}
    //移除观察者
    [_webView removeObserver:self
                  forKeyPath:NSStringFromSelector(@selector(estimatedProgress))];
    [_webView removeObserver:self
                  forKeyPath:NSStringFromSelector(@selector(title))];

```

##  四、JS和OC的交互

* JS调用OC

> 这个实现主要是依靠WKScriptMessageHandler协议类和WKUserContentController两个类：WKUserContentController对象负责注册JS方法，设置处理接收JS方法的代理，代理遵守WKScriptMessageHandler，实现捕捉到JS消息的回调方法，详情可以看第一步中对这两个类的介绍。

*  OC调用JS

```

 //OC调用JS  changeColor()是JS方法名，completionHandler是异步回调block
    NSString *jsString = [NSString stringWithFormat:@"changeColor('%@')", @"Js参数"];
    [_webView evaluateJavaScript:jsString completionHandler:^(id _Nullable data, NSError * _Nullable error) {
        NSLog(@"改变HTML的背景色");
    }];
    
    //改变字体大小 调用原生JS方法
    NSString *jsFont = [NSString stringWithFormat:@"document.getElementsByTagName('body')[0].style.webkitTextSizeAdjust= '%d%%'", arc4random()%99 + 100];
    [_webView evaluateJavaScript:jsFont completionHandler:nil];

```

## 五、本地HTML文件的实现

> 由于[示例Demo](https://github.com/wslcmk/WKWebView)的需要以及知识有限，我用仅知的HTML、CSS、JavaScript的一点皮毛写了一个HTML文件，比较业余，大神勿喷😁😁
   小白想学习这方面的知识可以看这里： http://www.w3school.com.cn/index.html

> 我用MAC自带的文本编辑工具，生成一个文件，改后缀名，强转为.html文件，同时还需要设置文本编辑打开HTML文件时显示代码（如下图），然后编辑代码。

![文本编辑偏好设置.png](https://upload-images.jianshu.io/upload_images/1708447-44824f3cec7bad72.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


如果我WKWebView使用的总结没帮到你，你也可以看看下面几篇文：
https://www.jianshu.com/p/833448c30d70
https://www.jianshu.com/p/4fa8c4eb1316  
[WKWebView的那些坑](https://mp.weixin.qq.com/s/rhYKLIbXOsUJC_n6dt9UfA?)

## 推荐学习资料:

> [Swift从入门到精通](https://ke.qq.com/course/392094?saleToken=1693443&from=pclink)

> [恋上数据结构与算法（一）](https://ke.qq.com/course/385223?saleToken=1887678&from=pclink)

> [恋上数据结构与算法（二）](https://ke.qq.com/course/421398?saleToken=1887679&from=pclink)

## Welcome To Follow Me

>  您的follow和start，是我前进的动力，Thanks♪(･ω･)ﾉ
> * [简书](https://www.jianshu.com/u/e15d1f644bea)
> * [微博](https://weibo.com/5732733120/profile?rightmod=1&wvr=6&mod=personinfo&is_all=1)
> * [掘金](https://juejin.im/user/5c00d97b6fb9a049fb436288)
> * [CSDN](https://blog.csdn.net/wsl2ls)
> * QQ交流群：835303405
> * 微信号：w2679114653

> 欢迎扫描下方二维码关注——奔跑的程序猿iOSer——微信公众号：iOS2679114653 本公众号是一个iOS开发者们的分享，交流，学习平台，会不定时的发送技术干货，源码,也欢迎大家积极踊跃投稿，(择优上头条) ^_^分享自己开发攻城的过程，心得，相互学习，共同进步，成为攻城狮中的翘楚！

![iOS开发进阶之路.jpg](http://upload-images.jianshu.io/upload_images/1708447-c2471528cadd7c86.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
