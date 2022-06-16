@[TOC](node.js+postman+mongodb搭建测试登录接口)

# 创建登录路由
编辑users.js添加用户登录的POST请求，使用`bcrypt`中间件来验证用户密码。
```javascript
// @login & registration
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

/*
 * $route POST /api/users/login
 * @desc return token jwt passport
 * @access public
 */
router.post("/login", (req, res) => {
    const email = req.body.email;
    const password = req.body.password;

    User.findOne({ email })
        .then(user => {
            if (!user) {
                return res.status(404).json({ email: "User does not exist!" });
            }

            // compare password
            bcrypt.compare(password, user.password)
                .then(isMatch => {
                    if (isMatch) {
                        res.json({ msg: "success!" });
                    } else {
                        return res.status(404).json({ password: "Wrong password!" });
                    }
                })
        })
})

module.exports = router;
```

在Postman Workspace中测试`POST http://localhost:5000/api/users/login`，Body选择`x-www-form-urlencoded`。在KEY和VALUE中填入测试内容：
```
email     godfrey5@eldenring.com
password  123321
```

Postman测试结果返回
```json
{
    "email": "User does not exist!"
}
```

在KEY和VALUE中填入测试内容：
```
email     godfrey@eldenring.com
password  123321
```

Postman测试结果返回
```json
{
    "password": "Wrong password!"
}
```

在KEY和VALUE中填入测试内容：
```
email     godfrey@eldenring.com
password  123456
```

Postman测试结果返回
```json
{
    "msg": "success!"
}
```

# 生成用户token
接下来要把登录成功时反馈success修改为返回一个用户token。这里用到的第三方中间件为`jsonwebtoken`。

安装中间件
```bash
npm install jsonwebtoken
```

编辑users.js使得用户在登录验证成功后获取到一个token。
```javascript
// @login & registration
const express = require("express");
const router = express.Router();
const bcrypt = require("bcrypt");
const jwt = require('jsonwebtoken');
const gravatar = require("gravatar");
const keys = require("../../config/keys");

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

/*
 * $route POST /api/users/login
 * @desc return token jwt passport
 * @access public
 */
router.post("/login", (req, res) => {
    const email = req.body.email;
    const password = req.body.password;

    User.findOne({ email })
        .then(user => {
            if (!user) {
                return res.status(404).json({ email: "User does not exist!" });
            }

            // compare password
            bcrypt.compare(password, user.password)
                .then(isMatch => {
                    if (isMatch) {
                        // define token generation rules
                        const jwt_rules = { id: user.id, name: user.name };
                        // generate a token which expires in 1hour using jsonwebtoken
                        jwt.sign(jwt_rules, keys.secretOrKey, { expiresIn: 3600 }, (err, token) => {
                            if (err) throw err;
                            res.json({
                                success: true,
                                token: "Bear " + token
                            });
                        });
                        //res.json({ msg: "success!" });
                    } else {
                        return res.status(404).json({ password: "Wrong password!" });
                    }
                })
        })
})

module.exports = router;
```

编辑keys.js增加secretOrKey属性用于jwt的sign函数：
```javascript
module.exports = {
    mongoURI: "mongodb+srv://<username>:<password>@cluster0.ylpaf.mongodb.net/node_app",
    secretOrKey: "secret"
}
```

在KEY和VALUE中填入测试内容：
```
email     godfrey@eldenring.com
password  123456
```

Postman测试结果返回
```json
{
    "success": true,
    "token": "Bear eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjYyYTQ0ODJjMDA5OTA5MzdkODE5ZWE2ZCIsIm5hbWUiOiJnb2RmcmV5IiwiaWF0IjoxNjU1MDIxMTcxLCJleHAiOjE2NTUwMjQ3NzF9.0Nmfckl0I_YK7ivCDXxNQWLODCszezLuylNV-ue6-yA"
}
```

这样用户就拿到了一小时以后过期的token。

# 验证用户token
用户登录以后可以拿token进行一些请求，此时需要对用户token进行验证。

安装第三方中间件：
```bash
npm install passport-jwt passport
```

编辑server.js引入并初始化passport模块：
```javascript
const express = require("express");
const mongoose = require("mongoose");
const bodyParser = require("body-parser");
const passport = require("passport");
const app = express();
const users = require("./routes/api/users"); 

const db = require("./config/keys").mongoURI;

app.use(bodyParser.urlencoded({ extended: false }));
app.use(bodyParser.json());

mongoose.connect(db)
    .then(() => console.log("MongoDB connected."))
    .catch(err => console.log(err));

// passport初始化
app.use(passport.initialize());
//把passport对象传递到passport.js文件中
require("./config/passport")(passport);

//app.get("/", (req, res) => {
//    res.send("Hello World!");
//})

app.use("/api/users", users);

const port = process.env.PORT || 5000;
app.listen(port, () => {
    console.log(`Server running on port ${port}`);
})
```

