@[TOC](vue2+axios实现注册页面加载动画消息和配置跨域代理)

# 安装和使用axios
安装axios工具：
```bash
$ cd client
$ npm install axios
```

在`client/src/main.js`中引入axios：
```javascript
import Vue from 'vue';
import App from './App.vue';
import axios from './http';
import ElementUI from 'element-ui';
import 'element-ui/lib/theme-chalk/index.css';

import router from './router';
import store from './store';

Vue.config.productionTip = false;
Vue.use(ElementUI);
Vue.prototype.$axios = axios;

new Vue({
  router,
  store,
  render: h => h(App)
}).$mount('#app')
```

# 定义加载动画和消息
在`client/src`目录下新建请求文件`http.js`，定义加载动画和方法：
```javascript
vim client/src/http.js

import axios from 'axios';
import { Message, Loading } from 'element-ui';

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
    return Promise.reject(error);
})

export default axios;
```

# 配置跨域代理
编辑`client/vue.config.js`配置前后端跨域请求：
```javascript
const path = require('path')
const debug = process.env.NODE_ENV !== 'production'

module.exports = {
    baseUrl: '/', // 根域上下文目录
    outputDir: 'dist', // 构建输出目录
    assetsDir: 'assets', // 静态资源目录 (js, css, img, fonts)
    lintOnSave: false, // 是否开启eslint保存检测，有效值：ture | false | 'error'
    runtimeCompiler: true, // 运行时版本是否需要编译
    transpileDependencies: [], // 默认babel-loader忽略mode_modules，这里可增加例外的依赖包名
    productionSourceMap: true, // 是否在构建生产包时生成 sourceMap 文件，false将提高构建速度
    configureWebpack: config => { // webpack配置，值位对象时会合并配置，为方法时会改写配置
        if (debug) { // 开发环境配置
            config.devtool = 'cheap-module-eval-source-map'
        } else { // 生产环境配置
        }
        // Object.assign(config, { // 开发生产共同配置
        //     resolve: {
        //         alias: {
        //             '@': path.resolve(__dirname, './src'),
        //             '@c': path.resolve(__dirname, './src/components'),
        //             'vue$': 'vue/dist/vue.esm.js'
        //         }
        //     }
        // })
    },
    chainWebpack: config => { // webpack链接API，用于生成和修改webapck配置，https://github.com/vuejs/vue-cli/blob/dev/docs/webpack.md
        if (debug) {
            // 本地开发配置
        } else {
            // 生产开发配置
        }
    },
    parallel: require('os').cpus().length > 1, // 构建时开启多进程处理babel编译
    pluginOptions: { // 第三方插件配置
    },
    pwa: { // 单页插件相关配置 https://github.com/vuejs/vue-cli/tree/dev/packages/%40vue/cli-plugin-pwa
    },
    devServer: {
        open: true,
        host: 'localhost',
        port: 8080,
        https: false,
        hotOnly: false,
        proxy: { // 配置跨域
            '/api': {
                target: 'http://localhost:5001/api/',
                ws: true,
                changOrigin: true,
                pathRewrite: {
                    '^/api': ''
                }
            }
        },
        before: app => { }
    }
}
```

运行`npm run dev`启动应用时遇到以下报错。

## 错误1：baseUrl
```
ERROR Invalid options in vue.config.js: "baseUrl" is not allowed.
```

在vue-cli.3.3版本后`baseUrl`被废除了，这里需要将`vue.config.js`中的
```
baseUrl
```
改为
```
publicPath
```

## 错误2：webpack config.devtool
```
Webpack has been initialized using a configuration object that does not match the API schema.
 - configuration.devtool should match pattern "^(inline-|hidden-|eval-)?(nosources-)?(cheap-(module-)?)?source-map$".
```

Webpack的配置不符合当前版本语法。将`vue.config.js`中的
```javascript
config.devtool = 'cheap-module-eval-source-map'
```
修改为 
```javascript
config.devtool = 'eval-cheap-module-source-map'
```

## 错误3：webpack devServer before
```
Dev Server has been initialized using an options object that does not match the API schema.
 - options has an unknown property 'before'.
```

