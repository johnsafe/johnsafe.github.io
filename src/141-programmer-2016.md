## 当微软牛津计划遇到微信 App——服务实现部分

文/王豫翔

>微软牛津计划（Project Oxford）提供了一系列机器学习 API，包含计算机视觉、语音识别和语言理解等认知服务。本文承接上期，继续为大家讲解它能为微信开发带来的有趣功能。

### 封装微软牛津计划 API 客户端

牛津计划的 API 是由一个基础 URL、服务名称、参数组成的服务，我们的 ProjecToxfordClientHelper 就是计划将牛津 API 的实现进行封装，为我们不同的 APIController 提供服务。KEY 可以通过注册牛津开发计划来获得。

```
public ProjecToxfordClientHelper()
{
    client = new HttpClient();
    var queryString = HttpUtility.ParseQueryString(string.Empty);
    client.DefaultRequestHeaders.Add("ContentType", "application/json");
    client.DefaultRequestHeaders.Add("Ocp-Apim-Subscription-Key", KEY);
}
```

接下来，我们要实现两种 POST 的提交，一种是提交流参数，一种是提交字符串参数。

实现提交字符串参数的 POST：

```
public async Task<projectoxfordresponsemodels> PostAsync(string querkey, object body, Dictionary<string, string=""> querystr = null)
{
    var queryString = HttpUtility.ParseQueryString(string.Empty);
    if (querystr != null)
    {
        foreach (var entry in querystr)
        {
            queryString[entry.Key] = entry.Value;
        }
    }
    var uri = string.Format("{0}/{1}?{2}", serviceHost, querkey, queryString);
    var jsonStr = Newtonsoft.Json.JsonConvert.SerializeObject(body);
    byte[] byteData = Encoding.UTF8.GetBytes(jsonStr);
 
    HttpResponseMessage response;
    using (var content = new ByteArrayContent(byteData))
    {
        content.Headers.ContentType = new MediaTypeHeaderValue("application/json");
        response = await client.PostAsync(uri, content);
        var msg = await response.Content.ReadAsStringAsync();
        return new ProjecToxfordResponseModels(msg, response.StatusCode);
    }
}</string,></projectoxfordresponsemodels>
```

所谓的字符串参数就是将实现 Fields 的对象以 JSON 格式序列化，然后 POST 给牛津 API。

```
var jsonStr = Newtonsoft.Json.JsonConvert.SerializeObject(body);
byte[] byteData = Encoding.UTF8.GetBytes(jsonStr);
```

所以要记得 content 的内容类型要定义为：

```
content.Headers.ContentType = new MediaTypeHeaderValue("application/json");
```

那类似图片这些流文件不能采用这个方法，所以我们重载了一个方法。

```
public async Task<projectoxfordresponsemodels> PostAsync(string querkey, byte[] body, Dictionary<string, string=""> querystr = null)
{
    var queryString = HttpUtility.ParseQueryString(string.Empty);
    if (querystr != null)
    {
        foreach (var entry in querystr)
        {
            queryString[entry.Key] = entry.Value;
        }
    }
 
    var uri = string.Format("{0}/{1}?{2}", serviceHost, querkey, queryString);
 
    HttpResponseMessage response;
    using (var content = new ByteArrayContent(body))
    {
        content.Headers.ContentType = new MediaTypeHeaderValue("application/octet-stream");
        response = await client.PostAsync(uri, content);
        var msg = await response.Content.ReadAsStringAsync();
        return new ProjecToxfordResponseModels(msg, response.StatusCode);
    }
}</string,></projectoxfordresponsemodels>
```

看下参数，流格式的内容需要以 Byte 数组的方式进行传递，但实际的处理中没有什么太大的不同，如果传递的是 Byte 数组就直接处理，否则先序列化为 Byte 数组，但是要注意的是，流媒体的 JSON 编码是不同的，如图1所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b6369b8cb8.png" alt="图1  流格式内容的处理过程" title="图1  流格式内容的处理过程" />

图1  流格式内容的处理过程

所以我们优化下代码为：

```
public async Task<projectoxfordresponsemodels> PostAsync(string querkey, object body, Dictionary<string, string=""> querystr = null)
{
    var queryString = HttpUtility.ParseQueryString(string.Empty);
    if (querystr != null)
    {
        foreach (var entry in querystr)
        {
            queryString[entry.Key] = entry.Value;
        }
    }
    var uri = string.Format("{0}/{1}?{2}", serviceHost, querkey, queryString);
 
    byte[] byteData = null;
 
    if (body.GetType() == typeof(byte[]))
    {
        byteData = (byte[])body;
    }
    else
    {
        var jsonStr = Newtonsoft.Json.JsonConvert.SerializeObject(body);
        byteData = Encoding.UTF8.GetBytes(jsonStr);
    }
 
    HttpResponseMessage response;
    using (var content = new ByteArrayContent(byteData))
    {
        content.Headers.ContentType = body.GetType() == typeof(byte[]) ? 
            new MediaTypeHeaderValue("application/octet-stream") : 
            new MediaTypeHeaderValue("application/json");
        response = await client.PostAsync(uri, content);
        var msg = await response.Content.ReadAsStringAsync();
        return new ProjecToxfordResponseModels(msg, response.StatusCode);
    }
}</string,></projectoxfordresponsemodels>
```

