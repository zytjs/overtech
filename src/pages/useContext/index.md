---
title: useContext 的全部
date: '2021-09-02'
spoiler: 一文搞懂 useContext
cta: 'react'
---

Context 提供了一个不用向每层组件手动添加 props，就能在组件树之间进行数据传递的方法。

它要解决的问题是，当孙子组件需要使用爷爷组件的变量时，在父组件上进行了实际上无用的传递。React 数据一般通过 props 由父到子传递，context 提供了一种在组件间共享数据的方式，不用我们显示的层层传递数据。

## 使用 Context 之前的考虑

[官方文档说到](https://zh-hans.reactjs.org/docs/context.html#before-you-use-context)，context 会使组件的复用性变差，所以建议谨慎使用。但是如果我们在纯业务组件中并不会考虑复用性，这时候使用就没有任何问题。

考虑到 context 解决的问题，有时候可以使用组件组合。这种方式是要把需要使用数据的组件提升到高层次组件，然后将组件本身层层传递下去。但是将底层逻辑提升，首先会污染高层组件，使它更复杂，然后还具有传染性，后面的组件还是需要向下层层传递。而且被提升的组件需要和父组件完全解耦，限制也较多。

## API

### React.createContext 、Context.Provider

```jsx
const MyContext = React.createContext(defaultValue)

<MyContext.Provider value={}>
  {children}
</MyContext.Provider>
```

**只有**当组件所处的树中没有匹配到 Provider 时，其 defaultValue 参数才会生效。注意：将 undefined 传递给 Provider 的 value 时，消费组件的 defaultValue 不会生效（有点类似解构赋值时，只有在 undefined 下赋值才有用）。

Provider 接收一个 value 属性，传递给消费组件。多个 Provider 也可以嵌套使用，里层的会覆盖外层的数据。

当 Provider 的 value 值发生变化时，它内部的**所有**消费组件都会重新渲染。这是因为更新 value 值一般是使用 setState，所以为引发后代更新。从 Provider 到其后代 consumer 组件的传播，不受制于 shouldComponentUpdate 函数，因此当 consumer 组件在其祖先组件跳过更新的情况下也能更新。

>> 那么在 context 更新时，如何避免重复渲染呢？

### Class.contextType  

`static contextType = MyContext`

此属性可以让你使用 this.context 来获取最近 Context 上的值，但是只能订阅单一 context，也就是最近的 Provider 提供的。

### Context.Consumer

这种方法需要一个函数作为子元素。这个函数接受 context 的值作为参数，并返回一个 React 节点。

### 使用技巧
  - 深层组件更新 context
  - 同一个组件消费多个 context
