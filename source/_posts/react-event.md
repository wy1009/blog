---
title: 全是代码——React 事件机制
date: 2019-06-21 20:27:04
categories: [框架/库/工具, React.js]
tags: [JavaScript, React.js, 源码]
---

起因其实是我用了 React Swiper 库。该库虽然挂着 React 的名头，但其实完全是由 Swiper.js 包装而成的，因而内里的事件机制仍旧是基于**原生**的。这导致我发现，我无法正常地将事件从 React 组件中阻止冒泡到 Swiper 的部分。

为了解决这个问题，很快，我从文档中查阅到，React 中的事件是由自己合成的“合成事件”，事件上的 `e.nativeEvent` 才是真正的原生事件。于是，我就尝试在事件处理函数中执行 `e.nativeEvent.stopPropagation()`，却仍旧无法阻止冒泡。

要搞清楚这个原因，就需要了解 React 的事件机制。

React 的事件机制是基于事件委托的。所有 React 的事件监听实际都是被绑定在 document 上的。

## 例子

本文其实就是对 React 事件机制部分的代码按照执行顺序解读。使用的例子代码如下：

``` JavaScript
// App.js

import React from 'react'
import Child from './Child'
import './App.css'

class App extends React.Component {

  clickHandler = (e) => {
    console.log('click callback', e)
  }

  render() {
    return (
      // 使用 <article> 是为了调试时更方便地一眼看出是什么标签
      // 其实该标签在语义化上不应被这样使用
      <article onClick={this.clickHandler}>
        <Child />
      </article>
    )
  }
}

export default App
```

``` JavaScript
// Child.js

import React from 'react'

class Child extends React.Component {
  childClickHandler = (e) => {
    console.log('child click callback', e)
  }

  render() {
    return (
      <div
        className="box"
        onClick={this.childClickHandler}
      >请点击</div>
    )
  }
}

export default Child
```

## 事件处理函数的绑定

### 从 `setInitialDOMProperties` 开始

``` JavaScript
function setInitialDOMProperties(
  tag: string,
  domElement: Element,
  rootContainerElement: Element | Document,
  nextProps: Object,
  isCustomComponentTag: boolean,
): void {
    // 省略部分代码
    if (registrationNameModules.hasOwnProperty(propKey)) {
      if (nextProp != null) {
        if (__DEV__ && typeof nextProp !== 'function') {
          warnForInvalidEventListener(propKey, nextProp);
        }
        ensureListeningTo(rootContainerElement, propKey);
      }
    }
    // 省略部分代码
  }
}
```

该方法处理实例的 props。`registrationNameModules` 的内容为：

![registrationNameModules](/images/react-event-registrationNameModules.png "registrationNameModules")

可以看出，如果 `propKey` 为 `registrationNameModules` 中的 key，则该 `propKey` 为事件相关，进入到对事件的处理步骤中。

就本文的例子而言，此时，我们的 `propKey` 应该为 `onClick`。

### `ensureListeningTo`

``` JavaScript
function ensureListeningTo(rootContainerElement, registrationName) {
  const isDocumentOrFragment =
    rootContainerElement.nodeType === DOCUMENT_NODE ||
    rootContainerElement.nodeType === DOCUMENT_FRAGMENT_NODE;
  const doc = isDocumentOrFragment
    ? rootContainerElement
    : rootContainerElement.ownerDocument;
  listenTo(registrationName, doc);
}
```

传入该方法的 `rootContainerElement` 即为 `#root`，即 React app 挂载的节点。`registrationName` 为 `onClick`。

