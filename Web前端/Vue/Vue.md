# Vue

## MVVM

<img src="pic\image-20210320110007195.png" alt="image-20210320110007195" style="zoom: 50%;" />

`DataBinding`：`Vue`自动将`data`绑定到`View`，一般使用在`{{}}`（`Mustache`语法）和`指令`中

`DOM Listeners`：利用`View`自身的`listener`绑定修改`data`

> `v-model`：指令实现各种表单的双向绑定



## 组件

> 类似于Android自定义封装控件
>
> <img src="pic\image-20210322132312123.png" alt="image-20210322132312123" style="zoom:50%;" />

### 定义组件

**全局注册**

```html
<template id="cpn">
    <div>
        我是模板抽离写法
    </div>
</template>
//...
<body>
    <div id="app">
        <my-component/>
    </div>
    <script>
     	Vue.component("my-component", {//定义组件并全局注册（任何Vue实例都能使用）
            template: "#cpn"
        })
        new Vue({
            el: "#app"
        })
    </script>
</body>
```

**局部注册**

```html
<template id="cpn">
    <div>
        我是模板抽离写法
    </div>
</template>
//...
<body>
    <div id="app">
        <my-component/>
    </div>
    <script>
        let component = {//定义组件
            template: "#cpn",
        }
         new Vue({
            el: "#app",
            components: {
                'my-component': component//注册组件（仅限当前Vue实例使用）
            }
        })
    </script>
</body>
```



### 组件间传递数据

> 子组件中声明`props`接收父组件传递的数据；子组件通过`$emit(方法名，data)`向父组件回调数据

**综合案例**

```html
<template id="cpn">
    <div>
        <h4>{{title}}：{{counter}}</h4>
        <h5>{{version}}</h5>
        <button @click="decreamtent">-</button>
        <button @click="increament">+</button>
        <button @click="counterListener">alert</button>
    </div>
</template>

<body>
    <div id="app">
        <ry-calculator :ry-counter="num" :ry-title="calculator" @counter-listener="getCounter"></ry-calculator>
    </div>
    
    <script>
        let cpn = {
            data() {
                return {
                    counter: this.ryCounter,//data直接引用属性值
                    title: this.ryTitle.title,
                    version: this.ryTitle.version
                }
            },
            props: {//父传子——定义props（自定义属性）
                ryCounter: {
                    type: Number,
                    default: 20
                },
                ryTitle: {
                    tyle: Object
                }
            },
            watch: {
                counter: function () {
                    this.counterListener()
                }
            },
            methods: {
                increament() {
                    this.counter++
                },
                decreamtent() {
                    this.counter--
                },
                counterListener() {
                     this.$emit("counter-listener",this.counter)//子传父——回调函数 
                }
            },
            template: "#cpn"
        }
        new Vue({
            el: "#app",
            data: {
                num: 100,
                calculator: {
                    title: "我是计数器",
                    version: "v1.0"
                }
            },
            components: {
                ryCalculator: cpn
            },
            methods: {
                getCounter(counter){
                    alert(counter)
                }
            },
        })
    </script>
</body>
```

`tip`:`html`由于对大小写不过敏，所以标签和属性`驼峰`写法无效，但是`Vue`会将`script`中驼峰写法的属性转换成 `-`

# Vue-Router

> 适用于构建单页面应用，基于路由和组件的。类似于Android的`FragmentManager`根据`url`切换组件

**路由懒加载**

> 将组件不打包到dist/app.js中，独立分包，实现需要的时候再加载js

写法

```javascript
// import About from '../components/HelloWorld.vue'
const About = () => import('../components/HelloWorld.vue') //懒加载
```















