## 当微软牛津计划遇到微信 App——微信实现部分



文/王豫翔

>微软牛津计划（Project Oxford）提供了一系列机器学习 API，包含计算机视觉、语音识别和语言理解等认知服务，它能为微信开发带来怎样有趣的功能？请看本文分解。

微软牛津计划提供了一组基于 Rest 架构的 API 和 SDK 工具包，帮助开发者轻轻松松使用微软的自然数据理解能力为自己的解决方案增加智能服务。利用微软牛津计划构建你自己的解决方案，支持任意语言及任意开发平台。主要提供了四个自然语言处理方面的核心问题解决方案：人脸识别、语音识别、计算机视觉，以及语言理解智能服务。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d552abebac.jpg" alt="图1 应用界面" title="图1 应用界面" />

图1 应用界面

微软提供了这么强大的 API，我第一时间就想，是不是可以迁移到微信平台上去做一些好玩的应用，不过在这之前，我没有做过任何微信开发的工作，所以本篇文章将分享整个实现的经验。

### ASP.NET WebAPI 实现微信接入验证

首先你需要一个微信公众号，很重要的是你需要完成认证，这点非常重要。当你完成公众号的基本设定后，我们需要为开发做第一件事情：让微信验证通过开发者中心页配置的服务器地址。微信服务器将发送 GET 请求到我们注册的服务器地址 URL 上，GET 请求携带四个参数：signature、timestamp、nonce、echostr。我们编写了一个 WebAPI 对微信的请求进行反馈。

```
public HttpResponseMessage Get(string signature, string timestamp, string nonce, string echostr)
{
    string[] ArrTmp = { TOKEN, timestamp, nonce };
    Array.Sort(ArrTmp);
    string tmpStr = string.Join("", ArrTmp);
    var result = FormsAuthentication.HashPasswordForStoringInConfigFile(tmpStr, "SHA1").ToLower();
 
    return new HttpResponseMessage (){ 
        Content = new StringContent(result, Encoding.GetEncoding("UTF-8"), "application/x-www-form-urlencoded") 
    };
}
```

上面这段代码的要点是返回值，很多工程师在使用 WebAPI 返回给微信验证时一直失败，是因为忽略了返回值的编码要求是 application/x-www-form-urlencoded。

### ASP.NET WebAPI 实现微信 JS-SDK 接口注入权限验证配置

我们的客户端采用微信的 JS-SDK，但是所有需要使用 JS-SDK 的页面必须先注入配置信息，否则将无法调用，当使用 JS-SDK 的时候，微信会将 appId、timestamp、nonceStr 和 signature 的参数进行加密和验算是否正确，所以我们需要提供一个正确的签名值。需要获得这个签名必须要完成两步，图2所示的 UML 描述了这个过程。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d553584deb.jpg" alt="图2 获取签名过程示意图" title="图2 获取签名过程示意图" />

图2 获取签名过程示意图

第一步：获取 Access Token

```
if (HttpRuntime.Cache["access_token"] == null)
{
    var queryString = HttpUtility.ParseQueryString(string.Empty);
    queryString["grant_type"] = "client_credential";
    queryString["appid"] = APPID;
    queryString["secret"] = APPSECRET;
 
    var uri = "https://api.weixin.qq.com/cgi-bin/token?" + queryString;
 
    HttpResponseMessage response;
    response = await client.GetAsync(uri);
    var msg = await response.Content.ReadAsStringAsync();
    var jsonobj = Newtonsoft.Json.Linq.JObject.Parse(msg);
 
    HttpRuntime.Cache.Add("access_token",
        (string)jsonobj["access_token"],
        null,
        DateTime.Now.AddMinutes((int)jsonobj["expires_in"]),
        new TimeSpan(0, 0, 0),
        System.Web.Caching.CacheItemPriority.AboveNormal,
        null
        );
}
```

第二步：获取 jsapi_ticket。

```
if (HttpRuntime.Cache["jsapi_ticket"] == null)
{
    var queryString = HttpUtility.ParseQueryString(string.Empty);
    queryString["access_token"] = (string)HttpRuntime.Cache["access_token"];
    queryString["type"] = "jsapi";
    var uri = "https://api.weixin.qq.com/cgi-bin/ticket/getticket?" + queryString;
    HttpResponseMessage response;
    response = await client.GetAsync(uri);
    var msg = await response.Content.ReadAsStringAsync();
    var jsonobj = Newtonsoft.Json.Linq.JObject.Parse(msg);
    HttpRuntime.Cache.Add("jsapi_ticket",
    (string)jsonobj["ticket"],
    null,
    DateTime.Now.AddMinutes((int)jsonobj["expires_in"]), 
    new TimeSpan(0, 0, 0), 
    System.Web.Caching.CacheItemPriority.AboveNormal, 
    null
   );
}
```

