# node-wechat

从去年开始微信开发越来越火了，体现在sdk和h5上（h5如果大家想听，可以回复），这里就简单介绍一下sdk开发

既然是noder，那肯定要用nodejs写，不然会被鄙视的。

## node-webot简介

node-webot是老朴几个人建的org，主要开发微信sdk相关的node modules

地址 https://github.com/node-webot/

- wechat 微信公共平台消息接口服务中间件
- weixin-robot 微信公共帐号自动回复机器人 A Node.js robot for wechat.
- wechat-oauth 授权
- wechat-api Wechat API/主动调用API

## wechat sdk开发常用包

- wechat-oauth
- wechat-api（菜单，消息回复等）
- wx_jsapi_sign（js-sdk）

## 创建菜单

wxapi.js

```
var API = require('wechat-api');
var config = require('config');

var menu_config = config.get('wx.wx_menu');
var app_id      = config.get('wx.app_id');
var app_secret  = config.get('wx.app_secret');

//配置
var api = new API(app_id, app_secret);

//测试
function app(){
  api.createMenu(menu_config, function(err, result){
    console.log(result);
  });
}

module.exports = app;

```

把数据放到配置文件中

config/default.json

```
{
  "domain": "xxxxxxxx.com",
  "is_debug": true,
  "mongodb": {
    "host": "127.0.0.1",
    "port": "27017",
    "db": "node-webot"
  },
  "sms": {
    "host": "127.0.0.1",
    "port": "5001"
  },
  "wx": {
    "app_id": "wx04014b02asdsfdsfsf",
    "app_secret": "cc4c224b50fkjlksdafkldfsakljsdfk",
    "wx_menu": {
      "button": [
        {
          "name": "菜单",
          "sub_button": [
            {
              "type": "view",
              "name": "资讯菜单",
              "url": "http://ip:port/all"
            }
          ]
        },
        {
          "name": "菜单2",
          "sub_button": [
            {
              "type": "view",
              "name": "我的11",
              "url": "http://ip:port/all"
            }
          ]
        },
        {
          "name": "助理",
          "sub_button": [
            {
              "type": "view",
              "name": "大学",
              "url": "http://ip:port/all"
            },
            {
              "type": "view",
              "name": "社区",
              "url": "http://ip:port/all"
            }
          ]
        }
      ]
    }
  }
}
```

## oauth

使用微信的用户，或者绑定微信账号

微信有一个唯一的id是openid，它是辨别的唯一标识。


routes/weixin.js

```
// 微信授权和回调
var client = new OAuth(app_id, app_secret);

// 主页,主要是负责OAuth认真
router.get('/', function(req, res) {
  var url = client.getAuthorizeURL('http://' + domain + '/weixin/callback','','snsapi_userinfo');
  res.redirect(url) 
})

/**
 * 认证授权后回调函数
 *
 * 根据openid判断是否用户已经存在
 * - 如果是新用户，注册并绑定，然后跳转到手机号验证界面
 * - 如果是老用户，跳转到主页
 */
router.get('/callback', function(req, res) {
  console.log('----weixin callback -----')
  var code = req.query.code;
  
  var User = req.model.UserModel;

  client.getAccessToken(code, function (err, result) {
    console.dir(err)
    console.dir(result)
    var accessToken = result.data.access_token;
    var openid = result.data.openid;
    
    console.log('token=' + accessToken);
    console.log('openid=' + openid);

    User.find_by_openid(openid, function(err, user){
      console.log('微信回调后，User.find_by_openid(openid) 返回的user = ' + user)
      if(err || user == null){
        console.log('user is not exist.')
        client.getUser(openid, function (err, result) {
          console.log('use weixin api get user: '+ err)
          console.log(result)
          var oauth_user = result;
          
          var _user = new User(oauth_user);
          _user.username = oauth_user.nickname;
          _user.nickname = oauth_user.nickname;
          
          _user.save(function(err, user) {
            if (err) {
              console.log('User save error ....' + err);
            } else {
              console.log('User save sucess ....' + err);
              req.session.current_user = void 0;
              res.redirect('/user/' + user._id + '/verify');
            }
          });
          
        });
      }else{
        console.log('根据openid查询，用户已经存在')
        // if phone_number exist,go home page
        if(user.is_valid == true){
          req.session.current_user = user;
          res.redirect('/mobile')
        }else{
          //if phone_number exist,go to user detail page to fill it
          req.session.current_user = void 0;
          res.redirect('/users/' + user._id + '/verify');
        }
      }
    });
  });
});
```

