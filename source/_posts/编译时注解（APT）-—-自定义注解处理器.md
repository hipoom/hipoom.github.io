---
title: 编译时注解（APT） — 自定义注解处理器
date: 2024-09-19 01:06:27
tags:
---

关于编译时注解（APT）由浅入深有三部分，分别是：

**1. [自定义注解处理器](https://www.jianshu.com/p/fb80995daa45)**
例如 ButterKnife、Room 根据注解生成新的类。

**2\. 利用JcTree在编译时修改代码**
像 Lombok 自动往类中新增 getter/setter 方法、往方法中插入代码行等。
这种方式不推荐使用，因为只对 Java 代码有效，对 Kotlin 代码无效。

**3\. [自定义 Gradle 插件在编译时修改代码](https://www.jianshu.com/p/1f580c5d39e1)**
例如一些代码插桩框架、日志框架、方法耗时统计框架等。


这篇文章以 demo 的形式，介绍如何从零开始创建一个自定义的注解处理器，并生成一个新的类。这个类中有一个静态方法，方法返回添加了自定义注解的所有类。 看懂这篇文章，你就能写出自己的 ButterKnife 啦~

本文中的源代码可以在这里查看: [https://github.com/hipoom/APT-Source-Code](https://github.com/hipoom/APT-Source-Code)

---

## 1. 环境搭建和 Gradle 配置

**1.1 创建注解 module**
我们在工程中新建一个 Java Library，module 名称定义为 annotation。 再定义一个自定义的注解类：

```Java
@Target(ElementType.TYPE)
public @interface DemoAnnotation {
}
```
第一步就完啦~ （如果不清楚元注解的使用，可以搜索其它文章了解）


**1.2 创建注解处理器 Module**
在工程中再创建一个 Java Library，名称定义为 annotation-processor，并在 build.gradle 中加入如下依赖：
```groovy
import org.gradle.internal.jvm.Jvm

apply plugin: 'java-library'

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])

    // 刚才定义的 Annotation 模块
    implementation project(":annotation")

    // 谷歌的 AutoService 可以让我们的注解处理器自动注册上
    implementation 'com.google.auto.service:auto-service:1.0-rc4'

    // 用于生成新的类、函数
    implementation "com.squareup:javapoet:1.9.0"

    // 谷歌的一个工具类库
    implementation "com.google.guava:guava:24.1-jre"

    implementation files(Jvm.current().toolsJar)
}

sourceCompatibility = "1.8"
targetCompatibility = "1.8"
```

**1.3 配置项目级的 build.gradle**
再在项目级的 build.gradle 中增加 android-apt 的依赖：
```groovy
buildscript {
    
    repositories { ... }
    
    dependencies {
        ...
        classpath "com.neenbedankt.gradle.plugins:android-apt:1.8"
    }
    
    ...
}

```

---

## 2. 实现自定义注解处理器
所有的自定义注解处理器都应该继承自 `AbstractProcessor` 类。  
我们也定义一个处理器，并实现几个模板方法：
```java
@AutoService(Processor.class)
public class DemoProcessor extends AbstractProcessor {

    /* ======================================================= */
    /* Fields                                                  */
    /* ======================================================= */

    /**
     * 用于将创建的类写入到文件
     */
    private Filer mFiler;


    /* ======================================================= */
    /* Override/Implements Methods                             */
    /* ======================================================= */

    @Override
    public synchronized void init(ProcessingEnvironment environment) {
        super.init(environment);
        mFiler = environment.getFiler();
    }

    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        // 这个方法是注解处理器的核心，稍后单独分析这个方法如何实现
        return false;
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        // 这个方法返回当前处理器 能处理哪些注解，这里我们只返回 DemoAnnotation
        return Collections.singleton(DemoAnnotation.class.getCanonicalName());
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        // 这个方法返回当前处理器 支持的代码版本
        return SourceVersion.latestSupported();
    }
}
```
**2.1 process() 方法详解**
我们的需求是生成一个新的类，类中有一个静态方法，方法返回添加了 @Annotation 注解的所有类。  
这些操作都需要我们在 `process()` 方法中去实现。步骤：
(1) 获取所有添加了注解的元素；
(2) 生成一个方法，方法的代码块是返回(1)中获取到的列表。
(3) 生成一个类，类中加入(2)中生成的方法；
(4) 将(3)中生成的类写入文件。

所以我们得到这个方法的实现：
```java
@Override
public boolean process(Set<? extends TypeElement> set, RoundEnvironment environment) {

    // 获取所有被 @DemoAnnotation 注解的类
    Set<? extends Element> elements = environment.getElementsAnnotatedWith(DemoAnnotation.class);

    // 创建一个方法，返回 Set<Class>
    MethodSpec method = createMethodWithElements(elements);

    // 创建一个类
    TypeSpec clazz = createClassWithMethod(method);

    // 将这个类写入文件
    writeClassToFile(clazz);

    return false;
}
```

接下来就让我们看看这三个关键的方法分别是怎么实现的:

**2.2 如何创建新的方法**

```java
/**
 * 创建一个方法，这个方法返回 elements 中的所有类信息。
 */
private MethodSpec createMethodWithElements(Set<? extends Element> elements) {

    // "getAllClasses" 是生成的方法的名称
    MethodSpec.Builder builder = MethodSpec.methodBuilder("getAllClasses");

    // 为这个方法加上 "public static" 的修饰符
    builder.addModifiers(Modifier.PUBLIC, Modifier.STATIC);

    // 定义返回值类型为 Set<Class>
    ParameterizedTypeName returnType = ParameterizedTypeName.get(
            ClassName.get(Set.class),
            ClassName.get(Class.class)
    );
    builder.returns(returnType);

    // 经过上面的步骤，
    // 我们得到了 public static Set<Class> getAllClasses() {} 这个方法,
    // 接下来我们实现它的方法体：

    // 方法中的第一行: Set<Class> set = new HashSet<>();
    builder.addStatement("$T<$T> set = new $T<>();", Set.class, Class.class, HashSet.class);
    
    // 上面的 "$T" 是占位符，代表一个类型，可以自动 import 包。其它占位符：
    // $L: 字符(Literals)、 $S: 字符串(String)、 $N: 命名(Names)

    // 遍历 elements, 添加代码行
    for (Element element : elements) {

        // 因为 @Annotation 只能添加在类上，所以这里直接强转为 ClassType
        ClassType type = (ClassType) element.asType();

        // 在我们创建的方法中，新增一行代码： set.add(XXX.class);
        builder.addStatement("set.add($T.class)", type);
    }

    // 经过上面的 for 循环，我们就把所有添加了注解的类加入到 set 变量中了，
    // 最后，只需要把这个 set 作为返回值 return 就好了：
    builder.addStatement("return set");

    return builder.build();
}
```

**2.3 如何创建新的类**

```java
/**
 * 创建一个类，并把参数中的方法加入到这个类中
 */
private TypeSpec createClassWithMethod(MethodSpec method) {
    // 定义一个名字叫 OurClass 的类
    TypeSpec.Builder ourClass = TypeSpec.classBuilder("OurClass");

    // 声明为 public
    ourClass.addModifiers(Modifier.PUBLIC);

    // 为这个类加入一段注释
    ourClass.addJavadoc("这个类是自动创建的哦~\n\n @author ZhengHaiPeng");

    // 为这个类新增一个方法
    ourClass.addMethod(method);

    return ourClass.build();
}
```

**2.4 如何将创建的类写入文件**
```java
/**
 * 将一个创建好的类写入到文件中参与编译
 */
private void writeClassToFile(TypeSpec clazz) {
    // 声明一个文件在 "me.moolv.apt" 下
    JavaFile file = JavaFile.builder("me.moolv.apt", clazz).build();

    // 写入文件
    try {
        file.writeTo(mFiler);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

---

## 3. 使用自定义注解处理器
在要使用的 module 中，例如 app，的 build.gradle 中加入依赖：
```groovy
apply plugin: 'com.android.application'

android {
    ...
}

dependencies {
    ...
    annotationProcessor project(":annotation-processor")
    implementation project(path: ':annotation')
}

```
执行 Android Studio 的 `Build` > `Make Project`, 就能在 app module 的 build/source/apt 路径下找到生成的类文件了：
```java
/**
 * 这个类是自动创建的哦~
 *
 * @author ZhengHaiPeng
 */
public class OurClass {
    public static Set<Class> getAllClasses() {
        Set<Class> set = new HashSet<>();
        set.add(MainActivity.class);
        return set;
    }
}
```

这样我们就实现了 自定义注解处理器，并生成代码啦，有疑问留言就好~


---
## 4. 如何为注解处理器传递参数？


APT 中的 processor 可能会用到一些参数，这些参数可以在 gradle 中配置。

**设置参数**
```gradle
android {
    ...
    defaultConfig {
        ...
        javaCompileOptions {
        
            annotationProcessorOptions {
                // 下面定义要传递的参数
                argument "key1", "value1"
                argument "key2", "value2"
            }
        }
    }
```

**获取参数**
在 processor 的 init 方法中可以获取参数：
```
@Override
public synchronized void init(ProcessingEnvironment env) {
    super.init(env);
    
    ...

    String value1 = env.getOptions().get("key1");
    
    ...
}
```




源码：[https://github.com/hipoom/APT-Source-Code](https://github.com/hipoom/APT-Source-Code)
