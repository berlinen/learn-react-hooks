# 来讲讲React Hooks是怎么实现的?

React 16.7-alpha 中新增了功能：Hooks 总结他的功能就是：让<mark>FunctionalComponent</mark>具有<mark>ClassComponent</mark>的功能。

```js
import React, { useState, useEffect } from 'react'

function FuunComp(props) {
  const [ data, setData ] = useState('initialState')

  function handleChange(e) {
      setData(e.target.value)
  }

  useEffect(() => {
      subscribeToSomething()

      return () => {
        unSubscribeToSomething()
      }
  })

  return (
    <input value={data} onChange={handleChange} />
  )
}
```

### 按照Dan的说法，设计Hooks主要是解决ClassComponent的几个问题：

1 很难复用逻辑（只能用HOC，或者render props），会导致组件树层级很深

2 产生巨大的组件（指很多代码必须写在类里面）

3 类组件很难理解，比如方法需要bind，this指向不明确

```js
export default withStyle(style)(connect(/*something*/)(withRouter(MyComponent)))
```

这就是一个4层嵌套的HOC组件

同时，如果你的组件内事件多，那么你的constructor里面可能会酱紫：

```js
class MyComponent extends React.Component {
    constructor () {
         // initiallize

    this.handler1 = this.handler1.bind(this)
    this.handler2 = this.handler2.bind(this)
    this.handler3 = this.handler3.bind(this)
    this.handler4 = this.handler4.bind(this)
    this.handler5 = this.handler5.bind(this)
    // ...more
}
```
虽然最新的class语法可以用handler = () => {}来快捷绑定，但也就解决了一个声明的问题，整体的复杂度还是在的。





