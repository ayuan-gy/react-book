{% method %}

# diffProperties

这是对 DOM 节点属性是否有变化的判断，对于以下节点，不管是否有变化，`updatePayload`都至少是一个空数组，也就是需要重新渲染

- `input`
- `option`
- `select`
- `textarea`

之后进入两个循环

### 第一个循环主要判断被删除的`props`

```js
if (
  nextProps.hasOwnProperty(propKey) ||
  !lastProps.hasOwnProperty(propKey) ||
  lastProps[propKey] == null
) {
  continue
}
```

这边的条件是：

- 如果新的`props`中有属性
- 或者老的`props`中没有这个属性
- 或者老的`prop`等于`null`

那么这个不是被删除的

如果找到被删除的属性，对于一些特殊属性需要执行操作，最明显的是`style`，需要把所有`style`属性都清空成空字符串

### 第二个循环就是找不同了

```js
const nextProp = nextProps[propKey]
const lastProp = lastProps != null ? lastProps[propKey] : undefined
if (
  !nextProps.hasOwnProperty(propKey) ||
  nextProp === lastProp ||
  (nextProp == null && lastProp == null)
) {
  continue
}
```

条件如下：

- 如果新的没有
- 或者新老属性相同
- 或者都等于`null`

这代表是相同的。对于`style`因为传入的是`object`，所以肯定不相同（除非开发者自己做了优化），所以需要等`style`中每个属性进行判断是否变化，并且找出变化的集合

对于`DANGEROUSLY_SET_INNER_HTML`，需要判断对象内的`__html`是否变化

`registrationNameModules`是事件相关的，默认只是空对象，React 有一套注入机制，讲`event`的时候再详细讲解。

### 小结

我们会发现最后返回的数组会是这个样子的：

```js
updatePayload = [key1, value1, key2, value2]
```

再`completeWork`阶段，是通过

```js
if (updatePayload) {
  markUpdate(workInProgress)
}
```

来判断是否需要更新的，所以即便返回空数组还是会需要更新

{% sample lang="js" %}

