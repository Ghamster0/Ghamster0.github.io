---
title: Vue Koa开发实战
date: 2019-09-10 16:10:01
tags:
- Node
- Vue
- Docker
---

## 简介

> 参考博客： [全栈开发实战：用Vue2+Koa1开发完整的前后端项目（更新Koa2）](https://molunerfinn.com/Vue+Koa/)
> 前置技能： 具备`Vue`和`Koa`基础知识，了解`Javascript`基础语法（和混乱），了解`Nodejs(npm)`常用操作

本文以新手视角，从零开始逐步构建一个`Vue`+`Koa2`的web应用，项目主要包括以下内容：

- 基于`Vue`组件构建单页面应用，包含登录、用户、管理员视图，由`Vue Router`控制页面跳转
- 使用`Koa`及相关插件提供`API`接口
- `Sequelize`数据库访问
- 基于`json web token`的登录验证
- 配置本地运行、打包docker镜像部署

<!-- more -->

为了简化构建(因为菜)，前端部分使用了`Vue Cli`，`Cli`的本质依旧是使用`Webpack`打包，但提供了一系列针对`Vue`的配置，使构建过程开箱即用；另外`Login.vue`使用了“参考博客”的源码。项目在一些阶段会打tag，并附上源码地址

由于以前从未接触过`nodejs`后台开发，本文可能存在一些局限和错误，欢迎指正

## 创建项目

安装`Nodejs`，建议更换淘宝源，[镜像地址](https://npm.taobao.org/)，指令：
```bash
npm config set registry https://registry.npm.taobao.org
```

安装`Vue`和`Vue Cli`
```bash
npm install vue
npm install -g @vue/cli
# 若使用vue serve和vue build命令需要安装全局扩展
npm install -g @vue/cli-service-global
```

创建项目
```bash
vue create vue-koa
```

新建`server`目录，作为`koa`代码目录，在目录下创建`app.js`作为入口文件，整体目录结构如下：
```bash
.
├── README.md
├── babel.config.js
├── node_modules
│   └── ...
├── package-lock.json
├── package.json
├── public
│   ├── favicon.ico
│   └── index.html
├── server # 后端源码目录
│   └── app.js # 后端入口
└── src # 前端源码目录
    ├── App.vue # vue根组件，main.js中将该组件挂载到index.html中
    ├── assets
    │   └── logo.png
    ├── components
    │   └── HelloWorld.vue
    └── main.js # 前端入口
```
本节源码：[GitHub Tag V0.0](https://github.com/Ghamster0/vue-koa/tree/v0.0)

## 接口定义

项目使用`jwt token`做登录验证，用户登录点击登录时，前端调用*获取token*接口，使用用户名和密码认证，接口返回经`jwt`加密的`token`；随后，前端发送所有请求均携带该`token`作为已登录凭证

按照标准，`token`类型为`Bearer`，对需要权限认证的接口，`request header`设置字段`{Authorization: 'Bearer <token>'}`；对于认证失败的请求，服务器应当返回401，`response header`设置字段`{'WWW-Authenticate': 'Bearer'}`

后端服务运行在3000端口，提供两个接口：

### 获取token

请求参数
```
Method: POST
Api: /api/auth
Body: {username: un, password: pw}
```
返回值
```json
{
  "code": 2000,
  "token": "eyqk"
}
```

### 获取当前用户信息

接口需要携带`token`，请求参数
```
Method: GET
Api: /api/user
```
返回值
```json
{
  "username": "艾广威",
  "roles": [
    "user"
  ],
  "iat": 1567656871,
  "exp": 1567660471
}
```

## 前端页面构建

这一节，将创建一个具有两级导航结构的页面，页面顶部导航栏为一级导航，侧边导航菜单为二级导航。点击顶部导航的菜单项，切换侧边导航菜单；点击侧边导航菜单，切换页面主体内容

项目使用`Vue Cli`构建，在根目录下创建`vue.config.js`,该文件会自动被`Vue Cli`识别。由于没有对`babel`做额外调整，可将`babel.config.js`文件删除

### 引入UI等组件

引入`element ui`组件库，简化页面排版

> 安装方式： `npm i element-ui -S`
> 这里使用全局使用方式，实际项目中建议按需引入，[参考文档](https://element.eleme.cn/#/zh-CN/component/quickstart#an-xu-yin-ru)

```js
/* /src/main.js */
import ElementUI from 'element-ui'
import 'element-ui/lib/theme-chalk/index.css'

Vue.use(ElementUI);
```

引入`Vuejs Logger`, 方便打印log

> 安装方式：`npm install vuejs-logger --save-exact`

```js
/* /src/main.js */
import vueLogger from 'vuejs-logger'

Vue.use(vueLogger)
```

### 引入Vue Router 建立二级路由

> 安装Vue Router，指令：`npm install vue-router`

在`/src`下建立如下目录结构：
```bash
.
├── App.vue # Vue根组件，包含顶部导航栏和一级 router-view 标签
├── assets
│   └── logo.png
├── components
│   ├── pages
│   │   ├── Admin.vue # 管理员视图，包含管理员侧边导航菜单元数据
│   │   ├── Login.vue
│   │   ├── Logout.vue
│   │   ├── User.vue # 用户视图
│   │   ├── admin
│   │   │   └── AC.vue
│   │   └── user
│   │       └── UC.vue
│   └── parts # 公用页面组件
│       ├── PageFooter.vue
│       └── SideMenuContent.vue # 侧边导航+主内容（二级 router-view 标签）
├── main.js
├── router.js # 前端路由配置
└── utils.js
```
页面结构分析：
- `/App.vue`：页面的根组件，定义顶部导航栏（一级导航）、底部页脚。中部是`router-view`标签，提供一级路由切换，如：点击导航栏的“管理员”，导航到`/admin`，`router-view`标签渲染为`/components/pages/Admin.vue`
- `/components/pages/Admin.vue`（`User.vue`类似）：该组件`data`的`menus`属性是一个列表对象，定义了侧边导航菜单的内容；使用`SideMenuContent.vue`模板渲染`menus`，支持二级菜单
- `/components/pages/parts/SideMenuContent.vue`：左侧为侧边导航（二级导航），右侧包含一个二级`router-view`标签，用作二级路由渲染
- `/components/pages/AC.vue` （`UC.vue`类似）：页面主内容，由`SideMenuContent`内的`router-view`渲染
- 更多细节查看本节结束给出的源码

接下来配置`Vue Router`：
```js
/* /src/router.js */
import VueRouter from 'vue-router'
import Logout from './components/pages/Logout.vue'
import Login from './components/pages/Login.vue'
import User from './components/pages/User.vue'
import UC from './components/pages/user/UC.vue'
import Admin from './components/pages/Admin.vue'
import AC from './components/pages/admin/AC.vue'

const routes = [
    {
        path: '/user', component: User,
        children: [
            {
                path: 'info', component: UC
            }
        ]
    },{
        path: '/admin', component: Admin,
        children: [
            {
                path: 'info', component: AC
            }
        ]
    },{
        path: '/login', component: Login
    },
    {
        path: '/logout', component: Logout
    }
];

const router = new VueRouter({
    mode: 'history', //使用history模式，避免url的host和uri之间显示很丑的"#"
    routes: routes
});

export default router
```

在`main.js`中引入`router`：

```js
import VueRouter from 'vue-router'
import router from './router.js'

new Vue({
  router: router,
  render: h => h(App),
}).$mount('#app')
```

> 由于我在`/src/components/parts/SideMenuContent.vue`动态创建了新的组件，需要启用运行时编译

配置启用运行时编译：

```js
/* /vue.config.js */
module.exports = {
    runtimeCompiler: true
}
```

此时运行`npm run serve`，访问8080端口，可以看到如下界面

![基本页面]](https://ghamster.gitee.io/ihservice/Vue_Koa开发实战/前端页面.png)

本节源码：[GitHub Tag V0.1](https://github.com/Ghamster0/vue-koa/tree/v0.1)

## 后端服务搭建

> 安装`koa`，指令：`npm install koa`

后端服务需要实现以下功能：

- 数据库访问
- 一个路由组件，提供*接口定义*章节定义的两个接口，以及接口的访问权限控制
- 一组中间件，负责请求的预处理和后处理

后端目录结构如下：
```bash
.
├── app.js
├── config.js
├── const.js
├── controller
│   ├── auth-controller.js
│   └── user-controller.js
├── middlewares
│   ├── auth
│   │   ├── auth-maker.js
│   │   └── jwt-resolver.js
│   └── error-handler.js
├── router.js # 路由，从controller导入
├── schema # 数据库初始化，及各表定义
│   ├── db.js
│   ├── role.js
│   └── user.js
├── secrets # 敏感信息，应加入.gitignore
│   ├── db.json # 数据库配置
│   └── jwt-key.txt # jwt密钥，纯文本字符串
├── service
│   └── user-service.js
└── utils.js
```

### 配置运行环境

> 应确保删除了`/babel.config.js`，否则会默认被`babel`加载导致启动失败

由于`node`不支持`es6`的`import`语法，这里需要使用`babel`做简单的语法转换。开发环境下使用`@babel/register`运行时转换即可（生产环境会在之后的章节解释），首先安装`@babel/register`

```bash
npm install --save-dev @babel/core @babel/cli @babel/preset-env
npm install --save-dev babel-preset-env
npm install @babel/register --save-dev
```

在根目录下添加`server.dev.js`文件，代码如下：

```js
/* /server.dev.js */
require('@babel/register')({
  'presets': [
    ['env', {
      'targets': {
        'node': true
      }
    }]
  ]
})
require('./server/app.js')
```

在`package.json`中添加启动脚本

```json
{
  "scripts": {
    "serve-koa": "node server.dev.js"
  }
}
```
稍后就可以使用`npm run serve-koa`启动后端服务

### 定义通用中间件

安装`koa-json`和`koa-bodyparser`

```bash
npm install koa-json
npm install koa-bodyparser
```

- `koa-json`：用于自动序列化`ctx.body`中的`Object`对象
- `koa-bodyparser`：用于将`ctx`中的`formData`解析到`ctx.request.body`中

在`main.js`中引入两个中间件，另外简单定义一个打印`hello world`的中间件，代码如下：

```js
import Koa from 'koa'
import koaBodyParser from 'koa-bodyparser'
import json from 'koa-json'
import path from 'path'

const app = new Koa();

app.use(koaBodyParser());
app.use(json());
app.use(async (ctx, next) => {
    ctx.body = { msg: "Hello World", path: ctx.path, method: ctx.method };
    await next();
});

app.listen(3000);
```

使用`npm run serve-koa`启动服务，使用`Postman`测试一下：

![中间件测试](https://ghamster.gitee.io/ihservice/Vue_Koa开发实战/中间件测试.png)

### 连接数据库

> 如果使用8.0以上版本的mysql，sequelize可能会报错，stackoverflow相关[链接](https://stackoverflow.com/questions/50093144/mysql-8-0-client-does-not-support-authentication-protocol-requested-by-server)  
> 解决方式：`ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password'`

项目使用`mysql`数据库存储用户数据，在数据库中创建两张表，`user`和`role`；使用`sequelize`框架进行数据库操作，首先安装`sequelize`和数据库驱动：

```bash
npm install --save sequelize
npm install --save mysql2
```

数据库配置信息以`json`文件格式存放在`/server/secrets/db.json`中，格式如下：

```json
{
    "host": "10.143.53.100",
    "port": 3306,
    "schema": "vueDemo",
    "username": "root",
    "password": "root"
} 
```

在`/server/secrets/config.js`中加载配置文件（同时也加载了`jwt`的密钥，这样做是为了方便后期部署时，将`secrets`目录下的文件存储到`docker`中）：

```js
/* /server/config.js */
import fs from "fs";
import path from 'path'

let secretPath = 'secrets'

export default {
    SECRET: fs.readFileSync(path.resolve(__dirname, secretPath, 'jwt-key.txt')),
    EXP_TIME: '1h',
    DATA_BASE: JSON.parse(fs.readFileSync(path.resolve(__dirname, secretPath, 'db.json')))
}
```

接下来配置`sequelize`并导出数据库上下文对象

```js
/* /server/schema/db.js */
import Sequelize from 'sequelize'
import config from '../config.js'

const dbConfig = config.DATA_BASE;
const sequelize = new Sequelize(`mysql://${dbConfig.username}:${dbConfig.password}@${dbConfig.host}:${dbConfig.port}/${dbConfig.schema}`,
    {
        pool: { //数据库连接池
            max: 5,
            min: 1,
            acquire: 30000,
            idle: 10000
        }
    })

export default sequelize
```

> 安装`uuid`用作自增主键，指令：`npm install uuid`

然后创建`user`表对应的对象（`role`表类似）

```js
/* /server/schema/user.js */
import Sequelize from 'sequelize'
import sequelize from './db.js'
import uuid from 'uuid'

const Model = Sequelize.Model;
class User extends Model { }

User.init({
    id: {
        type: Sequelize.UUID,
        defaultValue: uuid(), // id为空时，使用uuid自动生成主键
        primaryKey: true
    },
    name: {
        type: Sequelize.STRING,
        allowNull: false
    },
    passwd: {
        type: Sequelize.STRING,
        allowNull: false
    }
}, {
        sequelize,
        modelName: 'user'
    })

User.sync().then(() => { console.log('== Table: User init!') }); //初始化数据库，如果表不存在则自动创建

export default User
```

接下来就可以方便地使用`user`和`Role`进行数据库访问

### 实现接口

按照*接口定义*章节的描述，需要实现两个接口，其中`/api/user`需要鉴权，`/api/auth`可在未登录状态访问

整体思路及代码结构如下：

- `/server/middlewares/auth`目录存放权限验证相关代码，`jwt-resolver.js`解析请求的`Header`，解密`authorization`属性得到`User`对象（包括id、name和roles属性），将对象绑定到`Header`的`currentUser`属性。`auth-maker.js`导出一个`check`方法，接受`ctx`对象和一个`requireRole`属性，当`ctx.request.currentUser`不具备`requireRole`时抛出异常；可以将该方法放在需要权限验证的`controller`代码开始处

- `/server/controller`下的两个`controller`对应两个接口

- `/server/middlewares/error-handler.js`拦截所有异常，并为`statusCode`为401的请求设置`response header`->`{ 'WWW-Authenticate': 'Bearer' }`

#### User service

在`user-service.js`中添加以下方法，后面会用到：

```js
/* /server/service/user-service.js */
import User from '../schema/user'
import Role from '../schema/role';
import { ROLE_USER } from '../const';

export default {
    getUser: async (id) => {}, //返回id对应的User对象，如果不存在返回null
    checkUser: async (name, passwd) => {}, //返回name和passwd符合的User对象，不存在则返回null
    getRoles: async uid => { //返回该uid对应user具有的roles，不存在则返回ROLE_USER并更新数据库
        let rolesModel = await Role.findAll({ where: { uid: uid } });
        if (rolesModel.length <= 0) ... //省略更新逻辑
        const roles = [];
        rolesModel.forEach(r => roles.push(r.role))
        return roles
    }
}
```

#### 权限中间件

> 安装`jsonwebtoken`，指令：`npm install jsonwebtoken`

`jwt-resolver.js`解密`Header`的`authorization`字段，得到`user`对象，代码如下：

```js
/* /server/middlewares/auth/jwt-resolver.js */
import jwt from 'jsonwebtoken'
import config from '../../config.js'
import userService from '../../service/user-service.js'

export default async (ctx, next) => {
    let token;
    let authHeader = ctx.header.authorization; //从header中取出token
    if (authHeader) {
        let [authType, jwtToken] = authHeader.split(' '); 
        if (authType.toLowerCase() === 'bearer') {
            try {
                token = jwt.verify(jwtToken, config.SECRET); //使用jwt解密token
                ctx.header.currentUser = token; //将解析得到的user对象绑定到currentUser
            } catch (e) {
                console.log('Unresolved jwt token', e)
            }
        }
    }
    await next();
    // 省略自动更新token相关代码
}
```

`auth-maker.js`导出`check`方法，代码如下：

```js
/* /server/middlewares/auth/auth-maker.js */
export default {
    check: (ctx, requiredRole) => {
        let user = ctx.header.currentUser;
        if(!user){
            ctx.throw(401, "4010::Unauthorized"); // 未登录（提供token）
        }
        if(!user.roles.includes(requiredRole)){
            ctx.throw(401, "4011::PmissionDenied"); // 权限不足，如：roles=['user'], requiredRole='admin'
        }
    }
}
```

#### 配置路由

`auth-controller.js`实现了`/api/auth`接口，访问数据库检验`name`和`passwd`是否合法：

```js
/* /server/controller/auth-controller.js */
import jwt from 'jsonwebtoken'
import config from '../config.js'
import userService from '../service/user-service.js'
import userService from '../service/user-service.js'

export default {
    getAuth: async (ctx, next) => {
        const auth = ctx.request.body;
        const user = await userService.checkUser(auth.name, auth.passwd);
        if(!user){ // name和passwd错误时，抛出异常
            ctx.throw(401, "4010::Username or password error!")
        }
        const roles = await userService.getRoles(user.id) // 获取用户具有的role
        const token = {
            id: user.id,
            name: user.name,
            passwd: user.passwd,
            roles: roles
        }
        ctx.body = { code: 2000, token: jwt.sign(token, config.SECRET, { expiresIn: config.EXP_TIME }) }; // 签名token，返回
    }
}
```

`user-controller.js`与上面类似，只是在入口处进行权限验证：

```js
/* /server/controller/user-controller.js */
import authMaker from '../middlewares/auth/auth-maker.js'
import { ROLE_USER } from '../const.js';

export default {
    getUser: async ctx => {
        authMaker.check(ctx, ROLE_USER) //检验用户是否具有ROLE_USER权限，不满足时抛异常
        ctx.body = ctx.header.currentUser
    }
}
```

> 安装`koa-router`， 指令：`npm install koa-router`

接下来在`router.js`中引入以上两个`controller`，并指定对应的接口：

```js
/* /server/router.js */
import koaRouter from 'koa-router'
import auth from './controller/auth-controller.js'
import user from './controller/user-controller.js'

const router = koaRouter();
router.prefix('/api'); //对所有路由添加'/api'前缀

router.post('/auth', auth.getAuth); // 指定访问'/api/auth'的请求由auth.getAuth方法处理
router.get('/user', user.getUser);

export default router
```

#### 异常捕获

在`error-handler.js`中捕获由中间件或`controller`抛出的异常并处理

```js
/* /server/middlewares/error-handler.js */
import utils from '../utils.js'

export default async (ctx, next) => {
    try {
        await next();
    } catch (e) {
        ctx.status = e.statusCode || e.status || 500; //捕获异常并设置statusCode，默认500
        // '4010::Unauthorized' -> 业务错误代码:4010;错误信息:Unauthorized
        let [code, msg] = e.message.split('::'); 
        ctx.body = utils.errMsg(Number(code), msg);
        switch (ctx.status) {
            case 401: // 对401权限错误设置指示“系统接受认证方式”的header
                ctx.set({ 'WWW-Authenticate': 'Bearer' });
                break;
        }
    }
}
```

#### 引入以上组件

将以上的组件添加到`app.js`中，此时代码看起来应该是这样：

```js
/* /server/app.js */
import Koa from 'koa'
import koaBodyParser from 'koa-bodyparser'
import json from 'koa-json'
import path from 'path'
import errorHandler from './middlewares/error-handler.js'
import jwtResolver from './middlewares/auth/jwt-resolver.js'
import router from './router.js'

const app = new Koa();

app.use(errorHandler)
app.use(koaBodyParser());
app.use(json());
app.use(jwtResolver);
app.use(async (ctx, next) => {
    ctx.body = { msg: "Hello World", path: ctx.path, method: ctx.method };
    await next();
});
app.use(router.routes());

app.listen(3000);
```

> 需要提前创建数据库，但不需要提前创建表

运行`npm run serve-koa`，启动服务，控制台打印：

```bash
> vue-koa@0.1.0 serve-koa D:\pcode\vue-koa
> node server.dev.js

Executing (default): CREATE TABLE IF NOT EXISTS `users` (`id` CHAR(36) BINARY DEFAULT 'fc0870f8-faf7-4f23-9eee-65f869bff791' , `name` VARCHAR(255) NOT NULL, `passwd` VARCHAR(255) NOT NULL, PRIMARY KEY (`id`)) ENGINE=InnoDB;
Executing (default): CREATE TABLE IF NOT EXISTS `roles` (`id` CHAR(36) BINARY DEFAULT 'df463adb-be07-4c6c-9db7-46be31fbf725' , `uid` VARCHAR(255) NOT NULL, `role` VARCHAR(255) NOT NULL, PRIMARY KEY (`id`)) ENGINE=InnoDB;
Executing (default): SHOW INDEX FROM `roles`
== Table: Role init!
Executing (default): SHOW INDEX FROM `users`
== Table: User init!
```

向数据库生成的user表中插入一条记录：

```sql
INSERT INTO `vueDemo`.`users` (`name`, `passwd`) VALUES ('root', 'root');
```

接下来使用postman进行接口测试：

- `/api/auth`

![接口测试1](https://ghamster.gitee.io/ihservice/Vue_Koa开发实战/接口测试1.png)

- `/api/user`

将上一个接口测试返回的token添加到请求`Header`，测试结果如图：

![接口测试2](https://ghamster.gitee.io/ihservice/Vue_Koa开发实战/接口测试2.png)

本节源码：[GitHub Tag V0.2](https://github.com/Ghamster0/vue-koa/tree/v0.2)

## 前后端对接

在*前端页面构建*章节中，我们实现了基本的页面跳转逻辑；本节将在此基础上，对接后端服务，实现登录验证

前端登录流程：

- 用户在登录界面点击登录，前端将用户名和密码发送给`/api/auth`接口获取token
- 将获取到的token存储到浏览器的`sessionStorage`
- 前端访问后端接口的请求都携带该token
- 设置路由守卫，跳转到受保护路由时检测`sessinStroge`，若token无效则跳转到登录界面

项目使用`fetch`发送`http`请求，为了确保所有请求均携带token，并能响应token过期、无效等情况，可以对`fetch`做简单的封装放到`utils.js`中

另外，前后端分离会导致跨域问题，简单来说：假设前端服务运行在`localhost:8080`，后端服务运行在`localhost:3000`端口，由于浏览器中的页面是由8080端口的前端服务返回，那么页面的`js`代码只能发送到`localhost:8080`的请求，在页面中调用3000端口的`api`属于跨域。解决这个问题的方式总体有三种：

1. 声明允许跨域
2. 使用反向代理代理转发，如使用`nginx`将发送到8080端口，`/api`前缀的请求转发到3000端口，使得在浏览器“看来”请求并没有跨域
3. 消除跨域，将前端代码打包成静态文件，挂到后端服务下

这里采用第二种，在前端定义一个代理，转发`/api`前缀的请求，在`vue.config.js`中添加：

```js
/* /vue.config.js */
module.exports = {
    devServer: {
        proxy: {
            '/api': {
                target: 'http://localhost:3000/',
                changeOrigin: true
            }
        }
    }
}
```

### 登录验证

登录验证逻辑在`Login.vue`中，部分代码如下：

```js
/* /src/components/pages/Login.vue */
import utils from "../../utils.js"

export default {
  data() {...}, //定义account, password, targetUrl(=this.$route.query.targetUrl)
  methods: {
    login() {
      let auth = { //绑定到form表单的数据
        name: this.account,
        passwd: this.password
      };
      fetch("/api/auth", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(auth)
      })
        .then(res => res.json())
        .then(res => {
          if (res.code === 2000) {
            utils.saveToken(res.token); //将token保存到sessionStroge
            this.onAuthSuccess();
          } else {
            this.onAuthFail();
          }
        });
    },
    onAuthSuccess() {
      if (this.targetUrl) { //判断用户是直接访问登录页还是被重定向登录页
        this.$router.push({ path: this.targetUrl });
      } else {
        this.$router.push({ path: "/" });
      }
    }，
    onAuthFail() {...}
  }
};
```



注销登陆非常简单，只需清空`sessionStroge`即可

### 路由守卫

在`router.js`中，添加路由守卫，在路由跳转前判断路由是否受保护，以及`sessionStroge`中是否存储了有效token

```js
/* /src/router.js */
import utils from './utils.js'

router.beforeEach((to, from, next) => {
    if (to.path === '/login' || utils.vaildToken()) {
        next();
    } else {
        // targetUrl记录当前url，以便登录成功后跳转会当前页面
        next({path: '/login', query: {targetUrl: to.fullPath}})
    }
});

/* /src/utils.js */
function vaildToken() {
    const token = sessionStorage.getItem('access-token');
    const exp = sessionStorage.getItem('exp');
    return token && (Date.now() < exp * 1000) ? true : false;
}

export default {
    vaildToken: vaildToken
}
```

### 封装fetch

这部分主要做三件事：发送请求前将token设置到header；收到响应后判断是否需要更新本地token；若请求失败，生成错误提示。`wrappedFetch`部分代码如下：

```js
/* /src/utils */
async function wrappedFetch(resource, init) {
    let token = getToken();
    if (token) {
        init.headers.Authorization = 'Bearer ' + token; //添加header
    }
    let res = await fetch(resource, init);
    let r = await res.clone().json();
    if (res.ok) {
        if (r.ut) { //如果ut(updateToken)字段非空，则更新本地token
            saveToken(r.ut);
        }
        return res;
    } else {...} //处理请求失败的情况
}

export default {
    wrappedFetch: wrappedFetch
}
```

在`AC.vue`中，当点击refresh按钮时，使用`wrappedFetch`访问`/api/auth`接口，刷新user数据：

```js
/* /src/components/pages/admin */
export default {
  methods: {
    refreshUser() {
      utils
        .wrappedFetch("/api/user", { methods: "GET" })
        .then(res => res.json())
        .then(res => {
          this.user = res;
        })
        .catch(e => this.$log.info("Server error", e));
    }
  }
};
```

登录并访问`http://localhost:8080/admin/info`，点击refresh，测试结果如下：

![前端调用接口测试](https://ghamster.gitee.io/ihservice/Vue_Koa开发实战/前端调用接口测试.png)

点击“用户中心”->“退出登陆”确认功能正常

本节源码：[GitHub Tag V0.3](https://github.com/Ghamster0/vue-koa/tree/v0.3)

## 打包部署

开发环境下，分别为前后端启动服务，可以方便地使用模块热重载特性（`vue cli`默认支持，`koa`可以使用`nodemon`实现），有助于快速开发。生产环境下，将前端构建成静态文件，挂载到被`babel`转换过的后端代码下，可以提供更好的性能。

### 项目构建

配置`vue.config.js`，设置`vue cli`构建参数：

```js
/* /vue.config.js */
module.exports = {
    outputDir: 'dist/dist', // 构建输出目录，将后端转换后的代码放在dist下，将前端构建后的代码放在后端的dist下
    assetsDir: 'assets' // 提取asset到单独文件夹
}
```

在项目下根目录下添加`server.babelrc`，作为后端`babel`转换的配置文件，转换目标为`node`：

```babelrc
{
    "presets": [
        [
            "@babel/preset-env",
            {
                "targets": {
                    "node": true
                }
            }
        ]
    ]
}
```

配置`package.json`中的启动脚本：

- `serve-vue` 开发环境启动前端
- `serve-koa` 开发环境启动后端
- `build` 构建前端
- `compile` 转换后端
- `buildAll` 构建前后端
- `start` 生产环境启动项目

```json
{
  "scripts": {
    "serve-vue": "vue-cli-service serve --port 80",
    "serve-koa": "node server.dev.js",
    "build": "vue-cli-service build",
    "compile": "babel server -d dist --config-file ./server.babelrc --copy-files",
    "buildAll": "npm run compile && npm run build",
    "start": "cd dist && node app.js",
    "lint": "vue-cli-service lint"
  }
}
```

> 安装`koa static`，指令：`npm install koa-static`

还需要在后端代码中使用`koa-static`配置静态资源服务器，当所有路由匹配失败时尝试加载静态资源

> 安装`histroy api fallback`，指令：`npm install koa2-history-api-fallback`

另外由于前端使用了`Vue Router`的history路由模式，形如`/login`的请求（hash模式下对应为`/index.html# /login`）是无法找到对应的静态资源的。该请求的本质是请求`/index.html`页面，然后执行前端路由`router.push('/login')`。所以需要添加`historyApiFallback`，将所有未匹配到后端路由的（前端）路由映射到`index.html`

代码如下：

```js
/* /server/app.js */
...
app.use(router.routes());
// 一定放在router之后
app.use(historyApiFallback());
app.use(serve(path.resolve('dist')));

app.listen(3000);
```

至此，全部配置就完成了，然后我们运行`npm run buildAll && npm run start`，访问`localhost:3000/login`，不出意外会看到以下界面：

![页面未渲染](https://ghamster.gitee.io/ihservice/Vue_Koa开发实战/页面未渲染.png)

这是因为，`Koa`的默认返回`Content-Type`是`application/json`，而`koa static`未能正确设置该属性。我们可以使用`mime-types`识别资源类型，手动设置`Content-Type`

> 安装`mime-types`，指令：`npm install mime-types`

在`app.js`中添加一个中间件：

```js
/* /server/app.js */
app.use(historyApiFallback());
app.use(async (ctx, next)=>{
    await next();
    ctx.set('content-type', mime.lookup(path.join('dist', ctx.path)) || 'text/html');
})
app.use(serve(path.resolve('dist')));
```

重新构建并运行，即可看到正确的页面

### docker构建

`/server/secrets`下存储了数据库和`jwt`密钥等敏感信息，应当添加到`.gitignore`中，避免上传到github；同时我们不希望打包好的docker镜像中包含这些信息，而是从`docker secret`中加载。

项目的依赖可以分为运行时依赖和开发环境依赖，为了使最终的镜像只包含运行时依赖，以及避免每次构建重新安装依赖，我们需要分别打包构建环境、运行时环境镜像，并使用两阶段构建生成最终镜像

#### 存储敏感信息

首先在项目部署的docker服务器上，使用`docker secret`存储敏感信息。可以使用`docker secret create`命令，参见[docker文档](https://docs.docker.com/engine/swarm/secrets/)，或使用`Portainer`等工具。

以`Portainer`为例（截图只做演示，文件名参考上文）：

![创建secret](https://ghamster.gitee.io/ihservice/Vue_Koa开发实战/创建secret.gif)

secret会以文件的形式保存，在`docker-compose.yml`中指定使用后，会挂载到容器的`/run/secrets`下。接下来，修改后端的`config.js`，当运行环境为docker时，从`docker secrets`中加载这些配置：

```js
//
let secretPath = 'secrets'
if (process.env.ENV === 'docker') {
    secretPath = '/run/secrets'
}

export default {
    SECRET: fs.readFileSync(path.resolve(__dirname, secretPath, 'jwt-key.txt')),
    ...
}
```

#### 打包docker镜像

> 为了防止copy命令拷贝`dist`、`node_modules`等目录下的文件，添加`.dockerignore`文件

1. 将构建环境打包为单独的镜像，指令及`build.Dockerfile`：

```bash
docker build -t vue-koa-build-env:latest -f ./dockerfiles/build.Dockerfile .
```
```Dockerfile
FROM node:lts-alpine
WORKDIR /build
COPY package*.json ./
RUN npm install
```

2. 将运行环境打包为单独的镜像，指令及`runtime.Dockerfile`：

```
docker build -t vue-koa-runtime-env:latest -f ./dockerfiles/runtime.Dockerfile .
```
```Dockerfile
FROM node:lts-alpine

WORKDIR /app
COPY package*.json ./
RUN npm install --production
```

3. 打包项目镜像，指令及`Dockerfile` ：

```
docker build -t vue-koa:latest -f ./dockerfiles/Dockerfile .
```
```Dockerfile
FROM vue-koa-build-env:latest # stage0: 基于构建环境，构建项目
WORKDIR /build
COPY . .
RUN npm run buildAll

FROM vue-koa-runtime-env:latest # stage1: 基于运行环境，拷贝stage0的构建结果
WORKDIR /app
COPY --from=0 /build/dist ./dist

ENV ENV="docker" # 设置环境变量
EXPOSE 3000
CMD ["npm", "run", "start"] # 启动
```

4. 添加`docker-compose.yml`，配置加载的secrets，部分配置如下：

```yml
services:
  vue_koa:
    secrets:
      - db.json
      - jwt-key.txt

secrets:
  db.json:
    external: true
  jwt-key.txt:
    external: true
```

最后，在compose文件所在目录，执行`docker-compose up -d`即可启动服务

本节源码：[GitHub Tag V0.4](https://github.com/Ghamster0/vue-koa/tree/v0.4)

## 写在最后

之前刚完成的一个项目，使用了`Flask+Jinja2+JQuery`的技术栈，写的很不开心：模板渲染+`ajax`混用导致代码有些凌乱；缺失`ioc&aop`；`Flask`没有异步非阻塞……于是下一个项目选型的时候，我打算用`SpringBoot+Vue`，但方案被领导驳回，要求使用`nodejs`，于是就有了这篇新手向博客

蓦然想起当初面试百度的时候面试官的一句话：“语言不重要，重要的是...”，这句话的潜台词是“都得会！”。当然无论是用Java、Python还是JavaScript写Web，思想都是相通的，无非是不同语言提供了不同的特性

但是啊，曾经沧海难为水，当年用Spring那一套时其实没有觉得哪里便捷了，面试问起来也无非就会说个AOP、IOC，至于好在哪里，始终一知半解，直到有一天离开了这生态。之前在知乎吐槽Js没有大型成熟的后端框架，有回复“Nestjs了解一下”，我还真的去了解了一下，IOC怎么看怎么怪，AOP完善程度被Spring按在地上摩擦……加上ts的语法……怎么说呢，ts是我见过最诡异最反直觉的语法，比golang都严重

还用就是鸭子型语言用多了，真的挺怀念Java的……可能也就怀念下，万一回去了，大概又不习惯冗长的语法了，谁知道呢

说到底，语言不过是个工具……