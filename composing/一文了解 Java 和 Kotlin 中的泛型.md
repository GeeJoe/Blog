# 一文了解 Java/Kotlin 中的泛型

阅读本文你将了解：

- 什么是形变、协变、逆变和不型变
- 在 Java 和 Kotlin 中如何实现以上形变
- Java 和 Kotlin 中泛型的异同

在 Java/Kotlin 中，子类对象是可以赋值给一个父类类型的，但是父类对象不可以赋值给子类类型，例如：

```kotlin
// Dog 是 Animal 的子类
class Animal {}
class Dog: Animal() {}

val animal: Animal = dog // 把子类对象赋值给一个父类类型是可以的，dog 也是一种 Animal
val dog: Dog = animal // 把父类对象赋值给一个子类类型是不可以的，不是所有 animal 都是 dog
```

在引入泛型之后，情况变得更复杂：类型参数为子类的泛型类型不是类型参数为父类的泛型类型的子类，听起来很绕，看代码：

```kotlin
// dogs 不是 animals 的子类
val dogs: List<Dog>
val animals: List<Animal>
// dogs 和 animals 不具备任何继承关系，因此以下代码会编译报错
val animals: List<Animal> = dogs
```

> **类型参数**： 泛型中尖括号中的参数称为类型参数，比如 `List<String>` 中的 `String` 就是类型参数，和普通参数不同，类型参数传递的是一个类型而不是对象

为了描述方便，以下把所有「类型参数为子类的泛型」简称为「子类泛型」，「类型参数为父类的泛型」简称为「父类泛型」

对于从 Java 转到 Kotlin 的开发者们来说，要了解泛型，最好先搞懂 Java 中的泛型，再来看 Kotlin 的泛型时会变得易如反掌。

## Java 中的泛型

### 泛型的形变[(variance)](https://kotlinlang.org/docs/generics.html#variance)

- 协变(Covariance)：子类泛型是父类泛型的子类型，可以把子类泛型赋值给父类泛型
- 逆变(Contravariance)：父类泛型（可以看作）是子类泛型的子类型，可以把父类泛型赋值给子类泛型
- 不型变(Invariant)：子类泛型和父类泛型没有任何继关系，也不可以相互赋值

👆🏻说的是偏概念的描述，听起来特别绕，特别反人类，用代码来说人话就是（已知 `Dog` 是 `Animal` 的子类）：

- 协变(Covariance)：`List<Dog>` 是 `List<Animal>` 的子类型，`List<Dog>` 类型的对象可以赋值给 `List<Animal>` 类型的变量
- 逆变(Contravariance)：`List<Animal>` (可以看作) 是 `List<Dog>` 的子类型，`List<Animal>` 类型的对象可以赋值给 `List<Dog>` 类型的变量
- 不型变(Invariant)：`List<Animal>` 和 `List<Dog>` 不具备任何继关系，也不可以相互赋值

协变、逆变本来是数学中的概念，在 Java/Kotlin 中主要应用在泛型中。

### 不型变

Java 中泛型是不型变的，也就是上例子上 `List<Dog>` 不是 `List<Animals>` 的子类，因此  `List<Dog>` 不可以赋值给 `List<Animals>`，**值得注意的是，Java  中数组是协变的**:

```Java
List<Dog> dogs = new ArrayList<Dog>();
List<Animal> animals = dogs; // 编译报错

Dog[] dogs = new Dog[] { new Dog() };
Animal[] animals = dogs; // 编译正常
animals[0] = new Animals(); // 运行时异常
```

因此我们**在 Java 中要优先使用泛型集合**

### 协变

不型变性是为了保证类型安全，但带来的代价就是使得程序的灵活性降低。有时候我们希望把子类泛型对象作为实参传递给一个声明为父类泛型的形参，例如：

```java
public int getAnimalsCount(List<Animal> animals) {
  return animals.size();
}

List<Dog> dogs = new ArrayList<Dog>();
int dogsCount = getAnimalsCount(dogs); // 由于 Java 泛型的不型变，这里会编译报错的
```

以上，把 `dogs` 传递给 `getAnimalsCount` 方法用于计算狗狗的数量，这是一个特别合理的需求，因为不型变性导致这类需求无法实现是 Java 所不愿看到的，因此 Java 泛型通过**通配符**引入了协变：

```java
public int getAnimalsCount(List<? extends Animal> animals) {
  return animals.size();
}
```

`? extends Animal` 表示此方法可以接受 Animal 或者 Animal 子类的集合，这就使得泛型类型协变了

