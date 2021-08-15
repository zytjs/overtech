---
title: useEffect 的全部
date: '2021-07-27'
spoiler: 一文搞懂 useEffect
cta: 'react'
---

## effect 的执行时机
在浏览器完成 layout 和 paint 之后，延迟执行。这样设计不会延迟 UI 的展示，更快的展示 UI。
这点和类组件的 componentDidMount 不同，它会在浏览器绘制前执行。Hooks刚出现时，有人使用在 useEffect 中打日志的方式[测试性能](https://medium.com/@dan_abramov/this-benchmark-is-indeed-flawed-c3d6b5b6f97f)，会发现比 componentDidMount 执行的慢，原因就在于此。
>？？？：react 是怎么知道浏览器布局和绘制完成了呢，定时查询？所以会有**延迟**执行？

## 在任意一次渲染中，props 和 state 是始终保持不变的
也就是说在不同渲染中的 props 和 state 是相互独立的，使用到它们的任何值也是独立的（包括事件处理函数）。它们都属于一次特定的渲染，即使是事件处理函数中的异步函数，其中使用的值也是当次渲染的值。不会因为延迟使用最新的状态值。
来用一个实际的情况佐证一下：
```jsx
function Counter () {
  const [count, setCount] = useState(0)

  function onBtnClick = () => {
    setTimeout(() => {
      alert('count ' + count)
    }, 1000)
  }

  return (
    <div>
      <button onClick={() => {setCount(count + 1)}}>{count}</button>
      <button onClick={onBtnClick}>alert</button>
    </div>
  )
}
```
首先连续点击按钮一 3 下，然后点击按钮二 1 下，再点击按钮一 2 下。你猜最后弹窗的 count 值是多少？

如果你的答案是 5，那就错了，一起来看一下。
连续点击第一个按钮 3 下时，count 变为了 3，此时调用的 onBtnClick 获取到的 count 是 3。当再次点击第一个按钮，整个 Counter 函数会重新执行，onBtnClick 也会重新定义，完全不会影响到我们之前调用的 onBtnClick 和其中引用的 count。所以 alert 的值是 3。

每当状态更新时，整个函数都会重新执行，其中的变量、函数和之前定义的都是相互独立的，类似不同的帧。值得说明的是，dom 中使用的 count，非常单纯的就是每次重新执行后的一个常量的值，完全没有数据绑定、监听、代理之类的“黑科技”。

而 effect 拿到的总是定义它的那次渲染中的 props 和 state
> [函数式组件与类组件有何不同？](https://overreacted.io/zh-hans/how-are-function-components-different-from-classes/)

## 每次渲染都有它自己的 Effects
和组件中其它的事件处理函数一样，每次渲染都会产生属于那次渲染的 effect 函数。其中的变量也是那次渲染中的值。
如果想在 effect 的回调函数中使用最新的值，而不是捕获的当前渲染的值，可以使用 refs。值得注意的是，如果想要在过去的渲染中定义的函数里，读取“未来”的 props 和 state，这种方式有打破默认范式的意味，可能会导致代码的脆弱性，如果有其它方式代替逻辑，最好使用其它方式。

## effect 中的清理
effect 执行是在浏览器绘制之后，effect 的清除同样被延迟了。
上一个组件中 effect 的清除函数，会在下一个组件绘制到浏览器之后，执行 useEffect 之前执行。也就是说上一个组件的清除和下一个组件的 effect 执行时机是一致的，只是前者更早。
上一个组件和下一个组件可以是两个不同的函数组件，也可以是同一个函数组件的两次渲染。假如是同一个组件，上次渲染时产生的清除函数会在下一次渲染的绘制之后，effect 之前执行。

看一下下面的代码：
```jsx
useEffect(() => {
    ChatAPI.subscribeToFriendStatus(props.id, handleStatusChange);
    return () => {
      ChatAPI.unsubscribeFromFriendStatus(props.id, handleStatusChange);
    };
  });
```
props.id 从 10 变为 20 时，会发生什么呢？
 - react 会优先渲染 id 为 20 时的 UI
 - 浏览器绘制 UI，用户看到 id 为 20 时的 UI
 - 清除 id 为 10 时的订阅
 - 执行 id 为 20 时的 effect
可以看到，hooks 中 UI 的优先级更高，更注重用户体验，这也是 hooks 的“心智模型”之一。

那么，清除函数为什么还能拿到 id 为 10 的 props 呢？
答案就在上面，**effect 拿到的总是定义它的那次渲染中的 props 和 state**。一帧一帧的函数...有那么一丝感觉了吧～

## 同步，而非生命周期
从上面可以看到，Hooks 讲究的是初始渲染和后续更新的同步，而不是生命周期的复杂概念。
react 的每一次渲染所执行的函数、变量基本是一致的，这减少了程序的“熵”，不会让程序随着状态越来越多、更新越来越多，堆积越来越多的变化组合，导致定位 bug 很艰难。Hooks 的优势就在于每一次更新相当于重新开始，没有累积变化，程序看起来就会清晰很多。
同步是指，React 根据当前的 props 和 state 同步修改到 Dom。没有像类组件的 “mount” 和 “update” 那样的生命周期。effect 也是类似的“心智模型”，在浏览器渲染之后，根据当前的状态，同步做 React tree 之外的修改。

## 依赖
effect 的依赖可以条件式的避免渲染，只在变化的时候执行。
有时候可以通过某种方式省略依赖
  - 函数式更新
  - useReducer
使用 useCallback 常量化函数依赖

[] 依赖比较接近类组件的 componentDidMount 和 componentWillUnmount，在这里可以执行一次第一次渲染和卸载时的逻辑。
但是如果有些依赖尝试了以上方式后仍不可避免，那么只能在 effect 中使用条件判断的方式去执行某些逻辑，尽管这种方式可能打断了函数式组件同步执行的模型，但是好像没有什么更好的办法。
