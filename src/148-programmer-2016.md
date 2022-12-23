## 使用 Express.js 构建 Node.js REST API 服务

文/Azat Mardan    译 / 奇舞团

>当下 Web 开发中，瘦客户端和瘦服务端的架构变得越来越流行，瘦客户端一般基于 Backbone.js、AnglersJS、Ember.js 等框架构建，而瘦服务端通常代表着 REST 风格的 Web API 服务。这种模式现在越来越流行，已经有不少网站选择尝试把后端建成服务的形式。本文将带你了解如何使用 Node.js 构建 REST API 服务。

现在开始我们的项目（假定你对 RESTful API 基础已有了解），第一步是安装项目依赖的组件。本文中，我们会使用到 Mongoskin18——一个 MongoDB 操作库，它比原生的 MongoDB Node.js 驱动使用起来方便很多。此外，相比 Mongoose，Mongoskin 更轻量而且是无模式的。

Express.js 是一个对 Node.js 核心 Http 模块进行包装的库。它架构在 Connect 中间件之上，为开发者提供了相当多的便利。有些人可能会拿 Express.js 框架和 Ruby 的 Sinatra 框架进行对比，因为它们的特征都是特别灵活并且可配置性强。

首先，需要创建一个 ch8/rest-express 文件夹（也可以直接下载源码）：

```
$ mkdir rest-express
$ cd rest-express
```

我们知道，Node.js/NPM 提供了多种方式安装依赖，包括：

- 手动一个个安装；

- 把依赖写入到 package.json 文件中；

- 下载并复制模块目录。

为了简单起见，我们这里使用第二种，也就是写到 package.json 文件中。你需要创建一个 package.json 文件，然后把依赖模块相关的部分复制进去，也可以复制下面的整个文件：

```
{
  "name": "rest-express",
  "version": "0.0.1",
  "description": "REST API application with Express, Mongoskin, MongoDB, Mocha and Superagent",
  "main": "index.js",
  "directories": {
     "test": "test"
   },
  "scripts": {
     "test": "mocha test -R spec"
   },
 "author": "AzatMardan",
 "license": "BSD",
 "dependencies": {
 "express": "4.1.2",
 "mongoskin": "1.4.1",
 "body-parser": "1.0.2",
 "morgan": "1.0.1" },
 "devDependencies": {
    "mocha": "1.16.2",
    "superagent": "0.15.7",
    "expect.js": "0.2.0"
  }
}
```

然后，只需要执行一行命令就可以安装应用程序所依赖的模块了：

```
$ npm install
```

完成之后，node_modules 文件夹中会多出几个子文件夹：superagent、express、mongoskin 以及 expect 等。如果你希望更改 package.json 定义的模块版本，请一定要查阅模块的更新日志，获取模块准确的版本号。

### 使用 Mocha 和 Superagent 进行测试

在实现应用之前，我们首先来编写测试用例，用来测试将要实现的 REST API 服务器。在 TDD 模式中，可以借助这些测试用例来创建一个脱离 Node.js 的 JSON REST API 服务器，这里会使用到 Express.js 框架和操作 MongoDB 的 Mongoskin 库。

本文中，我们借助 Mocha 和 SuperAgent 库。这个测试是通过发送 HTTP 请求到服务器执行基本的 CURD 操作。

如果你已经了解 Mocha 的使用，或者希望直接进入 Express.js 应用的实现部分，可以跳过这一小节，或直接在命令行中使用 curl 命令进行测试。

假设我们的环境中已经安装了 Node.js、NPM 和 MongoDB，现在创建一个新文件夹（或者就使用你写测试用例的文件夹）。我们使用 Moncha 作为命令行工具，然后用 Express.js 和 SuperAgent 作为本地库。用下面的命令安装 Mocha CLI （如果不行的话请参考 $ mocha -V），在终端运行下面这行命令：

```
$ npm install -g mocha@1.16.2
```

Expect.js 以及 superagent 在前面的小节中应该已经安装完毕了。

>**提示**：我们可以把 Mocha 库安装到项目文件夹中，这样便可以在不同的项目中使用不同版本的 Mocha，在测试时只需要进入./node_modules/mocha/bin/mocha 目录即可。还有一种更好的办法，就是使用 Makefile。