上面的简单介绍一下

## wx_jsapi_sign(nodejs)

wx_jsapi_sign = wechat(微信) js-api signature implement.

[![npm version](https://badge.fury.io/js/wx_jsapi_sign.svg)](http://badge.fury.io/js/wx_jsapi_sign)

- 支持集群，将获得的前面写到cache.json里
- 使用cache.json保存，比如用redis省事，更省内存
- 足够小巧，便于集成

## Install 

    npm install --save wx_jsapi_sign

## Usage

copy config file

```
cp node_modules/wx_jsapi_sign/config.example.js config.js
```

**change appId && appSecret**

- [微信公众平台如何获取appid和appsecret](http://jingyan.baidu.com/article/6525d4b12af618ac7c2e9468.html)

then mount a route in app.js

```
var signature = require('wx_jsapi_sign');
var config = require('./config')();

....

app.post('/getsignature', function(req, res){
  var url = req.body.url;
  console.log(url);
  signature.getSignature(config)(url, function(error, result) {
        if (error) {
            res.json({
                'error': error
            });
        } else {
            res.json(result);
        }
    });
});
```

more usages see `test/public/test.html`

## Test

微信访问网址  `http://yourserver.com:1342/test`


## 原作者博客

http://blog.xinshangshangxin.com/2015/04/22/%E4%BD%BF%E7%94%A8nodejs-%E8%B8%A9%E5%9D%91%E5%BE%AE%E4%BF%A1JS-SDK%E8%AE%B0%E5%BD%95/


## 我为什么要重写这个模块？

微信签名7200秒可以查看一次，所以就要考虑缓存问题

- wx_jsapi_sign是一个开源项目，使用redis作为缓存，思路挺好，但依赖redis，还是有点重，没必要
- xinshangshangxin写的模块使用cache.json做缓存，然后把获取的时间保存到配置文件里，每次请求只要读取文件即可。这个思路非常好，依赖少，省内存，简单，支持集群

那哥们写的已经很不错了，但是有几个问题

- 他不太了解如何写通用模块（复用程度比较低）
- 代码比较乱（抽取了配置，很好，但想art-template之类的都混在里面看的人顿时头大）
- 没有发布到npm上

如果各位想了解是如何做的，请移步http://www.github.com/i5ting/wx_jsapi_sign

- 依赖管理（区分模块依赖的，还是开发阶段依赖的）
- npmjs.org上如何发布模块

## 配置文件实践

上面的微信的菜单，db连接，还是wx_jsapi_sign都是基于配置的，使用配置文件会有一些问题

比如config.json作为配置文件，你需要在版本控制里放一个config.example.json，然后把config.json放到.gitinore里。

然后每个使用的人，需要先拷贝一个config.example.json为config.json。

这样做的好处

- 敏感信息不能提交上去
- 部署服务器的时候，不会冲突（比如配置文件里有一个is_debug选项，服务器上必须是false，而开发阶段是要用true的）

另外配置文件格式，推荐json和yaml，而不是js，不要在配置文件了写逻辑，虽然没问题，但实为陋习。

## 纠正一下价值观


很多人都认为现在啥模块都有了，随便找一个用就可以了，但这种想法是危险的

- 第一件事儿是思考，而不是想找一个随便xx
- 不要造重复的轮子，除非必要（先到github或相关网站上找对应模块，要有可以修改对应模块源码的能力）
- 反身而诚，乐莫大焉

全文完

欢迎提问，欢迎分享

欢迎关注我的公众号【node全栈】  ![](node全栈-公众号.png)