我们用于签名的素材都到齐了，我们要实现签名算法了。实现的代码如下。

```
var pwd = string.Format("jsapi_ticket={0}&noncestr={1}×tamp={2}&url={3}",
    (string)HttpRuntime.Cache["jsapi_ticket"],
    noncestr,
    timestamp,
    url
    );
 
var tmpStr = FormsAuthentication.HashPasswordForStoringInConfigFile(pwd, "SHA1");
 
return Request.CreateResponse(HttpStatusCode.OK, tmpStr);
```

这时候我们前端的 HTML5 就可以正确的采用 JS-SDK 了。

### ASP.NET 获取微信客户端上传的图片

本来我以为这是个很简单的事情，后来才发现，使用微信 JS-SDK 的时候，微信的 HTML5 客户端不会将图片直接 POST 给我的服务端，而是先提交给微信服务器，然后我的服务端需要通过 serverId 来获得图片，大致的流程我绘制了 UML，见图3，大家可以理解下。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d5542c215a.jpg" alt="图3 获取微信客户端上传图片的过程" title="图3 获取微信客户端上传图片的过程" />

图3 获取微信客户端上传图片的过程

目前我们只关心服务器这段，我们将得到客户端传来的 serverID，从微信的服务器上下载图片到本地。我们实现的代码如下。

```
public async Task<string> Get(string mediaid)
{
    var queryString = HttpUtility.ParseQueryString(string.Empty);
    queryString["access_token"] = await Get();
    queryString["media_id"] = mediaid;
 
    var uri = "http://file.api.weixin.qq.com/cgi-bin/media/get?" + queryString;
 
    HttpResponseMessage response;
    response = await client.GetAsync(uri);
 
    var msg = await response.Content.ReadAsStreamAsync();
    var file = response.Content.Headers.ContentDisposition.FileName.Replace("\"", "");
    var helper = new ProjecToxfordClientHelper();
    var content = await FileHelper.ReadAsync (msg);
    FileHelper.SaveFile(content, file);
    return file;
}</string>
```

好了，到了现在，我们对微信服务器需要实现的接口都差不多了，接下来就可以设计微信的客户端了。

### WeUI 设计微信客户端首页样式

WeUI 是一套同微信原生视觉体验一致的基础样式库，由微信官方设计团队为微信网页开发量身设计，可以令用户的使用感知更加统一。在微信网页开发中使用 WeUI，有如下优势：

- 同微信客户端一致的视觉效果，令所有微信用户都能更容易地使用你的网站；

- 便捷获取快速使用，降低开发和设计成本；

- 微信设计团队精心打造，清晰明确，简洁大方。

该样式库目前包含 button、cell、dialog、progress、toast、article、icon 等各式元素，我们可以在https://github.com/weui/weui获得源代码和 DEMO。

我们先做首页，以了解 WeUI 样式库的使用方式。

建立 Index.html 引入样式库和配置 head 节点。

```
<meta charset="utf-8">
<title>脸探</title>
<meta name="viewport" content="initial-scale=1.0,user- scalable=no,maximum- scale=1,width=device-width"> 
 <link href="css/weui.css" rel="stylesheet">
 <link href="css/example.css" rel="stylesheet">
```

Body 节点下的直接子元素是<div class="page">，其他所有元素都在这个节点下，我们的 Index.html 页面设计了两个 div 元素分别是：

```
<div class="hd">
<div class="bd"></div></div>
```

hd 节点下的代码非常简单，就是 title 的描述。

```
<h1 class="page_title">脸探</h1>
<p class="page_desc">测测脸的相似度</p>
```

bd 包含的是一个`div class="weui_panel weui_panel_access"`，WeUI 提供的 Panel 非常容易设计图文组合列表，WeUI 提供了一系列很有用的类：weui\_panel、weui\_panel\_access、weui\_panel\_hd、weui\_panel\_bd。

我们的 Panel 的标题就可以用 weui\_panel\_hd 进行修饰。

```
<div class="weui_panel_hd"></div>
```

具体的内容可以被 weui\_panel\_bd 修饰。

