@[TOC](Vue2+element-ui配置Register和404页面)

# 安装element-ui
```bash
$ cd client
$ cnpm i element-ui -S
```

在`main.js`中引入element-ui：
```javascript
import Vue from 'vue'
import App from './App.vue'
import ElementUI from 'element-ui';
import 'element-ui/lib/theme-chalk/index.css';

import router from './router'
import store from './store'

Vue.config.productionTip = false
Vue.use(ElementUI);

new Vue({
  router,
  store,
  render: h => h(App)
}).$mount('#app')
```

# 配置Register.vue
创建`client/src/views/Register.vue`：
```html
<template>
    <div class="register">
        <section class="form_container">
            <div class="manage_tip">
                <span class="title">Kisu Management Platform</span>
            </div>
        </section>
    </div>
</template>

<scripts>
    export default {
    name: "register",
    components: {}
    };
</scripts>

<style scoped>
    .register {
        position: relative;
        width: 100%;
        height: 100%;
        background: url(../assets/bg.jpg) no-repeat center center;
        background-size: 100% 100%;
    }
    .form_container {
        width: 370px;
        height: 210px;
        position: absolute;
        top: 10%;
        left: 34%;
        padding: 25px;
        border-radius: 5px;
        text-align: center;
    }

    .form_container .manage_tip .title {
        font-family: "Microsoft YaHei";
        font-weight: bold;
        font-size: 26px;
        color: #fff;
    }
</style>
```

配置路由`client/src/router/index.js`：
```javascript
import Vue from 'vue'
import VueRouter from 'vue-router'
import Index from '../views/Index.vue'
import Register from '../views/Register.vue'

Vue.use(VueRouter)

const routes = [
  {
    path: '/',
    redirect: 'index'
  },
  {
    path: '/index',
    name: 'index',
    component: Index
    },
  {
      path: '/register',
      name: 'register',
      component: Register
  }
]

const router = new VueRouter({
  mode: 'history',
  base: process.env.BASE_URL,
  routes
})

export default router
```

在浏览器中打开：
```
http://localhost:8080/register
```

# 配置404.vue
创建`client/src/views/404.vue`：
```html
<template>
    <div class="notfound">
        <img src="../assets/404.webp" alt="">
    </div>
</template>

<style>
.notfound{
    width: 100%;
    height: 100%;
    overflow: hidden;
}
.notfound img{
    width: 100%;
    height: 100%;
}
</style>
```

配置路由`client/src/router/index.js`：
```javascript
import Vue from 'vue'
import VueRouter from 'vue-router'
import Index from '../views/Index.vue'
import Register from '../views/Register.vue'
import NotFound from '../views/404.vue'

Vue.use(VueRouter)

const routes = [
  {
    path: '/',
    redirect: 'index'
  },
  {
    path: '/index',
    name: 'index',
    component: Index
    },
  {
      path: '/register',
      name: 'register',
      component: Register
    },
  {
      path: '*',
      name: '/404',
      component: NotFound
  }
]

const router = new VueRouter({
  mode: 'history',
  base: process.env.BASE_URL,
  routes
})

export default router
```

在浏览器中打开：
```
http://localhost:8080/registerxyy
http://localhost:8080/lol23
...
```
