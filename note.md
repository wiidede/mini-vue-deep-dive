# Deep Dive with Evan You 笔记

## 1 - Intro

Vue 的工作流：

- 将 template 编译成 render function；
- render function 会生成虚拟 DOM，也就是 vNode；
- vNode 经过 Vue 的处理就可以生成真实的 DOM；
- 当数据变化，就生成新的 vNode，与旧的 vNode 进行对比（patch 阶段），将有变化的地方反应到真的的 DOM 上。

Vue的三个模块：

- reactivity module：将数据做响应式处理；
- compiler module：将 template 编译成 render function；
- renderer module：
  - render phase：render function 生成 vNode；
  - mount phase：vNode 转成真实的 DOM 节点挂载到网页上；
  - patch phase：比较旧的 vNode 和 新的 vNode，将有变化的部分反应到网页上；

## 2 - Rendering Mechanism（渲染机制）

vnode的好处：

- 组件渲染逻辑与真实的 DOM 节点解耦
- 复用框架更直接（比如可以自定义渲染方案）
- 在生成渲染函数前去构建出想要的 DOM 结构（比 template 更灵活地去构造一些特殊的结构）

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

## 3 - How & when to use Render functions

v-if --> `if` / `?:`

v-for --> `list.map(() => {return h(...)})`

就是说 render 函数里是纯粹的 javascript 语句。

如果你觉得使用 template 无法满足你想要表达的逻辑，使用 javascript 能够更好地组织你要写的代码的时候，你可以尝试使用 render 函数。

也就是说，日常开发，你可以用 template，因为他更简单直观，并且 Vue 在编译的时候做了很好地优化，如果你在开发一些底层组件，逻辑比较复杂的时候，就可以使用 render。

以下是一个使用 render 函数的例子，使用 render 函数，给 Stack 组件的所有子元素都包裹在一个 div 中，可以看到使用 render 函数去实现是比较简单的方法，相较于 template。


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

## 4 - Compiler & Renderer API

**Vue 3 Template Explorer**

使用 Vue 3 Template Explorer 就可以看到 Vue 3 把模板编译成了什么样的渲染函数。

一些特性（性能优化，在编译成 render function 时给 compiler 足够的提示，告诉 compiler ，什么需要处理，什么不需要处理）（这也是为什么写 template 比直接写 render 更高效的原因）：

- 标记静态节点（对象只会在初始化的时候创建一次、diff时直接不比较）
- 标记出会动态变化的属性（只检查动态变化的属性，跳过静态的属性）
- 缓存 event （这样 v-on 就不会被标记为动态变化的属性；如果在子组件上写 v-on 事件，在 patch 阶段并不会刷新整个子组件，而 Vue 2 是会的）
- 引入 block 的概念（需要动态更新的节点会被添加到 block 上，无论这个节点有多深；v-if 会开启一个新的 block，这个 block 又被其父 block 跟踪；总的来说就是在 diff 的时候不需要在去深度遍历判断，而是从 block 记录的动态节点数组上，去遍历会变化的 vNode）
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

## 5 -  Creating a Mount function

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

        // 将创建的 DOM 元素放到 container 中
        container.appendChild(el)
    }
    
    // 直接 h、mount 即可看到页面上已经出现了红色的 hello
    const vdom = h('div', {class: 'red'}, [
        h('span', null, 'hello')
    ])

    mount(vdom, document.getElementById('app'))
</script>
```

## 6 - Implementing a Patch Function

注意这里是从第5节课的基础上扩展出来的，可以直接从 patch 阶段开始看。

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

        // 将创建的 DOM 元素放到 container 中
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

## 7 - Intro to Reactivity

响应式的能力：某个值发生变化，相应的 值 / 副作用 也发生变化。

依赖追踪（dependency tracking）的基础形式：

```js
onStateChanged(() => {
    console.log(state.count)
})
```

即 state 发生变化，就自动执行相应的语句。

**Vue 3's Reactivity API：**

```js
import { ractive, watchEffect } from 'vue'

// 创建一个响应式对象
const state = reactive({
    count: 0
})

// 会收集所有的依赖，在执行过程，如果某个响应式属性被使用，那么整个函数就会执行
// 相当于上面提到的 onStateChanged
watchEffect(() => {
    console.log(state.count)
}) // 0

