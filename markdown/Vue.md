<center><h1>Vue</h1></center>
:class表示的是绑定class元素：{active:isActive, line:isLine}为一个对象，其中格式为 {(属性)active ：boolean}当boolean值为false的时候，active元素不起作用。当为true的时候，那么那个style下面的样式就起作用

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<style>
    .active{
        color: #93ffe0;
    }
</style>
<body>

<div id="app">
    <!--原本的class可以和后面的class合并，不会覆盖什么的-->
    <h1 class=“hello” :class="{active:isActive, line:isLine}">{{message}}</h1>
    <!--加入事件的绑定-->
    <button v-on:click="btnReverse">reverse</button>
</div>
<script src="js/vue.js"></script>
<script>
    const  app = new Vue({
        el: "#app",
        data:{
            message: "hello vue",
            isActive: true,
            isLine: true
        },
        methods: {
            btnReverse: function () {
                this.isActive = !this.isActive
            }
        }
    })
</script>
</body>
</html>
```



es6转成es5

使用babe



webpack使用vue



没写路径的时候，会从normal中找



npm install  vue --save  // 运行时也依赖

```html
 You are using the runtime-only build of Vue where the template compiler is not available. Either pre-compile the templates into render functions, or use the compiler-included build.  有bug，需要增加配置：
```



然后在webpack.config.js中加入别名：

```json
const path = require('path')
module.exports = {
    entry: path.join(__dirname,'src/main.js'),
    output: {
        path: path.resolve(__dirname,'dist'),
        filename: 'bundle.js',
        publicPath: 'dist/'
    },
    module: {
        rules: [
            {
                test: /\.css$/i,
                use: ['style-loader', 'css-loader'],
            },
            {
                test: /\.(png|jpg|gif)$/i,
                use: [
                    {
                        loader: 'url-loader',
                        options: {
                            limit: 81,
                            name: 'image/[name]-[hash:8].[ext]'
                        },
                    },
                ],
            },
        ],
    },
    resolve: {
        alias: {
            "vue$": 'vue/dist/vue.esm.js'
        }
    }
}
```



const app = new Vue()

中app可以不要的

spa

多个页面： 会用到vue-router



vue实例要抽离template



简单使用:

```js
const app = new Vue({
    el: '#app',
    template: `
      <h1>最美的不是下雨天：{{message}}</h1>
    `,
    data: {
        message: 'jeffchan'
    }
})
```



index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>index</title>
</head>
<body>
<div id="app">
   <!--这里的代码一般不改-->
</div>
<script src="../dist/bundle.js"></script>
</body>
</html>
```



抽离代码：



创建一个目录，放vue的代码：app.js

```json
export default {
    template: `
      <h1>最美的不是下雨天：{{message}}</h1>
    `,
    data() {
        return {
            message: 'jeffchan'
        }
    }
}
```



然后在main.js中引入即可：

```js
import {name} from "./js/constants";

console.log(name);

require('./css/back.css')

import Vue from 'vue';

// 这个名字可以随便取，因为是default的export
// 相当于是一个组件
import App1 from './vue/app'

const app = new Vue({
    el: '#app',
    template: '<App1/>',
    components: {
        App1
    }
})
```



使用vue文件的方式：

要安装vue-loader 和 vue-template-compiler

```
"vue-loader": "^15.9.3",//安装大于等于该版本
```

在webpack.config.js中的resolve中加入:

```k
extensions: ['.vue'],
```

到时就可以简写 app.vue -》 app



plugin可以加协议说明

可以自动生成html,打包到dist文件中

对js压缩：

uglifyjs插件



每次都要编译，很麻烦

搭建本地服务器： node

express框架：将修改放入内存，用于测试

webpack-dev-server --open



在开发：

有些配置是打包要的，

有些开发用的



配置文件分离

合并配置文件：

webpack-merge



webpack --config ./build/prod.config.js

指定配置文件启动

默认使用webpack.config.js的话，会自动找



vue-cli可以生成webpack配置

安装脚手架：以来nodejs和webpack

npm install -g @vue/cli



node_module： webpack等

pubilc 原封不动复制到dist

src源码

babel.config.js基本不改，用于配置babel



箭头函数

const aa = function(){}



es6 定义函数

const bb =  () =>{}

一个参数，括号可以省略掉



前端渲染 和 后端渲染



该表url,页面不刷新



location.hash = 'aaa'



// 可以返回

history.pushState({},'','home')

history.back()



history.go(-1) 往后退一步

go(1) 进一步



npm install vue-router --save

<router-link>// 设置路由, 

tag: 默认渲染成a标签

tag=button会渲染成按钮，

加replace就不可以返回

加active-class='aaa',加上aaa class

<router-view>用于渲染组件

通过this.$router.push('/home')来

映射路由，也可以使用.replace('/home')





默认选择首页

mode:history







