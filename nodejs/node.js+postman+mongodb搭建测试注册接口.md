@[TOC](node.js+postman+mongodb搭建测试注册接口)

# 准备工作
**申请一个免费的MongoDB**
到`https://www.mlab.com`注册申请一个500M的MongoDB数据库。登录后手动在创建Databases下的Collections中手动创建一个数据库`node_app`。

在个人首页点击Connect获取node.js连接MongoDB数据库的字符串为
```
mongodb+srv://<username>:<password>@cluster0.ylpaf.mongodb.net/node_app
```
将其中`<username>:<password>`修改为自己设定的数据库用户名和密码。

**下载安装Postman**
到`https://www.postman.com/`注册一个账号，下载安装Postman agent，即可方便地进行GET/POST/PUT等测试。

# mongodb连接串配置
安装mongoose用于连接数据库：
```bash
> npm install mongoose
> 
> cd C:\Users\xiaoming\source\repos\node_demo\node_app
> mkdir config
> cd config
> new-item keys.js -type file
```

编辑`keys.js`配置连接串：
```javascript
module.exports = {
    mongoURI: "mongodb+srv://<username>:<password>@cluster0.ylpaf.mongodb.net/node_app"
}
```

编辑`server.js`入口文件：
```javascript
const express = require("express");
const mongoose = require("mongoose");

const app = express();
const db = require("./config/keys").mongoURI;

mongoose.connect(db)
        .then(() => console.log("MongoDB connected."))
        .catch(err => console.log(err));

app.get("/", (req, res) => {
    res.send("Hello World!");
})

const port = process.env.PORT || 5000;
app.listen(port, () => {
    console.log(`Server running on port ${port}`);
})
```

检查是否能连接到数据库：
```bash
> nodemon server.js

[nodemon] 2.0.16
[nodemon] to restart at any time, enter `rs`
[nodemon] watching path(s): *.*
[nodemon] watching extensions: js,mjs,json
[nodemon] starting `node server.js`
Server running on port 5000
MongoDB connected.
```
数据库连接正常。

# GET请求测试
创建路由文件
```
C:\Users\xiaoming\source\repos\node_demo\node_app\routes\api\users.js
```

编辑`users.js`并添加GET请求：
```javascript
// login & registtration
const express = require("express");
const router = express.Router();

router.get("/test", (req,res) => {
    res.json({msg:"Login succeeded!"})
})

module.exports = router;
```

编辑`server.js`，导入并使用`users.js`：
```javascript
const express = require("express");
const mongoose = require("mongoose");

const app = express();
const users = require("./routes/api/users"); 

const db = require("./config/keys").mongoURI;
mongoose.connect(db)
    .then(() => console.log("MongoDB connected."))
    .catch(err => console.log(err));

// 设置app路由
app.get("/", (req, res) => {
    res.send("Hello World!");
})

// 使用users
app.use("/api/users", users);

const port = process.env.PORT || 5000;
app.listen(port, () => {
    console.log(`Server running on port ${port}`);
})
```

访问
```
http://localhost:5000/api/users/test
```
可以看到
```json
{"msg":"Login succeeded!"}
```

# 注册接口搭建
## 创建User数据模型
创建用户数据模型文件
```
C:\Users\xiaoming\source\repos\node_demo\node_app\models\User.js
```

编辑`User.js`创建用户数据模型：
```javascript
const mongoose = require("mongoose");
const Schema = mongoose.Schema;

// create Schema
const UserSchema = new Schema({
    name:{
        type: String,
        required: true
    },
    email: {
        type: String,
        required: true
    },
    password: {
        type: String,
        required: true
    },
    avatar: {
        type: String
    },
    date: {
        type: Date,
        default: Date.now
    },
})

module.exports = User = mongoose.model("users", UserSchema);
```

## 使用body-parser中间件
安装body-parser中间件，可以方便地处理HTTP请求。
```bash
> npm install body-parser
```

编辑`server.js`使用body-parser：
```javascript
const express = require("express");
const mongoose = require("mongoose");
const bodyParser = require("body-parser");

const app = express();
const users = require("./routes/api/users"); 

const db = require("./config/keys").mongoURI;

app.use(bodyParser.urlencoded({ extended: false }));
app.use(bodyParser.json());

mongoose.connect(db)
    .then(() => console.log("MongoDB connected."))
    .catch(err => console.log(err));

app.get("/", (req, res) => {
    res.send("Hello World!");
})

app.use("/api/users", users);

const port = process.env.PORT || 5000;
app.listen(port, () => {
    console.log(`Server running on port ${port}`);
})
```

## POST请求测试
编辑`users.js`增加POST请求：
```javascript
// @login & registtration
const express = require("express");
const router = express.Router();

/*
 * $route GET /api/users/test
 * @desc return requested json data
 * @access public
 */
router.get("/test", (req,res) => {
    res.json({msg:"Login succeeded!"})
})

/*
 * $route POST /api/users/register
 * @desc return requested json data
 * @access public
 */
router.post("/register", (req, res) => {
    console.log(req.body);
})

module.exports = router;
```

POST中暂时只有一个打印请求体的操作。

在Postman中的Workspace中测试：
```
POST http://localhost:5000/api/users/register
```
Body选择`x-www-form-urlencoded`，在KEY和VALUE中填入测试内容：
```
KEY	     VALUE
email    harlie@google.com
```