state.count++ // 1
```

## 8 - Building Reactivity from Scratch （从头开始构建 Reactivity）

```html
<script>
    // 记录当前的 effect
    let activeEffect;

    // 创建一个 Dep 类
    class Dep {
        // 构造的时候，记录 effect 到 subscribers 集合中
        // 这里直接把值记录在 Dep 中
        constructor(value) {
            this.subscribers = new Set();
            this._value = value;
        }

        // 获取的时候收集依赖
        get value() {
            this.depend();
            return this._value;
        }

        // 赋值的时候通知更新
        set value(newValue) {
            this._value = newValue;
            this.notify();
        }

        // 将当前的 effect 放入 subscribers 集合
        // 使用集合可以避免重复收集依赖
        depend() {
            if (activeEffect) {
                this.subscribers.add(activeEffect);
            }
        }

        // 更新的时候，把集合里的 effect 都执行一遍
        notify() {
            this.subscribers.forEach(effect => {
                effect()
            });
        }
    }

    // 把传入进来的回调函数设置成当前的 effect
    // 然后立马执行一遍回调函数
    // 执行完后清空当前的 effect
    function watchEffect(effect) {
        activeEffect = effect;
        effect();
        activeEffect = null;
    }


    // 接下来是测试的代码
    const ok = new Dep(true);
    const count = new Dep(0);

    watchEffect(() => {
        if (ok.value) {
            console.log(count.value);
        } else {
            console.log('error branch');
        }
    })

    count.value++;
    ok.value = false;
    // 尤大希望执行这一遍后，count 的依赖被删除，
    // 因为 watchEffect 中不会再执行到 count.value，
    // 但接下来改变 count 还是会触发 watchEffect，因为 count 的依赖还存在，
    // 所以触发了 console.log('error branch')
    // 所以 Vue 3 的源码中，每次都会整理依赖
    count.value++;
</script>

```

## 9 - Building the Reactive API

先用 Vue 2 的方法，也就是 Object.defineProperty 来实现：

```html
<script>
    let activeEffect;

    // 有了 reactive，Dep 中无需记录值，值就在原来的对象上
    // Dep 只是单纯地收集依赖，执行 effect
    class Dep {
        subscribers = new Set();

        depend() {
            if (activeEffect) {
                this.subscribers.add(activeEffect);
            }
        }

        notify() {
            this.subscribers.forEach(effect => {
                effect()
            });
        }
    }

    function watchEffect(effect) {
        activeEffect = effect;
        effect();
        activeEffect = null;
    }


    // Vue 2 采用 Object.defineProperty 为对象上的每一个 key 添加响应式
    // 这样做的缺点就是新增的属性无法自动添加响应式
    function reactive(raw) {
        Object.keys(raw).forEach(key => {
            const dep = new Dep();
            let value = raw[key];

            Object.defineProperty(raw, key, {
                get() {
                    // 获取 key 对应的值时添加依赖
                    dep.depend();
                    return value;
                },
                set(newValue) {
                    // 设置 key 对应的值时进行更新
                    value = newValue;
                    dep.notify();
                }
            })
        })

        return raw
    }

    const state = reactive({
        count: 0
    })

    watchEffect(() => {
        console.log(state.count)
    });

    state.count++   
</script>
```

接下来是 Vue 3 的做法，也就是修改 reactive 的实现。

Vue 2 采用 Object.defineProperty，对每一个 key 做响应式处理，而 Vue  3 使用 Proxy，是直接对整个对象本身做响应式处理，这样的好处就是新添加的属性，也可以被自动处理，并且使用 Proxy 可以直接处理数组，而不是像 Vue 2，针对数组特殊处理，修改数组的原型链，这种比较 hack 的方式进行处理。

```html
<script>
    let activeEffect;

    class Dep {
        subscribers = new Set();

        depend() {
            if (activeEffect) {
                this.subscribers.add(activeEffect);
            }
        }

        notify() {
            this.subscribers.forEach(effect => {
                effect()
            });
        }
    }

    function watchEffect(effect) {
        activeEffect = effect;
        effect();
        activeEffect = null;
    }

    // 创建一个存放 deps 的弱引用 Map，key 为 target 本身
    // 即需要响应式处理的对象本身
    // WeakMap 只能用 object 作为 key，并且无法被遍历
    // 当 target 不在需要的时候，可以正确地被垃圾处理机制回收
    const targetMap = new WeakMap();

    function getDep(target, key) {
        // 获取 target 对应的 deps，不存在就创建
        let depsMap = targetMap.get(target);
        if (!depsMap) {
            depsMap = new Map();
            targetMap.set(target, depsMap);
        }
        // 获取 target[key] 对应的 dep，不存在就创建
        let dep = depsMap.get(key);
        if (!dep) {
            dep = new Dep();
            depsMap.set(key, dep);
        }
        return dep
    }

    const reactiveHandlers = {
        // 因为 get 和 set 里都需要获取 dep，故抽成一个获取 dep 的函数
        get(target, key, receiver) {
            const dep = getDep(target, key);
            // 收集依赖
            dep.depend();
            // 使用 Reflect 确保与原始的 get 一致
            return Reflect.get(target, key, receiver);
        },
        set(target, key, value, receiver) {
            const dep = getDep(target, key);
            // Proxy 的 set 需要返回一个值
            const result = Reflect.set(target, key, value, receiver);
            // 通知更新
            dep.notify();
            return result;
        },
    }

    function reactive(raw) {
        // 使用 Proxy，更方便地拦截处理
        return new Proxy(raw, reactiveHandlers);
    }

    const state = reactive({
        count: 0
    })

    watchEffect(() => {
        console.log(state.count)
    });

    state.count++
