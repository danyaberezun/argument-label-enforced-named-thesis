# Interface argument names override problem: language comparison

Suppose you have an interface with a function, which has some (named) parameters. Suppose also that you then have an implementation of this interface, with a specification of the function above. But, for some reason, you want to use different names for the parameters. Is it possible? What will happen when you do a named call of such a function? What if you treat the class not as *the* class, but just as an *interface implementation*? 

On this page, we will do a comparison of how different languages handle such problems, and whether the languages even have this problem.

## Kotlin

```kotlin
interface Consumer<T> {
    fun consume(input: T)
}

class MessageConsumer : Consumer<Message> {
    override fun consume(message: Message) = …
}

fun send(message: Message) {
   val consumer: Consumer<Message> = …
   consumer.consume(message)
   consumer.consume(input = message)
   consumer.consume(message = message) // error, no overload with that parameter name / incorrect parameter name

   val messageConsumer: MessageConsumer = …
   messageConsumer.consume(message)
   messageConsumer.consume(message = message)
   messageConsumer.consume(input = message) // error, incorrect parameter name (but might as well be allowed)
}
```


```kotlin
interface Consumer {
    fun consume(input: String)
}

class MessageConsumer : Consumer {
    override fun consume(message: String) {
        println("Got message: $message")
    }
}

fun main() {
    val consumer: Consumer = MessageConsumer()
    val messageConsumer: MessageConsumer = MessageConsumer()
    
    consumer.consume(input = "Message") // Got message: Message
//     messageConsumer.consume(input = "Message") // No value passed for...
    
//     consumer.consume(message = "Message")   // and Cannot find a parameter
    messageConsumer.consume(message = "Message") // Got message: Message
}
```

Kotlin allows you to make an implementation/override of a function with different names for the variables. It displays a warning (`The corresponding parameter in the supertype … is named …. This may cause problems when calling this function with named arguments`) but still allows you to do it, although with interesting results sometimes. And mind the `No value/Cannot find a parameter` errors, which seem logical here.

```kotlin
open class Base {
    open fun print(a: String, b: String) {
        println("Base got a: $a, b: $b")
    }
}

class Derived : Base() {
    override fun print(b: String, a: String) {
        println("Derived got b: $b, a: $a")
    }
}

fun main() {
    val base: Base = Derived()
    val derived: Derived = Derived()
    base.print("first", "second")
    base.print(a = "first", b = "second")
    base.print(b = "first", a = "second")
    
    derived.print("first", "second")
    derived.print(a = "first", b = "second")
    derived.print(b = "first", a = "second")
}
```

```kotlin
Derived got b: first, a: second
Derived got b: first, a: second
Derived got b: second, a: first
Derived got b: first, a: second
Derived got b: second, a: first
Derived got b: first, a: second
```

Both functions here do the same thing: they print their first and second arguments, in this exact order.

In both cases, the function called is `Derived::print`, but we can see that the order of the arguments depends on the type of the variable, thus the difference between the outputs.

## Swift

```swift
protocol Consumer {
    func consume(input: String)
}

class MessageConsumer: Consumer {
    func consume(message: String) {
        print("Got message: " + message)
    }
}
```

But in Swift, you cannot implement a protocol using a function with different argument labels.

```swift
error: instance method 'consume(message:)' has different argument labels from those required by protocol 'Consumer' ('consume(input:)')
```

Although only labels must be equal. If you set the argument label the same as in protocol (and different from the parameter name), everything will be okay.

```swift
import Foundation

protocol Consumer {
    func consume(input: String)
}

class MessageConsumer: Consumer {
    func consume(input message: String) {
        print("Got message: " + message)
    }
}

let consumer: Consumer = MessageConsumer()
let messageConsumer: MessageConsumer = MessageConsumer()

consumer.consume(input: "message") // Got message: message
messageConsumer.consume(input: "message") // Got message: message
```

But the labels are now the same, so there is nothing to play with.

Now to the example of overloading:

