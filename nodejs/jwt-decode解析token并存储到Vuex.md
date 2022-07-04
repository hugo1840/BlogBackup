@[TOC](jwt-decode解析token并存储到Vuex)

# 安装jwt-decode
安装Token解析工具：
```bash
$ cd client
$ cnpm install jwt-decode
```

# 解析token
在`Login.vue`中引入`jwt-decode`来解析token：
```javascript
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
    import jwt_decode from 'jwt-decode';
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
                             //console.log(res);
                             const { token } = res.data;
                             // 存储token到LocalStorage
                             localStorage.setItem('eleToken', token);

                             //解析token
                             const decoded = jwt_decode(token);
                             console.log(decoded);

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

# Vuex store配置
编辑`client/src/store/index.js`，定义将token信息存储到Vuex的`actions`：
```javascript
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

const types = {
    SET_AUTHENTICATED: 'SET_AUTHENTICATED',
    SET_USER: 'SET_USER'
};

const state = {
    isAuthenticated: false,
    user: {}
};

const getters = {
    isAuthenticated: state => state.isAuthenticated,
    user: state => state.user
};

const mutations = {
    [types.SET_AUTHENTICATED](state, isAuthenticated) {
        if (isAuthenticated) state.isAuthenticated = isAuthenticated;
        else state.isAuthenticated = false;

    },
    [types.SET_USER](state, user) {
        if (user) state.user = user;
        else state.user = {};
    }
};

const actions = {
    setAuthenticated: ({ commit }, isAuthenticated) => {
        commit(types.SET_AUTHENTICATED, isAuthenticated);
    },
    setUser: ({ commit }, user) => {
        commit(types.SET_USER, user);
    }
};

export default new Vuex.Store({
  state,
  getters,
  mutations,
  actions
})
```

# 存储token
修改`Login.vue`中的methods，将token信息存储到Vuex：
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

                             //解析token
                             const decoded = jwt_decode(token);
                             //console.log(decoded);
                             //存储到Vuex
                             this.$store.dispatch('setAuthenticated', !this.isEmpty(decoded));
                             this.$store.dispatch('setUser', decoded);

                             //跳转index页面
                             this.$router.push('/index');
                         });
        
                 } else {
                     console.log('error submit!!');
                     return false;
                 }
             });
         },
         isEmpty(value) {
             return (
                 value === undefined ||
                 value === null ||
                 (typeof value === "object" && Object.keys(value).length === 0) ||
                 (typeof value === "string" && value.trim().length === 0)
             );
         }
     }
```

在浏览器中安装**Vue devtool**测试插件后，登录后可以在Root组件中看到解析后存储的用户token信息。不过如果刷新了网页，存储的token信息就会丢失。为了避免这种情况，我们需要在`App.vue`中进行相关配置。

编辑`client/src/App.vue`：
```html
<template>
  <div id="app">
    <router-view/>
  </div>
</template>

<script>
    import jwt_decode from 'jwt-decode';
    export default {
    name: "app",
    components: {},
    created() {
            if (localStorage.eleToken) {
                //解析token
                const decoded = jwt_decode(localStorage.eleToken);
                //存储到Vuex
                this.$store.dispatch('setAuthenticated', !this.isEmpty(decoded));
                this.$store.dispatch('setUser', decoded);
            }
    },
    methods: {
        isEmpty(value) {
            return (
                value === undefined ||
                value === null ||
                (typeof value === "object" && Object.keys(value).length === 0) ||
                (typeof value === "string" && value.trim().length === 0)
            );
        }
     }
 }
</script>

<style>
 html,
 body,
 #app {
     width: 100%;
     height: 100%
 }
</style>
```
