## 如何编写基于编译时注解的 Android 项目

文/ 张鸿洋

>本文通过具体实例描述了如何编写一个基于编译时注解的项目，归纳起来，主要步骤为：项目结构的划分、注解模块的实现、注解处理器的编写和对外公布的 API 模块编写。通过文本的学习应该能够了解基于编译时注解这类框架运行的原理，以及自己如何编写这样一类框架。

### 概述

在 Android 应用开发中，为了提升开发效率，我们常常会选择使用基于注解的框架，但是由于反射造成一定运行效率的损耗，所以我们会更青睐于编译时注解的框架，例如：

- Butter Knife 免去我们编写 View 的初始化以及事件的注入代码；

- EventBus3 方便我们实现组件间通讯；

- FragmentArgs 轻松地为 fragment 添加参数信息，并提供创建方法；

- ParcelableGenerator 可实现自动将任意对象转换为 Parcelable 类型，方便对象传输。

类似的库还有非常多，大多是为了自动帮我们完成日常编码中需要重复编写的部分（例如：每个 Activity 中的 View 都需要初始化，每个实现 Parcelable 接口的对象都需要编写很多固定写法的代码）。

这里并不是说上述框架就一定没有使用反射了。其实它们内部还是有部分实现依赖于反射，但是很少，且一般做了缓存处理，所以相对来说，效率影响很小。

在使用这类项目时，有时会遇到出现错误难以调试。主要原因还是很多使用者并不了解这类框架内部的原理，所以遇到问题时会消耗大量时间排查。

那么，于情于理，在编译时注解框架这么火的时刻，我们有理由学习这项技术。

**如何编写一个基于编译时注解的项目？**

本文将以编写一个 View 注入的框架为线索，详细介绍编写此类框架的步骤（注：本文使用的 IDE 为 Android Studio）。

### 编写前的准备

在编写此类框架的时候，一般需要建立多个 module，例如本文即将实现的例子：

- ioc-annotation 用于存放注解等，为 Java 模块；

- ioc-compiler 用于编写注解处理器，为 Java 模块；

- ioc-api 用于给用户提供使用的 API，本例为 Andriod 模块；

- ioc-sample 示例，本例为 Andriod 模块；

那么，除了示例模块以外，一般要建立3个 module，module 的名字可以自己考虑，上述给出了一个简单的参考。当然如果条件允许，有的开发者喜欢将存放注解和 API 这两个 module 合并为一个。

对于 module 间的依赖，因为编写注解处理器需要依赖相关注解，所以 ioc-compiler 依赖 ioc-annotation。而我们在使用的过程中，会用到注解以及相关 API，所以 ioc-sample 依赖 ioc-api，ioc-api 依赖 ioc-annotation。

### 注解模块的实现

注解模块，主要用于存放注解类，本例是模仿 Butter Knife 实现 View 注入，所以只需要一个注解类：

```
@Retention(RetentionPolicy.CLASS)
@Target(ElementType.FIELD)
public @interface
BindView
{
     int value();
}
```

我们设置的保留策略为 Class，注解用于 Field 上，在使用该注解时，直接将控件的 id 通过 value 进行设置即可。

开发者在编写时，分析自己需要几个注解类，并且设置合适的 @Target 以及 @Retention 即可。

### 注解处理器的实现

定义完注解后，就可以编写注解处理器了，这块有点复杂，但是也算有章可循。根据上文我们一般把注解处理器独立为一个新的模块，该模块依赖于上文的注解模块，为了方便，还可以引入 auto-service 库。

build.gradle 的依赖情况如下：

```
dependencies
{
     compile 'com.google.auto.service:auto-­‐service:1.0-­‐rc2'
     compile project (':ioc-­‐annotation')
}
```

auto-service 库可以帮我们生成注解处理器编写过程中所需的 META-INF 等信息。

#### 基本代码

注解处理器一般继承于 AbstractProcessor，刚才我们说有章可循，是因为部分代码的写法基本是固定的，如下：

```
@AutoService(Processor.class)
public class IocProcessor extends AbstractProcessor{
    private Filer mFileUtils;
    private Elements mElementUtils;
private Messager mMessager;
    @Override
    public synchronized void init(ProcessingEnvironment processingEnv){
        super.init(processingEnv);
        mFileUtils = processingEnv.getFiler();
        mElementUtils = processingEnv.getElementUtils();
        mMessager = processingEnv.getMessager();
    }
    @Override
    public Set<string> getSupportedAnnotationTypes(){
        Set<string> annotationTypes = new LinkedHashSet<string>();
        annotationTypes.add(BindView.class.getCanonicalName());
        return annotationTypes;
    }
    @Override
    public SourceVersion getSupportedSourceVersion(){
        return SourceVersion.latestSupported();
    }
    @Override
    public boolean process(Set<!--? extends TypeElement--> annotations, RoundEnvironment roundEnv){
    }</string></string></string>
```

在 AbstractProcessor 后，process()方法是必须实现的，也是我们编写代码的核心部分，后面会介绍。