```js
export function diffProperties(
  domElement: Element,
  tag: string,
  lastRawProps: Object,
  nextRawProps: Object,
  rootContainerElement: Element | Document,
): null | Array<mixed> {
  if (__DEV__) {
    validatePropertiesInDevelopment(tag, nextRawProps)
  }

  let updatePayload: null | Array<any> = null

  let lastProps: Object
  let nextProps: Object
  switch (tag) {
    case 'input':
      lastProps = ReactDOMInput.getHostProps(domElement, lastRawProps)
      nextProps = ReactDOMInput.getHostProps(domElement, nextRawProps)
      updatePayload = []
      break
    case 'option':
      lastProps = ReactDOMOption.getHostProps(domElement, lastRawProps)
      nextProps = ReactDOMOption.getHostProps(domElement, nextRawProps)
      updatePayload = []
      break
    case 'select':
      lastProps = ReactDOMSelect.getHostProps(domElement, lastRawProps)
      nextProps = ReactDOMSelect.getHostProps(domElement, nextRawProps)
      updatePayload = []
      break
    case 'textarea':
      lastProps = ReactDOMTextarea.getHostProps(domElement, lastRawProps)
      nextProps = ReactDOMTextarea.getHostProps(domElement, nextRawProps)
      updatePayload = []
      break
    default:
      lastProps = lastRawProps
      nextProps = nextRawProps
      if (
        typeof lastProps.onClick !== 'function' &&
        typeof nextProps.onClick === 'function'
      ) {
        // TODO: This cast may not be sound for SVG, MathML or custom elements.
        trapClickOnNonInteractiveElement(((domElement: any): HTMLElement))
      }
      break
  }

  assertValidProps(tag, nextProps)

  let propKey
  let styleName
  let styleUpdates = null
  for (propKey in lastProps) {
    if (
      nextProps.hasOwnProperty(propKey) ||
      !lastProps.hasOwnProperty(propKey) ||
      lastProps[propKey] == null
    ) {
      continue
    }
    if (propKey === STYLE) {
      const lastStyle = lastProps[propKey]
      for (styleName in lastStyle) {
        if (lastStyle.hasOwnProperty(styleName)) {
          if (!styleUpdates) {
            styleUpdates = {}
          }
          styleUpdates[styleName] = ''
        }
      }
    } else if (propKey === DANGEROUSLY_SET_INNER_HTML || propKey === CHILDREN) {
      // Noop. This is handled by the clear text mechanism.
    } else if (
      propKey === SUPPRESS_CONTENT_EDITABLE_WARNING ||
      propKey === SUPPRESS_HYDRATION_WARNING
    ) {
      // Noop
    } else if (propKey === AUTOFOCUS) {
      // Noop. It doesn't work on updates anyway.
    } else if (registrationNameModules.hasOwnProperty(propKey)) {
      // This is a special case. If any listener updates we need to ensure
      // that the "current" fiber pointer gets updated so we need a commit
      // to update this element.
      if (!updatePayload) {
        updatePayload = []
      }
    } else {
      // For all other deleted properties we add it to the queue. We use
      // the whitelist in the commit phase instead.
      ;(updatePayload = updatePayload || []).push(propKey, null)
    }
  }
  for (propKey in nextProps) {
    const nextProp = nextProps[propKey]
    const lastProp = lastProps != null ? lastProps[propKey] : undefined
    if (
      !nextProps.hasOwnProperty(propKey) ||
      nextProp === lastProp ||
      (nextProp == null && lastProp == null)
    ) {
      continue
    }
    if (propKey === STYLE) {
      if (__DEV__) {
        if (nextProp) {
          // Freeze the next style object so that we can assume it won't be
          // mutated. We have already warned for this in the past.
          Object.freeze(nextProp)
        }
      }
      if (lastProp) {
        // Unset styles on `lastProp` but not on `nextProp`.
        for (styleName in lastProp) {
          if (
            lastProp.hasOwnProperty(styleName) &&
            (!nextProp || !nextProp.hasOwnProperty(styleName))
          ) {
            if (!styleUpdates) {
              styleUpdates = {}
            }
            styleUpdates[styleName] = ''
          }
        }
        // Update styles that changed since `lastProp`.
        for (styleName in nextProp) {
          if (
            nextProp.hasOwnProperty(styleName) &&
            lastProp[styleName] !== nextProp[styleName]
          ) {
            if (!styleUpdates) {
              styleUpdates = {}
            }
            styleUpdates[styleName] = nextProp[styleName]
          }
        }
      } else {
        // Relies on `updateStylesByID` not mutating `styleUpdates`.
        if (!styleUpdates) {
          if (!updatePayload) {
            updatePayload = []
          }
          updatePayload.push(propKey, styleUpdates)
        }
        styleUpdates = nextProp
      }
    } else if (propKey === DANGEROUSLY_SET_INNER_HTML) {
      const nextHtml = nextProp ? nextProp[HTML] : undefined
      const lastHtml = lastProp ? lastProp[HTML] : undefined
      if (nextHtml != null) {
        if (lastHtml !== nextHtml) {
          ;(updatePayload = updatePayload || []).push(propKey, '' + nextHtml)
        }
      } else {
        // TODO: It might be too late to clear this if we have children
        // inserted already.
      }
    } else if (propKey === CHILDREN) {
      if (
        lastProp !== nextProp &&
        (typeof nextProp === 'string' || typeof nextProp === 'number')
      ) {
        ;(updatePayload = updatePayload || []).push(propKey, '' + nextProp)
      }
    } else if (
      propKey === SUPPRESS_CONTENT_EDITABLE_WARNING ||
      propKey === SUPPRESS_HYDRATION_WARNING
    ) {
      // Noop
    } else if (registrationNameModules.hasOwnProperty(propKey)) {
      if (nextProp != null) {
        // We eagerly listen to this even though we haven't committed yet.
        if (__DEV__ && typeof nextProp !== 'function') {
          warnForInvalidEventListener(propKey, nextProp)
        }
        ensureListeningTo(rootContainerElement, propKey)
      }
      if (!updatePayload && lastProp !== nextProp) {
        // This is a special case. If any listener updates we need to ensure
        // that the "current" props pointer gets updated so we need a commit
        // to update this element.
        updatePayload = []
      }
    } else {
      // For any other property we always add it to the queue and then we
      // filter it out using the whitelist during the commit.
      ;(updatePayload = updatePayload || []).push(propKey, nextProp)
    }
  }
  if (styleUpdates) {
    ;(updatePayload = updatePayload || []).push(STYLE, styleUpdates)
  }
  return updatePayload
}
```

{% endmethod %}
