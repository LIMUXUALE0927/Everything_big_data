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

## Scala 集合

### 不可变数组

```scala
// 创建不可变数组
val arr1 = new Array[Int](5)
// 另一种创建不可变数组的方式，来自于Array的伴生对象的apply方法
val arr2 = Array(1, 2, 3, 4, 5)

println(arr1(0))
arr1(1) = 1

// for 循环
// arr1.indices == 0 until arr1.length
for (i <- arr2.indices) {
  println(arr2(i))
}

for (elem <- arr2) {
  println(elem)
}

val iter = arr2.iterator
while (iter.hasNext) {
  println(iter.next())
}

arr2.foreach(println)

arr2.mkString(",")

// 添加元素（不可变数组添加后会返回新数组）
val arr3 = arr2 :+ 6
val arr4 = 0 +: arr2
```

```scala
// 多维数组
val arr = Array.ofDim[Int](2, 3) // 2 rows, 3 columns
arr(0)(0) = 1

for (i <- arr.indices; j <- arr(i).indices) {
  println(s"($i)($j) = ${arr(i)(j)}")
}

arr.foreach(row => row.foreach(println))
arr.foreach(_.foreach(println))
```

---

### 可变数组

```scala
import scala.collection.mutable.ArrayBuffer

val arr1 = new ArrayBuffer[Int]()
val arr2 = ArrayBuffer[Int](1, 2, 3, 4, 5)

println(arr1)
println(arr2)

// 添加元素
arr1.append(2)
arr1.prepend(1)
arr1.insert(2, 3)
println(arr1)

arr1.appendAll(arr2)
println(arr1)

// 删除元素
arr1.remove(0)  // 删除下标为 0 的元素
arr1 -= 2  // 删除第一个出现的 2

// 可变数组转不可变数组
val arr3 = arr1.toArray
// 不可变数组转可变数组
val arr4 = arr3.toBuffer
```

---

### 不可变 List

```scala
val list1 = List(1, 2, 3, 4, 5)

println(list1(1))
list1.foreach(println)

// 添加元素
val list2 = list1 :+ 6
val list3 = 0 +: list1

println(list3)

// 合并列表
val list4 = list1 ++ list1
println(list4)
```

### 可变 List

```scala
import scala.collection.mutable.ListBuffer

val list1 = new ListBuffer[Int]()
val list2 = ListBuffer(1, 2, 3)

// 添加元素
list1.append(2, 3)
list1.prepend(1)
list1 += 4 += 5
0 +=: list1
println(list1)

// 合并列表
val list3 = ListBuffer(4, 5)
list2.appendAll(list3)
println(list2)
```

---

### 不可变 Set

默认情况下，Scala 使用的是不可变的 Set，如果想使用可变的 Set，需要引入 `scala.collection.mutable.Set` 包。

```scala
val set1 = Set(1, 2, 3, 3)
println(set1)

// 添加元素
val set2 = set1 + 4

// 合并 Set
val set3 = set1 ++ Set(4, 5, 6)

// 删除元素
val set4 = set1 - 3
```

---

### 可变 Set

```scala
import scala.collection.mutable

val set1 = mutable.Set(1, 2, 3)
println(set1)

// 添加元素
set1.add(4)

// 删除元素
set1.remove(1)

// 合并两个集合
val set2 = mutable.Set(3, 4, 5)
set1 ++= set2
println(set1)
```

---

### 不可变 Map

```scala
val map1 = Map("a" -> 1, "b" -> 2, "c" -> 3)
val map2 = Map[String, Int]()
println(map1)

map1.foreach(println)

for (k <- map1.keys) println(k)
for (v <- map1.values) println(v)
for ((k, v) <- map1) println(k + "--" + v)

map1.get("a") // 注意结果为：Option[Int] = Some(1)
map1.get("a").get
map1("a")

map1.getOrElse("d", 0)
map1("d") // 报错
```

---

### 可变 Map