在config目录下新建`passport.js`并导入相关模块：
```javascript
const JwtStrategy = require('passport-jwt').Strategy,
    ExtractJwt = require('passport-jwt').ExtractJwt;

const mongoose = require("mongoose");
const User = mongoose.model("users");
const keys = require("./keys");

const opts = {}
opts.jwtFromRequest = ExtractJwt.fromAuthHeaderAsBearerToken();
opts.secretOrKey = keys.secretOrKey;

module.exports = passport => {
    passport.use(new JwtStrategy(opts, (jwt_payload, done) => {
        console.log(jwt_payload);
    }));
}
```

编辑users.js增加用户登录后的GET请求token验证（`passport.authenticate`）：
```javascript
// @login & registtration
const express = require("express");
const router = express.Router();
const bcrypt = require("bcrypt");
const jwt = require('jsonwebtoken');
const gravatar = require("gravatar");
const keys = require("../../config/keys");
const passport = require("passport");

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

/*
 * $route POST /api/users/login
 * @desc return token jwt passport
 * @access public
 */
router.post("/login", (req, res) => {
    const email = req.body.email;
    const password = req.body.password;

    User.findOne({ email })
        .then(user => {
            if (!user) {
                return res.status(404).json({ email: "User does not exist!" });
            }

            // compare password
            bcrypt.compare(password, user.password)
                .then(isMatch => {
                    if (isMatch) {
                        // generate token using jsonwebtoken
                        const jwt_rules = { id: user.id, name: user.name };
                        jwt.sign(jwt_rules, keys.secretOrKey, { expiresIn: 3600 }, (err, token) => {
                            if (err) throw err;
                            res.json({
                                success: true,
                                token: "Bear " + token
                            });
                        });
                        //res.json({ msg: "success!" });
                    } else {
                        return res.status(404).json({ password: "Wrong password!" });
                    }
                })
        })
})

/*
 * $route GET /api/users/current
 * @desc return current user info
 * @access private
 */
router.get("/current", passport.authenticate("jwt", {session:false}), (req, res) => {
    res.json({ msg: "success!" });
})

module.exports = router;
```

在Postman Workspace中测试`GET http://localhost:5000/api/users/current`。

Postman测试结果返回
```
Unauthorized
```

在Headers中添加：
```
KEY        	   VALUE
Authorization  Bear eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjYyYTQ0ODJjMDA5OTA5MzdkODE5ZWE2ZCIsIm5hbWUiOiJnb2RmcmV5IiwiaWF0IjoxNjU1MDIxMTcxLCJleHAiOjE2NTUwMjQ3NzF9.0Nmfckl0I_YK7ivCDXxNQWLODCszezLuylNV-ue6-yA
```
其中VALUE是登录时返回的Token。

重新测试会发现Postman一直显示“Sending Request...”。而VS控制台终端则会输出用户信息：
```json
{
  id: '62a4482c00990937d819ea6d',
  name: 'godfrey',
  iat: 1655024119,
  exp: 1655027719
}
```

# 利用token请求数据
编辑passport.js增加查询用户信息操作：
```javascript
const JwtStrategy = require('passport-jwt').Strategy,
    ExtractJwt = require('passport-jwt').ExtractJwt;

const mongoose = require("mongoose");
const User = mongoose.model("users");
const keys = require("./keys");

const opts = {}
opts.jwtFromRequest = ExtractJwt.fromAuthHeaderAsBearerToken();
opts.secretOrKey = keys.secretOrKey;

module.exports = passport => {
    passport.use(new JwtStrategy(opts, (jwt_payload, done) => {
        console.log(jwt_payload);

        //根据用户ID查询信息并返回
        User.findById(jwt_payload.id)
            .then(user => {
                if (user) {
                    return done(null, user);
                }
                return done(null, false);
            })
            .catch(err => console.log(err));
    }));
}
```

编辑users.js修改CURRENT GET请求的返回信息：
```javascript
/*
 * $route GET /api/users/current
 * @desc return current user info
 * @access private
 */
router.get("/current", passport.authenticate("jwt", {session:false}), (req, res) => {
    //res.json({ msg: "success!" });
    res.json({
        id: req.user.id,
        name: req.user.name,
        email: req.user.email
    });
})
```

使用token在Postman Workspace中测试`GET http://localhost:5000/api/users/current`。

Postman返回的查询结果为
```json
{
    "id": "62a4482c00990937d819ea6d",
    "name": "godfrey",
    "email": "godfrey@eldenring.com"
}
```