查看终端输出：
```
Server running on port 5000
MongoDB connected.
[Object: null prototype] { email: 'harlie@google.com' }
```
说明成功获取到了`req.body`。

## 使用User数据模型
首先安装bcrypt包。bcrypt可以用来加密注册用户的密码，避免在数据库中存储明文密码。在`https://www.npmjs.com/`上可以查看bcrypt包的用法介绍。
```bash
> npm install bcrypt
```

编辑`users.js`引入并使用User数据模型：
```javascript
// @login & registtration
const express = require("express");
const router = express.Router();
const bcrypt = require("bcrypt");

const User = require("../../models/User.js");

/*
 * $route GET /api/users/test
 * @desc return requested json data
 * @access public
 */
router.get("/test", (req,res) => {
    res.json({msg:"Login succeeded!"})
})

/*
 * $route POST /api/users/register
 * @desc return requested json data
 * @access public
 */
router.post("/register", (req, res) => {
    //console.log(req.body);

    // check if email already exists
    User.findOne({ email: req.body.email })
        .then((user) => {
            if (user) {
                return res.status(400).json({ email: "邮箱已被注册！" })
            } else {
                const newUser = new User({
                    name: req.body.name,
                    email: req.body.email,
                    password: req.body.password
                })

                // encrypt newUser password
                bcrypt.genSalt(10, function (err, salt) {
                    bcrypt.hash(newUser.password, salt, (err, hash) => {
                        if (err) throw err;

                        newUser.password = hash;
                        newUser.save()
                            .then(user => res.json(user))
                            .catch(err => console.log(err));
                    });
                });
            }
        })
})

module.exports = router;
```

在Postman中的Workspace中测试`POST http://localhost:5000/api/users/register`。Body选择`x-www-form-urlencoded`，在KEY和VALUE中填入测试内容：
```
email     godfrey@eldenring.com
name      godfrey
password  123456
```

查看测试输出
```json
{
    "name": "godfrey",
    "email": "godfrey@eldenring.com",
    "password": "$2b$10$hoGzFeIdZyCwEotsYhxEheoGNOCE4QnYYh/WkKoGkuPT0xZI9H10C",
    "_id": "62a4482c00990937d819ea6d",
    "date": "2022-06-11T07:45:48.437Z",
    "__v": 0
}
```

打开mongodb，在DATABASES下的node_app中查看，会发现多出了一个`users`的Collection，其中刚好存储了上面我们刚通过POST请求插入的一条数据。

## 使用gravatar处理头像
在`https://www.npmjs.com/package/gravatar`中查看gravatar的使用方法。

安装gravatar
```bash
> npm i gravatar
```

编辑`users.js`增加注册头像（avatar）处理：
```javascript
// @login & registtration
const express = require("express");
const router = express.Router();
const bcrypt = require("bcrypt");
const gravatar = require("gravatar");

const User = require("../../models/User.js");

/*
 * $route GET /api/users/test
 * @desc return requested json data
 * @access public
 */
router.get("/test", (req,res) => {
    res.json({msg:"Login succeeded!"})
})

/*
 * $route POST /api/users/register
 * @desc return requested json data
 * @access public
 */
router.post("/register", (req, res) => {
    //console.log(req.body);

    // check if email already exists
    User.findOne({ email: req.body.email })
        .then((user) => {
            if (user) {
                return res.status(400).json({ email: "Email already registered!" })
            } else {
                const avatar = gravatar.url(req.body.email, { s: '200', r: 'pg', d: 'mm' });

                const newUser = new User({
                    name: req.body.name,
                    email: req.body.email,
                    avatar,
                    password: req.body.password
                })

                // encrypt newUser password
                bcrypt.genSalt(10, function (err, salt) {
                    bcrypt.hash(newUser.password, salt, (err, hash) => {
                        if (err) throw err;

                        newUser.password = hash;
                        newUser.save()
                            .then(user => res.json(user))
                            .catch(err => console.log(err));
                    });
                });
            }
        })
})

module.exports = router;
```

在Postman中的Workspace中测试`POST http://localhost:5000/api/users/register`，Body选择`x-www-form-urlencoded`。

在KEY和VALUE中填入测试内容：
```
email     godfrey@eldenring.com
name      godfrey
password  123456
```

测试会返回报错
```json
{
    "email": "Email already registered!"
}
```

在KEY和VALUE中填入测试内容：
```
email      mohg@eldenring.com
name       mohg
password   123456
```

测试返回
```json
{
    "name": "mohg",
    "email": "mohg@eldenring.com",
    "password": "$2b$10$uSV2tmA5jH6veLTz1Lt5g.iD5QKtbJFXwGsJilDMxIqw7dZefpDz.",
    "avatar": "//www.gravatar.com/avatar/c5515cb5392d5e8a91b6e34a11120ff1?s=200&r=pg&d=mm",
    "_id": "62a44f12d2c5293f0b8e9c2b",
    "date": "2022-06-11T08:15:14.410Z",
    "__v": 0
}
```

在浏览器中打开
```
www.gravatar.com/avatar/c5515cb5392d5e8a91b6e34a11120ff1?s=200&r=pg&d=mm
```
查看默认头像。


