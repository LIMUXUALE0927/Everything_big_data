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

## Scala 函数和方法

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
def addByA(a: Int): Int => Int = a + _

val addBy2 = addByA(2)
addBy2(5) // 7

addByA(2)(5) // 7
```

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

---

### 实现自定义 while 循环

```scala
var n = 5

while (n >= 0) {
  println(n)
  n -= 1
}

def myWhile(condition: => Boolean)(body: => Unit): Unit = {
  if (condition) {
    body
    myWhile(condition)(body)
  }
}

n = 5
myWhile(n >= 0)({
  println(n)
  n -= 1
})
```

---

## Scala 面向对象

### 类

```scala
import scala.beans.BeanProperty

object Main {
  def main(args: Array[String]): Unit = {
    val student = new Student()
    student.age = 18
    student.gender = "female"
    println(student.age)
    println(student.gender)
  }
}

class Student {
  private var name: String = _ // _ 表示默认空值
  @BeanProperty
  var age: Int = _
  var gender: String = _
}
```

---

### 构造器

```scala
object Constructor {
    def main(args: Array[String]): Unit = {
        val p: Person = new Person()
        p.Person() // call main constructor

        val p1 = new Person("alice")
        val p2 = new Person("bob", 25)
        p1.Person()
    }
}
class Person {
    var name: String = _
    var age: Int = _
    println("call main construtor")

    def this(name: String) {
        this()
        println("call assist constructor 1")
        this.name = name
        println(s"Person: $name $age")
    }

    def this(name: String, age: Int) {
        this(name)
        this.age = age
        println("call assist constructor 2")
        println(s"Person: $name $age")
    }

    // just a common method, not constructor
    def Person(): Unit = {
        println("call Person.Person() method")
    }
}
```

Scala 推荐使用给主构造器传参的方式实现。

```scala
class Person(var name: String, var age: Int)
```

注意：这不等于

```scala
class Person(name: String, age: Int) {

}
```

---

### 单例对象/伴生对象

Scala 语言是完全面向对象的语言，所以并没有静态的操作（即在 Scala 中没有静态的概念）。但是为了能够和 Java 语言交互（因为 Java 中有静态概念），就产生了一种特殊的对象来模拟类对象，该对象为单例对象。若单例对象名与类名一致，则称该单例对象这个类的伴生对象，这个类的所有“静态”内容都可以放置在它的伴生对象中声明。

- 单例对象采用 object 关键字声明
- 单例对象对应的类称之为伴生类，伴生对象的名称应该和伴生类名一致。
- 单例对象中的属性和方法都可以通过伴生对象名（类名）直接调用访问。

```scala
//（1）伴生对象采用 object 关键字声明
object Person {
  var country: String = "China"
}

//（2）伴生对象对应的类称之为伴生类，伴生对象的名称应该和伴生类名一致。
class Person {
  var name: String = "bobo"
}

object Test {
  def main(args: Array[String]): Unit = {
    //（3）伴生对象中的属性和方法都可以通过伴生对象名（类名）直接调用访问
    println(Person.country)
  }
}
```

- 通过伴生对象的 apply 方法，实现不使用 new 方法创建对象
- 如果想让主构造器变成私有的，可以在()之前加上 private
- 当使用 new 关键字构建对象时，调用的其实是类的构造方法，当直接使用类名构建对象时，调用的其实时伴生对象的 apply 方法

```scala
object Test {
  def main(args: Array[String]): Unit = {
    //（1）通过伴生对象的 apply 方法，实现不使用 new 关键字创建对象。
    val p1 = Person()
    println("p1.name=" + p1.name)
    val p2 = Person("bobo")
    println("p2.name=" + p2.name)
  }
}

//（2）如果想让主构造器变成私有的，可以在()之前加上 private
class Person private(cName: String) {
  var name: String = cName
}

object Person {
  def apply(): Person = {
    println("apply 空参被调用")
    new Person("xx")
  }

