# Kotlin 中那些好用但是容易记错的扩展函数

在开始讲之前，我们先了解一下什么是扩展函数

## 简介

Kotlin Extensions 是 Kotlin 上一个非常实用的语法糖，他允许我们在不修改源类或者继承源类的情况给类新增一些方法或者属性，在调用扩展方法或者获取扩展属性的时候，看起来就像是调用该类原有的方法或者属性一样。

## 原理

Kotlin Extensions 的原理其实很简单，实际上扩展是一个语法糖，并不神奇，我们可以通过 AS 的工具看看 Kotlin Extensions 转换成对应的 Java 代码是怎样的

`Tools` -> `Kotlin` -> `show kotlin bytecode` -> `decompile`

```kotlin 
fun String.getNotEmpty(defaultValue: String): String {
    if (this.isEmpty()) return defaultValue
    return this
}

val String.extensionLength: Int
    get() = this.length
```

```java
public final class StringKt {
   @NotNull
   public static final String getNotEmpty(@NotNull String $this$getNotEmpty, @NotNull String defaultValue) {
      Intrinsics.checkParameterIsNotNull($this$getNotEmpty, "$this$getNotEmpty");
      Intrinsics.checkParameterIsNotNull(defaultValue, "defaultValue");
      CharSequence var2 = (CharSequence)$this$getNotEmpty;
      boolean var3 = false;
      return var2.length() == 0 ? defaultValue : $this$getNotEmpty;
   }
  
  public static final int getExtensionLength(@NotNull String $this$extensionLength) {
      Intrinsics.checkParameterIsNotNull($this$extensionLength, "$this$extensionLength");
      return $this$extensionLength.length();
   }
}
  
```

可以看到，通过 Kotlin 的字节码反编译后，对应的 Java 代码其实就是我们平时用 Java 写的工具类方法

## 怎么写

### inline

在看官方的 ktx 的时候，经常会看到扩展函数用 inline 修饰，这是个什么东西呢？

`inline` 是内联的意思，在调用使用 inline 修饰的方法时，会在调用该方法的地方把方法的实现「内联」到调用处，举个例子

```kotlin
fun main() {
    println("Hello")
    inlineFuc { 
        println("beautiful")
    }
    println("world")
}

inline fun inlineFuc(block:()->Unit) {
    block.invoke()
}
```

相当于

```kotlin
fun main() {
    println("Hello")
    println("beautiful") // 被内联到方法调用处
    println("world")
}
```

## 什么时候要写

内联函数在运行的时候可以减小内存开销，但是编译之后会使代码量变多，因此使用内联函数的时候也要注意，如果被内联的函数过于复杂，可能会使代码量剧增；

那么我们什么时候要写 inline 呢，有两个参考原则

1. 考虑提高性能

   - 内联函数一般适用于简单的函数，特别是在循环调用的时候，这样可以减小方法调用带来的栈开销
   - 如果扩展方法参数是高阶函数，尽可能要用内联；高阶函数在调用的时候是会实例化成一个方法，然后再调用的（kotlin 中内置了很多高阶函数，例如一个参数的 Function0、两个参数的 Function1 等等）

2. 需要在函数中使用类型参数时

   - 可以使用具体化关键字 reified，使用 reified 关键字修饰的类型称为「具体化类型参数」普通函数不支持「具体化类型参数」，只有内联函数支持「具体化类型参数」

     🌰 🌰🌰 🌰🌰 🌰🌰 🌰🌰 🌰🌰 🌰🌰 🌰🌰 🌰🌰 🌰🌰 🌰

     ```kotlin
     inline fun <reified T> TreeNode.findParentOfType(): T? {
         var p = parent
         //T 可以被当做一个具体化的类使用
         while (p != null && p !is T) {
             p = p.parent
         }
         return p as T?
     }
     ```

## 那些好用但是容易用错的 Scope Functions

Kotlin 中内置了很多扩展函数供我们直接使用，有一类叫做 Scope Functions (作用域函数)，这些函数名字看起来比较绕，刚接触的时候常常会不清楚该怎样用，甚至觉得两个函数根本没有区别，接下来就一起来看看这些函数到底有什么区别，以及我应该怎么使用他们。

## let

```kotlin
public inline fun <T, R> T.let(block: (T) -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block(this)
}
```

我不知道 let 本意是什么，但根据官方文档解释，我认为可以理解为 let's do someting 的意思，看其实现其实很简单，`let` 函数把当前对象 `this` 作为闭包的 `it` 参数，返回值是函数里面最后一行，或者指定 return

#### 使用场景

- 一般用于在判断非空之后执行某段代码

## 扩展函数 run

```kotlin
public inline fun <T, R> T.run(block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}
```

和 let 非常像，唯一的区别是，let 闭包里面 it 代表当前对象，run 闭包里面 this 代表当前对象

#### 使用场景

- 基本不用

## apply

```kotlin
public inline fun <T> T.apply(block: T.() -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block()
    return this
}
```

从实现看，apply 和 扩展函数 run 非常像，唯一的区别是，run 返回代码块的返回值，而 apply 返回当前对象

#### 使用场景

- apply 的使用场景是比较明确的，一般用于需要对当前对象做一些配置和修改，并返回自己的时候，可以使用 apply，有点像 Java 里的 Builder 模式

## also

```kotlin
public inline fun <T> T.also(block: (T) -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block(this)
    return this
}
```

可以看到 also 和 apply 非常像，唯一的区别是闭包中 it 代表当前对象

#### 使用场景

- 顾名思义，当我们对某个对象做了一系列操作之后，还想利用这个对象「也做一下什么事情」的时候，可以使用 also，这看起来比较难懂，后面统一举些 🌰

## 顶级函数 run

```kotlin
public inline fun <R> run(block: () -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}
```

run 还有一个顶级函数实现，不属于任何对象的扩展，它的作用是可以在闭包中执行任意一段代码，然后返回代码的返回值

#### 使用场景

`run` 的使用场景比较少，不过我一般这么认为

- 当我们需要执行一段相对独立的代码，可以使用 `run` 将这段代码包裹起来，看起来就像是执行了一个匿名方法

## 顶级函数 with

```kotlin
public inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return receiver.block()
}
```

with 也是一个顶级函数，和顶级函数 run 很像，唯一的区别是，顶级函数 with 中，闭包内 this 代表传入的第一个参数

#### 使用场景

- 当想用顶级函数 run 而且又想在闭包里使用 this 的时候

## 我到底应该使用哪一个

Scope fuction 的初衷是为了让代码更具可读性，如果对这些函数有一些了解，也许会发现很多扩展函数都是可以相互代替的，这个时候如果觉得「既然都能用，那就随便挑一个来用」的话就会背离 Kotlin 设计这些函数的初衷，反而会让代码变得更加「难懂」，因此我们应该遵循官方的使用建议，正确使用 scope function，宁可不用，也不要滥用。

### 举些 🌰

```kotlin
// let 使用场景
var name: String? = null
...
name?.let {
  println(it)
}

// apply
val intent = Intent().apply {
  this.data = Uri.parse("")
  this.action = "action"
  this.flags = 0
}
startActivity(this, intent)

// also
companion object { 
        @Volatile
        private var instance: Manager? = null

        @JvmStatic
        fun get(): Manager {
            return instance ?: synchronized(Manager::class.java) {
                Manager().also {
                    instance = it
                }
            }
        }
    }

// 顶级 run
var name: String? = null
...
val length = name?.length() ?: run {
  throw IllegalArgumentException()
}
```