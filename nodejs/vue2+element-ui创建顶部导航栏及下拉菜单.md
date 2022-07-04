@[TOC](vue2+element-ui创建顶部导航栏及下拉菜单)

# 引入顶部导航栏
创建`client/src/components/HeadNav.vue`顶部导航栏组件：
```html
<template>
    <header class="head-nav">
        <el-row>
            <el-col :span="6" class="logo-container">
                <img src="../assets/logo.png" class="logo" alt="" />
                <span class="title">马马虎虎资金管理系统</span>
            </el-col>
            <el-col :span="6" class="user">
                <div class="userinfo">
                    <img src="user.avatar" class="avatar" alt="" />
                    <div class="welcome">
                        <p class="name comename">欢迎</p>
                        <p class="name avatarname">褪色者</p>
                    </div>
                    <span class="username">
                        <!--下拉菜单-->
                    </span>
                </div>
            </el-col>
        </el-row>
    </header>
</template>

<script>
    export default {
        name: "head-nav"
    };
</script>

<style scoped>
    .head-nav {
        width: 100%;
        height: 60px;
        min-width: 600px;
        padding: 5px;
        background: #324057;
        color: #fff;
        border-bottom: 1px solid #1f2d3d;
    }

    .logo-container {
        line-height: 60px;
        min-width: 400px;
    }

    .logo {
        height: 50px;
        width: 50px;
        margin-right: 5px;
        vertical-align: middle;
        display: inline-block;
    }

    .title {
        vertical-align: middle;
        font-size: 22px;
        font-family: "Microsoft YaHei";
        letter-spacing: 3px;
    }

    .user {
        line-height: 60px;
        text-align: right;
        float: right;
        padding-right: 10px;
    }

    .avatar {
        width: 40px;
        height: 40px;
        border-radius: 50%;
        vertical-align: middle;
        display: inline-block;
    }

    .welcome {
        display: inline-block;
        width: auto;
        vertical-align: middle;
        padding: 0 5px;
    }

    .name {
        line-height: 20px;
        text-align: center;
        font-size: 14px;
    }

    .comename {
        font-size: 12px;
    }

    .avatarname {
        color: #409eff;
        font-weight: bolder;
    }

    .username {
        cursor: pointer;
        margin-right: 5px;
    }

    .el-dropdown {
        color: #fff;
    }
</style>
```

在`Index.vue`中引入顶部导航栏组件：
```javascript
<template>
    <div class="index">
        <HeadNav></HeadNav>
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
```

# 使用下拉菜单
编辑`HeadNav.vue`，利用Element-ui创建下拉菜单：
```html
<span class="username">
    <!--下拉菜单-->
    <el-dropdown trigger="click" @command="setDialogInfo">
        <span class="el-dropdown-link">
            <i class="el-icon-caret-bottom el-icon--right"></i>
        </span>
        <el-dropdown-menu slot="dropdown">
            <el-dropdown-item command="info">个人信息</el-dropdown-item>
            <el-dropdown-item command="logout">退出</el-dropdown-item>
        </el-dropdown-menu>
    </el-dropdown>
</span>
```

# 获取Vuex store数据
编辑`HeadNav.vue`，通过Vuex获取存储的用户token，在导航栏展示用户名和头像。
```html
<template>
    <header class="head-nav">
        <el-row>
            <el-col :span="6" class="logo-container">
                <img src="../assets/logo.png" class="logo" alt="" />
                <span class="title">马马虎虎资金管理系统</span>
            </el-col>
            <el-col :span="6" class="user">
                <div class="userinfo">
                    <img :src="user.avatar" class="avatar" alt="" />
                    <div class="welcome">
                        <p class="name comename">欢迎</p>
                        <p class="name avatarname">{{user.name}}</p>
                    </div>
                    <span class="username">
                        <!--下拉菜单-->
                        <el-dropdown trigger="click" @command="setDialogInfo">
                            <span class="el-dropdown-link">
                                <i class="el-icon-caret-bottom el-icon--right"></i>
                            </span>
                            <el-dropdown-menu slot="dropdown">
                                <el-dropdown-item command="info">个人信息</el-dropdown-item>
                                <el-dropdown-item command="logout">退出</el-dropdown-item>
                            </el-dropdown-menu>
                        </el-dropdown>
                    </span>
                </div>
            </el-col>
        </el-row>
    </header>
</template>

<script>
    export default {
        name: "head-nav",
        computed: {
            user() {
                return this.$store.getters.user;
            }
        }
    };
</script>
```					
	
# 配置Vuex store
编辑`client/src/store/index.js`，定义下拉菜单中退出时清除登录状态使用的`actions`：
```javascript
const actions = {
    setAuthenticated: ({ commit }, isAuthenticated) => {
        commit(types.SET_AUTHENTICATED, isAuthenticated);
    },
    setUser: ({ commit }, user) => {
        commit(types.SET_USER, user);
    },
    clearCurrentState: ({ commit }) => {
        commit(types.SET_AUTHENTICATED, false);
        commit(types.SET_USER, null);
    }
};
```

# 定义logout方法
在`HeadNav.vue`中定义点击下拉菜单退出时使用的`logout`方法：
```javascript
<script>
    export default {
        name: "head-nav",
        computed: {
            user() {
                return this.$store.getters.user;
            }
        },
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
                console.log("个人信息");
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
    };
</script>
```
	
	