### 逆变

同理，有时候我们希望把父类泛型对象作为实参传递给子类泛型的形参，例如

```java
// 用于监听小动物是否饿了的监听器，为了可以在不同动物上复用，使用了泛型接口
interface OnHungryListener<T> {
  void onHungry(T who)
}

public void observeDogsHungry(listener: OnHungryListener<Dog>) {
  ...
}

// 某天狗狗饥饿监听器坏了，我想用动物接监听器来代替
OnHungryListener<Animal> animalHungryListener = ...;
observeDogsHungry(animalHungryListener); // 编译报错
```

由于不型变性，上述代码依旧会编译报错，因此，Java 同样使用通配符参数来实现了逆变：

```java
interface OnHungryListener<? super T> {
  void onHungry(T who)
}
```

这样我们就可以往 `observeDogsHungry` 方法中传递一个 `Dog` 父类的监听器了，因此只要是狗狗父类型的监听器，都可以用来监听狗狗是否饥饿了。

### Java 泛型通配符

Java 使用通配符来表示类型参数，实现了协变和逆变，其中

- 上界通配符 `? extends T`:  限定了类型参数的**上限**，类型参数为 `T` 和所有 `T`的子类型的泛型对象，都可以赋值给 `? extend T` 的泛型类型
- 下界通配符 `? super T`:  限定了类型参数的**下限**，类型参数为 `T` 和所有 `T` 的父类型的泛型对象，都可以赋值给 `? super T` 的泛型类型
- 无限定通配符 `?`：表示无任何限制的类型参数，类型参数可以是任意类型，任何类型都是 `?` 的子类，因此类型参数是任意类型的泛型都可以赋值给 `?` 的泛型

> 无限定通配符 `?` 的使用场景相对较少，当我们发现我们无需对类型参数做任何限制的时候可以使用。例如我们实现一个返回任意类型集合大小的方法，我们就可以定义成 `public int getListSize(List<?> list)`; 
>
> 注意，我们不能定义为 `public int getListSize(List<Object> list)` 因为例如 `List<String>` 不是 `List<Object>` 的子类，但 `List<String>` 和其他任意类型的 `List` 都是 `List<?>` 的子类

### 协变和逆变的特性

- 协变：上界能确定的是父类，对于泛型集合只读不可写
- 逆变：下界能确定的是子类，对于泛型集合只写不可读

```java
// 协变，只读不可写
private void covariance(ArrayList<? extends Animal> animals) {
  Animal animal = animals.get(0); // 编译通过, 读出的是父类或者父类的子类，而 Animal 是任何类的父类
  animals.add(new Animal()); // 编译报错，写入需要父类或者父类的子类，如果写入要求子类，父类不可以赋值给子类
}
```

因为 `animals` 的类型参数可以是任意的 `Animal` 的子类，我们记 `T` 是任意的 `Animal` 的子类，`List.get()` 返回值类型是 T，因此`animals.get(0)` 返回的就是一个 `Animal` 子类型，子类可以赋值给父类

`List.add()`方法的参数类型是 T，`new Animal()` 得到的对象作为一个父类型的实参，不可以赋值给子类型的形参，因此编译失败

```java
// 逆变，只写不可读
private void contravariance(ArrayList<? super Dog> animals) {
  Dog animal = animals.get(0); // 编译报错，读出来的是子类或者子类的父类，如果是父类，父类不可以赋值给子类
  animals.add(new Dog()); // 编译通过，写入要求的是子类或者子类的父类，不过是那种，Dog 已经是所有类的下界，是任何类型的子类，子类可以赋值给父类
}
```

因为`animals` 的类型参数可以是任意 `Dog` 的父类，我们记 `T` 为任意的 `Dog` 的父类，`List.get()` 返回值类型是 `T`，因此 `animals.get(0)`返回的是一个 `Dog` 父类型对象，父类不可以赋值给子类

`List.add()`方法的参数类型是 T，`new Dog()` 得到一个 `Dog` 对象，子类可以赋值给父类

