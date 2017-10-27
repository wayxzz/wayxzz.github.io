
本文重点讲述Vue2渲染的整体流程，包括数据响应的实现（双向绑定）、模板编译、virtual dom原理等，希望读者看完有所收获。

## 前言

现代主流框架均使用一种`数据=>视图`的方式，隐藏了繁琐的dom操作，采用了声明式编程（`Declarative Programming`）替代了过去的类jquery的命令式编程（`Imperative Programming`)

```js
$("#xxx").text("xxx");
// 变为下者
view = render(state);
```

前者我们详细地写了如何去操作dom节点的过程，我们命令什么，它就操作什么；
后者则是我们输入了数据状态，输出视图（我们不关心中间的过程，它们均由框架帮助我们实现）；
前者固然直接，但是当应用变得复杂则代码将难以维护，而后者框架帮我们实现了一系列的操作，无需管理过程，优势显然可见。

为了实现这一点，就是实现如何输入数据，输出视图，我们就会注意到上面的render函数，render函数的实现，主要在对dom性能的优化上，当然实现方式也多种多样，直接的innerHTML、使用documentFragment、还有virtual dom，在不同场景下性能上有所不同，但是框架追求的是在大部分场景中框架已经满足你的优化需求，这里我们也不加以赘述，后文会提到。

当然还有数据变化侦测，从而`re-render`视图，数据变化侦测中，值得一提的是数据生产者（`Producer`）和数据消费者（`Consumer`）之间的联系，这里，我们可以暂且将系统（视图）作为一个数据的消费者，我们的代码设置数据的变化，作为数据的生产者
我们这里可以分为`系统不可感知数据变化`和`系统可感知数据变化`

> Rx.js中是将两者通信分成拉取（Pull)和推送（Push)，比较不好理解，这里我自己就分了个类

- 系统不可感知数据变化

像React／Angular这类框架并不知道数据什么时候变了，但是它视图什么时候更新呢，比如React就是通过setState发信号告诉系统有可能数据变了,然后通过virtual dom diff去渲染视图，angular则是有一个脏值检查流程，遍历比对

- 系统可感知数据变化

Rx.js/vue这一类响应式的，通过观察者模式，使用Observable (可观察对象)，Observer (观察者)(或者是watcher)去订阅（比如视图渲染这一类，其实也可以当成一个观察者去订阅数据了，后面会提到），系统是可以很准确知道哪里数据变了的，从而也就能实现视图更新渲染。

上者系统不可感知数据变化,粒度粗，有时候还得手动优化（比如pureComponet和shouldComponentUpdate)去跳过一些数据不会更新的视图从而提升性能
下者系统可感知数据变化,粒度细，但是绑定大量观察者，有大量的依赖追踪的内存开销

这里也就终于提到本文的主角Vue2，它采用了折中粒度的方式，粒度到组件级别上，由watcher订阅数据，当数据变化我们可以得知哪个组件数据变了，然后采用virtual dom diff的方式去更新相应组件。

后文我们也将展开它是如何实现这些过程的，我们可以先从一个简单的应用开始。

##从一个简单的应用看起

```js
<div id="app">
  {{ message }}
</div>

var app = new Vue({
  el: '#app',
  data: {
    message: 'Hello Vue!'
  }
})
app.message = `xxx`; // 发现视图发生了变化
```

从这里我们也可以提出几个问题，让后面原理的解析更有针对性。

- 数据响应？如何得知数据变化？(还有一个小细节，app.message如何拿到vue data中的message?)
- 数据变动如何和视图联系在一起？
- virtual dom是什么？virtual dom diff又是什么？

当然同时我们也会讲解一些收集依赖等相关的概念。

## 数据响应原理

### Object.defineProperty

Vue数据响应核心是使用了`Object.defineProperty`方法（IE9+)在对象中定义属性或者修改属性，其中存取描述符很关键的就是get和set,提供给属性getter和setter方法

可以看下面例子,我们拦截到了数据获取以及设置

