# React

> 虽然平时我们都喜欢说我们用`React`作为我们的核心框架，但其实大部分人都不知道`React`到底是个什么东东。事实上自从Facebook把`React`和`ReactDOM`分包发布之后，`React`就不仅仅是一开始的前端框架了，如果在15版本之后去看一下`react`和`react-dom`的源码大小，你就会发现，`react`仅仅1000多行代码，而`react-dom`却将近2w行。是的你没看错，而且你很可能也没有想错，其实大部分的框架逻辑都在`react-dom`当中，那么`react`到底是个什么东东呢？

### ReactElement

能来看源码的同学，我相信你们至少肯定知道`JSX`代码在编译之后是什么样子的

```js
// JSX
<div id="id">text</div>

// js
React.createElement('div', {id: 'id'}, 'text')
```

而这恰恰就是`react`这个包最最核心的代码，详细源码解析可以看[这里](/docs/core/create-element.html)。

其实`React`的核心产出，就是`element`，这个`element`是平台无关的，只跟你传入的参数有关，比如上面的例子，你传入的是一个字符串`div`，那么你的`element.type`就是`div`，而字符串的类型，对于`react-dom`平台就意味着是`HostComponent`也就是原生html标签。我们可以看一下`react`里定义的`type`类型

```js
export const IndeterminateComponent = 0; // 不确定是ClassComponent还是functionalComponent
export const FunctionalComponent = 1;  
export const ClassComponent = 2;
export const HostRoot = 3; // reactDom.render()接受的第二个参数
export const HostPortal = 4; // portal
export const HostComponent = 5;  // 原生html即诶点
export const HostText = 6;   // 文本节点
export const CallComponent_UNUSED = 7;
export const CallHandlerPhase_UNUSED = 8;
export const ReturnComponent_UNUSED = 9;
export const Fragment = 10;
export const Mode = 11;  // AsyncMode
export const ContextConsumer = 12;
export const ContextProvider = 13;
export const ForwardRef = 14;
export const Profiler = 15;
export const TimeoutComponent = 16;
```

当然在`React.createElement`的时候返回的内容里的`type`肯定不是这些值，而是后期`react-dom`会根据`element`的类型转换成`Fiber`的类型

这写类型在`React Fiber`里面叫做`TypeOfWork`，对于不同的类型，后期会有不同处理。而`react`在这里起的作用就是创建`element`并包含一些源信息，以便后期不同平台根据自己的特性进行处理。

### 内置类型

当然我们用`react`当然不仅仅用他的`createElement`，很多同学提到`context api`，确实，这也是`react`的API，但其实呢，他们也是为`ReactElement`服务的。

我们先来看看`context api`

```js
const { Provider, Consumer } = React.createContext('defaultValue')
```

具体API源码解析看[这里](/docs/core/context.html)

在使用的的时候我们会把`Provider`和`Consumer`作为`createElement`的参数使用，所以这两个其实是一个特殊的类型，他们不是`class`也不是`function`和字符串，他们是一个特定的对象，来告诉后期平台他们是特殊的节点。

```js
context.Provider = {
  $$typeof: REACT_PROVIDER_TYPE,
  _context: context,
};
```

还有即将在`17`版本中出现的`AsyncMode`，我们可以这么用:

```js
const AsyncMode = React.unstable_AsyncMode


<AsyncMode></AsyncMode>
```

那么`AsyncMode`是什么呢？其实他是一个`Symbol`，也就是说是一个唯一标志位，告诉平台这个节点的子节点要用异步的模式进行渲染，这就是他的任务。

### Component

当然还有一个少不了用的就是`React.Component`了，我们声明的`ClassComponent`都是继承自这个基类的。不过其实他没什么内容，只是在原型上定义了`setState, forceUpdate`等方法，而它们最终调用的是个个平台提供给实例的`updater`对象上的方法，所以源码非常简单，就不多详细讲解了，源码位置`packages/react/src/ReactBaseClass.js`，相信js基础过关的同学都看得懂。

```js
Component.prototype.setState = function(partialState, callback) {
  invariant(
    typeof partialState === 'object' ||
      typeof partialState === 'function' ||
      partialState == null,
    'setState(...): takes an object of state variables to update or a ' +
      'function which returns an object of state variables.',
  );
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
};
```

# 总结
总的来说`react`部分的源码是非常简单的，总共也就1000多行，还有很多是注释，大家看一遍也花不了多少时间，注意看的时候不要去考虑React是如何做渲染的，只看源码理解他的输入输出就行。