[ Effective Java, 3rd Edition](http://www.oracle.com/technetwork/java/effectivejava-136174.html) 的作者 Joshua Bloch 称那些你只能从中 **读取** 的对象为 **生产者** ，并称那些你只能 **写入** 的对象为 **消费者**。因此他提出了以下助记符：

***PECS** 代表生产者-Extends、消费者-Super（Producer-Extends, Consumer-Super）*

> 笔者认为，其实只要记住一个原则：子类可以赋值给父类，父类不可以赋值给子类，再结合泛型类型的上限和下限，自然可以推导出到底什么时候可以编译通过了

## Kotlin 中的泛型

Kotlin 的泛型可以看做是 Java 泛型的 “加强版” ，因此之前笔者也说了：了解了 Java 的泛型，再来看 Kotlin 泛型会变得易如反掌

之前提到 Java 中泛型是不型变的，而数组确实协变的，而在 Kotlin 上，泛型和数组都是不型变的，这样类型也就更加安全了，因此我说 —— Kotlin 泛型和 Java 泛型的加强版

在介绍其他 Kotlin 泛型的 “加强功能” 之前，我们先了解一下： Java 上的泛型形变，到 Kotlin 之后如何实现和表示

| 形变       | Java 中的表示 | Kotlin 中的表示 |
| ---------- | ------------- | --------------- |
| 协变       | `? extends T` | `out T`         |
| 逆变       | `? super T`   | `in T`          |
| 无限制符号 | `?`           | `*`             |

可以看到 Kotlin 中使用 `out` 和 `in` 关键字都是 **自解释** 的，`out` 只读，`in` 只写，这比 Java 容易理解多了

和 Java 类同的东西就以上这么多，下面讲一些不一样的东西

### 声明处形变（declaration-site variance）

与声明处形变对应的是**使用处形变 (use-site variance)**，先来看看这两个分别是什么意思：

- 声明处形变：在泛型类声明的时候定义形变
- 使用处形变：在使用泛型类的时候定义形变

```kotlin 
// 声明处形变: 在声明类的时候，就指定了类型参数为 out T, 此时泛型是协变的
interface SourceA<out T> {}

// 在声明的时候没有指定形变，此时该泛型类型的不型变的
interface SourceB<T> {}
// 使用处形变: 在使用 SourceB 作为参数的时候，我们指定了类型参数为 out String, 让 SourceB 发生了协变
fun useSource(source: SourceB<out String>) {}
```

> 在 Java 中只能在使用处发生形变，因此 Java 中没有声明处形变

**思考**： Kotlin 为什么要搞出声明处形变呢？我们说 Kotlin 泛型是 Java 泛型的加强版，这一定是为了解决一些 Java 所不能支持的场景

举例：一个确定只有只读能力的泛型类，使用声明处形变可以带来方便，不需要使用处每次指定

```kotlin
// 该泛型接口方法只有读的能力，没有写的能力，因此在声明处定义为 out 
interface Source<out T> {
    fun nextT(): T
}

// 在所有使用的地方就不需要每次都指定了
fun demo(strs: Source<String>) {
    val objects: Source<Any> = strs // 这个没问题，因为 T 是一个 out-参数
}
```

同理，对于逆变，一个很好的例子就是 `Comparable`：

```kotlin
// 接口方法只有写的能力，因此声明处定义 in
interface Comparable<in T> {
    operator fun compareTo(other: T): Int
}

// 使用时无须再指定逆变
fun demo(x: Comparable<Number>) {
    x.compareTo(1.0) // 1.0 拥有类型 Double，它是 Number 的子类型
    // 因此，我们可以将 x 赋给类型为 Comparable <Double> 的变量
    val y: Comparable<Double> = x // OK！
}
```

### 泛型约束

Kotlin 中还可以对泛型的类型参数做进一步限制，最常见的约束类型是与 Java 的 *extends* 关键字对应的 **上界**

```kotlin
fun <T : Comparable<T>> sort(list: List<T>) {  }
```

冒号之后指定的类型是**上界**：只有 `Comparable<T>` 的子类型可以替代 `T`。 例如：

```
sort(listOf(1, 2, 3)) // OK。Int 是 Comparable<Int> 的子类型
sort(listOf(HashMap<Int, String>())) // 错误：HashMap<Int, String> 不是 Comparable<HashMap<Int, String>> 的子类型
```

默认的上界（如果没有声明）是 `Any?`。在尖括号中只能指定一个上界。 如果同一类型参数需要多个上界，我们需要一个单独的 **where**-子句：

```
fun <T> copyWhenGreater(list: List<T>, threshold: T): List<String>
    where T : CharSequence,
          T : Comparable<T> {
    return list.filter { it > threshold }.map { it.toString() }
}
```

所传递的类型必须同时满足 `where` 子句的所有条件。在上述示例中，类型 `T` 必须*既*实现了 `CharSequence` *也*实现了 `Comparable`。