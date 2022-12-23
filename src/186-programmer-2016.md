## 手机游戏无障碍设计——猜地鼠之 Android 篇

文/何金源

>手机应用无障碍化逐渐受到重视，这项技术为盲人或者视力有障碍的人士带来了很大便利。那么对于手机游戏，同样也应该进行无障碍化，本文将以盲人猜地鼠游戏为例，讲解如何对手机游戏进行无障碍化设计，如何让原本无法操作变成可在无障碍模式下正常使用，最后总结手机游戏无障碍化的大体思路。

### 前言

目前市场上针对盲人进行无障碍化的手机游戏几乎没有，这对于障碍用户来讲，是一大遗憾。实际上，他们对游戏的渴望跟对应用无障碍化的渴望一样强烈，他们也希望能在手机上体验一把游戏的乐趣。下面，我们来探讨手机游戏无障碍化。

由于是首次针对手机游戏进行无障碍化，所以这里挑选了一款较为简单的游戏——猜地鼠。猜地鼠是基于 MasterMind 游戏的玩法（如图1）。原本玩法是两个人对玩，其中一人是出谜者，摆好不同颜色的球的位置；另一个人是猜谜者，每个回合猜各个位置应该放什么颜色的球。每回合结束后会有结果提示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58184e590e51e.jpg" alt="图1 MasterMind游戏" title="图1 MasterMind游戏" />

图1 MasterMind 游戏

而猜地鼠则是会有不同颜色的地鼠，用户每个回合要猜地鼠的颜色排列顺序。这是使用 Cocos2d-x 引擎开发的，引擎并没对无障碍模式做出优化，所以游戏开发完成后只有一个大焦点在界面上，当用户点击这个焦点时，没法进行下一步操作，怎么点也进不了游戏界面。如图2所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58184e6580fc8.png" alt="图2 优化前的游戏界面" title="图2 优化前的游戏界面" />

图2 优化前的游戏界面

### 构造无障碍虚拟节点

Cocos2d-x 引擎是支持跨平台的，所以我们可以针对不同平台区分处理无障碍。首先看下 Android 平台，在游戏中，Cocos2d-x 是把游戏里的界面、图片和文字等素材画到 GLSurfaceView 上，而 GLSurfaceView 是添加到 Cocos2dxActivity 的布局中。Cocos2dxActivity 是 Cocos2d-x 引擎在 Android 平台上最主要的 Activity，我们要对游戏进行无障碍化，就需要让这个 Activity 支持无障碍。

在 Android 平台，可以对自定义 View 进行无障碍化支持。具体的原理可以参考官网上的介绍，或者《程序员》8月发布的《Android 无障碍宝典》。所以我们第一步，就是为 Cocos2dxActivity 增加一个自定义 View。

```
public class MasterMind extends Cocos2dxActivity{
     
private AccessibilityGameView mGameView;
 
protected void onCreate(Bundle savedInstanceState){
super.onCreate(savedInstanceState);
         
Rect rectangle = new Rect();
getWindow().getDecorView().getWindowVisibleDisplayFrame(rectangle);
        AccessibilityHelper.setScreen(rectangle.width(), rectangle.height());
@SuppressWarnings("deprecation")
LinearLayout.LayoutParams params = new LinearLayout.LayoutParams(
                getWindowManager().getDefaultDisplay().getWidth(),
                getWindowManager().getDefaultDisplay().getHeight());
mGameView = new AccessibilityGameView(this);
addContentView(mGameView, params);
         
AccessibilityHelper.setGameView(mGameView);
         
ViewCompat.setImportantForAccessibility(mGLSurfaceView, ViewCompat.IMPORTANT_FOR_ACCESSIBILITY_NO);
    }
     
    static {
         System.loadLibrary("game");
    }
}
```

AccessibilityGameView 是个简单透明的自定义 View，直接通过 addContentView 方法加入到布局中，同时把 GLSurfaceView 设置为不需无障碍焦点。

