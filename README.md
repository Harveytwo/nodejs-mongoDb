#基于 nodejs 和 mongoDb 的前后端分离开发
目录结构：
<pre>
    ├── controllers     //数据库相关操作目录
    │   ├── base.js     //基础操作方法，此处我是封装了验证前端传来参数是否正确的方法
    │   ├── msgs.js     //相关业务逻辑代码，用于查询 /msgs 相关数据
    │   ├── user.js     //相关业务逻辑代码，用于查询 /users 相关数据
    ├── models          //通过 mongoose 创建数据库表格结构目录
    │   ├── msgs.js     //创建 msgs 表格
    │   ├── user.js     //创建 users 表格
    ├── routers         //路由的相关配置目录
    │   ├── index.js
    ├── schemas         //通过 mongoose 定义表格结构和相关操作和查询方法目录
    │   ├── msgs.js     //定义 msgs 表格的数据结构和相关方法
    │   ├── users.js    //定义 users 表格的数据结构和相关方法
    ├── app.js          //项目的入口文件
    ├── package.json
    ├── README.md
</pre>

##起步：项目入口文件
app.js
```
var express = require('express')
var bodyParser = require('body-parser')
var cookieParser = require('cookie-parser')
var cookieSession = require('cookie-session')
var mongoose = require('mongoose')

var app = express()

app.set('port', process.env.PORT || 3000)

//这里用户登录的账号，直接存到session里了
app.use(cookieParser())
app.use(cookieSession({
    name: 'session',
    secret: 'Harvey',
    keys: ['key1', 'key2'],
    // Cookie Options
    maxAge: 24 * 60 * 60 * 1000 // 24 hours
}))

//通过 mongoose 连接数据库，因数据库加密授权了，需要加上用户名和密码
var dbUrl = 'mongodb://name:password@localhost/test'
mongoose.connect(dbUrl)
var db = mongoose.connection;
db.on('error', console.error.bind(console, 'connection error:'));
db.once('open', function() {
  console.log('mongoose opened!');
});

//使用 body-parser 模块，我们就可以通过 req.body 获取前端传过来的数据
app.use(bodyParser.json())
app.use(bodyParser.urlencoded({ extended: false }))

//设置跨域访问
app.all('*', function(req, res, next) {
    res.header("Access-Control-Allow-Origin", "*");//设置来源域名，*表示所有的都可以请求（危险）
    res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Conten-Type, Accept");
    res.header("Access-Control-Allow-Methods","PUT,POST,GET,DELETE,OPTIONS");
    res.header("X-Powered-By",' 3.2.1')
    res.header("Content-Type", "application/json;charset=utf-8");
    next();
});

//引入 ./routers/index.js 中的相关路由配置
var index = require('./routers/index')
app.use('/', index);

app.listen(3000);
console.log('server 3000');

module.exports = app;
```

##运行项目
> 先安装 supervisor 提高 nodejs 调试效率
> 
> 安装相关依赖 npm isntall
> 
> 建议使用淘宝镜像 cnpm 安装比较快
> 
> 安装完毕，运行项目 supervisor app.js

浏览器输入 localhost:3000
当然这时候还没啥东西的啦！因为还没配置相关路由嘛...

##接下来：配置相关路由
./routers/index.js
```
var express = require('express')
var router = express.Router()
var User = require('../controllers/user')
var Msgs = require('../controllers/msgs')
var Base = require('../controllers/base')

var home = [
    { name: 'home', description: '我是首页', price: 12.12},
    { name: 'home', description: '我是首页', price: 12.12}
]

//先是配置 /，是 get 方式
//运行项目，浏览器输入 localhost:3000 就好返回上面定义的 home 数组了
router.get('/', checkLogin)
router.get('/', function(req, res) {
  res.json(home)
});

//用户注册、登录、注销登录的相关路由
//匹配相关路由，调用controllers 中相关的方法
router.post('/reg', User.showSignup)
router.post('/login', User.showSignin)
router.get('/users', User.showUsers)
router.get('/logout',User.logout)

router.get('/msgs', Msgs.showMsgs)

//用于检测用户是否登录接口
router.get('/checkLogin', function(req, res) {
  if (req.session.user) {
    res.json({code: 1, msg: '已登录', name: req.session.user.name})
  } else {
    res.json({code: -1, msg: '未登录'})
  }
});

function checkLogin(req, res, next) {
    if (!req.session.user) {
      console.log('未登入')
    }
    next();
}
function checkNotLogin(req, res, next) {
    if (req.session.user) {
      console.log('已登入')
    }
    next();
}

module.exports = router
```

##操作数据库
以上只是介绍如何运行项目和路由的相关配置，并且是返回自己写的数据，那如何返回数据库的数据，并对数据进行增删改查呢？

