## 游戏开发中的程序生成技术

文/申江涛

程序生成游戏内容简称 PCG（Procedural Content Generation），它使用了随机或者伪随机数技术，给游戏带来了无限的可能。相比于传统的由设计师将游戏世界中的一草一木都精心配制，PCG 的方法是去配置一些生成的规则，然后由生成算法自动去生成游戏世界。

过去，由于游戏主机和 PC 性能的限制，PCG 的内容非常简单，比如随机地牢或者游戏地图，但近年来随着 sandbox 品类的游戏兴起，比如风靡全球的《Minecraft》，PCG 能够发挥的作用越来越大。

下面就以程序生成地牢和程序生成地形为例，分享一些 PCG 的一些基本方法。

### 程序生成地牢

《Rogue》是最早使用程序生成技术的游戏之一，它最大的特性就是动态程序生成，可以让玩家每次玩游戏都有着完全不一样的体验。由这个游戏诞生出了一种以程序生成技术为代表的游戏种类，称之为“Rougelike”。

如果让程序生成一个地牢呢？那我们首先定义出地牢中的一些基本组件，主要包括：

- 房间：有一个或者多个出口；

- 走廊：是一个很窄很长的区域，拥有两个出口，它也可能是一个斜坡；

- 连接处：有着三个以上出口的小的空间。

如图1所示是几个简单的组件模型，在每个组件的出口，都有一个 mark，标记了位置和旋转量，用于组件的匹配。

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a0056a994a0.png" alt="图1  组件模型" title="图1  组件模型" />

图1  组件模型

现在将它们遵循一定的规则拼接在一起，就可以生成地牢了，规则如下：

- 房间只能连接走廊；

- 走廊连接房间或连接处；

- 连接处只能连接走廊。

接下来是生成算法的详细描述：

1. 初始化一个启始模块（可以选择有最多出口数的模块）；

2. 对每个未连接的出口生成一个符合条件的模块；

3. 重新建立一个目前未连接的出口的列表；

4. 重复步骤2和步骤3。

如图2、3、4所示展示了算法迭代的过程。

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a005bf81453.png" alt="图2  初始状态" title="图2  初始状态" />

图2  初始状态

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a005dda41d8.png" alt="图3  迭代三次 " title="图3  迭代三次 " />

图3  迭代三次

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a005fdc1fa2.png" alt="图4  迭代六次" title="图4  迭代六次" />

图4  迭代六次

上述的随机算法只是生成了地牢的框架，还需要生成很多其他的地牢要素，比如在地上可以打碎的罐子，墙上忽明忽暗的火炬，还有突然从身后窜出来的史莱姆。这些要素的生成方法和前面的大同小异，简单说就是在组件的地面或者墙上标记上一些 mark 点，在其上随机生成一些匹配的要素即可。

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a009304c41e.png" alt="图5  程序生成的房间" title="图5  程序生成的房间" />

图5  程序生成的房间

### 程序生成地形

许多开放世界游戏内容通常都包含了一个生成系统，该系统通过一个种子生成游戏世界。随机生成系统通常都会随机生成一些地形和生态，然后基于这些去分布资源生物等，这方面的代表作当属今年8月即将发售的游戏《No Man’s Sky》，该游戏中通常包含了数以亿计程序生成的星球，每个星球上都有着不同的植物系统、生物系统、甚至还有长相不同的外星人。

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a00962bad39.png" alt="图6  No Man’s Sky界面" title="图6  No Man’s Sky界面" />

图6  No Man’s Sky 界面

下面简述程序生成地形的技术，主要包括两个部分：一个是高度图的生成和 Mesh 的生成。

#### 通过噪声生成高度图

对于一个一维 Coherent noise，每一个 x 值都有一个 y 值与其对应，如图7所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a00992c1c66.png" alt="图7  一维Coherent noise图" title="图7  一维Coherent noise图" />

图7  一维 Coherent noise 图

如果用该曲线来表示地形的话，由于曲线过于平滑，没有细节，就会显得很不真实，这里的做法是通过将多个波形叠加，来得出一个比较真实的曲线，每一个叠加的波形都称之为 octave。

f(x) = noise(p) + Persistence ^ 1 * noise(Lacunarity ^1 * p) + Persistence ^ 2 * noise(Lacunarity ^2 * p) + ...

最终的曲线主要由两个参数控制，一个是 Lacunarity 表示频率的增加量，另一个是 Persistence 表示增幅的减小量，每个 octave 的波形都有着不同的意义，如图8所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a009b8ee3d1.png" alt="图8  octave的波形图" title="图8  octave的波形图" />

图8  octave 的波形图

