# 自定义eslint规则实践

## 动机
年前 smart 项目组分拆智适应和重构代码的过程中，发现了一个引用静态资源的问题：
```jsx
render () {
  return (
    <img src='/images/test.png' />
  )
}
```