```swift
import Foundation

class Base {
    func printMe(a: String, b: String) {
        print("Base got a: " + a + ", b: " + b)
    }
}

class Derived: Base {
    func printMe(b: String, a: String) {
        print("Derived got b: " + b + ", a: " + a)
    }
}

let base: Base = Derived()
let derived: Derived = Derived()
base.printMe(a: "first", b: "second")
// base.printMe(b: "first", a: "second") error: argument 'a' must precede argument 'b'

derived.printMe(a: "first", b: "second")
derived.printMe(b: "first", a: "second")
```

```swift
Base got a: first, b: second
Base got a: first, b: second
Derived got b: first, a: second
```

Without the `override` keyword, all we get is just two functions, with overload by parameter names. But if we try to add `override`, we will get the `argument labels do not match` error. Yes, Swift has overloading by parameter names.

## Scala

Scala is more similar to Kotlin than Swift, which is obvious, considering the fact that it works on JVM too. Therefore the expected behaviour is somewhat similar to that of Kotlin.

```scala
trait Consumer {
  def consume(input: String): Unit
}

class MessageConsumer extends Consumer {
  def consume(message: String): Unit = println("Got message: " + message)
}

val consumer: Consumer = MessageConsumer()
val messageConsumer: MessageConsumer = MessageConsumer()

consumer.consume(input = "message")
// consumer.consume(message = "message") // method does not have a parameter message

// messageConsumer.consume(input = "message") // method does not have a parameter input
messageConsumer.consume(message = "message")
```

```scala
Got message: message
Got message: message
```

```scala
class Base {
  def print(a: String, b: String): Unit = println("Base got a: " + a + ", b: " + b)
}

class Derived extends Base {
  override def print(b: String, a: String): Unit = println("Derived got b: " + b + ", a:  " + a)
}

val base: Base = Derived()
val derived: Derived = Derived()

base.print("first", "second")
base.print(a = "first", b = "second")
base.print(b = "first", a = "second")

derived.print("first", "second")
derived.print(a = "first", b = "second")
derived.print(b = "first", a = "second")
```

```scala
Derived got b: first, a:  second
Derived got b: first, a:  second
Derived got b: second, a:  first
Derived got b: first, a:  second
Derived got b: second, a:  first
Derived got b: first, a:  second
```

The same results as in Kotlin.

## Java

There are no named parameters in Java! Therefore, there is no such problem, like, at all. All parameters are passed in strict order. There are only parameter names, but no argument labels. Welp.

## C++

And also no named parameters in C++. They were proposed, but the proposal was rejected. [https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4172.htm](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4172.htm)

## Python

There are named parameters in python, and there is a way to try and use `class`es as interfaces, and to make a value somewhat behave like a value of its superclass, even if it feels a little bit wrong. But, due to the way named arguments are implemented, the results are a little bit new:

```python
class Consumer:
    def consume(self, my_input):
        print(f"Not really an interface")

class MessageConsumer(Consumer):
    def consume(self, message):
        print(f"Got message: {message}")

consumer = MessageConsumer()
consumer.__class__ = Consumer

messageConsumer = MessageConsumer()

consumer.consume(my_input="Message")
# messageConsumer.consume(my_input="Message") TypeError: MessageConsumer.consume() got an unexpected keyword argument 'my_input'

# consumer.consume(message="Message") TypeError: Consumer.consume() got an unexpected keyword argument 'message'
messageConsumer.consume(message="Message")
```

```python
Not really an interface
Got message: Message
```

Well, pretty much similar to the usual behaviour, with the exception being `Consumer.consume` called on `consumer`, resulting in “fake” method being called.

Anyway, the second example:

```python
class Base:
    def print(self, a, b):
        print(f"Base got a: {a}, b: {b}")

class Derived(Base):
    def print(self, b, a):
        print(f"Derived got b: {b}, a: {a}")

base = Derived()
base.__class__ = Base

derived = Derived()

base.print("first", "second")
```

```python
Base got a: first, b: second
Base got a: first, b: second
Base got a: second, b: first
Derived got b: first, a: second
Derived got b: second, a: first
Derived got b: first, a: second
```

Somewhat similar to Swift, but you don’t have fixed order of arguments.