</script>
```

## 10 - Creating a Mini Vue

将之前 vdom 和 reactivity 的代码都复制过来，并且删除演示用的代码。

你可以在本仓库中（mini-vue.html）查看完整的源代码。

大体结构如下：

```html
<script>
    // vdom

    function h(tag, props, children) {...}

    function mount(vnode, container) {...}

    function patch(n1, n2) {...}

    // reactivity

    let activeEffect;

    class Dep {...}

    function watchEffect(effect) {...}

    const targetMap = new WeakMap();

    function getDep(target, key) {...}

    const reactiveHandlers = {...}

    function reactive(raw) {...}

</script>
```

然后添加：

```html
<div id="app"></div>
<script>
    // vdom
    // reactivity
    
    // App 组件
    const App = {
        data: reactive({
            count: 0
        }),
        render() {
            return h('div', {
                // 这里需要在 mount 中添加事件处理，之前没有实现
                onClick: () => {
                    this.data.count++
                }
            }, String(this.data.count)) // 第三个参数这里只支持 String
        }
    }

    // 挂载 App
    function mountApp(component, container) {
        let isMounted = false
        let prevVdom
        // component 组件中有响应式对象发生变化，便会执行以下函数
        watchEffect(() => {
            if (!isMounted) {
                // 没有挂载，即初始化
                // 记录旧的 vdom
                prevVdom = component.render()
                // 挂载
                mount(prevVdom, container)
                isMounted = true
            } else {
                // 获取新的 vdom
                const newVdom = component.render()
                // patch
                patch(prevVdom, newVdom)
                prevVdom = newVdom
            }
        })
    }

    mountApp(App, document.getElementById('app'))
</script>
```

至此，一个 mini-vue 就完成了。

## 11 - The Composition API

这节课开始更少地记录，更多的可以阅读 Vue 的官方文档。

Composition API = Reactivity API + Lifecycle hooks

- ref
- setup
- effect
- watch
- usexxx

## 12 - Code organization

usexxx，更容易抽离，几乎都是函数式的调用。

export setup

## 13 - Logic Reuse

minix

higher-order component 高阶组件 （React 的做法）

scoped slot / render props

Composition API

## 14 - A Composition API Example

你甚至可以在`usexxx()`中使用 watchEffect

```html
<script src="https://unpkg.com/vue"></script>

<div id="app"></div>

<script>
    const {createApp, ref, watchEffect} = Vue
    
    // 进一步简化在组件中的 use
    function usePost(getId) {
        return useFetch(() => `https://jsonplaceholder.typicode.com/todos/${getId()}`)
    }

    // 抽出 fetch，并且你可以在的 useFetch 中使用 watchEffect 来监听传进来的值的变化
    function useFetch(getUrl) {
        const data = ref(null)
        const error = ref(null)
        const isPending = ref(true)

        watchEffect(() => {
            // reset
            data.value = null
            error.value = null
            isPending.value = true
            // fetch
            fetch(getUrl())
                .then(res => res.json())
                .then(_data => {
                    data.value = _data
                })
                .catch(err => {
                    error.value = err
                })
                .finally(() => {
                    isPending.value = false
                })
        })

        return {
            data,
            error,
            isPending
        }
    }

    const Post = {
        template: `
            <div v-if="isPending">loading</div>
            <div v-else-if="data">{{ data }}</div>
            <div v-else-if="error">Something went Wrong: {{ error.message }}</div>
        `,
        props: ['id'],
        setup(props) {
            // prop.id 被传到了 useFetch 的 watchEffect 中
            // 所以 prop.id 变化，即可重新 fetch
            const {data, error, isPending} = usePost(() => props.id)

            return {
                data,
                error,
                isPending
            }
        }
    }

    const App = {
        components: {Post},
        data() {
            return {
                id: 1
            }
        },
        template: `
            <button @click="id++">change ID</button>
            <Post :id="id"></Post>
        `
    }

    createApp(App).mount('#app')
</script>

```

## 15 - Parting Thoughts

- Vue 3 源码使用 TypeScript，掌握它很有帮助
- vue hooks 和 composition api 的逻辑很类似，可以比较轻松的移植
- render / template
