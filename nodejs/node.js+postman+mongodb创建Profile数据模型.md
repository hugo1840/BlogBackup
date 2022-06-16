@[TOC](node.js+postman+mongodb创建Profile数据模型)

# User增加identity身份标识
编辑User.js增加身份标识管理identity：
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
    identity: {
        type: String,
		required: true
    },
    date: {
        type: Date,
        default: Date.now
    },
})

module.exports = User = mongoose.model("users", UserSchema);
```

编辑users.js在`register|login|current`请求中增加identity：
```javascript
// @login & registration
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
//router.get("/test", (req,res) => {
//    res.json({msg:"Login succeeded!"})
//})

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
                return res.status(400).json("Email already registered!" )
            } else {
                const avatar = gravatar.url(req.body.email, { s: '200', r: 'pg', d: 'mm' });

                const newUser = new User({
                    name: req.body.name,
                    email: req.body.email,
                    avatar,
                    password: req.body.password,
                    identity: req.body.identity
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
                return res.status(404).json("User does not exist!" );
            }

            // compare password
            bcrypt.compare(password, user.password)
                .then(isMatch => {
                    if (isMatch) {
                        // generate token using jsonwebtoken
                        const jwt_rules = {
                            id: user.id,
                            name: user.name,
                            avatar: user.avatar,
                            identity: user.identify
                        };
                        jwt.sign(jwt_rules, keys.secretOrKey, { expiresIn: 3600 }, (err, token) => {
                            if (err) throw err;
                            res.json({
                                success: true,
                                token: "Bearer " + token
                            });
                        });
                        //res.json({ msg: "success!" });
                    } else {
                        return res.status(404).json("Wrong password!" );
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
    //res.json({ msg: "success!" });
    res.json({
        id: req.user.id,
        name: req.user.name,
        email: req.user.email,
        identity: req.user.identity
    });
})

module.exports = router;
```

# 创建Profile数据模型
在models目录下新增Profile.js：
```javascript
const mongoose = require("mongoose");
const Schema = mongoose.Schema;

// create Schema
const ProfileSchema = new Schema({
    type: {
        type: String
    },
    description: {
        type: String
    },
    income: {
        type: String,
        required: true 
    },
    expense: {
        type: String,
        required: true
    },
    cash: {
        type: String,
        required: true
    },
    remark: {
        type: String
    },
    date: {
        type: Date,
        default: Date.now
    },
})

module.exports = Profile = mongoose.model("profiles", ProfileSchema);
```

# 使用Profile数据模型
在入口文件server.js中引入Profile：
```javascript
const express = require("express");
const mongoose = require("mongoose");
const bodyParser = require("body-parser");
const passport = require("passport");
const app = express();
const users = require("./routes/api/users"); 
const profiles = require("./routes/api/profiles");

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
app.use("/api/profiles", profiles);

const port = process.env.PORT || 5000;
app.listen(port, () => {
    console.log(`Server running on port ${port}`);
})
```

在`/routes/api`目录下创建profiles.js：
```javascript
// @login & registration
const express = require("express");
const router = express.Router();
const passport = require("passport");

const Profile = require("../../models/Profile.js");

/*
 * $route GET /api/profiles/test
 * @desc return requested json data
 * @access public
 */
router.get("/test", (req,res) => {
    res.json({msg:"Profile works!"})
})


module.exports = router;
```

在POSTMAN中测试`http://localhost:5000/api/profiles/test`，测试返回
```json
{
    "msg": "Profile works!"
}
```