### 实现 Face/Detect

Detect 服务接受一个上传的图片，并且识别其中的人脸，如果找不到人脸则返回一个空的数组，否则返回人脸数据的数组，这些人脸数据包含了：FaceID、性别、年龄、微笑值、胡须情况等。

当我们上传了一张有效照片之后，牛津计划会返回给我们对照片中每一个识别成功的人脸的 FaceID，这个 ID 很重要，当我们需要再次了解照片中人脸的信息，我们不必再次上传照片，直接提交这个 FaceID 即可。

还记得我们说过微信客户端上传的图片是不能直接 POST 到我们 WebAPI 服务端的，我们必须从微信服务器去下载照片，然后上传给牛津 API，所以我们的代码需要如下实现。

```
[HttpGet]
[Route("face/detect/{weixnmediaid}")]
public async Task<httpresponsemessage> Detect(string weixnmediaid)
{
    var key = "detect";
    var file = await new WeixinController().Get(weixnmediaid);
    var content = FileHelper.ReadAsync (file);
 
    if (content != null)
    {
        var result = await client.PostAsync(key,
            content,
            new Dictionary<string, string=""> {
            {"returnFaceId","true"},
            {"returnFaceLandmarks","flase"},
            }
            );
 
        return client.CreateHttpResponseMessage(Request, result);
    }
    throw new HttpResponseException(HttpStatusCode.BadRequest);
}</string,></httpresponsemessage>
```

在这里我用了 RouteAttribute，微软的 ASP.NET WEBAPI 的 RouteAttribute 十分好用，我们可以将路由设计为非常友好的状态，而不用设计为带参数的 URL，现在我们的 API 看上去高大上多了。

现在我们封装了 Face/Detect 服务了，可以提供微信客户端和 PC 浏览器客户端的服务了。

### 实现 Face/Verify

Verify 是非常好玩的服务，他可以对比两张人脸是否一致，或者相似度多少。牛津的 VerifyAPI 比较简单，POST 两个 FaceID 即可得到一个结果，所以我们的封装也很简单。

```
[HttpGet]
[Route("face/verify/{faceId1}/{faceId2}")]
public async Task<httpresponsemessage> Verify(string faceId1, string faceId2)
{
    var key = "verify";
 
    var result = await client.PostAsync(
           key,
            new
            {
                faceId1 = faceId1,
                faceId2 = faceId2
            }
           );
 
    return client.CreateHttpResponseMessage(Request,result);
}</httpresponsemessage>
```

你可以看到，同样因为采用了 RouteAttribute 了，我们的 URL 非常优雅，值得注意的是我们的 API 提供的是 Get，至少我们调试方便了很多，不是吗？

### 使用 MongoDB 服务

其实在项目前期，我完全没有想到需要使用数据库，但是随着完成了 Face/Detect 和 Face/Verify 的封装后，我发现显然数据库是必须的，原因是：牛津的 FaceAPI 是收费的，当客户端每次调用的使用，都会消耗我们的宝贵资源，所以我们希望在如下两种情况下用户的请求不必再次访问牛津 FaceAPI。

1. 用户刷新页面时，不需要重新访问牛津 FaceAPI；

2. 当用户分享自己的测试结果，其他用户访问这个页面时，不必对同样的照片计算再次提交给牛津 API。

那么为什么我们不采用 SQL Server 呢？因为我们保存的是每次牛津的计算结果，这些结果之间没有任何关系型需求，我们不需要事务处理，我们需要查询效率极高，而且很重要的是，我们保存的牛津计算结果是 JSON 格式的数据，结合以上需求，显然采用 MongoDB 是明智的选择。

因为 MongoDB 的操作是强类型，所以我们必须为涉及到的数据源建立 Models。存储微信服务器得到的 MediaID 和本地文件名关系的 WeixinImgFileModels。

```
public class WeixinImgFileModels
{
    public ObjectId _id { set; get; }
    public string MediaId { set; get; }
    public string FileName { set; get; }
}
```

存储本地文件名和 Face 识别数据关系的 DetectResultModels。

