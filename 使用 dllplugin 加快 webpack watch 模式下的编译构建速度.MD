# 使用 dllplugin 加快 webpack watch 模式下的编译构建速度

<a name="17eeb4f1"></a>
## dllplugin 是什么
如果对 dllplugin 还不是很了解，可以先看一下 [官方文档](https://webpack.js.org/plugins/dll-plugin/#src/components/Sidebar/Sidebar.jsx)。它最主要的是引入了**动态链接库**的思想。这个动态链接库里包含了多个模块，这些模块只需要编译一次，在之后的构建过程中被动态链接库包含的模块将不会再重新编译，而是直接使用动态链接库中的代码。

<a name="af5c6f4f"></a>
## 为什么可以提升速度
如果你有对自己的项目使用 [webpack-bundle-analyzer ](https://github.com/webpack-contrib/webpack-bundle-analyzer)插件进行性能分析的话，就会发现，一般最耗性能的就是第三方库(即 `node_modules` 下的文件)。而这些库我们开发过程中是不会去改动的，也就是说，这些库只需要编译一次即可。所以，这些第三方库是最适合放到**动态链接库**里的。当这些最耗性能的文件不再需要每次都重新编译的时候，速度自然而然就会提升上去了。

<a name="autodll-webpack-plugin"></a>
## autodll-webpack-plugin
如果你看了官方文档 dllplugin 的使用之后，就会发现，配置流程相当繁琐。而 [autodll-webpack-plugin](https://github.com/asfktz/autodll-webpack-plugin) 就是用来简化流程的，只需要在 `webpack.config.js` 配置一个插件即可。下面贴一个我自己项目里的实际配置

<a name="b6613193"></a>
#### weback 配置信息
```javascript
// webpack.config.js
const AutoDllPlugin = require('autodll-webpack-plugin');

module.exports = {
  entry: // ...
  output: // ...
  plugins: [
    new AutoDllPlugin({
  		entry: { // vendor 文件入口
        vendor: [path.resolve(__dirname, 'src/vendor')],
      },
      filename: 'vendor.js',
      plugins: [ // vendor 构建过程需要的 plugins 在这里注册
        new ExtractTextPlugin({
          filename: 'vendor.css',
          allChunks: true,
        }),
      ],
      inherit: true, // 继承 webpack.config.js 里的配置(比如 loaders 等，但不包含 plugins)
    }),
 	],
```

<a name="vendor.js"></a>
#### vendor.js
也就是**动态链接库**，引入项目里用到的公用库

```javascript
// vendor.js
import 'react';
import 'dva';
import 'dva/router';
import 'dva/fetch';
import 'dva-loading';
import 'query-string';
import 'antd';
import '@alife/oui-group-relationship';
import '@ali/tracker';
import "immutability-helper";
import "react-dnd";
import 'moment';
import 'react-dnd-html5-backend';
import 'antd/lib/locale-provider/zh_CN';
import '@ali/sk-editor';
import '@ali/sk-editor/ExtractExt/config';
import '@ali/sk-editor/utils/beautify';
import 'react-monaco-editor';
```

<a name="layout.html"></a>
#### layout.html
用户访问的页面，单独引入 vendor.css 和 vendor.js
```html
<!doctype html>
<html>
  <head>
    <link href="/public/vendor.css" rel="stylesheet">
  </head>
  <body>
    <script src='/public/vendor.js'></script>
  </body>
</html>

```
<a name="d41d8cd9"></a>
## 

<a name="d41d8cd9"></a>
#### 
