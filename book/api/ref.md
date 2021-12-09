{% method %}

# createRef

是的就是返回一个`{ current: null }`的对象，最终`current`被赋值要等到后面渲染组件的时候，后面会有专门的章节来讲具体逻辑

{% sample lang="js" %}

```js
export function createRef(): RefObject {
  const refObject = {
    current: null,
  };
  if (__DEV__) {
    Object.seal(refObject);
  }
  return refObject;
}
```

{% endmethod %}


{% method %}

# forwardRef

`forwardRef`主要配合`HOC`使用，他接收`FunctionalComponent`作为参数，因为`FunctionalComponent`没有实例，所以在他上面使用`ref`没有什么意义，二`ref`本身不是`props`，无法在`FunctionalComponent`中获取到再赋值给他的子节点。

`forwardRef`让`FunctionalComponent`可以接收第二个参数`function FunComp(props, ref) {}`，第二个参数就是他的父辈组件作用在他上面的`ref`，他可以获取到并且当作变量传递给他的子孙节点。

`forwardRef`的代码也是异常简单，

可以看到他返回的是一个对象，里面有一个比较特殊的变量`$$typeof`，他的值是一个常量，用来表明这个个节点的类型是`forwardRef`，而我们传入的`FunctionalComponent`则是返回对象的另外一个属性，*在后期讲组件更新的时候，我们会再详细讲对于`forwardRef`更新*

{% sample lang="js" %}

```js

export default function forwardRef<Props, ElementType: React$ElementType>(
  render: (props: Props, ref: React$Ref<ElementType>) => React$Node,
) {
  
  // some dev code

  return {
    $$typeof: REACT_FORWARD_REF_TYPE,
    render,
  };
}

```

{% endmethod %}