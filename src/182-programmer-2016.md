## Android 无障碍宝典

文/何金源

Android 江湖上一直流传着一部秘籍——Android 无障碍宝典。传闻练成这部宝典，可在 Android 无障碍模式下，飞檐走壁，能人所不能。宝典分为三篇，分别是入门、进阶和高级，由浅入深，全面展示无障碍的基本方法及扩展应用。

Android 应用无障碍化，目的是为视觉障碍或其他有障碍的用户提供更好的服务。在无障碍模式下，用户的操作方式与平常不同，比如：

>选择（Hover）一个元素：单击

>点击（Click）一个元素：双击

>滚动：双指往上、下、左、右

>选择上或下一个项目：单指往上、下、左、右

>快速回到主画面：单指上滑+左滑

>返回键：单指下滑+左滑

>最近画面键：单指左滑+上滑

>通知栏：单指右滑+下滑

此外，还需要理解在无障碍模式下“无障碍焦点”这个概念。如图1所示，界面上以绿色方框来表示目前获得无障碍焦点的 View。拥有无障碍焦点的 View，会被 TalkBack 服务识别，TalkBack 会从 View 中取出相关的无障碍内容，然后提示给用户。

有了对无障碍模式初步的了解，就可以正式开始学习如何为应用无障碍化。

### 入门篇

#### 为 View 添加 ContentDescription

UI上的可操作元素都应该添加上 ContentDescription, 当此元素获得无障碍焦点时，TalkBack 服务就取出 View 的提示语（contentDescription），并朗读出来。

添加 ContentDescription 有两种方法，第一种是通过在 XML 布局中设置 android:contentDescripton 属性，如：

```
<Button
    android:id=”@+id/pause_button”
    android:src=”@drawable/pause”
    android:contentDescription=”@string/
pause”/>
```

但是很多情况下，View 的内容描述会根据不同情景需要而改变，比如 CheckBox 按钮是否被选中，以及 ListView 中 item 的内容描述等。这种则需要在代码中使用 setContentDescription 方法，如：

```
String contentDescription = "已选中 " + strValues[position];
label.setContentDescription(contentDescription);
```

#### 设置无障碍焦点

UI 上的元素，有的默认带有无障碍焦点，如 Button、CheckBox 等标准控件，有的如果不设置 contentDescription 是默认没有无障碍焦点。在开发应用过程中，还会遇到一些 UI 元素，是不希望它获取无障碍焦点的。以下方法可以改变元素的无障碍焦点：

```
public void setAccessibilityFocusable(View view, boolean focused){
    if(android.os.Build.VERSION.SDK_INT >= 16){
        if(focused){
            ViewCompat.setImportantForAccessibility(view, ViewCompat.IMPORTANT_FOR_ACCESSIBILITY_YES);
        }else{
            ViewCompat.setImportantForAccessibility(view, ViewCompat.IMPORTANT_FOR_ACCESSIBILITY_NO);
        }
    }
}
```

IMPORTANT\_FOR\_ACCESSIBILITY\_YES 表示这个元素应该有无障碍焦点，会被 TalkBack 服务读出描述内容；IMPORTANT\_FOR\_ACCESSIBILITY\_NO 表示屏蔽元素的无障碍焦点，手指滑动遍历及触摸此元素，都不会获得无障碍焦点，TalkBack 服务也不会读出其描述内容。

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/579f16f091c85.png" alt="图1  无障碍焦点" title="图1  无障碍焦点" />

图1  无障碍焦

#### 发出无障碍事件

IMPORTANT\_FOR\_ACCESSIBILITY\_YES 表示这个元素应该有无障碍焦点，会被 TalkBack 服务读出描述内容；IMPORTANT\_FOR\_ACCESSIBILITY\_NO 表示屏蔽元素的无障碍焦点，手指滑动遍历及触摸此元素，都不会获得无障碍焦点，TalkBack 服务也不会读出其描述内容。

```
view.postDelayed(new Runnable() {
    @Override
    public void run() {
        if(android.os.Build.VERSION.SDK_INT >= 14){
            view.sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_HOVER_ENTER);
        }
    }
},100);
```

这个方法是让 View 来自动发出被单击选中的无障碍事件，发出后，UI上的无障碍焦点则会马上赋给这个 View，从而达到抢无障碍焦点的效果。再比如：