```scala
import scala.collection.mutable

val map1 = mutable.Map("a" -> 1, "b" -> 2, "c" -> 3)
println(map1)

map1.put("d", 4)

map1.remove("d")

map1("a") = 10
println(map1)
```

---

### 元组

```scala
val tuple = (1, 2, 3, 4)
tuple._1
tuple._2
tuple._3
tuple._4

for (x <- tuple.productIterator) {
  println(x)
}

tuple.productArity // 元组的长度
```

---

### 常用方法

集合通用属性和方法：

- 线性序列才有长度`length`、所有集合类型都有大小`size`。
- 遍历`for (elem <- collection)`、迭代器`for (elem <- collection.iterator)`。
- 生成字符串`toString` `mkString`，像`Array`这种是隐式转换为 scala 集合的，`toString`是继承自`java.lang.Object`的，需要自行处理。
- 是否包含元素`contains`。

衍生集合的方式：

- 获取集合的头元素`head`（元素）和剩下的尾`tail`（集合）。
- 集合最后一个元素`last`（元素）和除去最后一个元素的初始数据`init`（集合）。
- 反转`reverse`。
- 取前后 n 个元素`take(n) takeRight(n)`
- 去掉前后 n 个元素`drop(n) dropRight(n)`
- 交集`intersect`
- 并集`union`，线性序列的话已废弃用`concat`连接。
- 差集`diff`，得到属于自己、不属于传入参数的部分。
- 拉链`zip`，得到两个集合对应位置元素组合起来构成二元组的集合，大小不匹配会丢掉其中一个集合不匹配的多余部分。
- 滑窗`sliding(n, step = 1)`，框住特定个数元素，方便移动和操作。得到迭代器，可以用来遍历，每个迭代的元素都是一个 n 个元素集合。步长大于 1 的话最后一个窗口元素数量可能个数会少一些。

集合的简单计算操作：

- 求和`sum` 求乘积`product` 最小值`min` 最大值`max`
- `maxBy(func)`支持传入一个函数获取元素并返回比较依据的值，比如元组默认就只会判断第一个元素，要根据第二个元素判断就返回第二个元素就行`xxx.maxBy(_._2)`。
- 排序`sorted`，默认从小到大排序。从大到小排序`sorted(Ordering[Int].reverse)`。
- 按元素排序`sortBy(func)`，指定要用来做排序的字段。也可以再传一个隐式参数逆序`sortBy(func)(Ordering[Int].reverse)`
- 自定义比较器`sortWith(cmp)`，比如按元素升序排列`sortWith((a, b) => a < b)`或者`sortWith(_ < _)`，按元组元素第二个元素升序`sortWith(_._2 > _._2)`。

```scala
val list = List(3, 12, -1, 9, 6)
list.sorted
list.sorted.reverse
list.sorted(Ordering[Int].reverse)
list.sortWith(_ < _) // ascending
list.sortWith(_ > _) // descending

val list2 = List(("a", 5), ("b", 1), ("c", 8), ("d", 2), ("e", -3), ("f", 4))
list2.sorted // sort by first element
list2.sorted.reverse
list2.sortBy(_._2) // sort by second element
list2.sortBy(_._2).reverse
list2.sortBy(_._2)(Ordering[Int].reverse)
list2.sortWith(_._2 < _._2) // sort by second element (ascending)
list2.sortWith(_._2 > _._2) // sort by second element (descending)
```

---

### 高级计算函数

- 大数据的处理核心就是映射（map）和规约（reduce）。
- 映射操作（广义上的 map）：
  - 过滤：自定义过滤条件，`filter(Elem => Boolean)`
  - 转化/映射（狭义上的 map）：自定义映射函数，`map(Elem => NewElem)`
  - 扁平化（flatten）：将集合中集合元素拆开，去掉里层集合，放到外层中来。`flatten`
  - 扁平化+映射：先映射，再扁平化，`flatMap(Elem => NewElem)`
  - 分组（group）：指定分组规则，`groupBy(Elem => Key)`得到一个 Map，key 根据传入的函数运用于集合元素得到，value 是对应元素的序列。
