> Vue.js 最核心的功能就是组件（Component），从组件的构建、注册到组件间通信，Vue 2.x 提供了更多方式，让我们更灵活地使用组件来实现不同需求。

# 一、构建组件
## 1.1 组件基础
一个组件由 template、data、computed、methods等选项组成。需要注意：

- template 的 DOM 结构必须有根元素
- data 必须是函数，数据通过 return 返回出去

```
// 示例：定义一个组件 MyComponent
var MyComponent = {{
  data: function () {
    return {
      // 数据
    }
  },
  template: '<div>组件内容</div>'
}
```

由于 HTML 特性不区分大小写， 在使用`kebab-case`(小写短横线分隔命名) 定义组件时，引用也需要使用这个格式如 `<my-component>`来使用；在使用`PascalCase`(驼峰式命名) 定义组件时`<my-component>`和`<MyComponent>`这两种格式都可以引用。
 
## 1.2 单文件组件.vue 
如果项目中使用打包编译工具 webpack，那引入 vue-loader 就可以使用 `.vue`后缀文件构建组件。
一个`.vue`单文件组件 (SFC) 示例：

```
// MyComponent.vue 文件
<template>
    <div>组件内容</div>
</template>

<script>
  export default {
      data () {
        return {
          // 数据
        }
      }
  }
</script>

<style scoped>
    div{
        color: red
    }
</style>
```
`.vue`文件使组件结构变得清晰，使用`.vue`还需要安装 vue-style-loader 等加载器并配置 webpack.config.js 来支持对 .vue 文件及 ES6 语法的解析。 
> 进一步学习可参考文章：[详解 SFC 与 vue-loader][1]

# 二、注册组件
## 2.1 手动注册
组件定义完后，还需要注册才可以使用，注册分为全局和局部注册：

```
// 全局注册，任何 Vue 实例都可引用
Vue.component('my-component', MyComponent)

// 局部注册，在注册实例的作用域下有效
var MyComponent = { /* ... */ }
new Vue({
    components: {
        'my-component': MyComponent
    }
})

// 局部注册，使用模块系统，组件定义在统一文件夹中
import MyComponent from './MyComponent.vue'

export default {
  components: {
    MyComponent // ES6 语法，相当于 MyComponent: MyComponent
  }
}
```
注意全局注册的行为必须在根 Vue 实例 (通过 new Vue) 创建之前发生。
## 2.2 自动注册
对于通用模块使用枚举的注册方式代码会非常不方便，推荐使用自动化的全局注册。如果项目使用 webpack，就可以使用其中的`require.context`一次性引入组件文件夹下所有的组件：

```
import Vue from 'vue'
import upperFirst from 'lodash/upperFirst' // 使用 lodash 进行字符串处理
import camelCase from 'lodash/camelCase'

const requireComponent = require.context(
  './components',   // 其组件目录的相对路径
  false,   // 是否查询其子目录
  /Base[A-Z]\w+\.(vue|js)$/   // 匹配基础组件文件名的正则表达式
)

requireComponent.keys().forEach(fileName => {
  // 获取组件配置
  const componentConfig = requireComponent(fileName)

  // 获取组件的 PascalCase 命名
  const componentName = upperFirst(
    camelCase(
      // 剥去文件名开头的 `./` 和结尾的扩展名
      fileName.replace(/^\.\/(.*)\.\w+$/, '$1')
    )
  )

  // 全局注册组件
  Vue.component(
    componentName,
    componentConfig.default || componentConfig
  )
})
```


# 三、组件通信
## 3.1 父单向子的 props
Vue 2.x 以后父组件用`props`向子组件传递数据，这种传递是单向/正向的，反之不能。这种设计是为了避免子组件无意间修改父组件的状态。

子组件需要选项`props`声明从父组件接收的数据，`props`可以是**字符串数组**和**对象**，一个 .vue 单文件组件示例如下

```vue
// ChildComponent.vue
<template>
    <div>
      <b>子组件：</b>{{message}}
    </div>
</template>

<script>
  export default {
    name: "ChildComponent",
    props: ['message']
  }
</script>
```

父组件可直接传单个数据值，也可以可以使用指令`v-bind`动态绑定数据：
```vue
// parentComponent.vue
<template>
    <div>
      <h1>父组件</h1>
      <ChildComponent message="父组件向子组件传递的非动态值"></ChildComponent>
      <input type="text" v-model="parentMassage"/>
      <ChildComponent :message="parentMassage"></ChildComponent>
    </div>
</template>

<script>
  import ChildComponent from '@/components/ChildComponent'
  export default {
    components: {
      ChildComponent
    },
    data () {
      return {
        parentMassage: ''
      }
    }
  }
</script>
```

配置路由后运行效果如下：
![clipboard.png](/img/bVbgZqB)

## 3.2 子向父的 $emit
当子组件向父组件传递数据时，就要用到**自定义事件**。子组件中使用 `$emit()`触发自定义事件，父组件使用`$on()`监听，类似观察者模式。

子组件`$emit()`使用示例如下：

```vue
// ChildComponent.vue
<template>
  <div>
    <b>子组件：</b><button @click="handleIncrease">传递数值给父组件</button>
  </div>
</template>

<script>
  export default {
    name: "ChildComponent",
    methods: {
      handleIncrease () {
        this.$emit('increase',5)
      }
    }
  }
</script>
```
父组件监听自定义事件 increase，并做出响应的示例：

