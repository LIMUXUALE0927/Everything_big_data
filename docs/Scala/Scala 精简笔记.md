## Scala Intro

### Scala VS Java

Scala 是一门多范式的编程语言，设计初衷是要集成面向对象编程和函数式编程的各种特性。

特点：

- 同样运行在 JVM 上，可以与现存程序同时运行。
- 可直接使用 Java 类库。
- 同 Java 一样静态类型。
- 语法和 Java 类似，比 Java 更加简洁（简洁而并不是简单），表达性更强。
- 同时支持面向对象、函数式编程。
- 比 Java 更面向对象。

关注点：

- 类型推断、不变量、函数式编程、高级程序构造。
- 并发：actor 模型。
- 和现有 Java 代码交互、相比 Java 异同和优缺。

和 Java 关系：

```
        javac               java
.java --------> .class ----------> run on JVM
.scala -------> .class ----------> run on JVM
        scalac              scala
```

## Hello World

```scala
object Main {
  def main(args: Array[String]): Unit = {
    println("Hello world!")
  }
}
```

---

## Scala 变量、类型

### Scala 中常见的类型

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202311031509486.png)

---

## Scala 流程控制

### For 循环

```scala
val str = "abcdefg"
// <-的本质：foreach
for (s <- str) {
  println(s)
}
// 本质：1.to(10).foreach(i => println(i))
// 通常的写法：调用者.方法(参数)
// Scala的特殊写法：调用者 方法 参数
for (i <- 1 to 10) {
  println(i)
  // Console println i
}
// until: 左闭右开区间
for (i <- 1 until 10) {
  println(i)
}
// 本质：1.to(10).by(2).foreach(i => println(i))
for (i <- 1 to 10 by 2) {
  println(i)
}
// 逆序
for (i <- 10 to 1 by -1) {
  println(i)
}
for (i <- 1 to 10 reverse) {
  println(i)
}
```

### For 循环引入变量

```scala
for (i <- 1 to 5; j = i * i) {
  println(j)
}
// 等价于
for {
  i <- 1 to 5
  j = i * i
} {
  println(j)
}
```

### For 循环推导式

```scala
val res = for (i <- 1 to 5) yield i * i

println(res) // Vector(1, 4, 9, 16, 25)

val res = for (i <- 1 to 5) yield {
  if (i % 2 == 0) {
    i
  } else {
    "odd"
  }

}

println(res) // Vector(odd, 2, odd, 4, odd)
```

### 循环守卫替代 continue

```scala
// 循环守卫：for + if，用于替代 continue
for (i <- 1 to 10 if i % 2 == 0) {
  println(i)
}
// 用于替代:
for (i <- 1 to 10) {
  if (i % 2 == 0) {
    println(i)
  }
}
```

### 嵌套循环

```scala
// 嵌套循环：用于替代多重for循环
for (i <- 1 to 3; j <- 1 to 3) {
  println(i, j)
}
// 打印九九乘法表
for (i <- 1 to 9; j <- 1 to i) {
  print(s"$j * $i = ${i * j}\t")
  if (i == j) {
    println()
  }
}
```

### 抛异常实现 break

```scala
import scala.util.control.Breaks

// 通过抛异常来实现 break
try {
  for (i <- 1 to 10) {
    println(i)
    if (i == 3) {
      throw new Exception("break")
    }
  }
} catch {
  case e: Exception => println(e.getMessage)
}

// 更优雅地实现 break，底层也是通过抛异常来实现的
Breaks.breakable {
  for (i <- 1 to 10) {
    println(i)
    if (i == 3) {
      Breaks.break()
    }
  }
}
```

---

## 函数和方法

```scala
def func(arg1: TypeOfArg1, arg2: ...): RetType = {
    ...
}
```

Scala 的访问修饰符：public, protected, private。默认为 public。

需要注意的是如果省略不写返回值类型的话，就不能写 return。如果要写 return，那么则必须声明返回值类型。

```scala
object Functions {
  def main(args: Array[String]): Unit = {
    println(add(1, 2))
    println(abs(-1))
    printAll("a", "b", "c")
    test("Tom", 20)
    test2(name = "Tom", age = 20)
  }

  def add(x: Int, y: Int): Int = {
    x + y
  }

  def abs(x: Int): Int = {
    if (x < 0) -x
    else x
  }

  // 可变参数
  def printAll(s: String*): Unit = {
    s.foreach(println)
  }

  // 默认参数，一般将默认参数放在最后
  def test(name: String, age: Int = 18): Unit = {
    println(s"$name is $age years old")
  }

  // 如果默认参数在中间，那么调用时需要指定参数名
  def test2(name: String = "default", age: Int): Unit = {
    println(s"$name is $age years old")
  }
}
```

### 函数省略特性

- return 可以省略，Scala 会使用函数体的逻辑上的最后一行代码作为返回值

```scala
def f1(s: String): String = {
  s
}
```

- 如果函数体只有一行代码，可以省略花括号

```scala
def f2(s: String): String = s
```

- 返回值类型如果能够推断出来，那么可以省略

```scala
def f3(s: String) = s
```

- 如果有 return，则不能省略返回值类型，必须指定

```scala
def f4(): String = {
  return "hello"
}
```

- **Scala 如果函数是没有返回值类型，可以省略等号，将无返回值的函数称之为过程**

```scala
def f6() {
  "hi"
}
```