```
public class AccessibilityGameView extends View {
     
private BaseSceneHelper mTouchHelper;
 
public AccessibilityGameView(Context context) {
super(context);
}
     
public void setCurSceneHelper(BaseSceneHelper helper){
mTouchHelper = helper;
    }
     
    @SuppressLint("NewApi")
    @Override
protected boolean
dispatchHoverEvent(MotionEvent event) {
if(mTouchHelper != null && mTouchHelper.dispatchHoverEvent(event)){
return true;
}
         
return super.dispatchHoverEvent(event);
  }
 
}
```

AccessibilityGameView 中包含 BaseSceneHelper 类，并将 HoverEvent 交给 BaseSceneHelper 处理。BaseSceneHelper 负责构造自定义无障碍虚拟节点，提供游戏中必要信息给用户。首先看 BaseSceneHelper 内部的实现。

```
public class BaseSceneHelper extends ExploreByTouchHelper {
protected ArrayList<accessibilityitem> mNodeItems;
public BaseSceneHelper(View forView) {
super(forView);
mNodeItems = new 
ArrayList<accessibilityitem>();
    }
    @Override
protected int getVirtualViewAt(float x, float y) {
for (int i = 0; i < mNodeItems.size(); i++) {
Rect rect = mNodeItems.get(i).mRect;
if (rect.contains((int) x, (int) y)) {
return i;
            }
        }
        return -1;
    }
    @Override
protected void getVisibleVirtualViews(List<integer> virtualViewIds) {
for (int i = 0; i < mNodeItems.size(); i++) {
virtualViewIds.add(mNodeItems.get(i).id);
        }
}
    @Override
    protected void onPopulateNodeForVirtualView(int virtualViewId,
            AccessibilityNodeInfoCompat node) {
if(mNodeItems.size() > virtualViewId){
node.setContentDescription(getContentDesc(virtualViewId));
            setParentRectFor(virtualViewId, node);
        }
    }
private String getContentDesc(int virtualViewId) {
        if (mNodeItems.size() > virtualViewId) {
return mNodeItems.get(virtualViewId).mDesc;
        }
return "";
    }
    …..
}
```

BaseSceneHelper 实现无障碍辅助类 ExploreByTouchHelper，通过维护 AccessibilityItem 列表，将所需要的无障碍虚拟节点的 Rect 和 Description 记录起来。当 BaseSceneHelper 被调用创建无障碍节点时，实时提供 AccessibilityItem。

```
public class AccessibilityItem {
 
    public Rect mRect;
    public String mDesc;
    public int id;
}
```

### 处理游戏场景切换

到此，就完成了 Java 层的无障碍化，但是游戏中需要的无障碍焦点，得从游戏代码中触发。对猜地鼠进行无障碍化，一共要处理4个场景。一进入游戏就能看到的菜单场景（如图3），会有两个按钮需要无障碍化；从开始游戏按钮，我们会进入游戏场景（如图4），这个场景相对比较复杂，需要对每个不同颜色的地鼠进行无障碍化，然后需要对当前回合的结果提示进行无障碍化，还要提供一个结束游戏的按钮；游戏共9个回合，如果都猜不中则是输了，如果猜中则赢得游戏，两种情况都会弹出游戏结束场景（如图5）。这个场景需要告诉用户最终结果是什么，花费多长时间，以及提供一个重新开始游戏的操作。最后一个场景是帮助场景（如图6），它其中只需要告诉用户这个游戏的玩法，以及提供返回菜单场景的操作就行。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58184e7b1f77e.png" alt="图3 菜单场景" title="图3 菜单场景" />

图3 菜单场景

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58184e849fe52.png" alt="图4 游戏场景" title="图4 游戏场景" />

图4 游戏场景

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58184e911abc1.png" alt="图5 结束场景" title="图5 结束场景" />

图5 结束场景

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58184e9df2fd9.png" alt="图6 帮助场景" title="图6 帮助场景" />

图6 帮助场景

接着来看下如何对场景无障碍化，举个例子，对菜单场景进行无障碍化。首先，需要去掉原来的大焦点；接着，为开始游戏按钮和怎么玩按钮提供无障碍焦点和无障碍信息，也就是说，要构造两个无障碍节点。对 AccessibilityGameView 添加两个无障碍虚拟节点。这样，用户就能操作到场景中这两个按钮。

进入到游戏代码中，两个按钮是 CCMenuItemLabel 控件，通过调用 rect()方法，可以获得两个按钮在屏幕中的大小。

