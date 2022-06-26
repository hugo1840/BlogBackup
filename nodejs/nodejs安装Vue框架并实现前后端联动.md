@[TOC](nodejs安装Vue框架并实现前后端联动)

# 安装Vue框架
全局安装Vue框架：
```bash
> npm i vue -g
```

安装vue/cli命令行工具：
```bash
> npm install -g @vue/cli
```
查看版本：
```bash
> vue -V
@vue/cli 5.0.6
```

创建client：
```bash
> vue create client
```

选择手动选择特性：
```bash
Vue CLI v5.0.6
? Please pick a preset:
  Default ([Vue 3] babel, eslint)
  Default ([Vue 2] babel, eslint)
> Manually select features
```

使用空格键选中安装Babel/Router/Vuex：
```bash
? Check the features needed for your project: (Press <space> to select, <a> to toggle all, <i> to invert selection, and <enter> to proceed)
>(*) Babel
 ( ) TypeScript
 ( ) Progressive Web App (PWA) Support
 (*) Router
 (*) Vuex
 ( ) CSS Pre-processors

? Choose a version of Vue.js that you want to start the project with (Use arrow keys)
> 3.x
  2.x
  
? Use history mode for router? (Requires proper server setup for index fallback in production) Yes
? Where do you prefer placing config for Babel, ESLint, etc.? In package.json
? Save this as a preset for future projects? (y/N) n
```

回车确认开始安装：
```
✨  Creating project in C:\Users\Xiaoming\source\repos\node_demo\node_app\client.
⚙️  Installing CLI plugins. This might take a while...
🎉  Successfully created project client.
```

启动前端：
```
> cd client
> npm run serve
App running at:
  - Local:   http://localhost:8080/
```
在浏览器中可以打开该页面。

在另一个终端中启动后端：
```
> nodemon
```
在浏览器中打开`http:://localhost:5000/api/users/test`。

Ctrl+C分别关闭前后端。


# concurrently前后端联动
使用concurrently中间件可以同时运行前端和后台服务。

安装concurrently：
```bash
> npm install -g concurrently
```

修改`node_app\client\src\package.json`添加`npm start`命令：
```json
{
  "name": "client",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "serve": "vue-cli-service serve",
    "build": "vue-cli-service build",
    "start": "npm run serve"
  },
  "dependencies": {
    "core-js": "^3.8.3",
    "vue": "^3.2.13",
    "vue-router": "^4.0.3",
    "vuex": "^4.0.0"
  },
  "devDependencies": {
    "@vue/cli-plugin-babel": "~5.0.0",
    "@vue/cli-plugin-router": "~5.0.0",
    "@vue/cli-plugin-vuex": "~5.0.0",
    "@vue/cli-service": "~5.0.0"
  },
  "browserslist": [
    "> 1%",
    "last 2 versions",
    "not dead",
    "not ie 11"
  ]
}
```

修改`node_app\package.json`添加前后端连载命令`npm run dev`：
```json
{
  "name": "node_app",
  "version": "1.0.0",
  "description": "restful api",
  "main": "server.js",
  "scripts": {
    "client-install": "npm install --prefix client",
    "client": "npm start --prefix client",
    "server": "nodemon server.js",
    "start": "node server.js",
    "dev": "concurrently \"npm run server\" \"npm run client\""
  },
  "author": "xiaoming",
  "license": "ISC",
  "dependencies": {
    "bcrypt": "^5.0.1",
    "body-parser": "^1.20.0",
    "express": "^4.18.1",
    "gravatar": "^1.8.2",
    "jsonwebtoken": "^8.5.1",
    "mongoose": "^6.3.6",
    "passport": "^0.6.0",
    "passport-jwt": "^4.0.0"
  }
}
```

在终端执行：
```bash
> cd C:\Users\Xiaoming\source\repos\node_demo\node_app
> npm run dev
```

后台会同时执行：
```bash
> npm run server  #启动后端（nodemon server.js）
> npm run client  #启动前端（npm start --prefix client）
```