```
if(android.os.Build.VERSION.SDK_INT >= 16){
    AccessibilityEvent event = AccessibilityEvent.obtain(AccessibilityEvent.TYPE_ANNOUNCEMENT);
    event.setPackageName(view.getContext().getPackageName());
    event.setClassName(view.getClass().getName());
    event.setSource(view);
    event.getText().add(desc);
    view.getParent().requestSendAccessibilityEvent(view, event);
}
```

AccessibilityEvent.TYPE\_ANNOUNCEMENT 是代表元素需要 TalkBack 服务来读出描述内容。其中 desc 是描述内容，将它放到 event 的 getText()中，然后请求 View 的父类来发出事件。

### 进阶篇

#### 介绍 AccessibilityDelegate

Android 中 View 含有 AccessibilityDelegate 这个子类，它可被注册进 View 中，主要作用是为了增强对无障碍化的支持。

查看 View 的源码可发现，注册 Accessibility

Delegate 方法很简单：

```
Public void setAccessibilityDelegate(AccessibilityDelegate delegate) {
    mAccessibilityDelegate = delegate;
}
```

注册后，View 对无障碍的处理，则会交给 AccessibilityDelegate，如：

```
public void onInitializeAccessibilityNodeInfo(AccessibilityNodeInfo info) {
    if (mAccessibilityDelegate != null) {
        mAccessibilityDelegate.onInitializeAccessibilityNodeInfo(this, info);
    } else {
        onInitializeAccessibilityNodeInfoInternal(info);
    }
}
```

onInitializeAccessibilityNodeInfo 是 View 源码中初始化无障碍节点信息的方法，从上面代码看出，当 mAccessibilityDelegate 是开发注册的 AccessiblityDelegate 时，则会执行 AccessiblityDelegate 中的 onInitializeAccessibilityNodeInfo 方法。再看看 AccessibilityDelegate 类中的 onInitializeAccessibilityNodeInfo：

```
public void onInitializeAccessibilityNodeInfo(View host, AccessibilityNodeInfo info) {
    host.onInitializeAccessibilityNodeInfoInternal(info);
}
```

Host 是被注册 AccessibilityDelegate 的 View，onInitializeAccessibilityNodeInfoInternal 是 View 中真正初始化无障碍节点信息的方法。即是说，注册了 AccessibilityDelegate 并没有改变 View 原来对无障碍的操作，而是在这个操作之后增加了处理。

#### AccessibilityDelegate 的应用

下面介绍下 AccessibilityDelegate 可以提供哪些无障碍应用，注册 AccessibilityDelegate 是在 API 14 以上才开放的接口，API 14 以下如需使用，可以接入 support v4 包中的AccessibilityDelegateCompt。注册方法如下：

```
if (Build.VERSION.SDK_INT >= 14) {
     View view = findViewById(R.id.view_id);
     view.setAccessibilityDelegate(new AccessibilityDelegate() {
         public void onInitializeAccessibilityNodeInfo(View host,
                 AccessibilityNodeInfo info) {
             super.onInitializeAccessibilityNodeInfo(host, info);
             // 对info做出扩展性支持
     });
 }
```

要对 info 做出扩展支持，还得先了解 AccessibilityNodeInfo 这个类。Android 开发都知道，UI 上的元素是通过 View 来实现，而 AccessibilityNodeInfo 则是存储 View 的无障碍信息（如contentDescription）及无障碍状态（如 focusable、visiable、clickable 等），同时它还肩负着 TalkBack 服务和 View 之间通讯的桥梁作用。如果要修改 View 的无障碍提示，比如修改 View 的类型提示，可以这样做：

```
if(android.os.Build.VERSION.SDK_INT >= 14){
    view.setAccessibilityDelegate(new AccessibilityDelegate(){
        @Override
        public void onInitializeAccessibilityNodeInfo(View host, AccessibilityNodeInfo info) {
            super.onInitializeAccessibilityNodeInfo(host, info);
            if(contentDesc != null) {
                info.setContentDescription(contentDesc);
            }
            info.setClassName(className);
        }
    });
}
```

