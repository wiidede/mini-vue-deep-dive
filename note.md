# Deep Dive with Evan You 笔记

## 1. Intro

Vue 的工作流：

- 将 template 编译成 render function；
- render function 会生成虚拟 DOM，也就是 vNode；
- vNode 经过 Vue 的处理就可以生成真实的 DOM；
- 当数据变化，就生成新的 vNode，与旧的 vNode 进行对比（patch 阶段），
  将有变化的地方反应到真的的 DOM 上

Vue的三个模块：

- reactivity module：将数据做响应式处理
- compiler module：将 template 编译成 render function
- renderer module
  - render phase：render function 生成 vNode；
  - mount phase：vNode 转成真实的 DOM 节点挂载到网页上
  - patch phase：比较旧的 vNode 和 新的 vNode，
    将有变化的部分反应到网页上

## 2. Rendering Mechanism（渲染机制）

vnode的好处：

- 组件渲染逻辑与真实的 DOM 节点解耦
- 复用框架更直接（比如可以自定义渲染方案）
- 在生成渲染函数前去构建出想要的 DOM 结构
（比 template 更灵活地去构造一些特殊的结构）

渲染函数，Vue 3 的改进：

```js
import { h } from 'vue' // 从全局引入 h 函数，而不再作为 render 函数的第一个参数

function render() {
    return h('div', { // 扁平化的对象结构
        id: 'foo',
        onClick: this.onClick // on 开头的会绑定一个 listener
    }, 'hello')
}
```

## 3. How & when to use Render functions

v-if --> `if` / `?:`

v-for --> `list.map(() => {return h(...)})`

就是说 render 函数里是纯粹的 javascript 语句。

如果你觉得使用 template 无法满足你想要表达的逻辑，
使用 javascript 能够更好地组织你要写的代码的时候，
你可以尝试使用 render 函数。

也就是说，日常开发，你可以用 template，因为他更简单直观，
并且 Vue 在编译的时候做了很好地优化，如果你在开发一些底层组件，
逻辑比较复杂的时候，就可以使用 render。

以下是一个使用 render 函数的例子，使用 render 函数，
给 Stack 组件的所有子元素都包裹在一个 div 中，
可以看到使用 render 函数去实现是比较简单的方法，相较于 template。


```html
<script src="https://unpkg.com/vue"></script>
<style>
    .mt-4 {
        margin: 10px;
    }
</style>

<div id="app"></div>

<script>
    const {h, createApp} = Vue

    // 定义 Stack 组件
    const Stack = {
        props: ['size'], // 视频中尤大貌似没有指明 props
        render() {
            // 获取默认插槽
            const slot = this.$slots.default
                ? this.$slots.default()
                : []

            // 将插槽中的所有内容套上 div，并且读取 props 上的 size 属性，
            // 并构成类名
            return h('div', {class: 'stack'}, slot.map(child => {
                return h('div', {class: `mt-${this.$props.size}`}, [
                    child
                ])
            }))
        }
    }

    // App 组件
    const App = {
        template: `
            <Stack size="4">
            <div>hello</div>
            <Stack size="4">
                <div>hello</div>
                <div>hello</div>
            </Stack>
            </Stack>
        `,
        components: {
            Stack
        }
    }

    // 创建 vue 实例并挂载到 DOM 上
    createApp(App).mount('#app')
</script>
```

## 4. Compiler & Renderer API

**Vue 3 Template Explorer**

使用 Vue 3 Template Explorer 就可以看到 Vue 3 把模板编译成了什么样的渲染函数

一些特性（性能优化，在编译成 render function 时给 compiler 足够的提示，
告诉 compiler ，什么需要处理，什么不需要处理）
（这也是为什么写 template 比直接写 render 更高效的原因）：

- 标记静态节点（对象只会在初始化的时候创建一次、diff时直接不比较）
- 标记出会动态变化的属性（只检查动态变化的属性，跳过静态的属性）
- 缓存 event （这样 v-on 就不会被标记为动态变化的属性；如果在子组件上写 v-on 事件，
  在 patch 阶段并不会刷新整个子组件，而 Vue 2 是会的）
- 引入 block 的概念（需要动态更新的节点会被添加到 block 上，无论这个节点有多深；
  v-if 会开启一个新的 block，这个 block 又被其父 block 跟踪；
  总的来说就是在 diff 的时候不需要在去深度遍历判断，而是从 block 记录的动态节点数组上，去遍历会变化的 vNode）
- 动态节点的标记有不同的类型（更方便的 diff）

所以，从一个更高的角度来看 Vue 就是想实现：

```html
<div id="app"></div>

<script>
    function h(tag, props, children) {

    }

    function mount(vnode, container) {

    }

    const vdom = h('div', {class: 'red'}, [
        h('span', null, ['hello'])
    ])

    mount(vdom, document.getElementById('app'))
</script>
```

即：h 生成 vdom，将 vdom 挂载到真实 dom 上

## 5. Creating a Mount function

实现一个简单的 mount