```
……
m_pItemMenu = CCMenu::create();
    for (int i = 0; i < menuCount; ++i) {
 CCLabelTTF* label;
label = textAddOutline(menuNames[i].c_str(),
                                       "fonts/akaDylan Plain.ttf", 30, ccWHITE, 1);
        CCMenuItemLabel* pMenuItem = CCMenuItemLabel::create(label, this,
menu_selector(HelloWorld::menuCallback));
m_pItemMenu->addChild(pMenuItem, i + 10000);
pMenuItem->setPosition(
                ccp( VisibleRect::center().x, (VisibleRect::bottom().y + (menuCount - i) * LINE_SPACE) ));
CCRect rect = pMenuItem->rect();
const char * str = GameConstants::getMenuNodeDesc(i);
AccessibilityWrapper::getInstance()->addMenuSceneRect(i, str, rect.getMinX(),rect.getMaxX(),rect.getMinY(),rect.getMaxY());
```

在取得按钮的大小后，将按钮描述 str 和 rect 交给 AccessibilityWrapper。AccessibilityWrapper 将会通过 JNI 的方法调用 Java 层代码，通知 BaseSceneHelper 来构造无障碍虚拟节点。

```
public class BaseSceneHelper extends ExploreByTouchHelper {
 
protected ArrayList<accessibilityitem> mNodeItems;
 
    ....
     
public void updateAccessibilityItem(int i, String desc){
        if(mNodeItems.size() > i){
            AccessibilityItem item = mNodeItems.get(i);
item.mDesc = desc;
        }
    }
    public void addAccessibilityItem(AccessibilityItem item) {
        mNodeItems.add(item);
    }
    public void destroyScene() {
        mNodeItems.clear();
    }
}
static AccessibilityWrapper * s_Instance = NULL;
 
AccessibilityWrapper * AccessibilityWrapper::getInstance(){
if(s_Instance == NULL){
s_Instance = new AccessibilityWrapper();
    }
 
    return s_Instance;
}
 
void AccessibilityWrapper::addMenuSceneRect(int i, const char * s, float l, float r, float t, float b){
    JniMethodInfo minfo;
    bool isHave = JniHelper::getStaticMethodInfo(minfo,
            "cn/robust/mastermind/AccessibilityHelper","addMenuSceneRect","(ILjava/lang/String;IIII)V");
    if(!isHave){
            //CCLog("jni:openURL 函数不存在");
    }else{
        int left = (int)l;
        int right = (int)r;
        int top = (int)t;
        int bottom = (int) b;
        jstring jstr = minfo.env->NewStringUTF(s);
minfo.env->CallStaticVoidMethod(minfo.classID,minfo.methodID, i, jstr, left, right, top, bottom);
    }
}
```

Java 层被调用的类是 AccessibilityHelper，在 addMenuSceneRect 方法中构造 AccessibilityItem，放到菜单页面的 BaseSceneHelper 中。

```
public class AccessibilityHelper {
    private static BaseSceneHelper mMenuRef;
    private static BaseSceneHelper mPlayRef;
    private static BaseSceneHelper mOverRef;
    private static BaseSceneHelper mHelpRef;
    private static BaseSceneHelper mCurRef;
    ……
public static void addMenuSceneRect(int i, String d, int l, int r, int t, int b){
        int sl = getScreenX(l);
        int sr = getScreenX(r);
        int st = getScreenY(b);
        int sb = getScreenY(t);
        AccessibilityItem item = new AccessibilityItem(i, d, sl, sr, st, sb);
        if(mMenuRef != null){
            mMenuRef.addAccessibilityItem(item);
        }
    }
}
```

游戏跟应用的无障碍化不同，它只有一个 activity，场景切换并不会改变 activity。所以需要在上个场景结束时，把上个场景的无障碍虚拟节点删除，在下个场景出现之前，把下个场景的无障碍虚拟节点构造好。具体代码如下：

