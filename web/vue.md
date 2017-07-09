## 项目初始化
### Vue-cli
Vue.js 提供一个[官方命令行工具](https://github.com/vuejs/vue-cli)，可用于快速搭建大型单页应用。该工具提供开箱即用的构建工具配置，带来现代化的前端开发流程。只需几分钟即可创建并启动一个带热重载、保存时静态检查以及可用于生产环境的构建配置的项目：   
```shell
# 全局安装 vue-cli
$ npm install --global vue-cli
# 创建一个基于 webpack 模板的新项目
$ vue init webpack my-project
# 安装依赖，走你
$ cd my-project
$ npm install
$ npm run dev
```

## 实例
```javascript
var vm = new Vue({
    // el -> HTML中的标签
    el: '#app',
    // 模版
    template: "<p>This is a template {{ message }}</p>"
    // 数据
    data: {
        message: 'hello veu',
    }
})
```

## 组件
### 全局组件  
```js
Vue.component('my-header', {
    template: '<p> This is header </p>'
})
```
使用：
```html
<div id="app">
    <my-header></my-header>
</div>
```

### 局部组件 
```js
var Child = {
  template: '<div>A custom component!</div>'
}
new Vue({
  // ...
  components: {
    // <my-component> 将只在父模板可用
    'my-component': Child
  }
})
```

## 全局API