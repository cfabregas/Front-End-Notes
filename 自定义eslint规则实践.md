# 自定义eslint规则实践


## 暴露的问题

年前 smart 项目组分拆智适应和重构代码的过程中，发现了一个引用静态资源的问题，示例代码如下：
```jsx
render () {
  return (
    <img src='/images/test.png' />
  )
}
```
为什么说这是一个问题，是因为以下的原因：
- 我们对项目中所有的 *引用到的* 图片资源都在 `webpack` 构建的过程中通过 `url-loader` 做了处理，为每一个文件名加上了哈希值，部署在CDN服务器。只要图片改变（即使图片的文件名没变）就会改变哈希值，用户在访问的时候需要重新请求服务器，平时则使用本地缓存即可
- 对于 `webpack` 来讲，他所认为的 *引用到的* 的图片资源，通常包含以下引用方式：
```jsx
// ES6
import image from 'images/test.png'
<img src={image} />

// CommonJS
<img src={require('images/test.png')} />
```
```css
// css
div {
  background: url('~images/test.png');
  background: url('./images/test.png'); /* 经过 css-loader 解析 */
}
```
可见一开始出现的纯文本的引用方式，不会让 `webpack` 意识到这是一个静态文件，他也不会做任何处理。


## 之前的处理

在这种情况下，项目为了兼容这一类情况，引入了 `CopyWebpackPlugin` 插件，粗暴地将整个静态资源文件夹直接拷贝了一份到项目的输出文件夹并上传服务器，因为你没法知道到底那些文件被 `img` 标签悄悄地引用到了。

这种做法明显是重复劳动，降低了 `webpack` 打包的性能不说，更重要的是用户端在缓存过期前无法检测到图片内容已经更新（MD5值变化），图片名却没变的情况，还在傻傻的用着本地缓存。


## 提出的方案

解决这个问题最好的方案无疑是：
> 强制所有的 `img` 标签在引用静态资源时必须通过 `require()` 的方式引入

这样能一举解决上述的问题，而且在构建时连 `CopyWebpackPlugin` 都省掉了。问题在于，要规范所有的团队成员的写法，口头约定肯定是不可靠的，我们不能这样做。

当然，主要还是因为太low。


## 那就手写一条eslint规则吧


## 基础知识储备： AST

`AST` (抽象语法树，Abstract Syntax Tree)，你可以理解为这是 `JavaScript` 世界的最底层了，再往下，就是关于转换和编译的“黑魔法”领域了，告辞！

`eslint` 规范代码的运行原理，是在预编译阶段将代码解析成 `AST` ，并遍历它做静态分析。想知道一段代码解析成 `AST` 后长什么样，可以借助这个网站在线查看：
> [https://astexplorer.net/](https://astexplorer.net/)

在这之前我也没有接触过 `AST` ，所以我需要参考一些文档，然后现学现卖，大约是以下几个：
- [eslint 官方文档：创建规则](https://cn.eslint.org/docs/developer-guide/working-with-rules)
- [前端代码质量进阶：自定义 eslint 规则校验业务逻辑](https://segmentfault.com/a/1190000014684778)

## 考虑到的情况

上面提到的文档十分清晰和友好，所以相关的基础概念我也不再赘述，直接从业务角度上来讲，这条规则应该怎么实现。

实际的业务代码中，静态图片的引用方式可能是多变的，比如以下几种方式必须都要覆盖到：
```jsx
render () {
  const { test } = this.state

  return (
    <div>
      {/* 字面量 */}
      <img src='/images/test.png' />

      {/* 模板语法 */}
      <img src={`/images/test.png`} />
      <img src={`/images/${test}.png`} />

      {/* 与或表达式 */}
      <img src={test || '/image/test.png'} />
      <img src={test && '/image/test.png'} />

      {/* 三元表达式 */}
      <img src={test ? test : '/images/test.png'} />
    </div>
  )
}
```

很明显，包括但不限于上述的引用方式，都是不合法的，都需要筛选出来并予以警告。之所以说不限于，是因为这里没法列举所有可能性，因为比如模板语法可以实现嵌套，三元表达式也可能有多层判断，这些都需要用递归的方式深挖，不能轻易放过。

## 项目代码展示

[use-images-by-require](https://github.com/zhike-team/eslint-plugin-zhike/blob/master/rules/use-images-by-require.js)
