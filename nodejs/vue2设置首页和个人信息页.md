@[TOC](vue2设置首页和个人信息页)

# 设置首页
编辑`client/src/views/Home.vue`：
```html
<template>
    <div class="home">
        <div class="container">
            <h1 class="title">MamaHuhu资金管理系统</h1>
            <p class="lead">千里之堤，溃于蚁穴。细节决定成败！</p>
        </div>
    </div>
</template>

<style scoped>
    .home {
        width: 100%;
        height: 100%;
        background: url(../assets/showcase.png) no-repeat;
        background-size: 100% 100%;
    }

    .container {
        width: 100%;
        height: 100%;
        box-sizing: border-box;
        padding-top: 100px;
        background-color: rgba(0, 0, 0, 0.7);
        text-align: center;
        color: white;
    }

    .title {
        font-size: 30px;
    }

    .lead {
        margin-top: 50px;
        font-size: 22px;
    }
</style>
```

编辑`client/src/views/Index.vue`，添加`router-view`以显示Home组件：
```html
<template>
    <div class="index">
        <HeadNav></HeadNav>
        <router-view></router-view>
    </div>
</template>

<script>
    import HeadNav from "../components/HeadNav"
    export default {
      name: 'index',
      components: {
        HeadNav
      }
    };
</script>

<style>
    .index {
        width: 100%;
        height: 100%;
        overflow: hidden;
    }

</style>
```

编辑`client/src/router/index.js`，为Home配置路由：
```javascript
import Vue from 'vue'
import VueRouter from 'vue-router'
import Index from '../views/Index.vue'
import Register from '../views/Register.vue'
import Login from '../views/Login.vue'
import NotFound from '../views/404.vue'
import Home from '../views/Home.vue'

Vue.use(VueRouter)

const routes = [
  {
    path: '/',
    redirect: 'index'
  },
  {
    path: '/index',
    name: 'index',
    component: Index,
      children: [
          { path: '', component: Home },
          { path: '/home', name: 'home', component: Home }
      ]
   },
  {
      path: '/register',
      name: 'register',
      component: Register
    },
  {
        path: '/login',
        name: 'login',
        component: Login
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

router.beforeEach((to, from, next) => {
    const isLogin = localStorage.eleToken ? true : false;
    if (to.path == '/login' || to.path == '/register') {
        next();
    } else {
        isLogin ? next() : next('/login');
    }
});

export default router
```

# 设置个人信息页
编辑`client/src/router/index.js`，为个人信息页面配置路由：
```javascript
import Vue from 'vue'
import VueRouter from 'vue-router'
import Index from '../views/Index.vue'
import Register from '../views/Register.vue'
import Login from '../views/Login.vue'
import NotFound from '../views/404.vue'
import Home from '../views/Home.vue'
import InfoShow from '../views/InfoShow'

Vue.use(VueRouter)

const routes = [
  {
    path: '/',
    redirect: 'index'
  },
  {
    path: '/index',
    name: 'index',
    component: Index,
      children: [
          { path: '', component: Home },
          { path: '/home', name: 'home', component: Home },
          { path: '/infoshow', name: 'infoshow', component: InfoShow }
      ]
   },
  {
      path: '/register',
      name: 'register',
      component: Register
    },
  {
        path: '/login',
        name: 'login',
        component: Login
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

router.beforeEach((to, from, next) => {
    const isLogin = localStorage.eleToken ? true : false;
    if (to.path == '/login' || to.path == '/register') {
        next();
    } else {
        isLogin ? next() : next('/login');
    }
});

export default router
```

创建个人信息展示页面`InfoShow.vue`：
```html
<template>
    <div class="infoshow">
        <el-row type="flex" class="row-bg" justify="center">
            <el-col :span="8">
                <div class="user">
                    <img :src="user.avatar" class="avatar" alt="" />
                </div>
            </el-col>
            <el-col :span="16">
                <div class="userinfo">
                    <div class="user-item">
                        <i class="fa fa-user"></i>
                        <span>{{user.name}}</span>
                    </div>
                    <div class="user-item">
                        <i class="fa fa-cog"></i>
                        <span>{{user.identity == "manager" ? "经理":"员工"}}</span>
                    </div>
                </div>
            </el-col>
        </el-row>
    </div>
</template>

<script>
    export default {
        name: 'infoshow',
        computed: {
            user() {
                return this.$store.getters.user;
            }
        }
    }

</script>

<style scoped>
    .infoshow {
        width: 100%;
        height: 100%;
        box-sizing: border-box;
        /* padding: 16px; */
    }

    .row-bg {
        width: 100%;
        height: 100%;
    }

    .user {
        text-align: center;
        position: relative;
        top: 30%;
    }

    .user img {
        width: 150px;
        border-radius: 50%;
    }

    .user span {
        display: block;
        text-align: center;
        margin-top: 20px;
        font-size: 20px;
        font-weight: bold;
    }

    .userinfo {
        height: 100%;
        background-color: #eee;
    }

    .user-item {
        position: relative;
        top: 30%;
        padding: 26px;
        font-size: 28px;
        color: #333;
    }
</style>
```

在`client/public/index.html`中引入`font-awesome.css`：
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width,initial-scale=1.0">
    <link rel="icon" href="<%= BASE_URL %>favicon.ico">
    <link href="//cdn.bootcss.com/font-awesome/4.7.0/css/font-awesome.css" rel="stylesheet">
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

编辑`client/src/components/HeadNav.vue`，修改下拉菜单的`showInfoList`方法：
```javascript
methods: {
            setDialogInfo(cmdItem) {
                //console.log(cmdItem);
                switch (cmdItem) {
                    case "info":
                        this.showInfoList();
                        break;
                    case "logout":
                        this.logout();
                        break;
                }
            },
            showInfoList() {
                this.$router.push('/infoshow');
            },
            logout() {
                //清除token
                localStorage.removeItem('eleToken');
                //设置Vuex store
                this.$store.dispatch('clearCurrentState');
                //跳转到登录页面
                this.$router.push('/login');
            }
        }
```