className 是类型名称，这里如果是 Button.class.getName()，则 TalkBack 会对这个 View 读“XXX 按钮”，“XXX”是 contentDescription，而“按钮”则是 TalkBack 服务添加的（如果是英文环境，则是“XXX button”）。使用上面的方法，可以为非按钮控件加上“按钮”的提示，方便无障碍用户识别 UI 上元素的作用，同时又不必把“按钮”提示强加入 contentDescription 中。除了修改 AccessibilityNodeInfo 外，使用 AccessibilityDelegate 还可以影响无障碍事件，如：

```
if (android.os.Build.VERSION.SDK_INT >= 14) {
    view.setAccessibilityDelegate(new AccessibilityDelegate() {
 
        @Override
        public void sendAccessibilityEvent(View host, int eventType) {
    // 弹出Popup后，不自动读各项内容
            if (eventType != AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED) {
                super.sendAccessibilityEvent(host, eventType);
            }
        }
    });
}
```

无障碍模式下，弹出 Dialog，则会把 Dialog 中的所有元素都读一遍。这个方法可以把弹起窗口的无障碍事件拦截，Dialog 弹起就不会再自动读各项内容。

### 高级篇

有了 AccessibilityDelegate 这把利器之后，开发可以轻松对应 Android 中大部分的无障碍化，但是如果想要做到游刃有余，还得深造更高级的功夫——自定义 View 无障碍化。

应用开发过程中，总会需要自定义 View 来实现特殊的 UI 效果，当一个自定义 View 中包含多种 UI 元素时，无障碍模式下并不能区分包含的多种 UI 元素，而只为自定义 View 添加一个大无障碍焦点。如图2所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/579f1721f1be3.png" alt="图2  自定义View只有一个大无障碍焦点" title="图2  自定义View只有一个大无障碍焦点" />

图2  自定义 View 只有一个大无障碍焦点

图中只有一个大无障碍焦点，因为这是一个 View，里面的文字及蓝色的矩形都是绘制出来的。

```
@Override
public void onDraw(Canvas c) {
    super.onDraw(c);
    if (mTitle != null) {
        drawTitle(c);
    }
    for (int i = 0; i < mSize; i++) {
        drawBarAtIndex(c, i);
    }
    drawAxisY(c);
}
```

代码中绘制出来的元素不可被 TalkBack 识别出来，所以开发需要多做一步，对自定义 View 无障碍化。这里将介绍如何通过官方提供的 ExploreByTouchHelper 来实现。

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/579f17952acbd.png" alt="图3  自定义View内元素获得无障碍焦点" title="图3  自定义View内元素获得无障碍焦点" />

图3是使用 ExploreByTouchHelper 实现的最终效果，每一个小矩形都能获取到无障碍焦点，且可以进行选中高亮。实现 ExploreByTouchHelper 需要五步。

第一步，委托处理无障碍。

```
public class BarGraphView extends View {
    private final BarGraphAccessHelper mBarGraphAccessHelper;
    public BarGraphView(Context context, AttributeSet attrs, int defStyle) {
            super(context, attrs, defStyle);
            ...
            mBarGraphAccessHelper = new BarGraphAccessHelper(this);
            ViewCompat.setAccessibilityDelegate(this, mBarGraphAccessHelper);
    }
     
    @Override
    public boolean dispatchHoverEvent(MotionEvent event) {
        if ((mBarGraphAccessHelper != null)
                && mBarGraphAccessHelper.dispatchHoverEvent(event)) {
            return true;
        }
        return super.dispatchHoverEvent(event);
    }
}
```

mBarGraphAccessHelper 继承 ExploreByTouchHelper，可通过注册 AccessibilityDelegate 的方式来注册给自定义 BarGraphView，同时让  mBarGraphAccessHelper 来处理 Hover 事件（无障碍模式下的点击）的分发。

第二步，标记无障碍虚拟节点 ID。

```
private class BarGraphAccessHelper extends ExploreByTouchHelper {
    private final Rect mTempParentBounds = new Rect();
    public BarGraphAccessHelper(View parentView) {
        super(parentView);
    }
    @Override
    protected int getVirtualViewIdAt(float x, float y) {
        final int index = getBarIndexAt(x, y);
        if (index >= 0) {
            return index;
        }
        return ExploreByTouchHelper.INVALID_ID;
    }
    @Override
    protected void getVisibleVirtualViewIds(List<integer> virtualViewIds) {
        final int count = getBarCount();
        for (int index = 0; index < count; index++) {
        virtualViewIds.add(index);
        }
    }
}
```

