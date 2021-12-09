# React.Component

`Component`的实现非常简单

```js
function Component(props, context, updater) {
  this.props = props;
  this.context = context;
  this.refs = emptyObject;
  this.updater = updater || ReactNoopUpdateQueue;
}
Component.prototype.isReactComponent = {};
Component.prototype.setState = function(partialState, callback) {
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
};
Component.prototype.forceUpdate = function(callback) {
  this.updater.enqueueForceUpdate(this, callback, 'forceUpdate');
};
```

就是一些属性我们常用的属性，唯一一个不熟悉的是`updater`，**这个属性是在组件实例化的时候传入的，跟不同平台有关，比如`RN`和`ReactDOM`就有不同的实现**。

而事实上我们在使用`ClassComponent`的时候也不过是把我们声明的`class`作为参数传给`createElement`，在这个时候并没有实例化，实例化其实是在更新过程中才做的事情。

**所以`ClassComponent`不过是一个载体，因为我们需要使用组件内部的`state`和`side effect`，而类的实例也是相对比较封闭的一个概念，所以React团队才选择用`class`来作为基础的组件声明（此处怀念一下曾经的React.createClass）。同时在React团队找到了更好的使用`state`和`side effect`的方式的时候，也就是React16.7的`Hooks`，那么`class`形式的组件声明也将渐渐淡出开发者的视线**


### PureComponent

`PureComponent`跟`Component`完全就是一样的，唯一的区别是

```js
pureComponentPrototype.isPureReactComponent = true;
```

他的原型上多了一个标示表明继承自这个类的组件是`pure`的，那么后期ReactDOM对节点树进行更新的时候，会对`pure`的组件增加一个 **浅比较`props`**的逻辑，来减少不必要的组件更新。
