---
title: 关于我写了这么多年 React，居然还能忽然懵掉这件事
date: 2021-09-07 16:42:42
categories: [框架/库/工具, React.js]
tags: [JavaScript, React.js]
---

## 起因

需求是实现一个伪对话框，也就是实现一个“两人互发消息”的效果。
第一反应，当然是把需要展示出来的消息存入数组，作为 React 组件的 state。这样，在需要发出消息的时候，就把消息推入这个数组，造成重新渲染即可。
所以，我写出了这样的代码：

``` JavaScript
class Home extends React.Component {
  constructor() {
    super()

    this.state = {
      tag: '', // 中文学段。因为初版本实验性质，暂时使用中文作为选项索引。
      msgList: [],
      selected: {
        gradeSelected: '',
      },
    }
  }

  componentDidMount() {
    // “很高兴见到你”
    this.addMsgItem(0)

    // “你最想提升哪门学科的成绩呢？”
    setTimeout(() => {
      this.addMsgItem(1)
    }, 2000)

    // “初一 | 想提升的学科是：”
    getUserInfo({
      imei: getQuery('imei'),
    }).then(res => {
      setTimeout(() => {
        this.setState(
          {
            tag: res.slTag.tagName,
          },
          () => {
            this.addMsgItem('gradeSelect')
          }
        )
      }, 3000)
    })
  }

  // 给聊天框增加信息
  addMsgItem = key => {
    let { msgList } = this.state
    const { tag, selected } = this.state

    // 静态文本消息
    if (typeof key === 'number') {
      msgList = [...msgList, staticMsgItems[key]]
    }

    switch (key) {
      // 选择成绩
      case 'gradeSelect':
        msgList = [
          ...msgList,
          {
            children: (
              <div className={s.selectSubjectContainer}>
                <header className={s.subHeader}>{tag} | 想提升的学科是：</header>

                <ul className={s.subjectList}>
                  {subjectList.map(item => (
                    <li className={s.subjectItem} key={item}>
                      <button
                        type="button"
                        className={cx({
                          [s.subjectItemBtn]: true,
                          [s.active]: item === selected.gradeSelected,
                        })}
                        onClick={() => {
                          this.setState({
                            selected: {
                              ...selected,
                              gradeSelected: item,
                            },
                          })
                        }}
                      >
                        {item}
                      </button>
                    </li>
                  ))}
                </ul>
              </div>
            ),
          },
        ]
        break

      default:
        break
    }

    this.setState({
      msgList,
    })
  }

  render() {
    const { msgList } = this.state

    return (
      <ul className={s.container}>
        {msgList.map(item => (
          <MessageItem className={s.msgItem}>{item.children}</MessageItem>
        ))}
      </ul>
    )
  }
}

export default Home
```

`addMsgItem` 即为向数组中插入消息的方法。

可以看到，插入的第三条消息为一个可交互的组件，有着自己的样式和交互逻辑。我就毫不犹豫地将第三条消息的 jsx 直接插入了数组，渲染了出来。

到这一步都没有发现问题。直到后面，在对这个组件进行交互的时候，我发现这个这个组件上的按钮“点不了”。

## 调查

按理说，点击按钮应该使按钮变为“选择状态”，也就是变红。但现在无论怎样点击都不能够变红。
调查了下按钮的回调，发现 `onClick` 回调是好好触发了，记录“哪个按钮被选择”的 state 也已经正常更改了。
怀疑是 state 变动没有触发渲染。但是在 render 打 log，发现 render 函数也是执行了的。
很迷茫，不知道为什么，“哪个按钮被点击”的 state 已经改变了，`item` 等于 `selected.gradeSelected`，但是对应的按钮就是不变红。

## bug 原因

当时脑子也有些乱。后来把整个渲染过程想了一遍，才发现问题所在。
这个问题的原因在于，我直接把 jsx 存入了数组。
如果 jsx 是纯展示性质的，在存入数组之后不会有任何变化，那这个操作没有什么问题。但是，这个 jsx 实际上是需要依赖当前的 state 做出变化的。
这样，一旦我把它存入数组，它在数组之中就不会再次被改变了。组件 state 的变化与它毫无关系（“哪个按钮被选中”的 state 和数组 state 完全独立，互不影响）。
React 组件重渲染的本质是执行 render 方法。在当次渲染执行 render 的时候，render 方法计算 jsx 并返回。在计算过程中，取的是当前最新的 state。
而如果 jsx 不是由 render 方法在当次渲染实时计算出来的，而是一早就存入 state 的，当然不会受到每次渲染变更的 state 的影响，一直维持着最初 push 进数组的那个状态。

## 解决方式

也就是在这个时候，我才意识到了 antd 设计的睿智。
antd 其实也存在着同样的场景，即渲染表格的时候，需要设置 `columns`，用于配置每个单元格渲染的内容。
在渲染内容为 jsx 的时候，`columns` 要求配置一个 `render` 方法，而不直接是需要渲染的那段 jsx。这显然就是为了让这段 jsx 能够在渲染过程中重新生成，而不是静态存在于 state 中。
回到我的例子。在配置消息列表时，我也可以在列表项中提供一个 `render` 方法，而不是一段静态的 jsx。在需要渲染 jsx 时，调用当前列表项的 `render` 方法即可。