- 规约操作（广义的 reduce）：
  - 简化/规约（狭义的 reduce）：对所有数据做一个处理，规约得到一个结果（比如连加连乘操作）。`reduce((CurRes, NextElem) => NextRes)`，传入函数有两个参数，第一个参数是第一个元素（第一次运算）和上一轮结果（后面的计算），第二个是当前元素，得到本轮结果，最后一轮的结果就是最终结果。`reduce`调用`reduceLeft`从左往右，也可以`reduceRight`从右往左（实际上是递归调用，和一般意义上的从右往左有区别，看下面例子）。
  - 折叠（fold）：`fold(InitialVal)((CurRes, Elem) => NextRes)`相对于`reduce`来说其实就是`fold`自己给初值，从第一个开始计算，`reduce`用第一个做初值，从第二个元素开始算。`fold`调用`foldLeft`，从右往左则用`foldRight`（翻转之后再`foldLeft`）。具体逻辑还得还源码。从右往左都有点绕和难以理解，如果要使用需要特别注意。
- 以上：

```scala
val list = List(1, 10, 100, 3, 5, 111)

// 1. map functions
// filter
val evenList = list.filter(_ % 2 == 0)
println(evenList)

// map
println(list.map(_ * 2))
println(list.map(x => x * x))

// flatten
val nestedList: List[List[Int]] = List(List(1, 2, 3), List(3, 4, 5), List(10, 100))
val flatList = nestedList(0) ::: nestedList(1) ::: nestedList(2)
println(flatList)

val flatList2 = nestedList.flatten
println(flatList2) // equals to flatList

// map and flatten
// example: change a string list into a word list
val strings: List[String] = List("hello world", "hello scala", "yes no")
val splitList: List[Array[String]] = strings.map(_.split(" ")) // divide string to words
val flattenList = splitList.flatten
println(flattenList)

// merge two steps above into one
// first map then flatten
val flatMapList = strings.flatMap(_.split(" "))
println(flatMapList)

// divide elements into groups
val groupMap = list.groupBy(_ % 2) // keys: 0 & 1
val groupMap2 = list.groupBy(data => if (data % 2 == 0) "even" else "odd") // keys : "even" & "odd"
println(groupMap)
println(groupMap2)

val worldList = List("China", "America", "Alice", "Curry", "Bob", "Japan")
println(worldList.groupBy(_.charAt(0)))

// 2. reduce functions
// narrowly reduce
println(List(1, 2, 3, 4).reduce(_ + _)) // 1+2+3+4 = 10
println(List(1, 2, 3, 4).reduceLeft(_ - _)) // 1-2-3-4 = -8
println(List(1, 2, 3, 4).reduceRight(_ - _)) // 1-(2-(3-4)) = -2, a little confusing

// fold
println(List(1, 2, 3, 4).fold(0)(_ + _)) // 0+1+2+3+4 = 10
println(List(1, 2, 3, 4).fold(10)(_ + _)) // 10+1+2+3+4 = 20
println(List(1, 2, 3, 4).foldRight(10)(_ - _)) // 1-(2-(3-(4-10))) = 8, a little confusing
```

集合应用案例：

- Map 的默认合并操作是用后面的同 key 元素覆盖前面的，如果要定制为累加他们的值可以用`fold`。

```scala
// merging two Map will override the value of the same key
// custom the merging process instead of just override
val map1 = Map("a" -> 1, "b" -> 3, "c" -> 4)
val map2 = mutable.Map("a" -> 6, "b" -> 2, "c" -> 5, "d" -> 10)
val map3 = map1.foldLeft(map2)(
    (mergedMap, kv) => {
        mergedMap(kv._1) = mergedMap.getOrElse(kv._1, 0) + kv._2
        mergedMap
    }
)
println(map3) // HashMap(a -> 7, b -> 5, c -> 9, d -> 10)
```