# 创建index.vue页面
删除Vue自带的以下内容：
```
client/src/assets/logo.png
client/src/components/HelloWorld.vue
client/src/views/AboutView.vue
client/src/views/HomeView.vue
```

新建`client/src/views/index.vue`前端页面：
```html
<template>
    <div class="index">
        Blank Space by Taylor Swift
     </div>
</template>

<scripts>
    export default {
        name: "index",
        components: {}
    };
</scripts>
```

编辑`client/src/router/index.js`路由文件：
```javascript
import { createRouter, createWebHistory } from 'vue-router'
import Index from '../views/index.vue'


const routes = [
  {
    path: '/',
    redirect: '/index'
    },
    {
        path: '/index',
        name: 'index',
        component: Index
    }
]

const router = createRouter({
  history: createWebHistory(process.env.BASE_URL),
  routes
})

export default router
```

编辑`client/src/App.vue`文件，删除下面的内容：
```html
<nav>
    <router-link to="/">Home</router-link> |
    <router-link to="/about">About</router-link>
</nav>
```

只留下：
```html
<template>
  
  <router-view/>
</template>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}

nav {
  padding: 30px;
}

nav a {
  font-weight: bold;
  color: #2c3e50;
}

nav a.router-link-exact-active {
  color: #42b983;
}
</style>
```

在浏览器中打开`http://localhost:8080`会自动给跳转到`http://localhost:8080/index`，页面显示：
```
Blank Space by Taylor Swift
```

编辑`client/src/App.vue`删除原有样式：
```html
<template>
  
  <router-view/>
</template>

<style>
    html,
    body,
#app {
  width: 100%;
  height: 100%
}

</style>
```

# 创建css样式文件
编辑`client/public/index.html`添加css文件路径：
```html
<!DOCTYPE html>
<html lang="">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width,initial-scale=1.0">
    <link rel="icon" href="<%= BASE_URL %>favicon.ico">
    <link rel="stylesheet" href="css/reset.css"> 
    <title><%= htmlWebpackPlugin.options.title %></title>
  </head>
  <body>
    <noscript>
      <strong>We're sorry but <%= htmlWebpackPlugin.options.title %> doesn't work properly without JavaScript enabled. Please enable it to continue.</strong>
    </noscript>
    <div id="app"></div>
    <!-- built files will be auto injected -->
  </body>
</html>
```

新建`client/public/css/reset.css`样式文件：

```css
/* http://meyerweb.com/eric/tools/css/reset/
   v2.0 | 20110126
   License: none (public domain)
*/

html, body, div, span, applet, object, iframe,
h1, h2, h3, h4, h5, h6, p, blockquote, pre,
a, abbr, acronym, address, big, cite, code,
del, dfn, em, img, ins, kbd, q, s, samp,
small, strike, strong, sub, sup, tt, var,
b, u, i, center,
dl, dt, dd, ol, ul, li,
fieldset, form, label, legend,
table, caption, tbody, tfoot, thead, tr, th, td,
article, aside, canvas, details, embed,
figure, figcaption, footer, header, hgroup,
menu, nav, output, ruby, section, summary,
time, mark, audio, video {
	margin: 0;
	padding: 0;
	border: 0;
	font-size: 100%;
	font: inherit;
	vertical-align: baseline;
}
/* HTML5 display-role reset for older browsers */
article, aside, details, figcaption, figure,
footer, header, hgroup, menu, nav, section {
	display: block;
}
body {
	line-height: 1;
}
ol, ul {
	list-style: none;
}
blockquote, q {
	quotes: none;
}
blockquote:before, blockquote:after,
q:before, q:after {
	content: '';
	content: none;
}
table {
	border-collapse: collapse;
	border-spacing: 0;
}

.el-loading{
  position: absolute;
  z-index: 2000;
  background-color: rgba(255,255,255,.7);
  margin: 0;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  -webkit-transition: opacity .3s;
  transition: opacity .3s;
}
.el-loading-spinner{
  top: 50%;
  margin-top: -21px;
  width: 100%;
  text-align: center;
  position: absolute;
}
```

上面的样式文件来自`http://meyerweb.com/eric/tools/css/reset/`。