现在让我们在这个 ch8/rest-express 文件夹中创建一个 test/index.js 文件，它将包含6个测试用例：

- 创建一个新对象；

- 通过对象 ID 检索对象；

- 检索整个集合；

- 通过对象 ID 更新对象；

- 通过对象 ID 检查对象是否更新；

- 通过对象 ID 删除对象。

SuperAgent 的链式函数使发送 HTTP 请求变成一件很容易的事，这里每个用例都会用到。文件从引入依赖模块开始：

```
var superagent = require('superagent')
var expect = require('expect.js')
```

然后，我们开始写第一个测试用例，它被包括在一个用例组（包含描述信息和回调函数）中。测试的思想非常简单，我们创建一系列发送到本地服务器的 HTTP 请求（由 SuperAgent 发出），不同的用例发送到不同的 URL，在请求中会携带一些数据，并把处理逻辑写在请求的回调函数中。TDD 中的多个断言之间的关联非常紧密，就像面包和黄油一样。如果需要严格的测试，可以考虑 BDD 模式，但在此项目中还不需要。

```
describe('express rest api server', function(){
  var id
  it('post object', function(done){
    superagent.post('http://localhost:3000/collections/test')
    .send({ name: 'John',
      email: 'john@rpjs.co'
  })
  .end(function(e,res){
    expect(e).to.eql(null)
    expect(res.body.length).to.eql(1)
    expect(res.body[0]._id.length).to.eql(24)
    id = res.body[0]._id
    done()
   })
})
```

如你所见，我们检查了下面这些内容：

- 返回的错误对象需要为空：eql(null)；

- 响应对象的数组应该含且只含有一个元素：to.eql(1)；

- 第一个响应对象中应该包含一个24字节长度的 _id 属性，它是一个标准的 MongoDB 对象 ID 类型。

我们把新创建的对象 ID 保存到全局变量中，在后面的查找、更新以及产出操作中还会用到，这里有一个用例用来测试检索对象。注意一下，这里 SuperAgent 的请求方法改成了 get()，而且需要在 URL 中包含对象 ID。你可以把 console.log 的注释取消掉，这样就可以在控制台中查看到完整的 HTTP 响应数据：

```
it('retrieves an object', function(done){
  superagent.get('http://localhost:3000/collections/test/'+id)
    .end(function(e, res){
      expect(e).to.eql(null)
      expect(typeofres.body).to.eql('object')
      expect(res.body._id.length).to.eql(24)
      expect(res.body._id).to.eql(id)
      done()
   })
})
```

在测试异步代码中，不要漏掉这里的 done() 函数，否则 Mocha 的测试程序会在收到服务器响应之前结束。

接下来的用例在处理响应返回的 ID 数组时用到了 map() 函数，使它显得更有趣一些。我们使用 contain 方法在这个数组中查找我们的 ID （存在 ID 变量中），它比原生的 indexOf() 方法更加优雅。得到的结果会保留最多不超过10条记录，按照 ID 倒序排序，由于我们的对象刚刚添加进去，所以会在第一条：

```
it('retrieves a collection', function(done){
  superagent.get('http://localhost:3000/collections/test')
    .end(function(e, res){
      expect(e).to.eql(null)
      expect(res.body.length).to.be.above(0)
      expect(res.body.map(function (item){
        returnitem._id
      })).to.contain(id)
      done()
   })
})
```

接下来要测试的是更新对象。通过给 SuperAgent 的请求函数传一个参数对象，向服务端提交一些数据，然后断言这个操作返回的结果是 msg=success：

```
it('updates an object', function(done){
 superagent.put('http://localhost:3000/collections/test/'+id)
    .send({name: 'Peter',
      email: 'peter@yahoo.com'})
    .end(function(e, res){
      expect(e).to.eql(null)
      expect(typeofres.body).to.eql('object')
      expect(res.body.msg).to.eql('success')
      done()
    })
})
```

最后两个用例，一个是还原前面做的修改，另一个是删掉这个对象，和前面两个测试用例的做法非常类似。下面是完整的 ch8/rest-express/test/index.js 文件的代码：