- 经典案例：单词计数：分词，计数，取排名前三结果。

```scala
// count words in string list, and get 3 highest frequency words
def wordCount(): Unit = {
    val stringList: List[String] = List(
        "hello",
        "hello world",
        "hello scala",
        "hello spark from scala",
        "hello flink from scala"
    )

    // 1. split
    val wordList: List[String] = stringList.flatMap(_.split(" "))
    println(wordList)

    // 2. group same words
    val groupMap: Map[String, List[String]] = wordList.groupBy(word => word)
    println(groupMap)

    // 3. get length of the every word, to (word, length)
    val countMap: Map[String, Int] = groupMap.map(kv => (kv._1, kv._2.length))

    // 4. convert map to list, sort and take first 3
    val countList: List[(String, Int)] = countMap.toList
        .sortWith(_._2 > _._2)
        .take(3)

    println(countList) // result
}
```

- 单词计数案例扩展，每个字符串都可能出现多次并且已经统计好出现次数，解决方式，先按次数合并之后再按照上述例子处理。

```scala
// strings has their frequency
def wordCountAdvanced(): Unit = {
    val tupleList: List[(String, Int)] = List(
        ("hello", 1),
        ("hello world", 2),
        ("hello scala", 3),
        ("hello spark from scala", 1),
        ("hello flink from scala", 2)
    )

    val newStringList: List[String] = tupleList.map(
        kv => (kv._1.trim + " ") * kv._2
    )

    // just like wordCount
    val wordCountList: List[(String, Int)] = newStringList
        .flatMap(_.split(" "))
        .groupBy(word => word)
        .map(kv => (kv._1, kv._2.length))
        .toList
        .sortWith(_._2 > _._2)
        .take(3)

    println(wordCountList) // result
}
```

- 当然这并不高效，更好的方式是利用上已经统计的频率信息。

```scala
def wordCountAdvanced2(): Unit = {
    val tupleList: List[(String, Int)] = List(
        ("hello", 1),
        ("hello world", 2),
        ("hello scala", 3),
        ("hello spark from scala", 1),
        ("hello flink from scala", 2)
    )

    // first split based on the input frequency
    val preCountList: List[(String, Int)] = tupleList.flatMap(
        tuple => {
            val strings: Array[String] = tuple._1.split(" ")
            strings.map(word => (word, tuple._2)) // Array[(String, Int)]
        }
    )

    // group as words
    val groupedMap: Map[String, List[(String, Int)]] = preCountList.groupBy(_._1)
    println(groupedMap)

    // count frequency of all words
    val countMap: Map[String, Int] = groupedMap.map(
        kv => (kv._1, kv._2.map(_._2).sum)
    )
    println(countMap)

    // to list, sort and take first 3 words
    val countList = countMap.toList.sortWith(_._2 > _._2).take(3)
    println(countList)
}
```

---

## Scala 隐式转换

在 Scala 中，隐式转换是一种特性，它允许编译器在需要时自动地进行类型转换或添加方法调用。通过隐式转换，可以方便地扩展现有类的功能，提供更富表现力和灵活性的代码。

以下是关于 Scala 隐式转换的一些重要概念和用法：

- **隐式转换函数（Implicit Conversion Functions）**：隐式转换函数是一种特殊的函数，它用于将一个类型转换为另一个类型。隐式转换函数使用 `implicit` 关键字进行声明，并且通常定义在隐式转换函数的作用域内。当编译器发现类型不匹配的表达式时，它会在作用域内查找适用的隐式转换函数，并自动应用转换。例如：

```scala
implicit def intToString(i: Int): String = i.toString

val str: String = 42  // 编译器自动应用隐式转换将 Int 转换为 String
println(str)  // 输出: "42"
```