该方法首先判断 `rootContainerElement` 是否为 document 或 [documentFragmemt](https://developer.mozilla.org/zh-CN/docs/Web/API/DocumentFragment)，是则直接使用，不是则找到该节点对应的 document。

接着，调用 `listenTo`，传入参数 `registrationName`（`onClick`），与 `doc`（document）。

### `listenTo`

``` JavaScript
export function listenTo(
  registrationName: string,
  mountAt: Document | Element,
) {
  const isListening = getListeningForDocument(mountAt);
  const dependencies = registrationNameDependencies[registrationName];

  for (let i = 0; i < dependencies.length; i++) {
    const dependency = dependencies[i];
    if (!(isListening.hasOwnProperty(dependency) && isListening[dependency])) {
        // 省略部分代码
        default:
          // By default, listen on the top level to all non-media events.
          // Media events don't bubble so adding the listener wouldn't do anything.
          const isMediaEvent = mediaEventTypes.indexOf(dependency) !== -1;
          if (!isMediaEvent) {
            trapBubbledEvent(dependency, mountAt);
          }
          break;
      }
      isListening[dependency] = true;
    }
  }
}
```

该方法用于指定相应事件的事件处理方法应在冒泡阶段还是捕获阶段触发。注意，此时的提到的“事件处理方法”是指绑定在 document 上用于分发事件的方法，而不是你在 React 组件中写的那个对应的事件处理方法。

`scroll`、`focus`、`blur`、`cancel`、`close` 事件在捕获阶段处理，也就是执行 `trapCapturedEvent` 方法。`invalid`、`submit`、`reset`以及媒体相关的事件不作处理。除此之外，其他所有时间都是在冒泡阶段处理的，也就是执行上文代码中的 `trapBubbledEvent` 方法。

同时可以注意一下，此处使用了 `isListening` 来记录对应处理方法是否已经被绑定在了 document 上。毕竟是事件委托，同样的一种事件只需要在 document 上绑定一次即可。

### `trapBubbledEvent`

``` JavaScript
export function trapBubbledEvent(
  topLevelType: DOMTopLevelEventType,
  element: Document | Element,
) {
  if (!element) {
    return null;
  }
  const dispatch = isInteractiveTopLevelEventType(topLevelType)
    ? dispatchInteractiveEvent
    : dispatchEvent;

  addEventBubbleListener(
    element,
    getRawEventName(topLevelType),
    // Check if interactive and wrap in interactiveUpdates
    dispatch.bind(null, topLevelType),
  );
}
```

`trapBubbledEvent` 其实就是在 document 上绑定了一个于冒泡阶段触发的方法 `dispatch`。根据该事件是否是交互事件（即用户做交互的相关事件），`dispatch` 方法可能是 `dispatchInteractiveEvent` 或 `dispatchEvent`。`dispatchInteractiveEvent` 实际只是先行做了一些处理，比如事件的优先级等，最终还是会调用 `dispatchEvent` 方法。

这样，将 `dispatchEvent` 方法绑定在了 document 上，并将 `topLevelType`，即字符串 `click` 作为参数传入方法，我们的事件绑定阶段就结束了。

接下来，**如果你点击了“请点击”字样**，就会触发 document 上绑定的 `dispatchEvent` 方法。

## 事件处理函数的执行

### 事件分发

#### `dispatchEvent`

点击“请点击”之后，立即出发 document 上绑定的 `dispatchEvent`。

``` JavaScript
export function dispatchEvent(
  topLevelType: DOMTopLevelEventType, // 对应浏览器事件名，本例中为 ‘click’
  nativeEvent: AnyNativeEvent, // 原生事件
) {
  if (!_enabled) {
    return;
  }

  const nativeEventTarget = getEventTarget(nativeEvent); // 取出 event.target
  let targetInst = getClosestInstanceFromNode(nativeEventTarget); // 通过 event.target 取得对应的 fiberNode，这个关系是被提前储存起来的
  if (
    targetInst !== null &&
    typeof targetInst.tag === 'number' &&
    !isFiberMounted(targetInst)
  ) {
    // If we get an event (ex: img onload) before committing that
    // component's mount, ignore it for now (that is, treat it as if it was an
    // event on a non-React tree). We might also consider queueing events and
    // dispatching them after the mount.
    targetInst = null;
  }

  const bookKeeping = getTopLevelCallbackBookKeeping(
    topLevelType, // 对应浏览器事件名，本例中为 ‘click’
    nativeEvent, // 浏览器原生事件
    targetInst, // 触发浏览器事件的 DOM 节点（event.target）对应的 fiberNode
  );

  try {
    // Event queue being processed in the same cycle allows
    // `preventDefault`.
    batchedUpdates(handleTopLevel, bookKeeping);
  } finally {
    releaseTopLevelCallbackBookKeeping(bookKeeping);
  }
}
```

传入的 `topLevelType` 即字符串 `click`，`nativeEvent` 是原生事件。

通过 `getEventTarget` 从原生事件中取到触发事件的 DOM 节点（`nativeEvent.target`），然后通过 `getClosestInstanceFromNode` 取得该 DOM 节点对应的 fiberNode。DOM 节点对应的 fiberNode 是被提前记录在节点的属性中的。

如果该 DOM 节点没有对应 fiberNode，或该 fiberNode 没有被安装在页面或文档片段中，则不做任何处理。判断 fiberNode 有没有被正确安装的方式是，不断寻找该 fiberNode 的 `return` 关系的 fiberNode，直到找到 `HostRoot`。如果找不到，则说明该 fiberNode 没有被正确安装。

如果该 fiberNode 状态正常，则将 `topLevelType`（对应浏览器的事件名）、`nativeEvent`（浏览器原生事件）、`targetInst`（触发事件的 DOM 节点对应的 fiberNode）整合起来，作为一个 `bookKeeping`，并附上一个 `ancestors` 属性，该属性值为一个空数组。关于这个属性，我们下面细说。

#### `batchedUpdates`

整合好一个 `bookKeeping` 之后，我们将该 `bookKeeping` 与方法 `handleTopLevel` 一起传入方法 `batchedUpdates`，执行 `batchedUpdates`。

``` JavaScript
export function batchedUpdates(fn, bookkeeping) {
  // 省略部分代码
  return _batchedUpdatesImpl(fn, bookkeeping);
}

// _batchedUpdatesImpl 实际是该方法
function batchedUpdates<A, R>(fn: (a: A) => R, a: A): R {
  const previousIsBatchingUpdates = isBatchingUpdates;
  isBatchingUpdates = true;
  try {
    return fn(a);
  } finally {
    isBatchingUpdates = previousIsBatchingUpdates;
    if (!isBatchingUpdates && !isRendering) {
      performSyncWork();
    }
  }
}
```

可以看到，最终调用了 `fn`，`fn` 则为 `handleTopLevel`。

#### `handleTopLevel`

``` JavaScript
function handleTopLevel(bookKeeping) {
  let targetInst = bookKeeping.targetInst;

  // Loop through the hierarchy, in case there's any nested components.
  // It's important that we build the array of ancestors before calling any
  // event handlers, because event handlers can modify the DOM, leading to
  // inconsistencies with ReactMount's node cache. See #1105.
  let ancestor = targetInst;
  do {
    if (!ancestor) {
      bookKeeping.ancestors.push(ancestor);
      break;
    }
    const root = findRootContainerNode(ancestor);
    if (!root) {
      break;
    }
    bookKeeping.ancestors.push(ancestor);
    ancestor = getClosestInstanceFromNode(root);
  } while (ancestor);

  // bookKeeping.ancestors 列表中其实只有一个值，就是触发事件的 `e.target` 对应的 fiberNode
  // 详细解释见下文
  for (let i = 0; i < bookKeeping.ancestors.length; i++) {
    targetInst = bookKeeping.ancestors[i];
    runExtractedEventsInBatch(
      bookKeeping.topLevelType, // 对应浏览器事件名，本例中为 ‘click’
      targetInst, // e.target，即触发事件的 DOM 元素对应的 fiberNode
      bookKeeping.nativeEvent, // 原生事件对象
      getEventTarget(bookKeeping.nativeEvent), // e.target，即触发事件的 DOM 元素
    );
  }
}
```

**关于这个方法，我有一个奇妙的个人想法。**

首先，看上半部分，也就是给 `ancestor` 填值的部分。你会发现，这部分代码首先会将触发事件的 DOM 元素对应的 fiberNode 本身放入 `ancestor` 列表中，然后取 `HostRoot`，也就是页面上的 `#root`，对 `#root` 执行 `getClosestInstanceFromNode`，而这个方法对 `#root` 执行必然会返回 `null`。

``` JavaScript
/**
 * Given a DOM node, return the closest ReactDOMComponent or
 * ReactDOMTextComponent instance ancestor.
 */
export function getClosestInstanceFromNode(node) {
  if (node[internalInstanceKey]) {
    return node[internalInstanceKey];
  }

  // 传入根节点，进入本段代码
  while (!node[internalInstanceKey]) {
    // 因为已经是 HostRoot 节点，无论如何向上寻找，node[internalInstanceKey] 始终不会有值
    // 最终会返回 null
    if (node.parentNode) {
      node = node.parentNode;
    } else {
      // Top of the tree. This node must not be part of a React tree (or is
      // unmounted, potentially).
      return null;
    }
  }

  let inst = node[internalInstanceKey];
  if (inst.tag === HostComponent || inst.tag === HostText) {
    // In Fiber, this will always be the deepest root.
    return inst;
  }

  return null;
}
```

这样，就会导致 `handleTopLevel` 方法中给 `ancestor` 列表里塞的值，永远都只有一个，就是触发事件元素对应的那个 fiberNode。对，就一个，就是这个。所以注释里写的什么“保存 DOM 结构”根本不对，你列表里就保存了这么一个 fiberNode，保存啥结构了你就保存……

考虑到这段代码不同行提交记录的时间差距相当大，我估计这是一个没改干净的历史遗留问题。跑着正常，但意思看着是不对的。

所以后面的循环也变得毫无意义，其实就是对触发事件的那一个 fiberNode 做的操作。做的操作是什么呢？也就是调用 `runExtractedEventsInBatch`。

#### `runExtractedEventsInBatch`

``` JavaScript
export function runExtractedEventsInBatch(
  topLevelType: TopLevelType, // 对应浏览器事件名，本例中为 ‘click’
  targetInst: null | Fiber, // e.target，即触发事件的 DOM 元素对应的 fiberNode
  nativeEvent: AnyNativeEvent, // 原生事件对象
  nativeEventTarget: EventTarget, // e.target，即触发事件的 DOM 元素
) {
  // 第一部分：提取事件部分，此时生成并提取合成事件
  // 开始执行
  const events = extractEvents(
    topLevelType,
    targetInst,
    nativeEvent,
    nativeEventTarget,
  );

  // 第二部分，执行事件监听方法
  runEventsInBatch(events, false);
}
```

这段代码分为两部分重要操作。第一部分是利用 `extractEvents` 方法生成并提取合成事件，需要传入事件名、触发事件 DOM 元素对应的 fiberNode，原生事件，以及触发事件的 DOM 元素。

让我们先看看这个方法。

### 提取合成事件

#### 获取当前类型事件应执行的 `extractEvents`（提取事件方法）

##### `extractEvents`

``` JavaScript
function extractEvents(
  topLevelType: TopLevelType, // 对应浏览器事件名，本例中为 ‘click’
  targetInst: null | Fiber, // e.target，即触发事件的 DOM 元素对应的 fiberNode
  nativeEvent: AnyNativeEvent, // 原生事件对象
  nativeEventTarget: EventTarget, // e.target，即触发事件的 DOM 元素
): Array<ReactSyntheticEvent> | ReactSyntheticEvent | null {
  let events = null;
  for (let i = 0; i < plugins.length; i++) {
    // Not every plugin in the ordering may be loaded at runtime.
    // plugins 即为可能的 plugin 列表，包含 extractEvents 等参数
    const possiblePlugin: PluginModule<AnyNativeEvent> = plugins[i];
    if (possiblePlugin) {
      const extractedEvents = possiblePlugin.extractEvents(
        topLevelType,
        targetInst,
        nativeEvent,
        nativeEventTarget,
      );
      if (extractedEvents) {
        events = accumulateInto(events, extractedEvents);
      }
    }
  }
  return events;
}
```

`plugins` 总共有五种，分别是 `SimpleEventPlugin` 、 `EnterLeaveEventPlugin` 、 `ChangeEventPlugin` 、 `SelectEventPlugin`、 `BeforeInputEventPlugin` 五种，判断事件属于哪一种 Plugin 是通过每种 Plugin 对应的 `extractEvents` 判断的。如果事件属于对应 Plugin，则该 Plugin 对应的 `extractEvents` 方法会生成并返回对应的合成事件，否则返回 `undefined`。

以 click 事件为例，循环 Plugin 列表，首先执行 `SimpleEventPlugin` 对象上的 `extractEvents` 方法。

##### `SimpleEventPlugin.extractEvents`

``` JavaScript
const SimpleEventPlugin: PluginModule<MouseEvent> & {
  isInteractiveTopLevelEventType: (topLevelType: TopLevelType) => boolean,
} = {
  // 省略部分属性
  extractEvents: function(
    topLevelType: TopLevelType,
    targetInst: null | Fiber,
    nativeEvent: MouseEvent,
    nativeEventTarget: EventTarget,
  ): null | ReactSyntheticEvent {
    const dispatchConfig = topLevelEventsToDispatchConfig[topLevelType];
    // 本例中，dispatchConfig 为 {
    //   dependencies: ["click"]
    //   isInteractive: true,
    //   phasedRegistrationNames: {
    //     bubbled: "onClick",
    //     captured: "onClickCapture",
    //   },
    // }
    if (!dispatchConfig) {
      return null;
    }
    let EventConstructor;
    switch (topLevelType) {
      // 省略部分代码
      case DOMTopLevelEventTypes.TOP_CLICK:
        // Firefox creates a click event on right mouse clicks. This removes the
        // unwanted click events.
        if (nativeEvent.button === 2) {
          return null;
        }
        // 省略部分代码
        EventConstructor = SyntheticMouseEvent;
        break;
      // 省略部分代码
    }
    const event = EventConstructor.getPooled(
      dispatchConfig,
      targetInst,
      nativeEvent,
      nativeEventTarget,
    );
    accumulateTwoPhaseDispatches(event);
    return event;
  },
};
```

可以看到，在事件为 click 的情况下，`EventConstructor` 被赋值为 `SyntheticMouseEvent`。赋值 `EventConstructor` 之后，调用 `EventConstructor.getPooled`。那么，首先应该来看看， `SyntheticMouseEvent` 是什么呢？

#### 创建或从池中取出合成事件实例

##### `SyntheticMouseEvent`

``` JavaScript
const SyntheticMouseEvent = SyntheticUIEvent.extend({
  // 属性若干
});
```

``` JavaScript
const SyntheticUIEvent = SyntheticEvent.extend({
  // 属性若干
});
```

从字面可以看出，`SyntheticMouseEvent` 继承于 `SyntheticUIEvent`，`SyntheticUIEvent` 继承于 `SyntheticEvent`。那么，这个继承关系是如何实现的呢？

``` JavaScript
SyntheticEvent.extend = function(Interface) {
  const Super = this;

  const E = function() {};
  E.prototype = Super.prototype;
  const prototype = new E();

  function Class() {
    return Super.apply(this, arguments);
  }
  Object.assign(prototype, Class.prototype);
  Class.prototype = prototype;
  Class.prototype.constructor = Class;

  Class.Interface = Object.assign({}, Super.Interface, Interface);
  Class.extend = Super.extend;
  addEventPoolingTo(Class);

  return Class;
};
```

从代码上看，继承方式正是最经典的寄生组合继承，详情可见[《JavaScript面向对象的程序设计-继承》](https://wy1009.github.io/2017/04/16/JavaScript面向对象的程序设计-继承/)。

这样，我们就知道了，`SyntheticMouseEvent` 实际上是一个继承于 `SyntheticUIEvent` 与 `SyntheticEvent` 的构造函数。

在通过条件判断，取得事件对应的构造函数之后，我们继续执行 `extractEvents` 接下来的代码，即 `EventConstructor.getPooled`。

##### `EventConstructor.getPooled`

``` JavaScript
function getPooledEvent(dispatchConfig, targetInst, nativeEvent, nativeInst) {
  const EventConstructor = this;

  // 如果事件池中有事件，直接取出一个来使用
  if (EventConstructor.eventPool.length) {
    const instance = EventConstructor.eventPool.pop();
    EventConstructor.call(
      instance,
      dispatchConfig,
      targetInst,
      nativeEvent,
      nativeInst,
    );
    return instance;
  }

  // 如果没有，则新建一个事件
  return new EventConstructor(
    // dispatchConfig 与 'click' 相关的一些配置。
    // dispatchConfig: {
    //   dependencies: ['click'],
    //   isInteractive: true,
    //   phasedRegistrationNames: {
    //     bubbled: 'onClick',
    //     captured: 'onClickCapture',
    //   },
    // }
    dispatchConfig,
    targetInst,
    nativeEvent,
    nativeInst,
  );
}
```

可以看到，React 的事件机制做了一个性能上的优化，即“事件池”。如果事件池中有事件，则直接取出使用。如果没有，则新建一个事件。剧透一下，这个新建的事件在使用完毕后，就会被扔进事件池。

在本例中，此时，我们的事件池中还没有事件，因此执行 `new EventConstructor(arguments...)`。

##### `SyntheticEvent`

执行 `new EventConstructor(arguments...)`，该 `EventConstructor` 继承于 `SyntheticEvent`，因而执行该方法。

``` JavaScript
function SyntheticEvent(
  dispatchConfig,
  targetInst,
  nativeEvent,
  nativeEventTarget,
) {
  this.dispatchConfig = dispatchConfig;
  this._targetInst = targetInst;
  this.nativeEvent = nativeEvent;

  const Interface = this.constructor.Interface; // 感觉该“接口”对象中包含的实际上是一个 React 事件所需要的全部属性
  for (const propName in Interface) {
    if (!Interface.hasOwnProperty(propName)) {
      continue;
    }
    const normalize = Interface[propName];
    if (normalize) {
      // 如果接口对象的对应属性不为空，则执行该属性，将其返回值赋给事件实例的对应属性
      this[propName] = normalize(nativeEvent);
    } else {
      // 如果接口对象的对应属性为空
      if (propName === 'target') {
        // 如果该属性为 target,则直接将我们原生事件的 event.target 赋值给该属性
        // 不写注释我也不知道为什么要单独把这个属性抽出来赋值呀……只能猜测是有可能变动了
        this.target = nativeEventTarget;
      } else {
        // 否则，直接将原生对象的对应属性赋值给该属性
        this[propName] = nativeEvent[propName];
      }
    }
  }

  const defaultPrevented =
    nativeEvent.defaultPrevented != null
      ? nativeEvent.defaultPrevented
      : nativeEvent.returnValue === false;
  if (defaultPrevented) {
    this.isDefaultPrevented = functionThatReturnsTrue;
  } else {
    this.isDefaultPrevented = functionThatReturnsFalse;
  }
  this.isPropagationStopped = functionThatReturnsFalse;
  return this;
}
```

可以看到，这个方法基本就是将预设好的“接口对象”，即 `Interface` 对象，中的内容一一遍历，并将原生事件的对应属性/预设接口的执行结果赋值给 React 事件实例的过程。

执行完这里，我们就执行完了 `new EventConstructor(argument...)`，也就是执行完了更上一层函数的 `EventConstructor.getPooled(arguments...)`，得到了一个 React 合成事件实例。

``` JavaScript
// extractEvents 方法的代码片段

// 已执行完
const event = EventConstructor.getPooled(
  dispatchConfig,
  targetInst,
  nativeEvent,
  nativeEventTarget,
);

// 下一步开始执行
accumulateTwoPhaseDispatches(event);
return event;
```

下一步，我们应开始执行 `accumulateTwoPhaseDispatches(event)`，即在创建的合成事件上保存对应事件监听函数的过程。

#### 在合成事件实例上保存事件监听方法

##### `accumulateTwoPhaseDispatches`

积累两个阶段的 dispatch。

`accumulateTwoPhaseDispatches(event)` -> `forEachAccumulated(events, accumulateTwoPhaseDispatchesSingle)` （判断传入的 event 是否为数组，如果是，逐项执行 `accumulateTwoPhaseDispatchesSingle(event)` ，如果不是，则直接执行 `accumulateTwoPhaseDispatchesSingle(event)` ） -> `accumulateTwoPhaseDispatchesSingle(event)` -> `traverseTwoPhase(event._targetInst, accumulateDirectionalDispatches, event)` 。

##### `traverseTwoPhase`

遍历两个阶段。

``` JavaScript
/**
 * Simulates the traversal of a two-phase, capture/bubble event dispatch.
 * @param inst 触发事件的 dom 元素对应的 fiberNode
 * @param fn accumulateDirectionalDispatches 方法
 * @param arg event，之前我们合成的 React 事件
 */
export function traverseTwoPhase(inst, fn, arg) {
  const path = [];

  // 循环取触发事件的 dom 元素的 fiberNode 的父 fiberNode（仅包括 HostComponent 类型的节点）
  // 形成一条 fiberNode 链条。详见下面对 getParent 方法的描述。
  // 本例 path 最终为 [div, article]
  while (inst) {
    path.push(inst);
    inst = getParent(inst);
  }

  let i;
  for (i = path.length; i-- > 0; ) {
    fn(path[i], 'captured', arg);
  }
  for (i = 0; i < path.length; i++) {
    fn(path[i], 'bubbled', arg);
  }
}
```

`getParent` 代码如下。

``` JavaScript
function getParent(inst) {
  do {
    inst = inst.return; // TODO: If this is a HostRoot we might want to bail out.
    // That is depending on if we want nested subtrees (layers) to bubble
    // events to their parent. We could also go through parentNode on the
    // host node but that wouldn't work for React Native and doesn't let us
    // do the portal feature.
  } while (inst && inst.tag !== HostComponent);

  if (inst) {
    return inst;
  }

  return null;
}
```

`return` 关系详见[对 Fiber 的介绍](https://wy1009.github.io/2018/12/02/a-cartoon-intro-to-fiber/)。可以看到，该方法会向上追溯最近的 HostComponent，即能够用于绑定事件的 fiberNode，不包括不能绑定事件的组件节点、文本节点等，然后返回。

这样，在 `traverseTwoPhase` 方法中，我们就得到了一条用于传递事件的节点链条。

接下来，我们按模拟捕获和冒泡的顺序，分别从根节点开始循环到触发事件的节点/从触发事件的节点循环到根节点，对每个节点执行 `accumulateDirectionalDispatches` 方法。

##### `accumulateDirectionalDispatches`

``` JavaScript
/**
 * Tags a `SyntheticEvent` with dispatched listeners. Creating this function
 * here, allows us to not have to bind or create functions for each event.
 * Mutating the event's members allows us to not have to create a wrapping
 * "dispatch" object that pairs the event with the listener.
 */
function accumulateDirectionalDispatches(inst, phase, event) {
  // 通过上步循环到的 fiberNode、阶段（捕获 or 冒泡）以及 event（事件实例）
  // 得到节点上的事件监听函数
  const listener = listenerAtPhase(inst, event, phase);

  if (listener) {
    // 如果有 event 对应的事件监听方法，则将该方法积累在 event._dispatchListeners 属性中
    // accumulateInfo 方法很简单，就是组合传入的值。
    // 这个方法相当于以数组的形式将所有的事件监听方法保存在 event 上
    event._dispatchListeners = accumulateInto(
      event._dispatchListeners,
      listener,
    );

    // 同样，将所有绑定了事件监听方法的 fiberNode 以数组的形式保存在 event 上
    // 可以与事件监听方法一一对应
    event._dispatchInstances = accumulateInto(event._dispatchInstances, inst);
  }
}
```

这样，在循环执行了这个方法之后，就会按先捕获再冒泡的顺序，循环所有链条上的节点，将事件监听方法按顺序储存在事件实例上，并与绑定事件监听方法的节点一一对应。

<img src="/images/react-event-01.png" style="width: 300px;" />

到这里，前面的 `accumulateTwoPhaseDispatches` 也方法也完全执行完毕，返回了一个 `event`，即 React 的合成事件实例。

``` JavaScript
// extractEvents 方法的代码片段

// 已执行完
const event = EventConstructor.getPooled(
  dispatchConfig,
  targetInst,
  nativeEvent,
  nativeEventTarget,
);

// 已执行完
accumulateTwoPhaseDispatches(event);

// 返回合成事件实例
return event;
```

至此，`extractEvents` 方法全部执行完毕，接着执行下一步，`runEventsInBatch` 方法。即真正意义上调用事件监听方法。

``` JavaScript
export function runExtractedEventsInBatch(
  topLevelType: TopLevelType, // 对应浏览器事件名，本例中为 ‘click’
  targetInst: null | Fiber, // e.target，即触发事件的 DOM 元素对应的 fiberNode
  nativeEvent: AnyNativeEvent, // 原生事件对象
  nativeEventTarget: EventTarget, // e.target，即触发事件的 DOM 元素
) {
  // 第一部分：提取事件部分，此时生成并提取合成事件
  // 已经执行完毕
  const events = extractEvents(
    topLevelType,
    targetInst,
    nativeEvent,
    nativeEventTarget,
  );

  // 第二部分，执行事件监听方法
  // 开始执行
  runEventsInBatch(events, false);
}
```

### 执行事件回调

#### `runEventsInBatch`

``` JavaScript
function runEventsInBatch(events) {
  if (events !== null) {
    eventQueue = accumulateInto(eventQueue, events);
  } // Set `eventQueue` to null before processing it so that we can tell if more
  // events get enqueued while processing.


  var processingEventQueue = eventQueue;
  eventQueue = null;

  if (!processingEventQueue) {
    return;
  }

  forEachAccumulated(processingEventQueue, executeDispatchesAndReleaseTopLevel);
  !!eventQueue ? invariant(false, 'processEventQueue(): Additional events were enqueued while processing an event queue. Support for this has not yet been implemented.') : void 0; // This would be a good time to rethrow if any of the event handlers threw.

  rethrowCaughtError();
}
```

可以看到，该方法其实是可以同时传入多个事件，形成一个事件队列的。然后对事件队列中的事件分别执行 `executeDispatchesAndReleaseTopLevel(event)` ，`executeDispatchesAndReleaseTopLevel(event)` 中只有一行代码，即执行 `executeDispatchesAndRelease(event)`。

##### executeDispatchesAndRelease

``` JavaScript
var executeDispatchesAndRelease = function (event) {
  if (event) {
    executeDispatchesInOrder(event);

    if (!event.isPersistent()) {
      event.constructor.release(event);
    }
  }
};
```

调用 `executeDispatchesInOrder` 。在调用过后，如果事件没有被标记为持久化（`if (!event.isPersistent()) {}`），则释放事件。

##### executeDispatchesInOrder

``` JavaScript
/**
 * Standard/simple iteration through an event's collected dispatches.
 */
export function executeDispatchesInOrder(event) {
  const dispatchListeners = event._dispatchListeners;
  const dispatchInstances = event._dispatchInstances;

  if (Array.isArray(dispatchListeners)) {
    // 如果该事件触发的事件监听方法不止一个，则依次执行
    for (let i = 0; i < dispatchListeners.length; i++) {
      // 如果事件被阻止传播，则不再继续执行其他事件监听方法
      // 注意，该属性虽然在中文语境中常被称为“阻止冒泡”，但其在冒泡阶段和捕获阶段均有效
      // 其实是“阻止传播”，而不是单单阻止冒泡
      if (event.isPropagationStopped()) {
        break;
      }
      // Listeners and Instances are two parallel arrays that are always in sync.
      executeDispatch(
        event,
        dispatchListeners[i],
        dispatchInstances[i],
      );
    }
  } else if (dispatchListeners) {
    // 否则执行一个事件监听方法即可
    executeDispatch(event, dispatchListeners, dispatchInstances);
  }

  // 执行完事件监听方法之后，清空事件记录的事件监听方法即相应绑定方法的 fiberNode
  event._dispatchListeners = null;
  event._dispatchInstances = null;
}
```

##### executeDispatch

``` JavaScript
/**
 * Dispatch the event to the listener.
 * @param {SyntheticEvent} event SyntheticEvent to handle，当前的合成事件实例
 * @param {function} listener Application-level callback，事件监听方法
 * @param {*} inst Internal component instance，绑定事件监听方法的 fiberNode
 */
function executeDispatch(event, listener, inst) {
  const type = event.type || 'unknown-event'; // 设置事件名
  event.currentTarget = getNodeFromInstance(inst); // 设置 e.currentTarget
  invokeGuardedCallbackAndCatchFirstError(type, listener, undefined, event);
  event.currentTarget = null; // 执行完事件监听方法之后，清空 e.currentTarget，便于下一个事件监听方法执行前更改 currentTarget
}
```

`invokeGuardedCallbackAndCatchFirstError` 执行 `invokeGuardedCallback` 并 catch Error， `invokeGuardedCallback` 调用 `invokeGuardedCallbackImpl` 。

##### invokeGuardedCallback

``` JavaScript
let invokeGuardedCallbackImpl = function<A, B, C, D, E, F, Context>(
  name: string | null,
  func: (a: A, b: B, c: C, d: D, e: E, f: F) => mixed,
  context: Context,
  a: A,
  b: B,
  c: C,
  d: D,
  e: E,
  f: F,
) {
  const funcArgs = Array.prototype.slice.call(arguments, 3);
  try {
    // 事件监听函数被执行
    func.apply(context, funcArgs);
  } catch (error) {
    this.onError(error);
  }
};
```

`func` 即我们的 `listener`。可以看到，至此，事件监听函数终于被成功执行了。

### 将使用过的事件实例释放到资源池中

``` JavaScript
var executeDispatchesAndRelease = function (event) {
  if (event) {
    // 已执行完毕
    executeDispatchesInOrder(event);

    // 如果事件不是“可持久化”的，则释放事件
    if (!event.isPersistent()) {
      event.constructor.release(event);
    }
  }
};
```

`event.constructor.release`，即为 `releasePooledEvent` 方法。

#### releasePooledEvent

``` JavaScript
function releasePooledEvent(event) {
  const EventConstructor = this;
  invariant(
    event instanceof EventConstructor,
    'Trying to release an event instance into a pool of a different type.',
  );
  event.destructor();
  if (EventConstructor.eventPool.length < EVENT_POOL_SIZE) {
    EventConstructor.eventPool.push(event);
  }
}
```

#### event.distructor

``` JavaScript
destructor: function () {
  var Interface = this.constructor.Interface;

  for (var propName in Interface) {
    {
      Object.defineProperty(this, propName, getPooledWarningPropertyDefinition(propName, Interface[propName]));
    }
  }

  this.dispatchConfig = null;
  this._targetInst = null;
  this.nativeEvent = null;
  this.isDefaultPrevented = functionThatReturnsFalse;
  this.isPropagationStopped = functionThatReturnsFalse;
  this._dispatchListeners = null;
  this._dispatchInstances = null;
  {
    Object.defineProperty(this, 'nativeEvent', getPooledWarningPropertyDefinition('nativeEvent', null));
    Object.defineProperty(this, 'isDefaultPrevented', getPooledWarningPropertyDefinition('isDefaultPrevented', functionThatReturnsFalse));
    Object.defineProperty(this, 'isPropagationStopped', getPooledWarningPropertyDefinition('isPropagationStopped', functionThatReturnsFalse));
    Object.defineProperty(this, 'preventDefault', getPooledWarningPropertyDefinition('preventDefault', function () {}));
    Object.defineProperty(this, 'stopPropagation', getPooledWarningPropertyDefinition('stopPropagation', function () {}));
  }
}
```

在这个方法中，清空当前合成事件实例上的所有属性。然后回到上一步。

``` JavaScript
function releasePooledEvent(event) {
  const EventConstructor = this;
  invariant(
    event instanceof EventConstructor,
    'Trying to release an event instance into a pool of a different type.',
  );

  // 已执行完毕
  event.destructor();

  // 如果池中的事件小于一个预设值，则池子还没放满，将清空后的事件实例放入池中
  if (EventConstructor.eventPool.length < EVENT_POOL_SIZE) {
    EventConstructor.eventPool.push(event);
  }
}
```

如果池子中的事件数量小于一个预设值，则池子还没放满，将清空后的事件放入池中。下次提取合成事件时，就可以直接从池中取，而不需要再次创建。

#### releaseTopLevelCallbackBookKeeping

最终，我们连我们存下的 `bookKeeping` 也需要释放。

``` JavaScript
function releaseTopLevelCallbackBookKeeping(instance) {
  instance.topLevelType = null;
  instance.nativeEvent = null;
  instance.targetInst = null;
  instance.ancestors.length = 0;
  if (callbackBookkeepingPool.length < CALLBACK_BOOKKEEPING_POOL_SIZE) {
    callbackBookkeepingPool.push(instance);
  }
}
```

操作同样，都是先清空实例，然后看池子中是否还有位置，如果有位置，则放入池中。

至此，React 事件机制就已经介绍完成了。