```
<div class="weui_panel_bd"></div>
```

weui\_panel\_bd 的子元素如下：

```
<a href="" class=" weui_media_box weui_media_appmsg ">
    <div class=" weui_media_hd">
 <imgsrc="img 4432144_111855038929_2.jpg"="" alt="">
    </imgsrc="img></div>
    <div class=" weui_media_bd">
    <h4 class=" weui_media_title ">标题</h4>
    <p class="weui_grid_label">内容 </p>
  </div>  
</a>
```

了解了如何布局一个列表项，那首页就容易完成了，代码如下。

```
<div class="page">
   <div class="hd">
      <h1 class="page_title">脸探</h1>
      <p class="page_desc">测测脸的相似度</p>
   </div>
   <div class="bd">
      <div class="weui_panel weui_panel_access">
       <div class="weui_panel_hd">娃像谁     </div>
       <div class="weui_panel_bd">
          <a href="family.html" class="weui_media_box weui_media_appmsg">
             <div class="weui_media_hd">
                 <img class="weui_media_appmsg_thumb" src="fonts/family.jpg" alt="">
             </div>
             <div class="weui_media_bd">
                 <h4 class="weui_media_title">三人照</h4>
                 <p class="weui_media_desc">上传一家三口三人照，立即知道孩子与父母相像指数</p>
             </div>
         </a>
         <a href="family3.html" class="weui_media_box weui_media_appmsg">
           <div class="weui_media_hd">
              <img class="weui_media_appmsg_thumb" src="fonts/one.jpg" alt="">
           </div>
           <div class="weui_media_bd">
              <h4 class="weui_media_title">单人照</h4>
              <p class="weui_media_desc">上传一家三口各自照片，立即知道孩子与父母相像指数</p>
             </div>
           </a>
         </div>
       </div>
    </div>
    <div class="bd">
       <div class="weui_panel weui_panel_access">
       <div class="weui_panel_hd">夫妻相    </div>
    <div class="weui_panel_bd">
       <a href="couple2.html" class="weui_media_box weui_media_appmsg">  
         <div class="weui_media_hd">
            <img class="weui_media_appmsg_thumb" src="fonts/couple.jpg" alt="">
        </div>
       <div class="weui_media_bd">
         <h4 class="weui_media_title">双人照</h4>
         <p class="weui_media_desc">上传你和TA的双人照，你立即知道你们的天生缘分指数</p>
     </div>
   </a>
   <a href="couple.html" class="weui_media_box weui_media_appmsg">
     <div class="weui_media_hd">
       <img class="weui_media_appmsg_thumb" src="fonts/one.jpg" alt="">
     </div>
     <div class="weui_media_bd">
       <h4 class="weui_media_title">单人照</h4>
           <p class="weui_media_desc">上传你们两人各自照片，你立即知道你们的天生缘分指数</p>
         </div>
        </a>
      </div>
    </div>
  </div>
</div>
```

我们得到的首页效果大致如图4所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d55551f930.jpg" alt="图4 微信客户端首页效果图" title="图4 微信客户端首页效果图" />

图4 微信客户端首页效果图

### 设计微信客户端功能页样式

以娃像谁-单人照的页面为例，页面代码如下。

```
<div class="pic_panel">
    <div class="parent" id="parent1">
        <i class="icon iconfont icon-210 human"></i>
    </div>
    <div class="parent" id="parent2">
        <i class="icon iconfont icon-nv human"></i>
    </div>
    <div class="clear"></div>
    <div class="parent1like like"></div>
    <div class="parent2like like"></div>
    <div class="clear"></div>
    <div class="picture" id="child">
        <i class="icon iconfont icon-child human"></i>
    </div>
    <form>
        <input type="button" class="next" id="uploadImage" value="GO !!!">
    </form>
</div>
```

id="parent1" 和 id="parent2" 为存放父母照片的容器，id="child"为存放孩子照片的容器，点击容器触发选择照片，选择完成点击按钮作比较。class="parent1like"和class="parent2like" 分别显示 id="child"分别与 id="parent1"和id="parent2" 对比的结果。

我们得到的页面效果类似图5所示的样子。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d5565898c8.jpg" alt="图5 娃像谁—单人照的页面效果图" title="图5 娃像谁—单人照的页面效果图" />

图5 娃像谁—单人照的页面效果图

### 实现微信客户端交互

在之前我们写了一个 WebAPI 接口来实现微信 JS-SDK 接口注入权限验证配置，现在我们的客户端需要调用这个接口来做验证了。客户端你需要引用 jweixin-1.0.0.js。