```
public class DetectResultModels
{
    public ObjectId _id { set; get; }
    public string faceId { set; get; }
    public string FileName { set; get; }
    public double Age { set; get; }
    public string Gender { set; get; }
    public double Smile { set; get; }
}
```

存储一对 FaceID 的比较结果。

```
public class VerifyModels
{
    public ObjectId _id { set; get; }
    public string FaceID1 { set; get; }
    public string FaceID2 { set; get; }
    public double Confidence { set; get; }
    public bool IsIdentical { set; get; }
}
```

### 改进 Face/Detect 控制器

现在 Face/Detect 和 Face/Verify 将支持将用户提交的结果持久化。我们先考虑下 Face/Detect 现在的变化，原先我们的流程是：从微信客户端获得 MediaID，通过这个 MediaID 从微信服务器下载图片，然后将这个图片提交给牛津，以获得 FaceID，见图2。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b6a6ab8651.png" alt="图2  改进前获取FaceID的历程" title="图2  改进前获取FaceID的历程" />

图2  改进前获取 FaceID 的历程

现在我们需要考虑的更周到了：当从微信客户端得到 MediaID，我们需要查看下本地文件夹中是否有匹配的文件，在提交给牛津之前我们也需要从 MangoDB 数据库中查询是否有匹配的上次的提交结果。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b6b9ae015e.png" alt="图3 改进后获取FaceID的历程" title="图3 改进后获取FaceID的历程" />

图3 改进后获取 FaceID 的历程

我们先改进微信服务的代码，使得只有 MongoDB 没有存储 MediaID 和对应的文件时，再从微信的服务器去下载图片。

```
public async Task<string> Get(string mediaid)
{
    var mongo = new MongoDBHelper("weixinImgFile");
 
    //查询MongoDB中是否存储了MediaID对应的照片文件
    var doc = await mongo.SelectOneAsync(x => x["mediaid"] == mediaid);
    if (doc != null)
    {
        return doc["filename"].ToString();
    }
 
    //http://file.api.weixin.qq.com/cgi-bin/media/get?access_token=ACCESS_TOKEN&media_id=MEDIA_ID
    var queryString = HttpUtility.ParseQueryString(string.Empty);
    queryString["access_token"] = await Get();
    queryString["media_id"] = mediaid;
 
    var uri = "http://file.api.weixin.qq.com/cgi-bin/media/get?" + queryString;
 
    HttpResponseMessage response;
    response = await client.GetAsync(uri);
 
    var msg = await response.Content.ReadAsStreamAsync();
    var fileName = response.Content.Headers.ContentDisposition.FileName.Replace("\"", "");
    var helper = new ProjecToxfordClientHelper();
    var content = await FileHelper.ReadAsync(msg);
 
    FileHelper.SaveFile(content, fileName);
 
    await mongo.InsertAsync(Newtonsoft.Json.JsonConvert.SerializeObject(
        new {
            Mediaid = mediaid,
            FileName = fileName
        }
        ));
 
    return fileName;
}</string>
```

然后我们来改进 FaceController 的 DetectAPI，使得先在 MongoDB 中查询对应照片的分析结果，当没有之前查询的结果，再去牛津进行分析。

```
[HttpGet]
[Route("face/detect/{weixnmediaid}")]
public async Task<httpresponsemessage> Detect(string weixnmediaid)
{
    var key = "detect";
 
    //得到从微信服务器下载的文件名
    var fileName = await new WeixinController().Get(weixnmediaid);
 
    var mongo = new MongoDBHelper<detectresultmodels>("facedetect");
 
    //照片之前有没有下载过
    var docArr = await mongo.SelectMoreAsync(x => x.FileName == fileName);
    if (docArr.Count > 0)
    {
        var resultJson = docArr.Select(
            doc => new
            {
                faceId = doc.faceId,
                filename = doc.FileName,
                age = doc.Age,
                gender = doc.Gender,
                smile = doc.Smile
            }
            ).ToJson();
 
        return client.CreateHttpResponseMessage(
            Request,
            new Models.ProjecToxfordResponseModels(resultJson, HttpStatusCode.OK));
    }
    //如果MongoDB中没有该照片对应的Face信息
    var content = await FileHelper.ReadAsync(fileName);
 
    if (content != null)
    {
        var result = await client.PostAsync(key,
            content,
            new Dictionary<string, string=""> {
            {"returnFaceId","true"},
            {"returnFaceLandmarks","flase"},
            {"returnFaceAttributes","age,gender,smile"}
            }
            );
 
        if (result.StatusCode == HttpStatusCode.OK)
        {
            var tmpJArr = Newtonsoft.Json.Linq.JArray.Parse(result.Message);
            //将牛津结果写入数据库
            foreach (var tmp in tmpJArr)
            {
                await mongo.InsertAsync(new DetectResultModels()
                {
                    FileName = fileName,
                    faceId = (string)tmp["faceId"],
                    Age = (double)tmp["faceAttributes"]["age"],
                    Gender = (string)tmp["faceAttributes"]["gender"],
                    Smile = tmp["faceAttributes"]["smile"] != null ? (double)tmp["faceAttributes"]["smile"] : 0
                });
            }
            var resultJson = tmpJArr.Select(x => new
              {
                  faceId = x["faceId"],
                  age = (double)x["faceAttributes"]["age"],
                  gender = (string)x["faceAttributes"]["gender"],
                  smile = x["faceAttributes"]["smile"] != null ? (double)x["faceAttributes"]["smile"] : 0
              }).ToJson();
 
            return client.CreateHttpResponseMessage(
                Request,
                new Models.ProjecToxfordResponseModels(resultJson, HttpStatusCode.OK));
        }
    }
    throw new HttpResponseException(HttpStatusCode.BadRequest);
}</string,></detectresultmodels></httpresponsemessage>
```

