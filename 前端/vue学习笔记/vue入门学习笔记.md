vue入门学习笔记
=================

Vue.js前段开发快速入门与专业应用

### vue-cli

安装
```
    npm install vue-cli -g
```

创建项目

```
    vue init webpack myproject
```

### vue实例创建

通过构造函数Vue({option})创建vue实例。vue实例相当于是ViewModel，通过与视图View的数据绑定实现展示和展示逻辑的实现，同时从模型Model层获取数据和业务逻辑实现。

option可以包含数据、模板、挂载元素、方法、生命周期钩子等选项。

###### 模板
模板通过el：#app设置,一般使用ID选择器，为实例提供挂载元素，在HTML中ID设置app的元素即为与vue实例绑定的视图View。


###### 数据
数据通过data：{}设置，这些数据会在模板中进行绑定并使用。通过vm.$data即可获取数据。

数据在模板中通过{{name}}的方式使用，且这是数据是响应式的。

特别的，之后Vue初始化是传入的数据才是响应式的，因此在初始化时应该讲所有预期会用到的数据都做设定，无初始值是可以设置undifined或null。

###### 方法
方法通过methods：{}设置，注册的方法在HTML中通过 v-on:click="name" 即可使用。

###### 生命周期
生命周期函数通过created：function(){}设置，我们可以通过这些定义好的生命周期钩子来做业务逻辑。

###### 计算属性
计算属性通过computed:{}来设置，计算属性是在data中的数据需要做一些变换之后再进行使用时，很好的一种方法。

```
var vm = new Vue({
    data : {
        city : 杭州,
        cents : 100
    }
    computed : {
        fullCity : function(){
            return this.city+'市'
        }，
        price : {
            set : function(newValue){
                this.cents = newValue*100
            },
            get : function(){
                return this.cents/100
            }
        }
    }
})
```

通过{{funCity}}即可获取到杭州市。而price则与cents做了关系绑定，能够响应对方的变化。


### 数据绑定语法
Vue.js的核心是一个响应式的数据绑定关系，简历绑定后，DOM将于数据保持同步，无需手动维护DOM。
例如Vue如下定义

```
var vm = new Vue({
    el : #app,
    data : {
        id : 1,
        index : 0,
        name : ‘vue’,
        avatar : 'http://......',
        count : [1,2,3,4,5],
        names : ['Vue1.0','Vue2.0']
    }    
})
```

###### 文本插值

在文本中直接通过{{name}}来只获取，例如

```
<span>Hello {{name}}</span> //Hello Vue.
```

###### HTML属性
直接在HTML标签里显示

```
<div v-bind:id="'my id is '+id"></div> //<div id="my id is 1"></div>
```

###### 绑定表达式
既可以通过表达式来处理数据，当只支持当个表达式，且不支持JS语句。
```
{{index+1}}
{{index == 0?'a':'b'}}
{{name.split('').join('|')}}
```

###### 过滤器
以管道符‘|’来使用过滤器，过滤器本质上是一个函数，可以自定义。

```
{{name | filterA | filterB}}
```

### 指令

Vue中的指令都带有v-前缀。

###### v-bind

bind指令，为元素的属性绑定数据，可以是任何属性，包括style、class。
```
<img v-bind:src="avatar"/> //img的src和数据vm.avatar绑定。
```

###### v-model
为表单元素进行双向数据绑定。

可以在Text、Radio、Checkbox、Select等表单元素使用。

```
<input type="text" v-model="message" />//将输入内容与vm.message的数据进行双向绑定
```


###### 条件渲染指令v-if/v-else

根据数据值来判断是否输出该DOM元素。其中v-else必须紧跟v-if使用才会生效。
```
<div v-if="flag == '1'">Yes</div> //当vm.flag='1'时输出YES
<div v-else-if="flag == '2'">No</div> //当vm.flag='2'时输出NO
<div v-else>{{flag}}</div>//否则输出flag的值
```

###### 条件渲染指令v-show

控制切换元素的CSS属性Display，无论条件是true还是false，都会进行渲染进，只是是否显示。
```
<div v-show="show">show</div>
```

