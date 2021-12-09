> React16.3发布了新的Context API，并且已经确认了将在下一个版本废弃老的Context API。所以大家更新到新的Context API是无可厚非的事情。而这篇文章会从原理的角度为大家分析为什么要用新的API--不仅仅是因为React官方要更新，毕竟更新了你也可以用16版本的React来使用老的API--而是因为新的API性能比老API **高出太多**

# 用法

我们先来看一下两个版本的Context API如何使用

```js
// old version
class Parent extends Component{
  getChildContext() {
    return {type: 123}
  }
}

Parent.childContextType = {
  type: PropTypes.number
}

const Child = (props, context) => (
  <p>{context.type}</p>
)

Child.contextTypes = {
  type: PropTypes.number
}
```

通过在父组件上声明`getChildContext`方法为其子孙组件提供`context`，我们称其`ProviderComponent`。注意必须要声明`Parent.childContextType`才会生效，而子组件如果需要使用`context`，需要显示得声明`Child.contextTypes`

```js
// new version
const { Provider, Consumer } = React.createContext('defaultValue')

const Parent = (props) => (
  <Provider value={'realValue'}>
    {props.children}
  </Provider>
)

const Child = () => {
  <Consumer>
    {
      (value) => <p>{value}</p>
    }
  </Consumer>
}
```

新版本的API，React提供了`createContext`方法，这个方法会返回两个*组件*：`Provider`和`Consumber`，`Provider`用来提供`context`的内容，通过向`Provider`传递`value`这个`prop`，而在需要用到对应`context`的地方，用**相同来源的**`Consumer`来获取`context`，`Consumer`有特定的用法，就是他的`children`必须是一个方法，并且`context`的值使用参数传递给这个方法。

# 性能对比
正好前几天React devtool发布了`Profiler`功能，就用这个新功能来查看一下两个API的新能有什么差距吧，先看一下例子