```
var superagent = require('superagent')
var expect = require('expect.js')
 
describe('express rest api server', function(){
  var id
  it('post object', function(done){
    superagent.post('http://localhost:3000/collections/test')
     .send({ name: 'John',
       email: 'john@rpjs.co'
     })
   .end(function(e,res){
     expect(e).to.eql(null)
     expect(res.body.length).to.eql(1)
     expect(res.body[0]._id.length).to.eql(24)
     id = res.body[0]._id
     done()
    })
})
 
it('retrieves an object', function(done){
  superagent.get('http://localhost:3000/collections/test/'+id)
    .end(function(e, res){
      expect(e).to.eql(null)
      expect(typeofres.body).to.eql('object')
      expect(res.body._id.length).to.eql(24)
      expect(res.body._id).to.eql(id)
      done()
     })
})
 
it('retrieves a collection', function(done){
  superagent.get('http://localhost:3000/collections/test')
    .end(function(e, res){
      expect(e).to.eql(null)
      expect(res.body.length).to.be.above(0)
      expect(res.body.map(function (item){
        return item._id
      })).to.contain(id)
      done()
   })
})
 
it('updates an object', function(done){
  superagent.put('http://localhost:3000/collections/test/'+id)
    .send({name: 'Peter',
      email: 'peter@yahoo.com'})
    .end(function(e, res){
      expect(e).to.eql(null)
      expect(typeofres.body).to.eql('object')
      expect(res.body.msg).to.eql('success')
      done()
    })
})
 
it('checks an updated object', function(done){
  superagent.get('http://localhost:3000/collections/test/'+id)
    .end(function(e, res){
      expect(e).to.eql(null)
      expect(typeofres.body).to.eql('object')
      expect(res.body._id.length).to.eql(24)
      expect(res.body._id).to.eql(id)
      expect(res.body.name).to.eql('Peter')
      done()
    })
})
 
it('removes an object', function(done){
  superagent.del('http://localhost:3000/collections/test/'+id)
    .end(function(e, res){
      expect(e).to.eql(null)
      expect(typeofres.body).to.eql('object')
      expect(res.body.msg).to.eql('success')
      done()
     })
  })
})
```

现在我们来运行这个测试，在命令行中运行 $ mocha test/index.js 或者 npm test。不过得到的结果一定是失败，因为服务器还没有启动。

如果你有多个项目，需要使用多个版本的 Mocha，那么你可以把 Mocha 安装到项目目录的 node_modules 文件夹下，然后执行：./node_modules/mocha/bin/mocha ./test。

>**注意**：默认情况下，Mocha 只返回少量的信息。如果需要得到更详细的结果，可以使用 -R <name> 参数 (即：$ mocha test -R spec 或者 $ mocha test -R list)。

### 使用 Express 和 Mongoskin 实现 REST API 服务器

现在我们创建一个 ch8/rest-express/index.js 文件作为程序的入口文件。

首先，当然是引入所有的依赖组件：

```
var express = require('express'),
  mongoskin = require('mongoskin'),
  bodyParser = require('body-parser'),
  logger = require('morgan')
```

Express.js 从 3.x 版本开始，简化了实例化应用的方法，使用下面一行代码来创建一个服务器对象：

```
var app = express()
```

我们使用 bodyParser.urlencoded() 和 bodyParser.json() 两个中间件从响应体中提取参数和数据。通过 app.use() 函数调用这些中间件（这些代码看起来似乎更像是配置语句）：

```
app.use(bodyParser.urlencoded())
app.use(bodyParser.json())
app.use(logger())
```

express.logger() 中间件并不是必需的，它的作用是方便我们监控请求。中间件是 Express.js 和 Connect 中一种非常强大的设计，它使组织和复用代码变得十分简便。

express.urlencoded() 和 express.json() 可以帮我们省去解析 HTTP 响应体的麻烦，Mongoskin 则帮我们实现了用一行代码连接到 MongoDB 数据库：

```
var db = mongoskin.db('mongodb://@localhost:27017/test', {safe:true})
```

>**注意**：如果你需要连接到远程数据库（例如 MongoHQ），需要使用到统一资源标示符（URI），提醒一下，其中不含空格：mongodb://[username:password@]host1[:port1][,host2[:port2],...[,hostN[:portN]]][/[database][?options]]，用你真实的用户名、密码、主机地址、端口号替换其中对应的位置。

下面的语句是一个辅助函数，用来把普通的十六进制字符串转化成 MongoDB ObjectID 数据类型：

```
var id = mongoskin.helper.toObjectID
```

