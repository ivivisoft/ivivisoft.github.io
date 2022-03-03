---
title: 小程序登录的正确打开方式
excerpt: 小程序登录经历过了几次改版，其中有很多的细小知识点值得我们注意，本文介绍微信小程序登录的正确打开方式。
date: 2022-03-03 12:00:50
category: Wechat
tags:
- 登录
---

#### 小程序网络组件

[RequestTask](https://developers.weixin.qq.com/miniprogram/dev/api/network/request/RequestTask.html) [wx.request(Object object)](https://developers.weixin.qq.com/miniprogram/dev/api/network/request/wx.request.html)

**RequestTask说明**

| 方法                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [RequestTask.abort()](https://developers.weixin.qq.com/miniprogram/dev/api/network/request/RequestTask.abort.html) | 中断请求任务。                                               |
| [RequestTask.onHeadersReceived(function callback)](https://developers.weixin.qq.com/miniprogram/dev/api/network/request/RequestTask.onHeadersReceived.html) | 监听 HTTP Response Header 事件。会比请求完成事件更早。       |
| [RequestTask.offHeadersReceived(function callback)](https://developers.weixin.qq.com/miniprogram/dev/api/network/request/RequestTask.offHeadersReceived.html) | 取消监听 HTTP Response Header 事件。                         |
| [RequestTask.onChunkReceived(function callback)](https://developers.weixin.qq.com/miniprogram/dev/api/network/request/RequestTask.onChunkReceived.html) | 监听 Transfer-Encoding Chunk Received 事件。当接收到新的chunk时触发。 |
| [RequestTask.offChunkReceived(function callback)](https://developers.weixin.qq.com/miniprogram/dev/api/network/request/RequestTask.offChunkReceived.html) | 取消监听 Transfer-Encoding Chunk Received 事件。             |

**wx.request(Object object)属性**

*此处只列比较常用的属性，全部属性请查看[链接](https://developers.weixin.qq.com/miniprogram/dev/api/network/request/wx.request.html)。*

| 属性     | 类型                      | 默认值 | 必填 | 说明                                                         |
| -------- | ------------------------- | ------ | ---- | ------------------------------------------------------------ |
| url      | string                    |        | 是   | 开发者服务器接口地址                                         |
| data     | string/object/ArrayBuffer |        | 否   | 请求的参数                                                   |
| header   | Object                    |        | 否   | 设置请求的 header，header 中不能设置 Referer。 `content-type` 默认为 `application/json` |
| timeout  | number                    |        | 否   | 超时时间，单位为毫秒                                         |
| method   | string                    | GET    | 否   | HTTP 请求方法                                                |
| success  | function                  |        | 否   | 接口调用成功的回调函数                                       |
| fail     | function                  |        | 否   | 接口调用失败的回调函数                                       |
| complete | function                  |        | 否   | 接口调用结束的回调函数（调用成功、失败都会执行）哪怕是abort掉的请求！ |

**总结一下：所有的小程序接口基本上都有两个特征：**

1. 参数都是一个对象。便于记忆的同时方便扩展。

2. 都有相同的结果处理方式：都有success、fail、complete三个回调属性。

**接口执行的各种情况下的errMsg对象介绍。**

| 回调属性 | errMsg对象                                                   |
| -------- | ------------------------------------------------------------ |
| success  | {errMsg:"request:ok"...}                                     |
| fail     | {errMsg:"request:fail "...} 有的系统这个fail后面有个空格，所以要使用这个判断，最好是使用正则表达式。也可以使用indexOf函数，大于-1进行判断。 |
| abort    | {errMsg:"request:fail abort"...}                             |

   **示例代码**

```javascript
  let reqTask = wx.request({
      url: getApp().globalData.api,
      success(res) {
        if (res.errMsg === "request:ok") console.log("res", res);
      },
      fail(err) {
        // if(err.errMsg.indexOf('request:fail')>-1) console.log('err', err);
        if (/^request:fail/i.test(err.errMsg)) console.log("err", err);
      },
      complete(res) {
        console.log("resOrErr", res);
      },
    });
   const reqTaskOnHeadersReceived = (headers) => {
      reqTask.offHeadersReceived(reqTaskOnHeadersReceived);
      console.log("headers", headers);
      // 由于请求还未完全结束，所以我们没办法获得请求的状态码，但是我们可以通过返回的requestBody的长度来进行判断。
      // 两点说明：1. 两个~~可以把字符串数字快速转化为数字。
      // 2. 为什么取小于19，是由于后台返回没有权限的requestBody的时候Content-length为“18”，正常情况下是大于19的。所以具体多少得看一下具体情况。
      if (~~headers.header["Content-length"] < 19) reqTask.abort();
    };
    reqTask.onHeadersReceived(reqTaskOnHeadersReceived);
```



####  小程序登录接口

1. [wx.getUserProfile(Object object)](https://developers.weixin.qq.com/miniprogram/dev/api/open-api/user-info/wx.getUserProfile.html)

   获取用户信息。页面产生点击事件（例如 `button` 上 `bindtap` 的回调中）后才可调用，每次请求都会弹出授权窗口，用户同意后返回 `userInfo`。该接口用于替换 `wx.getUserInfo`，详见 [用户信息接口调整说明](https://developers.weixin.qq.com/community/develop/doc/000cacfa20ce88df04cb468bc52801?highLine=login)。

2. [wx.checkSession(Object object)](https://developers.weixin.qq.com/miniprogram/dev/api/open-api/login/wx.checkSession.html)

   检查登录态是否过期。 通过 wx.login 接口获得的用户登录态拥有一定的时效性。用户越久未使用小程序，用户登录态越有可能失效。反之如果用户一直在使用小程序，则用户登录态一直保持有效。具体时效逻辑由微信维护，对开发者透明。开发者只需要调用 wx.checkSession 接口检测当前用户登录态是否有效。

   登录态过期后开发者可以再调用 wx.login 获取新的用户登录态。调用成功说明当前 session_key 未过期，调用失败说明 session_key 已过期。更多使用方法详见 [小程序登录](https://developers.weixin.qq.com/miniprogram/dev/framework/open-ability/login.html)。

3. [wx.login(Object object)](https://developers.weixin.qq.com/miniprogram/dev/api/open-api/login/wx.login.html)

   调用接口获取登录凭证（code）。通过凭证进而换取用户登录态信息，包括用户在当前小程序的唯一标识（openid）、微信开放平台帐号下的唯一标识（unionid，若当前小程序已绑定到微信开放平台帐号）及本次登录的会话密钥（session_key）等。用户数据的加解密通讯需要依赖会话密钥完成。更多使用方法详见 [小程序登录](https://developers.weixin.qq.com/miniprogram/dev/framework/open-ability/login.html)。

#### 后端登录接口代码实现

**后端使用NodeJS，web框架KOA版本^2.13.4，路由框架@koa/router版本^10.1.1，框架request，版本 ^2.88.2，jsonwebtoken用来加密解密token信息，版本^8.5.1**

```javascript
// app.js
const Koa = require("koa");
const Router = require("@koa/router");
const WeixinAuth = require("./lib/koa2-weixin-auth");
const jsonwebtoken = require("jsonwebtoken");

const app = new Koa();
// 小程序机票信息
const miniProgramAppId = "*********";
const miniProgramAppSecret = "***********";
const weixinAuth = new WeixinAuth(miniProgramAppId, miniProgramAppSecret);

const JWT_SECRET = "JWTSECRET";
// 路由中间件需要安装@koa/router
// 开启一个带群组的路由
const router = new Router({
  prefix: "/user",
});
// 这是正规的登陆方法
// 添加一个参数，sessionKeyIsValid，代表sessionKey是否还有效
router.post("/weixin-login", async (ctx) => {
  let { code, userInfo, encryptedData, iv, sessionKeyIsValid } =
    ctx.request.body;
   // 解析openid
  const token = await weixinAuth.getAccessToken(code);
  userInfo.openid = token.data.openid;
  // 这里可以自己进行处理，比方说记录到数据库,处理token等
  let authorizationToken = jsonwebtoken.sign(
    { name: userInfo.nickName },
    JWT_SECRET,
    { expiresIn: "1d" }
  );
  Object.assign(userInfo, { authorizationToken });
  ctx.status = 200;
  ctx.body = {
    code: 200,
    msg: "ok",
    data: userInfo,
  };
});
```

```javascript
// lib/koa2-weixin-auth.js
const querystring = require("querystring");
const request = require("request");

const AccessToken = function (data) {
  if (!(this instanceof AccessToken)) {
    return new AccessToken(data);
  }
  this.data = data;
};

/*!
 * 检查AccessToken是否有效，检查规则为当前时间和过期时间进行对比
 *
 * Examples:
 * ```
 * token.isValid();
 * ```
 */
AccessToken.prototype.isValid = function () {
  return (
    !!this.data.session_key &&
    new Date().getTime() < this.data.create_at + this.data.expires_in * 1000
  );
};

/**
 * 根据appid和appsecret创建OAuth接口的构造函数
 * 如需跨进程跨机器进行操作，access token需要进行全局维护
 * 使用使用token的优先级是：
 *
 * 1. 使用当前缓存的token对象
 * 2. 调用开发传入的获取token的异步方法，获得token之后使用（并缓存它）。

 * Examples:
 * ```
 * var OAuth = require('oauth');
 * var api = new OAuth('appid', 'secret');
 * ```
 * @param {String} appid 在公众平台上申请得到的appid
 * @param {String} appsecret 在公众平台上申请得到的app secret
 */
const Auth = function (appid, appsecret) {
  this.appid = appid;
  this.appsecret = appsecret;
  this.store = {};

  this.getToken = function (openid) {
    return this.store[openid];
  };

  this.saveToken = function (openid, token) {
    this.store[openid] = token;
  };
};

/**
 * 获取授权页面的URL地址
 * @param {String} redirect 授权后要跳转的地址
 * @param {String} state 开发者可提供的数据
 * @param {String} scope 作用范围，值为snsapi_userinfo和snsapi_base，前者用于弹出，后者用于跳转
 */
Auth.prototype.getAuthorizeURL = function (redirect_uri, scope, state) {
  return new Promise((resolve, reject) => {
    const url = "https://open.weixin.qq.com/connect/oauth2/authorize";
    let info = {
      appid: this.appid,
      redirect_uri: redirect_uri,
      scope: scope || "snsapi_base",
      state: state || "",
      response_type: "code",
    };
    resolve(url + "?" + querystring.stringify(info) + "#wechat_redirect");
  });
};

/*!
 * 处理token，更新过期时间
 */
Auth.prototype.processToken = function (data) {
  data.create_at = new Date().getTime();
  // 存储token
  this.saveToken(data.openid, data);
  return AccessToken(data);
};

/**
 * 根据授权获取到的code，换取access token和openid
 * 获取openid之后，可以调用`wechat.API`来获取更多信息
 * @param {String} code 授权获取到的code
 */
Auth.prototype.getAccessToken = function (code) {
  return new Promise((resolve, reject) => {
    const url = "https://api.weixin.qq.com/sns/jscode2session";
    //由于此框架版本很久没有更新了，此处地址发生了变化，需要修改为以上地址，不然会出现
    //41008错误。这也是没有直接使用框架，引用本地使用的原因。
    // const url = "https://api.weixin.qq.com/sns/oauth2/access_token";
    const info = {
      appid: this.appid,
      secret: this.appsecret,
      js_code: code,
      grant_type: "authorization_code",
    };
    request.post(url, { form: info }, (err, res, body) => {
      if (err) {
        reject(err);
      } else {
        const data = JSON.parse(body);
        resolve(this.processToken(data));
      }
    });
  });
};

/**
 * 根据refresh token，刷新access token，调用getAccessToken后才有效
 * @param {String} refreshToken refreshToken
 */
Auth.prototype.refreshAccessToken = function (refreshToken) {
  return new Promise((resolve, reject) => {
    const url = "https://api.weixin.qq.com/sns/oauth2/refresh_token";
    var info = {
      appid: this.appid,
      grant_type: "refresh_token",
      refresh_token: refreshToken,
    };
    request.post(url, { form: info }, (err, res, body) => {
      if (err) {
        reject(err);
      } else {
        const data = JSON.parse(body);
        resolve(this.processToken(data));
      }
    });
  });
};

/**
 * 根据openid，获取用户信息。
 * 当access token无效时，自动通过refresh token获取新的access token。然后再获取用户信息
 * @param {Object|String} options 传入openid或者参见Options
 */
Auth.prototype.getUser = async function (openid) {
  const data = this.getToken(openid);
  console.log("getUser", data);
  if (!data) {
    var error = new Error(
      "No token for " + options.openid + ", please authorize first."
    );
    error.name = "NoOAuthTokenError";
    throw error;
  }
  const token = AccessToken(data);
  var accessToken;
  if (token.isValid()) {
    accessToken = token.data.session_key;
  } else {
    var newToken = await this.refreshAccessToken(token.data.refresh_token);
    accessToken = newToken.data.session_key;
  }
  console.log("accessToken", accessToken);
  return await this._getUser(openid, accessToken);
};

Auth.prototype._getUser = function (openid, accessToken, lang) {
  return new Promise((resolve, reject) => {
    const url = "https://api.weixin.qq.com/sns/userinfo";
    const info = {
      access_token: accessToken,
      openid: openid,
      lang: lang || "zh_CN",
    };
    request.post(url, { form: info }, (err, res, body) => {
      if (err) {
        reject(err);
      } else {
        resolve(JSON.parse(body));
      }
    });
  });
};

/**
 * 根据code，获取用户信息。
 * @param {String} code 授权获取到的code
 */
Auth.prototype.getUserByCode = async function (code) {
  const token = await this.getAccessToken(code);
  return await this.getUser(token.data.openid);
};

module.exports = Auth;
```

#### 小程序端登录代码实现

```html
<!--pages/index.wxml-->
<view class="page-section">
    <text class="page-section__title">微信登录</text>
    <view class="btn-area">
        <button  bindtap="getUserProfile" type="primary">登录</button>
    </view>
</view>
```

```javascript
// pages/index.js
Page({
  /**
   * 页面的初始数据
   */
  data: {},
  // 正确的登录方式
  getUserProfile() {
    // 推荐使用wx.getUserProfile获取用户信息，开发者每次通过该接口获取用户个人信息均需用户确认
    // 开发者妥善保管用户快速填写的头像昵称，避免重复弹窗
    wx.getUserProfile({
      desc: "用于完善会员资料", // 声明获取用户个人信息后的用途，后续会展示在弹窗中，请谨慎填写
      success: (res) => {
        let { userInfo, encryptedData, iv } = res;
        const requestLoginApi = (code) => {
          // 发起网络请求
          wx.request({
            url: "http://localhost:3000/user/weixin-login",
            method: "POST",
            header: {
              "content-type": "application/json",
            },
            data: {
              code,
              userInfo,
              encryptedData,
              iv,
            },
            success(res) {
              console.log("请求成功", res.data);
              let token = res.data.data.authorizationToken;
              wx.setStorageSync("token", token);
              onUserLogin(token);
              console.log("authorization", token);
            },
            fail(err) {
              console.log("请求异常", err);
            },
          });
        };
        const onUserLogin = (token) => {
          getApp().globalData.token = token;
          wx.showToast({
            title: "登录成功了",
          });
        };
        //必须进行session是否过期检查，不然会出现第一次点击登录，服务器报Illegal Buffer
        //的错误，但是第二次点击登录正常。
        wx.checkSession({
          success: (res) => {
            // session_key 未过期，并且在本生命周期一直有效
            console.log("在登陆中");
            let token = wx.getStorageSync("token");
            if (token) onUserLogin(token);
          },
          fail: (res) => {
            // session_key已经失效，需要重新执行登录流程
            wx.login({
              success(res0) {
                if (res0.code) {
                  requestLoginApi(res0.code);
                } else {
                  console.log("登录失败!" + res0.errMsg);
                }
              },
            });
          },
        });
      },
    });
  },
});

```

#### 针对登录代码可以做哪些优化？

对于一个软件，就代码层面而言，需要追求最基本的几个方面（远不止这些，但是先姑且先做个好这些吧）：

1. 可维护性（maintainability）

   所谓的“维护”无外乎就是修改 bug、修改老的代码、添加新的代码之类的工作。所谓“代码易维护”就是指，在不破坏原有代码设计、不引入新的 bug 的情况下，能够快速地修改或者添加代码。所谓“代码不易维护”就是指，修改或者添加代码需要冒着极大的引入新 bug 的风险，并且需要花费很长的时间才能完成。

2. 可读性（readability）

   软件设计大师 Martin Fowler 曾经说过：“Any fool can write code that a computer can understand. Good programmers write code that humans can understand.”翻译成中文就是：“任何傻瓜都会编写计算机能理解的代码。好的程序员能够编写人能够理解的代码。”Google 内部甚至专门有个认证就叫作 Readability。只有拿到这个认证的工程师，才有资格在 code review 的时候，批准别人提交代码。可见代码的可读性有多重要，毕竟，代码被阅读的次数远远超过被编写和执行的次数。我们需要看代码是否符合编码规范、命名是否达意、注释是否详尽、函数是否长短合适、模块划分是否清晰、是否符合高内聚低耦合等等。

3. 可扩展性（extensibility）

   可扩展性也是一个评价代码质量非常重要的标准。代码预留了一些功能扩展点，你可以把新功能代码，直接插到扩展点上，而不需要因为要添加一个功能而大动干戈，改动大量的原始代码。

4. 可复用性（reusability）

   代码的可复用性可以简单地理解为，尽量减少重复代码的编写，复用已有的代码。

那么接下来就来优化一下代码吧：

##### 模块化

可以把登录的代码模块化，代码如下：

```javascript
// lib/login.js
function loginWithCallback(cb) {
  // 推荐使用wx.getUserProfile获取用户信息，开发者每次通过该接口获取用户个人信息均需用户确认
  // 开发者妥善保管用户快速填写的头像昵称，避免重复弹窗
  wx.getUserProfile({
    desc: "用于完善会员资料", // 声明获取用户个人信息后的用途，后续会展示在弹窗中，请谨慎填写
    success: (res) => {
      let { userInfo, encryptedData, iv } = res;
      const requestLoginApi = (code) => {
        // 发起网络请求
        wx.request({
          url: "http://localhost:3000/user/weixin-login",
          method: "POST",
          header: {
            "content-type": "application/json",
          },
          data: {
            code,
            userInfo,
            encryptedData,
            iv,
          },
          success(res) {
            console.log("请求成功", res.data);
            let token = res.data.data.authorizationToken;
            wx.setStorageSync("token", token);
            onUserLogin(token);
            console.log("authorization", token);
          },
          fail(err) {
            console.log("请求异常", err);
          },
        });
      };

      const onUserLogin = (token) => {
        getApp().globalData.token = token;
        wx.showToast({
          title: "登录成功了",
        });
        if (cb && typeof cb == "function") cb(token);
      };
      wx.checkSession({
        success: (res) => {
          // session_key 未过期，并且在本生命周期一直有效
          console.log("在登陆中");
          let token = wx.getStorageSync("token");
          if (token) onUserLogin(token);
        },
        fail: (res) => {
          // session_key已经失效，需要重新执行登录流程
          wx.login({
            success(res0) {
              if (res0.code) {
                requestLoginApi(res0.code);
              } else {
                console.log("登录失败!" + res0.errMsg);
              }
            },
          });
        },
      });
    },
  });
}

export default loginWithCallback;
```

##### Promise化

回调地狱问题，不利于代码的阅读，所以接下来我们基于[Promise](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise)进行代码优化。有了 Promise 对象，就可以将异步操作以同步操作的流程表达出来，避免了层层嵌套的回调函数。此外，Promise 对象提供统一的接口，使得控制异步操作更加容易。

**Promise的几个方法简介**

| 方法名                                                       | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [Promise.prototype.then](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise/then) | 方法返回的是一个新的 Promise 对象，因此可以采用链式写法。这种设计使得嵌套的异步操作，可以被很容易得改写，从回调函数的"横向发展"改为"向下发展"。 |
| [Promise.prototype.catch](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise/catch) | 是 Promise.prototype.then(null, rejection) 的别名，用于指定发生错误时的回调函数。Promise 对象的错误具有"冒泡"性质，会一直向后传递，直到被捕获为止。也就是说，错误总是会被下一个 catch 语句捕获。 |
| [Promise.prototype.finally](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise/finally) | 方法返回一个[`Promise`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise)。在promise结束时，无论结果是fulfilled或者是rejected，都会执行指定的回调函数。这为在`Promise`是否成功完成后都需要执行的代码提供了一种方式。 |
| [Promise.all](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise/all) | 这避免了同样的语句需要在[`then()`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise/then)和[`catch()`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise/catch)中各写一次的情况。Promise.all 方法用于将多个 Promise 实例，包装成一个新的 Promise 实例。Promise.all 方法接受一个数组作为参数，`var p = Promise.all([p1,p2,p3]);`p1、p2、p3 都是 Promise 对象的实例。（Promise.all 方法的参数不一定是数组，但是必须具有 iterator 接口，且返回的每个成员都是 Promise 实例。）p 的状态由 p1、p2、p3 决定，分成两种情况。  （1）只有p1、p2、p3的状态都变成fulfilled，p的状态才会变成fulfilled，此时p1、p2、p3的返回值组成一个数组，传递给p的回调函数。 （2）只要p1、p2、p3之中有一个被rejected，p的状态就变成rejected，此时第一个被reject的实例的返回值，会传递给p的回调函数。 |
| [Promise.race](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise/race) | Promise.race 方法同样是将多个 Promise 实例，包装成一个新的 Promise 实例。`var p = Promise.race([p1,p2,p3]);`上面代码中，只要p1、p2、p3之中有一个实例率先改变状态，p的状态就跟着改变。那个率先改变的Promise实例的返回值，就传递给p的返回值。 |
| [Promise.any](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise/any) | 接收一个Promise可迭代对象，只要其中的一个 promise 成功，就返回那个已经成功的 promise 。所有子实例都处于rejected状态，总的promise才处于rejected状态。 |
| [Promise.allSettled](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled) | 返回一个在所有给定的promise都已经`fulfilled`或`rejected`后的promise，并带有一个对象数组，每个对象表示对应的promise结果。相比之下，`Promise.all()` 更适合彼此相互依赖或者在其中任何一个`reject`时立即结束。 |

##### 小程序API接口Promise化并且把需要登录的调用接口模块化

1. 安装插件。请先查看[npm支持](https://developers.weixin.qq.com/miniprogram/dev/devtools/npm.html)文档。

```bash
npm install --save miniprogram-api-promise
```

2. 在微信开发者工具右方详情中勾选使用npm模块，并在菜单栏工具中点击构建npm。
3. 初始化代码。

```javascript
// app.js
import {promisifyAll} from 'miniprogram-api-promise'
import login from "../lib/login";
const wxp ={}
promisifyAll(wx,wxp)
// 需要token的请求统一处理登录和设置header，并且处理错误信息
wxp.requestNeedLogin = async function (args) {
  let token = wx.getStorageSync("token");
  if (!token) {
    token = await loginWithPromise();
  }
  if (!args.header) args.header = {};
  args.header["Authorization"] = `Bearer ${token}`;
  return wxp.request(args).catch(console.error);
};
// app.js
App({
  wxp:wxp,
});
```

4. 改写login.js代码

```javascript
// lib/login.js
function login() {
  return new Promise((resolve, reject) => {
    // 推荐使用wx.getUserProfile获取用户信息，开发者每次通过该接口获取用户个人信息均需用户确认
    // 开发者妥善保管用户快速填写的头像昵称，避免重复弹窗
    wx.getUserProfile({
      desc: "用于完善会员资料", // 声明获取用户个人信息后的用途，后续会展示在弹窗中，请谨慎填写
       success:async (res0) => {
        let {
          userInfo,
          encryptedData,
          iv
        } = res0;
        const app = getApp();
        try {
          app.wxp.checkSession();
        } catch (err) {
          reject(err);
        }
        let token = wx.getStorageSync("token");
        if (!token) {
          let res1 = await app.wxp.login().catch(err => reject(err));
          let code = res1.code;
          let res = await app.wxp.request({
            url: "http://localhost:3000/user/weixin-login",
            method: "POST",
            header: {
              "content-type": "application/json",
            },
            data: {
              code,
              userInfo,
              encryptedData,
              iv,
            }
          }).catch(err => reject(err));
          token = res.data.data.authorizationToken;
          wx.setStorageSync("token", token);
          app.globalData.token = token;
          wx.showToast({
            title: "登录成功了",
          });
          resolve(token);
        }
      },
    });
  })
}

export default login;
```

5. 调用代码

```html
<view class="container page-head">
  <text class="page-section__title">需要登录的请求调用</text>
  <view class="btn-area">
    <button bindtap="request1" type="primary">请求1</button>
    <button bindtap="request2" type="primary">请求2</button>
  </view>
</view>
```

```javascript
// pages/index.js
Page({
  /**
   * 页面的初始数据
   */
  data: {},
  request1() {
    getApp().wxp.requestNeedLogin({
        url: "http://localhost:3000/user/home?name=andying",
      }).then(console.log)
  },
  request2() {
    getApp().wxp.requestNeedLogin({
        url: "http://localhost:3000/user/home?name=eva",
      }).then(console.log)
  },
});
```