getVirtualViewIdAt 和 getVisibleVirtualViewIds 都是 ExploreByTouchHelper 类需要实现的方法，分别代表获取虚拟无障碍节点的 id 以及设置虚拟无障碍节点的 id。由于自定义 View 里的元素非继承于 View，如要在无障碍模式下被识别，则需要构造一个虚拟无障碍节点。构造方法已封装到 ExploreByTouchHelper 里，开发只需要告诉 ExploreByTouchHelper 有哪些虚拟无障碍节点的 id 即可。无障碍节点 id 需要满足以下条件：id 是一个接一个的，稳定且为非负整数。设置好无障碍虚拟节点 id 后，根据用户操作 UI 上的 xy 坐标，取得对应的无障碍虚拟节点 id，通过 getVirtualViewIdAt 方法告诉 ExploreByTouchHelper 类。

第三步，填充无障碍节点的属性。

```
private class BarGraphAccessHelper extends ExploreByTouchHelper {
    ...
    private CharSequence getDescriptionForIndex(int index) {
        final int value = getBarValue(index);
        final int templateRes = ((mHighlightedIndex == index) ?
                R.string.bar_desc_highlight : R.string.bar_desc);
        return getContext().getString(templateRes, index, value);
    }
 
    @Override
    protected void populateEventForVirtualViewId(int virtualViewId, AccessibilityEvent event) {
        final CharSequence desc = getDescriptionForIndex(virtualViewId);
        event.setContentDescription(desc);
    }
 
    @Override
    protected void populateNodeForVirtualViewId(
            int virtualViewId, AccessibilityNodeInfoCompat node) {
        final CharSequence desc = getDescriptionForIndex(virtualViewId);
        node.setContentDescription(desc);
        node.addAction(AccessibilityNodeInfoCompat.ACTION_CLICK);
        final Rect bounds = getBoundsForIndex(virtualViewId, mTempParentBounds);
        node.setBoundsInParent(bounds);
    }
}
```

构造了虚拟无障碍节点后，便可以往节点里塞无障碍信息。populateEventForVirtualViewId 是将无障碍信息填入无障碍事件中。populateNodeForVirtualViewId 是初始化每个虚拟无障碍节点，设置 contentDescription，注册所需要处理的 Action，以及设置无障碍焦点的边框。setBoundsInParent 一定要设置有效的边框，否则会导致虚拟无障碍节点无法获取无障碍焦点。

第四步，提供用户无障碍交互支持。

```
private class BarGraphAccessHelper extends ExploreByTouchHelper {
    ...
    @Override
    protected boolean performActionForVirtualViewId(
        int virtualViewId, int action, Bundle arguments) {
        switch (action) {
        case AccessibilityNodeInfoCompat.ACTION_CLICK:
            onBarClicked(virtualViewId);
            return true;
        }
 
        return false;
    }
}
 
private void onBarClicked(int index) {
    setSelection(index);
    if (mBarGraphAccessHelper != null) {
        mBarGraphAccessHelper.sendEventForVirtualViewId(
                index, AccessibilityEvent.TYPE_VIEW_CLICKED);
    }
}
```

小矩形点击后是会被选中且高亮的，在 performActionForVirtualViewId 中实现对应的点击事件处理。经过这四步，自定义 View 就可以完美支持无障碍化了！

可以看出，ExploreByTouchHelper 简化了虚拟节点层次结构的构造，封装 AccessibilityNodeProvider 的实现，更完善的控制 Hover 事件、无障碍事件。有了它，Android 无障碍化再也不是难题。

Android 无障碍化宝典的内容就介绍到此，在实际开发中，遇到的无障碍化问题都比较细小和琐碎，希望以上介绍能提供一点帮助。很多 Android 开发以为无障碍化就是为控件加上 ContentDescription，其实还有空描述、混乱焦点、焦点顺序、描述准确性等地方需要注意和优化。只有用心、持续地改进和优化，才能做出真正无障碍的产品。