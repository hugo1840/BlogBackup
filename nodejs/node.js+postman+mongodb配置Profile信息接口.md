@[TOC](node.js+postman+mongodb配置Profile信息接口)

# 配置数据库地址
编辑keys.js更改数据库连接地址。

在mongoDB中创建一个新的数据库，连接字符串为：
```javascript
module.exports = {
    mongoURI: "mongodb+srv://<username>:<password>@cluster0.ylpaf.mongodb.net/node_vue_element",
    secretOrKey: "secret"
}
```

在Postman中测试注册接口：
```json
{
    "name": "bigsean",
    "email": "bigsean@hotmail.com",
    "password": "$2b$10$y0DBUuwSF.e1ZRjYO4nubeef2XCZnB7nAJFofAPlWssMrnTkTu.e2",
    "avatar": "//www.gravatar.com/avatar/d4dd8a04a74f5380dec36688d7642477?s=200&r=pg&d=mm",
    "identity": "manager",
    "_id": "62ab145b5927ee42876cfb58",
    "date": "2022-06-16T11:30:35.529Z",
    "__v": 0
}
```

# 通过add接口新增记录
编辑profiles.js新增add接口：
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

/*
 * $route POST /api/profiles/add
 * @desc 创建信息接口
 * @access Private
 */
router.post("/add", passport.authenticate("jwt", { session: false }), (req, res) => {
    const profileFields = {};

    if (req.body.type) profileFields.type = req.body.type;
    if (req.body.description) profileFields.description = req.body.description;
    if (req.body.income) profileFields.income = req.body.income;
    if (req.body.expense) profileFields.expense = req.body.expense;
    if (req.body.cash) profileFields.cash = req.body.cash;
    if (req.body.remark) profileFields.remark = req.body.remark;

    new Profile(profileFields).save().then(profile => {
        res.json(profile);
    });

})

module.exports = router;
```

通过login接口获取一个token：
```json
{
    "success": true,
    "token": "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjYyYWIxNDViNTkyN2VlNDI4NzZjZmI1OCIsIm5hbWUiOiJiaWdzZWFuIiwiYXZhdGFyIjoiLy93d3cuZ3JhdmF0YXIuY29tL2F2YXRhci9kNGRkOGEwNGE3NGY1MzgwZGVjMzY2ODhkNzY0MjQ3Nz9zPTIwMCZyPXBnJmQ9bW0iLCJpYXQiOjE2NTUzODAzNTcsImV4cCI6MTY1NTM4Mzk1N30.jce0DDoxRAu23BLQq4RNqqg8FQ1YdeRBfVzYsLU8vQs"
}
```

测试接口`http://localhost:5000/api/profiles/add`，Headers中填写：
```
Authorization	Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjYyYWIxNDViNTkyN2VlNDI4NzZjZmI1OCIsIm5hbWUiOiJiaWdzZWFuIiwiYXZhdGFyIjoiLy93d3cuZ3JhdmF0YXIuY29tL2F2YXRhci9kNGRkOGEwNGE3NGY1MzgwZGVjMzY2ODhkNzY0MjQ3Nz9zPTIwMCZyPXBnJmQ9bW0iLCJpYXQiOjE2NTUzODAzNTcsImV4cCI6MTY1NTM4Mzk1N30.jce0DDoxRAu23BLQq4RNqqg8FQ1YdeRBfVzYsLU8vQs
```

Body中填写相关信息。测试返回：
```json
{
    "type": "groupon",
    "description": "One Ticket Purchase",
    "income": "20",
    "expense": "20",
    "cash": "500",
    "remark": "Ticket for adult",
    "_id": "62ab1b7903586b7bb1850be8",
    "date": "2022-06-16T12:00:57.673Z",
    "__v": 0
}
```

# 获取所有记录
编辑profiles.js，新增获取profile所有信息的接口：
```javascript
/*
 * $route GET /api/profiles
 * @desc 请求所有信息
 * @access Private
 */
router.get("/", passport.authenticate("jwt", { session: false }), (req, res) => {
    Profile.find()
        .then(profile => {
            if (!profile) {
                return res.status(404).json("Null Content!");
            }
            res.json(profile);
        }).catch((err) => res.status(404).json(err));
});
```

测试接口`http://localhost:5000/api/profiles`，在Headers中填写token。

测试返回一个数组：
```json
[
    {
        "_id": "62ab1b7903586b7bb1850be8",
        "type": "groupon",
        "description": "One Ticket Purchase",
        "income": "20",
        "expense": "20",
        "cash": "500",
        "remark": "Ticket for adult",
        "date": "2022-06-16T12:00:57.673Z",
        "__v": 0
    }
]
```

通过add接口插入一条新的记录，再次测试：
```json
[
    {
        "_id": "62ab1b7903586b7bb1850be8",
        "type": "groupon",
        "description": "One Ticket Purchase",
        "income": "20",
        "expense": "20",
        "cash": "500",
        "remark": "Ticket for adult",
        "date": "2022-06-16T12:00:57.673Z",
        "__v": 0
    },
    {
        "_id": "62ab2234f5050a39e759ee10",
        "type": "groupon",
        "description": "Game Purchase",
        "income": "30",
        "expense": "30",
        "cash": "300",
        "remark": "Ghost of Tsushima",
        "date": "2022-06-16T12:29:40.670Z",
        "__v": 0
    }
]
```

如果不勾选Headers中的token信息，返回：
```
Unauthorized
```