我们一般会实现 getSupportedAnnotationTypes()和 getSupportedSourceVersion()两个方法，这两个方法一个返回支持的注解类型，一个返回支持的源码版本，参考上面的代码，写法基本是固定的。

除此以外，我们还会选择复写 init()方法，该方法传入一个参数 processingEnv，可以帮助我们初始化辅助类：

- Filer mFileUtils：跟文件相关的辅助类，生成JavaSourceCode；

- Elements mElementUtils：跟元素相关的辅助类，帮助我们去获取一些元素相关的信息；

- Messager mMessager：跟日志相关的辅助类；

这里简单提一下 Elemnet，我们简单认识下它的几个子类。

Element 

```
- VariableElement //一般代表成员变量
  - ExecutableElement //一般代表类中的方法
  - TypeElement //一般代表代表类
  - PackageElement //一般代表Package
```

#### process 的实现

process 中的实现，相比较会复杂一点，一般可以归结为两个步骤：

- 收集信息；

- 生成代理类（本文把编译时生成的类叫代理类）。

什么叫收集信息呢？就是根据注解声明，拿到对应的 Element，然后获取所需要的信息，信息肯定是为了后面生成 JavaFileObject 所准备的。

拿本例来说，我们会针对每个类生成一个代理类，比如 MainActivity 会生成一个 MainActivity$$ViewInjector。那么如果多个类中声明了注解，就对应了多个类，这里就需要：

- 一个类对象，代表具体某个类的代理类生成的全部信息，本例中为 ProxyInfo；

- 一个集合，存放上述类对象（到时候遍历生成代理类），本例中为 Map<String, ProxyInfo>，key 为类的全路径。

这里的描述有点模糊没关系，一会儿结合代码就好理解了。

#### 收集信息

```
private Map<string, proxyinfo=""> mProxyMap = new HashMap<string, proxyinfo="">();
@Override
public boolean process(Set<!--? extends TypeElement--> annotations, RoundEnvironment roundEnv){
    mProxyMap.clear();
    Set<!--? extends Element--> elements = roundEnv.getElementsAnnotatedWith(BindView.class);
    //一、收集信息
    for (Element element : elements){
        //检查element类型
        if (!checkAnnotationUseValid(element)){
            return false;
        }
        //field type
        VariableElement variableElement = (VariableElement) element;
        //class type
        TypeElement typeElement = (TypeElement) variableElement.getEnclosingElement();//TypeElement
        String qualifiedName = typeElement.getQualifiedName().toString();
 
        ProxyInfo proxyInfo = mProxyMap.get(qualifiedName);
        if (proxyInfo == null){
            proxyInfo = new ProxyInfo(mElementUtils, typeElement);
            mProxyMap.put(qualifiedName, proxyInfo);
        }
        BindView annotation = variableElement.getAnnotation(BindView.class);
        int id = annotation.value();
        proxyInfo.mInjectElements.put(id, variableElement);
 }
    return true;
}</string,></string,>
```

首先我们调用一下 mProxyMap.clear();，因为 process 可能会多次调用，避免生成重复的代理类而造成类名已存在异常。

然后，以 roundEnv.getElementsAnnotatedWith 拿到我们通过 @BindView 注解的元素。该方法的返回值，按照预期应该是 VariableElement 集合，因为定义的 @BindView 是用于类中的成员变量上的。

接下来，for 循环我们的元素，首先检查类型是否为 VariableElement。

拿到对应的类信息 TypeElement，继而生成 ProxyInfo 对象。此处通过一个 mProxyMap 进行存储，key 为 qualifiedName，即类的全路径。防止一个类生成多个 ProxyInfo 对象，ProxyInfo 与类在本例中是一一对应的。

接下来，会将与 ProxyInfo 对应类中被 @BindView 声明的 VariableElement 以键值对的方式加入到 ProxyInfo 中去，键为我们声明时设置的控件的 id。

这样就完成了信息收集，随后便可去生成代理类了。

#### 生成代理类

```
@Override
public boolean process(Set<!--? extends TypeElement--> annotations, RoundEnvironment roundEnv){
    //...省略收集信息的代码，以及try,catch相关
    for(String key : mProxyMap.keySet()){
        ProxyInfo proxyInfo = mProxyMap.get(key);
        JavaFileObject sourceFile = mFileUtils.createSourceFile(
                proxyInfo.getProxyClassFullName(), proxyInfo.getTypeElement());
            Writer writer = sourceFile.openWriter();
            writer.write(proxyInfo.generateJavaCode());
 writer.flush();
            writer.close();
    }
    return true;
}
```

可以看到生成代理类的代码非常简短，主要就是遍历我们的 mProxyMap，然后取得每一个 ProxyInfo。最后通过 mFileUtils.createSourceFile 来创建文件对象，类名为 proxyInfo.getProxyClassFullName()，写入的内容为 proxyInfo.generateJavaCode()。

看来生成 Java 代码的方法都在 ProxyInfo 里面。

#### 生成 Java 代码

这里我们主要关注其如何生成 Java 代码。