###### 列表渲染v-for
根据接受到的数组，重复渲染DOM元素及其内部子元素。v-for内置了索引变量$index,可以在循环内使用。

```
<ul>
    <li v-for="(item,index) in items">
        <h3>{{index}}</h3>
        <h3>{{item.title}}</h3>
    </li>
</ul>
```


*我们可以将指令直接作用到template标签上，这样template标签内的所有元素都将受到该指令的影响，但渲染结果是不会有template标签的。*

###### 事件监听指令v-on
为对应的DOM元素绑定事件监听器，这个监听器可以在Vue实例的选项属性methods中定义，v-on:后参数接受所有的原生事件名称。如果需要在函数中获取原生DOM事件，可以用$event传入。多次绑定同一事件时，会按顺序执行。
```
<button v-on:click="sayFrom($event,'from param')">say</button>//其中sayFrom函数在选项属性methods中定义
```

v-on可以使用修饰符：

.stop:等同于event.stopProgagation();
.prevent:等同于event.preventDefault();
.capture:使用capture模式添加事件监听器;
.self:只当是一家是从监听元素本身触发时才触发回调;

```
<form v-on:submit.stop.prevent="onSubmit"></from>//阻止默认提交事件且阻止冒泡
```


按键修饰符：enter、tab、delete、esc、space、up、down、left、right。也可用通按键的keycode值使用。
```
<form v-on:submit="onSubmit" v-on:keyup.enter="submit"></form>
```

###### v-text
更新元素的textContext，直接使用{{msg}}的话，在beforeCompile期间msg值为编译到{{msg}}中，用户能看到一瞬间的“{{msg}}”显示。
```
<span v-text="msg"></span>
<span>{{msg}}</span>
```

###### v-HTML
更新元素的innerHTML,与v-text一样能避免闪现
```
<div v-HTML="HTML"></div>
<div>{{HTML}}</div>
```

###### v-el
为DOM元素注册一个索引，可以通过vue实例的$els属性调用
```
<div v-el:demo>there is a el demo</div>
vm.$els.demo.innerText // -> there is a el demo
```

###### v-ref

###### v-pre

### 组件

###### 组件创建
```
var MyComponent = Vue.extend({
    template : '<p>this is a component</p>'
    })
```
通过Vue.component进行全局注册，需要在创建Vue实例之前注册。
```
Vue.component('my-component',MyComponent);//我们可以使用<my-component>方式在HTML中使用该组件
```
也可以局部注册，局部注册只需要在使用的组件创建时，通过components属性进行设置。
```
Var ParentComponent = Vue.extend({
    template : '<div>
            <p>this is a parent component</p>
            <my-component/>
        </div>',
    components : MyComponent,
    })
```

###### 组件选项

组件选项的data属性，需用通过方法放回一个新的对象。组件是可以服用的，如果直接设置，会导致多个组件共用一个data对象实例，el属性也如此。

###### 组件的Props
子组件无法直接调用父组件的数据，父组件需要通过Props将数据传递给子组件。

可以在组件定义的时候声明需要的数据，在使用时进行设置，可以设置静态数据，也可以设置父组件的data数据。
```
Vue MyComponent = Vue.extend({
    props : ['message']
    })

<div id="app">
    <my-component v-bind:messge="message"></my-component>
</div>

Vue.component('my-component',MyComponent);

var vm = new Vue({
    el : '#app'，
    data : {
        message : 'default'
    }
    })
```

默认数据绑定是单向的，通过.sync可以实现双向绑定，通过.once可以实现单次绑定。

###### 组件通信

$parent: 父组件实例。

$children: 包含所有子组件实例。

$root: 组件所在的根实例。

###### 组件义事件传递
初始化实例或注册子组件的是后可通过events进行自定义事件监听。

也可以通过$on方法，在特定情况或方法内进行监听事件。