```js
var obj = {};
Object.defineProperty(obj, 'msg', {
  get () {
    console.log('get')
  },
  set (newValue) {
    console.log('set', newValue)
  }
});
obj.msg // get
obj.msg = 'hello world' // set hello world
```

顺便提到那个小细节的问题: app.message如何拿到vue data中的message

其实也是跟Object.defineProperty有关,Vue在初始化数据的时候会遍历data代理这些数据

```js
function initData (vm) {
    let data = vm.$options.data
    vm._data = data
    const keys = Object.keys(data)
    let i = keys.length
    while (i--) {
        const key = keys[i]
        proxy(vm, `_data`, key)
    }
    observe(data)
}
```

proxy做了哪些操作呢？

```js
function proxy (target, sourceKey, key) {
    Object.defineProperty(target, key, {
        enumerable: true,
        configurable: true,
        get () {
            return this[sourceKey][key]
        }
        set () {
            this[sourceKey][key] = val
        }
    })
}
```

其实就是用`Object.defineProperty`多加了一层的访问,因此我们就可以用`app.message`访问到`app.data.message`,也算个`Object.defineProperty`小应用吧

讲完这语法的核心层面得知了如何知道数据发生变化，但是响应，是还有回应的，接下来来谈下Vue是如何实现数据响应的？
其实就是解决下面的问题，如何实现$watch?

```js
const vm = new Vue({
  data:{
    msg: 1,
  }
})
vm.$watch("msg", () => console.log("msg变了"));
vm.msg = 2; //输出「msg变了」
```

### 观察者模式（Observer, Watcher, Dep)

Vue实现响应式有三个很重要的类，Observer类，Watcher类，Dep类, 我这里先笼统介绍一下（详细可见源码英文注解）

- Observer类主要用于给Vue的数据defineProperty增加getter/setter方法，并且在getter/setter中收集依赖或者通知更新
- Watcher类来用于观察数据（或者表达式）变化然后执行回调函数（其中也有收集依赖的过程），主要用于$watch API和指令上
- Dep类就是一个可观察对象，可以有不同指令订阅它（它是多播的）

观察者模式,跟发布/订阅模式有点像,但是其实略有不同，发布/订阅模式是由统一的事件分发调度中心，on则往中心中数组加事件（订阅），emit则从中心中数组取出事件（发布），发布和订阅以及发布后调度订阅者的操作都是由中心统一完成

但是观察者模式则没有这样的中心，观察者订阅了可观察对象，当可观察对象发布事件，则就直接调度观察者的行为，所以这里观察者和可观察对象其实就产生了一个依赖的关系，这个是发布／订阅模式上没有体现的。

> 其实Dep就是dependence依赖的缩写

如何实现观察者模式呢？

我们先看下面代码，下面代码实现了Watcher去订阅Dep的过程，Dep由于是可以被多个Watcher所订阅的，所以它拥有着订阅者数组，订阅了它，就把Watcher放入数组即可。

```js
class Dep {
    constructor () {
        this.subs = []
    }
    notify () {
        const subs = this.subs.slice()
        for (let i = 0; i < subs.length; i++) {
            subs[i].update()
        }
    }
    addSub (sub) {
        this.subs.push(sub)
  }
    }

class Watcher {
    constructor () {
    }
    update () {
    }
}

let dep = new Dep()
dep.addSub(new Watcher()) // Watcher订阅了依赖
```

我们实现了订阅，那通知发布呢，也就是上面的notify在哪里实现呢？

我们到这里就可以联系到数据响应，我们需要的是数据变化去通知更新，那显然是会在defineProperty中的setter中去实现了，聪明的你应该想到了，我们可以把每一个数据当成一个Dep实例，然后setter的时候去notify就行了，所以我们可以在defineProperty中new Dep()，通过闭包setter就可以取到Dep实例了

就像下面这样

```js
function defineReactive (obj, key, val) {
    const dep = new Dep()
    Object.defineProperty(obj, key, {
        enumerable: true,
        configurable: true,
        get: function reactiveGetter () {
            //...
        },
        set: function reactiveSetter (newVal) {
            //...
            dep.notify()
        }
    })
}
```