```
#ProxyInfo
//key为id，value为对应的成员变量
public Map<integer, variableelement=""> mInjectElements = new HashMap<integer, variableelement="">();
public String generateJavaCode(){
    StringBuilder builder = new StringBuilder();
    builder.append("package " + mPackageName).append(";\n\n");
    builder.append("import com.zhy.ioc.*;\n");
    builder.append("public class ").append(mProxyClassName).append(" implements " + SUFFIX + "<" + mTypeElement.getQualifiedName() + ">");
    builder.append("\n{\n");
    generateMethod(builder);
    builder.append("\n}\n");
    return builder.toString();
}
private void generateMethod(StringBuilder builder){
     builder.append("public void inject("+mTypeElement.getQualifiedName()+" host , Object object )");
    builder.append("\n{\n");
    for(int id : mInjectElements.keySet()){
        VariableElement variableElement = mInjectElements.get(id);
        String name = variableElement.getSimpleName().toString();
        String type = variableElement.asType().toString() ;
        builder.append(" if(object instanceof android.app.Activity)");
        builder.append("\n{\n");
        builder.append("host."+name).append(" = ");
        builder.append("("+type+")(((android.app.Activity)object).findViewById("+id+"));");
        builder.append("\n}\n").append("else").append("\n{\n");
        builder.append("host."+name).append(" = ");
        builder.append("("+type+")(((android.view.View)object).findViewById("+id+"));");
        builder.append("\n}\n");
    }
    builder.append("\n}\n");
}</integer,></integer,>
```

这里主要就是靠收集到的信息，拼接完成的代理类对象了。看起来会比较头疼，不过给出一个生成后的代码对比着看，会好很多。

```
package com.zhy.ioc_sample;
import com.zhy.ioc.*;
public class MainActivity$$ViewInjector implements ViewInjector<com.zhy.ioc_sample.mainactivity>{
    @Override
    public void inject(com.zhy.sample.MainActivity host , Object object ){
        if(object instanceof android.app.Activity){
            host.mTv = (android.widget.TextView)(((android.app.Activity)object).findViewById(2131492945));
        }
        else{
            host.mTv = (android.widget.TextView)(((android.view.View)object).findViewById(2131492945));
        }
    }
}</com.zhy.ioc_sample.mainactivity>
```

对比看其实就是根据收集到的成员变量（通过 @BindView 声明的），以及具体要实现的需求生成 Java 代码。

需要注意，生成的代码实现了一个接口 ViewInjector<T>，该接口是为了统一所有的代理类对象的类型。因为我们在使用的时候，需要强转代理类对象为该接口类型，调用其方法（在本文 API 模块有相关的示例）。接口上支持的泛型，实际使用时会传入实际类对象，例如 MainActivity。因为我们在生成代理类中的代码，会通过“实际类.成员变量”的方式进行访问。所以，使用编译时注解的成员变量一般都不允许 private 修饰符修饰（有的允许，但是需要提供可访问到的 getter、setter 方法）。

并且，这里采用了完全拼接的方式编写 Java 代码，也可以使用一些开源库，来通过 Java API 的方式来生成代码，例如：

- JavaPoet：A Java API for generating .java source files.

到此就完成了代理类的生成，并且任何的注解处理器的编写方式基本都遵循着收集信息、生成代理类的步骤。

### API 模块的实现

有了代理类之后，我们一般还会提供 API 供用户去访问，例如本例的访问入口是：

```
//Activity中
 Ioc.inject(Activity);
 //Fragment中，获取ViewHolder中
 Ioc.inject(this, view);
```

模仿了 Butter Knife，第一个参数为宿主对象，第二个参数为实际调用 findViewById 的对象。当然，在 Actiivty 中，两个参数就一样了。

那么，API 一般如何编写呢？

其实很简单，只要了解了其原理，这个 API 就干两件事：

- 根据传入的 host 寻找生成的代理类，例如 MainActivity->MainActity$$ViewInjector；

- 强转为统一的接口，调用接口提供的方法。

这两件事应该不复杂，第一件事是拼接代理类名，然后反射生成对象，第二件事为强转调用。

```
public class Ioc{
    public static void inject(Activity activity){
        inject(activity , activity);
    }
    public static void inject(Object host , Object root){
        Class<!--?--> clazz = host.getClass();
        String proxyClassFullName = clazz.getName()+"$$ViewInjector";
       //省略try,catch相关代码 
        Class<!--?--> proxyClazz = Class.forName(proxyClassFullName);
        ViewInjector viewInjector = (com.zhy.ioc.ViewInjector) proxyClazz.newInstance();
        viewInjector.inject(host,root);
    }
}
public interface ViewInjector<t>{
    void inject(T t , Object object);
}</t>
```

代码很简单，拼接代理类的全路径，然后通过 newInstance 生成实例，并强转，调用代理类的 inject 方法。

此处一般情况会对生成的代理类做缓存处理，比如使用 Map 存储，没有再生成这样我们就完成了一个编译时注解框架的编写。以上所有代码已上传至 GitHub，地址：https://github.com/hymanAndroid/ioc-apt-sample
