
@[TOC](vue2+axios配置前端路由守卫并获取登录token)

# 编写登录页面
编写`client/src/views/Login.vue`登陆页面：
```html
<template>
    <div class="login">
        <section class="form_container">
            <div class="manage_tip">
                <span class="title">马马虎虎资金管理系统</span>
                <el-form :model="loginUser" :rules="rules" ref="loginForm" label-width="60px" class="loginForm">
                    <el-form-item label="邮箱" prop="email">
                        <el-input v-model="loginUser.email" placeholder="请输入邮箱地址"></el-input>
                    </el-form-item>
                    <el-form-item label="密码" prop="password">
                        <el-input type="password" v-model="loginUser.password" placeholder="请输入密码"></el-input>
                    </el-form-item>
                    <el-form-item>
                        <el-button typeof="primary" class="submit_btn" @click="submitForm('loginForm')">登录</el-button>
                    </el-form-item>
                    <div class="tiparea">
                        <p>还没有账号？现在<router-link to="/register">注册</router-link></p>
                    </div>
                </el-form>
            </div>
        </section>
    </div>
</template>

<script>
    export default {
    name: "login",
    components: {},
    data() {

        return {
            loginUser: {
                email: '',
                password: '' 
            },
            rules: {
                email: [
                    {
                        type: "email",
                        required: true,
                        message: "邮箱格式不正确",
                        trigger: "blur"
                    }
                ],
                password: [
                    {
                        required: true,
                        message: "密码不能为空",
                        trigger: "blur"
                    },
                    {
                        min: 6,
                        max: 30,
                        message: "密码长度在6至30个字符之间",
                        trigger: "blur"
                    }
                ]
            }
        }
     },
     methods: {
         submitForm(formName) {
             this.$refs[formName].validate(valid => {
                 if (valid) {
                     this.$axios.post("/api/users/login", this.loginUser)
                         .then(res => {
                             //拿到token
                             console.log(res);
                         });
                     //跳转页面
                     this.$router.push('/index');
                 } else {
                     console.log('error submit!!');
                     return false;
                 }
             });
         }
     }
 }
</script>

<style scoped>
    .login {
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
        top: 20%;
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

    .loginForm {
        margin-top: 20px;
        background-color: #fff;
        padding: 20px 40px 20px 20px;
        border-radius: 5px;
        box-shadow: 0px 5px 10px #cccc;
    }

    .submit_btn {
        width: 100%;
    }

    .tiparea {
        text-align: right;
        font-size: 12px;
        color: #333;
    }

    .tiparea p a {
        color: #409eff;
    }
</style>
```

编辑`client/src/router/index.js`，配置登录页面路由：
```javascript
import Vue from 'vue'
import VueRouter from 'vue-router'
import Index from '../views/Index.vue'
import Register from '../views/Register.vue'
import Login from '../views/Login.vue'
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

export default router
```

# 获取登录token
编辑`Login.vue`中的`submitForm`，登陆成功后获取token并存储到`localStorage`。
```javascript
methods: {
         submitForm(formName) {
             this.$refs[formName].validate(valid => {
                 if (valid) {
                     this.$axios.post("/api/users/login", this.loginUser)
                         .then(res => {
                             //拿到token
                             //console.log(res);
                             const { token } = res.data;
                             // 存储token到LocalStorage
                             localStorage.setItem('eleToken', token);
                             //跳转index页面
                             this.$router.push('/index');
                         });
        
                 } else {
                     console.log('error submit!!');
                     return false;
                 }
             });
         }
     }
```


# 配置路由守卫
在未登录状态下，只能访问注册页面或者登陆页面，无法访问Index等其他页面。编辑`client/src/router/index.js`：
```javascript
import Vue from 'vue'
import VueRouter from 'vue-router'
import Index from '../views/Index.vue'
import Register from '../views/Register.vue'
import Login from '../views/Login.vue'
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

//路由守卫
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

# 配置请求/响应拦截
编辑`http.js`，在axios请求拦截中配置统一请求头，在响应拦截中配置token过期时的处理动作。
```javascript
import axios from 'axios';
import { Message, Loading } from 'element-ui';
import router from './router';

// 定义加载动画方法
let loading;
function startLoading() {
    loading = Loading.service({
        lock: true,
        text: "稍待片刻...",
        background: 'rgba(0,0,0,0,7)'
    });
}

function endLoading() {
    loading.close();
}

// 请求拦截
axios.interceptors.request.use(config => {
    startLoading();

    if (localStorage.eleToken) {
        // 设置统一的request header
        config.headers.Authorization = localStorage.eleToken;
    }

    return config;
}, error => {
    return Promise.reject(error);
})

// 响应拦截
axios.interceptors.response.use(response => {
    endLoading();
    return response;
}, error => {
    endLoading();
    Message.error(error.response.data);

    //获取错误状态码，并清除过期的token
    const { status } = error.response;
    if (status == 401) {
        Message.error('Token已过期，请重新登录！');
        // 清除失效token
        localStorage.removeItem('eleToken');
        // 跳转到登录页面
        router.push("/login");
    }

    return Promise.reject(error);
})

export default axios;
```







