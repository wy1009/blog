---
title: Vue.js keep-alive踩坑
date: 2018-03-08 15:39:56
categories: [框架/库/工具, Vue.js]
tags: [JavaScript, Vue.js, 踩坑]
---

发现面试好像会喜欢问“项目中遇到的难点”这样的问题。但说实话，难点这种东西，难了不会，会了不难，一旦解决了就觉得也还好，过段时间也就忘了，再过段时间，哪怕能想起来，也一定觉得不是什么值得说出来的问题。
总之，确实怎么回想也想不出什么难点，就只好从现在开始记录了。
就从keep-alive踩坑开始吧。
本文主要为了给个人记录一个过程，因此按照踩坑顺序记录，而不是一个知识体系。
btw过程中发现了好多博客的错误……

首先，移动端，一个列表页，一个详情页。从列表页点入详情页，再从详情页回退到列表页，实际上列表页是完全不需要刷新的。在此需求上，能做到类似App那样“前进刷新，后退不刷新”的效果，当然是极好的。

## 判断前进还是后退

单纯监听前进还是后退，很容易想到监听`popstate`事件。但是这个事件不管前进还是后退都能触发，且不能和Vue的生命周期结合起来，所以Pass。

其他人的实现提供了两种方法，一种是[另辟蹊径：vue单页面，多路由，前进刷新，后退不刷新](https://segmentfault.com/a/1190000012083511)，严格监听是从哪一个路由跳到哪一个路由，从而监听是前进还是后退。这显然是很死板的，当然，如果需求简单，仅仅需要顾及几个页面，那反而很安全。另一种是[vue实现前进刷新，后退不刷新](https://juejin.im/post/5a69894a518825733b0f12f2)，使用路由的路径层级去判断是前进还是后退，也就是说，前进后退和路由层级严格挂钩，路由和前进后退的方向都变得不自由了。

我开始想的是第三种方法，在路由信息对象上记录数字，用这个数字标识层级。但是很快意识到，这种方法也不直观，最直观的方法莫过于直接压一个栈。当要去的路由等于栈顶的路由的时候，就相当于是后退了。

```
let routerStack = []
router.beforeEach((to, from, next) => {
  if (to.name === routerStack[routerStack.length - 1]) {
    // 后退
    routerStack.pop()
  } else {
    // 前进
    routerStack.push(to.name)
  }
})
```

## 切换两个router-view控制组件是否keepAlive

关于这个实现，我发现很多人都用了出自[vue-router 之 keep-alive](https://www.jianshu.com/p/0b0222954483)的方法：

```
// App.vue

<keep-alive>
  <router-view v-if="$route.meta.keepAlive"></router-view>
</keep-alive>

<router-view v-if="!$route.meta.keepAlive"></router-view>
```

```
// router.js

router.beforeEach((to, from, next) => {
  if (to.name === routerStack[routerStack.length - 1]) {
    // 后退
    routerStack.pop()
    from.meta.keepAlive = false
    to.meta.keepAlive = true
  } else {
    // 前进
    routerStack.push(from.name)
  }
  next()
})
```

在路由信息对象上记录一个keepAlive属性，用来控制路由是否keepAlive。要后退时，将置当前路由keepAlive为false，而退向的路由keepAlive为true。简单直接，不需要维护复杂队列。

但这种方法显然是有问题的。第一个问题，第一轮，先前进，list->项目1的detail1->项目1的detail2，在操作中，创建每个路由，激活了keepAlive，然后后退，将后退过的路由全部取消keepAlive。然后第二轮，再前进，此时没有keepAlive，所以进入到了`keep-alive`组件外的那个router-view，访问list->项目2的detail1->项目2的detail2，然后后退，请注意！

此时后退的时候，因为实际上只是直接激活了上面`keep-alive`组件中的组件，因此呈现的路由并不是我们第二轮访问过的，而是第一轮访问过的路由。如果我们两次访问的都一模一样，甚至列表页scrollTop的位置都一样，那还无所谓。可我们第一轮访问的是项目1的详情，第二轮访问的是项目2的详情。这就出现了错误。

这是第一个问题，第二个问题，同样的，因为该方法对页面的刷新依赖于后退操作，所以如果从一个页面跳转到已经keep-alive的另一个页面，则页面不会刷新。也就是说，页面跳转路径不能形成环。

因此，我放弃了这种方法，开始使用官方的include方法。

## 控制include属性切换组件是否keepAlive

```
router.beforeEach((to, from, next) => {
  if (to.name === routerStack[routerStack.length - 1]) {
    // 后退
    routerStack.pop()
    // 因为前进时，发现是要访问已经缓存过的路由，则会销毁缓存重新创建
    // 所以后退时（重新创建的也会在后退那一步删掉），缓存列表中不一定还有to路由
    // 如果后退过程中发现to在aliveList中是空的，就加入到aliveList中
    if (aliveList.indexOf(to.name) === -1) {
      aliveList.push(to.name)
    }
    removeFromAliveList(from.name)
  } else {
    // 前进
    routerStack.push(from.name)
    // 前进时，发现是要访问已经缓存过的路由，销毁缓存重新创建
    if (aliveList.indexOf(to.name) !== -1) {
      removeFromAliveList(to.name)
      setTimeout(function () {
        aliveList.push(to.name)
      }, 0)
    }
  }
  next()
})

function removeFromAliveList (name) {
  const i = aliveList.indexOf(name)
  i !== -1 && return aliveList.splice(i, 1) // aliveList即include属性中的列表
  return false
}
```

include有一个很大的问题，就是只能够控制组件是否keepAlive，却不能够销毁已经被移出列表的组件。也就是说，反复对include操作移入移出，被移出的组件会一直留在内存中。而再次移入，当然会创建新的组件实例。

如果你觉得这句话是对的，千万别急着走。我不指望我的博客给别人带来什么收获，但至少不能误人子弟。**因为上句话是错误的。**

如果你知道这句话是错的，别忙着反驳，我后来调试Vue源码的时候也意识到了……所以我说，这篇博客是用来记录我踩坑的路程的。

## 手动destroy组件

因为以为include不能够自行销毁组件实例，所以需要手动销毁。

如果要手动销毁组件，你会意识到，include属性也变得没有意义了。本身，include属性就是在第一次访问组件时才开始keep-alive，而我们也是在第一次访问组件时将组件名推入include属性的列表中，这个操作是重复的。如果include属性不承接销毁实例的操作，需要我们手动销毁，那么这个属性就变得毫无意义了。因此，去除include属性。

如果要手动销毁组件，可以在beforeRouteLeave时检测，如果是后退，则销毁组件。但是，我们在去向一个已缓存的路由时，也需要销毁已缓存的路由组件。

但是，我们仍旧需要查看前进的路由是否已经在aliveList中了，所以，虽然我们不需要include属性了，但是原本赋给include属性的aliveList还是需要的。同时，为了做到随时可以销毁aliveList中的实例，我们需要将实例也记录在aliveList中。

因此，我更改了我的数据结构，将aliveList变更为`{ name: name, vm: vm }`的形式。

```
Vue.mixin({
  beforeRouteEnter (to, from, next) {
    // 逻辑和include时一样，但是手动调用vm.$destroy
    // beforeRouteEnter中的vm如下方式取得
    next((vm) => {
      aliveList.push(vm)
    })
  }
})
```

这样，就迎来了本次踩坑过程让我感到最奇异的一个bug，也致使我去调试源码，顺便把组件从include中移除没有被销毁的原因也找到了。

表现是这样的，假设有五个页面page1~page5，按照以下路径走：4->3->4->3->2->3，就会出现两个page3。可是，按照逻辑来看，page3应该是已经在缓存中的，那么为什么会出现这样的问题呢？

我调试起Vue的源码，越发觉得奇怪。因为从代码运行来看，page3的实例确实是从缓存中取出来的，那么为什么会这样呢？

我打印出了所有的实例，最终发现了原因。以上六个路径，三个page3都是不同的。而page4是相同的，缓存中的page4，两次beforeRouteEnter中的page4实例，都是相同的。同时，page3一直存在于cache表中，虽然它已经被$destroy掉了。

这样，bug的原因就很容易想到了。第二次进入page4时，会销毁page3，但是只是对实例调用了$destroy，而没有将page3从cache表中移除，该实例就一直存在于cache表中。再次访问page3时，仍旧会取这个实例，而这个实例已经被销毁了，就会新建实例。导致在对page3手动销毁一次之后，每次访问page3，都会创建一个新的实例，而不会达到keep-alive的效果。

所以说，这是我之前那个“取消include属性”的决定所带来的bug。我以为destroy掉实例就是移除cache了，但是内部实际上有一个cache表。取消include属性，意味着Vue没有维护cache表。

## 从include列表移除组件，组件没有被销毁的原因

Vue这部分的源码如下：

```
function pruneCacheEntry (
  cache,
  key,
  keys,
  current
) {
  var cached$$1 = cache[key];
  // 判断被移出缓存的组件是否是当前组件，如果不是当前组件才会调用$destroy
  if (cached$$1 && (!current || cached$$1.tag !== current.tag)) {
    cached$$1.componentInstance.$destroy();
  }
  cache[key] = null;
  remove(keys, key);
}
```

根据这段代码，Vue会判断被移出include的组件是否是当前组件，如果是当前组件，才会对组件实例调用`$destroy()`。而我做的所有操作都是在beforeEach或者beforeRouteEnter中的，而在这个钩子中，“当前组件”仍旧是上一个组件，即我们要移出缓存销毁掉的组件，而不是我们要to的下一个组件。这才是导致include没有销毁实例的原因。

最终，我们对这个问题的解决方案也已经很明了了。跟着官方的脚步不动摇，使用官方的方法才是最稳妥的。我将需要移除的组件存入sessionStorage中，然后在activated中去移除。这样，就避开了在beforeEach和beforeRouteEnter中，“当前组件”尚未改变的问题。
