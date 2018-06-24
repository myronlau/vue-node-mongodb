# 搭建Vue+Node+MongoDB的前后端分离项目

## 目的

>使用Vue官方的项目构建工具Vue-Cli构建前端项目，整合Express作为后台，达到前后端分离开发的目的。

## 思路

>1.实现前后端分离，各自开发。这里前后端分离的思路：前端用Vue开发静态页面，路由通过Vue-Router进行，后端用Node仅用于编写API给前端调用获取数据。
>
>2.前端开发时通过**Vue-Cli**中提供的proxyTable进行代理，由此可跨域调用Node编写的API。
>
>3.前后端各自开发完成，测试无误后，前端通过webpack打包压缩，后端拉取前端打包压缩好的文件即部署完成。

## 用到的东西

* Vue-Cli
* Vue-Resource
* Node + Express
* MongoDB

## 步骤
##### 1. 全局安装Vue-Cli
```bash
npm i -g vue-cli
```

##### 2. 搭建并初始化项目
```bash
vue init webpack vue-node-mongodb
```

##### 3. 按提示一直下一步，完成后按提示输入命令
```bash
cd vue-node-mongodb
npm install
npm run dev
```
##### 4. 如无意外，打开: http://localhost:8080 即可看到如下成功启动的页面：

##### 5. 为了直观，把"src"文件夹重命名为"client"，并把webpack.base.conf中的"src"替换为"client"。

##### 6. 开发登录界面
删除HelloWorld.vue，新建Login.vue，编写Login.vue代码
```javascript
<template>
	<div>
		<input class="form-control" id="inputEmail3" placeholder="请输入账号" v-model="account">
		<input type="password" class="form-control" id="inputPassword3" placeholder="请输入密码" v-model="password">
		<button type="submit" class="btn btn-default" @click="login">登录</button>
	</div>
</template>

<script>
	export default {
	data() {
			return {
					account : '',
					password : ''
			}
	},
	methods:{
		login() {
			// 获取已有账号密码
			this.$http.get('/api/login/getAccount')
				.then((response) => {
					// 响应成功回调
					console.log(response)
					let params = { 
						account : this.account,
						password : this.password
					};
					// 创建一个账号密码
					return this.$http.post('/api/login/createAccount',params);
				})
				.then((response) => {
					console.log(response)
				})
				.catch((reject) => {
					console.log(reject)
				});
			}
		}
	}
</script>
```

修改client目录下的App.vue，注释掉：`<img src="./assets/logo.png">`

##### 7. 搭建Node
准备后台(server)目录：

在项目的根目录新建一个叫server的目录，用于放置Node的东西。进入server目录，再新建三个js文件： 
- app.js （入口文件） 
- db.js （设置数据库相关） 
- api.js （编写接口） 
现在整体目录结构是这样的： 

安装Express: 
```bash
npm install --save express
```
安装MongoDB中间件mongoose：
```bash
npm install --save mongoose
```
安装axios：
```bash
npm install --save axios
```

编写入口文件app.js：
```javascript
// 引入编写好的api
const api = require('./api'); 
// 引入文件模块
const fs = require('fs');
// 引入处理路径的模块
const path = require('path');
// 引入处理post数据的模块
const bodyParser = require('body-parser')
// 引入Express
const express = require('express');
const app = express();

app.use(bodyParser.json());
app.use(bodyParser.urlencoded({extended: false}));
app.use(api);
// 访问静态资源文件 这里是访问所有dist目录下的静态资源文件
app.use(express.static(path.resolve(__dirname, '../dist')))
// 因为是单页应用 所有请求都走/dist/index.html
app.get('*', function(req, res) {
    const html = fs.readFileSync(path.resolve(__dirname, '../dist/index.html'), 'utf-8')
    res.send(html)
})
// 监听8088端口
app.listen(8088);
console.log('success listen…………');
```

编写数据库文件db.js：
```javascript
// Schema、Model、Entity或者Documents的关系请牢记，Schema生成Model，Model创造Entity，Model和Entity都可对数据库操作造成影响，但Model比Entity更具操作性。
const mongoose = require('mongoose');
// 连接数据库 如果不自己创建 默认test数据库会自动生成
mongoose.connect('mongodb://localhost/test');

// 为这次连接绑定事件
const db = mongoose.connection;
db.once('error',() => console.log('Mongo connection error'));
db.once('open',() => console.log('Mongo connection successed'));
/************** 定义模式loginSchema **************/
const loginSchema = mongoose.Schema({
    account : String,
    password : String
});

/************** 定义模型Model **************/
const Models = {
    Login : mongoose.model('Login',loginSchema)
}

module.exports = Models;
```

编写后台API api.js：
```javascript
// 可能是我的node版本问题，不用严格模式使用ES6语法会报错
"use strict";
const models = require('./db');
const express = require('express');
const router = express.Router();

/************** 创建(create) 读取(get) 更新(update) 删除(delete) **************/

// 创建账号接口
router.post('/api/login/createAccount',(req,res) => {
    // 这里的req.body能够使用就在index.js中引入了const bodyParser = require('body-parser')
    let newAccount = new models.Login({
        account : req.body.account,
        password : req.body.password
    });
    // 保存数据newAccount数据进mongoDB
    newAccount.save((err,data) => {
        if (err) {
            res.send(err);
        } else {
            res.send('createAccount successed');
        }
    });
});
// 获取已有账号接口
router.get('/api/login/getAccount',(req,res) => {
    // 通过模型去查找数据库
    models.Login.find((err,data) => {
        if (err) {
            res.send(err);
        } else {
            res.send(data);
        }
    });
});

module.exports = router;
```
修改client/router下的index.js，把"HelloWorld"都修改为"Login"

##### 8.至此我们的后端代码就编写好了，进入server目录，敲上 node app命令，node就会跑起来，这时在浏览器输入http://localhost:8088/api/login/getAccount就能访问到这个接口了 

##### 9. 回到前端，尝试请求接口
现在我们点击登录按钮去请求接口，当然还是不行的，因为使用npm run dev 进行开发时，其实webpack会启动一个8080的web服务用于我们进行开发，而我们后端是在8088端口的，所以我们肯定请求不到后端的接口。

修改config/index.js的proxyTable如下：
```javascript
 proxyTable: {
        '/api': {
        target: 'http://localhost:8088/api/',
        changeOrigin: true,
        pathRewrite: {
          '^/api': ''
        }
      }
    }
```
这时，我们在前端接口地址前加上/api，就会指向http://localhost:8088/api/，于是我们就能访问到后端的接口了！让我们来点击一下登录按钮，会发现接口请求成功了！再去数据库看看！也插入了一条新数据！成功！ 

##### 10. 前后端开发完成，最后一步，前端打包，后端部署。 
前端打包就很简单了，一个命令： 
```bash
npm run build
```
这就生成了一个dist目录，里面就是打包出来的东西。

进入server目录，使用命令：
```bash
node app
```
启动之后打开：http://127.0.0.1:8088 就可以看到项目正式运行的状态的了。

##### 到此为止，我们就完成了整个前后端各自开发到正式部署的流程。
