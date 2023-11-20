你是否有遇到过这样的烦恼：

- 项目中有很多需要在应用启动阶段进行初始化的操作
- 初始化逻辑存在先后依赖关系，A 初始化逻辑必须等待 B 初始化逻辑执行完才能执行
- 有些初始化逻辑仅在指定进程才需要执行
- 当项目膨胀起来，初始化逻辑开始混乱难以维护，启动性能问题也开始暴露

在项目初期，刚开始大家都各自写各自的逻辑，随着业务迭代，越来越多需要启动阶段进行初始化的逻辑引入，新加入的需要初始化的逻辑根本不知道该往哪里写了，而且稍不注意就会把整个启动阶段初始化的逻辑改坏，引发惊天大 bug。相信很多同学都会遇到这样的痛点，这个时候你一定需要这款产品：Startup —— 一站式解决以上所有问题的应用启动初始化器！

什么？你以为我要介绍的是 Android Jetpack 官方推出的 [App Startup](https://developer.android.com/topic/libraries/app-startup) 吗？当然不是，今天要介绍的是 [App Startup Pro max](https://github.com/GeeJoe/Startup.git) !

有点开玩笑了，写这个库之前，笔者确实先是考虑的直接使用官方的 App Startup，但是在使用过程遇到了一些 App Startup 并不能支持的特性，比如

- 只支持在一个进程初始化，因为内部是通过 `ContentProvider` 来提供初始化时机，需要我们在 `Manifest` 中注册它提供的 `androidx.startup.InitializationProvider`，并指定一个进程，通常是主进程；虽然在 1.1.0 版本更新之后，把  `androidx.startup.InitializationProvider` 改成了非 final，允许派生子类，然后注册多个 Provider 指定不同进程，但仍然不够灵活。
- 没有提供线程级别控制，无法指定某些初始化在指定线程进行
- 循环依赖的检测在运行时，效率不够高
- 自动注入需要依赖修改 Manifest，不够直观和方便

因此，笔者考虑在 App Startup 的基础上实现一个 Pro max 版本的初始化库，解决以上所有的问题！Github 链接：https://github.com/GeeJoe/Startup.git

> 实际上最后的实现版本，对官方 App Startup 基本没有太多参考了，除了名字以外😏

# 一秒上手使用

1. 应用启动时，初始化库

   ```kotlin
     class MyApplication : Application() {
   
         override fun attachBaseContext(base: Context) {
             super.attachBaseContext(base)
             Startup.init(base)
         }
     }
   ```

2. 定义组件初始化逻辑

   ```kotlin
   @Initializer
   class LogInitializer : Initializer {
       override fun init(context: Context, processName: String) {
           // ...
       }
   }
   ```

# 注解灵活配置

```kotlin
// 配置依赖项、运行进程、线程
@Initializer(
    dependencies = [BInitializer::class, CInitializer::class],
    threadMode = ComponentInfo.ThreadMode.WorkThread,
    supportProcess = ["sub", "main"]
)
class LogInitializer : Initializer {
    override fun init(context: Context, processName: String) {
        // ...
    }
}

```

- 一些优先级不那么高的耗时初始化逻辑，可以放在子线程中执行来提高启动速度
- 配置在子线程执行的初始化器会在启动时并行执行
- 配置在主线程的初始化器会在启动时串行执行
- 有依赖关系的初始化器会保证执行的先后顺序

# 编译时检测

如果有循环依赖、不合理线程、进程配置关系，会在编译时给出错误信息，快速定位问题

```kotlin
// 自己依赖自己，会在编译时报错 CycleDependencyException
@Config(dependencies = [CycleInit::class])
class CycleInit : Initializer {
    override fun init(context: Context, processName: String) {
        // no-op
    }
}
```

![image-20231117152006673](Startup -- 一站式解决应用启动初始化问题.assets/image-20231117152006673.png)

除了循环依赖检测，还支持以下编译时检测：

1. `IllegalAnnotationException` 注解使用错误时抛出异常。只能注解到实现了 `Initializer` 接口的类上
2. `IllegalProcessException` 当定义的进程信息不合法的时候抛出异常。比如 A 初始化器依赖于 B 初始化器；B 初始化器的 supportProcess 集合必须大于等于 A 初始化器
3. `IllegalThreadException` 当定义的线程信息不合法的时候抛出异常。比如 A 在主线程初始化，B 在子线程初始化， A 不能依赖 B

总而言之一句话：**只要编译能通过，就表示所有初始化配置没问题，让使用者专注于各组件自身的初始化逻辑，再也不用担心配置出错导致运行时异常！**

# 原理解析

这个库内部是怎么实现的呢，下面跟随笔者从 0 开始撸一遍！

## 第一步：定义接口

首先当然是定义初始化器的接口，我们主打使用方便，接口怎么简洁怎么来

```kotlin
// 不同的初始化器需要实现这个接口，在 init 方法中定义自己的初始化逻辑
interface Initializer {
    fun init(context: Context)
}

// 提供给接入方使用的入口类，需要在 Application 启动的时候调用一下 init
class Startup {
  	fun init(context: Context) { 
    		// 这里最终会遍历所有初始化器，并调用其 init 方法
    }
}
```

## 第二步：如何实现自动注册初始化器

我们最终希望接入者只需要首次接入的时候，在 `Application` 启动时调用一下 `Startup.init()`。后续使用者定义好初始化器之后就可以直接使用了。因此我们需要考虑如何让新增的初始化器可以自动注册到库中，以便 `Startup.init()` 被调用的时候能被找到并执行 `init`。这里我们采用注解处理器来实现，使用注解有几个好处：

- 可以方便的在注解中配置线程、进程、依赖项
- 注解处理器可以根据注解信息在编译时生成代码，这样就可以实现使用了注解的类生成注册的代码

> 关于注解处理器的使用，感兴趣可以看我的另外一篇文章 [《使用注解自动生成代码》](https://juejin.cn/post/6955740450061811720)

### 定义注解

```kotlin
/**
 *
 * 用于标注一个 Initializer, Startup 库会自动将此 Initializer 注册到 [IInitializerRegistry] 中
 *
 * @param dependencies 依赖列表
 * @param threadMode 在指定线程下初始化, 默认主线程
 * @param supportProcess 支持在哪些进程初始化，如果不指定进程，默认只在主进程初始化, "*" 表示所有进程
 */
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
@MustBeDocumented
annotation class Config(
    val dependencies: Array<KClass<*>> = [],
    val threadMode: ComponentInfo.ThreadMode = ComponentInfo.ThreadMode.MainThread,
    val supportProcess: Array<String> = []
)

```

注解中定义了三个参数，分别用于配置依赖项、线程和进程

### 注解处理器

接下来我们就得实现注解处理器，在编译时，扫描所有使用了这个注解的类，并解析其中的参数，最终我们希望得到一个初始化器列表，以便后续进行进一步处理。

首先我们先抽象一个数据类，用于表示解析出来的初始化器相关的信息，我们叫 `ComponentInfo`

```kotlin
/**
 * @param name 初始化组件名字
 * @param supportProcess 支持在指定进程列表初始化
 * @param threadMode  在指定线程下初始化
 * @param dependencies 依赖列表
 * @param instanceProvider 获取实例的 lambda
 */
data class ComponentInfo(
    val name: String,
    val supportProcess: List<String> = emptyList(),
    val threadMode: ThreadMode = MainThread,
    val dependencies: List<String> = emptyList(),
    val instanceProvider: () -> Any
)
```

接下来实现注解处理器，解析所有使用了注解的类：

```kotlin
@AutoService(Processor::class)
@SupportedSourceVersion(SourceVersion.RELEASE_8)
class InitializerProcessor : AbstractProcessor() {

    companion object {
        private const val TAG = "InitializerProcessor"
    }

    override fun getSupportedAnnotationTypes(): MutableSet<String> {
        return mutableSetOf(
            Config::class.java.canonicalName
        )
    }

    override fun process(
        set: MutableSet<out TypeElement>,
        roundEnvironment: RoundEnvironment
    ): Boolean {
        ...
      	val componentList = buildComponentInfoList(roundEnvironment)
      	...
    }

    private fun buildComponentInfoList(roundEnvironment: RoundEnvironment): List<ComponentInfo> {
        return roundEnvironment.getElementsAnnotatedWith(Config::class.java)
            .filter {
              	// 如果使用者在没有实现 Initializer 接口的类上使用注解，会在编译时报错
                if (it !is TypeElement) {
                    throw IllegalAnnotationException("${Config::class.java.simpleName} 只能注解到实现了 Initializer 接口的类上 --> ${it.simpleName}")
                }
                true
            }
            .map {
                val annotation = it.getAnnotation(Config::class.java)
                val componentName = (it as TypeElement).qualifiedName.toString()
                val supportProcess = annotation.supportProcess
                val threadMode = annotation.threadMode
                val dependencies = annotation.getClazzArrayNameList()
                ComponentInfo(
                    name = componentName,
                    supportProcess = supportProcess.toList(),
                    threadMode = threadMode,
                    dependencies = dependencies.toList()
                ) { }
            }
    }

    private fun Config.getClazzArrayNameList(): List<String> {
        try {
            return this.dependencies.map { it.java.canonicalName }
        } catch (mte: MirroredTypesException) {
            if (mte.typeMirrors.isEmpty()) return emptyList()
            return mte.typeMirrors.map {
                ((it as DeclaredType).asElement() as TypeElement).qualifiedName.toString()
            }
        }
    }

}

```

## 第三步：如何构建依赖链

解析出所有的注解类之后，我们得到了一个 `ComponentInfo` 列表，接下来我们需要把这个列表分成两类，一类是主线程执行的，一类是子线程执行的，然后分别构建他们的依赖链

#### 分类

```kotlin
/**
 * 获取所有在主线程执行的初始化器
 */
private fun List<ComponentInfo>.getMainThreadComponentList(): List<ComponentInfo> {
    return this.filter { it.threadMode == ComponentInfo.ThreadMode.MainThread }
}

/**
 * 获取所有在子线程执行的初始化器
 */
private fun List<ComponentInfo>.getWorkThreadComponentList(): List<ComponentInfo> {
    return this.filter { it.threadMode == ComponentInfo.ThreadMode.WorkThread }
}

```

#### 构建依赖链

依赖链的构建是这个库最核心也是最难的地方，由于依赖关系是使用者自由定义的，很有可能会造成循环依赖；并且由于支持定义线程和进程，依赖关系又更复杂一些，比如：

- A 初始化器依赖于 B 初始化器；B 初始化器的 supportProcess 集合必须大于等于 A 初始化器
- A 在主线程初始化，B 在子线程初始化， A 不能依赖 B

先抛开线程和进程这些额外的限制，我们考虑一个最原始的依赖链的构建，这其实是一个 *单向无环图* 的构建过程，官方的 App Startup 源码中也有这部分，还有 `CoordinatorLayout` 中构建所有子 View 之间的依赖关系也是用到了类似的构建过程：[详戳](https://juejin.cn/post/7297838901090500662)

首先定义一个依赖链的数据类，用于表示一条依赖链：`DependencyChain`

```kotlin
/**
 * 初始化器依赖链, 单向无循环依赖，用一个有序的 LinkedHashSet 表示
 */
data class DependencyChain(val chain: LinkedHashSet<ComponentInfo>)
```

然后我们提供一个构建依赖链的 Builder 类，专门用于依赖链的构建：`DependencyChainBuilder`

```kotlin
class DependencyChainBuilder(
    private val logger: ILogger,
    private val allComponentList: Map<String, ComponentInfo>
) {

    private val building = CopyOnWriteArraySet<ComponentInfo>()
    private val built = CopyOnWriteArraySet<ComponentInfo>()

    /**
     * 构建初始化器依赖链列表
     *
     * 比如 A、B、C、D、E、F、G 可能构建出以下依赖链列表
     *
     * A -> B -> C
     * D -> E
     * F
     * G
     *
     * @param components 待构造依赖链列表的 Component 列表
     */
    fun buildComponentChainList(components: List<ComponentInfo>): List<DependencyChain> {
        val result = mutableListOf<DependencyChain>()
        components.forEach {
            val dependencyChain = DependencyChain(linkedSetOf())
            dfsBuildDirectedAcyclicGraph(
                it,
                null,
                building = building,
                built = built,
                dependencyChain = dependencyChain
            )
            if (dependencyChain.chain.isNotEmpty()) {
                result.add(dependencyChain)
            }
        }
        logger.i(
            String.format(
                "buildComponentChainList(${Thread.currentThread()}) ->\n- input=%s\n- output=%s",
                components.toSimpleString(),
                result.toTypedArray().contentToString()
            )
        )
        return result
    }

    /**
     * dfs 构建依赖链（有向无环图）
     * @param component 当前 Component 节点
     * @param subComponent 当前 Component 节点的依赖节点
     * @param building 全局记录正在构建依赖链的 Component 列表
     * @param built 全局记录已经确定链位置的 Component 列表
     * @param dependencyChain 记录本次确定链位置的 Component 列表, 是本次依赖链产物
     */
    private fun dfsBuildDirectedAcyclicGraph(
        component: ComponentInfo,
        subComponent: ComponentInfo?,
        building: MutableSet<ComponentInfo>,
        built: MutableSet<ComponentInfo>,
        dependencyChain: DependencyChain
    ) {
        // 断言非循环依赖
        assertNotCycleDependency(component, building)
        // 已经构建好了就跳过
        if (built.contains { it.name == component.name }) {
            return
        }
        building.add(component)
        // 先初始化依赖的 Component
        chainDependencyComponent(component, building, built, dependencyChain)
        // 断言非异常进程定义
        assertLegalProcess(component.name, subComponent?.name)
        // 断言非异常线程定义
        assertLegalThread(component.name, subComponent?.name)
        // 构建
        building.remove(component)
        built.add(component)
        dependencyChain.chain.add(component)
    }

    /**
     * 断言非循环依赖
     *
     * @param componentInfo 当前 Component 节点
     * @param building 全局记录正在构建依赖链的 Component 列表
     */
    private fun assertNotCycleDependency(
        componentInfo: ComponentInfo,
        building: MutableSet<ComponentInfo>
    ) {
        if (building.contains { it.name == componentInfo.name }) {
            val message = String.format(
                "Cannot initialize %s. Cycle detected.", componentInfo.name
            )
            throw CycleDependencyException(message)
        }
    }

    /**
     * 断言合法定义的进程信息：
     * A 初始化器依赖于 B 初始化器
     * A: 子初始化器
     * B: 父初始化器
     * 父初始化器的 supportProcess 必须大于等于子初始化器.
     *
     * @param componentName 父初始化器
     * @param subComponentName 子初始化器
     */
    private fun assertLegalProcess(
        componentName: String,
        subComponentName: String?
    ) {
        if ((subComponentName != null) &&
            !getSupportProcessList(componentName)
                .contains(getSupportProcessList(subComponentName))
        ) {
            val message = String.format(
                "父 Initializer(%s) 的 supportProcess 必须大于等于子 Initializer(%s).",
                componentName,
                subComponentName
            )
            throw IllegalProcessException(message)
        }
    }

    /**
     * 断言合法定义的线程信息：
     * A: 在主线程初始化
     * B: 在子线程初始化
     * A 初始化器不能依赖于 B 初始化器
     *
     * @param componentName 父初始化器
     * @param subComponentName 子初始化器
     */
    private fun assertLegalThread(
        componentName: String,
        subComponentName: String?
    ) {
        if (subComponentName.isNullOrEmpty()) {
            return
        }
        val component = allComponentList[componentName]
        val subComponent = allComponentList[subComponentName]
        if (component?.threadMode == ComponentInfo.ThreadMode.WorkThread &&
            subComponent?.threadMode == ComponentInfo.ThreadMode.MainThread
        ) {
            val message = String.format(
                "运行在主线程的 Initializer(%s) 不能依赖于运行在子线程的 Initializer(%s).",
                subComponent,
                component
            )
            throw IllegalThreadException(message)
        }
    }

    /**
     * 链接当前结点的所有依赖节点
     *
     * @param componentInfo 当前 Component 节点
     * @param building 全局记录正在构建依赖链的 Component 列表
     * @param built 全局记录已经确定链位置的 Component 列表
     * @param dependencyChain 记录本次确定链位置的 Component 列表, 是本次依赖链产物
     */
    private fun chainDependencyComponent(
        componentInfo: ComponentInfo,
        building: MutableSet<ComponentInfo>,
        built: MutableSet<ComponentInfo>,
        dependencyChain: DependencyChain
    ) {
        val dependencies = componentInfo.dependencies
        if (dependencies.isEmpty()) {
            return
        }
        for (clazzName in dependencies) {
            val dependencyComponentInfo = allComponentList[clazzName]
            if (dependencyComponentInfo == null || built.contains { it.name == clazzName }) {
                continue
            }
            dfsBuildDirectedAcyclicGraph(
                dependencyComponentInfo,
                componentInfo,
                building,
                built,
                dependencyChain
            )
        }
    }

    /**
     * 判断当前列表包含的内容是否大于等于 targetList
     * 如果当前列表包含 [ALL_PROCESS] 代表肯定包含 targetList
     */
    private fun List<String>.contains(targetList: List<String>): Boolean {
        if (this.contains(ALL_PROCESS)) return true
        targetList.forEach {
            if (!this.contains(it)) {
                return false
            }
        }
        return true
    }

    /**
     * 正常情况下这里不会为空，除非注解中真的没有定义 supportProcess, 那默认就在主进程初始化
     *
     * @param componentName 当前初始化器名字
     */
    private fun getSupportProcessList(componentName: String): List<String> {
        val processList =
            allComponentList[componentName]?.supportProcess ?: emptyList()
        if (processList.isEmpty()) {
            return arrayListOf(MAIN_PROCESS)
        }
        return processList
    }
}
```

## 第四步：生成代码

通过前面的步骤，我们得到了一系列的依赖链，接下来我们需要把这些信息 “注册” 到一个类地方，以便使用者接入时调用 `Startup.init()` 能够获取到这些初始化器，并调用其 `init()` 方法。

根据我们的需求，我们需要一个能够获取所有依赖链的注册表类 `IInitializerRegistry`，入口类 `Startup` 能够拿到这个 `IInitializerRegistry`，以便拿到初始化器，按照依赖链的顺序依次执行 `init` 方法

接口定义如下：

```kotlin
interface IInitializerRegistry {
  
    companion object {
        const val GENERATED_CLASS_PACKAGE_NAME = "com.geelee.startup"
        const val GENERATED_CLASS_NAME = "InitializerRegistry"
    }

    fun getMainThreadComponentChainList(): List<DependencyChain>

    fun getWorkThreadComponentChainList(): List<DependencyChain>
}
```

有了这个注册表，我们就可以在 `Statup` 中拿到所有的初始化器进行遍历了

```kotlin
class Startup private constructor(
    private val appContext: Context,
    private val registry: IInitializerRegistry,
    private val logger: IStartupLogger
) {
  
    fun init() {
        registry.getMainThreadComponentChainList().forEach {
          	// ...
        }
      	registry.getWorkThreadComponentChainList().forEach {
          	//...
        }
    }

    
}

```

`IInitializerRegistry` 是一个抽象接口，他的实现类是 `InitializerRegistry`，是注解处理器生成的代码，生成代码的逻辑比较简单，这里使用了 `KotlinPoet`，感兴趣的同学可以直接看下[源码](https://github.com/GeeJoe/Startup)。

由于生成的类名和路径都是我们可以定义的，因此在 `Startup` 中我们可以通过反射拿到这个接口的实现类

```kotlin

@JvmStatic
fun build(
    context: Context,
    registry: IInitializerRegistry = Class.forName("${GENERATED_CLASS_PACKAGE_NAME}.${GENERATED_CLASS_NAME}")
        .newInstance() as IInitializerRegistry,
    logger: IStartupLogger = DefaultLogger()
): Startup {
    val appContext = context.applicationContext ?: context
    return Startup(appContext, registry, logger)
}

@JvmStatic
fun init(context: Context) {
    build(context).init()
}

```

## 第五步：启动初始化流程

万事具备，最后我们需要启动这些链条，前期我们将初始化器分成了主线程和子线程执行，其中我们希望

- 主线程各个依赖链之间串行执行
- 子线程各个依赖链之间并行执行
- 每一条依赖链中的初始化器按依赖顺序先后执行

可以看到整个库我们都是纯 Kotlin 开发，这里我们当然要使用 Kotlin 协程来实现了：

```kotlin
class Startup private constructor(
    private val appContext: Context,
    private val registry: IInitializerRegistry,
    private val logger: IStartupLogger
) {

    private val currentProcess = appContext.processName()

    fun init() {
        initInMainThread(currentProcess)
        MainScope().launch {
            initInWorkThreadAsync(currentProcess)
        }
    }

    /**
     * 子线程的依赖链之间并行执行
     */
    private suspend fun initInWorkThreadAsync(currentProcess: String) =
        withContext(Dispatchers.Default) {
            val taskInParallel = mutableListOf<Deferred<Unit>>()
            registry.getWorkThreadComponentChainList().forEach {
                taskInParallel.add(async { it.initOneByOneAsync(currentProcess) })
            }
            taskInParallel.forEach { it.await() }
        }

    /**
     * 主线程的依赖链之间串行执行
     */
    fun initInMainThread(currentProcess: String) {
        registry.getMainThreadComponentChainList().forEach {
            it.initOneByOne(currentProcess)
        }
    }

    /**
     * 依次执行链表中每个初始化器的初始化逻辑
     */
    private fun DependencyChain.initOneByOne(currentProcess: String) {
        this.chain.forEach { it.init(currentProcess) }
    }

    /**
     * 依次执行链表中每个初始化器的初始化逻辑(异步方法)
     */
    private suspend fun DependencyChain.initOneByOneAsync(currentProcess: String) {
        val dependencyChain = this
        withContext(Dispatchers.Default) {
            dependencyChain.initOneByOne(currentProcess)
        }
    }

    /**
     * 执行单个初始化器的初始化操作
     */
    private fun ComponentInfo.init(currentProcess: String) {
        if (!isSupportProcess(appContext, currentProcess)) {
            logger.i(
                "Skip ${this.name} cause it suppose to init at process${
                    this.supportProcess.toTypedArray().contentToString()
                } but the current process is $currentProcess"
            )
            return
        }
        try {
            val initializer = this.instance as Initializer
            logger.i(String.format("Initializing %s at %s", this.name, Thread.currentThread()))
            initializer.init(appContext, currentProcess)
            logger.i(String.format("Initialized %s at %s", this.name, Thread.currentThread()))
        } catch (e: Throwable) {
            if (logger.isDebugVersion()) {
                throw e
            } else {
                logger.e("init failed: ${e.message}", e)
            }
        }
    }
}

```