通过将多个 Octave 进行叠加，得到了一个比较有高低起伏，又有细节的波形。如果是二维的噪声，得到的就是一张二维的噪声图，核心代码如下：

```
for(int i = 0; i< octaves; i++)
 {
float sampleX = x / scale * frequency;
    float sampleY = y / scale * frequency;
 
    float perlinValue = Mathf.PerlinNoise(sampleX, sampleY) * 2 - 1;
    noiseHeight += perlinValue * amplitude;
 
     amplitude *= persistance;
     frequency *= lacunarity;
}
```

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a00a13a3956.png" alt="图9  叠加3个octave的结果" title="图9  叠加3个octave的结果" />

图9  叠加3个 octave 的结果

#### 3D 地形生成

3D 地形的生成主要是生成 Mesh，在 Unity3D 中生成一个三角形的代码方法如下：

```
Viod GenerateTriangle
{
     mesh = new Mesh();
        uvs = new Vector2[3];
 
        colors = new Color[3];
        colors[0] = Color.red;
        colors[1] = Color.green;
        colors[2] = Color.blue;
 
        Vector3[] vertices = new Vector3[3];
        vertices[0] = new Vector3(-1, 0, 0);
        vertices[1] = new Vector3(0, 1.732f, 0);
        vertices[2] = new Vector3(1, 0, 0);
        mesh.vertices = vertices;
 
        Vector3[] normals = new Vector3[3];
        normals[0] = Vector3.back;
        normals[1] = Vector3.back;
        normals[2] = Vector3.back;
 
        uvs[0] = new Vector2(0, 0);
        uvs[1] = new Vector2(0, 1);
        uvs[2] = new Vector2(1, 1);
 
        int[] triangles = new int[3] { 0, 1, 2 };
        mesh.triangles = triangles;
        mesh.normals = normals;
        mesh.uv = uvs;
        mesh.colors = colors;
        GetComponent<meshfilter>().mesh = mesh;
}</meshfilter>
```

从上面例子可以得出创建 Mesh 的过程就是生成一系列三角形的过程，而每个三角形都包含了每个顶点的坐标、顶点顺序、法线、uv 坐标、顶点颜色。

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a00a966ba7a.png" alt="图10  3D地形生成结果" title="图10  3D地形生成结果" />

图10  3D 地形生成结果

现在每个像素点就是一个三角形的顶点，那么需要生成顶点的个数为，v = w * h，三角形数量 t = (w-1)(h-1) * 2个，每个顶点间的距离为1，则可以生成一个 w*h 的平面。

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a00ad6bde77.png" alt="图11  w*h平面" title="图11  w*h平面" />

图11  w*h 平面

接下来，可以将噪声的值对应到网格顶点的 y 值，同时乘以一个放大系数。

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a00af8b283c.png" alt="图12  噪声的值对应网格顶点的y值" title="图12  噪声的值对应网格顶点的y值" />

图12  噪声的值对应网格顶点的 y 值

再尝试为顶点着色，可以直接根据噪声值来对应顶点的颜色，这里设定值低于0.3的为深海，0.3-0.4之间的为浅水，0.4-0.45为沙地，0.45-0.55为草坪，0.55-0.6为深色的草坪，0.6-0.7为浅色岩石，0.7-0.9为深色岩石，0.9-1.0为山顶，每种区域都对应不同的颜色下面是给顶点加上颜色之后同时计算了三角形的 normal 之后的结果。

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a00b304c180.png" alt="图13  地形区域" title="图13  地形区域" />

图13  地形区域

上面所描述的只是最基本的地形生成过程，通常随机地形的生成还包括很多需要解决的问题，比如无缝大地形、地形的 LOD、多线程生成优化等，接下来通过 LOD 减少 Mesh 中三角形数量。

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a00b8171c7d.png" alt="图14  LOD = 0" title="图14  LOD = 0" />

图14  LOD = 0

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a00ba3e8231.png" alt="图15  LOD=4" title="图15  LOD=4" />

图15  LOD=4

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a00bc5bc459.png" alt="图16  LOD=8" title="图16  LOD=8" />

图16  LOD=8

在更加复杂的生成系统中，还需要生成洞穴，植被，生物群落等内容，这些高级内容不论是对生成算法还是对系统的架构都有着很大的技术挑战。

### 小结

本文简单介绍了两种游戏中的程序生成技术，然而 PCG 可以做的更多，比如程序生成的 AI、程序生成的音效等等。如果将 PCG 与其他游戏要素进行融合，将会得到更多可能，但同时也带来了很大的技术和美术上的挑战。

#### 参考文献

[1] A Real-Time Procedural Universe, Part One: Generating Planetary Bodies
[2] Procedural generation Wiki