  def apply(name: String): Person = {
    println("apply 有参被调用")
    new Person(name)
  }
}
```

---

### apply() 方法

在 Scala 中，`apply()` 方法是一个特殊的方法，它允许对象像函数一样被调用。当我们调用一个对象时，Scala 编译器会自动查找和调用该对象的 `apply()` 方法。

要使用 `apply()` 方法，您需要在对象的类中定义一个名为 `apply` 的方法。该方法可以具有任意数量的参数，并且可以返回任何类型的值。当您像调用函数一样调用对象时，编译器将通过调用 `apply()` 方法来执行相关的操作。

以下是一个示例，演示了如何在 Scala 中使用 `apply()` 方法：

```scala
class MyClass(val name: String) {
  def apply(): String = {
    s"Hello, $name!"
  }
}

// 创建一个 MyClass 对象
val myObj = new MyClass("Alice")

// 调用对象，实际上调用了对象的 apply() 方法
val result = myObj()

println(result) // 输出: Hello, Alice!
```

在上述示例中，我们定义了一个名为 `MyClass` 的类，该类具有一个名为 `apply` 的方法。`apply` 方法没有参数，返回一个字符串。然后，我们创建了 `MyClass` 的一个实例 `myObj`，并像调用函数一样调用了该实例。编译器自动调用了 `myObj` 的 `apply()` 方法，并返回了 `Hello, Alice!`。

`apply()` 方法的使用非常灵活，您可以根据需要在对象中定义不同的 `apply()` 方法，以便接受不同数量和类型的参数，返回不同类型的值。这使得对象在使用上更像函数，可以提供更好的语法和语义的灵活性。

当您在 Scala 中定义一个带有 `apply()` 方法的对象时，它可以以一种更简洁和直观的方式与其他代码进行交互。`apply()` 方法的使用场景包括但不限于以下几种：

- 创建对象实例：通过在伴生对象（companion object）中定义 `apply()` 方法，您可以使用类名直接创建对象实例，而无需使用 `new` 关键字。这使得代码更简洁易读。

```scala
class Person(val name: String, val age: Int)

object Person {
  def apply(name: String, age: Int): Person = new Person(name, age)
}

val person = Person("Alice", 30) // 使用类名调用 apply() 方法创建对象实例
```

- 构建工厂模式：通过在伴生对象中定义 `apply()` 方法，您可以创建一个构建对象的工厂。这样，您可以使用自定义的参数列表创建对象，并且可以在 `apply()` 方法中根据不同的参数值实现不同的逻辑。

```scala
trait Animal
class Dog(name: String) extends Animal
class Cat(name: String) extends Animal

object Animal {
  def apply(animalType: String, name: String): Animal = {
    animalType match {
      case "dog" => new Dog(name)
      case "cat" => new Cat(name)
      case _ => throw new IllegalArgumentException("Invalid animal type")
    }
  }
}

val dog = Animal("dog", "Buddy") // 使用 apply() 方法创建 Dog 对象
val cat = Animal("cat", "Whiskers") // 使用 apply() 方法创建 Cat 对象
```

- 简化集合操作：Scala 中的许多集合类（如 List、Map 等）都定义了 `apply()` 方法，使得可以通过索引或键来访问元素。这使得集合的访问和操作更加直观和便捷。

```scala
val list = List(1, 2, 3, 4, 5)
val element = list(2) // 使用 apply() 方法获取列表中的元素，结果为 3

val map = Map("a" -> 1, "b" -> 2, "c" -> 3)
val value = map("b") // 使用 apply() 方法获取映射中的值，结果为 2
```

总结来说，`apply()` 方法在 Scala 中用于提供一种更直观、简洁和灵活的方式来创建对象、实现工厂模式以及简化集合操作。通过定义和使用 `apply()` 方法，您可以以一种更函数式的风格编写代码，并提高代码的可读性和可维护性。

---

### Trait

在 Scala 中，Trait 是一种特殊的类别，它可以定义方法和字段，类似于 Java 中的接口。Trait 可以被类和对象混入（mixin），从而扩展它们的功能。Trait 可以包含抽象方法、具体方法、字段和特征（traits）。

下面是一些 Trait 的特点和使用方式：

- 定义 Trait：

```scala
trait Printable {
  def print(): Unit
}
```

- Trait 的混入（Mixin）：

```scala
class MyClass extends Printable {
  def print(): Unit = {
    println("Printing...")
  }
}

