## 前言
vue 是如何将编译器中的代码转换为页面真实元素的？这个过程涉及到模板编译成 AST 语法树，AST 语法树构建渲染函数，渲染函数生成虚拟 dom，虚拟 dom 编译成真实 dom 这四个过程。前两个过程在我们 vue 源码解读系列文章的上一期已经介绍过了，所以本文会接着上一篇文章继续往下解读，着重分析后两个过程。
## 整体流程
解读代码之前，先看一张 vue 编译和渲染的整体流程图:
![](https://note.youdao.com/yws/res/9138/WEBRESOURCE3035931892114daff0716d8de5640a1d)

vue 会把用户写的代码中的 <template></template> 标签中的代码解析成 AST 语法树，再将处理后的 AST 生成相应的render函数，render 函数执行后会得到与模板代码对应的虚拟 dom，最后通过虚拟 dom 中新旧 vnode 节点的对比和更新，渲染得到最终的真实 dom。
有了这个整体的概念我们再来结合源码分析具体的数据渲染过程。
## 从vm.$mount开始
vue 中是通过 $mount 实例方法去挂载 vm 的,数据渲染的过程就发生在 vm.$mount 阶段。在这个方法中，最终会调用 mountComponent 方法来完成数据的渲染。我们结合源码看一下其中的几行关键代码：

```
 updateComponent = () => {
      vm._update(vm._render(), hydrating) // 生成虚拟dom，并更新真实dom
    }
```
这是在 mountComponent 方法的内部，会定义一个 updateComponent 方法，在这个方法中 vue 会通过 vm._render() 函数生成虚拟 dom，并将生成的 vnode 作为第一个参数传入 vm._update() 函数中进而完成虚拟 dom 到真实 dom 的渲染。第二个参数 hydrating 是跟服务端渲染相关的，在浏览器中不需要关心。这个函数最后会作为参数传入到 vue 的 watch 实例中作为 getter 函数，用于在数据更新时触发依赖收集，完成数据响应式的实现。这个过程不在本文的介绍范围内，在这里只要明白，当后续 vue 中的 data 数据变化时，都会触发 updateComponent 方法，完成页面数据的渲染更新。具体的关键代码如下：

```
  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        // 触发beforeUpdate钩子
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  hydrating = false

  // manually mounted instance, call mounted on self
  // mounted is called for render-created child components in its inserted hook
  if (vm.$vnode == null) {
    vm._isMounted = true
    // 触发mounted钩子
    callHook(vm, 'mounted')
  }
  return vm
}
```
代码中还有一点需要注意的是，在代码结束处，会做一个判断，当 vm 挂载成功后，会调用 vue 的 mounted 生命周期钩子函数。这也就是为什么我们在 mounted 钩子中执行代码时，vm 已经挂载完成的原因。
## vm._render()
接下来具体分析 vue 生成虚拟 dom 的过程。前面说了这一过程是调用vm._render()方法来完成的，该方法的核心逻辑是调用vm.$createElement方法生成vnode，代码如下：

```
vnode = render.call(vm._renderProxy, vm.$createElement)
```
其中vm.renderProxy是个代理，代理vm，做一些错误处理，vm.$createElement 是创建vnode的真正方法，该方法的定义如下：

```
vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)
```
可见最终调用的是createElement方法来实现生成vnode的逻辑。在进一步介绍createElement方法之前，我们先理清楚两个个关键点，1.render的函数来源，2.vnode到底是什么
#### render方法的来源

在 vue 内部其实定义了两种 render 方法的来源，一种是如果用户手写了 render 方法，那么 vue 会调用这个用户自己写的 render 方法，即下面代码中的 vm.$createElement；另外一种是用户没有手写 render 方法，那么vue内部会把 template 编译成 render 方法，即下面代码中的 vm._c。不过这两个 render 方法最终都会调用createElement方法来生成虚拟dom

```
// bind the createElement fn to this instance
  // so that we get proper render context inside it.
  // args order: tag, data, children, normalizationType, alwaysNormalize
  // internal version is used by render functions compiled from templates
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
  // normalization is always applied for the public version, used in
  // user-written render functions.
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)
```
#### vnode类
vnode 就是用一个原生的 js 对象去描述 dom 节点的类。因为浏览器操作dom的成本是很高的，所以利用 vnode 生成虚拟 dom 比创建一个真实 dom 的代价要小很多。vnode 类的定义如下：

```
export default class VNode {
  tag: string | void; // 当前节点的标签名
  data: VNodeData | void; // 当前节点对应的对象
  children: ?Array<VNode>; // 当前节点的子节点
  text: string | void; // 当前节点的文本
  elm: Node | void; // 当前虚拟节点对应的真实dom节点
  ....
  
  /*创建一个空VNode节点*/
  export const createEmptyVNode = (text: string = '') => {
    const node = new VNode()
    node.text = text
    node.isComment = true
    return node
  }
  /*创建一个文本节点*/
  export function createTextVNode (val: string | number) {
    return new VNode(undefined, undefined, undefined, String(val))
  }
   ....
```
可以看到 vnode 类中仿照真实 dom 定义了很多节点属性和一系列生成各类节点的方法。通过对这些属性和方法的操作来达到模仿真实 dom 变化的目的。

### createElement
有了前面两点的知识储备，接下来回到 createElement 生成虚拟 dom 的分析。createElement 方法中的代码很多，这里只介绍跟生成虚拟 dom 相关的代码。该方法总体来说就是创建并返回一个 vnode 节点。 在这个过程中可以拆分成三件事情：1.子节点的规范化处理； 2.根据不同的情形创建不同的 vnode 节点类型；3.vnode 创建后的处理。下面开始分析这3个步骤：
##### 子节点的规范化处理

```

  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }
```
为什么会有这个过程，是因为传入的参数中的子节点是 any 类型，而 vue 最终生成的虚拟 dom 实际上是一个树状结构，每一个 vnode 可能会有若干个子节点，这些子节点应该也是 vnode 类型。所以需要对子节点处理，将子节点统一处理成一个 vnode 类型的数组。同时还需要根据 render 函数的来源不同，对子节点的数据结构进行相应处理。
##### 创建vnode节点
这部分逻辑是对tag标签在不同情况下的处理，梳理一下具体的判断case如下：

1. 如果传入的 tag 标签是字符串，则进一步进入下列第 2 点和第 3 点判断，如果不是字符串则创建一个组件类型 vnode 节点。
2. 如果是内置的标签，则创建一个相应的内置标签 vnode 节点。
3. 如果是一个组件标签，则创建一个组件类型 vnode 节点。
4. 其他情况下，则创建一个命名空间未定义的 vnode 节点。

```
  let vnode, ns
  if (typeof tag === 'string') {
    let Ctor
    // 获取tag的名字空间
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)

    // 判断是否是内置的标签，如果是内置的标签则创建一个相应节点
    if (config.isReservedTag(tag)) {
      // platform built-in elements
      if (process.env.NODE_ENV !== 'production' && isDef(data) && isDef(data.nativeOn)) {
        warn(
          `The .native modifier for v-on is only valid on components but it was used on <${tag}>.`,
          context
        )
      }
      vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
      // 如果是组件，则创建一个组件类型节点
      // 从vm实例的option的components中寻找该tag，存在则就是一个组件，创建相应节点，Ctor为组件的构造类
    } else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      // unknown or unlisted namespaced elements
      // check at runtime because it may get assigned a namespace when its
      // parent normalizes children

      //其他情况，在运行时检查，因为父组件可能在序列化子组件的时候分配一个名字空间
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
    // direct component options / constructor
    // tag不是字符串的时候则是组件的构造类，创建一个组件节点
    vnode = createComponent(tag, data, context, children)
  }
```

##### vnode创建后的处理
这部分同样也是一些 if/else 分情况的处理逻辑：
1. 如果 vnode 成功创建，且是一个数组类型，则返回创建好的 vnode 节点
2. 如果 vnode 成功创建，且有命名空间，则递归所有子节点应用该命名空间
3. 如果 vnode 没有成功创建则创建并返回一个空的 vnode 节点

```
  if (Array.isArray(vnode)) {
    // 如果vnode成功创建，且是一个数组类型，则返回创建好的vnode节点
    return vnode
  } else if (isDef(vnode)) {
    // 如果vnode成功创建，且名字空间，则递归所有子节点应用该名字空间
    if (isDef(ns)) applyNS(vnode, ns)
    if (isDef(data)) registerDeepBindings(data)
    return vnode
  } else {
    // 如果vnode没有成功创建则创建空节点
    return createEmptyVNode()
  }
```

## vm._update()
vm._update() 做的事情就是把 vm._render() 生成的虚拟 dom 渲染成真实 dom。_update() 方法内部会调用 vm.__patch\_\_ 方法来完成视图更新，最终调用的是 createPatchFunction 方法，该方法的代码量和逻辑都非常多，它定义在 src/core/vdom/patch.js 文件中。下面介绍下具体的 patch 流程和流程中用到的重点方法：
#### 重点方法
1. createElm：该方法会根据传入的虚拟 dom 节点创建真实的 dom 并插入到它的父节点中
2. sameVnode：判断新旧节点是否是同一节点。
3. patchVnode：当新旧节点是相同节点时，调用该方法直接修改节点，在这个过程中，会利用 diff 算法，循环进行子节点的的比较,进而进行相应的节点复用或者替换。
4. updateChildren方法：diff 算法的具体实现过程

#### patch流程
##### 第一步：
判断旧节点是否存在，如果不存在就调用 createElm()  创建一个新的 dom 节点，否则进入第二步判断。

```
 if (isUndef(oldVnode)) {
      // empty mount (likely as component), create new root element
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)
 }
```

##### 第二步：
通过 sameVnode() 判断新旧节点是否是同一节点，如果是同一个节点则调用 patchVnode() 直接修改现有的节点，否则进入第三步判断

```
const isRealElement = isDef(oldVnode.nodeType)
if (!isRealElement && sameVnode(oldVnode, vnode)) {
    // patch existing root node
    /*是同一个节点的时候直接修改现有的节点*/
    patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
}
```

##### 第三步：
如果新旧节点不是同一节点，则调用 createElm()创建新的dom，并更新父节点的占位符，同时移除旧节点。

```
else {
    ....
    createElm(
          vnode,
          insertedVnodeQueue,
          // extremely rare edge case: do not insert if old element is in a
          // leaving transition. Only happens when combining transition +
          // keep-alive + HOCs. (#4590)
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
    )
     // update parent placeholder node element, recursively
        /*更新父的占位符节点*/
        if (isDef(vnode.parent)) {
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode)
          while (ancestor) {
            for (let i = 0; i < cbs.destroy.length; ++i) {
              cbs.destroy[i](ancestor)  /*调用destroy回调*/
            }
            ancestor.elm = vnode.elm
            if (patchable) {
              for (let i = 0; i < cbs.create.length; ++i) {
                cbs.create[i](emptyNode, ancestor)  /*调用create回调*/
              }
              // #6513
              // invoke insert hooks that may have been merged by create hooks.
              // e.g. for directives that uses the "inserted" hook.
              const insert = ancestor.data.hook.insert
              if (insert.merged) {
                // start at index 1 to avoid re-invoking component mounted hook
                for (let i = 1; i < insert.fns.length; i++) {
                  insert.fns[i]()
                }
              }
            } else {
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }
        // destroy old node
        if (isDef(parentElm)) {
          removeVnodes([oldVnode], 0, 0) /* 删除旧节点 */
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode) /* 调用destroy钩子 */
        }
}
```

##### 第四步：
返回 vnode.elm，即最后生成的虚拟 dom 对应的真实 dom，将 vm.$el 赋值为这个 dom 节点，完成挂载。

其中重点的过程在第二步和第三步中，特别是 diff 算法对新旧节点的比较和更新很有意思，diff 算法会单独写一篇文章来详细介绍。

## 其他注意点
#### sameVnode的实际应用
在patch的过程中，如果两个节点被判断为同一节点，会进行复用。这里的判断标准是

1.key相同

2.tag（当前节点的标签名）相同

3.isComment（是否为注释节点）相同

4.data的属性相同

平时写 vue 时会遇到一个组件中用到了 A 和 B 两个相同的子组件，可以来回切换。有时候会出现改变了 A 组件中的值，切到 B 组件中，发现 B 组件的值也被改变成和 A 组件一样了。这就是因为 vue 在 patch 的过程中，判断出了 A 和 B 是 sameVnode，直接进行复用引起的。根据源码的解读，可以很容易地解决这个问题，就是给 A 和 B 组件分别加上不同的 key 值，避免 A 和 B 被判断为同一组件。
#### 虚拟DOM如何映射到真实的DOM节点
vue 为平台做了一层适配层，浏览器平台的代码在 /platforms/web/runtime/node-ops.js。不同平台之间通过适配层对外提供相同的接口，虚拟 dom 映射转换真实 dom 节点的时候，只需要调用这些适配层的接口即可，不需要关心内部的实现。

## 最后
通过上述的源码和实例的分析，我们完成了 Vue 中 数据渲染 的完整解读。如果想要了解更多的 Vue 源码。欢迎进入我们的 [github地址](https://github.com/DQFE/vue) 进行查看，里面对每一行 Vue 源码都做了注释，方便大家的理解。