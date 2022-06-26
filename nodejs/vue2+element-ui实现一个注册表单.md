@[TOC](vue2+element-ui实现一个注册表单)

# element-ui表单
在`client/src/views/Register.vue`中插入表单：
```html
<template>
    <div class="register">
        <section class="form_container">
            <div class="manage_tip">
                <span class="title">Kisu Management Platform</span>
                <el-form :model="registerUser" :rules="rules" ref="registerForm" label-width="80px" class="registerForm">
                    <el-form-item label="用户名" prop="name">
                        <el-input v-model="registerUser.name" placeholder="请输入用户名"></el-input>
                    </el-form-item>
                    <el-form-item label="邮箱" prop="email">
                        <el-input v-model="registerUser.email" placeholder="请输入邮箱地址"></el-input>
                    </el-form-item>
                    <el-form-item label="密码" prop="password">
                        <el-input type="password" v-model="registerUser.password" placeholder="请输入密码"></el-input>
                    </el-form-item>
                    <el-form-item label="确认密码" prop="password2">
                        <el-input type="password" v-model="registerUser.password2" placeholder="请再次输入密码"></el-input>
                    </el-form-item>
                    <el-form-item label="选择身份">
                        <el-select v-model="registerUser.identity" placeholder="请选择身份">
                            <el-option label="经理" value="manager"></el-option>
                            <el-option label="员工" value="employee"></el-option>
                        </el-select>
                    </el-form-item>
                    <el-form-item>
                        <el-button typeof="primary" class="submit_btn" @click="submitForm('registerForm')">注册</el-button>
                    </el-form-item>
                </el-form>
            </div>
        </section>
    </div>
</template>

<script>
    export default {
    name: "register",
    components: {}
    };
</script>

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

打开`http://localhost:8080/register`会发现看不到任何内容。查看浏览器控制台有如下报错：
```
[Vue warn]: Property or method "registerUser" is not defined on the instance but referenced during render.
```

修改Register.vue中的script标签，定义`registerUser`属性：
```html
<script>
    export default {
    name: "register",
    components: {},
    data(){
      return {
        registerUser: {
          name: '',
          email: '',
          password: '',
          password2: '',
          identity: ''
        }
      }
    }
 };
</script>
```

刷新网页`http://localhost:8080/register`，可以看到表单。浏览器控制台只剩下如下报错：
```
Property or method "rules" is not defined on the instance but referenced during render.
```

**注**：如果中文显示乱码，用笔记本打开vue文件，另存为`UTF-8`编码格式，然后替换掉原来的vue文件即可。

# 修改表单样式
修改Register.vue添加表单和按钮样式：
```html
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
    .registerForm {
        margin-top: 20px;
        background-color: #fff;
        padding: 20px 40px 20px 20px;
        border-radius: 5px;
        box-shadow: 0px 5px 10px #cccc;
    }

    .submit_btn {
        width: 100%;
    }
</style>
```

# 表单内容验证
修改Register.vue中的`<script>`标签，添加`rules`表单验证规则：
```html
<script>
    export default {
    name: "register",
    components: {},
    data() {
        var validatePass2 = (rule, value, callback) => {
            if (value === '') {
                callback(new Error('请再次输入密码'));
            } else if (value !== this.registerUser.password) {
                callback(new Error('两次输入密码不一致!'));
            } else {
                callback();
            }
        };

        return {
            registerUser: {
                name: '',
                email: '',
                password: '',
                password2: '',
                identity: ''
            },
            rules: {
                name: [
                    {
                        required: true,
                        message: "用户名不能为空",
                        trigger: "blur"
                    },
                    {
                        min: 2,
                        max: 30,
                        message: "用户名长度在2至30个字符之间",
                        trigger: "blur"
                    }
                ],
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
                ],
                password2: [
                    {
                        validator: validatePass2,
                        trigger: "blur"
                    }
                ]
            }
        }
    }
 };
</script>
```

修改Register.vue中的script标签，添加点击按钮时触发的`submitForm`方法：
```html
<script>
    export default {
    name: "register",
    components: {},
    data() {
        var validatePass2 = (rule, value, callback) => {
            if (value === '') {
                callback(new Error('请再次输入密码'));
            } else if (value !== this.registerUser.password) {
                callback(new Error('两次输入密码不一致!'));
            } else {
                callback();
            }
        };

        return {
            registerUser: {
                name: '',
                email: '',
                password: '',
                password2: '',
                identity: ''
            },
            rules: {
                name: [
                    {
                        required: true,
                        message: "用户名不能为空",
                        trigger: "blur"
                    },
                    {
                        min: 2,
                        max: 30,
                        message: "用户名长度在2至30个字符之间",
                        trigger: "blur"
                    }
                ],
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
                ],
                password2: [
                    {
                        validator: validatePass2,
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
                     alert('submit!');
                 } else {
                     console.log('error submit!!');
                     return false;
                 }
             });
         }
     }
 }
</script>
```

**References**
【1】[Element表单组件](https://element.eleme.cn/#/zh-CN/component/form)
