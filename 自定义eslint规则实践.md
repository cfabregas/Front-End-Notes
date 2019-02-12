# 自定义eslint规则实践

## 动机
年前 smart 项目组分拆智适应和重构代码的过程中，发现了一个引用静态资源的问题，示例代码如下：
```jsx
render () {
  return (
    <img src='/images/test.png' />
  )
}
```
为什么说这是一个问题，是因为以下的原因：
- 项目中对所有的“引用到的”图片资源都在 `webpack` 构建的过程中通过 `url-loader` 做了处理，为每一个文件名加上了哈希值，上传到OSS服务器。只要图片改变（即使图片的文件名没变）就会改变哈希值，用户在访问的时候需要重新请求服务器，平时使用本地缓存即可
- 对于 `webpack` 来讲，他所认为的“引用到的”的图片资源，只包含以下引用方式：
```js
// es6
import image from 'images/test.png'
```
```js
// es5
retuire('images/test.png')
```
```css
// css
div {
  background: url('~images/test.png')
  background: url('./images/test.png') /* 经过 css-loader 解析 */
}
```