```
public static void onMenuSceneLoad(int scene){
if(mGameViewRef.get() != null && mMenuRef != null && 
mPlayRef != null && mOverRef != null){
switch (scene) {
case 0:
                ViewCompat.setAccessibilityDelegate(mGameViewRef.get(), mMenuRef);
                handleNewScene(mMenuRef);
break;
case 1:
                ViewCompat.setAccessibilityDelegate(mGameViewRef.get(), mPlayRef);
                handleNewScene(mPlayRef);
break;
case 2:
                ViewCompat.setAccessibilityDelegate(mGameViewRef.get(), mOverRef);
                handleNewScene(mOverRef);
break;
case 3:
                ViewCompat.setAccessibilityDelegate(mGameViewRef.get(), mHelpRef);
                handleNewScene(mHelpRef);
            }
        }
    }
     
private static void 
handleNewScene(BaseSceneHelper newScene){
if(mCurRef != null ){
mCurRef.destroyScene();
        }
        mCurRef = newScene;
        mGameViewRef.get().setCurSceneHelper(newScene);
         
    }
```

### 游戏引擎的特殊处理

前面讲到的构造无障碍节点和处理场景切换，这两个流程都是从游戏代码中发起。而 Cocos2d-x 引擎中游戏代码是 C++ 写的，所以这里通过 jni 技术，在 C++ 代码中调用 Android 工程的 Java 代码，把构造无障碍节点和处理场景切换两件事告诉上层，并展示到 UI 上（如图7）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58184eb25e312.png" alt="图7 引擎无障碍化通讯流程" title="图7 引擎无障碍化通讯流程" />

图7 引擎无障碍化通讯流程

然而，在游戏世界中的坐标系与在手机中坐标系有些区别，如图8所示，在游戏中坐标原点是在左下角，而在手机屏幕中坐标原点在左上角。所以按钮的边框 R1 要转换成边框 R2，需要对左上角坐标点和右下角坐标点进行变换。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58184ec3eb17f.png" alt="图8 坐标转换图" title="图8 坐标转换图" />

图8 坐标转换图

比如左上角的点，在游戏中是(x1, y1)，y1 的值是红线段的长度。在右边，左上角的(x2,y2)，y2的值是蓝色线段的长度。x2 的值通过图8中的公式求出，其中 SW 是手机屏幕的宽度，而游戏场景设定屏幕大小为480 * 720。

要求 y2 的值，先算出 RH 的值，RH 是游戏场景在手机屏幕中的高度。由于游戏是宽度适应，手机上下会有黑边，RH 是手机屏幕高度SH减去黑边 BH，然后就可以根据公式求出 y2 的值。在代码中转换方法如下：

```
private static int getScreenX(int x){
        int ret = x * sWidth / sGameWith;
        return ret;
    }
     
    private static int getScreenY(int y){
        // 针对不同的拉伸方式要有不同的转换，这里是kResolutionShowAll
        int realHeight = sGameHeight * sWidth / sGameWith;
        int blackHeight = (sHeight - realHeight) / 2;
        int y1 = sGameHeight - y;
        int ret = y1 * realHeight / sGameHeight + blackHeight;
        return ret;
    }
```

经过坐标转换后，屏幕上的无障碍焦点就能完美的盖在菜单按钮上。当无障碍用户双击操作则会触发菜单按钮的点击操作。当用户触摸到菜单按钮位置时，则菜单按钮获取无障碍焦点。

### 总结

从猜地鼠游戏无障碍化看出，手机游戏要实现无障碍化并非难事。首先要为游戏界面添加自定义 View，并且对这个自定义 View 进行无障碍化，使得它具有构造出虚拟无障碍节点的能力。然后，就要在游戏代码中计算出要展示给用户的无障碍节点的边框大小，经过坐标的转换，计算出在手机屏幕中展示的边框大小。将边框大小与描述信息，通过JNI层，通知到界面。以此类推，在各个场景中都进行无障碍焦点边框的处理，并且在切换场景时，把上个场景的无障碍焦点都清除掉。每个场景中，随着游戏操作变化，界面上的元素也可能发生变化，这时只要动态的维护场景的无障碍焦点，则用户就能感知游戏的变化。

当然，这种实现方式，目前仅支持变化不多、比较简单的游戏（比如像猜地鼠这样的智力游戏），对于其他类型的游戏，可能需要更多的无障碍化实现，希望这个例子能给予游戏开发者解决灵感，为更多的游戏实现无障碍化。