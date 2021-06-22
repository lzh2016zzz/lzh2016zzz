---
title: React入门之hooks
tags: 读书笔记,react
---

React 是主流的前端框架，v16.8 版本引入了全新的 API，叫做 React Hooks，颠覆了以前的用法。这个 API 是 React 的未来，有必要深入理解。本文谈谈我的理解，简单介绍它的用法，帮助大家快速上手。

## 一、组件类的缺点
React 的核心是组件。v16.8 版本之前，组件的标准写法是类（class）。下面是一个简单的组件类。
```
import React, { Component } from "react";

export default class Button extends Component {
  constructor() {
    super();
    this.state = { buttonText: "Click me, please" };
    this.handleClick = this.handleClick.bind(this);
  }
  handleClick() {
    this.setState(() => {
      return { buttonText: "Thanks, been clicked!" };
    });
  }
  render() {
    const { buttonText } = this.state;
    return <button onClick={this.handleClick}>{buttonText}</button>;
  }
}
```
这个组件类仅仅是一个按钮，但可以看到，它的代码已经很"重"了。真实的 React App 由多个类按照层级，一层层构成，复杂度成倍增长。再加入 Redux，就变得更复杂。

Redux 的作者 Dan Abramov 总结了组件类的几个缺点。

*  大型组件很难拆分和重构，也很难测试。
*  业务逻辑分散在组件的各个方法之中，导致重复逻辑或关联逻辑。
*  组件类引入了复杂的编程模式，比如 render props 和高阶组件。

##  二、函数组件

React 团队希望，组件不要变成复杂的容器，最好只是数据流的管道。开发者根据需要，组合管道即可。 组件的最佳写法应该是函数，而不是类。

React 早就支持函数组件，下面就是一个例子。
```
function Welcome(props) {
  return <h1>Hello, {props.name}</h1>;
}
```
但是，这种写法有重大限制，必须是纯函数，不能包含状态，也不支持生命周期方法，因此无法取代类。
React Hooks 的设计目的，就是加强版函数组件，完全不使用"类"，就能写出一个全功能的组件。


## 三、useState()：状态钩子
useState()用于为函数组件引入状态（state）。纯函数不能有状态，所以把状态放在钩子里面。

本文前面那个组件类，用户点击按钮，会导致按钮的文字改变，文字取决于用户是否点击，这就是状态。使用useState()重写如下。

```
import React, { useState } from "react";

export default function  Button()  {
  const  [buttonText, setButtonText] =  useState("Click me,   please");

  function handleClick()  {
    return setButtonText("Thanks, been clicked!");
  }

  return  <button  onClick={handleClick}>{buttonText}</button>;
}
```

上面代码中，Button 组件是一个函数，内部使用useState()钩子引入状态。

useState()这个函数接受状态的初始值，作为参数，上例的初始值为按钮的文字。该函数返回一个数组，数组的第一个成员是一个变量（上例是buttonText），指向状态的当前值。第二个成员是一个函数，用来更新状态，约定是set前缀加上状态的变量名（上例是setButtonText）。


## 四、useContext()：共享状态钩子
如果需要在组件之间共享状态，可以使用useContext()。
现在有两个组件 Navbar 和 Messages，我们希望它们之间共享状态。
```
<div className="App">
  <Navbar/>
  <Messages/>
</div>
```
第一步就是使用 React Context API，在组件外部建立一个 Context。
```
const AppContext = React.createContext({});
```
组件封装代码如下。
```
<AppContext.Provider value={{
  username: 'superawesome'
}}>
  <div className="App">
    <Navbar/>
    <Messages/>
  </div>
</AppContext.Provider>
```
上面代码中，AppContext.Provider提供了一个 Context 对象，这个对象可以被子组件共享。
Navbar 组件的代码如下。
```
const Navbar = () => {
  const { username } = useContext(AppContext);
  return (
    <div className="navbar">
      <p>AwesomeSite</p>
      <p>{username}</p>
    </div>
  );
}
```

上面代码中，useContext()钩子函数用来引入 Context 对象，从中获取username属性。
Message 组件的代码也类似。

```
const Messages = () => {
  const { username } = useContext(AppContext)

  return (
    <div className="messages">
      <h1>Messages</h1>
      <p>1 message for {username}</p>
      <p className="message">useContext is awesome!</p>
    </div>
  )
}
```