```html
<style>
    .red {
        color: red;
    }
</style>
<div id="app"></div>

<script>
    // 渲染函数 直接返回一个简单的 vNode
    function h(tag, props, children) {
        return {
            tag,
            props,
            children
        }
    }

    // 将 vNode 挂载到真实的 DOM 上面
    function mount(vnode, container) {
        // 根据 vNode 信息创建 DOM
        const el = vnode.el = document.createElement(vnode.tag)

        // props
        if (vnode.props) {
            for (const key in vnode.props) {
                const value = vnode.props[key];
                // 设置 DOM 的属性
                el.setAttribute(key, value)
            }
        }
        // children
        if (vnode.children) {
            // 如果是字符串，直接修改 textContent
            if (typeof vnode.children === 'string') {
                el.textContent = vnode.children
            } else {
                // 数组里面默认都是 vNode
                vnode.children.forEach(child => {
                    mount(child, el)
                })
            }
        }

        // 将创建的 DOM 元素放到 conatiner 中
        container.appendChild(el)
    }
    
    // 直接 h、mount 即可看到页面上已经出现了红色的 hello
    const vdom = h('div', {class: 'red'}, [
        h('span', null, 'hello')
    ])

    mount(vdom, document.getElementById('app'))
</script>
```

## 6. Implemeting a Patch Function

注意这里是从第5节课的基础上扩展出来的

```html
<style>
    .red {
        color: red;
    }

    .green {
        color: green;
    }
</style>
<div id="app"></div>

<script>
    // 渲染函数 直接返回一个简单的 vNode
    function h(tag, props, children) {
        return {
            tag,
            props,
            children
        }
    }

    // 将 vNode 挂载到真实的 DOM 上面
    function mount(vnode, container) {
        // 根据 vNode 信息创建 DOM
        const el = vnode.el = document.createElement(vnode.tag)

        // props
        if (vnode.props) {
            for (const key in vnode.props) {
                const value = vnode.props[key];
                // 设置 DOM 的属性
                el.setAttribute(key, value)
            }
        }
        // children
        if (vnode.children) {
            // 如果是字符串，直接修改 textContent
            if (typeof vnode.children === 'string') {
                el.textContent = vnode.children
            } else {
                // 数组里面默认都是 vNode
                vnode.children.forEach(child => {
                    mount(child, el)
                })
            }
        }

        // 将创建的 DOM 元素放到 conatiner 中
        container.appendChild(el)
    }

    // 直接 h、mount 即可看到页面上已经出现了红色的 hello
    const vdom = h('div', {class: 'red'}, [
        h('span', null, 'hello')
    ])

    mount(vdom, document.getElementById('app'))

    // patch 阶段
    function patch(n1, n2) {
        // 如果两个 node 是相同类型的
        if (n1.tag === n2.tag) {
            // 拿到旧的节点的 DOM
            const el = n2.el = n1.el
            // props
            const oldProps = n1.props || {}
            const newProps = n2.props || {}
            // 遍历新的属性，如果和旧的属性不相等，那么就设置新的属性
            for (const key in newProps) {
                const oldValue = oldProps[key]
                const newValue = newProps[key]
                if (newValue !== oldValue) {
                    el.setAttribute(key, newValue)
                }
            }
            // 遍历旧的属性，如果发现新的属性没有这项了，则删除旧的属性
            for (const key in oldProps) {
                if (!(key in newProps)) {
                    el.removeAttribute(key)
                }
            }

            // children
            const oldChildren = n1.children
            const newChildren = n2.children
            if (typeof newChildren === 'string') {
                if (typeof oldChildren === 'string') {
                    // 新的孩子和旧的孩子都是 string 类型，即都是文本类型
                    if (newChildren !== oldChildren) {
                        // 直接替换 textContent
                        el.textContent = newChildren
                    }
                } else {
                    // 新孩子是文本节点，但是旧孩子是数组，直接设置 textContent，
                    // 这样会覆盖旧的子 DOM 节点，并丢弃它们
                    el.textContent = newChildren
                }
            } else {
                if (typeof oldChildren === 'string') {
                    // 新孩子是数组，但是旧孩子是文本节点
                    // 清空旧的文本节点
                    el.innerText = ''
                    // 遍历数组，将子 vNode 挂载到 el 上
                    newChildren.forEach(child => {
                        mount(child, el)
                    })
                } else {
                    // 新孩子和旧孩子都为数组
                    // diff 阶段
                    // diff 在 Vue 内部有两种方式，一种是通过 key，一种是遍历比较
                    // 这里直接遍历
                    // 获取共有的长度，直接每个子节点都进行 patch
                    // 这样可能非常低效，但是足够直观并且能保证一致性
                    const commonLength = Math.min(oldChildren.length, newChildren.length)
                    for (let i = 0; i < commonLength; i++) {
                        patch(oldChildren[i], newChildren[i])
                    }
                    if (newChildren.length > oldChildren.length) {
                        // 如果新孩子更长，将新孩子多余的节点添加到 DOM 上
                        newChildren.slice(oldChildren.length).forEach(child => {
                            mount(child, el)
                        })
                    } else if (newChildren.length < oldChildren.length) {
                        // 如果旧孩子更长，将旧孩子多余的节点从 DOM 上移除
                        oldChildren.slice(newChildren.length).forEach(child => {
                            el.removeChild(child.el)
                        })
                    }
                }
            }

        } else {
            // 如果两个 node 是不同类型的，则需要用新的 node 替换 旧的 node
            // 这里省略
            // replace
        }
    }

    const vdom2 = h('div', {class: 'green'}, [
        h('span', null, 'changed!')
    ])

    patch(vdom, vdom2)
</script>
```

这里其实会有很多判断的分支，就是 patch 阶段会有各种情况。

Vue 有自己的测试 Vue next coverage，表明有多少代码被测试覆盖到了。