app.param() 方法是 Express.js 中间件的另一种形式。它的作用是当 URL 中出现对应的参数时进行一些操作。在我们这个例子中，当 URL 中出现以冒号开头的 collectionName （在后面的路由规则中能看到）时，我们会选择一个特定的集合：

```
app.param('collectionName', function(req, res, next, collectionName){
  req.collection = db.collection(collectionName)
  return next()
})
```

为了达到更好的用户体验，这里添加一个根路由，用来提示用户在他们访问的 URL 中包含要查找的集合名字：

```
app.get('/', function(req, res, next) {
  res.send('Select a collection, e.g., /collections/messages')
})
```

下面是非常重要的逻辑，它实现了对列表按 _id 属性进行排序，并限制最多只返回10个元素：

```
app.get('/collections/:collectionName', function(req, res, next) {
  req.collection.find({},{
    limit:10, sort: [['_id',-1]]
  }).toArray(function(e, results){
    if (e) return next(e)
    res.send(results)
  })
})
```

不知道你注意到 URL 中出现的:collectionName 字符串没有？它配合之前的 app.param() 中间件，提供了一个 req.collection 对象，我们把它指向数据库中一个特殊的集合。

创建对象的接口（POST /collections/:collectionName）比较容易，因为我们只需要把整个请求传给 MongoDB 就行了。

```
app.post('/collections/:collectionName', function(req, res, next) {
  req.collection.insert(req.body, {}, function(e, results){
    if (e) return next(e)
    res.send(results)
  })
})
```

这种方法，或者叫架构，通常被称作“自由 JSON 格式的 REST API”，因为客户端可以抛出任意格式的数据，而服务器总能进行正常的响应。

检索单一对象的方法比 find() 方法速度更快，但是它们使用的是不同的接口（请注意，前者会直接返回结果对象，而不是句柄）。同样，我们借助 Express.js 的魔法，从:id 中提取到 ID 参数，它被保存在 req.params.id 中：

```
app.get('/collections/:collectionName/:id', function(req, res, next) {
  req.collection.findOne({
    _id: id(req.params.id)
  }, function(e, result){
    if (e) return next(e)
    res.send(result)
  })
})
```

PUT 请求的有趣之处在于，update() 方法返回的不是变更的对象，而是变更对象的计数。同时，{$set:req.body}是一种特殊的 MongoDB 操作（操作名以$符开头），它用来设置值。

第二个参数{safe:true, multi:false}是一个保存配置的对象，它用来告诉 MongoDB，等到执行结束后才运行回调，并且只处理一条（第一条）请求。

```
app.put('/collections/:collectionName/:id', function(req, res, next) {
  req.collection.update({
    _id: id(req.params.id)
   }, {$set:req.body}, {safe:true, multi:false},
   function(e, result){
    if (e) return next(e)
    res.send((result === 1) ? {msg:'success'} : {msg:'error'})
   }
  );
})
```

最后一个，DELETE 请求，它同样会返回定义好的 JSON 格式的信息（JSON 对象包含一个 msg 属性，当处理成功时它的内容是字符串 success，如果处理失败则是编码后的错误消息）：

```
app.del('/collections/:collectionName/:id', function(req, res, next) {
  req.collection.remove({
    _id: id(req.params.id)
   },
   function(e, result){
     if (e) return next(e)
     res.send((result === 1) ? {msg:'success'} : {msg:'error'})
    }
  );
})
```

>**注意**：在 Express.js 中，app.del() 方法是 app.delete() 方法的一个别名。

最后一行代码用来启动服务器，并监听3000端口：

```
app.listen(3000, function(){
   console.log ('Server is running')
})
```

下面是 Express.js 4.1.2 REST API 服务器的完整代码（ch8/rest-express/index.js），供你参考：