```vue
// parentComponent.vue
<template>
    <div>
      <h1>父组件</h1>
      <p>数值：{{total}}</p>
      <ChildComponent @increase="getTotal"></ChildComponent>
    </div>
</template>

<script>
  import ChildComponent from '@/components/ChildComponent'
  export default {
    components: {
      ChildComponent
    },
    data () {
      return {
        total: 0
      }
    },
    methods: {
      getTotal (count) {
        this.total = count
      }
    }
  }
</script>
```
访问 parentComponent.vue 页面，点击按钮后子组件将数值传递给父组件：
![clipboard.png](/img/bVbgZDF)


## 3.3 子孙的链与索引
组件的关系有很多时跨级的，这些组件的调用形成多个父链与子链。父组件可以通过`this.$children`访问它所有的子组件，可无限递归向下访问至最内层的组件，同理子组件可以通过`this.$parent`访问父组件，可无限递归向上访问直到根实例。

以下是子组件通过父链传值的部分示例代码：

```vue
// parentComponent.vue
<template>
    <div>
      <p>{{message}}</p>
      <ChildComponent></ChildComponent>
    </div>
</template>


// ChildComponent.vue
<template>
  <div>
    <b>子组件：</b><button @click="handleChange">通过父链直接修改数据</button>
  </div>
</template>

<script>
  export default {
    name: "ChildComponent",
    methods: {
      handleChange () {
        this.$parent.message = '来自 ChildComponent 的内容'
      }
    }
  }
</script>
```
显然点击父组件页面的按钮后会收到子组件传过来的 message。
> 在业务中应尽量避免使用父链或子链，因为这种数据依赖会使父子组件紧耦合，一个组件可能被其他组件任意修改显然是不好的，所以组件父子通信常用`props`和`$emit`。

## 3.4 中央事件总线 Bus
子孙的链式通信显然会使得组件紧耦合，同时兄弟组件间的通信该如何实现呢？这里介绍中央事件总线的方式，实际上就是**用一个vue实例（Bus）作为媒介**，需要通信的组件都引入 Bus，之后通过分别触发和监听 Bus 事件，进而实现组件之间的通信和参数传递。


首先建 Vue 实例作为总线：
```
// Bus.js
import Vue from 'vue'
export default new Vue;
```
需要通信的组件都引入 Bus.js，使用 `$emit`发送信息： 

```vue
// ComponentA.vue
<template>
  <div>
    <b>组件A：</b><button @click="handleBus">传递数值给需要的组件</button>
  </div>
</template>

<script>
  import Bus from './bus.js' 
  export default {
    methods: {
      handleBus () {
        Bus.$emit('someBusMessage','来自ComponentA的数据')
      }
    }
  }
</script>
```  
需要组件A信息的就使用`$on`监听：

```
// ComponentB.vue
<template>
  <div>
    <b>组件B：</b><button @click="handleBus">接收组件A的信息</button>
    <p>{{message}}</p>
  </div>
</template>

<script>
  import Bus from './bus.js' 
  export default {
    data() {
      return {
        message: ''
      }
    },
    created () {
      let that = this // 保存当前对象的作用域this
      Bus.$on('someBusMessage',function (data) {
        that.message = data
      })
    },
    beforeDestroy () {
      // 手动销毁 $on 事件，防止多次触发
      Bus.$off('someBusMessage', this.someBusMessage)
    }
  }
</script>
```


# 四、递归组件
组件可以在自己的 template 模板中调用自己，需要设置 name 选择。

```vue
// 递归组件 ComponentRecursion.vue
<template>
  <div>
    <p>递归组件</p>
    <ComponentRecursion :count="count + 1" v-if="count < 3"></ComponentRecursion>
  </div>
</template>

<script>
  export default {
    name: "ComponentRecursion",
    props: {
      count: {
        type: Number,
        default: 1
      }
    }
  }
</script>
```
如果递归组件没有 count 等限制数量，就会抛出错误（Uncaught RangeError: Maximum call stack size exceeded）。

父页面使用该递归组件，在 Chrome 中的 **Vue Devtools** 可以看到组件递归了三次：
![clipboard.png](/img/bVbg0TJ)

> 递归组件可以开发未知层级关系的独立组件，如级联选择器和树形控件等。


# 五、动态组件
如果将一个 Vue 组件命名为 Component 会报错（Do not use built-in or reserved HTML elements as component id: Component），因为 Vue 提供了特殊的元素 `<component>`来动态挂载不同的组件，并使用 `is` 特性来选择要挂载的组件。

以下是使用`<component>`动态挂载不同组件的示例：

```vue
// parentComponent.vue
<template>
 <div>
    <h1>父组件</h1>
    <component :is="currentView"></component>
    <button @click = "changeToViewB">切换到B视图</button>
 </div>
</template>

<script>
  import ComponentA from '@/components/ComponentA'
  import ComponentB from '@/components/ComponentB'
  export default {
   components: {
      ComponentA,
      ComponentB
    },
   data() {
      return {
        currentView: ComponentA // 默认显示组件 A
      }
    },
    methods: {
      changeToViewB () {
        this.currentView = ComponentB // 切换到组件 B
      }
    }
  }
</script>
```
改变 `this.currentView`的值就可以自由切换 AB 组件：
![clipboard.png](/img/bVbg0WS)

> 与之类似的是`vue-router`的实现原理，前端路由到不同的页面实际上就是加载不同的组件。


----------
#### 要继续加油呢，少年！

  [1]: https://segmentfault.com/a/1190000015708749