只要我们的业务需要使用微信 JS-SDK，则都需要完成接口注入的权限验证，验证的方式我们来一步步分析见图6。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d557451531.jpg" alt="图6 JS-SDK接口注入的权限验证过程" title="图6 JS-SDK接口注入的权限验证过程" />

图6 JS-SDK 接口注入的权限验证过程

页面将 noncestr（这个可以是页面定义一个常数）、timestamp（其实也可以是常数）、url 当前页面地址提交给我们最早写的/api/weixin 接口，然后将返回的签名提交给 wx.config 即可。下面的代码可以作为你的模板使用。

```
$(function () {
   var timestamp = Date.parse(new Date())/1000;
    var localurl = encodeURIComponent(window.location.href.split('#')[0]);
    $.ajax({
      url: 'http://www.********.cn/wxapi/api/weixin',
      dataType: "json",
      data: {
        noncestr: 'FFUmZdbWVT9mVP7a',
        timestamp: timestamp,
        url: window.location.href.split('#')[0]
},
      success: function (data) {
wxFace(data.toLowerCase());
      }
 })
function wxFace(signature) {
   wx.config({
       debug: false,
       appId: 'wxec54ec7f720993da',
       timestamp: timestamp,
       nonceStr: 'FFUmZdbWVT9mVP7a',
       signature: signature,
       jsApiList: [
          'checkJsApi',
          'onMenuShareTimeline',
          'onMenuShareAppMessage',
          'chooseImage',
          'previewImage',
          'uploadImage',
          'downloadImage'
         ]            });
      }
 });
```

然后我们定义选择图片函数，当选择 id="parent1"、 id="parent2"、id="child"时调用。

```
function chooseUpload(selector) {
    wx.chooseImage({
        success: function (res) {
            $("#loading").show();
         $(function () {
            $.each(res.localIds, function (i, n) {
               wx.uploadImage({
               localId: res.localIds.toString(), // 需要上传的图片的本地ID，由chooseImage接口获得                        
                 isShowProgressTips: 0, // 默认为1，显示进度提示
                 success: function (res1) {
                   $.ajax({
                      url: 'http://www.******.cn/wxapi/face/detect/' + res1.serverId,                                   
                     dataType: "json",
                     success: function (data) {
                       $("#loading").hide(); 
             if (JSON.parse(data).length == 1) { 
               $(selector).html('<img src="' + n + '"> <br>')                                          .data('faceId', JSON.parse(data)[0].faceId);
             } else if (JSON.parse(data).length > 1) { 
               alert('请选择单人照哦')
            } else { 
               alert('啊，我看不到你的脸~')
            }
          }
       })
      },
      fail: function (res) {
        alert(JSON.stringify(res));
       }
     });
    });
   });
  }
 });
}
```

触发点击事件调用上传图片函数。

```
document.querySelector('#parent1').onclick = function () {
    chooseUpload('#parent1')
};
定义函数，将拿到的两张照片的id做对比。
  function verify(selector, parent, child) {
      $("#loading").show();
      $.ajax({
          url: 'http://www.******.cn/wxapi/face/verify/' + parent + '/' + child,    
          dataType: "json",
          success: function (data) {
              $("#loading").hide();
              $(selector).html('相似度：' + (JSON.parse(data).confidence * 100).toFixed(2) + '%')
          }
      })      
  }
```

最后是我们分享朋友圈的功能实现。

```
var shareData = {
      title: '测测孩子跟谁像',//分享的标题
      desc: '来看看孩子跟爸爸比较像还是跟妈妈比较像',//分享的描述
      link: window.location.href,//分享的快照
      imgUrl: 'http://www. .******.cn/WeFace/fonts/family.jpg'//分享的链接
  };
  wx.onMenuShareAppMessage(shareData);
  wx.onMenuShareTimeline(shareData);
```

分享结果如图7所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d55886d0af.jpg" alt="图7 朋友圈分享功能效果图" title="图7 朋友圈分享功能效果图" />

图7 朋友圈分享功能效果图

### 总结

本文主要针对如何使用 APS.NET WebAPI 实现微信注入进行了深入讲解。下期会承接本文，重点分享服务的实现过程，内容主要有：调用封装微软牛津计划 API、使用 MongoDB 存储数据和客户端如何使用。全文阅读完毕后，你将可以自己去编写更有价值的应用了。
