+++
title = "使用WePY进行小程序开发"
date = "2019-03-28"
tags = [
    "小程序",
    "wepy",
]
categories = [
    "小程序开发",
]
+++

一直以来开发小程序都是中规中矩的使用微信小程序开发工具，原生开发小程序。最近偶然发现了一个腾讯出品的小程序开发框架WePY，是对小程序做了进一步的封装，看起来亮点多多。
## 优势
- 使用类似于目前最为火爆的前端框架Vue.js的开发风格，前端开发者可以轻松掌握
- 支持使用第三方的NPM资源，可使用的组件更为丰富
- 支持最新的JS语法，支持ES6、ES7的若干新特性
- 单文件模式，不再像原生小程序那样一个页面需要四个文件

WePY的优势还在于相比其他类似的小程序开发框架，它的开源时间较早，有极其丰富的组件库，也有很多的Demo可供参考，比如[微信小程序wepy框架开发资源汇总](https://github.com/aben1188/awesome-wepy)。

## 开始安装
1. 全局安装WePY命令行工具wepy-cli，如果有权限问题请在前面加上sudo。
``` bash
npm install wepy-cli -g
```
2. 使用`wepy init`创建WePY项目。
``` bash
// 使用空模板创建
wepy init empyt myproject
// 使用基础模板创建
wepy init standard myproject
// 使用其他GitHub上的demo
wepy init wepyjs/wepy-wechat-demo myproject
```
3. 在项目目录下安装依赖
``` bash
cd myproject
npm install
```
4. 运行项目
``` bash
// 测试(该命令下会运行 wepy build --watch)
npm run dev
// 正式(项目包会减小,该命令下会运行 cross-env NODE_ENV=production wepy build --no-cache)
npm run build
```
执行上面的命令后，再项目根目录会生成一个叫做`dist`的文件夹，用微信开发者工具打开这个目录，里面就是典型的小程序架构的项目了。
编译小程序项目前记得要把一些设置关掉，如关闭ES6转ES5选项，关闭代码样式自动补全选项，关闭代码压缩上传选项，开启不检查域名选项。
![](https://upload-images.jianshu.io/upload_images/1587104-e0be9e403cd46903.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
## IDE配置
在IDE方面我使用的口碑较好的WebStorm来进行WePY开发。默认情况下WebStorm是不识别WePY类型的文件的，为了使其实现对wpy文件的渲染，如下图所示，打开WebStorm的偏好设置，在Vue.js的Template里面添加wpy后缀的识别。
![](https://upload-images.jianshu.io/upload_images/1587104-d6c5cbb5095650b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
另外也推荐使用VS Code去进行WePY开发，VS Code是目前最流行的代码编辑器之一，有很丰富的插件扩展，比如Vetur-wepy插件就直接对wpy文件进行了正确的渲染，minapp插件对小程序的标签、属性进行了正确的补全等等。如果喜欢轻量级编辑器的朋友可以选择这款只有几十M的VS Code。

## 相关文件配置
- #### package.json
  - scripts: WePY项目可执行的npm scripts,默认有dev, build, test三个命令，可以通过`npm run dev/build/test`来执行。比如`npm run dev`默认情况下就是执行的`wepy build --watch`命令，`npm run build`就是执行的`cross-env NODE_ENV=production wepy build --no-cache`命令。
    > 建议把dev命令改为`cross-env NODE_ENV=development wepy build --watch`，这样在使用`npm run dev`或者`npm run build`时，在wepy.config.js中可以通过process.env.NODE_ENV拿到不同的环境变量值。
  
  - dependencies: 项目所依赖的模块
  - devDependencies: 只下载使用某些模块，而不下载这些模块的测试和文档框架
- #### package-lock.json
所有node_modules里面的模块的版本、地址、依赖等相关信息。它和package.json的关系就类似于iOS项目里面podfile和podfile.lock文件的关系。
- #### project.config.json
  - setting: 对应标准小程序项目里面的project.config.json文件里面的setting设置。es6为是否开启es6转es5，postcss为上传代码时是否自动代码补全，minified为上传代码时是否代码压缩，urlCheck为是否检查安全域名和TLS版本。
  - miniprogramRoot: 编译的微信小程序项目所在目录，默认为"./dist"。
- #### wepy.config.js
  - wpyExt: WePY文件的后缀名，默认为.wpy，如果改为.vue，同样能解决页面文件代码高亮的问题。
  - compilers: 支持[sass](https://github.com/sass/node-sass), [less](http://lesscss.org/#using-less-usage-in-code), [postcss](https://www.html.cn/archives/7317), [babel](https://babeljs.io/docs/en/options), [stylus](https://www.zhangxinxu.com/jq/stylus/js.php), [typescript](https://www.tslang.cn/docs/home.html)语法的使用。