在上面的示例中，我们定义了一个将 `Int` 类型转换为 `String` 类型的隐式转换函数 `intToString`。当我们将整数 `42` 赋值给字符串类型的变量时，编译器会自动应用该隐式转换函数。

- **隐式参数（Implicit Parameters）**：隐式参数允许我们定义在函数或方法中的参数，在调用时使用隐式值而不是显式传递。隐式参数使用 `implicit` 关键字进行声明，并且通常定义在作用域内。当调用带有隐式参数的函数时，编译器会在作用域内查找适用的隐式值，并自动传入函数。例如：

```scala
def greet(name: String)(implicit greeting: String): Unit = {
  println(s"$greeting, $name!")
}

implicit val defaultGreeting: String = "Hello"

greet("Alice")  // 编译器自动传入隐式参数 defaultGreeting
```

在上面的示例中，我们定义了一个带有隐式参数 `greeting` 的 `greet` 函数。我们还定义了一个隐式值 `defaultGreeting`，它被自动传入函数作为隐式参数。当我们调用 `greet("Alice")` 时，编译器会自动应用隐式值 `"Hello"` 作为 `greeting` 参数的值。

- **隐式类（Implicit Classes）**：隐式类是一种特殊的类，它用于在现有类上添加附加的方法。隐式类使用 `implicit` 关键字进行声明，并且必须定义在顶层对象、类或特质中。当隐式类的实例调用附加的方法时，编译器会自动将实例转换为隐式类，并应用相应的方法。例如：

```scala
implicit class StringOps(s: String) {
  def greet(): Unit = {
    println(s"Hello, $s!")
  }
}

"Alice".greet()  // 编译器自动将字符串转换为 StringOps，并应用 greet 方法
```

在上面的示例中，我们定义了一个隐式类 `StringOps`，它在 `String` 类型上添加了一个 `greet` 方法。当我们使用字符串字面量 `"Alice"` 调用 `greet()` 方法时，编译器会自动将字符串转换为 `StringOps` 的实例，并应用 `greet` 方法。

---

## Scala 模式匹配

```scala
value match {
    case caseVal1 => returnVal1
    case caseVal2 => returnVal2
    ...
    case _ => defaultVal
}
```

- 每一个 case 条件成立才返回，否则继续往下走。
- `case`匹配中可以添加模式守卫，用条件判断来代替精确匹配。

```scala
def abs(num: Int): Int= {
    num match {
        case i if i >= 0 => i
        case i if i < 0 => -i
    }
}
```

- 模式匹配支持类型：所有类型字面量，包括字符串、字符、数字、布尔值、甚至数组列表等。
- 你甚至可以传入 `Any` 类型变量，匹配不同类型常量。
- 需要注意默认情况处理，`case _` 也需要返回值，如果没有但是又没有匹配到，就抛出运行时错误。默认情况 `case _` 不强制要求通配符（只是在不需要变量的值建议这么做），也可以用 `case abc` 一个变量来接住，可以什么都不做，可以使用它的值。
- 通过指定匹配变量的类型（用特定类型变量接住），可以匹配类型而不匹配值，也可以混用。
- 需要注意类型匹配时由于泛型擦除，可能并不能严格匹配泛型的类型参数，编译器也会报警告。但`Array`是基本数据类型，对应于 java 的原生数组类型，能够匹配泛型类型参数。

```scala
// match type
def describeType(x: Any) = x match {
    case i: Int => "Int " + i
    case s: String => "String " + s
    case list: List[String] => "List " + list
    case array: Array[Int] => "Array[Int] " + array
    case a => "Something else " + a
}
println(describeType(20)) // match
println(describeType("hello")) // match
println(describeType(List("hi", "hello"))) // match
println(describeType(List(20, 30))) // match
println(describeType(Array(10, 20))) // match
println(describeType(Array("hello", "yes"))) // not match
println(describeType((10, 20))) // not match
```

- 对于数组可以定义多种匹配形式，可以定义模糊的元素类型匹配、元素数量匹配或者精确的某个数组元素值匹配，非常强大。

