---
title: 编译时注解（APT） — 基于 Javassist 的代码注入
date: 2024-09-19 01:13:23
tags:
---

关于编译时注解（APT）由浅入深有三部分，分别是：

**1. [自定义注解处理器](https://www.jianshu.com/p/fb80995daa45)**
例如 ButterKnife、Room 根据注解生成新的类。

**2. 利用JcTree在编译时修改代码**
像 Lombok 自动往类中新增 getter/setter 方法、往方法中插入代码行等。
这种方式不推荐使用，因为只对 Java 代码有效，对 Kotlin 代码无效。

**3. 自定义 Gradle 插件在编译时修改代码**  （本文）
例如一些代码插桩框架、日志框架、方法耗时统计框架等。


---

## 1. 环境搭建及Gradle配置

### 1.1 配置 Project 级的 build.gradle:

```groovy
buildscript {
    // 省略...
    
    dependencies {
        // Gradle相关api
        classpath "com.android.tools.build:gradle:3.3.2"
        
        // 注解处理器 相关
        classpath "com.neenbedankt.gradle.plugins:android-apt:1.8"

        // 支持 Kotlin
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:1.3.31"
        
        // javassist
        classpath "org.javassist:javassist:3.21.0-GA"
    }
}
```

### 1.2 配置实现插件的 Module
创建一个 Java Library 或者 Android Library。

#### 1.2.1 修改 module 的 build.gradle
修改 build.gradle 文件如下：

```groovy
apply plugin: 'java'
apply plugin: 'kotlin'

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])

    // 引入 Gradle 的 SDK
    implementation gradleApi()
    
    // 引入 Transform 相关API
    implementation "com.android.tools.build:gradle:" + Gradle_Version

    // Javassist
    implementation 'org.javassist:javassist:3.21.0-GA'
}

repositories {
    google()
    jcenter()
    mavenLocal()
}

sourceCompatibility = "1.8"
targetCompatibility = "1.8"
```


#### 1.2.2 创建 META-INF
创建 resources 文件夹，整体结构如下：
![文件结构](./1.webp)



1. 其中文件 `your.gradle.plugin.name.properties` 表明当前 module 下有一个gradle插件，插件的名称是 `your.gradle.plugin.name`。
2. 一个 module 可以定义多个插件，每一个插件都需要在 gradle-plugins 文件夹下注册一个 xxx.properties 文件。

#### 1.2.3 编辑 .properties 文件
注册文件的内容只有一行，用于关联该插件的具体实现类：
```
implementation-class=your.plugin.implement.Class
```
你需要把 `your.plugin.implement.Class` 替换为你自己的实现类全类名。

---

## 2. 实现插件
下文中的 `Plugin` 类指的是 org.gradle.api.Plugin 类，
`Transform` 类指的是 com.android.build.api.transform.Transform 类。

**大致步骤：**
* 实现一个 `Plugin` 的子类 和 一个 `Transform` 的子类。
* 在子类的 apply 方法中，注册一个 `Transform` 实例。

我们在用到这个插件的 Module 的 build.gradle 中，添加 `apply plugin: '插件名称'` 实际上就是在调用 Plugin 实例的 apply 方法。

**Transform 是什么？**
Tansform是一个抽象类。每个 Transform 对象都是在打包过程中，从 .class 文件 生成 .dex 文件 期间，要执行的操作。
我们可以用 Transform 处理注解信息，修改已存在的类和方法等。


### 2.1 实现 Plugin 子类
这一步简单，继承 Plugin 实现一个自定义的插件类，然后在 apply 方法中注册一个 Transformer 即可。
**注意**：该类的全类名需要和在xxx.properties文件中注册的全类名一样。

```Kotlin
class CustomPlugin : Plugin<Project> {

    override fun apply(project: Project) {
        val hasAppPlugin = project.plugins.hasPlugin(AppPlugin::class.java)
        if (!hasAppPlugin) {
            return
        }
        val appExtension = project.extensions.findByType(AppExtension::class.java) ?: return
        appExtension.registerTransform(CustomTransform(project, appExtension))
    }

}
```

`CustomPlugin` 可以替换为其他名称，只要和xxx.properties中注册的一致就行。
`CustomTransform` 也可以替换为你自己需要的名称。


### 2.2 实现 Transform 子类

#### 2.2.1 Tansform 的重要方法

**transform**:
```Kotlin
/**
 * Tansform 的实现类最重要的方法，用于做具体的数据转换。
 * 可以通过参数 transformInvocation 得到所有的 .class 等输入。  
 */
fun transform(transformInvocation: TransformInvocation?)
```

**getInputTypes**:
```Kotlin
/**
 * 返回当前 Transform 需要的输入的类型。  
 * ContentType 常用的类型有：
 * CLASSES:  编译好的.class文件
 * RESOURCES:  原始的Java文件
 * NATIVE_LIBS:  C/C++库
 */
fun getInputTypes():  MutableSet\<ContentType\>
```

**getScopes**:
```Kotlin
/**
 * 返回当前 Transform 应用的范围。 
 * Scope 常用的类型 PROJECT、SUB_PROJECT、EXTERNAL_LIBRARIES 等。
 * 通常返回常量集合 SCOPE_FULL_PROJECT 即可。
 */
fun getScopes(): MutableSet\<Scope\>
```

**getName**:
```Kotlin
/**
 * 返回当前 Transform 唯一的名称。 
 */
fun getName(): String**
```

**isIncremental**
```Kotlin
/**
 * 当前 Transform 是否支持增量编译。
 */
fun isIncremental(): Boolean
```

#### 2.2.2 实现 transform 方法
假设我们有这么一个需求：修改添加了 @DemoAnnotation 注解的方法，使得该方法在执行原始代码块之前和之后都打印一句话。例如将方法:
``` Java
void function() {
    System.out.println("这是方法的原始内容")
}
```
修改为：
```Java
void function() {
    { Log.i("自定义插件", "function方法开始执行了！") }
    System.out.println("这是方法的原始内容")
    { Log.i("自定义插件", "function方法执行完毕了！") }
}
```


transform 方法要做的步骤：
* 获取所有的输入 => inputs;
* 创建 ClassPool 对象 => classPool;
* 把系统类路径、inputs 包含的路径加入到 classPool 备用；
* 遍历 inputs:
	* 获取每一个 input 的文件夹：
		+ 递归遍历文件夹，处理每一个类
		+ 获取当前文件夹的输出路径
		+ 将 input 文件夹复制到 输出文件夹
	*  获取每一个 input 的 Jar：
		+ 同文件夹的处理方式


**具体实现：**
```Kotlin
private var mProject: Project? = null
private var mAppExtension: AppExtension? = null
private val mClassPool = ClassPool()

constructor(project: Project, appExtension: AppExtension) {
    mProject = project
    mAppExtension = appExtension
}

override fun transform(transformInvocation: TransformInvocation?) {
    super.transform(transformInvocation)

    // 获取所有输入
    val inputs : Collection<TransformInput> = transformInvocation?.inputs ?: return

    // 获取 OutputProvider
    val outputProvider = transformInvocation.outputProvider

    // 将系统的类加入到搜索路径中
    mClassPool.appendSystemPath()
    val bootClasses = mAppExtension?.bootClasspath
    bootClasses?.forEach { file ->
        mClassPool.appendClassPath(file.absolutePath)
    }

    // 把所有需要打包到 apk 中的类都加入到搜索路径中
    inputs.forEach { input ->
        input.jarInputs.forEach { mClassPool.appendClassPath(it.file.absolutePath) }
        input.directoryInputs.forEach { mClassPool.appendClassPath(it.file.absolutePath) }
    }

    // 遍历每一个输入
    inputs.forEach { input ->

        // 遍历每一个文件夹
        input.directoryInputs.forEach { directory ->

            // 遍历当前文件夹下的所有类，并逐一处理
            FileScanner.scan(directory.file) { file -> handleClass(directory.file, file) }

            // 获取输出路径
            val output = outputProvider.getContentLocation(
                directory.name,
                directory.contentTypes,
                directory.scopes,
                Format.DIRECTORY
            )
			
			// 将修改后的文件夹复制到输出路径
            FileUtils.copyDirectory(directory.file, output)
        }

        // 遍历每一个 Jar
        input.jarInputs.forEach { jar ->

            // 获取输出路径
            val output = outputProvider.getContentLocation(
                jar.file.absolutePath,
                jar.contentTypes,
                jar.scopes,
                Format.JAR
            )
			
			// 虽然不处理 Jar 中的类，但也需要复制到输出目录
            FileUtils.copyFile(jar.file, output)
        }
    }
}
```

#### 2.2.3 实现 handleClass 方法
上面的 transform 方法是通用的流程，我们再来看怎么具体处理每一个文件。
因为输入的都是 .class 文件，所以每个文件就有且只有一个类。

**主要步骤：**
* 根据文件读取到 Class 信息
* 获取所有定义的方法
* 遍历所有方法，并判断该方法是否需要修改
* 如果需要修改，修改该方法
* 将修改后的类写入文件

**具体实现：**
```Kotlin
private fun handleClass(directory: File, file: File) {

    if (!file.path.endsWith(".class")) {
        return
    }

    if (file.path.endsWith("/R.class")) {
        return
    }

	// 获取文件流
    val inputStream = file.inputStream()

	// 读取类信息
    val reader = ClassReader(inputStream)
    val className = reader.className.replace('/', '.')
    val tempCls: CtClass = mClassPool.get(className)

	// 可以通过下面这些方法获取这个类的更多信息：
    // val annotations = tempCls.annotations
    // val methods = tempCls.methods;
    // val nestedClasses = tempCls.nestedClasses
    // val constructors = tempCls.constructors
    // val declaredClasses = tempCls.declaredClasses
    // val declaringClass = tempCls.declaringClass

    var hasModified = false

    // 遍历这个类定义的所有方法
    tempCls.declaredMethods.forEach { method ->

		// 根据自己的需求，判断是否需要修改这个方法
        val needModify = tempCls.hasAnnotation("your.annotation.ClassName")
        if (needModify) {
		    // 修改这个方法
            modifyMethod(method, tempCls)
            hasModified = true
        }
    }

    inputStream.close()

	// 如果这个类有修改，把新的类写入文件
    if (hasModified) {
        tempCls.writeFile(directory.absolutePath)
    }
    return false
}
```

#### 2.2.4 实现 modifyMethod 方法
到这一步，我们可以根据自己的需求，对这个方法进行修改了。
作为示例，我们为这个方法的执行前后都加上一句日志：

```Kotlin
private fun modifyMethod(method: CtMethod, clazz: CtClass) {
    try {
        if (clazz.isFrozen) {
            clazz.defrost()
        }

        // 拼接代码
        method.insertBefore("android.util.Log.i(\"TAG\", \"${clazz.name} 类的 ${method.name} 方法开始执行\");")
        method.insertAfter ("android.util.Log.i(\"TAG\", \"${clazz.name} 类的 ${method.name} 方法执行结束\");")

    } catch (e: Exception) {
        e.printStackTrace()
    }
}
```
`insertAfter` 方法会自动在所有 return 的地方都加上代码，开发者不用考虑提前return的问题。

结合上一篇 [使用 APT 生成代码](https://www.jianshu.com/p/fb80995daa45) 的方法，可以在 modifyMethod 中插入自动生成的代码实现更强的功能。