在webpack 3.x中，`before`是有效属性，到了4.x之后，就换成了`onBeforeSetupMiddleware`。将`vue.config.js`中的
```javascript
before: app => { }
```
修改为
```javascript
onBeforeSetupMiddleware: function (devServer) {
            if (!devServer) {
                throw new Error('webpack-dev-server is not defined');
            }

            devServer.app.get('/some/path', function (req, res) {
                res.json({ custom: 'response' });
            });
        }
```
		
## 错误4：webpack devServer hotOnly
```
Dev Server has been initialized using an options object that does not match the API schema.
 - options has an unknown property 'hotOnly'.
```
将`vue.config.js`中的
```
hotOnly: false
```
修改为
```
hot: true
```

最终的`vue.config.js`内容为
```javascript
const path = require('path')
const debug = process.env.NODE_ENV !== 'production'

module.exports = {
    // baseUrl: '/', // 根域上下文目录
    publicPath: '/', // 根域上下文目录
    outputDir: 'dist', // 构建输出目录
    assetsDir: 'assets', // 静态资源目录 (js, css, img, fonts)
    lintOnSave: false, // 是否开启eslint保存检测，有效值：ture | false | 'error'
    runtimeCompiler: true, // 运行时版本是否需要编译
    transpileDependencies: [], // 默认babel-loader忽略mode_modules，这里可增加例外的依赖包名
    productionSourceMap: true, // 是否在构建生产包时生成 sourceMap 文件，false将提高构建速度
    configureWebpack: config => { // webpack配置，值位对象时会合并配置，为方法时会改写配置
        if (debug) { // 开发环境配置
            //config.devtool = 'cheap-module-eval-source-map'
            config.devtool = 'eval-cheap-module-source-map'
        } else { // 生产环境配置
        }
        // Object.assign(config, { // 开发生产共同配置
        //     resolve: {
        //         alias: {
        //             '@': path.resolve(__dirname, './src'),
        //             '@c': path.resolve(__dirname, './src/components'),
        //             'vue$': 'vue/dist/vue.esm.js'
        //         }
        //     }
        // })
    },
    chainWebpack: config => { // webpack链接API，用于生成和修改webapck配置，https://github.com/vuejs/vue-cli/blob/dev/docs/webpack.md
        if (debug) {
            // 本地开发配置
        } else {
            // 生产开发配置
        }
    },
    parallel: require('os').cpus().length > 1, // 构建时开启多进程处理babel编译
    pluginOptions: { // 第三方插件配置
    },
    pwa: { // 单页插件相关配置 https://github.com/vuejs/vue-cli/tree/dev/packages/%40vue/cli-plugin-pwa
    },
    devServer: {
        open: true,
        host: 'localhost',
        port: 8080,
        https: false,
        // hotOnly: false,
        hot: true,
        proxy: { // 配置跨域
            '/api': {
                target: 'http://localhost:5001/api/',
                ws: true,
                changOrigin: true,
                pathRewrite: {
                    '^/api': ''
                }
            }
        },
        // before: app => { }
        onBeforeSetupMiddleware: function (devServer) {
            if (!devServer) {
                throw new Error('webpack-dev-server is not defined');
            }

            devServer.app.get('/some/path', function (req, res) {
                res.json({ custom: 'response' });
            });
        }
    }
}
```

# 配置注册页面
编辑`Register.vue`中的`submitForm`方法，加入axios跨域请求：
```javascript
methods: {
         submitForm(formName) {
             this.$refs[formName].validate(valid => {
                 if (valid) {
                     //alert('submit!');
                     this.$axios.post("/api/users/register", this.registerUser)
                         .then(res => {
                             //注册成功
                             this.$message({
                                 message: "账号注册成功！",
                                 type: 'success'
                             });
                         });
                     //跳转到登录页面
                     this.$router.push('/login');
                 } else {
                     console.log('error submit!!');
                     return false;
                 }
             });
         }
     }
```	 
	 

打开`http://localhost:8080/register`，填写用户互信息并点击注册，短暂加载动画过后，会提示注册成功，并跳转到`http://localhost:8080/login`页面（因为我们还没写登录页面，会显示404信息）。

在mongoDB数据库中应该可以看到新增了通过注册网页注册用户的信息。
	 

**References**
【1】[vue.config.js](https://cli.vuejs.org/zh/config/#vue-config-js)
【2】[webpack中文文档](https://webpack.docschina.org/configuration/dev-server/)
