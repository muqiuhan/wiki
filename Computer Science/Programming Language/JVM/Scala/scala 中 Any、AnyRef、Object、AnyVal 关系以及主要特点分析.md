#scala #fp

- Any 是一个 abstract 类， scala 中的顶级父类
- AnyVal 是一个 abstract 类，继承 Any，目的是取代 primary 类型
- AnyRef 是一个 trait，继承 Any，重写了 Any 中部分方法
- Any 和 Object 定义上没有任何关系
- AnyRef 和 Object 定义上没有任何关系
- scala 的继承体系是通过 Any、AnyRef  实现的。为了兼容 java 的继承体系，scala 编译器将 AnyRef 置于与 Object 同等地位，二者的  Class 类型相同。因此凡是 Object 继承体系的子类，都是  AnyRef  的子类；而 AnyRef 继承体系生成的子类，也是 Object 的子类。这些子类既是 AnyRef 又是 Object，这样 scala 保证定义的类能够被 jvm 加载，而编码时我们按照 scala 语言书写即可。
- scala 让编程者感觉 Any 类是  scala 的顶级父类。作为  jvm 来说，Object 才是顶级父类，scala 编译器必然将 Any、AnyRef 编译为 Object 的子类型，这是 scala 编译器来实现的。 编程者完全无感。
- scala 类构造同时结合了 c++ 和 javascript 特点，使用方式类似于 javascript。

**总结：scala 语法规定了自己的继承体系(Any)，本质跟 java 不同，兼容 java。**


```scala 3
package scala

abstract class Any {
  def equals(that: Any): Boolean // 值比较
  def hashCode(): Int // hash 值
  def toString(): String

  final def getClass(): Class[_] = sys.error("getClass")

  final def ==(that: Any): Boolean = this equals that // 值比较，参数可为 null
  final def !=(that: Any): Boolean = !(this == that) // 值比较
  final def ## : Int = sys.error("##") // hash 值，参数可为 null
  final def isInstanceOf[T0]: Boolean = sys.error("isInstanceOf") //是否为 T0 实例
  final def asInstanceOf[T0]: T0 = sys.error("asInstanceOf") //强转为 T0
}

abstract class AnyVal extends Any {
  def getClass(): Class[_ <: AnyVal] = null
}

trait AnyRef extends Any

![复制代码](https://assets.cnblogs.com/images/copycode.gif)

 继承示例：

![复制代码](https://assets.cnblogs.com/images/copycode.gif)

// 父类  
class PersonFather(name: String, age: Int) {
  println("PersonRather is created!")

  def walk(): Unit = {
    println("Person walking ...")
  }
}
  
// 继承
class StudentSon(name: String, age: Int, var stuNo: Int) extends PersonFather(name, age) {
  println("StudentSon is created!")

  override def walk(): Unit = {
    super.walk()
    println("Student walking ......")
  }
}