### 改进 Face/Verify

Face/Verify 的逻辑要简单的多，因为不需要涉及到第三方微信的服务，我们原先的逻辑是每次将得到的 Face1ID 和 Face2ID 提交给牛津以得到结果，见图4。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b6b682e729.png" alt="图4  改进前Face/Verify的处理逻辑" title="图4  改进前Face/Verify的处理逻辑" />

图4  改进前 Face/Verify 的处理逻辑

现在我们将先查询 MongoDB 数据库，如果存储了之前的结果我们就直接返回，否则提交给牛津服务器。但是我们在查询结果的时候需要注意，每次客户端给我们的 Face1ID 和 Face2ID 不一定是相同的次序，所以要解决这个问题，我们有两种办法。

1. 查询的时候两种次序排列都查询一次；

2. 存储的时候两种次序排列都存储一次。

考虑到查询性能，我选择存储冗余，在存储的时候把两种排列次序都存储一次。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b6c3b7637a.png" alt="图5  改进后Face/Verify的处理逻辑" title="图5  改进后Face/Verify的处理逻辑" />

图5  改进后 Face/Verify 的处理逻辑

按上面的思路，我们对代码做了修改。

```
[HttpGet]
[Route("face/verify/{faceId1}/{faceId2}")]
public async Task<httpresponsemessage> Verify(string faceId1, string faceId2)
{
    var key = "verify";
 
    var mongo = new MongoDBHelper<verifymodels>("faceverify");
 
    //先检查数据库中是否有上次比较的结果
    var doc = await mongo.SelectOneAsync(x =>
        (x.FaceID1 == faceId1 && x.FaceID2 == faceId2)
        );
    if (doc != null)
    {
        var mongoResult = new
        {
            faceID1 = doc.FaceID1,
            faceID2 = doc.FaceID2,
            confidence = doc.Confidence,
            isIdentical = doc.IsIdentical
        }.ToJson();
 
        return client.CreateHttpResponseMessage(
            Request,
            new Models.ProjecToxfordResponseModels(mongoResult, HttpStatusCode.OK));
    }
 
    //如果之前的结果没有查询到，则提交牛津查询
    var result = await client.PostAsync(
           key,
            new
            {
                faceId1 = faceId1,
                faceId2 = faceId2
            }
           );
 
    if (result.StatusCode == HttpStatusCode.OK)
    {
 
        var tmp = Newtonsoft.Json.Linq.JObject.Parse(result.Message);
        //如果为了加速查询的话，我们采用两次写入
        await mongo.InsertAsync(new VerifyModels()
        {
            FaceID1 = faceId1,
            FaceID2 = faceId2,
            Confidence = (double)tmp["confidence"],
            IsIdentical = (bool)tmp["isIdentical"]
        });
 
        await mongo.InsertAsync(new VerifyModels()
        {
            FaceID1 = faceId2,
            FaceID2 = faceId1,
            Confidence = (double)tmp["confidence"],
            IsIdentical = (bool)tmp["isIdentical"]
        });
 
        var resultJson = new
        {
            faceID1 = faceId1,
            faceID2 = faceId2,
            confidence = (double)tmp["confidence"],
            isIdentical = (bool)tmp["isIdentical"]
        }.ToJson();
 
        return client.CreateHttpResponseMessage(
            Request,
            new Models.ProjecToxfordResponseModels(resultJson, HttpStatusCode.OK));
    }
    return client.CreateHttpResponseMessage(Request, result);
}</verifymodels></httpresponsemessage>
```

### 总结

到目前为止，如何使用 APS.NET WebAPI 实现微信注入、调用封装微软牛津计划 API、使用 MongoDB 存储数据和客户端如何使用，都已经说清楚了，基本上，你已经可以自己去编写更有价值的应用了。
