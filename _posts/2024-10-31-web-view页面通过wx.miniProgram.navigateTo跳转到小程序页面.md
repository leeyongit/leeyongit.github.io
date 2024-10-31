---
title: 小程序和H5之间互相跳转实现方法 
categories: [小程序]
tags: [web-view]
---

### 小程序和H5之间的跳转实现方法

---

### 1. 小程序内打开H5页面

在小程序中打开H5页面只需使用 `web-view` 标签。例如：

```html
<web-view src="https://www.baidu.com" bindload="bindload" binderror="binderror"></web-view>
```

### 2. H5跳转到小程序

#### 使用明文 Scheme 拉起小程序

1. 开发者无需调用平台接口。在小程序账号设置 -> 隐私与安全 -> 明文 Scheme 拉起小程序声明后，可根据以下格式拼接 `appid` 和 `path` 参数生成 URL Scheme 链接。**注意**：通过明文 URL Scheme 打开小程序的页面 `path` 必须是已发布的小程序页面，不可携带 `query` 参数，这里以首页为例，通过首页接收参数。

2. 在H5页面，通过按钮点击跳转到小程序，代码如下：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=0, minimum-scale=1.0, maximum-scale=1.0">
    <title>收银台</title>
    <script src="https://cdn.staticfile.org/jquery/1.10.2/jquery.min.js"></script>
    <style>
        * { margin: 0; padding: 0; }
        .a01 {
            width: 250px;
            padding: 8px 10px;
            border: 1px solid #999;
            font-size: 16px;
            margin: 20px auto;
            display: block;
            background: #036DE7;
            color: #fff;
            text-decoration: none;
            border-radius: 5px;
        }
    </style>
</head>
<body>
    <a class="a01" href="weixin://dl/business/?appid=wx54411a4fcada6129&path=pages/pay/wx-pay&query=order_sn=YR202410292323152767053607&env_version=trial">
        进入体验版
    </a>
    <a class="a01" href="weixin://dl/business/?appid=wx54411a4fcada6129&path=pages/pay/wx-pay&query=order_sn=YR202410292323152767053607">
        进入正式版
    </a>
</body>
<script>
    function getQueryParams() {
        const params = {};
        const queryString = window.location.search.substring(1);
        const regex = /([^&=]+)=([^&]*)/g;
        let match;
        
        while (match = regex.exec(queryString)) {
            params[decodeURIComponent(match[1])] = decodeURIComponent(match[2]);
        }
        
        return params;
    }

    const urlParams = getQueryParams();
    const obj = {
        flag: 1,
        regionId: 1006,
        schoolid: 61,
        ...urlParams
    };    

    let queryString = Object.keys(obj).map(key => `${encodeURIComponent(key)}=${encodeURIComponent(obj[key])}`).join('&');
    const res = encodeURIComponent(queryString);
    const newhref = 'weixin://dl/business/?appid=wx54411a4fcada6129&path=pages/pay/wx-pay&env_version=trial&query=' + res;

    window.location.href = newhref;
</script>
</html>
```

> **注意**：将“小程序的appid”替换为目标小程序的 `appid`。多个参数可放入一个 JSON 对象中，并根据实际业务需求动态设置。把代码放入接口请求成功回调中可实现携带动态参数跳转。

#### 小程序页面代码示例

在小程序页面接收参数并跳转：

```javascript
onLoad: function (e) {
    console.log("onLoad首页页面地址获取的参数e:", e);
    let regionId = e.regionId;
    let flag = e.flag;
    let schoolid = e.schoolid;

    if (flag == "1") {
        console.log("从H5页面跳转过来，进入支付页面");
        uni.redirectTo({
            url: "/pages/pay/wx-pay?regionId=" + regionId + "&flag=" + flag + "&schoolid=" + schoolid,
        });
    } else {
        console.log("从小程序首页进入");
    }
}
```

### 3. 小程序内的H5返回小程序

#### 在 `web-view` 页面通过 `wx.miniProgram.navigateTo` 跳转到小程序页面

在 H5 页面引入微信 JSSDK：

```html
<script src="https://res.wx.qq.com/open/js/jweixin-1.3.2.js#wechat_redirect"></script>
```

> **注意**：`#wechat_redirect` 用于解决 iOS 端 JSSDK 调用无响应的问题。

在需要跳转的事件中调用：

```javascript
wx.miniProgram.navigateTo({
    url: '/pages/ad/index'
});
```

---

#### 其他提示

1. 网页内 `iframe` 的域名需配置在白名单中。
2. 开发者工具中，可通过 `web-view` 组件右键 - 调试，打开调试界面。
3. 每个页面只能包含一个 `web-view`，且会自动撑满页面。
4. `web-view` 页面与小程序仅支持通过 JSSDK 提供的接口进行通信。
5. 避免链接中包含中文字符，以防 iOS 打开页面时出现白屏问题，建议使用 `encodeURIComponent` 编码。