- **如果函数无参，但是声明了参数列表，那么调用时，小括号可加可不加**

```scala
def f7() = "hi"
f7
```

- **如果函数没有参数列表，那么小括号可以省略，调用时小括号必须省略**

```scala
def f8 = "hi"
println(f8)
```

---

### 匿名函数

```scala
(param: Type) => {function body}
```

```scala
val f = (name: String) => println(name)
f("John")
```

---

### 函数高阶用法

Scala 中的函数类似于 Int，也是一种数据类型，只不过 Int 类型的祖先类是 AnyValue，函数的祖先类是 AnyRef。

函数类型变量声明：

```scala
var func: (String) => Int
```

- 函数可以作为值：

```scala
// define function
def foo(n: Int): Int = {
    println("call foo")
    n + 1
}
// function assign to value, also a object
val func = foo _ // represent the function foo, not function call
val func1: Int => Int = foo // specify the type of func1
println(func) // Main$$$Lambda$674/0x000000080103c588@770beef5
println(func == func1) // false, not a same object
```

- 函数可以作为参数：

```scala
// function as arguments
def dualEval(op: (Int, Int) => Int, a: Int, b: Int) = {
    op(a, b)
}
def add(a: Int, b: Int): Int = a + b
println(dualEval(add, 10, 100))
val mul:(Int, Int) => Int = _ * _
println(dualEval(mul, 10, 100))
println(dualEval((a, b) => a + b, 1000, 24))
```

- 函数可以作为返回值：

```scala
// function as return value
def outerFunc(): Int => Unit = {
    def inner(a: Int): Unit = {
        println(s"call inner with argument ${a}")
    }
    inner // return a function
}
println(outerFunc()(10)) // inner return ()
```

:star: 偏函数：Scala 偏应用函数是一种表达式，不必提供函数需要的所有参数，只需要提供部分，或不提供所需参数

```scala
import java.util.Date

def log(date: Date, message: String) = {
  println(date + "----" + message)
}

val date = new Date

// logWithDateBound 的类型是：String => Unit 的函数
val logWithDateBound = log(date, _: String) // _: Type

logWithDateBound("message1")
```

### 柯里化

柯里化：将一个参数列表的多个参数，变成多个参数列表的过程。也就是将普通多参数函数变成高阶函数的过程。

```scala
def add(a: Int)(b: Int) = a + b
// f 是一个偏函数
var f: Int => Int = add(1)
f(2) // 3
```

### 闭包

闭包是一个函数，返回值依赖于声明在函数外部的一个或多个变量。闭包通常来讲可以简单的认为是可以访问一个函数里面局部变量的另外一个函数。

```scala
def minusXY(x: Int) = (y: Int) => x - y

val f: Int => Int = minusXY(20)
f(1) // 19
f(2) // 18
```

这里的 `y: Int => x - y` 是一个匿名函数，引用了外部变量 x，因此该函数和 x 整体形成一个闭包。

### 惰性加载

当函数返回值被声明为 `lazy` 时，函数的执行将会被推迟，知道我们首次对此取值，该函数才会被执行。这种函数成为惰性函数。

```scala
object Main {
  def main(args: Array[String]): Unit = {
    // just like pass by name
    lazy val result: Int = sum(13, 47)
    println("before lazy load")
    println(s"result = ${result}") // first call sum(13, 47)
  }

  def sum(a: Int, b: Int): Int = {
    println("call sum")
    a + b
  }
}
```

---

### 传值函数 & 传名函数

当在 Scala 中传递参数给函数时，可以选择使用 call by value 或 call by name 两种参数传递方式。

在 call by value 中，**函数参数的值在传递之前就会被计算并传递给函数**。这意味着无论函数内部是否使用该参数，它都会被计算一次。而在 call by name 中，函数参数的表达式会被传递，而不是计算后的值。在**函数内部使用参数时，才会对表达式进行求值**。

call by value 避免了参数的重复求值，效率相对较高；而 call by name 避免了在函数调用时刻的参数求值，而将求值推延至实际调用点，但有可能造成重复的表达式求值。

下面是一个示例，演示了 call by value 和 call by name 的区别：

```scala
def callByValue(x: Int): Unit = {
  println("x1 = " + x)
  println("x2 = " + x)
}

def callByName(x: => Int): Unit = {
  println("x1 = " + x)
  println("x2 = " + x)
}

def calculateValue(): Int = {
  println("Calculating value...")
  42
}

// 使用 call by value
callByValue(calculateValue())

// 使用 call by name
callByName(calculateValue())
```

在上述示例中，`callByValue` 函数使用了 call by value，而 `callByName` 函数使用了 call by name。

当调用 `callByValue(calculateValue())` 时，`calculateValue()` 表达式会被计算，并且结果 42 会被传递给 `callByValue` 函数。因此，输出会是：

```
Calculating value...
x1 = 42
x2 = 42
```

而当调用 `callByName(calculateValue())` 时，`calculateValue()` 表达式并不会立即被计算，而是作为表达式传递给 `callByName` 函数。只有在函数内部使用参数时，才会对表达式进行求值。因此，输出会是：

```
Calculating value...
x1 = 42
Calculating value...
x2 = 42
```

可以看到，在 call by name 的情况下，`calculateValue()` 表达式被求值了两次，因为它在函数内部被使用了两次。而在 call by value 的情况下，`calculateValue()` 表达式只被求值了一次，因为它在传递给函数之前就已经被计算了。
