---
layout: post
title: "关于ionic1.x打包成APP，输入框调出输入法按返回键造成页面后退的问题解决"
date: 2016-12-06 10:45
tags: ionic
category: 随笔
description: 解决ionic在国产输入法按返回键造成页面后退的问题解决。

---
本人亲自试用的解决方案，先贴上完整代码：在项目的app.js里配置如下
```javascript
angular.module('starter', ['ionic', 'starter.controllers', 'starter.services', 'common'])  
  
  .run(function ($ionicPlatform, $ionicHistory, $ionicPopup, $timeout) {  
    $ionicPlatform.ready(function () {  
      // Hide the accessory bar by default (remove this to show the accessory bar above the keyboard  
      // for form inputs)  
      if (window.cordova && window.cordova.plugins && window.cordova.plugins.Keyboard) {  
        cordova.plugins.Keyboard.hideKeyboardAccessoryBar(true);  
        cordova.plugins.Keyboard.disableScroll(true);  
      }  
      if (window.StatusBar) {  
        // org.apache.cordova.statusbar required  
        StatusBar.styleDefault();  
      }  
      //延迟设置isVisible为false，防止第三方输入法返回退出当前页面  
      window.addEventListener('native.keyboardhide', function (e) {  
        cordova.plugins.Keyboard.isVisible = true;  
        $timeout(function () {  
          cordova.plugins.Keyboard.isVisible = false;  
        }, 100);  
      });  
    });  
  
    $ionicPlatform.registerBackButtonAction(function (event) {  
      event.preventDefault();  
      if ($ionicHistory.backView()) {  
        if (cordova.plugins.Keyboard.isVisible) {  
          cordova.plugins.Keyboard.close();  
        } else {  
          $ionicHistory.goBack();  
        }  
      } else {  
        ionic.Platform.exitApp();  
      }  
      return false;  
    }, 101);  
  })  
```

我们新建的ionic项目应该默认 引入了键盘插件：
```javascript
cordova.plugins.Keyboard 
``` 
要解决这个问题，我们需要重新定义按返回键的事件。我们可以在registerBackButtonAction方法中覆盖原来的逻辑。起方法代码如下：
```javascript
registerBackButtonAction: function(fn, priority, actionId) {  
  
          if (!self._hasBackButtonHandler) {  
            // add a back button listener if one hasn't been setup yet  
            self.$backButtonActions = {};  
            self.onHardwareBackButton(self.hardwareBackButtonClick);  
            self._hasBackButtonHandler = true;  
          }  
  
          var action = {  
            id: (actionId ? actionId : ionic.Utils.nextUid()),  
            priority: (priority ? priority : 0),  
            fn: fn  
          };  
          self.$backButtonActions[action.id] = action;  
  
          // return a function to de-register this back button action  
          return function() {  
            delete self.$backButtonActions[action.id];  
          };  
        }  

```
fn是按返回键回调函数，priority则指定了该回调函数执行的优先级。默认的优先级有：
```xml
*   Return to previous view = 100  
*   Close side menu = 150  
*   Dismiss modal = 200  
*   Close action sheet = 300  
*   Dismiss popup = 400  
*   Dismiss loading overlay = 500  
```
priority=100时，即为返回上一个视图的执行条件，因此我们只需要设置一个优先级比100高同时又比其他默认优先级要低，以防止影响其他组件的响应。这里我们可以设置为101。
```javascript
$ionicPlatform.registerBackButtonAction(function (event) {  
     event.preventDefault();  
     if ($ionicHistory.backView()) {  
       if (cordova.plugins.Keyboard.isVisible) {  
         cordova.plugins.Keyboard.close();  
       } else {  
         $ionicHistory.goBack();  
       }  
     } else {  
       ionic.Platform.exitApp();  
     }  
     return false;  
   }, 101);  
```
上面的方法，首先判断当前视图栈中是否还有历史视图，没有的话调用exitApp直接退出程序。

否则就要判断键盘的显示状态，这里就是问题的关键。如何在键盘打开的情况下，按返回键不触发返回上一个视图呢？

我们可以通过监听键盘隐藏事件，当按返回键键盘隐藏的时候延迟设置键盘可见性，以防在返回键被按下的监听函数里根据isVisible为true直接调用goBack退出取当前视图。
```javascript
window.addEventListener('native.keyboardhide', function (e) {  
       cordova.plugins.Keyboard.isVisible = true;  
       $timeout(function () {  
         cordova.plugins.Keyboard.isVisible = false;  
       }, 100);  
     });  

```
上面的代码中，监听到键盘隐藏时先把isVisible设置为true，然后延迟100ms再设置为false，此时在registerBackButtonAction判断isVisible状态为true就不会调用goBack退出当前视图了。

我们可以在这两个监听函数里通过$ionicPopup打印日志来看这个判断过程。

提供一个简单的信息打印方法：
```java
showAlert : function (message,title,buttonName) {  
       //使用原生UI  
       if (window.cordova && window.cordova.plugins)  
         $cordovaDialogs.alert(message, title, buttonName);  
       else  
         $ionicPopup.alert({  
           title: title,  
           template: message,  
           okText: buttonName  
         });  
```