然后这里就又产生了一个问题, 你都把Dep实例放里面了，我怎么让我的Watcher实例订阅到这个Dep实例呢，Vue在这里实现了精妙的一笔，从get里面做手脚，在get中是可以取到这个Dep实例的，所以可以在执行watch操作的时候，执行获取数值，触发getter去收集依赖

```js
function defineReactive (obj, key, val) {
    const dep = new Dep()
    const property = Object.getOwnPropertyDescriptor(obj, key)

    const getter = property && property.get
    const setter = property && property.set

    let childOb = observe(val)
    Object.defineProperty(obj, key, {
        enumerable: true,
        configurable: true,
        get: function reactiveGetter () {
            const value = getter ? getter.call(obj) : val
            if (Dep.target) {
                dep.depend() // 等价执行dep.addSub(Dep.target)，在这里收集
            }
            return value
        },
        set: function reactiveSetter (newVal) {
            const value = getter ? getter.call(obj) : val
            if (newVal === value) {
                return
            }
            if (setter) {
                setter.call(obj, newVal)
            } else {
                val = newVal
            }
            dep.notify()
        }
    })
}
```

这里我们也要结合Watcher的实现来看

```js
class Watcher () {
    constructor (vm, expOrFn, cb, options) {
        this.cb = cb
        this.value = this.get()
    }
    get () {
        pushTarget(this) // 标记全局变量Dep.target
        let value = this.getter.call(vm, vm) // 触发getter
        if (this.deep) {
            traverse(value)
        }
        popTarget() // 标记全局变量Dep.target
        return value
    }
    update () {
        this.run()
    }
    run () {
        const value = this.get() // new Value
        // re-collect dep
        if (value !== this.value ||
            isObject(value)) {
            const oldValue = this.value
            this.value = value
            this.cb.call(this.vm, value, oldValue)
        }
    }
}
```

所以我们在new Watcher的时候会执行一个求值的操作，然后因为标记了这个Watcher触发的，所以收集了依赖，也就是观察者订阅了依赖（这个求值有可能不止触发了一个getter,有可能触发了很多个getter,那就收集了多个依赖），我们可以再注意一下上面的run操作，也就是dep.notify()后watcher会执行的操作，还会出现一个get操作，我们可以注意到这里重新收集了一波依赖！（当然里面有相关的去重操作）

我们再回来回顾上面我们要解决的小例子

```js
const vm = new Vue({
  data: {
    msg: 1,
  }
})
vm.$watch("msg", () => console.log("msg变了"));
vm.msg = 2; //输出「变了」
```

$watcher其实就是一个new Watcher的封装
即new Watcher(vm, 'msg', () => console.log("msg变了"))

- 首先是new Vue遍历了数据，给数据defineProperty加上了getter/setter方法
- 我们`new Watcher(vm, 'msg', () => console.log("msg变了"))`,首先标记了全局变量`Dep.target = 该Watcher实例`，然后执行msg的get操作，触发到了它的getter，然后dep成功获取到它的订阅者，放入它的订阅者数组，最后我们将`Dep.target = null`
- 最后设置vm.msg = 2,触发到了setter,闭包中的dep.notify,遍历订阅者数组，执行相应的回调操作。

其实讲到这里，核心的响应式原理就讲得差不多了。但是其实Object.defineProperty并不是万能的:

- 数组的push／pop等操作
- 不能监测数组length长度的变化
- 数组的arr[xxx] = yyy无法感知
- 同样的，对象属性的添加和删除无法感知

为了解决这些本身js限制的问题

- Vue首先是对数组方法进行变异，用__proto__继承那些方法（如果不行则直接一个个defineProperty到数组上），具体的变异方法就是在后面加上dep.notify的操作
- 至于属性的添加和删除，我们可以想象到，增加属性，那我们根本没有defineProperty，删除属性则连我们之前的defineProperty都给删了，所以这里Vue增加了一个$set/$delete的API去实现这些操作，同样也是在最后加上了dep.notify的操作
- 当然以上就不是单纯靠defineProperty中每一个数据所对应的dep来实现了，在Observer类也有一个dep实例，同时会给数据挂载一个__ob__属性去获取它的Observer实例，像数组和对象的上面特殊操作，在watch收集依赖的时候都会把这个依赖收集到，然后最后使用的是这个dep去notify更新

