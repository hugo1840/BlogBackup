@[TOC](Visual Studio+Node.js开发环境安装配置)

# 基础安装
到微软官方下载[Visual Studio社区版](https://visualstudio.microsoft.com/zh-hans/)。安装后选择安装node.js开发环境。

到nodejs官网下载[官方介质](https://nodejs.org/zh-cn/download/releases/)。不用选择太新的版本。

安装后在终端查看版本信息
```bash
> node -v
v14.19.3

> npm -v
6.14.17
```
查看配置信息
```bash
> npm list -global
> npm config list
```

# 一些可有可无的配置
如果不想全局模块和缓存文件都存放在C盘（默认在`C:\Users\xiaoming\AppData\Roaming\npm`），可以修改他们的位置。

```bash
> cd D:
> mkdir node.js
> cd node.js
> mkdir node_global
> mkdir node_cache 

> npm config set prefix "D:\node.js\node_global"
> npm config set cache "D:\node.js\node_cache"
```

再次执行npm命令，会报错：
```bash
Error: EINVAL: invalid argument, mkdir D:\node.js\node_global"'
```

修改配置文件`C:\Users\xiaoming\.npmrc`，添加如下内容：
```bash
prefix=D:\node.js\node_global
cache=D:\node.js\node_cache
```

在环境变量 > 系统变量 > PATH中添加：
```bash
D:\node.js\node_global
```

再次执行npm命令，正常运行：
```
> npm config list
; cli configs
metrics-registry = "https://registry.npmjs.org/"
scope = ""
user-agent = "npm/6.14.17 node/v14.19.3 win32 x64"

; userconfig C:\Users\xiaoming\.npmrc
cache = "D:\\node.js\\node_cache"
prefix = "D:\\node.js\\node_global"

; builtin config undefined

; node bin location = C:\Program Files\nodejs\node.exe
; cwd = C:\Users\xiaoming
; HOME = C:\Users\xiaoming
; "npm config ls -l" to show all defaults.
```


如果npm安装包比较慢，可以安装使用淘宝的**cnpm**镜像。淘宝的cnpm命令管理工具可以代替默认的npm管理工具。
```bash
> npm install -g cnpm --registry=https://registry.npm.taobao.org
```

在系统变量 > xiaoming的用户变量中新建用户变量`NODE_PATH`，设置为
```bash
D:\node.js\node_global\node_modules
```

在系统变量 > xiaoming的用户变量`PATH`中，将
```bash
C:\Users\xiaoming\AppData\Roaming\npm
```
修改为
```bash
D:\node.js\node_global\node_modules
```

即可在命令行中直接使用cnpm命令：
```bash
> cnpm -v
cnpm@8.2.0 (D:\node.js\node_global\node_modules\cnpm\lib\parse_argv.js)
npm@8.12.1 (D:\node.js\node_global\node_modules\cnpm\node_modules\npm\index.js)
node@14.19.3 (C:\Program Files\nodejs\node.exe)
npminstall@6.3.0 (D:\node.js\node_global\node_modules\cnpm\node_modules\npminstall\lib\index.js)
prefix=D:\node.js\node_global
win32 x64 10.0.19044
registry=https://registry.npmmirror.com
```

# 搭建一个简单的Web服务器
新建项目文件夹如下
```bash
C:\Users\xiaoming\source\repos\node_demo\node_app
```

在该目录下右键打开Visual Studio。在VS中，使用快捷键**Ctrl+反引号**，在VS中打开PowerShell终端。

切换到项目路径下
```bash
> cd C:\Users\xiaoming\source\repos\node_demo\node_app
```

初始化项目
```bash
> npm init
```
补充相应信息，回车结束命令后当前路径下会生成一个`package.json`文件。

创建入口文件（这里使用了PowerShell命令，也可以手动创建）
```bash
> new-item server.js -type file
```

安装express模块（可用来快速创建Web服务器）
```bash
> npm install express
```

编辑入口文件server.js

```javascript
const express = require("express");
const app = express();

// 设置app路由
app.get("/", (req, res) => {
    res.send("Hello World!");
})

const port = process.env.PORT || 5000;
app.listen(port, () => {
    console.log(`Server running on port ${port}`);
})
```

# 安装nodemon
Nodemon支持在修改代码后无需重启即可生效，适用于开发环境。
```bash
> npm install nodemon -g
```

在VS打开的终端中执行cnpm和nodemon命令报错：
```
nodemon : 无法加载文件 D:\node.js\node_global\nodemon.ps1,因为在此系统上禁止运行脚本。
```

在PowerShell中执行以下命令来修改执行权限：
```
> Set-ExecutionPolicy -Scope CurrentUser
位于命令管道位置 1 的 cmdlet Set-ExecutionPolicy
请为以下参数提供值:
ExecutionPolicy: RemoteSigned
```

即可正常执行：
```bash
> nodemon -v
2.0.16
```

# 启动Web服务器
第一种启动方式
```bash
> node server.js
```
这种方式启动时，每次改变代码后都需要重启，不适用于开发测试场景。执行`Ctrl+C`停止运行，

第二种启动方式
```bash
> nodemon server.js
```
这种方式启动时，每次改变代码后都不需要重启，适用于开发场景。

修改package.json中的scripts部分：
```javascript
"scripts": {
    "start": "node server.js",
    "server": "nodemon server.js"
  }
  ```
  
之后在终端执行
```bash
> npm run start
```
相当于执行 `node server.js`。

执行
```bash
> npm run server
```
相当于执行 `nodemon server.js`。

在浏览器打开`http://localhost:5000/`，可以见到
```
Hello World!
```
