---
title: JavaScript调用App原生代码(iOS、Android)通用解决方案
date: 2017-03-05 11:58:00
categories:
- 术-技术综合实践
tags:
- iOS
---

![](http://upload-images.jianshu.io/upload_images/24274-bc0541b22d718aea.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 以前写的一篇 [关于 H5 与 App 原生交互方案](https://github.com/SublimeCodeIntel/SublimeCodeIntel/)，很多人问有没有实例代码，今天来说一个对 iOS 与 Android 通用的代码实践

### 实际场景

场景：现在有一个 H5 活动页面，上面有一个登陆按钮，要求点击登陆按钮以后，唤出 App 内部的登录界面，当登录成功以后将用户的手机号返回给 H5 页面，显示出来。
这个场景应该算是比较完整的一次 H5 中的 JavaScript 与 App 原生代码进行交互了，这个过程，我们制定的方案满足以下几点：

1.  满足基本的交互流程的功能
2.  Android 与 iOS 都能适用
3.  H5 的前端开发者，在书写 JavaScript 的业务代码的时候不需要为了迁就移动端语言的特性而写特殊的磨合代码
4.  方便调试

### 交互流程

上一篇文章里提到，当 H5 页面上的 JavaScript 代码要调用原生的页面或者组件的时候，调用最好是双向的，一来一回，这样比较容易满足一些比较复杂的业务场景，就像上面的场景一样，有调用，有回调告知 H5 调用的结果。前端开发写的 JavaScript 代码基本上都是异步风格的，就拿上面的场景，如果登录是 H5 前端的，那么这个流程就会是：

![](https://dn-mhke0kuv.qbox.me/5c846cf8df18e537caca.png)

代码如下:

```
function loginClick() {
    loginComponent.login(function (error,result) {
        //处理登录完成以后的逻辑
    });
}
var loginComponent = {
    callBack:null,
    "login":function (callBack) {
        this.show();
        this.callBack = callBack;
    },
    show:function (loginComponent) {
        //登录组件显示的逻辑
    },
    confirm:function (userName,password) {
        ajax.post('https://xxxx.com/login',function (error,result) {
           if(this.callBack !== null){
                this.callBack(error,result);
           }
        });
    }
}
```

如果要改成调用原生登录，那么这个流程就应该是这样:

![](https://dn-mhke0kuv.qbox.me/ef74ad05956528b2a82e.png)
确定了流程，接下来就可以详细设计和实现

### 原生与 JavaScript 的桥梁

为了实现上述流程，并且能让 H5 的前端开发尽可能少的语法损失，我们需要构建一个 JavaScript 与原生 App 进行交互的桥梁，这个桥梁来处理与 App 的协议交互，兼容 iOS 与 Android 的交互实现。

![](https://dn-mhke0kuv.qbox.me/2869123fc6c8ec206585.png)

Android 与 iOS 都支持在打开 H5 页面的时候，向 H5 页面的 window 对象上注入一个 JavaScript 可以访问到的对象，Android 端使用的是

```
webView.addJavascriptInterface(myJavaScriptInterface, “bridge”);
```

iOS 则可以使用 JavaScriptCore 来完成：

```
#import <Foundation/Foundation.h>
#import <JavaScriptCore/JavaScriptCore.h>
@protocol PICBridgeExport <JSExport>
@end
@interface PICBridge : NSObject<PICBridgeExport>
@end


self.jsContext =  [self.webView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];
    self.bridge =[[PICBridge alloc]init];
```

这里面 Android 的 myJavaScriptInterface 与 PICBridge 都是作为与 JavaScript 进行通信的桥梁。
我们使用设计这个桥梁的时候，需要使用一个具体的语法约定和数据约定，比方说，当前端开发调用 App 登录的时候，他一定是希望就像调用其他 JavaScript 的组件一样，而登录的结果通过传入 callBack 的函数来完成，对于 callBack 函数，我们希望借助 NodeJS 的规范:

```
function(error,res) {
    //回调函数第一个参数是错误，第二个参数是结果
}
```

以上我们可以看到，bridge 必须有能力将前端开发写的 JavaScript 回调函数传入到 App 内部，然后 App 处理完逻辑以后通过回调函数来告知前端处理，并且这个需要通过约定好的数据格式来传递入参和返回值。
为了完成双向通信，我们就需要在 JavaScript 设置一个 bridge，原生再注入一个 bridge，这两个 bridge 按照一定的数据约定来进行双向通信和分发逻辑。

### 原生端注入到 JS 当中的“桥”(iOS 端)

通过使用 JavaScriptCore 这个库，我们能很容易的将 JavaScript 传入的回调函数在 objective-c 或者是 swift 端持有，并回去回调这个回调函数。

```
#import <Foundation/Foundation.h>
#import <JavaScriptCore/JavaScriptCore.h>
@protocol PICBridgeExport <JSExport>
JSExportAs(callRouter, -(void)callRouter:(JSValue *)requestObject callBack:(JSValue *)callBack);
@end
@interface PICBridge : NSObject<PICBridgeExport>
-(void)addActionHandler:(NSString *)actionHandlerName forCallBack:(void(^)(NSDictionary * params,void(^errorCallBack)(NSError * error),void(^successCallBack)(NSDictionary * responseDict)))callBack;
@end
```

需要说明的是，JavaScript 没有函数参数标签的概念，JSExportAs 是用来将 objective-c 的方法映射为 JavaScript 的函数。
-(void)callRouter:(JSValue _)requestObject callBack:(JSValue _)callBack);
这个方法是暴露给 JavaScript 端调用的。
第一个参数 requestObject 是一个 JavaScript 对象，传入到 objective-c 中以后就可以转换为 key-value 结构的字典，那么这个字典的数据约定是：

```
{
    'Method':'Login',
    'Data':null
}
```

其中 Method 是 App 内部对外提供的 API，而这个 Data 则是该 API 需要的入参。
第二个参数是一个 callBack 函数，该类型的 JSValue 可以调用 callWithArguments:方法来 invoke 这个回调函数。
前面已经说明，回调函数的第一个参数是 error，第二个参数是一个结果，而回调的结果我们也进行一下约定，那就是：

```
{
    'result':{}
}
```

这样的好处是，业务逻辑可以讲返回的结果放入 result 中，跟 result 同级别的我们还可以加入统一的签名认证的东西，在此暂时不延伸。
原生端的 bridge 的来实现一下 callRouter:

```
-(void)callRouter:(JSValue *)requestObject callBack:(JSValue *)callBack{
    NSDictionary * dict = [requestObject toDictionary];
    NSString * methodName = [dict objectForKey:@"Method"];
    if (methodName != nil && methodName.length>0) {
        NSDictionary * params = [dict objectForKey:@"Data"];
        __weak PICBridge * weakSelf = self;
//因为JavaScript是单线程的，需要尽快完成调用逻辑，耗时操作需要异步提交到主线程中执行
        dispatch_async(dispatch_get_main_queue(), ^{
            [weakSelf callAction:methodName params:params success:^(NSDictionary *responseDict) {
                    if (responseDict != nil) {
                        NSString * result = [weakSelf responseStringWith:responseDict];
                        if (result) {
                            [callBack callWithArguments:@[@"null",result]];
                        }
                        else{
                            [callBack callWithArguments:@[@"null",@"null"]];
                        }
                    }
                    else{
                        [callBack callWithArguments:@[@"null",@"null"]];
                    }
            } failure:^(NSError *error) {
                    if (error) {
                        [callBack callWithArguments:@[[error description],@"null"]];
                    }
                    else{
                        [callBack callWithArguments:@[@"App Inner Error",@"null"]];
                    }
            }];
        });
    }
    else{

        [callBack callWithArguments:@[@NO,[PICError ErrorWithCode:PICUnkonwError].description]];
    }
    return;
}
//将返回的结果字典转换为字符串通过回调函数传回给JavaScript
-(NSString *)responseStringWith:(NSDictionary *)responseDict{
    if (responseDict) {
        NSDictionary * dict = @{@"result":responseDict};
        NSData * data = [NSJSONSerialization dataWithJSONObject:dict options:NSJSONWritingPrettyPrinted error:nil];
        NSString * result = [[NSString alloc]initWithData:data encoding:NSUTF8StringEncoding];
        return result;
    }
    else{
        return nil;
    }
}
```

callAction 函数实际上就是分发业务逻辑用的

```
-(void)callAction:(NSString *)actionName params:(NSDictionary *)params success:(void(^)(NSDictionary * responseDict))success failure:(void(^)(NSError * error))failure{
    void(^callBack)(NSDictionary * params,void(^errorCallBack)(NSError * error),void(^successCallBack)(NSDictionary * responseDict)) = [self.handlers objectForKey:actionName];
    if (callBack != nil) {
        callBack(params,failure,success);
    }
}
```

这个 callBack Block 是在 self.handlers 的字典中存储，比较复杂，block 第一个参数是传入的入参，后面两个参数是成功以后的回调和失败以后的回调，以便业务逻辑完成后进行回调给 JavaScript。
同时会有注册业务逻辑的方法：

```
-(void)addActionHandler:(NSString *)actionHandlerName forCallBack:(void(^)(NSDictionary * params,void(^errorCallBack)(NSError * error),void(^successCallBack)(NSDictionary * responseDict)))callBack{
    if (actionHandlerName.length>0 && callBack != nil) {
        [self.handlers setObject:callBack forKey:actionHandlerName];
    }
}
```

至此，原生端路由实现完毕。

### JavaScript 端路由

先贴上完整代码：

```
(function(win) {

    var ua = navigator.userAgent;
    function getQueryString(name) {
        var reg = new RegExp('(^|&)' + name + '=([^&]*)(&|$)', 'i');
        var r = window.location.search.substr(1).match(reg);
        if (r !== null) return unescape(r[2]);
        return null;
    }

    function isAndroid() {
        return ua.indexOf('Android') > 0;
    }

    function isIOS() {
        return /(iPhone|iPad|iPod)/i.test(ua);
    }
    var mobile = {

        /**
         *通过bridge调用app端的方法
         * @param method
         * @param params
         * @param callback
         */
        callAppRouter: function(method, params, callback) {
            var req = {
                'Method': method,
                'Data': params
            };
            if (isIOS()) {
                win.bridge.callRouter(req, function(err, result) {
                    var resultObj = null;
                    var errorMsg = null;
                    if (typeof(result) !== 'undefined' && result !== 'null' && result !== null) {
                        resultObj = JSON.parse(result);
                        if (resultObj) {
                            resultObj = resultObj['result'];
                        }
                    }
                    if (err !== 'null' && typeof(err) !== 'undefined' && err !== null) {
                        errorMsg = err;
                    }
                    callback(err, resultObj);
                });
            } else if (isAndroid()) {
                //生成回调函数方法名称
                var cbName = 'CB_' + Date.now() + '_' + Math.ceil(Math.random() * 10);
                //挂载一个临时函数到window变量上，方便app回调
                win[cbName] = function(err, result) {
                    var resultObj;
                    if (typeof(result) !== 'undefined' && result !== null) {
                        resultObj = JSON.parse(result)['result'];
                    }
                    callback(err, resultObj);
                    //回调成功之后删除挂载到window上的临时函数
                    delete win[cbName];
                };
                win.bridge.callRouter(JSON.stringify(req), cbName);
            }
        },
        login: function() {
            // body...
            this.callAppRouter('Login', null, function(errMsg, res) {
                // body...

                if (errMsg !== null && errMsg !== 'undefined' && errMsg !== 'null') {

                } else {
                    var name = res['phone'];
                    if (name !== 'undefined' && name !== 'null') {
                        var button = document.getElementById('loginButton');
                        button.innerHTML = name;
                    }
                }
            });
        }
    };

    //将mobile对象挂载到window全局
    win.webBridge = mobile;
})(window);
```

在 window 上挂在一个叫 webBridge 的对象，其他业务 JavaScript 可以通过 webBridge.login 来进行调用原生端开放的 API。
callAppRouter 方法的实现我们来分析一下：
如果判断是 iOS 设备，则使用 iOS 注册的 bridge 对象进行调用 callRouter 方法：

```
if (isIOS()) {
                win.bridge.callRouter(req, function(err, result) {
                    var resultObj = null;
                    var errorMsg = null;
                    if (typeof(result) !== 'undefined' && result !== 'null' && result !== null) {
                        resultObj = JSON.parse(result);
                        if (resultObj) {
                            resultObj = resultObj['result'];
                        }
                    }
                    if (err !== 'null' && typeof(err) !== 'undefined' && err !== null) {
                        errorMsg = err;
                    }
                    callback(err, resultObj);
                });
            }
```

req 是标准的包含 Method 和 Data 的对象，紧接着传入回调函数，回调函数有 err 与 result，里面做好各种类型检查。
着重说一下 Android 端的实现，因为 Android 端的 JavaScript 方法注册，参数类型只能字符串，java 语言本身没有匿名函数的概念，所以只能给 Java 端传入回调函数的名字，而回调函数的实现则在 JavaScript 端持有。

```
else if (isAndroid()) {
                //生成回调函数方法名称
                var cbName = 'CB_' + Date.now() + '_' + Math.ceil(Math.random() * 10);
                //挂载一个临时函数到window变量上，方便app回调
                win[cbName] = function(err, result) {
                    var resultObj;
                    if (typeof(result) !== 'undefined' && result !== null) {
                        resultObj = JSON.parse(result)['result'];
                    }
                    callback(err, resultObj);
                    //回调成功之后删除挂载到window上的临时函数
                    delete win[cbName];
                };
                win.bridge.callRouter(JSON.stringify(req), cbName);
            }
```

本质上就是将其他业务 JavaScript 代码传入的 callBack 函数通过随机生成函数名，挂在到 window 变量上，回调以后将其删除：delete win[cbName]。
当调用 Java 端的 bridge.callRouter(JSON.stringify(req), cbName)，Java 端拿到 cbName，在完成业务逻辑后，按照标准数据格式，在 JavaScript 执行的上下文中，回调这个名字的方法。
至此，前端的 webBridge 完成。

最后附上 Demo 地址：
https://github.com/Neojoke/Picidae.git
此 Demo 是通过 H5 调用原生的登录界面，登录成功以后将手机号在 H5 上的登录按钮显示出来，完成一整套逻辑交互。喜欢的给个 ✨，有任何问题大家多多交流！