现在我们再来看看Vue官网的这张图

![]()

至少目前我们对右半部分很清晰了，Data如何和Watcher联系已经很清楚，但是Render Function，Watcher怎么Trigger Render Function这个还需要去解答，当然还有左下角的Virtual DOM Tree

### 数据与视图如何联系

这里摘出一段关键的Vue代码

```js
class Watcher () {
    constructor (vm, expOrFn, cb, options) {
    }
}
updateComponent = () => {
    // hydrating有关ssr本文不涉及
    vm._update(vm._render(), hydrating)
}
vm._watcher = new Watcher(vm, updateComponent, noop)
// noop是回调函数，它是空函数
```

这个其实就是Watcher和Render的核心关系

还记得我们上面所说的，在执行`new Watche`r会有一个求值的操作，这里的求值是一个函数表达式,也就是执行`updateComponent`，执行`updateComponent`后，会再执行`vm._render()`,传参数给`vm._update(vm._render(), hydrating)`,收集完依赖以后才结束，这里有两个关键的点，`vm._render`在做什么？`vm._update`在做什么？

#### vm._render

我们看下`Vue.prototype._rende`r是何方神圣（以下为删减代码）

```js
Vue.prototype._render = function (): VNode {
    const vm: Component = this
    const {
        render,
        staticRenderFns,
        _parentVnode
    } = vm.$options
    // ...
    let vnode
    try {
        // vm._renderProxy我们直接当成vm，其实就是为了开发环境报warning用的
        vnode = render.call(vm._renderProxy, vm.$createElement)
    } catch (e) {

    }

    // set parent
    vnode.parent = _parentVnode
    return vnode
}
```

所以它这里我们可以看到里面是执行了render函数，render函数来自options，然后返回了vnode

所以到这里我们可以把我们的目光移到这个render函数从哪里来的

如果熟悉Vue2的朋友可能知道，Vue提供了一个选项是render就是作为这个函数的，假如没有提供这个选项呢我们不妨看看生命周期

![]()

我们可以看到`Compile template into render function`（没有template会将el的outerHTML当成template),所以这里就有一个模板编译的过程

### 模板编译

再摘一段核心代码

```js
const ast = parse(template.trim(), options) // 构建抽象语法树
optimize(ast, options) // 优化
const code = generate(ast, options) // 生成代码
return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
}
```

我们可以看到上面分成三部分

- 将模板转化为抽象语法树
- 优化抽象语法树
- 根据抽象语法树生成代码

那里面具体做了什么呢？

- 第一部分其实就是各种正则了，对左右开闭标签的匹配以及属性的收集，通过栈的形式，不断出栈入栈去匹配以及更换父节点，最后生成一个对象，包含children,children又包含children的对象
- 第二部分则是以第一部分为基础，根据节点类型找出一些静态的节点并标记
- 第三部分就是生成render函数代码了

所以最后会产生这样的效果

模板

```html
<div id="container">
  <p>Message is: {{ message }}</p>
</div>
```

生成render函数

```js
(function() {
    with (this) {
        return _c('div', {
            attrs: {
                "id": "container"
            }
        }, [_c('p', [_v("Message is: " + _s(message))])])
    }
})
```

这里我们又可以结合上面的代码了

```js
vnode = render.call(vm._renderProxy, vm.$createElement)
```

其中_c就是vm.$createElement

vm.$createElement其实就是一个创建vnode的一个API

知道了vm._render()创建了vnode返回，接下来就是vm._update了

#### vm._update

vm._update部分也是跟virtual dom有关，下一节具体介绍，我们可以先透露下函数的功能，顾名思义，就是更新视图，根据传入的vnode更新到视图中。

### 数据到视图的整体流程

所以到这里我们就可以得出一个数据到视图的整体流程的结论了

