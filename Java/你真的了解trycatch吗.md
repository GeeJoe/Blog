# 你真的了解 try catch 吗



### finally 语句在 try return 语句执行之后 try return 返回之前执行

```java
public static void main(String[] args) {
        System.out.println(test2());
    }

    public static int test2() {
        int b = 20;

        try {
            System.out.println("try block");
            return b += 80;
        }
        catch (Exception e) {
            System.out.println("catch block");
        }
        finally {
            System.out.println("finally block");
            if (b > 25) {
                System.out.println("b>25, b = " + b);
            }
        }
        return 200;
    }
    
// 输出
try block
finally block
b>25, b = 100
100
```

### finally 块中的 return 语句会覆盖 try 块中的 return 返回

```java
public static int test2() {
        int b = 20;

        try {
            System.out.println("try block");
            return b += 80;
        }
        catch (Exception e) {
            System.out.println("catch block");
        }
        finally {
            System.out.println("finally block");
            if (b > 25) {
                System.out.println("b>25, b = " + b);
            }
            return 200;
        }
    }

// 输出
try block
finally block
b>25, b = 100
200
```

### 如果 finally 语句中没有 return 语句覆盖返回值，那么原来的返回值可能因为 finally 里的修改而改变也可能不变

分两种情况

1. finally 中修改的是基本数据类型，那么不会影响 try 中返回值
2. finally 中修改的是引用类型，分两种情况
   1. 修改引用类型的对象的值，会导致 try 中值改变
   2. 把一个新的类的实例赋值给改引用类型变量，不会导致 try 中值改变

```java
    public static int test2() {
        int b = 20;

        try {
            System.out.println("try block");
            return b += 80;
        }
        catch (Exception e) {
            System.out.println("catch block");
        }
        finally {
            System.out.println("finally block");
            if (b > 25) {
                System.out.println("b>25, b = " + b);
            }
            b += 150;//相当于修改方法内部参数的值，并不会影响方法外部的值
        }
        return 200;
    }
// 输出
try block
finally block
b>25, b = 100
100

public static void main(String[] args) {
        System.out.println(getDog().name);
    }

    public static Dog getDog() {
        Dog dog = new Dog();

        try {
            dog.name = "try";
            return dog;
        }
        catch (Exception e) {
            dog.name = "catch";
        }
        finally {
            dog.name = "finally";// 对对象的修改直接映射到外部的 dog 
            dog = null; // dog 指向一个新的地址，并不应影响外面 dog 的地址
        }

        return dog;
    }

    static class Dog {
        String name = "init";
    }
// 输出
finally
```

结论如上，但是原理是什么呢？
其实可以把 try catch finally 理解为三个不同的方法，共同操作的变量相当于把这个变量以参数的形式传递到这三个方法中；

1. 对于基本数据类型，传递到一个方法的形参，是把当前值的拷贝传递进来，在方法内部修改形参的值，是不会影响方法外部的值的
2. 对于引用类型，传递到一个方法的形参，是把当前引用的地址值传递进来，在方法内部修改引用地址对应的对象的成员变量，会导致原来的对象变化
3. 对于引用类型，传递到一个方法的形参，是把当前引用的地址值传递进来，在方法内部示例化一个新的对象，然后把新对象的地址复制给参数，然后对新对象的修改是不会影响到老对象的
