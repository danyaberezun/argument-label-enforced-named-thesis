# Additional findings

Any results of the research, that do not directly affect the two main features, but still worth sharing

## Multiple variadic arguments: other languages and Kotlin

In Kotlin, as in many older programming languages, there can be only one variadic argument in a function. But do we really need this limitation, or is it here present as a remnant of old times? There are languages, where one can use as many variadics as they want, including Swift and Scala, the latter being a JVM language.

Here is an example in Swift with the usage of two variadic parameters in one function. 

```swift
extension UIView {
  func addSubviews(_ views: UIView..., constraints: NSLayoutConstraint...) {
    views.forEach {
      addSubview($0)
      $0.translatesAutoresizingMaskIntoConstraints = false
    }
    constraints.forEach { $0.isActive = true }
  }
}

myView.addSubviews(v1, v2, constraints: v1.widthAnchor.constraint(equalTo: v2.widthAnchor),
                                        v1.heightAnchor.constraint(equalToConstant: 40),
                                        /* More Constraints... */)
```

Of course, one could replace some of the variadics with arrays, but replacing only one of them looks inconsistent, and replacing both results in additional visual overhead without particular reason.
Further discussions about lifting the limitation in Swift: [on forums](https://forums.swift.org/t/lifting-the-1-variadic-param-per-function-restriction/33787?u=owenv)

Another example from discussions and proposal [SE-0284](https://github.com/apple/swift-evolution/blob/main/proposals/0284-multiple-variadic-parameters.md):

```swift
func assertArgs(
      _ args: String...,
      parseTo driverKind: DriverKind,
      leaving remainingArgs: ArraySlice<String>,
      file: StaticString = #file, line: UInt = #line
    ) throws { /* Implementation Omitted */ }

try assertArgs("swift", "-foo", "-bar", parseTo: .interactive, leaving: ["-foo", "-bar"])
```

Looks pretty inconsistent with the first argument being passed as a variadic argument, and the second being passed as an `ArraySlice`.

All in all, it seems that it can be beneficial to lift the limitation. And, what is also important, it does not need any introduction of a new syntax: in Kotlin every argument after `vararg` already needs to be in named form, therefore it becomes pretty trivial to separate further variadic arguments from others.

## Multiple variadic arguments

While being only a part of the total list of ideas, this one is interesting for it being implemented with particular ease.

## Multiple variadic arguments in Swift

In Swift, one can actually use multiple variadic arguments in one function with ease. 

That was added in one of the proposals, but it was more of a restriction lifting than a new thing: the only change was to remove the limitation from the parser. Proposal link: [SE-0284](https://github.com/apple/swift-evolution/blob/main/proposals/0284-multiple-variadic-parameters.md)

As with all parameters after a variadic, all subsequent variadics must be labelled

There is a discussion about it: [on swift forum](https://forums.swift.org/t/lifting-the-1-variadic-param-per-function-restriction/33787?u=owenv)

## Multiple variadic arguments in Scala

What is interesting, is that Scala, a JVM-based language, also can have multiple variadic arguments, by usage of the currying syntax to separate different variadic arguments from each other.

Here is [the discussion on stackoverflow](https://stackoverflow.com/questions/30133367/scala-why-cant-a-method-have-multiple-vararg-arguments) about it.

And here is the part about currying (also known as multiple parameters list) in the official documentation: [multiple parameters lists](https://docs.scala-lang.org/tour/multiple-parameter-lists.html)

Simple example of it in use:

```scala
def varargUse(ints: Int*)(strings: String*) =
        ints.mkString(" ") + string.mkString(" ")

varargsUse(1, 2, 3)("a", "b")
// >> 1 2 3a b: java.lang.String
```

How does it work? Pretty simple: before compilation to JVM, two of the compilation parts are desugaring and uncurrying, during which such varargs are being turned into `scala/collection/immutable/Seq<Ljava/lang/Object>` and merged into a single parameters list.

This way, by the time of compilation into JVM Bytecode, there is no trace of variadics left in the source.

And, [judging by the Github search](https://github.com/search?q=language%3AScala++%2Fdef.*%5BA-Za-z%5D%2B%5C*.*%5BA-Za-z%5D%2B%5C*.*%3D%2F&type=code), some people do use multiple variadic arguments in their function.

## Variadic arguments in Kotlin and Java

In the process of compilation of both Kotlin and Java into bytecode, all variadic arguments are also desugared and therefore turned into arrays. The only difference here is that these languages use actual JVM arrays, and also mark the function with variadic argument with flag `ACC_VARARGS`. But, as it turns out, if you compile the same Kotlin code using an array and using variadics, the result will be the same to the point of initial source code lines difference and presence of the flag `ACC_VARARGS`. 

What is the particular use of this flag, apart from indicating the presence of a variadic argument *in the initial source code?* 

