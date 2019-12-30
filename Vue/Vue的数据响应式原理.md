### 什么是数据响应式

>Vue.js 一个核心思想是数据驱动。所谓数据驱动，是指视图是由数据驱动生成的，我们对视图的修改，不会直接操作 DOM，而是通过修改数据。——[《Vue.js技术揭秘》](https://ustbhuangyi.github.io/vue-analysis/v2/data-driven/)

在没有 Vue 和 React 这些具有数据响应式特性的前端框架之前，我们从服务端提供过的接口获取到数据要渲染在 html 页面上时，抑或是需要获取表单的值进行计算回显到页面上时，都需要建立很多 DOM 事件监听器，并进行许多的 DOM 操作。举个最简单的例子：用户在 html 页面的表单中输入两个加数 a 和 b ，计算得到两个加数之和，并展示到页面上。这时我们需要监听表单中 a 和 b 两个输入框的变化，拿到变化值相加然后通过修改指定 id 或者 class 的选择器对应的 DOM 来修改两数之和。但是当输入表单不止两个时，就会令人捉急。


而 Vue 的数据响应式大大地优化了这一整套的流程。面对大量的 UI 交互和数据更新，数据响应式让我们做到从容不迫——我们需要去了解数据的改变是怎样被触发，更新触发后的数据改变又是怎样反馈到视图。接下来我们通过对部分 Vue 源码（本文主要针对v2.6.11的 Vue 源码进行分析）的简单分析和学习来深入了解 Vue 中的数据响应式是怎样实现的。

我们将这一部分的代码分析大致分为三部分：让数据变成响应式、依赖收集和派发更新。

### 数据响应式的中心思想

![](https://user-gold-cdn.xitu.io/2019/12/27/16f452d0374871e6?w=1200&h=750&f=png&s=65183)

该原理图源自 Vue 官方教程的[深入响应式原理](https://cn.vuejs.org/v2/guide/reactivity.html)，这张图向我们展示了 Vue 的数据响应式的中心思想。

我们先来介绍图中的四个模块，**黄色部分**是 Vue 的渲染方法，视图初始化和视图更新时都会调用 `vm._render` 方法进行重新渲染。渲染时不可避免地会 touch 到每个需要展示到视图上的数据（**紫色部分**），触发这些数据的 get 方法从而收集到本次渲染的所有依赖。收集依赖和更新派发都是基于**蓝色部分**的 Watcher 观察者。而当我们在修改这些收集到依赖的数据时，会触发数据中的 set 属性方法，该方法会修改数据的值并 notify 到依赖到它的观察者，从而触发视图的重新渲染。**绿色部分**是渲染过程中生成的 Virtual DOM Tree，这棵树不仅关系到视图渲染，更是 Vue 优化视图更新过程的基础。

简言之，数据响应式的中心思想，是通过重写数据的 get 和 set 属性方法，让数据在被渲染时把所有用到自己的订阅者存放在自己的订阅者列表中，当数据发生变化时将该变化通知到所有订阅了自己的订阅者，达到重新渲染的目的。

### 让数据变成响应式

Vue 的数据响应式原理是由 JS 标准内置对象方法 **Object.defineProperty** 来实现的（这个方法**不兼容 IE8 和 FF22 及以下版本**浏览器，这也是为什么 Vue 只能在这些版本之上的浏览器中才能运行）。这个方法的作用是在一个对象上定义一个新属性，或者修改一个对象的现有属性。那么数据响应式用这个方法为**什么对象**添加或修改了**什么属性**呢？我们从 Vue 的初始化讲起。

![](https://user-gold-cdn.xitu.io/2019/12/27/16f452d6f985941b?w=1553&h=695&f=png&s=277122)

#### Vue 的初始化

```javascript
function Vue (options) {
    // ...
    this._init(options)
}
initMixin(Vue)     
stateMixin(Vue)     
// ...
export default Vue
```
在 [*src/core/index.js*](https://github.com/DQFE/vue/blob/dev/src/core/index.js#L1) 中我们可以看到一个 Vue 方法被导出，这个 Vue 方法是在 *src/core/instance/index* 中定义的，参数是 options。进行 vue 实例化时主要调用了 _init，它对当前传入的 options 进行了一些处理（主要是判断当前实例化的是否为组件，使用 mergeOptions 方法对 options 进行加工，此处不做赘述），然后又调用了一系列方法进行了生命周期、事件、渲染器的初始化，我们主要来关注 **initState** 这个方法（[*src/core/instance/state.js*](https://github.com/DQFE/vue/blob/dev/src/core/instance/state.js#L48)）:
```javascript
export function initState (vm: Component) {
    vm._watchers = []
    const opts = vm.$options
    if (opts.props) initProps(vm, opts.props)
    if (opts.methods) initMethods(vm, opts.methods)
    // a*
    if (opts.data) {
        initData(vm)
    } else {
        observe(vm._data = {}, true /* asRootData */)
    }
    if (opts.computed) initComputed(vm, opts.computed)
    if (opts.watch && opts.watch !== nativeWatch) {
        initWatch(vm, opts.watch)
    }
}
```

#### 将 Vue 实例上的 data 变成响应式数据

上面的方法中对 props、methods、data、computed 和 watch 进行了初始化，这些都是 Vue 实例化方法中传入参数 options 对象的一些属性，这些属性都需要被**响应式化**。而针对于 data 的初始化分了两种情况[a*]，一种是 options 中没有 data 属性的，该方法会给 data 赋值一个空对象并进行 observe（该方法之后我们会详细讲述），如果有 data 属性，则调用 [**initData**](https://github.com/DQFE/vue/blob/dev/src/core/instance/state.js#L112) 方法进行初始化。我们主要通过对 data 属性的初始化来分析Vue中的数据响应式原理。
```javascript
function initData (vm: Component) {
    let data = vm.$options.data
    // a*
    data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
    if (!isPlainObject(data)) {
        data = {}
        process.env.NODE_ENV !== 'production' && warn(
          'data functions should return an object:\n' +
          'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
          vm
        )
    }
    // b*
    const keys = Object.keys(data)
    const props = vm.$options.props
    const methods = vm.$options.methods
    let i = keys.length
    while (i--) {
        // c*
        const key = keys[i]
        if (process.env.NODE_ENV !== 'production') {
          if (methods && hasOwn(methods, key)) {
            warn(
              `Method "${key}" has already been defined as a data property.`,
              vm
            )
          }
        }
        if (props && hasOwn(props, key)) {
          process.env.NODE_ENV !== 'production' && warn(
            `The data property "${key}" is already declared as a prop. ` +
            `Use prop default value instead.`,
            vm
          )
        } else if (!isReserved(key)) {
          proxy(vm, `_data`, key)
        }
    }
    // d*
    observe(data, true)
}
```
initData 方法对 options 中的 data 进行处理，主要是有**两个目的**:

- 一方面**将 data 代理到 Vue 实例上**，同时检查 data 中是否使用了 Vue 中的保留字、是否与 props、methods 属性中的数据同名
- 使用 **observe** 方法将data中的属性变成响应式的

之前我们提到在调用 initState 方法之前会有对 Vue 实例化传入的 options 参数进行处理的环节（mergeOptions），这个过程会将 data 处理成一个函数（例如我们在 vue 组件中的 data 属性），之后会调用 `callHook(vm, 'beforeCreate')` 方法来触发 beforeCreate 的生命周期钩子，这个方法的调用可能会对 data 属性进行进一步处理，所以方法一开始会**对 data 统一做处理使其成为一个function**[a*]。

接下来会对 data 中的每一个数据进行遍历[b*]，遍历过程将会使用  `hasOwn(methods, key)`、`hasOwn(props, key)`、`!isReserved(key)` 方法对该数据是否占用保留字、是否与 props 和 methods 中的属性重名进行判断，然后调用 proxy 方法将其代理到 Vue 实例上。此处需要注意的是：**如果 data 中的属性与 props、methods 中的属性重名，那么在 Vue 实例上调用这个属性时的优先级顺序是 props > methods > data**。

最后对每一个 data 中的属性调用 `observe` 方法，该方法的功能是赋予 data 中的属性可被监测的特性。这个方法和其中使用到的 Observe 类是在[*src/core/observer/index.js*](https://github.com/DQFE/vue/blob/dev/src/core/observer/index.js#L37)文件中，observe 方法主要是调用 **Observer 类构造方法**，将每一个 data 中的 value 变成可观察、响应式的：
```javascript
constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    
    def(value, '__ob__', this)

    if (Array.isArray(value)) {
      //...
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }
```
可以看到该构造函数有以下几个目的：

- 针对当前的数据对象新建一个订阅器；
- 为每个数据的 value 都添加一个__ob__属性，该属性不可枚举并指向自身；
- 针对数组类型的数据进行单独处理（包括赋予其数组的属性和方法，以及 observeArray 进行的数组类型数据的响应式）；
- this.walk(value)，遍历对象的 key 调用 defineReactive 方法；

`defineReactive`是真正为数据添加 get 和 set 属性方法的方法，它将 data 中的数据定义一个响应式对象，并给该对象设置 get 和 set 属性方法，其中 get 方法是对依赖进行收集， set 方法是当数据改变时通知 Watcher 派发更新。

### 依赖收集

依赖收集的原理是：当视图被渲染时，会触发渲染中所使用到的数据的 get 属性方法，通过 get 方法进行依赖收集。

我们先看下 [get](https://github.com/DQFE/vue/blob/dev/src/core/observer/index.js#L160) 属性方法：
```javascript
get: function reactiveGetter () {
  const value = getter ? getter.call(obj) : val
  if (Dep.target) {
    dep.depend()
    if (childOb) {
      childOb.dep.depend()
      if (Array.isArray(value)) {
        dependArray(value)
      }
    }
  }
  return value
}
```
可以看到 get 属性方法中调用了一个 dep.depend 方法，该方法正是依赖收集的开始。我们通过下图来简要概述依赖收集的一个整体过程：

![](https://user-gold-cdn.xitu.io/2019/12/27/16f452dfc2cdd772?w=1582&h=480&f=png&s=321949)

此处涉及到了两个重要的部分——依赖到底是什么？依赖要存放在哪里？这两部分刚好对应 Vue 中两个类，一个是 Watcher 类，而依赖就是 Watcher 类的实例；另一个是 Dep 类，而依赖就是存放在 Dep 实例的 subs 属性（数组类型）中进行管理的。

其真实的逻辑我们通过举例来说明：当前正在渲染组件 ComponentA，会将当前全局唯一的监听器置为当前正在渲染组件 Component A 的 watcher，这个组件中使用到了数据 `data () { return { a: b + 1} }`，此时触发 b 的 getter 会将当前的 watcher 添加到 b 的订阅者列表 subs 中。也就是说如果 ComponentA 依赖 b，则将该组件的渲染 Watcher 作为订阅者加入b的订阅者列表中。**换言之，组件 A 用到了数据 b，则组件 A 依赖数据 b；反馈在依赖收集过程中，就是组件 A 的 watcher 会被添加到数据 b 的依赖数组 subs 中**。接下来我们就针对 Dep 类和 Watcher 类进行简要分析。

其真实的逻辑我们举例来说明：当前正在渲染组件 ComponentA，会将当前全局唯一的监听器置为这个 Watcher，这个组件中使用到了数据 `data () { return { a: b + 1} }`，此时触发 b 的 getter 会将当前的 watcher 添加到 b 的订阅者列表 subs 中。也就是说如果 ComponentA 依赖 b，则将该组件的渲染 Watcher 作为订阅者加入 b 的订阅者列表中。

![](https://user-gold-cdn.xitu.io/2019/12/27/16f452dfc2cdd772?w=1582&h=480&f=png&s=321949)



#### Dep - 订阅器

数据对象中的 get 方法主要使用 depend 方法进行依赖收集，depend 是 Dep 类中的属性方法，继续来看 [Dep 类](https://github.com/DQFE/vue/blob/dev/src/core/observer/dep.js#L13)是怎样实现的：
```javascript
export default class Dep {
    // a*
    static target: ?Watcher;
    id: number;
    subs: Array<Watcher>;
    constructor () {
        this.id = uid++
        this.subs = []
    }

    addSub (sub: Watcher) {
        this.subs.push(sub)
    }

    depend () {
        if (Dep.target) {
            Dep.target.addDep(this)
        }
    }
}

Dep.target = null
const targetStack = []

export function pushTarget (target: ?Watcher) {
    targetStack.push(target)
    Dep.target = target
}

export function popTarget () {
    targetStack.pop()
    Dep.target = targetStack[targetStack.length - 1]
}
```
Dep 类中有三个属性：`target`、`id` 和 `subs`，分别表示当前全局唯一的静态数据依赖的监听器 Watcher、该属性的 id 以及订阅这个属性数据的订阅者列表[a*]，其中subs其实就是存放了所有订阅了该数据的订阅者们。另外还提供了将订阅者添加到订阅者列表的 `addSub方法`、从订阅者列表删除订阅者的 `removeSub` 方法。

`Dep.target` 是当前全局唯一的订阅者，这是因为同一时间只允许一个订阅者被处理。target 的意思就是指当前正在处理的目标订阅者，而这个订阅者是有一个订阅者栈 `targetStack`，当某一个组件执行到某个生命周期的 hook 时（例如 mountComponent），会将当前目标订阅者 target 置为这个 watcher，并将其压入 `targetStack` 栈顶。

接下来是添加当前数据依赖的 `depend方法`，`Dep.target` 对象是一个 Watcher 类的实例，调用 Watcher 类的 [addDep 方法](https://github.com/DQFE/vue/blob/dev/src/core/observer/watcher.js#L128)，我们看下 addDep 方法将当前的依赖 Dep 实例放在了哪里：
```javascript
addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
        this.newDepIds.add(id)
        this.newDeps.push(dep)
        if (!this.depIds.has(id)) {
            dep.addSub(this)
        }
    }
}
```
我们稍后会对 Watcher 类进行分析，目前先来简单看下 addDep 这个属性方法做了什么。可以看到入参是一个 Dep 类实例，这个实例实际上是当前全局唯一的订阅者，newDepIds 是当前数据依赖 dep 的 id 列表，newDeps 是当前数据依赖 dep 列表，depsId 则是上一个 tick 的数据依赖的 id 列表。这个方法主要的逻辑就是调用当前数据依赖 dep 的类方法 `addSub`，而这个方法在上面Dep 类方法中可以看到，就是将当前全局唯一的 watcher 实例放入这个数据依赖的订阅者列表中。

#### target Watcher的产生

我们以生命周期中的 [mountComponent](https://github.com/DQFE/vue/blob/dev/src/core/instance/lifecycle.js#L141) 这一个阶段为例来分析上面提到的 Dep.target 这个全局唯一的当前订阅者 Watcher 是怎样产生的。

```javascript
export function mountComponent (
    vm: Component, 
    el: ?Element, hydrating?: boolean
): Component {
    // ...
    let updateComponent
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) { 
        updateComponent = () => { // 首次渲染视图会进入，生成虚拟Node，更新DOM}
    } else {
        updateComponent = () => { // 数据更新时会进入，调用渲染函数对视图进行更新
        vm._update(vm._render(), hydrating)
        }
    }
    new Watcher(vm, updateComponent, noop, {
        before () {
            if (vm._isMounted && !vm._isDestroyed) {
                callHook(vm, 'beforeUpdate')
            }
        }
    }, true /* isRenderWatcher */)
    // ...
    return vm
}
```
通过mountComponent的代码可以看到在触发了`beforeMount`生命周期钩子后，紧接着创建了一个用于渲染Watcher实例。我们通过Watcher类的构造函数来看创建 [Watcher](https://github.com/DQFE/vue/blob/dev/src/core/observer/watcher.js#L45) 实例时做了哪些工作：
```javascript
constructor (
    vm: Component,              // 当前渲染的组件
    expOrFn: string | Function,  // 当前Watcher实例的getter属性方法
    cb: Function,               // 回调函数
    options?: ?Object,          // 手动传入watch时的option
    isRenderWatcher?: boolean    // 是否为渲染watcher的标识
  ) {
    this.vm = vm
    if (isRenderWatcher) {
        vm._watcher = this
    }
    vm._watchers.push(this)
    // 此处处理我们传入的watcher参数（在组件内手动新建watch属性）
    if (options) {} else {}
    // ...
    // parse expression for getter
    if (typeof expOrFn === 'function') {
        this.getter = expOrFn
    } else {
        this.getter = parsePath(expOrFn)
        if (!this.getter) {
            this.getter = noop
        }
    }
    this.value = this.lazy ? undefined : this.get()
}
```
通过 Watcher 类的构造函数传参可以看到 mountComponent 生命周期中创建的 Watcher 是一个渲染 Watcher，将当前的 getter 设置为了 updateComponent 方法（也就是重新渲染视图的方法），最后调用了 get 属性方法，我们接下来看 [get 方法](https://github.com/DQFE/vue/blob/dev/src/core/observer/watcher.js#L101)做了什么：
```javascript
get () {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
        value = this.getter.call(vm, vm)
    } catch (e) {
        if (this.user) {
            handleError(e, vm, `getter for watcher "${this.expression}"`)
        } else { throw e }
    } finally {
        if (this.deep) {
            traverse(value)
        }
        popTarget()
        this.cleanupDeps()
    }
    return value
}
```
get 方法首先调用了在 `Dep类`文件中定义的全局方法 pushTarget，将 Dep.target 置为当前正在执行的渲染 Watcher，并将这个 watcher 压到了 targetStack。接下来调用 wathcer 的 getter 方法：由于此时的 getter 就是 updateComponent 方法，执行 updateComponent 方法，也就是 `vm._update(vm._render(), hydrating)` 进行渲染，渲染过程中必然会 touch 到 data 中相关依赖的属性，此时就会触发各个属性中的 get 方法（也就是将当前的渲染 Watcher 添加到所有渲染中使用到的数据的依赖列表 subs 中）。

渲染完之后会将当前的渲染 Watcher 从 targetStack 推出，进行下一个 watcher 的任务。最后会进行*依赖的清空*，每次数据的变化都会引起视图重新渲染，每一次渲染又会进行依赖收集，又会触发 data 中每个属性的 get 方法，这时会有一种情况发生：`<div v-if="someBool">{{ a }}</div><div v-else>{{ b }}</div>` 第一次当 someBool 为真时，进行渲染，当我们把 someBool 的值 update 为 false 时，这时候属性 a 不会被使用，所以如果不进行依赖清空，会导致不必要的依赖通知。依赖清空主要是对 newDeps 进行更新，通过对比本次收集的依赖和之前的依赖进行对比，把不需要的依赖清除。

### 派发更新

![](https://user-gold-cdn.xitu.io/2019/12/27/16f452e5de243465?w=1643&h=650&f=png&s=264238)

当我们修改一个存在 data 属性上的值时，会触发数据中的 [set 属性方法](https://github.com/DQFE/vue/blob/dev/src/core/observer/index.js#L173)，首先会判断并修改该属性数据的值，并触发自身的 dep.notify 方法开始更新派发：

```javascript
set: function reactiveSetter (newVal) {
  const value = getter ? getter.call(obj) : val
  if (newVal === value || (newVal !== newVal && value !== value)) {
    return
  }
  // ...
  childOb = !shallow && observe(newVal)
  dep.notify()
}
```
接下来让我们进入[`dep.notify`](https://github.com/DQFE/vue/blob/dev/src/core/observer/dep.js#L37)这个方法：
```javascript
notify () {
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
        subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
```
notify 这个方法是每一个数据在响应式化之后自己内部闭包的 dep 实例的方法，还记得之前讲过的 subs 保存着所有订阅该数据的订阅者，所以在 notify 方法中首先对 subs 这个订阅者序列按照其 id 进行了排序。接下来就是调用每一个订阅者 watcher 实例的 `update` 方法进行更新的派发。update 方法是在 Watcher 类文件中，使用了队列的方式来管理订阅者的更新派发，其中主要调用了 [`queueWatcher`](https://github.com/DQFE/vue/blob/dev/src/core/observer/scheduler.js#L164) 这个方法来实现该逻辑：
```javascript
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {    // a*
    has[id] = true
    if (!flushing) {        // b*
      queue.push(watcher)
    } else {
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    if (!waiting) {         // c*
      waiting = true
      // ...
      nextTick(flushSchedulerQueue)
    }
  }
}
```
进入该方法后，首先使用一个名为 has 的 Map 来保证每一个 watcher 仅推入队列一次[a*]。`flushing` 这个标识符表示目前这个队列是否正在进行更新派发，如果是，那么将这个 id 对应的订阅者进行替换，如果已经路过这个 id，那么就立刻将这个 id 对应的 watcher 放在下一个排入队列[b*]。接下来根据`waiting` 这个标识符来表示当前是否正在对这个队列进行更新派发[c*]，如果没有的话，就可以调用 [`nextTick(flushSchedulerQueue)`](https://github.com/DQFE/vue/blob/dev/src/core/observer/scheduler.js#L71) 方法进行真正的更新派发了。

```javascript
function flushSchedulerQueue () {
  flushing = true
  let watcher, id

  queue.sort((a, b) => a.id - b.id)
  
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    watcher.run()
    // ...
  }
```
这个方法首先对队列中的 watcher 按照其 id 进行了排序，排序的主要目的有三（[Vue.js技术揭秘](https://ustbhuangyi.github.io/vue-analysis/v2/reactive/setters.html#%E8%BF%87%E7%A8%8B%E5%88%86%E6%9E%90)）：
- 组件的更新由父到子；因为父组件的创建过程是先于子的，所以 watcher 的创建也是先父后子，执行顺序也应该保持先父后子。
- 用户的自定义 watcher 要优先于渲染 watcher 执行；因为用户自定义 watcher 是在渲染 watcher 之前创建的。
- 如果一个组件在父组件的 watcher 执行期间被销毁，那么它对应的 watcher 执行都可以被跳过，所以父组件的 watcher 应该先执行。

排序结束之后，便对这个队列进行遍历，并执行`watcher.run()`方法去实现数据更新的通知，run 方法的逻辑是：
- 新的值与老的值不同时会触发通知；
- 但是当值是对象或者 deep 为 true 时无论如何都会进行通知


所以 watcher 有两个功能，一个是将属性的值进行更新，另一个就是可以执行 watch 中的回调函数 handler(newVal, oldVal)，这也是为何我们可以在 watcher 中拿到新旧两个值的原因。

至此，数据的更新派发也通知到了各个组件，对应的视图需要进一步的更新渲染，依赖收集和更新派发……

通过对数据响应式这一部分的源码分析，可以看到 Vue 为了实现监听数据动态变化来进行对应的视图渲染和其他操作，为每一个数据都闭包了一个订阅者管理器，即 Dep 实例，并且为每一个对数据的依赖都创建了一个 Wathcer 实例作为订阅者存放在数据自己的订阅者列表 subs 中。渲染时通过数据中的 get 属性方法来收集依赖，数据更新时通过其 set 属性方法来通知到对应的 watcher 去做相应的更新操作或执行某个回调函数。

本文仅针对对象类型数据展开了数据响应式原理和代码的分析，但是对于对象类型数据中新增属性的响应式以及数组类型数据的响应式都未涉及，而这些更能够体现出 Vue 的数据响应式的整体性设计。在 Vue3 推出后，Vue 的数据响应式的实现过程中，数据劫持的方式和所支持的数据类型都有更多优化，依赖收集的方式和逻辑与之前也有所不同，具有了更易维护的特性。通过对这些源码的分析，不仅能够了解其内部的实现原理，让我们在使用这些框架的过程中得心应手，更能够通过其代码的组织和版本演进去理解作者的创作思路。