## 五、useReducer()：action 钩子
React 本身不提供状态管理功能，通常需要使用外部库。这方面最常用的库是 Redux。

Redux 的核心概念是，组件发出 action 与状态管理器通信。状态管理器收到 action 以后，使用 Reducer 函数算出新的状态，Reducer 函数的形式是(state, action) => newState。

useReducers()钩子用来引入 Reducer 功能。
```
const [state, dispatch] = useReducer(reducer, initialState);
```
上面是useReducer()的基本用法，它接受 Reducer 函数和状态的初始值作为参数，返回一个数组。数组的第一个成员是状态的当前值，第二个成员是发送 action 的dispatch函数。
下面是一个计数器的例子。用于计算状态的 Reducer 函数如下。
```
const myReducer = (state, action) => {
  switch(action.type)  {
    case('countUp'):
      return  {
        ...state,
        count: state.count + 1
      }
    default:
      return  state;
  }
}
```
组件代码如下。
```
function App() {
  const [state, dispatch] = useReducer(myReducer, { count:   0 });
  return  (
    <div className="App">
      <button onClick={() => dispatch({ type: 'countUp' })}>
        +1
      </button>
      <p>Count: {state.count}</p>
    </div>
  );
}
```
由于 Hooks 可以提供共享状态和 Reducer 函数，所以它在这些方面可以取代 Redux。但是，它没法提供中间件（middleware）和时间旅行（time travel），如果你需要这两个功能，还是要用 Redux。


## 六、useEffect()：影响(效果)钩子
useEffect()用来引入具有副作用的操作，最常见的就是向服务器请求数据。以前，放在componentDidMount里面的代码，现在可以放在useEffect()。
useEffect()的用法如下。

```
useEffect(()  =>  {
  // Async Action
}, [dependencies])
```
上面用法中，useEffect()接受两个参数。第一个参数是一个函数，异步操作的代码放在里面。第二个参数是一个数组，用于给出 Effect 的依赖项，只要这个数组发生变化，useEffect()就会执行。第二个参数可以省略，这时每次组件渲染时，就会执行useEffect()。
下面看一个例子。

```
const Person = ({ personId }) => {
  const [loading, setLoading] = useState(true);
  const [person, setPerson] = useState({});

  useEffect(() => {
    setLoading(true); 
    fetch(`https://swapi.co/api/people/${personId}/`)
      .then(response => response.json())
      .then(data => {
        setPerson(data);
        setLoading(false);
      });
  }, [personId])

  if (loading === true) {
    return <p>Loading ...</p>
  }

  return <div>
    <p>You're viewing: {person.name}</p>
    <p>Height: {person.height}</p>
    <p>Mass: {person.mass}</p>
  </div>
}
```
上面代码中，每当组件参数personId发生变化，useEffect()就会执行。组件第一次渲染时，useEffect()也会执行。

## 七丶 useSelector() :获取state钩子
useSelector()用来共享状态,从redux的store中提取数据,代替反人类的connect语法. useSelector()的用法如下:
```
  const filter = useSelector((state) => state.articleFilter);
```

注意：选择器函数应该是纯函数，因为它可能在任意时间点多次执行。


## 八 useDispatch() : dispatch函数引用钩子

返回Redux store中对dispatch函数的引用。你可以根据需要使用它。使用方法如下:

```
import React from 'react'
import { useDispatch } from 'react-redux'

export const CounterComponent = ({ value }) => {
  const dispatch = useDispatch()

  return (
    <div>
      <span>{value}</span>
      <button onClick={() => dispatch({ type: 'increment-counter' })}>
        Increment counter
      </button>
    </div>
  )
}

```

将回调使用dispatch传递给子组件时，建议使用来进行回调useCallback，因为否则，由于更改了引用，子组件可能会不必要地呈现。

```
import React, { useCallback } from 'react'
import { useDispatch } from 'react-redux'

export const CounterComponent = ({ value }) => {
  const dispatch = useDispatch()
  const incrementCounter = useCallback(
    () => dispatch({ type: 'increment-counter' }),
    [dispatch]
  )

  return (
    <div>
      <span>{value}</span>
      <MyIncrementButton onIncrement={incrementCounter} />
    </div>
  )
}

export const MyIncrementButton = React.memo(({ onIncrement }) => (
  <button onClick={onIncrement}>Increment counter</button>
))
```