```scala
for (arr <- List(
    Array(0),
    Array(1, 0),
    Array(1, 1, 0),
    Array(10, 2, 7, 5),
    Array("hello", 20, 50)
)) {
    val result = arr match {
        case Array(0) => "0"
        case Array(1, 0) => "Array(1, 0)"
        case Array(x: Int, y: Int) => s"Array($x, $y)" // Array of two elements
        case Array(0, _*) => s"an array begin with 0"
        case Array(x, 1, z) => s"an array with three elements, no.2 is 1"
        case Array(x:String, _*) => s"array that first element is a string"
        case _ => "somthing else"
    }
    println(result)
```

- `List` 匹配和 `Array` 差不多，也很灵活。还可用用集合类灵活的运算符来匹配。
  - 比如使用 `::` 运算符匹配 `first :: second :: rest`，将一个列表拆成三份，第一个第二个元素和剩余元素构成的列表。
- 注意模式匹配不仅可以通过返回值当做表达式来用，也可以仅执行语句类似于传统 `switch-case` 语句不关心返回值，也可以既执行语句同时也返回。
- 元组匹配：
  - 可以匹配 n 元组、匹配元素类型、匹配元素值。如果只关心某个元素，其他就可以用通配符或变量。
  - 元组大小固定，所以不能用 `_*`。

变量声明匹配：

- 变量声明也可以是一个模式匹配的过程。
- 元组常用于批量赋值。
- `val (x, y) = (10, "hello")`
- `val List(first, second, _*) = List(1, 3, 4, 5)`
- `val List(first :: second :: rest) = List(1, 2, 3, 4)`

`for` 推导式中也可以进行模式匹配：

- 元组中取元素时，必须用 `_1 _2 ...`，可以用元组赋值将元素赋给变量，更清晰一些。
- `for ((first, second) <- tupleList)`
- `for ((first, _) <- tupleList)`
- 指定特定元素的值，可以实现类似于循环守卫的功能，相当于加一层筛选。比如 `for ((10, second) <- tupleList)`
- 其他匹配也同样可以用，可以关注数量、值、类型等，相当于做了筛选。
- 元组列表匹配、赋值匹配、`for` 循环中匹配非常灵活，灵活运用可以提高代码可读性。

匹配对象：

- 对象内容匹配。
- 直接 `match-case` 中匹配对应引用变量的话语法是有问题的。编译报错信息提示：不是样例类也没有一个合法的 `unapply/unapplySeq` 成员实现。
- 要匹配对象，需要实现伴生对象 `unapply` 方法，用来对对象属性进行拆解以做匹配。

样例类：

- 第二种实现对象匹配的方式是样例类。
- `case class className` 定义样例类，会直接将打包 `apply` 和拆包 `unapply` 的方法直接定义好。
- 样例类定义中主构造参数列表中的 `val` 甚至都可以省略，如果是 `var` 的话则不能省略，最好加上的感觉，奇奇怪怪的各种边角简化。

对象匹配和样例类例子：

```scala
object MatchObject {
    def main(args: Array[String]): Unit = {
        val person = new Person("Alice", 18)

        val result: String = person match {
            case Person("Alice", 18) => "Person: Alice, 18"
            case _ => "something else"
        }
        println(result)

        val s = Student("Alice", 18)
        val result2: String = s match {
            case Student("Alice", 18) => "Student: Alice, 18"
            case _ => "something else"
        }
        println(result2)
    }
}


class Person(val name: String, val age: Int)
object Person {
    def apply(name: String, age: Int) = new Person(name, age)
    def unapply(person: Person): Option[(String, Int)] = {
        if (person == null) { // avoid null reference
            None
        } else {
            Some((person.name, person.age))
        }
    }
}

case class Student(name: String, age: Int) // name and age are vals
```

偏函数：

