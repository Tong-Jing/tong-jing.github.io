# 一、Vue 基础

## options 根属性
- el
- template
- data
- components
- methods
- props
- watch: 监视单一复杂类型
- computed: 监视多个

## 指令
- v-if/v-show
- v-if/v-else-if
- v-bind/v-on: bind是 属性赋值。v-on事件绑定
- v-bind/v-model: bind是单向（vue-page）v-model双向

# Virtual DOM

# 二、数据驱动
## render  
vm._render 最终是通过执行 createElement 方法并返回的是 vnode，它是一个虚拟 Node。Vue 2.0 相比 Vue 1.0 最大的升级就是利用了 Virtual DOM。因此在分析 createElement 的实现前，我们先了解一下 Virtual DOM 的概念


# 三、生命周期

- beforeCreate
- created
- beforeMount   在执行 vm._render() 函数渲染 VNode 之前，执行了 beforeMount 钩子函数
- mounted 在执行完 vm._update() 把 VNode patch 到真实 DOM 后，执行 mouted 钩子，对于同步渲染的子组件而言，mounted 钩子函数的执行顺序也是先子后父
- beforeUpdate 的执行时机是在渲染 Watcher 的 before 函数中
- updated: 组件 mount 时，会实例化一个渲染的 Watcher 去监听 vm 上的数据变化重新渲染
- beforeDestroy 钩子函数的执行时机是在 $destroy 函数执行最开始的地方，接着执行了一系列的销毁动作，包括从 parent 的 $children 中删掉自身，删除 watcher，当前渲染的 VNode 执行销毁钩子函数等，执行完毕后再调用 destroy 钩子函数
- destroy  在 $destroy 的执行过程中，它又会执行 vm.__patch__(vm._vnode, null) 触发它子组件的销毁钩子函数，这样一层层的递归调用，所以 destroy 钩子函数执行顺序是先子后父，和 mounted 过程一样
- activated  keep-alive 组件定制的钩
- deactivated

# 响应式原理


