# React State Hooks的闭包陷阱，在使用Hooks之前必须掌握

先看以下代码

```jsx
import React from 'react'
import ReactDOM from 'react-dom'

const buttonStyles = {
  border: '1px solid #ccc',
  background: '#fff',
  fontSize: '2em',
  padding: 15,
  margin: 5,
  width: 200,
}
const labelStyles = {
  fontSize: '5em',
  display: 'block',
}

function Stopwatch() {
  const [lapse, setLapse] = React.useState(0)
  const [running, setRunning] = React.useState(false)

  React.useEffect(
    () => {
      if (running) {
        const startTime = Date.now() - lapse
        const intervalId = setInterval(() => {
          setLapse(Date.now() - startTime)
        }, 0)
        return () => clearInterval(intervalId)
      }
    },
    [running],
  )

  function handleRunClick() {
    setRunning(r => !r)
  }

  function handleClearClick() {
    setRunning(false)
    setLapse(0)
  }

  return (
    <div>
      <label style={labelStyles}>{lapse}ms</label>
      <button onClick={handleRunClick} style={buttonStyles}>
        {running ? 'Stop' : 'Start'}
      </button>
      <button onClick={handleClearClick} style={buttonStyles}>
        Clear
      </button>
    </div>
  )
}

function App() {
  const [show, setShow] = React.useState(true)
  return (
    <div style={{textAlign: 'center'}}>
      <label>
        <input
          checked={show}
          type="checkbox"
          onChange={e => setShow(e.target.checked)}
        />{' '}
        Show stopwatch
      </label>
      {show ? <Stopwatch /> : null}
    </div>
  )
}

ReactDOM.render(<App />, document.getElementById('root'))
```

这里的代码想要实现的功能如下：

1 点击 Start 开始执行 interval，并且一旦有可能就往 lapse 上加一

2 点击 Stop 后取消 interval

3 点击 Clear 会取消 interval，并且设置 lapse 为 0

但是这个例子在实际执行过程中会出现一个问题，那就是在 interval 开启的情况下，直接执行 clear，会停止 interval，但是显示的 lapse 却不是 0，那么这是为什么呢？

出现这样的情况主要原因是：useEffect 是异步的，也就是说我们执行 useEffect 中绑定的函数或者是解绑的函数，**都不是在一次 setState 产生的更新中被同步执行的。**啥意思呢？我们来模拟一下代码的执行顺序：

在我们点击来 clear 之后，我们调用了 setLapse 和 setRunning，这两个方法是用来更新 state 的，所以他们会标记组件更新，然后通知 React 我们需要重新渲染来。

然后 React 开始来重新渲染的流程，并很快执行到了 Stopwatch 组件。

注意以上都是同步执行的过程，所以不会存在在这个过程中 setInterval 又触发的情况，所以在更新 Stopwatch 的时候，如果我们能同步得执行 useEffect 的解绑函数，那么就可以在这次 JavaScript 的调用栈中清除这个 interval，而不会出现这种情况。

是恰恰因为 useEffect 是异步执行的，他要在 React 走完本次更新之后才会执行解绑以及重新绑定的函数。那么这就给 interval 再次触发的机会，这也就导致来，我们设置 lapse 为 0 之后，他又在 interval 中被更新成了一个计算后的值，之后才被真正的解绑。

那么我们如何解决这个问题呢？

## 使用 useLayoutEffect

useLayoutEffect 可以看作是 useEffect 的同步版本。使用 useLayoutEffect 就可以达到我们上面说的，在同一次更新流程中解绑 interval 的目的。

那么同学们肯定要问了，既然 useLayoutEffect 可以避免这个问题，那么为什么还要用 useEffect 呢，直接所有地方都用 useLayoutEffect 不就好了。

这个呢主要是因为 useLayoutEffect 是同步的，如果我们要在 useLayoutEffect 调用状态更新，或者执行一些非常耗时的计算，可能会导致 React 运行时间过长，阻塞了浏览器的渲染，导致一些卡顿的问题。这块呢我们有机会再单独写一篇文章来分析，这里就不再赘述。

## 不使用 useLayoutEffect


