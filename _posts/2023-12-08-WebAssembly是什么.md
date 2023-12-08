---
title: WebAssembly是什么
categories: [WebAssembly]
tags: [WebAssembly]
---


2019 年 12 月 5 日，WebAssembly正式加入 HTML、CSS 和 JavaScript 的 Web 标准大家庭。很多事情都会受益于这一全新的标准，并且它在浏览器中的性能表现是空前的。此前，开发者迈赫迪·扎伊德（Mehdi Zed）发文简要介绍了这个正在进行中的小变革，对此行了翻译，希望能给你带来启发。

## WebAssembly的诞生背景
1995 年，布伦丹·艾希（Brendan Eich）用了不到 10 天就创建了 JavaScript。那时，JavaScript 的设计并非以速度见长。它基本上是用于表单验证的，同时速度非常缓慢。随着时间的流逝，它也在一天天变好。

在 2008 年，异军突起的谷歌推出了自己全新的浏览器 Chrome。Chrome 内部有一个名为 V8 的 JavaScript 引擎，V8 的革命性进步是对 JavaScript 的即时（JIT）编译。从解释代码到 JIT 编译的这种转变大幅提升了 JavaScript 的性能，从而让整个浏览器的速度变得飞快。如此快的速度将催生 Node.js或 Electron 等技术，并推动 JavaScript 迎来爆炸式增长。

在 2015 年，WebAssembly首次发布，并提供了一个运行在 Unity 下的游戏的小型演示。这款游戏是直接在浏览器中运行的。

在 2019 年，W3C 使 WebAssembly成为了新的 Web 标准。就像 V8 引擎一样，WebAssembly即将带来全新的性能革命。它的身影已经出现在了 Web 的赛道上，枪声一响便遥遥领先。

## 什么是 WebAssembly？
WebAssembly（缩写为 wasm）是一种使用非 JavaScript 代码，并使其在浏览器中运行的方法。这些代码可以是 C、C++ 或 Rust 等。它们会被编译进你的浏览器，在你的 CPU 上以接近原生的速度运行。这些代码的形式是二进制文件，你可以直接在 JavaScript 中将它们当作模块来用。

WebAssembly不能替代 Javascript，相反，这两种技术是相辅相成的。通过 JavaScript API，你可以将 WebAssembly模块加载到你的页面中。也就是说，你可以通过 WebAssembly来充分利用编译代码的性能，同时保持 JavaScript 的灵活性。

WebAssembly这个名字有点误导人，它确实适用于 Web，但它的使用场景远不止于此。此外，WebAssembly不是编程语言，它是一种中间格式，叫字节码，可以作为其他语言的编译目标。

## 它是如何工作的？
- 第一步：你使用 C、C++ 或其他语言生成源代码，这段代码应该可以解决某个问题，或者完成某段对浏览器中的 JavaScript 来说太过复杂的流程。

- 第二步：使用 Emscripten 将你的源代码编译为 WebAssembly，这一步完成时，你将得到一个 wasm文件。

- 第三步：你将在网页上使用这个 wasm文件，将来你可以像其他 ES6 模块一样加载这个文件。

下面以简单的代码示例来说明这个过程：

首先，你需要一小段 C++ 代码来编译。以下是一个添加两位数的函数：
```c++
int add(int firstNumber, int secondNumber) {
  return firstNumber + secondNumber;
}
```
接着转到你选择的 Linux 发行版。第一步是下载并安装 Emscripten：
```sh
# 安装依赖项（是的，你可以使用 python 的较新版本）
sudo apt-get install python2.7 git
# 通过一个 git 克隆获取 emscripten
git clone https://github.com/emscripten-core/emsdk.git
# 下载，安装并激活 sdk
cd emsdk
./emsdk install latest
./emsdk activate latestl
source ./emsdk_env.sh
# 确认安装的内容可以正常运行
emcc --version
# 将这个 c++ 文件编译到一个 webassembly 模板
emcc helloWebassembly.cpp -s WASM=1 -o helloWebassembly.html
# 启动 HTML 并观察结果
emrun helloWebassembly.html
```
如代码所示，这是极客处理 wasm 文件的办法。还有一种更简单的方法，你可以在站点查看。

接下来，将你的 C++ 代码放在左侧。然后，你将在 WAT 部分中获得导出函数的名称（_Z3addii），你只需点进 download ，就可以获取 wasm文件，非常简单。

现在，你就可以让 WebAssembly直接运行在浏览器中了，WebAssembly对象在支持的浏览器中会自动挂在在window中，非常简单。
```html
<html>
  <head>
    <title>WASM test</title>
    <link rel="stylesheet" href="/stylesheets/style.css" />
  </head>

  <body>
    <script>
      const getRandomNumber = () => Math.floor(Math.random() * 10000);

      window.WebAssembly && WebAssembly.instantiateStreaming(
        fetch("http://localhost/add.wasm")
      )
        .then(obj => obj.instance.exports._Z3addii)
        .then(add => {
          document.getElementById("addTarget").textContent = add(
            getRandomNumber(),
            getRandomNumber()
          );
        });
    </script>
    <h1>Résultat du C++</h1>
    <p id="addTarget"></p>
  </body>
</html>
```