- 在组件级别，vue会执行一个new Watcher
- new Watcher首先会有一个求值的操作，它的求值就是执行一个函数，这个函数会执行render，其中可能会有编译模板成render函数的操作，然后生成vnode(virtual dom)，再将virtual dom应用到视图中
- 其中将virtual dom应用到视图中（这里涉及到diff后文会讲），一定会对其中的表达式求值(比如{{message}},我们肯定会取到它的值再去渲染的），这里会触发到相应的getter操作完成依赖的收集
- 当数据变化的时候，就会notify到这个组件级别的Watcher,然后它还会去求值，从而重新收集依赖，并且重新渲染视图

我们再一次来看看Vue官网的这张图

![]()

## Virtual DOM

我们上一节隐藏了很多Virtual DOM的细节，是因为Virtual DOM大篇幅有可能让我们忘记我们所要探究的问题，这里我们来揭开Virtual DOM的谜团，它其实并没有那么神秘。

### 为什么会有Virtual DOM?

做过前端性能优化的朋友应该都知道，DOM操作都是很慢的，我们要减少对它的操作
为啥慢呢？
我们可以尝试打出一层DOM的key

![]()

我们可以看出它的属性是庞大，更何况这只是一层
同时直接对DOM的操作，就必须很注意一些有可能触发重排的操作。

那Virtual DOM是什么角色呢？它其实就是我们代码到操作DOM的一层缓冲，既然操作DOM慢，那我操作js对象快吧，我就操作js对象，然后最后把这个对象再一起转换成真正的DOM就行了

所以就变成 代码 => Virtual DOM( 一个特殊的js对象） => DOM

### 什么是Virtual DOM

上文其实我们就解答了什么是虚拟DOM,它就是一个特殊的js对象
我们可以看看Vue中的Vnode是怎么定义的？

```js
export class VNode {
    constructor (
        tag?: string,
        data?: VNodeData,
        children?: ?Array<VNode>,
        text?: string,
        elm?: Node,
        context?: Component,
        componentOptions?: VNodeComponentOptions,
        asyncFactory?: Function
    ) {
        this.tag = tag
        this.data = data
        this.children = children
        this.text = text
        this.elm = elm
        this.ns = undefined
        this.context = context
        this.functionalContext = undefined
        this.key = data && data.key
        this.componentOptions = componentOptions
        this.componentInstance = undefined
        this.parent = undefined
        this.raw = false
        this.isStatic = false
        this.isRootInsert = true
        this.isComment = false
        this.isCloned = false
        this.isOnce = false
        this.asyncFactory = asyncFactory
        this.asyncMeta = undefined
        this.isAsyncPlaceholder = false
    }
}
```

用以上这些属性就能来表示一个DOM节点

### Virtual DOM算法

这里我们讲的就是涉及上面`vm.update`的操作

- 首先是js对象（Virtual DOM）描述树（vm._render)，转换dom插入(第一次渲染）
- 状态变化，生成新的js对象（Virtual DOM），比对新旧对象
- 将变更应用到DOM上，并保存新的js对象（Virtual DOM），重复第二步操作

用js对象描述树(生成Virtual DOM），Vue中就是先转成AST生成code,然后通过$creatElement通过Vnode的那种形式生成Virtual DOM (vm._render的操作)

这里我们可以具体看下vm._update（其实就是Virtual DOM算法的后两步）

```js
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    if (vm._isMounted) {
        callHook(vm, 'beforeUpdate')
    }
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    // ...
    if (!prevVnode) {
        // initial render
        // 第一次渲染
        vm.$el = vm.__patch__(
            vm.$el, vnode, hydrating, false /* removeOnly */,
            vm.$options._parentElm,
            vm.$options._refElm
        )
    } else {
        // updates
        // 更新视图
        vm.$el = vm.__patch__(prevVnode, vnode)
    }
    // ...
}
```

可以看到一个关键点vm.__patch__，其实它就是Virtual DOM Diff的核心，也是它最后把真实DOM插入的

### Virtual DOM Diff

引用一张React经典的图来帮助大家理解吧，左右同一颜色圈起来的就是比较／复用的范围

![]()

步入正题，我们看看Vue的patch函数

```js
function patch (oldVnode, vnode, hydrating, removeOnly, parentElm, refElm) {
    if (isUndef(vnode)) {
        if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
        return
    }

    let isInitialPatch = false
    const insertedVnodeQueue = []

    if (isUndef(oldVnode)) {
        // empty mount (likely as component), create new root element
        // 老节点不存在，直接创建元素
        isInitialPatch = true
        createElm(vnode, insertedVnodeQueue, parentElm, refElm)
    } else {
         onst isRealElement = isDef(oldVnode.nodeType)
        if (!isRealElement && sameVnode(oldVnode, vnode)) {
            // patch existing root node
            // 新节点和老节点相同，则给老节点打补丁
            patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly)
        } else {
            // ... 省略ssr代码
            // replacing existing element
            // 新节点和老节点相同，直接替换老节点
            const oldElm = oldVnode.elm
            const parentElm = nodeOps.parentNode(oldElm)
            createElm(
                vnode,
                insertedVnodeQueue,
                // extremely rare edge case: do not insert if old element is in a
                // leaving transition. Only happens when combining transition +
                // keep-alive + HOCs. (#4590)
                oldElm._leaveCb ? null : parentElm,
                nodeOps.nextSibling(oldElm)
            )
        }
    }
    // ...省略代码
    return vnode.elm
}
```

所以patch大概做下面几件事

- 判断老节点存不存在
- 不存在则为首次渲染，直接创建元素
- 存在的话则sameVnode使用判断根节点是否相同
- 相同则使用patchVnode给老节点打补丁
- 不相同则使用新节点直接替换老节点

对于sameVnode判断，其实就是简单比较了几个属性判断

```js
function sameVnode (a, b) {
    return (
        a.key === b.key && (
            (
                a.tag === b.tag &&
                a.isComment === b.isComment &&
                isDef(a.data) === isDef(b.data) &&
                sameInputType(a, b)
            ) || (
                isTrue(a.isAsyncPlaceholder) &&
                a.asyncFactory === b.asyncFactory &&
                isUndef(b.asyncFactory.error)
            )
        )
    )
}
```

对于patchVnode,其实就是比较节点的子节点，分别对新老节点的拥有的子节点做判断，假如两者都没有或者一者有一者没有，就比较容易，直接删除或者增加即可，但是假如两者都有子节点，这里就涉及到列表对比以及一些复用操作了，实现的方法是updateChildren

```js
function patchVnode (oldVnode, vnode, insertedVnodeQueue, removeOnly) {
    if (oldVnode === vnode) {
      // 新老节点相同
      return
    }
    // ... 省略代码
    if (isUndef(vnode.text)) {
      // 假如新节点没有text
      if (isDef(oldCh) && isDef(ch)) {
        // 假如老节点和新节点都有子节点
        // 不相等则更新子节点
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
      } else if (isDef(ch)) {
        // 新节点有子节点，老节点没有
                // 老节点加上
        if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
      } else if (isDef(oldCh)) {
        // 老节点有子节点，新节点没有
                // 老节点移除
        removeVnodes(elm, oldCh, 0, oldCh.length - 1)
      } else if (isDef(oldVnode.text)) {
        // 老节点有文本，新节点没有文本
        nodeOps.setTextContent(elm, '')
      }
    } else if (oldVnode.text !== vnode.text) {
      // 假如新节点和老节点text不相等
      nodeOps.setTextContent(elm, vnode.text)
    }
    if (isDef(data)) {
      if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
    }
}
```

我们最后再来看看这个updateChildren,这部分其实就是leetcode.com/problems/ed… 最小编辑距离问题，这里也并没有用复杂的动态规划算法(复杂度为O(m * n)）去实现最小的移动操作,而是选择可牺牲一定的dom操作去优化部分场景，复杂度可以降低到O(max(m, n)，比较分别首尾节点，如果没有匹配到，则使用第一个节点key（这里就是我们常在v-for用的）去找相同的key去patch比较，假如没有key的话，则是直接遍历找相似的节点，有则patch移动，没有则创建新节点

这里告诉我们,列表假如有可能有复用的节点，可以使用唯一的key去标识，提升patch效率，但是也不能乱设置key,假如根本不一样，但是你设置一样的话，会导致框架没找到真正相似的节点去复用，反而降低效率，会增加一个创建dom的消耗
这里代码较多，有兴趣的读者可以深入阅读，这里我就不画图了，读者也可以找网上的相应updateChildren的图，有助于理解patch的过程

```js
function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
    let oldStartIdx = 0
    let newStartIdx = 0
    let oldEndIdx = oldCh.length - 1
    let oldStartVnode = oldCh[0]
    let oldEndVnode = oldCh[oldEndIdx]
    let newEndIdx = newCh.length - 1
    let newStartVnode = newCh[0]
    let newEndVnode = newCh[newEndIdx]
    let oldKeyToIdx, idxInOld, vnodeToMove, refElm

    // removeOnly is a special flag used only by <transition-group>
    // to ensure removed elements stay in correct relative positions
    // during leaving transitions
    const canMove = !removeOnly

    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      if (isUndef(oldStartVnode)) {
        // 假如老节点的第一个子节点不存在
        // 老节点头指针就往下一个移动
        oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
      } else if (isUndef(oldEndVnode)) {
        // 假如老节点的最后一个子节点不存在
        // 老节点尾指针就往上一个移动
        oldEndVnode = oldCh[--oldEndIdx]
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        // 假如新节点的第一个和老节点的第一个相同
        // patch该节点并且新老节点头指针分别往下一个移动
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue)
        oldStartVnode = oldCh[++oldStartIdx]
        newStartVnode = newCh[++newStartIdx]
      } else if (sameVnode(oldEndVnode, newEndVnode)) {
        // 假如新节点的最后一个和老节点的最后一个相同
        // patch该节点并且新老节点尾指针分别往上一个移动
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue)
        oldEndVnode = oldCh[--oldEndIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        // 假如新节点的最后一个和老节点的第一个相同
        // patch该节点并且新节点尾指针往上一个移动，老节点头指针往下一个移动
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue)
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
        oldStartVnode = oldCh[++oldStartIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
        // 假如新节点的第一个和老节点的最后一个相同
        // patch该节点并且老节点尾指针往上一个移动，新节点头指针往下一个移动
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue)
        canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
        oldEndVnode = oldCh[--oldEndIdx]
        newStartVnode = newCh[++newStartIdx]
      } else {
        // 创建老节点key to index的映射
        if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
        idxInOld = isDef(newStartVnode.key)
          ? oldKeyToIdx[newStartVnode.key] // 假如新节点第一个有key,找该key下老节点的index
          : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx) // 假如新节点没有key,直接遍历找相同的index
        if (isUndef(idxInOld)) { // New element
          // 假如没有找到index,则创建节点
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm)
        } else {
          // 假如有index,则找出这个需要move的老节点
          vnodeToMove = oldCh[idxInOld]
          /* istanbul ignore if */
          if (process.env.NODE_ENV !== 'production' && !vnodeToMove) {
            warn(
              'It seems there are duplicate keys that is causing an update error. ' +
              'Make sure each v-for item has a unique key.'
            )
          }
          if (sameVnode(vnodeToMove, newStartVnode)) {
            // move老节点和新节点的第一个基本相同则开始patch
            patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue)
            // 设置老节点空
            oldCh[idxInOld] = undefined
            canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
          } else {
            // 不同则还是创建新节点
            // same key but different element. treat as new element
            createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm)
          }
        }
        newStartVnode = newCh[++newStartIdx]
      }
    }
    if (oldStartIdx > oldEndIdx) {
      // 假如老节点的头指针超过了尾部的指针
      // 说明缺少了节点
      refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
      addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
    } else if (newStartIdx > newEndIdx) {
      // 假如新节点的头指针超过了尾部的指针
      // 说明多了节点
      removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx)
    }
  }
```