# 通过id获取一条记录
编辑profiles.js，新增通过id获取profile一条记录的接口：
```javascript
/*
 * $route GET /api/profiles/:id
 * @desc 请求单个信息
 * @access Private
 */
router.get("/:id", passport.authenticate("jwt", { session: false }), (req, res) => {
    Profile.findOne({_id: req.params.id})
        .then(profile => {
            if (!profile) {
                return res.status(404).json("Null Content!");
            }
            res.json(profile);
        }).catch((err) => res.status(404).json(err));
});
```

测试接口`http://localhost:5000/api/profiles/62ab2234f5050a39e759ee10`，其中profiles斜杠后面是上面返回的任意一条记录的id。

测试返回对应的一条记录：
```json
{
    "_id": "62ab2234f5050a39e759ee10",
    "type": "groupon",
    "description": "Game Purchase",
    "income": "30",
    "expense": "30",
    "cash": "300",
    "remark": "Ghost of Tsushima",
    "date": "2022-06-16T12:29:40.670Z",
    "__v": 0
}
```

假设我们填写的id不存在，会返回类似下面的错误信息：
```json
{
    "stringValue": "\"62ab2234f5050a39e759\"",
    "valueType": "string",
    "kind": "ObjectId",
    "value": "62ab2234f5050a39e759",
    "path": "_id",
    "reason": {},
    "name": "CastError",
    "message": "Cast to ObjectId failed for value \"62ab2234f5050a39e759\" (type string) at path \"_id\" for model \"profiles\""
}
```

# 通过edit接口编辑记录
编辑profiles.js增加根据Id编辑信息的接口：
```javascript
/*
 * $route POST /api/profiles/edit
 * @desc 编辑信息接口
 * @access Private
 */
router.post("/edit/:id", passport.authenticate("jwt", { session: false }), (req, res) => {
    const profileFields = {};

    if (req.body.type) profileFields.type = req.body.type;
    if (req.body.description) profileFields.description = req.body.description;
    if (req.body.income) profileFields.income = req.body.income;
    if (req.body.expense) profileFields.expense = req.body.expense;
    if (req.body.cash) profileFields.cash = req.body.cash;
    if (req.body.remark) profileFields.remark = req.body.remark;

    Profile.findByIdAndUpdate(
        { _id: req.params.id },
        { $set: profileFields },
        { new: true }
    ).then(profile => res.json(profile));

});
```

测试接口`http://localhost:5000/api/profiles/edit/62ab1b7903586b7bb1850be8`，Headers中填写token（如果token过期了，通过login接口重新获取）。

在Body中填写要更新的信息，测试返回：
```json
{
    "_id": "62ab1b7903586b7bb1850be8",
    "type": "stock",
    "description": "Apple",
    "income": "2000",
    "expense": "2000",
    "cash": "10000",
    "remark": "purchase of Apple stock",
    "date": "2022-06-16T12:00:57.673Z",
    "__v": 0
}
```

通过接口`http://localhost:5000/api/profiles`，返回更新后的所有记录：
```json
[
    {
        "_id": "62ab1b7903586b7bb1850be8",
        "type": "stock",
        "description": "Apple",
        "income": "2000",
        "expense": "2000",
        "cash": "10000",
        "remark": "purchase of Apple stock",
        "date": "2022-06-16T12:00:57.673Z",
        "__v": 0
    },
    {
        "_id": "62ab2234f5050a39e759ee10",
        "type": "groupon",
        "description": "Game Purchase",
        "income": "30",
        "expense": "30",
        "cash": "300",
        "remark": "Ghost of Tsushima",
        "date": "2022-06-16T12:29:40.670Z",
        "__v": 0
    }
]
```

# 通过delete接口删除记录
编辑profiles.js添加删除记录的接口：
```javascript
/*
 * $route POST /api/profiles/delete
 * @desc 删除信息接口
 * @access Private
 */
router.delete("/delete/:id",
    passport.authenticate("jwt", { session: false }),
    (req, res) => {
        Profile.findOneAndRemove({ _id: req.params.id }).then(profile => {
             //profile.save().then(profile => res.json(profile));
            res.json(profile);
        })
        .catch(err => res.status(404).json("Deletion Failed!"));
});
```

在Postman中测试`DELETE http://localhost:5000/api/profiles/delete/62ab1b7903586b7bb1850be8`，Headers中填写token。

测试返回被删除的记录：
```json
{
    "_id": "62ab31007fa459f2ee76ba76",
    "type": "stock",
    "description": "Apple",
    "income": "2000",
    "expense": "2000",
    "cash": "10000",
    "remark": " Stock Purchase",
    "date": "2022-06-16T13:32:48.496Z",
    "__v": 0
}
```

通过接口`http://localhost:5000/api/profiles`，返回所有记录：
```json
[
    {
        "_id": "62ab31107fa459f2ee76ba79",
        "type": "stock",
        "description": "TikTok",
        "income": "3000",
        "expense": "3000",
        "cash": "10000",
        "remark": " Stock Purchase",
        "date": "2022-06-16T13:33:04.614Z",
        "__v": 0
    },
    {
        "_id": "62ab34db9a02a321e93a4b7b",
        "type": "stock",
        "description": "Google",
        "income": "5000",
        "expense": "5000",
        "cash": "30000",
        "remark": " Stock Purchase",
        "date": "2022-06-16T13:49:15.786Z",
        "__v": 0
    }
]
```