```
var express = require('express'),
  mongoskin = require('mongoskin'),
  bodyParser = require('body-parser'),
  logger = require('morgan')
 
var app = express()
 
app.use(bodyParser.urlencoded())
app.use(bodyParser.json())
app.use(logger())
 
var db = mongoskin.db('mongodb://@localhost:27017/test', {safe:true})
var id = mongoskin.helper.toObjectID
 
app.param('collectionName', function(req, res, next, collectionName){
  req.collection = db.collection(collectionName)
  return next()
})
 
app.get('/', function(req, res, next) {
  res.send('Select a collection, e.g., /collections/messages')
})
 
app.get('/collections/:collectionName', function(req, res, next) {
  req.collection.find({}, {limit: 10, sort: [['_id', -1]]})
    .toArray(function(e, results){
      if (e) return next(e)
         res.send(results)
      }
   )
})
 
app.post('/collections/:collectionName', function(req, res, next) {
  req.collection.insert(req.body, {}, function(e, results){
    if (e) return next(e)
    res.send(results)
   })
})
 
app.get('/collections/:collectionName/:id', function(req, res, next) {
  req.collecjtion.findOne({_id: id(req.params.id)}, function(e, result){
    if (e) return next(e)
    res.send(result)
   })
})
 
app.put('/collections/:collectionName/:id', function(req, res, next) {
  req.collection.update({_id: id(req.params.id)},
    {$set: req.body},
    {safe: true, multi: false}, function(e, result){
    if (e) return next(e)
    res.send((result === 1) ? {msg:'success'} : {msg:'error'})
  })
})
 
app.del('/collections/:collectionName/:id', function(req, res, next) {
  req.collection.remove({_id: id(req.params.id)}, function(e, result){
    if (e) return next(e)
    res.send((result === 1) ? {msg:'success'} : {msg:'error'})
  })
})
 
app.listen(3000, function(){
  console.log ('Server is running')
})
```

现在在命令行中执行：

```
$ node .
```

这条命令和 $ node index 是等价的。

然后，新开一个命令行窗口（前面一个不要关），运行测试程序：

```
$ mocha test
```

如果希望得到一个更好看的结果报告，可以运行下面这个命令（参见图1）：

```
$ mocha test -R nyan
```

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/57287045a066f.jpg" alt="图1  谁会不喜欢包含一个Nyan猫的库呢" title="图1  谁会不喜欢包含一个Nyan猫的库呢" />

图1  谁会不喜欢包含一个 Nyan 猫的库呢

如果你确实不喜欢 Mocha 或者 BDD（和 TDD），CURL 是另一种可选方案，它的使用方式如图2所示：

```
curl http://localhost:3000/collections/curl-test
```

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/572870dd6d287.jpg" alt="图2  使用CURL进行一次GET请求" title="图2  使用CURL进行一次GET请求" />

图2  使用 CURL 进行一次 GET 请求

>**注意**：你还可以使用浏览器来发起GET 请求。例如，在服务器开启时通过浏览器访问http://localhost:3000/test。

使用 CURL 发送 POST 请求也十分方便（参见图3）：

```
$ curl -d "name=peter&email=peter337@rpjs.co" http://localhost:3000/collections/
curl-test
```

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/5728716bd2700.jpg" alt="图3  通过CURL发送POST请求后收到的响应" title="图3  通过CURL发送POST请求后收到的响应" />

图3  通过 CURL 发送 POST 请求后收到的响应

发送 DELETE 或 PUT 请求需要使用参数--request NAME，当然不要忘记在 URL 中添加 ID，例如：

```
$ curl --request DELETE http://localhost:3000/collections/curl-test/52f6828a23985a6565000008
```

如果你希望了解 CURL 命令以及参数，这里有一篇不错的教程，很简短，推荐阅读：CURL Tutorial with Examples of Usage（http://www.yilmazhuseyin.com/blog/dev/curl-tutorial-examples-usage/）

本文，我们写的测试代码比应用本身的代码还要多，所以很多人可能懒得使用 TDD。但是相信我，所谓磨刀不误砍柴工，养成使用 TDD 的好习惯能帮你节省大量的时间，而且在越复杂的项目中表现越明显。

你也许会有些疑惑，本文是讲 REST API，为什么我们要花时间来介绍 TDD？答案是，因为 REST API 本身并没有一个可以展示的界面，它是提供给程序（客户端或其他终端）来访问的。所以当需要测试 API 时，我们没有太多选择，要么写一个客户端程序，要么手动使用 CURL 命令（也可以在浏览器的控制台中使用 jQuery 的 $.ajax() 方法）。但其实最好的方法还是使用测试用例，如果我们把逻辑梳理清楚，那么每个用例都像一个小的客户端程序一样。

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/572872209c3f7.jpg" alt="" title="" />
