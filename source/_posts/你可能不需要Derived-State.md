---
title: 你可能不需要 Derived State
date: 2018-11-13 20:37:09
categories: [框架/库/工具, React.js]
tags: [JavaScript, React.js]
---

React 16.4 [为 `getDerivedStateFromProps` 修复了一个 bug](https://reactjs.org/blog/2018/05/23/react-v-16-4.html#bugfix-for-getderivedstatefromprops)，这个 bug 会导致一些在 React 组件中存在的 bug 不断重现。如果本次更新暴露了你的应用正在使用反模式（不推荐的做法）且在本次更新后可能出现问题，我们对此感到抱歉。在这篇博文中，我们会解释一些使用 derived state 时常见的反模式和我们更推荐的替代做法。

在很长一段时间里，如果想在 props 改变时更新 state，且不造成多余的渲染，我们只能依靠生命周期 `componentWillReceiveProps`。在 16.3 版本，[我们介绍了一种可以替代 `componentWillReceiveProps` 的生命周期—— `getDerivedStateFromProps`](https://reactjs.org/blog/2018/03/29/react-v-16-3.html#component-lifecycle-changes) ——来更安全地解决同样的问题。与此同时，我们发现大家在使用这两种生命周期时存在很多错误的观念，也发现了一些会造成微妙而令人困扰的 bug 的反模式。16.4 版本中对 `getDerivedStateFromProps` 的 bugfix 使 derived state 更加可预料，因而误用它所产生的结果也更加容易被注意到。

> ** 注意 **
> 本文描述的所有反模式都同时适用于旧的 `componentWillReceiveProps` 和新的 `getDerivedStateFromProps`。

<!-- more -->

## 什么时候使用 derived state
`getDerivedStateFromProps` 只用于一个目的——让组件可以在 **props 改变**时更新内部的 state。我们之前的博文提供了一些例子，比如 [基于改变中的 offset prop 记录当前的滚动方向](https://reactjs.org/blog/2018/03/27/update-on-async-rendering.html#updating-state-based-on-props) 或者 [加载代码指定的外部数据](https://reactjs.org/blog/2018/03/27/update-on-async-rendering.html#fetching-external-data-when-props-change)。

我们没有举很多例子，因为作为一个普遍的规则，**应该尽量少而谨慎地使用 derived state。**在 derived state 上出的问题，可以减少的无非两种：1. 无条件从 props 更新数据到 state；2. 不管 props 和 state 是否匹配都更新 state。（两种我们在下面都会详细讲。）

* 如果你使用 derived state 是为了记录仅基于当前 props 的计算结果，你不需要 derived state。看看下面的 [试试 memoization](https://reactjs.org/blog/2018/06/07/you-probably-dont-need-derived-state.html#what-about-memoization)。
* 如果你无条件更新 state，或者不管 props 和 state 是否匹配都更新 state，那你的组件恐怕太经常重置 state 了。下面会详细讲。

## 使用 derived state 时的常见 bug
术语“受控”和“非受控”通常指表格输入，但也可以用来描述组件的数据存在哪里。通过 props 传的数据可以被称为是“**受控**”的（因为父组件控制了数据），仅存在于内部 state 的数据可以被认为是“**非受控**”的（因为父组件不能直接改变它）。

使用 derived state 最常见的错误是把“受控”和“非受控”混在一起。在 state 同时被 derived state 和 `setState` 更新的时候，数据没有真实的单一来源。上面提到的 [外部数据加载示例](https://reactjs.org/blog/2018/03/27/update-on-async-rendering.html#fetching-external-data-when-props-change) 听起来可能类似，但在一些重要的方面并不相同。在加载示例中，对于作为来源的 props 和加载中的 state 都有干净的真实来源。当作为来源的 props 改变了，加载的 state **总是**都应该被重写。反过来，state 只会在 props **改变**时被重写，在重写之外的其他方面才会被组件本身管理。

不遵守这些约束中的任何一个都会出现问题，问题通常以两种形式出现。

### 反模式：无条件把 props 复制到 state 上
一个常见的错误观念是，`getDerivedStateFromProps` 和 `componentWillReceiveProps` 只在 props 改变的时候被调用。这两个生命周期其实会在父组件渲染的时候被调用，不管 props 是否和之前不一样。因此，使用这两个生命周期无条件重写 state 是不安全的。**这样做会导致 state 更新丢失。**

让我们考虑一个例子来证明这个问题。以下是一个 `EmailInput` 组件，在 state 中“镜像”了一个 email prop：

``` JavaScript
class EmailInput extends Component {
  state = { email: this.props.email }

  render() {
    return <input onChange={this.handleChange} value={this.state.email} />
  }

  handleChange = event => {
    this.setState({ email: event.target.value })
  }

  componentWillReceiveProps(nextProps) {
    // 这里将擦掉所有本地的 state 更新
    // 不要这样做
    this.setState({ email: nextProps.email })
  }
}
```

一开始，这个组件可能看起来没问题。state 被 props 指定的值初始化，当我们在 `<input>` 中打字的时候，state 会更新。但如果父组件渲染，我们在 `<input>` 中打的字就会丢失！（[demo](https://reactjs.org/blog/2018/03/27/update-on-async-rendering.html#fetching-external-data-when-props-change)）即使我们在重置 state 之前对比 `nextProps.email !== this.state.email` 也是一样的。

在这个简单的例子中，增加 `shouldComponentUpdate` 使只有 email prop 改变的时候才渲染可以解决这个问题。然而在实践中，组件通常接受多个 props，其他 prop 的改变也会触发渲染，导致不正确的重置。方法和对象的 props 也经常被内联创建，使我们很难使 `shouldComponentUpdate` 仅在真正改变的时候返回 true。（[demo](https://codesandbox.io/s/jl0w6r9w59)）。因此， `shouldComponentUpdate` 最好被用于性能优化，而不是确定 derived state 的正确性。

希望现在大家已经清楚为什么不能无条件把 props 复制到 state 上了。在讲可能的解决方法之前，我们先来看看另一个造成问题的同类反模式：如果我们只在 email prop 改变的时候更新 state 会怎么样呢？

### 反模式：当 props 改变的时候重置 state
继续上面的例子，我们可以只在 `props.email` 改变的时候更新 state：

``` JavaScript
class EmailInput extends Component {
  state = {
    email: this.props.email,
  }

  componentWillReceiveProps(nextProps) {
    if (nextProps.email !== this.props.email) {
      this.setState({
        email: nextProps.email,
      })
    }
  }
}
```

> **注意**
> 虽然上面的例子使用的是 `componentWillReceiveProps`，但道理在 `getDerivedStateFromProps` 中是一样的。

这样就有了很大的改善。现在组件只会在 props 改变的时候才会删除我们输入的内容了。

这样仍旧有一个小问题。想象有一个使用上面这个组件的密码管理应用。当在使用同一个 email 的两个账户详情之间切换的时候，input 将不会重置。这是因为两个账户传入的 prop 是一样的。这会让用户感到很奇怪，如果有两个 email 相同的账户，一个账户详情中没有保存的内容会在另一个账户的详情中出现。（[demo](https://codesandbox.io/s/mz2lnkjkrx)）

这样设计很不好，但是我们却很容易犯。（[我就曾经犯过](https://twitter.com/brian_d_vaughn/status/959600888242307072)）幸运的是，有两个更好的解决方法。两种方法的关键都是，对于任何一点数据，我们都需要选择一个拥有它的组件作为真实的来源，并且避免在其他组件中复制它。让我们挨个看看这两种解决方法。

## 更好的解决方法
### 推荐：完全受控组件
要避免上面提到的问题，一种方法是把 state 从组件中移除。如果 email prop 只作为 props 存在，那么我们就不需要担心和 state 有冲突。我们甚至可以把 `EmailInput` 转变为一个更轻量级的函数式组件：

``` JavaScript
function EmailInput(props) {
  return <input onChange={props.onChange} value={props.email} />
}
```

这种方法简化了我们的组件的实践，但如果我们仍旧想要存一个草稿值，父级表单组件需要手动做这件事。（[点击查看这种模式的 demo](https://codesandbox.io/s/7154w1l551)）（译者注：我觉得“草稿值”指的是用户正在操作而没有真正提交的值。）

### 推荐：带 key 的完全不受控组件
另一个解决方法是，让我们的组件完全拥有“草稿”的 email state。这样，组件能够始终接收一个 prop 作为初始值，但会忽视掉对于 prop 后续的更新：

``` JavaScript
class EmailInput extends Component {
  state = { email: this.props.defaultEmail }

  handleChange = (event) => {
    this.setState({ email: event.target.value })
  }

  render() {
    return <input onChange={this.handleChange} value={this.state.email} />
  }
}
```

为了能在其他情况下使用的时候重置值（比如我们设想的密码管理应用的情况），我们可以使用 React 的特殊属性：`key`。当 `key` 改变的时候，React 会[创建一个新的组件实例，而不是更新目前的这一个](https://reactjs.org/docs/reconciliation.html#keys)。key 通常被用于动态列表，但在这种情况下也很有用。在我们的例子中，我们可以使用 user ID 作为key，使得每次新用户被选择时都重新创建 email input。

``` HTML
<EmailInput defaultEmail={this.props.user.email} key={this.props.user.id}>
```

每次 ID 改变，`EmailInput` 组件都会被重新创建，它的 state 也会被重置为 `defaultValue` 的值。（[点击查看这种模式的 demo](https://codesandbox.io/s/6v1znlxyxn)）。用这种方法，你不需要给每一个 input 增加 `key`，直接给整个 form 表单加一个 `key` 恐怕更合理。每次 key 改变，所有在 form 表单里的组件都会被重新创建，state 也被更新为初始值。

在大多数情况下，如果需要重置 state，这都是最好的方法。

> **注意**
> 虽然这种方法听起来很慢，但是性能差距其实是微不足道的。如果组件在更新时有很重的逻辑，使用 key 甚至可以更快，因为该子树的 diff 被绕过了。

#### 可替代方法 1：使用一个 id prop 重置非受控组件
如果 `key` 因为某些原因不起作用（比如组件初始化的代价非常昂贵），一个可行的笨方法是在 `getDerviedStateFromProps` 中观察 user id 的更改：

``` JavaScript
class EmailInput extends Component {
  state = {
    email: this.props.defaultEmail,
    prevPropsUserID: this.props.userID,
  }

  static getDerivedStateFromProps(props, state) {
    if (props.userID !== state.prevPropsUserID) {
      return {
        prevPropsUserID: props.userID,
        email: props.defaultEmail,
      }
    }

    return null
  }
}
```

这样做也具有灵活性，能够只重置一部分内部的 state。（[点击查看这种模式的 demo](https://codesandbox.io/s/rjyvp7l3rq)）

> **注意**
> 虽然上面的例子展示的是 `getDerivedStateFromProps`，但同样的方法也可以被用于 `componentWillReceiveProps`。

#### 可替代方法 2：使用实例方法重置非受控组件
在更加少见的情况下，你可能需要重置 state，但是没有合适的 ID 作为 `key`。一个解决方案是在每次想要重置的时候，利用随机值或者自增的数字作为 `key`。另一个可行的方法是，暴露一个实例方法来命令式地重置内部的 state：

``` JavaScript
class EmailInput extends Component {
  state = {
    email: this.props.defaultEmail,
  }

  resetEmailForNewUsers(newEmail) {
    this.setState({
      email: newEmail,
    })
  }
}
```

接着，父组件可以[使用 ref 来调用这个方法](https://reactjs.org/docs/glossary.html#refs)。（[点击查看这种模式的 demo](https://codesandbox.io/s/l70krvpykl)）

Ref 在像这样的例子里很有用，但是一般来讲，我们建议少使用 Ref。即使是在这个 demo 中，这种命令式的方法也不理想，因为会造成两次渲染，而不是不使用时的一次。

### 总结
总结一下，当设计一个组件的时候，很重要的是决定它的数据是受控的还是非受控的。

比起试图**在 state 中“镜像” prop**，不如使组件**受控**，在父组件的 state 中合并两个不同的值。比如，比起子组件接收“用于提交的”（committed） `props.value` 并追踪一个“草稿”（draft）的 `state.value`，不如让父组件同时有 `state.draftValue` 和 `state.committedValue` 并直接控制子组件的值。这使得数据流更加明确而可预计。

对于**非受控**组件，如果你试图在一个特别的 prop（通常是 ID）改变时重置 state，你可以选择：

* **推荐：使用 key 属性重置所有内部的 state。**
* 可替代 1：仅重置特定的 state 字段，监听特殊值的变动（比如 `props.userID`）。
* 可替代 2：你也可以考虑退而求其次，用 refs 的命令式实例方法。

## 要不要试试 memoization 呢？
我们也看到 derived state 被用于确认某些用于渲染的值仅在 input 改变的时候重新计算。这种方法被称为 [`memoization`](https://en.wikipedia.org/wiki/Memoization)。

使用 derived state 做缓存（memoization）不一定不好，但通常不是最优解。管理 derived state 存在固有的复杂性，且每增加一个属性都会更加复杂。比如，如果我们给组件的 state 增加第二个 derived 字段，那么我们需要分别追踪这两者的更改。

来看一个例子，一个组件接收一个 prop——一个列表——并根据用户输入的查询字段渲染匹配的项。我们可以使用 derived state 储存过滤后的列表：

``` JavaScript
class Example extends Component {
  state = {
    filterText: '',
  }

  // 注意，这个例子不是推荐的用法
  // 在这个例子下面有我们推荐的方法
  static getDerivedStateFromProps(props, state) {
    if (
      props.list !== state.prevPropsList ||
      state.prevFilterText !== state.filterText
    ) {
      return {
        prevPropsList: props.list,
        prevFilterText: state.filterText,
        filteredList: props.list.filter(item => item.text.includes(state.filterText))
      }
    }

    return null
  }

  handleChange = event => {
    this.setState({ filterText: event.target.value })
  }

  render() {
    return (
      <Fragment>
        <input onChange={this.handleChange} value={this.state.filterText}>
        <ul>{this.state.filteredList.map(item => <li key={item.id}>{item.text}</li>)}</ul>
      </Fragment>
    )
  }
}
```

这种做法避免了对 `filteredList` 不必要的重新计算。但是这种做法比正常要复杂得多。为了正确地更新过滤后的列表，需要同时分别追踪和监听 prop 和 state 中的变化。在这个例子中，我们可以用 `PureComponent` 简化，把过滤操作移动到 render 方法中。

``` JavaScript
// 纯组件只在有 state 或者 prop 改变的时候重新渲染
// 对 state 和 prop 做浅比较来决定是否改变
class Example extends PureComponent {
  // state 只需要管理当前的 filter text 值
  state = {
    filterText: '',
  }

  handleChange = event => {
    this.setState({ filterText: event.target.value })
  }
  
  render() {
    const filteredList = this.props.list.filter(item => item.text.includes(this.state.filterText))

    return (
      <Fragment>
        <input onChange={this.handleChange} value={this.state.filterText} />
        <ul>{filteredList.map(item => <li key={item.id}>{item.text}</li>)}</ul>
      </Fragment>
    )
  }
}
```

上面的方法比 derived state 的版本更加干净简单。在个别情况下，这不够好——对很大的列表来说，过滤可能会慢，而在其他 prop 改变的时候，`PureComponent` 不会阻止渲染。为了解决这两个问题，我们可以添加 memoization 避免对列表不必要的重新过滤：

``` JavaScript
import memoize from 'memoize-one'

class Example extends Component {
  state = { filterText: '' }

  filter = memoize(
    (list, filterText) => list.filter(item => item.text.includes(filterText))
  )

  handleChange = event => {
    this.setState({ filterText: event.target.value })
  }

  render() {
    const filteredList = this.filter(this.props.list, this.state.filterText)

    return (
      <Fragment>
        <input onChange={this.handleChange} value={this.state.filterText}>
        <ul>{filteredList.map(item => <li key={item.id}>{item.text}</li>)}</ul>
      </Fragment>
    )
  }
}
```

这样简单得多，且表现得和 derived state 版本一模一样。

当使用 memoization 时，记住一些限制：

1. 在大多数情况下，你需要**把用来做缓存的方法加到组件实例上**。这防止一个组件的多个实例互相重置对方缓存的值。
2. 通常，为了防止随着时间推移内存泄露，你会想要用具有**缓存大小限制**的助手。（在上面的例子中，我们使用 `memoize-one`，因为它只缓存最近的参数和结果。）
3. 如果 `props.list` 在每次父组件渲染的时候都会重新创建，那么本节中显示的任何方法都不会起作用。但在大多数情况下，这种设置是合适的。

## 结语

在实际应用中，组件通常同时包含受控行为和不受控行为。这是没关系的！如果每一个值都有一个清晰的真实来源，你就可以避免上面的反模式。

值得重新考虑的是，`getDerivedStateFromProps`（以及一般的派生状态）是一种高级功能，因为其复杂性，应该少而谨慎地使用。如果你的用例不在这些模式中，请在 [Github](https://github.com/reactjs/reactjs.org/issues/new) 或 [Twitter](https://twitter.com/reactjs) 中分享给我们！

## 译者附
### 原文
来自 React 官方博客，[你可能不需要派生状态](https://reactjs.org/blog/2018/06/07/you-probably-dont-need-derived-state.html)。

### 以上文章综合总结
这样说可能不太好，但我觉得这篇文章的逻辑还是有一点混乱的……

最前面，需求仿佛还是 在 prop 更新的时候，state 也需要做出反应。稍往后，作者就根据自己的业务需求，把代码上的需求更改为了“prop 只需要作为 state 的初始化数值，不需要随 prop 更新做出反应”。所以你会发现，它推荐的方法并不能真正替代反模式实现的效果，因为在 `key` 相同的情况下，推荐做法无法做到在 props 每次更新时都做出响应。

你可以说，作者的需求就是这样的呀。但是，提出 `key` 方法是根据作者自己假设的另一个需求，即存在切换用户的需求。既然作者可以假设能够切换用户，为什么就不能假设，比如，外部需要一个按钮重置所有组件的 email 值（但不影响其他值）呢？这样 `key` 方法不就行不通了吗？

哪怕切换用户是一开始就提出的，我也认为不妥。大家看 React 官方博客的文章，寻求的必然是通用做法，而不是完全根据一个狭窄的实际用例去区分“好方法”和“坏方法”。

因此，我觉得第二种反模式，实际上不能说是反模式。可以注意一下，第二种“反模式”，即在 props 更新时比较新旧 props，实际上与 `key` 推荐方法的第一种替代是同样的做法。也就是说，在 props 更新时比较新旧 props 并不是“反模式”，在可以切换用户的情况下，比较 email 值，而不是比较 user ID 的值，这才是反模式。

在我看来，后面追加的 memoization 章节，这才是真正可以用于实现“某项 prop/state 更改（而不是如切换用户等情况下的整体更改）后才做某些操作”的需求。

memoization 的核心其实就是将最终结果缓存起来（对应 memoization 的反模式：分别缓存单独的 state/prop），在组件需要更新的情况下（state/prop 改变或是父组件渲染），组件本身还是会更新，但其应该渲染的值是被缓存起来的，如果传入的参数一样，就直接返回值，不需要重新做重复的计算/请求等操作。

### React 16.4+ 新的生命周期
![React 16.4+ 新的生命周期](/images/React-16-4-lifecycle.png "React 16.4+ 新的生命周期")

### 关于如何完全取代 `componentWillReceiveProps`
根据 React 官方文档，有以下几点：

1. 如果你需要在 props 改变后表现某种副作用（比如，请求发送或动画），使用 [`componentDidUpdate`](https://reactjs.org/docs/react-component.html#componentdidupdate) 生命周期代替。
2. 如果你想要仅在 prop 改变的时候重新计算某些数据，使用 memoization 代替。
3. 如果你想要在 prop 改变的时候重置某些 state，考虑使组件完全受控或完全不受控（并使用 `key`）代替。
4. 在十分少见的情况下，你可能想使用 [`getDerivedStateFromProps`](https://reactjs.org/docs/react-component.html#static-getderivedstatefromprops) 作为最后的选择。

综合本篇文章，我认为第三条是不完全正确的。完全受控 + key 根本做不到在 props 某项改变的时候更新，只能做到在 props 整体改变，新数据与之前没有关联的时候（比如对用户的管理系统，切换了用户），整体更新。

最终总结，我认为，官方提供的在 props 改变时更新的方法有以下几点：

1. 使用 `componentDidUpdate` 代替；
2. 干脆绕过去，把需要在 props 改变时更新的操作移到父组件，这样不需要在子组件比较更新，直接把最终结果下发到子组件即可；
3. 如果 props 是整体全部改变，数据与之前没有关联（比如用户管理系统，切换了用户，id 都不同了），利用 id 作为 `key`，将整个组件重置；
4. memoization；
5. 万不得已再使用的 `getDerivedStateFromProps`。