[不知道Profiler的看这里](https://juejin.im/post/5ba1f995f265da0a972e1657)

```js
// old api demo
import React from 'react'
import PropTypes from 'prop-types'

export default class App extends React.Component {
  state = {
    type: 1,
  }

  getChildContext() {
    return {
      type: this.state.type
    }
  }

  componentDidMount() {
    setInterval(() => {
      this.setState({
        type: this.state.type + 1
      })
    }, 500)
  }

  render() {
    return this.props.children
  }
}

App.childContextTypes = {
  type: PropTypes.number
}

export const Comp = (props, context) => {
  const arr = []
  for (let i=0; i<100; i++) {
    arr.push(<p key={i}>{i}</p>)
  }

  return (
    <div>
      <p>{context.type}</p>
      {arr}
    </div>
  )
}

Comp.contextTypes = {
  type: PropTypes.number
}
```

```js
// new api demo
import React, { Component, createContext } from 'react'

const { Provider, Consumer } = createContext(1)

export default class App extends Component {

  state = {
    type: 1
  }

  componentDidMount() {
    setInterval(() => {
      this.setState({
        type: this.state.type + 1
      })
    }, 500)
  }

  render () {
    return (
      <Provider value={this.state.type}>
        {this.props.children}
      </Provider>
    )
  }

}

export const Comp = () => {
  const arr = []
  for (let i=0; i<100; i++) {
    arr.push(<p key={i}>{i}</p>)
  }

  return (
    <div>
      <Consumer>
        {(type) => <p>{type}</p>}
      </Consumer>
      {arr}
    </div>
  )
}
```

```js
// index.js
import React from 'react';
import ReactDOM from 'react-dom';
import './index.css';

import App, {Comp} from './context/OldApi'

// import App, { Comp } from './context/NewApi'

ReactDOM.render(
  <App><Comp /></App>,
  document.getElementById('root')
)
```

代码基本相同，主要变动就是一个`interval`，每500毫秒给`type`加1，然后我们来分别看一下`Profiler`的截图

[不知道Profiler的看这里](https://juejin.im/post/5ba1f995f265da0a972e1657)

##### 老API

![老API](http://pegnygxjm.bkt.clouddn.com/old-context-demo1-1.png)

##### 新API

![新API](http://pegnygxjm.bkt.clouddn.com/new-context-demo1-1.png)

可见这两个性能差距是非常大的，老的API需要7点几毫秒，而新的API只需要0.4毫秒，而且新的API只有两个节点重新渲染了，而老的API所有节点都重新渲染了（下面还有很多节点没截图进去，虽然每个可能只有0.1毫秒或者甚至不到，但是积少成多，导致他们的父组件Comp渲染时间很长）

# 进一步举例
在这里可能有些同学会想，新老API的用法不一样，因为老API的`context`是作为`Comp`这个`functional Component`的参数传入的，所以肯定会影响该组件的所有子元素，所以我在这个基础上修改了例子，把数组从`Comp`组件中移除，放到一个新的组件`Comp2`中

```js
// Comp2
export class Comp2 extends React.Component {
  render() {
    const arr = []
    for (let i=0; i<100; i++) {
      arr.push(<p key={i}>{i}</p>)
    }

    return arr
  }
}

// new old api Comp
export const Comp = (props, context) => {
  return (
    <div>
      <p>{context.type}</p>
    </div>
  )
}

// new new api Comp
export const Comp = () => {
  return (
    <div>
      <Consumer>
        {(type) => <p>{type}</p>}
      </Consumer>
    </div>
  )
}
```

现在受`context`影响的渲染内容新老API都是一样的，只有`<p>{type}</p>`，我们再来看一下情况

##### 老API

![老API](http://pegnygxjm.bkt.clouddn.com/old-context-demo2-1.png)

##### 新API

![新API](http://pegnygxjm.bkt.clouddn.com/new-context-demo2-1.png)

*忽视比demo1时间长的问题，应该是我电脑运行时间长性能下降的问题，只需要横向对比新老API就可以了*

从这里可以看出来，结果跟Demo1没什么区别，老API中我们的`arr`仍然都被重新渲染了，导致整体的渲染时间被拉长很多。

事实上，这可能还不是最让你震惊的地方，我们再改一下例子，我们在`App`中不再修改`type`，而是新增一个`state`叫`num`，然后对其进行递增

```js
// App
export default class App extends React.Component {
  state = {
    type: 1,
    num: 1
  }

  getChildContext() {
    return {
      type: this.state.type
    }
  }

  componentDidMount() {
    setInterval(() => {
      this.setState({
        num: this.state.num + 1
      })
    }, 500)
  }

  render() {
    return (
      <div>
        <p>inside update {this.state.num}</p>
        {this.props.children}
      </div>
    )
  }
}
```

##### 老API

![老API](http://pegnygxjm.bkt.clouddn.com/old-context-demo3-1.png)

##### 新API

![新API](http://pegnygxjm.bkt.clouddn.com/new-context-deme3-1.png)

可以看到老API依然没有什么改观，他依然重新渲染所有子节点。

再进一步我给`Comp2`增加`componentDidUpdate`生命周期钩子

```js
componentDidUpdate() {
  console.log('update')
}
```

在使用老API的时候，每次App更新都会打印

![](http://pegnygxjm.bkt.clouddn.com/old-context-update.png)

而新API则不会

# 总结

从上面测试的结果大家应该可以看出来结果了，这里简单的讲一下原因，因为要具体分析会很长并且要涉及到源码的很多细节，所以有空再写一片续，来详细得讲解源码，大家有兴趣的可以关注我。

要分析原理要了解React对于每次更新的处理流程，React是一个树结构，要进行更新只能通过某个节点执行`setState、forceUpdate`等方法，在某一个节点执行了这些方法之后，React会向上搜索直到找到`root`节点，然后把`root`节点放到更新队列中，等待更新。

所以React的更新都是从`root`往下执行的，他会尝试重新构建一个新的树，在这个过程中能复用之前的节点就会复用，**而我们现在看到的情况，就是因为复用算法根据不同的情况而得到的不同的结果**

我们来看一小段源码

```js
if (
  !hasLegacyContextChanged() &&
  (updateExpirationTime === NoWork ||
    updateExpirationTime > renderExpirationTime)
) {
  // ...
  return bailoutOnAlreadyFinishedWork(
    current,
    workInProgress,
    renderExpirationTime,
  );
}
```

如果能满足这个判断条件并且进入`bailoutOnAlreadyFinishedWork`，那么有极高的可能这个节点以及他的子树都不需要更新，React会直接跳过，我们使用新的`context API`的时候就是这种情况，**但是使用老的`context API`是永远不可能跳过这个判断的**

老的`context API`使用过程中，一旦有一个节点提供了`context`，那么他的所有子节点都会被视为有`side effect`的，因为React本身并不判断子节点是否有使用`context`，以及提供的`context`是否有变化，所以一旦检测到有节点提供了`context`，那么他的子节点在执行`hasLegacyContextChanged`的时候，永远都是true的，而没有进入`bailoutOnAlreadyFinishedWork`，就会变成重新`reconcile`子节点，虽然最终可能不需要更新DOM节点，但是重新计算生成`Fiber`对象的开销还是又得，一两个还好，数量多了时间也是会被拉长的。

以上就是使用老的`context API`比新的API要慢很多的原因，大家可以先不深究得理解一下，在我之后的源码分析环节会有更详细的讲解。