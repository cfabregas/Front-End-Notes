# 题库项目HMR实践


## 动机

题库组的项目目前没有热更新模块，每次更新代码，想看效果的时候都需要刷新页面，不胜其扰，决定给题库的项目加上热更新模块，以提高开发效率。
实践的过程中，虽然没有遇到太大的困难，但也绕了一点弯子，所以觉得有必要做一个记录。


## 第一步：官网文档

- 按照惯例，直接上 `webpack` 官网查热模块替换的文档：[热模块替换 | webpack 中文网](https://www.webpackjs.com/guides/hot-module-replacement/)。
- 最简单的情况下，在 `webpack-dev-server` 环境下运行的项目，添加 `HMR` 就很轻松，但这并不适用于题库项目。
- 因为我们的项目是通过 `webpack-dev-middleware` 运行在自定义服务器上的，所以我们要绕一点弯子：[webpack-hot-middleware](https://github.com/webpack-contrib/webpack-hot-middleware)。


## 第二步：Webpack Hot Middleware

- 这是一个 `webpack` 插件包，首先我们安装这个插件：
```bash
npm install --save-dev webpack-hot-middleware
```
- 在dev环境下的webpack配置文件下添以下配置：
```js
entry: ['./src/index.js', 'webpack-hot-middleware/client'],
plugins: [
  new webpack.HotModuleReplacementPlugin(),
]
```
- 在dev环境下的server配置文件下添以下配置：
```js
var webpack = require('webpack')
var webpackConfig = require('./webpack.config')
var compiler = webpack(webpackConfig)

app.use(require("webpack-dev-middleware")(compiler, {
  noInfo: true,
  publicPath: webpackConfig.output.publicPath,
}))
app.use(require("webpack-hot-middleware")(compiler))
```
- 运行代码，页面并没有更新，而是出现了一个 `warning` ：
```plain text
[HMR] The following modules couldn't be hot updated: (Full reload needed)
This is usually because the modules which have changed (and their parents) do not know how to hot reload themselves. See https://webpack.js.org/concepts/hot-module-replacement/ for more details.
```
- 这是因为我们的项目是由 `React` 构建的，在触发热更新的时候， `webpack` 只更新js文件，并不会执行 `React` 的生命周期，所以无法触发视图层的重新渲染。
- 但是来都来了，就再绕一个弯吧：[react-hot-loader](https://github.com/gaearon/react-hot-loader/)。


## 第三步：React Hot Loader

- 这是一个 `React` 高阶组件，作用是将组件暴露给 `HMR` 模块，实时更新 `React` 组件。首先照例安装（注意这个组件接下来会在项目中引入，所以应该作为常规依赖而非dev依赖。并且可以放心的是，这个组件不会再生产环境下生效。）：
```bash
npm install --save react-hot-loader
```
- 在项目的 `.babelrc` 文件中添加如下配置：
```js
// .babelrc
{
  "plugins": ["react-hot-loader/babel"],
}
```
- 修改项目的根组件，使其暴露：
```js
// App.js
import React from 'react'
import { hot } from 'react-hot-loader'

class App extends React.Component {
  ... // come codes
}

export default hot(module)(App)
```
- 运行代码：emmm...看上去没什么问题的样子。


## 第四步：优化

- 这个时候项目其实已经可以正常运作了，然而每次热更新， `React` 都会报一个让强迫症无法忍受的的小 `error` ：
```plain text
You cannot change <Router history>
```
- 虽然不知道为什么，但是不怕，我们有 `Stack Overflow` 。经过热心网友的指点我们得知， `Redux` 和 `React Redux` 在[更新文档](https://github.com/reduxjs/react-redux/releases/tag/v2.0.0)中强调，2.x之后就不再支持热更新了，因为会引起某些难以预测的问题。
- 所以，我们在根组件中启用热更新，无疑会暴露整个 `store` 和 `Router` ，那就再优化一下吧，来都来了。
- 我尝试的优化方法是：只暴露路由文件下的 `Switch` 组件，毕竟根目录和路由文件在项目的日常维护的迭代中，大多数时候都不会去修改：
```js
// routers/index.js
import { hot } from 'react-hot-loader'

const Routers = () => {
  <Switch>
    <Route {...Props} />
    ...other Router
  </Switch>
}

const HotSwitch = hot(module)(Routers)

export const AppRouter = () => (
  <Router history={history}>
    <HotSwitch />
  </Router>
)
```
- 运行代码：稳！