- 偏函数是函数的一种，通过偏函数我们可以方便地对参数做更精确的检查，例如偏函数输入类型是 `List[Int]`，需要第一个元素是 0 的集合，也可以通过模式匹配实现的。
- 定义：

```scala
val partialFuncName: PartialFunction[List[Int], Option[Int]] = {
    case x :: y :: _ => Some(y)
}
```

- 通过一个变量定义方式定义，`PartialFunction` 的泛型类型中，前者是参数类型，后者是返回值类型。函数体中用一个 `case` 语句来进行模式匹配。上面例子返回输入的 `List` 集合中的第二个元素。
- 一般一个偏函数只能处理输入的一部分场景，实际中往往需要定义多个偏函数用以组合使用。
- 例子：

```scala
object PartialFunctionTest {
    def main(args: Array[String]): Unit = {
        val list: List[(String, Int)] = List(("a", 12), ("b", 10), ("c", 100), ("a", 5))

        // keep first constant and double second value of the tuple
        // 1. use map
        val newList = list.map(tuple => (tuple._1, tuple._2 * 2))
        println(newList)

        // 2. pattern matching
        val newList1 = list.map(
            tuple => {
                tuple match {
                    case (x, y) => (x, y * 2)
                }
            }
        )
        println(newList1)

        // simplify to partial function
        val newList2 = list.map {
            case (x, y) => (x, y * 2) // this is a partial function
        }
        println(newList2)

        // application of partial function
        // get absolute value, deal with: negative, 0, positive
        val positiveAbs: PartialFunction[Int, Int] = {
            case x if x > 0 => x
        }
        val negativeAbs: PartialFunction[Int, Int] = {
            case x if x < 0 => -x
        }
        val zeroAbs: PartialFunction[Int, Int] = {
            case 0 => 0
        }

        // combine a function with three partial functions
        def abs(x: Int): Int = (positiveAbs orElse negativeAbs orElse zeroAbs) (x)
        println(abs(-13))
        println(abs(30))
        println(abs(0))
    }
}
```

---

## Scala 样例类

在 Scala 中，`case class` 是一种特殊的类，用于定义不可变的数据结构。它们具有一些特殊的属性和方法，使得它们非常适合用于模式匹配和数据传递。

以下是 `case class` 的一些特点和用法：

1. **不可变性**：`case class` 默认是不可变的，即一旦创建，它们的属性值是不可修改的。这有助于编写更安全和可靠的代码。

2. **自动生成的方法**：编译器会自动生成一些常用的方法，如构造函数、getter 和 setter 方法、`equals` 和 `hashCode` 方法等。这使得使用 `case class` 更加方便。

3. **模式匹配**：`case class` 是 Scala 中模式匹配（pattern matching）的重要组成部分。模式匹配允许你根据数据结构的形状和属性值来进行条件匹配和提取。

4. **值比较**：`case class` 的 `equals` 方法会按照属性值比较两个对象是否相等，而不是比较引用。这使得可以直接使用 `==` 运算符进行值比较。

下面是一个简单的 `case class` 的示例：

```scala
case class Person(name: String, age: Int)

// 创建一个 Person 对象
val person = Person("Alice", 30)

// 获取属性值
val name = person.name
val age = person.age

// 输出对象
println(person) // 输出：Person(Alice,30)

// 模式匹配
person match {
  case Person("Alice", age) => println(s"Hello, Alice! Your age is $age")
  case Person(name, _) => println(s"Hello, $name!")
}
```

在上面的示例中，`Person` 是一个 `case class`，它有两个属性 `name` 和 `age`。我们可以使用 `Person` 的构造函数来创建对象，并通过属性名访问属性值。我们还可以使用模式匹配来根据对象的属性值进行条件匹配和提取。

总而言之，`case class` 是 Scala 中用于定义不可变数据结构的特殊类，它们提供了方便的方法和模式匹配的支持。这使得在 Scala 中处理复杂的数据结构变得更加简单和直观。