val obj = new MyClass()
obj.print() // 输出: Printing...
```

- Trait 的多重混入：

```scala
trait Drawable {
  def draw(): Unit = {
    println("Drawing...")
  }
}

class MyDrawing extends Printable with Drawable {
  def print(): Unit = {
    println("Printing...")
  }
}

val drawing = new MyDrawing()
drawing.print() // 输出: Printing...
drawing.draw() // 输出: Drawing...
```

- Trait 的抽象方法：

```scala
trait Drawable {
  def draw(): Unit
}

class Circle extends Drawable {
  def draw(): Unit = {
    println("Drawing a circle...")
  }
}

val circle = new Circle()
circle.draw() // 输出: Drawing a circle...
```

Trait 的优点包括：

- 提供了一种方式来实现代码重用和多重继承，而不会引入多继承的问题。
- Trait 可以包含具体方法的实现，从而提供默认实现，减少了重复代码。
- Trait 可以被多个类或对象混入，并且可以根据需要组合和扩展功能。
- Trait 可以作为接口的替代品，提供了更强大和灵活的特性。

需要注意的是，Trait 不能被实例化，它只能被混入到类或对象中使用。Trait 可以继承其他 Trait，形成 Trait 的层次结构，进一步扩展和组合功能。Trait 也可以包含抽象字段和具体字段，以及需要实现的抽象方法。

总结来说，Trait 是 Scala 中一种强大的机制，用于实现代码的复用、功能的组合和扩展。通过使用 Trait，您可以以一种灵活和可组合的方式编写可重用的代码。

---

### 方法和函数的区别

在 Scala 中，方法（Method）和函数（Function）是两个相关但有着不同特点的概念。

**方法 (Method)** 是属于类或对象的成员，它是一段可重用的代码块，用于执行特定的操作。方法可以有参数、返回值以及可选的访问修饰符。方法必须被定义在类或对象内部，并通过实例或者类来调用。方法的定义和调用方式通常和面向对象编程的思想相符。

下面是一个示例方法的定义和调用：

```scala
class MyClass {
  def add(a: Int, b: Int): Int = {
    a + b
  }
}

val obj = new MyClass()
val result = obj.add(2, 3) // 调用方法，结果为 5
```

**函数 (Function)** 是一个独立的值，它可以被赋值给变量、作为参数传递和作为返回值返回。在 Scala 中，函数被视为一等公民，因此可以像其他值一样进行操作和传递。函数定义可以使用 `=>` 符号来指定输入参数和函数体，函数可以没有参数或者多个参数，可以有返回值或者无返回值。

下面是一个示例函数的定义和使用：

```scala
val add: (Int, Int) => Int = (a, b) => a + b

val result = add(2, 3) // 调用函数，结果为 5
```

在这个例子中，我们定义了一个接收两个整数参数并返回它们之和的函数 `add`。通过将函数赋值给变量，我们可以像调用方法一样使用该函数。

函数和方法的主要区别如下：

1. **语法定义**：方法必须属于类或对象，并使用 `def` 关键字进行定义。函数则可以独立存在，可以使用函数字面量的语法进行定义。

2. **调用方式**：方法通过实例或者类来调用，使用点符号（`.`）来访问。函数可以直接使用函数名和参数列表进行调用。

3. **作为值**：方法不可以作为独立的值进行操作和传递。函数可以作为变量进行赋值、作为参数传递和作为返回值返回。

4. **一等公民**：函数在 Scala 中被视为一等公民，具有与其他值相同的地位和操作能力。方法则是类和对象的一部分，并不具备函数的一等公民特性。

需要注意的是，在 Scala 中，方法可以被隐式地转换为函数。这意味着可以将方法作为参数传递给期望函数类型的方法或函数，Scala 会自动将其转换为函数。这种转换是由编译器在背后完成的，使得方法和函数在使用上可以互相替代。

综上所述，方法和函数在 Scala 中有一些区别，包括语法定义、调用方式、作为值的能力和一等公民特性。这些概念的理解对于编写和组织 Scala 代码非常重要。

---
