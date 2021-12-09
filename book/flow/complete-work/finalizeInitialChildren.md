{% method %}

# finalizeInitialChildren

这里其实主要是事件绑定然后对于`form`组件：

- `input`
- `option`
- `select`
- `textarea`

设置初始`wrapper`

事件绑定放到[`event`]()里面讲解。

{% sample lang="js" %}

```js
export function finalizeInitialChildren(
  domElement: Instance,
  type: string,
  props: Props,
  rootContainerInstance: Container,
  hostContext: HostContext,
): boolean {
  setInitialProperties(domElement, type, props, rootContainerInstance)
  return shouldAutoFocusHostComponent(type, props)
}

export function setInitialProperties(
  domElement: Element,
  tag: string,
  rawProps: Object,
  rootContainerElement: Element | Document,
): void {
  const isCustomComponentTag = isCustomComponent(tag, rawProps)

  // TODO: Make sure that we check isMounted before firing any of these events.
  let props: Object
  switch (tag) {
    case 'iframe':
    case 'object':
      trapBubbledEvent(TOP_LOAD, domElement)
      props = rawProps
      break
    case 'video':
    case 'audio':
      // Create listener for each media event
      for (let i = 0; i < mediaEventTypes.length; i++) {
        trapBubbledEvent(mediaEventTypes[i], domElement)
      }
      props = rawProps
      break
    case 'source':
      trapBubbledEvent(TOP_ERROR, domElement)
      props = rawProps
      break
    case 'img':
    case 'image':
    case 'link':
      trapBubbledEvent(TOP_ERROR, domElement)
      trapBubbledEvent(TOP_LOAD, domElement)
      props = rawProps
      break
    case 'form':
      trapBubbledEvent(TOP_RESET, domElement)
      trapBubbledEvent(TOP_SUBMIT, domElement)
      props = rawProps
      break
    case 'details':
      trapBubbledEvent(TOP_TOGGLE, domElement)
      props = rawProps
      break
    case 'input':
      ReactDOMInput.initWrapperState(domElement, rawProps)
      props = ReactDOMInput.getHostProps(domElement, rawProps)
      trapBubbledEvent(TOP_INVALID, domElement)
      // For controlled components we always need to ensure we're listening
      // to onChange. Even if there is no listener.
      ensureListeningTo(rootContainerElement, 'onChange')
      break
    case 'option':
      ReactDOMOption.validateProps(domElement, rawProps)
      props = ReactDOMOption.getHostProps(domElement, rawProps)
      break
    case 'select':
      ReactDOMSelect.initWrapperState(domElement, rawProps)
      props = ReactDOMSelect.getHostProps(domElement, rawProps)
      trapBubbledEvent(TOP_INVALID, domElement)
      // For controlled components we always need to ensure we're listening
      // to onChange. Even if there is no listener.
      ensureListeningTo(rootContainerElement, 'onChange')
      break
    case 'textarea':
      ReactDOMTextarea.initWrapperState(domElement, rawProps)
      props = ReactDOMTextarea.getHostProps(domElement, rawProps)
      trapBubbledEvent(TOP_INVALID, domElement)
      // For controlled components we always need to ensure we're listening
      // to onChange. Even if there is no listener.
      ensureListeningTo(rootContainerElement, 'onChange')
      break
    default:
      props = rawProps
  }

  assertValidProps(tag, props)

  setInitialDOMProperties(
    tag,
    domElement,
    rootContainerElement,
    props,
    isCustomComponentTag,
  )

  switch (tag) {
    case 'input':
      // TODO: Make sure we check if this is still unmounted or do any clean
      // up necessary since we never stop tracking anymore.
      inputValueTracking.track((domElement: any))
      ReactDOMInput.postMountWrapper(domElement, rawProps, false)
      break
    case 'textarea':
      // TODO: Make sure we check if this is still unmounted or do any clean
      // up necessary since we never stop tracking anymore.
      inputValueTracking.track((domElement: any))
      ReactDOMTextarea.postMountWrapper(domElement, rawProps)
      break
    case 'option':
      ReactDOMOption.postMountWrapper(domElement, rawProps)
      break
    case 'select':
      ReactDOMSelect.postMountWrapper(domElement, rawProps)
      break
    default:
      if (typeof props.onClick === 'function') {
        // TODO: This cast may not be sound for SVG, MathML or custom elements.
        trapClickOnNonInteractiveElement(((domElement: any): HTMLElement))
      }
      break
  }
}
```

{% endmethod %}