通过$emit可以在该组件自身上触发事件.
```
var vm = new Vue({
    el : '#app'，
    data : {
        todo : []
    }，
    events :{
        'add' : function(msg){
            this.todo.push(msg);
            //return true; 放回true时，事件会继续向上冒泡。
        }
    }，
    methods : {
        begin : function(){
            this.$on('add',function(msg){
                    this.todo.push(msg);
                })
        }，
        onClick :function(){
            this.$emit('add','there is a message');
        }
    }
    })
```

子组件如果要触发父组件的事件，可以通过$dispatch进行派发。事件会沿着父链冒泡传递，触发回调是会听着，若触发函数明确返回true会继续冒泡

子组件：
```
methods : {
    toParent : function(){
        this.$disptach('add','message from child!')
    }
}
```

父组件如果需要将事件传递给所有子组件，可以通过$broadcast进行广播。

```
methods : {
    toChild : function(){
        this.$broadcast('add',"message from parent!")
    }
}
```

### Vue-router
###### router-link
使用router-link进行页面跳转定义，默认会被渲染成a标签。

默认使用go方式跳转，如果不需要留下历史页面（回退可以回去），可以设置replace为true，使用replace方式跳转。

特别注意的是to中不要设置为“./home”，否则触发都是在当前的路径下加上一级home

```
<router-link to="/home">Home</router-link>
```

###### router-view
使用router-view定义匹配到的组件template（模板）插入的位置。
```
<div id="app">
    <img src="./assets/logo.png">
    <router-view/>
</div>
```

###### 路由定义
导入VueRouter，创建实例后通过routes设置所有页面,

二级导航的情况下可以通过设置children来实现嵌套路由,

需要把某种模式匹配到的所有路由，全都映射到同个组件时，使用动态路由，通过':'来标记动态路径参数，通过this.$route.params来获取动态参数。
```
import Vue from 'vue';
import VueRouter from 'vue-router';

Vue.use(VueRouter);

var router = new VueRouter({routes: [
    //基本实现
  {
    path: '/',
    name: 'HelloWorld',
    component: HelloWorld
  },
    //嵌套路由实现
  {
    path: '/user',
    name: 'User',
    component: User,
    children:[
      {
        path: 'info',  //特别注意不应写成‘/info’
        component: Info,
      },
      {
        path: 'account',
        component: Account,
      },
    ]
  },
    //动态路由实现
  {
    path: '/product/:id',
    name: 'Product',
    component: Product
  }
]
})
```


###### 路由对象

通过this.$route的方式调用路由对象

1. $route.path:路由的绝对路径。
2. $route.params:包含路由中动态片段和全匹配片段的键值对，通过$route.params.id的方式获取。
3. $route.query:查询参数键值对。
4. $route.router:路由实例，组件中可以通过$route.router来调用go,replace方法来进行跳转。
5. $route.matched:包含当前匹配的路径中所有片段对应的配置参数对象，包括2，3。
6. $route.name:路由设置的name属性。


###vue-resource

###### 全局配置

```
Vue.http.options.root = '/root';
Vue.http.headers.common['Authorization'] = 'Basic YXBpOnBhc3N3b3Jk';
```

###### 组件实例配置
在初始化Vue组件实例的时候再options里配置。
```
new Vue({
    http:{
        root:'/root',
        headers:{
            token:‘xxxxx’
        }
    })
```

###### 方法调用时配置

```
this.$http
        .get('http://api.douban.com/v2/movie/search', 
        {'Access-Control-Allow-Origin':true,params:{tag:'喜剧'},
        headers: {responseType: 'application/json'}
        })
        .then(function(resp){
          console.log(resp.data)
        },function(resp){
          console.log('error')
        })
```


###### 请求跨域问题

Vue采用的是前后端分离的方案，开发时本机前段页面会直接调用后端服务，接口就存在跨域问题。

解决这个问题，需要在本机开启一个代理服务器只需要在 config/index.js文件内做如下设置

```
module.exports = {
  dev: {
    proxyTable: {
      '/api':{
        target:'http://api.douban.com/',
        changeOrigin:true,
        pathRewrite:{
          '^/api':''
        }
      }
    },
}
```


### 状态管理 Vuex

