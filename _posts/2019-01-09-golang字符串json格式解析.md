---
title: golang字符串json格式解析
categories: [后端, Golang]
tags: [golang, json]
---

最近在用golang写关于微信方面的东西，首先遇到的就是将字符串转换成golang的json格式，利用corpid和corpsecret返回的是一个json格式的字符串，其格式如下：

```json
{"access_token":"uAUS6o5g-9rFWjYt39LYa7TKqiMVsIfCGPEN4IZzdAk5-T-ryVhL7xb8kYciuU_m","expires_in":7200}
```
我们可以把它转换成一个map[string]interface{}类型的数据，相关代码如下：

```go
str := "{\"access_token\":\"uAUS6o5g-9rFWjYt39LYa7TKqiMVsIfCGPEN4IZzdAk5-T-ryVhL7xb8kYciuU_m\",\"expires_in\":7200}"
var dat map[string]interface{}
if err := json.Unmarshal([]byte(str), &dat); err == nil {
    fmt.Println(dat)
    fmt.Println(dat["expires_in"])
} else {
    fmt.Println(err)
}
```
运行的结果如下：

```sh
map[access_token:uAUS6o5g-9rFWjYt39LYa7TKqiMVsIfCGPEN4IZzdAk5-T-ryVhL7xb8kYciuU_m expires_in:7200]
7200
 ```

我们还可以定义一个结构体，将数据转换成对应的结构体对象，再获取相应的数据，定义一个weixintoken结构体：

```sh
type weixintoken struct {
    Tokens string `json:"access_token"`
    Expires int `json:"expires_in"`
}
```
注意相应变量首字母的大小写(首字母小写不可见，大写可见，具体查看golang的变量相关的内容)，将JSON绑定到结构体,结构体的字段一定要大写,否则不能绑定数据。

```go
ret := "{\"access_token\":\"uAUS6o5g-9rFWjYt39LYa7TKqiMVsIfCGPEN4IZzdAk5-T-ryVhL7xb8kYciuU_m\",\"expires_in\":7200}"
var config weixintoken
if err := json.Unmarshal([]byte(ret), &config); err == nil {
    fmt.Println(config)
    fmt.Println(config.Tokens)
} else {
    fmt.Println(err)
}

```
运行结果如下：

```sh
{"access_token":"uAUS6o5g-9rFWjYt39LYa7TKqiMVsIfCGPEN4IZzdAk5-T-ryVhL7xb8kYciuU_m","expires_in":7200}
{uAUS6o5g-9rFWjYt39LYa7TKqiMVsIfCGPEN4IZzdAk5-T-ryVhL7xb8kYciuU_m 7200}
```