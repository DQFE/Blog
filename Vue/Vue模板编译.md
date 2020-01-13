### Vue模板编译


> 版本：2.6.10

#### 源码分析准备工作

git地址`(源码+注释)`：https://github.com/DQFE/vue/tree/v2-source-learning
git clone 后打开


#### template和render

在 vue 开发过程中，根据个人开发习惯或者需求，我们可以使用 template 模板或者 render 函数来创建 html。render 函数相较模板更接近编译器。而 template 模式下，其实是 vue 做了编译工作，将 template 模板编译成 render 函数。

参见 [$mount](https://github.com/vuejs/vue/blob/v2.6.10/src/platforms/web/entry-runtime-with-compiler.js) 方法，未配置 render 函数的情况下（即 template 模式），vue 通过调用 compileToFunctions 方法来实现 template 模板的编译，产出 render 和 staticRenderFns。这个 render 和我们编写的 render 意义是一样的，会返回 vnode 节点以供页面渲染和 update 时的 patch 工作。staticRenderFns 是 vue 对 template 模板中静态节点的一个优化，避免静态节点的 patch 和重复渲染，来提高性能。
```javascript
const { render, staticRenderFns } = compileToFunctions(template, someOptions, this)
```
[compileToFunctions](https://github.com/vuejs/vue/blob/v2.6.10/src/platforms/web/compiler/index.js) 是从 createCompiler 函数的返回值中解构出来的。createCompiler 顾名思义，是用来创建编译器的。它主要是通过将 baseCompile 作为从参数调用 createCompilerCreator （编译器创建者的创建者。。）产出的。

vue 采用 `createCompilerCreator` 这样的实现方式主要是为了**便于兼容各个平台**。我们可以根据平台或者其他一些个性化需求提供自己的 baseCompile 函数传入 createCompileCreator 函数，这样就可以构建一个我们自己的编译器函数供外部使用（不需要去考虑缓存优化、错误收集等等这些问题）。

[baseCompile](https://github.com/vuejs/vue/blob/v2.6.10/src/compiler/index.js) 是模板编译的主方法，完成从 template 到 render 函数的一个转化工作。

两个入参，template：我们编写的 template 模板内容；options：编译器的一些平台化选项参数，即不同平台（web、weex）上的vue编译渲染的个性化的一些配置参数。

和大多数编译过程类似，vue 的模板编译可以大致分成三个阶段：词法分析、句法分析和代码生成。

在 baseCompile 方法中，parse 函数担当了词法分析和句法分析的责任。optimize  主要是遍历 ast 树去寻找并标记所有纯静态节点，为代码优化做准备工作。generate将 parse 后的 ast 树转化为目标平台的渲染函数 render，并基于 optimize 的标记结果来生成 staticRenderFns。
```JavaScript
function baseCompile (
  template: string,
  options: CompilerOptions
): CompiledResult {
  /**
   * 调用parse将模板解析成ast
   */
  const ast = parse(template.trim(), options)
  optimize(ast, options)
  // 根据给定的AST生成目标平台的代码
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
}
```
#### parse
[parse](https://github.com/vuejs/vue/blob/v2.6.10/src/compiler/parser/index.js) 是模板解析的核心方法。它的主要逻辑就是 parseHTML 的调用。parseHTML 方法实际上做的就是词法分析的工作，将 template 模板解析成 token 字符流。它有两个入参，第一个是 template 模板，第二个是一系列钩子函数，这些钩子函数 start、end、chars、comment 的工作就是句法分析，将 token 字符流协助转化成 ast 节点。

因此，其实在 vue 模板编译中，词法分析和句法分析并不独立，它俩是耦合的。

最终形成的ast树的结构的控制逻辑在 parse 方法里。
parseHTML 只是从前往后去遍历 template，它通过<>去识别标签的开始和结束，用各种正则匹配表达式来作为辅助工具，来判断出当前这一段 html 是开始标签、结束标签、标签内内容还是纯注释，并且提取出开始标签里的各个属性配置，然后去调用相应的钩子。
而 parse 方法里的主要钩子函数：start 和 end。start 钩子函数根据会进行属性的处理，并生成 ast 节点。end 钩子函数会对这个元素进行闭合。假设调用 start 钩子生成 A 标签的节点，那么在遇到 A 标签的 end 钩子调用前，中间所有的其它节点都是这个节点的后代节点。parse 是通过最近的相同标签来判断是否是一个元素的，和我们写 html 的规范保持一致。
根据这个逻辑，parse 从 html 的第一个元素开始作为根节点一步步往下构建出完整的抽象语法树。
```typescript
parseHTML(template, {
    start (tag, attrs, unary, start, end) {},
    end (tag, start, end) {},
    chars (text: string, start: number, end: number) {},
    comment (text: string, start, end) {}
}
```
接下来我们结合源码来看一下具体的词法分析和句法分析分别的实现。
##### 词法分析：parseHTML
我们首先通过 [parseHTML](https://github.com/vuejs/vue/blob/v2.6.10/src/compiler/parser/html-parser.js) 来理解词法分析的实现。
在 html-parser 这个文件里，vue 定义了一大堆正则匹配表达式，包括属性 attribute、动态属性 dynamicArgAttribute、开始标签的开始标志 startTagOpen、开始标签的关闭标志 startTagClose、结束标签 endTag 等等等等。理解这些正则表达式除了可以更好的理解源码，也可以帮助我们了解更多 vue 的用法。
```javascript
export function parseHTML (html, options) {
  const stack = [] 
  let index = 0 
  let last, lastTag 
  while (html) {
    last = html 
    if (!lastTag || !isPlainTextElement(lastTag)) {    
        const endTagMatch = html.match(endTag)
        if (endTagMatch) {
           const curIndex = index
           advance(endTagMatch[0].length)
           parseEndTag(endTagMatch[1], curIndex, index)
           continue
        }
 
        const startTagMatch = parseStartTag() 
        if (startTagMatch) {
           handleStartTag(startTagMatch)
        }   
      }
    } 
  }
  // Clean up any remaining tags
  parseEndTag()
  function advance (n){}
  function parseStartTag () {}
  function handleStartTag (match) {}
  function parseEndTag (tagName, start, end) {}
}
```
我们先了解一下方法开头的几个重要变量：
* stack：存储未解析完的非一元标签的节点的栈，主要目的是为了给未正常闭合的节点进行闭合处理
* index：记录当前字符流的读入位置（读取的 html 的位置），
* last：存储剩余的尚未解析的模板内容（html 剩余部分），
* lastTag：存储stack栈顶节点的标签。

我们注意源码中的这两处代码，它们就是用来识别当前 html 开头是开始标签还是结束标签，结束标签一般以</tagName>的形式存在，因此可直接通过正则匹配；开始标签较结束标签复杂一些，会带有一些属性，因此需要通过 parseStartTag 去识别并提取里面的属性信息。
> const endTagMatch = html.match(endTag)
> const startTagMatch = parseStartTag() 

在识别之后，分别通过 parseEndTag 和 handleStartTag 对标签进行处理。 handleStartTag 主要做两件事：1. 将非一元标签推入 stack 栈内；2. 调用 start 钩子。parseEndTag 会结合当前节点的标签和 stack 进行比较，对中间可能残留的一些未正常闭合的节点进行闭合处理，闭合处理指的就是调用 options.end 钩子。
我们以下面这段 html 为例，来了解每次循环是怎么处理 html 内容的。
```html
<div class="myroot" :desc="desc">
	<p>内容1<div></p>
	<inner-con 
		v-for="f in data"
		:key="f.key"
		@click="onClick(f)"/>
</div>
```
**第一次循环**：
parseStartTag：解析最开头的 div 开始标签内容，提取出该节点的 attrs，同时调用 advance 方法从 html 中剔除当前开始标签内容，html 变为`"<p>内容1..."`，为下一次循环做准备。
handleStartTag：在 stack 中存储当前节点标签，并调用 start 钩子。


![f5a187be78dbd30575bdad9a41b025d8.png](https://user-gold-cdn.xitu.io/2020/1/3/16f6a2d2102bade7?w=2159&h=1002&f=png&s=200701)

**第二次循环**：
与循环 1 相同，处理 p 开始标签。

![8793a25f5d3bcb09da1888323fbdbcd2.png](https://user-gold-cdn.xitu.io/2020/1/3/16f6a2d21bdacf10?w=400&h=130&f=png&s=9169)

**第三次循环**：
处理 p 标签后的内容“内容1”，调用 chars 钩子。

**第四次循环**：
处理“<div>”,注意，这里会做一个异常情况的处理。对于 p 文本标签而言，它里面是不会出现类似 div 这种块标签的。
handleStartTag 方法内通过 isNonPhrasingTag 识别出当前标签 div 不是文本标签。为了使最终页面正常显示，会对上一个 p 标签做闭合处理。因此这里会调用 parseEndTag 方法：先从 stack 中 pop 出 p 标签，并调用 end 钩子结束 p 标签。而后再调用 start 钩子处理当前 div 标签。
此时 stack 里剩余两个 div 标签。
![e38af88b0c654d234a7fec127216fd35.png](https://user-gold-cdn.xitu.io/2020/1/3/16f6a2d2206836e9?w=2017&h=566&f=png&s=133998)

**第五次循环**：
通过 html.match(endTag) 识别并提取出结束标签"</p>"，然后调用 parseEndTag 方法。
从循环 4 我们得知，parseEndTag 主要是从 stack 中 pop 出需要结束的标签，并调用 end 钩子。按照规范的 html 写法，stack 的最后一个标签理当和当前结束标签相同。那么如果不同呢？现在我们处理"</p>"就会发现，stack 里并不存在 p 标签。parseEndTag 方法会对类似异常进行处理。
parseEndTag 方法的思路是：

1. 从 stack 末尾开始遍历寻找第一个相同标签
2. 如果没找到，那么这段 html 会被自动忽略。对于 br 标签和 p 标签的解析会做特殊处理，以保持与浏览器的行为一致。类似此例中，会做一个补足工作，对 p 先调用 start 钩子，再调用 end 钩子。
3. 如果找到了，然而不是 stack 的尾元素，意味着中间有一些未闭合的标签。parseEndTag 会做一个自动闭合工作。类似`<div><h1><h2></div>`，当我们处理</div>这个结束标签的时候，stack=['div','h1','h2']。对div闭合处理前，会先将 h1,h2 进行闭合。

因此，stack 依旧剩余两个 div 标签。

**第六次循环**：
处理 `inner-con` ，与循环 1 类似。区别在于，此时通过正则匹配识别当前标签自闭合。不会将该标签推入 stack 栈内，调用 start 钩子方法时，会同步自闭合特征。

**第七次循环**：
处理最后的“</div>”。调用 parseEndTag 进行闭合处理。stack 内剩余一个 div 标签。

经过 7 次循环，最初的 html 已经被完全处理完。但是 stack 里还有一个 div 标签。这个时候，可以看到 while 循环体外又调用了一次 parseEndTag。这一次调用的主要作用是为了闭合 stack 内的剩余标签。完整的标签处理结果以及对应钩子函数的调用流程可参见下图：

![965eeb1a112e641195f3a4d3801581aa.png](https://user-gold-cdn.xitu.io/2020/1/3/16f6a2d26078bccd?w=1862&h=1105&f=png&s=135656)
经过 handleStartTag 和 parseEndTag 做的一些闭合和补足工作，实际上其最终的效果应当是等同于下面这段 html：
```html
<div class="myroot" :desc="desc">
    <p>内容1</p>
    <div>
        <p></p>
        <inner-con 
            v-for="f in data"
            :key="f.key"
            @click="onClick(f)"/>
    </div>
</div>
```
##### 句法分析
经过上面对 parseHTML 的理解，我们知道了每当遇到开始标签时，会调用 `parse` 中定义的 `start` 钩子，遇到结束标签时，会调用 `end` 钩子。接下来，我们来看看 Vue 是怎么通过这几个钩子来生成 ast 树的。

我们还是从源码出发，先看看 `parse` 方法的整体组成。
###### 入参
> `template`: 我们需要解析的模板 html
> `options`: 则是编译器的一些**平台化选项参数**，即不同平台（web、weex）上的vue编译渲染的个性化的一些配置参数。

```javascript
let transforms, preTransforms, postTransforms
function parse (
    template: string,
    options: CompilerOptions
  ): ASTElement | void {
    transforms = pluckModuleFunction(options.modules, 'transformNode')// 根据编译器的选项参数对平台化的变量进行了初始化以及一些其他变量的定义
    const stack = []
    let root
    let currentParent
    let inVPre = false
    let inPre = false
 
    parseHTML(template, {
        start (tag, attrs, unary, start, end) {},
        end (tag, start, end) {},
        chars (text, start, end) {},
        comment (text, start, end) {}
    }
}
```
###### 变量介绍
* `stack`: 存储处理过的**未结束的节点**；
* `root`: 定义**根节点**，作为最终的 ast 树返回；
* `currentParent`: 当前处理节点的**父节点**；
* `inVPre`和`inPre`: 这两个变量主要用来标记节点是否在有**v-pre属性的标签内或者在pre标签内**，这决定最终将如何对节点的属性进行处理。

###### **工具函数介绍**
* `process` 函数：对当前元素描述对象做进一步处理（**在元素描述对象上添加各种各样的具有标识作用的属性**）。
    我们看 `parse` 方法的大纲就可以发现其内包含大量 `process` 函数，从命名上也可以看出这是针对元素各个属性和指令进行了区分处理（v-for、v-if、v-pre等）。
 
![](https://user-gold-cdn.xitu.io/2020/1/3/16f6a2f2fa340549?w=430&h=673&f=png&s=141568)

* `transform` 函数：指的是存储在 `transforms`， `preTransforms`，`postTransforms` 这三个变量中的函数。它们来源于对应平台下的 options.modules 配置。以 web 平台为例，如下图，实际上就存储在 `src/platforms/web/compiler/modules` 。这三个变量其实是根据调用时机进行区分命名的, 是不同调用时机下的函数。
**和 `process` 函数的功能相同**，唯一的区别就是**平台化的区分**，`process` 函数是不区分平台执行的，而 `transform` 函数是处理对应平台下的相关逻辑的。

![](https://user-gold-cdn.xitu.io/2020/1/3/16f6a30d6c91f4f4?w=359&h=779&f=png&s=87713)


我们在后面将展开介绍 `process` 和 `transform` 是如何对元素进行处理的。


###### 钩子函数介绍
这里我们主要对 start ， end 这两个关键性钩子函数以及它们所依赖的一些业务逻辑相关的函数做一个了解。

**start 钩子函数**
start 钩子的主要任务是：

1. 根据 parseHTML 的结果（tag 和 attrs）以及当前的父节点CurrentParent **创建 AST 元素**
2. 进行 AST 元素的处理：包括最早执行的 preTransforms，inVPre 和 inPre 赋值以及一些 process 操作
3. 根元素的赋值
4. 若元素为一元标签（自闭合），对元素做关闭处理（closeElement）；反之，将当前元素赋值给CurrentParent，并将其推入 stack 内，为下一个元素的任务1做准备。
```javascript
start(tag, attrs, unary){
    let element = createASTElement(tag, attrs, currentParent)
    preTransforms[i](element) // 对当前节点 element 遍历执行 preTransforms
    // inVPre 和 inPre 赋值
    process*(element) //一些 process 操作
    if (!root) {
        root = element
    }

    if (!unary) {
        currentParent = element
        stack.push(element)
    } else {
        closeElement(element)
    }
}
```

**end 钩子函数**
end 钩子函数比较简单，主要的逻辑都在closeElement这个方法里。除此以外，很重要的一个就是状态的回退，都将 currentParent 变量的值回退到之前的元素，保证当前处理的元素拥有正确的父级，stack里面也需要将这个已经结束的元素pop出来。
```javascript
end (tag) {
    const element = stack[stack.length - 1]
    stack.length -= 1
    currentParent = stack[stack.length - 1]
    closeElement(element)
}
```
**补充：AST 元素的主要组成结构**
```javascript
{
    "type": 1, // 
    "tag": "div", // 标签
    "attrsList": [], // 原始属性数组，parseHTML 解析结果
    "attrsMap": {}, // 原始属性数组对应属性字典
    "children": [], // 子元素数组
    "staticClass": "myroot",
    "hasBindings": false, // 是否包含动态属性
}
```
**`process` 和 `transform` 是如何对 AST 元素进行处理的**
我们以 processFor 为例，process 类函数主要做两件事：
1. 通过 getAndRemoveAttr 方法判断当前元素是否有相应属性并提取出相应属性值。
2. 对属性值进行解析，并通过 extend 方法将解析结果挂载到 el 上。
```javascript
// v-for="(obj,ind) in list"
export function processFor (el: ASTElement) {
    let exp
    if ((exp = getAndRemoveAttr(el, 'v-for'))) {
    // exp = "(obj,ind) in list"
        const res = parseFor(exp)
        // res = { for: 'list', alias: 'obj', iterator1: 'ind' }
        if (res) {      
            extend(el, res)
        } 
    }
}
```
**业务函数之 closeElement**
closeElement 方法里主要做了四件事：
1. **父子结构的维系**：将当前节点存储到其父节点的 children 数组中
2. **元素剩余属性的处理 processElement**：processElement 其实是其它剩余的一系列 process 函数的集合，包括 processKey、processRef、processAttrs 等等。通过这些函数对节点的剩余属性进行处理，在这个过程中，还会调用 transforms 方法。web 平台上的 transforms 主要是对 class 和 style 做动态绑定值和静态值的处理。
3. **pre状态的重置**：将 inVPre 和 inPre 这两个属性回退为 false，避免影响后续节点属性的处理；
4. 调用 postTransforms 方法组对节点做最后的处理。
```javascript
function closeElement (element) {
    currentParent.children.push(element)
    if (!inVPre && !element.processed) {
        element = processElement(element, options)
    }
    if (element.pre) {
        inVPre = false
    }
    if (platformIsPreTag(element.tag)) {
        inPre = false
    }
    for (let i = 0; i < postTransforms.length; i++) {
        postTransforms[i](element, options)
    }
}
```

在了解 `parse` 方法的整体结构、重要变量以及其中使用到的函数功能以后，我们依旧以 `parseHTML` 中的 demo 为例，来看看其形成的 AST 树的最终形态（省略部分属性）。

```json
{
    "tag": "div",
    "children": [
        {
            "tag": "p",
            "children": [
                {
                    "text": "内容1"
                }
            ]
        },
        {
            "tag": "div",
            "children": [
                {
                    "tag": "p",
                },
                {
                    "tag": "inner-con",
                    "for": "data",
                    "alias": "f",
                    "key": "f.key",
                    "hasBindings": true,
                    "events": {
                        "click": {
                            "value": "onClick(f)",
                            "dynamic": false
                        }
                    }
                }
            ],
        }
    ],
    "staticClass": "myroot",
    "hasBindings": true
}
```


#### optimize
代码优化的主要工作就是遍历ast树去寻找并标记所有纯静态节点。主要目的是，把这些纯静态节点作为常量，避免每次重新渲染的时候都去创建新节点，在 updateComponent 里面的 vnode patch 这个重要流程里也可以跳过这些节点，从而提高性能。
静态节点的判断主要依赖于 isStatic 方法，整个的判断规则是：
1. 表达式文本节点一定是非静态节点。表达式文本节点如其命名，节点内含有类似'我是{{name}},今年{{age}}岁' 这样的字面量表达式。
2. 普通文本节点或者注释节点一定是静态节点
3. 自身或其祖先节点是 pre 标签或者有 v-pre 属性的节点属于静态节点。
4. 符合以下任意一个条件的节点属于非静态节点：1) 有动态绑定属性，e.g. `<div :class="testClass">` ; 2) 自身或其祖先节点含有 v-for、 v-if 结构化属性 ；3）自定义组件；
5. 子节点为非静态，则其父节点也为非静态节点。

#### generate
generate 方法的核心是：根据 AST 树生成渲染函数。
注意，generate 方法最终产出的有两部分：
1. render 渲染函数体
2. 所有纯静态节点的渲染函数体数组；

且这里的函数体都是以 `with(this){return ${code}}` 类似的字符串形式返回的。

我们知道 AST 树其实是以节点形式返回的， generate 方法就是对根节点进行 genElement 方法处理。在 genElement 方法里会根据传入节点的各个属性去进行判断处理，staticRoot 标志当前节点为静态节点，staticProcessed 标志此节点已经过静态节点处理，以此类推。
每一个处理方法其实只处理节点的一个属性，genElement 这个方法里的 `if else` 看起来是对不同属性不同处理，但是 genOnce、genFor 等等这些 gen 方法内部，其实是会再次调用 genElement 方法去进行这个节点的一个`递归处理`的。这也是为什么要用 onceProcessed、forProcessed 这些` processed 属性进行节点标记`的原因，避免陷入死循环。等处理完节点上所有的 once、for、if、slot 等等属性，整个递归其实就结束了， 最后根据组件和普通节点去进行最终的渲染函数体的生成，并进行返回。
我们在前文`#生成 AST 树`里已知 `inner-con` 对应的 AST 元素是：
```javacript
{
    "tag": "inner-con",
    "for": "data",
    "alias": "f",
    "key": "f.key",
}
```
我们就以 genFor 函数为例来看一下 gen 方法在生成渲染函数体的过程中具体做了什么、返回的又是什么。
```javascript
export function genFor ( el: any, state: CodegenState): string {
    const exp = el.for 
    const alias = el.alias
    const iterator1 = el.iterator1 ? `,${el.iterator1}` : '' 
    const iterator2 = el.iterator2 ? `,${el.iterator2}` : ''
  
    el.forProcessed = true // avoid recursion
    return `${altHelper || '_l'}((${exp}),` +
      `function(${alias}${iterator1}${iterator2}){` +
        `return ${(altGen || genElement)(el, state)}` +
      '})'
}
```
genFor 节点内会通过 forProcessed 属性标记此节点已进行 for 处理，并根据其 for、 alias、 iterator属性返回一串执行 _l 方法处理的代码字符串。
PS: 这里出现的 _l, genElement 方法中的 _c 以及其他 gen 方法中出现的 _m、_u、_e 等等，都是挂载在 Vue 上的一些中间渲染方法，通过这些渲染方法最终生成dom元素。可以在 [render-helpers](https://github.com/DQFE/vue/tree/v2-source-learning/src/core/instance/render-helpers) 中查找渲染函数体中下划线打头的方法对应的具体方法。

genStatic 方法和其它 gen 方法有些不同，主要区别在于当前节点的渲染函数体其实并不是直接返回的，而是会被存储在 state.staticRenderFns 数组内，用于之后的渲染，最终返回这个节点的渲染函数体里_m的入参是其真正的渲染函数体在 state.staticRenderFns 数组内的下标。所以静态节点的渲染是通过 staticRenderFns 来实现的。

通过 genElement 方法，实现了从 AST 元素到节点的渲染函数体的一个转化，render 就是 vue 渲染模板时的函数体字符串，staticRenderFns 则是一组静态节点的渲染函数体字符串。

```javascript
with(this){
return _c(
    'div',
    {staticClass:"myroot",attrs:{"desc":desc}},         
    [ _c('p',[_v("内容1")]),
      _c('div',
        [ _c('p'),
         _l((data),function(f){
            return _c('inner-con',
                    {key:f.key,attrs:{"data":f},on:{"click":function($event){return onClick(f)}}})
         })],2)])}
```