先设计数据库表结构咯
使用 mongoose
在 schemas 目录创建相应的 js 文件
此处就以 uer.js 为例：
```
var mongoose = require('mongoose')

var bcrypt = require('bcryptjs')//一种加密算法，这里用来加密密码
var SALT_WORK_FACTOR = 10

//定义表格结构和相关数据类型
var UserSchema = new mongoose.Schema({
  name: {
    unique: true,
    type: String
  },
  password: String,
  //用户角色
  // 0: nomal user
  // 1: verified user
  // 2: professonal user
  // >10: admin
  // >50: super admin
  role: {
    type: Number,
    default: 0
  },
  meta: {
    createAt: {
      type: Date,
      default: Date.now()
    },
    updateAt: {
      type: Date,
      default: Date.now()
    }
  }
})

UserSchema.pre('save', function(next) {
  var user = this

  if (this.isNew) {
    this.meta.createAt = this.meta.updateAt = Date.now()
  }
  else {
    this.meta.updateAt = Date.now()
  }

  bcrypt.genSalt(SALT_WORK_FACTOR, function(err, salt) {
    if (err) return next(err)

    bcrypt.hash(user.password, salt, function(err, hash) {
      if (err) return next(err)

      user.password = hash
      next()
    })
  })
})
//定义密码的比较方法，登录时需要用到
UserSchema.methods = {
  comparePassword: function(_password, cb) {
    bcrypt.compare(_password, this.password, function(err, isMatch) {
      if (err) return cb(err)

      cb(null, isMatch)
    })
  }
}

//定义相关的查询方法，需要时可以调用
UserSchema.statics = {
  fetch: function(cb) {
    return this
      .find({})
      .sort('meta.updateAt')
      .exec(cb)
  },
  findById: function(id, cb) {
    return this
      .findOne({_id: id})
      .exec(cb)
  }
}

module.exports = UserSchema
```

这里只是定义了表格，但是还没创建，接下来就是创建了
./models/user.js
```
var mongoose = require('mongoose')
var UserSchema = require('../schemas/user')
//创建表格，表名为 users, 默认会加上 s
var User = mongoose.model('User', UserSchema)

module.exports = User
```

表格定义和创建好了，就剩操作了
./controllers/user.js
```
var User = require('../models/user')
var Base = require('./base')

function Users(user) {
    this.name = user.name
}
//注册时调用该方法
exports.showSignup = function(req, res) {
    var _user = req.body
    //调用 ./base.js 中的方法，判断前端传来的参数是否正确
    var _params = ['name', 'password', 'passwordRepeat']
    var check = Base.checkParams(_user, _params)
    if(check === 1) {
        //注册时，判断用户是否已注册
        User.findOne({name:_user.name}, function(err, user) {
            if(err) {
                console.log(err);
            }
            if(user) {
                res.json({msg: '用户已存在', code: -1})
            } else {
                if(_user.password !== _user['passwordRepeat']) {
                    res.json({msg: '两次输入密码不一致', code: -1})
                } else {
                    var user = new User(_user);
                    user.save(function(err,user) {
                        if(err) {
                            console.log(err);
                        } else {
                            res.json({msg: '注册成功', code: 1})
                        }
                    })
                }
            }
        })
    } else {
        res.json(check)
    }
}
//显示总共有多少用户 /users
exports.showUsers = function(req,res) {
    User.find({}, function(err, user) {
        if(err) {
            console.log(err)
        }
        var users = []
        user.forEach(function(doc, index) {
            //实例化 doc 对象，我们只需要取 name 字段
            var user = new Users(doc)
        users.push(user);
        })
        res.json(users)
    })
}
//登录时调用的方法
exports.showSignin = function(req,res) {
    var _user = req.body;
    var _params = ['name', 'password']
    var check = Base.checkParams(_user, _params)
    var name = _user.name;
    var password = _user.password;

    if(check == 1) {
        User.findOne({name:name},function(err,user) {
            if(err) {
                console.log(err);
            }
            
            if(!user) {
                res.json({msg: '用户不存在', code: -1})
                // return res.redirect('/signup');
            }
            user.comparePassword(password,function(err,isMatch) {
                if(err) {
                    console.log(err);
                }
                
                if(isMatch) {
                    //登录成功后，会使用 session 保持登录态，session 的相关使用在 app.js 中有过介绍
                    req.session.user = user;
                    res.json({msg: '登陆成功', code: 1})
                } else {
                    res.json({msg: '密码不正确', code: -1})
                }
            })
        })
    } else {
        res.json(check)
    }
}

//注销登录时调用的方法，清除 session 清除登录态
exports.logout = function(req,res) {
    delete req.session.user;
    res.json({msg: '注销登陆成功', code: 1})
}

```

**至此，我们已经把用户登录、注册、注销登录和查询已注册用户的接口写好了。

接下来就是部署到服务器上了，这个留着下节再讲...
哈哈，我也还没弄懂很多，半桶水，只是简单部署到服务器，但数据库方面和 Nginx 方面还需研究，下次弄懂，再一起补充啦！
