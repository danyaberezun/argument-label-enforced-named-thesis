# Argument Labels

## Description

As was stated in the introduction, the main idea of Argument Labels is to separate the identifier used in the function calls with named arguments (**argument label**) and the identifier used inside the body of the function (**parameter name**).

Such separation would require several things to be implemented, including:
1. A way to specify two separate identifiers (parameter name and argument label) for an argument in function declaration
2. Linking the parameter name to the usages in the function body
3. Using argument label in the named arguments resolution/mapping.

One possible example of a function using argument labels (here in brackets):

```kotlin
fun <E> List<E>.index([of] element: E): Int? {
    // Use the `element` variable to find its index in the list
}

someList.index(of = someElement)

fun distance([between] a: Int, [and] b: Int): Int {
    return abs(a - b)
}

val d = distance(between = 6, and = 9)
```

### Discussion history

Firstly discussed in the same issue as the Enforced Named Argument Form, it is currently moved to a separate issue: [KT-34895](https://youtrack.jetbrains.com/issue/KT-34895/Internal-and-external-name-for-a-parameter-aka-Argument-Label). 

At that point, the idea is to allow developers to specify two names for an argument in their functions: external and internal. The external name acts as a use-site name, the one seen during the function call using the named form of arguments, and the internal acts as a parameter name, which is used in the body of a function.

As one can see, the idea is being preserved in this work, with the external names being argument labels and the internal names being parameter names.

There were many further discussions on the issue, and some posts at [Kotlin Discussions](discuss.kotlinlang.org) revolve around these features. Several findings and points from them will be presented in this document.

### Possible benefits

Adding a feature to a programming language, especially a big one, requires much work and can lead to significant changes and increased support and attention from the language development team. Therefore, it needs proper motivation and requests from the language community, which is the case in this situation.

Two common reasons for this feature and the Enforcing of Named Argument Form are already described in the introduction document, even though we still can briefly mention them here:

1. The introduction of argument labels can make the function calls look like sentences of natural language, which makes the code more self-documenting
2. Changing APIs and libraries under development can benefit from argument labels, as with them, one may be able to change the internal name of an argument without the need to change anything in the call sites.

More about these reasons can be read in the introduction document, while more specific reasons are described in the following parts.

#### Different meanings for arguments inside and outside

Sometimes, the information that a developer needs to know about an argument differs from that of an end user. It is not necessary that some information should be forgotten, but sometimes, different sides need to have accents on different properties of the arguments.

One possible example of this situation can be found here, in the situation where it is difficult to come up with a meaningful name that will reflect both sides of the value.

```kotlin
// Suppose we have a public API for requests
// And we do not use the argument labels feature
interface Repo {
    fun startRequest(query: String,
                     successAction: () -> Unit,
                     failureAction: (Throwable) -> Unit,
                     **observeOn: Scheduler**) /* <-- Okay so far... */
}

// From user's side everything is okay:
val repo = Repo()
repo.startRequest(
        query = "Foo",
        successAction = { /* ... */ },
        failureAction = { throwable -> /* ... */ },
        **observeOn = Schedulers.computation()** /* <-- Very clear what this does */
)
// But from the developer side things will turn a little bit unobvious
internal class DefaultRepo : Repo {
    override fun startRequest(query: String,
                              successAction: () -> Unit,
                              failureAction: (Throwable) -> Unit,
                              observeOn: Scheduler) {

        val disposable = retrofitApi.fooRequest(query = query)
                **.observeOn(observeOn)** /* who is `observeOn`? (in this particular case we can say that from the type name, but this is not always the case) */
                .doOnSuccess(successAction)
                .doOnError(failureAction)
                .subscribe()
    }
}
// Inverse situation: choose `scheduler` as a name for the argument
val repo = Repo()
repo.startRequest(
        query = "Foo",
        successAction = { /* ... */ },
        failureAction = { throwable -> /* ... */ },
        **scheduler = Schedulers.computation()** /* what is this scheduler used for? */
)
```

It can be seen that in this example, if we choose to have the parameter named "observeOn", its meaning will be unclear at the point it is being used, but if we choose the name "scheduler", then the purpose of the parameter is being unclear for the one using the function.


#### Interface implementation with different argument names

There is a problem related to the implementation of generic interfaces for more specific cases: usually, it would be more meaningful to use different names, as can be seen in the following example from the related issue: [KT-59531](https://youtrack.jetbrains.com/issue/KT-59531/Add-a-way-to-make-parameter-names-of-interface-functions-non-stable)

```kotlin
interface WidgetVisitor<R, D> {
    fun visit(widget: Widget, data: D): R
}

class WidgetVisitorImpl(val context: Context) : WidgetVisitor<Unit, Context> {
    override fun visit(widget: Widget, context: Context) {  // <-- warning on "context"
        ...
    }
}
```

Currently, this kind of overriding produces the warning: 

```
The corresponding parameter in the supertype 'WidgetVisitor' is named 'data'. This may cause problems when calling this function with named arguments.
```

Indeed, if the names of a parameter in a supertype and subtype differ, it becomes difficult to pass the parameter in the named form. Using the name from the supertype will only be possible when the value is currently in the type of supertype, and the same with the subtype:

```kotlin
val a: WidgetVisitor<Unit, Context> = WidgetVisitorImpl(Context())
val b: WidgetVisitorImpl = WidgetVisitorImpl(Context())
    
a.visit(Widget(), data=Context())
a.visit(Widget(), context=Context()) // Does not compile
b.visit(Widget(), data=Context()) // Does not compile
b.visit(Widget(), context=Context())
```

The second and the third calls produce the "Cannot find a parameter with this name: context/data" error.

This whole situation can be avoided by introducing argument labels. If one uses argument labels, they will only have to keep the argument labels the same across the sub/supertypes while being able to have different parameter names, depending on the implementation:

```kotlin
interface WidgetVisitor<R, D> {
    fun visit(widget: Widget, data: D): R // data is both a label and a name
}

class WidgetVisitorImpl(val context: Context) : WidgetVisitor<Unit, Context> {
    override fun visit(widget: Widget, data context: Context) {  // <-- label is still data
        ...
    }
}
```

This way, there will not be any problems with calls using the named argument form, as they both will use the `data` label, but in the implementation, it will be possible to use `context` instead of meaningless in the new context name `data`. 

Remark: this is not the only possible way to solve this problem. There were several proposed, like [having a special name for unnamed arguments](https://stackoverflow.com/questions/50672203/kotlin-explicitly-unnamed-function-arguments) or [disallowing named form for some arguments](https://youtrack.jetbrains.com/issue/KT-9872/Disallow-calling-a-method-with-named-argument), which shows that the problem itself is something people encounter during their work.

#### Unused arguments

Highly related to the previous topic, [some ask for syntax for nameless parameters](https://youtrack.jetbrains.com/issue/KT-8112/Provide-syntax-for-nameless-parameters). In the original issue, the motivation is to use an "empty" name for unused parameters, catch statements and situations where a function accepts a function with a specific amount of parameters, as seen in the example in [issue KTIJ-10594](https://youtrack.jetbrains.com/issue/KTIJ-10594).

With argument labels introduced, empty names can be implemented as either empty (that means, set to a special value, like `_` in Rust, for example) argument labels if one wants to disallow using named form for a parameter (same method, as in Swift), or empty parameter name, to indicate that it is not being used inside the function body.

#### Overload by parameter names

Some wish to have the ability to have different functions with the same name and signature, which differ only in the way the arguments are labelled. As mentioned in the issue [KT-43016](https://youtrack.jetbrains.com/issue/KT-43016/Support-method-overloads-on-parameter-names-like-Swift), Swift does have such overload, and, when compiling to Kotlin/Native on iOS with specific annotations, it does work. However, one may think having a feature specific to only one platform is strange. 

Argument labels, again, can be used to provide such overload if included in the resolution process. However, it may be challenging to implement from the point of Java Virtual Machine.

#### Reusing the name after processing

Suppose a developer has a function that accepts an object that needs an additional layer of processing from the developer's side. Suppose that this layer is also not supposed to be handled or even known by the end user of the function. In this situation, one would have to write something of the following kind:

```kotlin
fun doHeavyLifting(data: Data) {
    val processedData: Data = doSomePreprocessing(data)
    // do other stuff using a new, longer name
}
```

The disadvantages of this approach are the following:
1. The developer has to use a new name, which has to differ from the function parameter and be longer or less comfortable to use
2. The developer could easily mix `processedData` and `data`, which can result in some hard-to-detect bugs

One can propose changing the original name to `rawData`, but this can lead to confusion from the end user of the function.

Argument labels can serve as an excellent approach to handle this problem:

```kotlin
fun doHeavyLifting(data rawData: Data) {
    val data: Data = doSomePreprocessing(rawData)
    // do other stuff using a short and simple name
}
```

Now, with `data` as an argument label, the end user does not see any `raw` prefix, which could lead to confusion, but the developer sees it and can use the identifier `data` for further use as an argument label do not go into the function body scope.

One can say that it can also be solved by introducing mutable arguments, but further discussion is required on whether it is an acceptable pattern to implement.

### Possible drawbacks

Along with the possible benefits, one may argue that this feature should not be introduced into Kotlin. Even though it was not the purpose of this document to elaborate on these points, there are still some worth noting.

#### Existance of type aliases

One may say that argument labels can be easily replaced by the rich type system of Kotlin (and using type aliases). They are already present in the language, they can pass additional information, and one can even make some kind of a "parameter name overload" by using different type aliases.

The counterargument to this point is that it can turn into creating a new type for every place instead of using primitives, especially in cases where there is no additional logic or limitations on the value passed. Further thoughts can be found in this [reply on the forum](https://discuss.kotlinlang.org/t/kotlin-internal-and-external-parameter-name-propose/7906/12).

#### Increased confusion in function declaration

With the increased size of function declarations, it may be harder to read them. Two identifiers, a modifier keyword and a type, are already four "words" for just one parameter. Furthermore, it can also be too easy to mix up the argument label and the parameter name.

Apart from that, having different names in the function body and call place can actually be confusing for somebody reading the call and then jumping inside the function body just to find that the argument they were looking at is called something completely different, while the only place with the link between these two is the function declaration.

However, if one is going to read the function body, one will have to read the function declaration, at least to know the argument types, for example.

#### Meaningless labels

Even though it is good that calls may read like language sentences, by further look, it becomes difficult to parse the meaning behind the argument with names like `by` or `using`. The type can become the source of information at that point, but then the purpose of argument labels is lost. The complete discussion starts at [this reply](https://discuss.kotlinlang.org/t/internal-function-parameter-name/17634/5).

One can say, in response, that the meaning of such arguments can be understood from the context, such as names of values passed, other arguments, name of the function and others. In addition, it is up to the user to give names that make sense. 

#### Remark: perhaps we should change the approach to arguments (Argument Objects)?

There is an idea to use argument objects for situations where many arguments are passed in function. The argument object in this situation is a simple data class, which contains some arguments of the original function (configuration, for example) and can easily (with some special syntax) be passed in the function call as well as unpacked on the function side without significant overload.

It is stated that such an idea could provide ABI compatibility for a function with the ability to change the names and order of arguments by having them in the argument object.

More discussions about this can be found in documents related to the Argument Object feature.

## Ways to implement

After a basic introduction and the vision behind why this feature could be implemented (or not), we should discuss what exactly needs to be implemented, what things need to be considered, and how the feature can be implemented.

### The formal task

By "supporting argument labels feature", we will denote the presence of the following changes to the Kotlin compiler:

1. The ability and syntax to have two identifiers for an argument in functions, methods and constructors declarations. One of these identifiers will serve as an **argument label**, an external name, and the other will serve as a **parameter name**, an internal name. If only one identifier is specified, it should serve as both.
2. The parameter name should be added to the scope of the function body, with the corresponding behaviour being similar to the existing argument names. This parameter name should not be visible outside the function body.
3. The argument label should be used during the mapping of function call arguments to the parameters. The labels should not be visible inside the function body. It still has to be possible to use a different order of named arguments from the one they are specified in the declaration.

#### Remark: what about lambdas?

We have not mentioned lambdas in the first point. Why?

Turns out, it is currently impossible to use named form of arguments for calls of functional types (such as lambdas). Therefore, if the named form is impossible, adding argument labels for lambdas is meaningless.

Another approach would be to allow the named form for lambdas, but it will require dealing with a more complex parsing stage due to the destructuring declaration.

#### Remark: functions, methods and constructors?

Perhaps one may want to include argument labels only to a subset of those mentioned here. We have not found any specific reasons to ignore any of those, except constructors, as the introduction of argument labels may lead to slight confusion on what will count as a class member and what will be used only in the constructor call in named form.

### Things to consider

#### Argument Label Syntax

To begin with, we can look at the Swift syntax for argument labels, as it is the most widespread language with this feature supported:

```swift
func greet(person: String, from hometown: String) -> String {
    return "Hello \(person)!  Glad you could visit from \(hometown)."
}
print(greet(person: "Bill", from: "Cupertino"))
```

Here, the argument label `from` is introduced by adding an identifier before the parameter name `hometown`. Look at how the identifier `person` serves both as an argument label and a parameter name.

In the past, some argued that the Swift approach is not possible to use in Kotlin, as there are already parameter modifiers in this place:

```kotlin
send(vararg message: String)
send(noinline message: () -> String)
send(crossinline message: () -> String)
```

However, it appears that those three are the only parameter modifiers present and are parameter keywords (or weak keywords), meaning that they are keywords when applicable, and if not, they are identifiers. That allows us to use the same syntax for Argument labels in Kotlin as in Swift:

```kotlin
fun send(message: String,
         withDelayBeforeSending delay: Delay = Delay.DEFAULT_DELAY) { //... }
```

Even though there were several other approaches proposed, one using the `as` keyword to specify the parameter name (with the argument label taking place before the type specification):

```kotlin
fun send(message: String,
         withDelayBeforeSending: Delay = Delay.DEFAULT_DELAY as delay) { //... }
```

However, there are several problems that have to be solved for this syntax: Where do we add modifiers to such value parameters? Before `withDelayBeforeSending` or before `delay`? If we use it in primary constructors, where do we place `var` and `val`?

The other possible syntax approach uses brackets for specifying argument labels while placing labels in the same place as in Swift:

```kotlin
fun send(message: String,
                 port: ValidPort,
                 securityType: SecurityType,
                 [withDelayBeforeSending] delay: Delay = Delay.DEFAULT_DELAY) { //... }
```

It may be worth concluding a poll or research on which syntax will be more understandable to users, as we have noticed different opinions on this question, regardless of the Swift option being the only one already existing in programming languages.

#### Unnamed parameters

We have already mentioned the unnamed parameters in the benefits section. Other languages seem to agree on this question: Rust, Swift and Python all use `_` as a special name, marking the parameter as "unused" or "unnamed".

The case with labels can be solved just like that --- one marks the label as `_`, and we then prohibit using it for named arguments (during the argument-parameter mapping, for example).

If we want to mark the internal parameter names as unused or empty, one will have to probably work on preventing them from going into the scope completely. However, one can still freely specify a proper argument label and specify the parameter name as "empty".

#### Argument labels and inheritance

Another point worth noting is how argument labels should behave with inheritance. Should they be inherited and kept the same on overrides, or could the user freely change them? Should an argument "inherit" the label without restating it in the function declaration (for example, when we want to specify a different parameter name)? What about the other way (specifying only the argument label, with the parameter name being inherited from the ancestor)?

There was not much discussion on these questions, but a few points could be made when looking at these questions from the point of already discussed possible benefits and issues:

1. It seems beneficial in most cases to insist on keeping the same argument labels during inheritance while allowing the user to change the parameter names for better fitting to the internal context of the function body.
2. Perhaps it could be helpful to inherit the label from the ancestor if we require it to be the same across all ancestors-inheritors. However, how should it be marked? Currently, the position in syntax is that "if only one identifier is specified, it should serve as both an argument label and a parameter name". 
3. Maybe point 1. should be changed in regard to the special, "empty" names or in situations when the call through the original "super-class" is unadvised or prohibited for some reason. Perhaps we want to prohibit using a parameter in the named form in the original function while allowing such behaviour in some of its inheritors.

#### Remark: set of questions from the original issue
- how to specify that a parameter doesn't have an external name and cannot be provided in a named form (Swift: `func foo(_ parameterName)`)
- how to specify that a parameter does have a different external name, but doesn't have to be provided in a named form only
- how to require a parameter to be provided in a named form only without duplicating its external name
- how to make a parameter unnamed internally ([KT-8112](https://youtrack.jetbrains.com/issue/KT-8112/Provide-syntax-for-nameless-parameters)) without spelling its external name if it can be inferred from e.g. a supertype method.

The first is discussed in the part about unnamed arguments; the second and third are solved by separating argument labels and enforcing named form into two separate features. The fourth one is harder, but some discussions were made in the "Argument labels and inheritance" part.

### Existing solutions

Before moving to the possible implementation details of our own, we should first try to look at how these features are emulated in Kotlin if they are, and how they are implemented in other programming languages.

#### Imitations in Kotlin

Surprisingly, there is no solution for introducing argument labels in Kotlin present. Perhaps some are providing two functions, one (the external) with the argument labels and with the sole purpose of passing the arguments into the internal function with the parameter name, even though this pattern is not widespread by any means.

#### About named form in other languages

Currently, several popular programming languages do not even have the regular named form for passing arguments in a function call, like Java or C++. Still, some do have, and among them, a few support argument labels.

#### Approach in Swift

We have already seen how Swift supports argument labels from the point of syntax, but let us restate it once more:

```swift
func greet(person: String, from town: String) -> String {
    return "Hi \(person) from \(town)."
}
print(greet(person: "Bill", from: "Cupertino"))
```

As one can see, in Swift, it is possible to specify one identifier, which will serve as both an argument label and a parameter name. However, it is also possible to specify them separately. 

Two important things to note are that the named form in Swift is enforced by default, but despite this, the arguments in function invocation have to be specified in the exact order as in the declaration.

Another thing about the inheritance (or, in this case, implementation of a "protocol") can be seen in the following example:

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

This example of code does not compile, producing the following error:

```swift
error: instance method 'consume(message:)' has different argument labels from those required by protocol 'Consumer' ('consume(input:)')
```

That means instances, overrides, and implementations must have the same argument labels as their ancestors. However, only the argument labels have to be the same; parameter names can be changed freely, as shown by the following example:

```swift
protocol Consumer {
    func consume(input: String)
}

class MessageConsumer: Consumer {
    func consume(input message: String) {
        print("Got message: " + message)
    }
}
```

On the implementation side, argument labels are implemented as a proper second name for an argument. The field is present along with the parameter name in the corresponding nodes in the syntax tree. It is being checked in the function calls resolution to match the one used in the call. However, due to the arguments having to match the order of the declaration, no intellectual resolution and mapping is being done there.

#### Argument Labels in Gleam

Another language related to our investigation is [Gleam](https://gleam.run/), a new programming language from the creators of Rust. 

Even though it does not force named form of arguments of any kind, it does introduce labelled arguments.

Despite having Rust-like syntax and being compiled into Erlang and Javascript, argument labels are supported in Swift likeness, as one can see on the following listing:

```gleam
pub fn main() {
  // Without using labels
  io.debug(calculate(1, 2, 3))

  // Using the labels
  io.debug(calculate(1, add: 2, multiply: 3))

  // Using the labels in a different order
  io.debug(calculate(1, multiply: 3, add: 2))
}

fn calculate(value: Int, add addend: Int, multiply multiplier: Int) {
  value * multiplier + addend
}
```

There are two significant differences from the Swift, however:

1. The labelled arguments can be used in any order, with the only limitation being that the labelled arguments have to be after the positional ones.
2. If a label is not specified for an argument, then it can only be used in the positional form. 

To understand the second part more, here is the call of the `calculate` function, which does not compile:

```gleam
calculate(value: 1, add: 2, multiply: 3)
```

It produces the following error:

```
Unexpected label: You have already supplied all the labelled arguments that this constructor accepts.
```

Even though the 1. is something we already have in Kotlin, the approach by 2. does not seem feasible in our situation, as we have to support the old Kotlin syntax, and if we are to adopt this part of Gleam likeness, all previous named form usages would be rendered invalid.

### Specific implementation ideas

Now, after we move through the existing implementation, we need to think about which we can use for prototype or implementation, regarding their benefits and drawbacks.

#### Syntactic sugar: jumper function

One of the more primitive approaches is to implement this feature as a syntactic sugar via splitting the initial function with argument labels into two.

First, we do the parsing as usual, but with allowance for two identifiers for a parameter by a slight change in the related function. Second, when transforming the parsed code source tree (PSI or lightTree) into FIR, we check whether a function has argument labels in any of its arguments. And then, if those are present, instead of regular processing of such functions, we generate two new functions as follows:

1. The first one will be denoted as the external one. It will have the name of the original function, and the parameter names will be set to the function's argument labels, but the body will be replaced with the call to the internal function, with all the received parameters being simply passed to the internal call

2. The second one will be denoted as the internal one. It will be the original name with some internal suffix added, hiding it from the user. Its parameters will be the parameter names of the original function, and its body will be the body of the original function.

These functions will be generated during the desugaring stage, and then the compilation will follow as usual. In this case, the original function will not be generated (therefore, there will be only two new functions, the internal and external ones). If a function does not contain argument labels, its transformation will not be affected in any way.

The main disadvantage of this approach is that it creates additional functions, therefore decreasing both the compilation and execution times, with the increased size of the resulting artifact. 

The advantage is that this approach can be much easier to implement than the first because it is a syntactic sugar. Apart from that, by generating the functions in the early stages, we do not have to worry about platform compatibility, as the backend receives these functions as regular functions like they were written by a user, despite having the new feature in the syntax.

There are more additional things to worry about in this approach, however: 

1. Variadic arguments must be handled specially (add a spread operator), increasing the overhead.
2. For each method in a class, an additional method will be generated, so we will have to keep track of all the method modifiers.
3. "Internal" methods will still be accessible through reflections
4. Primary constructors will also require additional handling; for example, the primary constructor could just delegate the behaviour to the secondary. Still, that would require additional thoughts.

#### New field for argument label

As an argument label is, essentially, just an additional name for a function argument, it naturally comes that one of the possible ways to implement this feature is to simply add this new field to related structures, just like it is done in the Swift programming language. All structures holding the `ValueParameter` on various levels, from parser and PSI to FIR and, probably, IR, will be affected by this addition. Apart from that, some additional changes would need to be made during the resolution stages (specifically, argument-parameter mapping), as the new names will have to be considered during the call using either the named form or inside the function body.

The main disadvantage of this approach is that it requires adding changes to many different classes and, in the worst-case scenario, checking every existing creation of related classes in the vast code base of the Kotlin compiler, along with passing the new name down to platform-dependent code. Such work can be overwhelming for the prototype to implement, and the required work is not easy to estimate.

Nevertheless, on the side of advantages, this method is expected to be pretty robust, as it just adds a new name without any additional duplication. Apart from that, it should work in many different cases initially not discussed but logically related, such as class constructors and methods.

### Possible technical details

This section collects different details and moments that should be noted during the implementation.

#### How to split argument name?

If we are introducing a new identifier, we need to decide what we do with the old one and what --- with the new one. There, naturally, are two possible choices:

1. Leave the existing identifier as a parameter name and use the new one as an argument label. 

2. Make the new identifier a parameter name and leave the old one to be just an argument label.

In the first case, we will have to change the places related to the arguments to parameter mapping and, possibly, overload resolution (more about it in the following part). Apart from that, we will have to understand how to put the new name in the compilation artifacts to allow separate compilation of files (again, more about it later).

In the second case, however, we must change which names are being passed into various scopes (mainly -- function body scope).

We also need to think about which name will become the member if used as a class's primary constructor. Still, it makes sense to declare that the parameter name is a member of the class when the argument labels are used only in the constructor call.

#### Separated compilation

If we want to use argument labels in a library, which will be compiled and used in a different project, we will have to pass the argument labels to the compilation artifacts.

Currently, Kotlin already passes argument names of parameters (at least to .class files or metadata). However, if we are to introduce argument names as a new identifier, we will have to somehow add it into IR and serialization/deserialization processes.

However, the "sugar" approach or making the new identifier a parameter name can help avoid this work.

#### Interoperability with Java

Kotlin has to support interoperability with Java, which means Kotlin functions can be called from the Java code and vice versa.

How is this related to our work? Not much, as Java does not have any named argument form. All the arguments can only be passed in positional form, which is probably because Java compilers do not always have to save the argument names to the compiled artifacts. The latter also restricts using named form of Java functions from Kotlin code.

#### Not-JVM backends

We have not deeply discussed or looked at other backends for Kotlin apart from the JVM one. Still, if the feature is implemented on the sugar level, there should not be such a problem. Still, something can probably be thought about the Kotlin/Native platform, especially since it is indeed close to Swift, which has the feature in question.

#### Inheritance

We need to understand how the labels are preserved during the interface implementation/function overrides. How can different approaches and choices affect the results? Do we enforce or recommend having the same argument labels or parameter names?

#### Overload resolution

Related to the previous problem, argument names are used in the candidate resolution in some cases. Should we change this behaviour so that argument labels are used in this process? Should we introduce any other changes to it?

#### Related diagnostics

The end user of the compiler should be provided with meaningful error messages in various new cases related to using argument labels. Do we allow non-unique argument labels? Probably not. Same parameter names? Not. Intersections between argument labels and parameter names? Why not?

Should we warn the user when it tries to use the parameter name in the named argument call or directly throw an error? Probably the latter; otherwise, we must prohibit intersections between labels and names and/or enforce the same order as in the declaration.

What about using labels instead of parameter names inside the function body?

## Evaluation

To test some the ideas described, collect additional findings and have a starting point for possible further implementations, the prototypes for the argument labels feature were developed.

A prototype here is a version of a Kotlin compiler, with modifications made to support the new feature. The feature can possibly be covered by tests and different benchmarks in the prototype, but still have poor code quality and/or questionable design choices. Apart from that, it may not work or be untested for some specific cases, which are mostly noted in the corresponding part.

### Prototypes implemented

Due to initial lack of experience working with Kotlin compiler source code, initially the syntactic sugar prototype was implemented. However, some time after, a more "proper" prototype, with a new field added, was created separately. Here we will discuss the implementation details of both of them.

#### Via jumper function (sugar)

For the parser part, the needed change was to modify the `parseFunctionParameterRest function, which, as one can see from the name, parses the function parameter after parsing its modifiers. The change needed was to add the case of two identifiers instead of one, and if so, parse them. After this, a parameter with an argument label will be represented by having two consecutive identifiers instead of one. 

We decided to use the Swift syntax for argument labels for now, but it is possible to change it to use other variants proposed in the corresponding part of this document with changes only in parser and the translation into FIR part.

For the desugaring we added (an unefficient) function to check the presence of argument labels in a function, and then added separate parameter to the transformation function, which controlls, whether to generate a jumper body and use argument labels, or to proceed with regular body, but change the name and use parameter names.

After that no additional changes were made, as that was enough for basic tests to work.

#### Via proper field

For the role of the new identifier, we settled on the argument label, as it seems as the one requiring less changes to the already existing code and a faster path to a seemingly working prototype.

The parser part was changed the same way as in the "sugar" implementation.

After the work in parser, a new field was added to the FIR nodes via the generator and configurator of them. This field was then initialized by the mew identifier during the transformation of ValueParameter lightTree/PSI node to FIR, or by the parameter name in case the valueParameter lacked an argument label.

After that, the process of argument to parameter mapping in the resolution stage (`FirArgumentsToParameterMapper.kt`) was modified. The internal mapping of names to the function parameters was changed to map argument labels instead, and a check was added to the situation, when someone is using the parameter name instead of the argument label for an argument (via an additional mapping).

Lastly, several diagnostics were added, regarding the improper usage of argument labels or some of the argument labels being not unique.

### Implementation results

The two described prototypes can be found in the following branches of the Github repository:

1. [Prototype using jumper function (sugar)](https://github.com/MarkTheHopeful/kotlin/tree/argument-label-proto)
2. [Prorotype using additional field in structures](https://github.com/MarkTheHopeful/kotlin/tree/argument-label-proto-2)

Both can be used in the following way:

1. Checkout the needed branch
2. Run `./gradlew dist`
3. Use the compiler from `./dist/kotlinc/bin/kotlinc "filename"` to compile the file using the prototype

Now we should move to the evaluation of the prototypes.

#### Tests and behaviour

For the first prototype, the main accent was to make it work at least on simple examples, and it, in fact, did work on them. Top-level functions with argument labels worked correctly, with an exception for functions with variadic arguments, as the needed treatment was not added. Interaction with parameter modifiers (`noinline, crossinline`) was fine, no additional problems were noted. Some basic tests were introduced to the test sets (`fir/analysis-tests/testData/resolve/argumentLabel`). The already present tests without the usage of argument labels were not affected. Additional attention to this prototype was not given. 

For the second prototype, more in-depth testing was concluded, although the commited test set wasn't changed much yet. (Most of the files are still present as local):
1. Regular behaviour on top level functions, with some arguments having parameter modifiers has been checked, and everything (including variadic arguments) worked correctly.
2. Regular behaviour on class methods was checked, with methods having different visibility and various parameter modifiers. It also worked correctly and consistently.
3. The behaviour was also checked for contructors and operators (such as `invoke`), where the proper and expected results were achieved. The parameter names served as members (fields) in case of contrustors.
4. For class methods, situations were a method of a descender overrides one in an ancestor. The override worked correctly (in the same way as without argument labels), although the warning was not present if different argument labels were used. No problems were encountered when different parameter names were used, as the mapping is now being done by argument labels.
5. For non-unique argument labels (two or more parameters with the same argument label), an addition to already existing diagnostic on function parameter names was made (simply check the argument labels separately). The diagnostic worked and produced correct error message.
6. Additional attention was given to trailing lambdas and their usage in named form, especially in argument labels. It worked fine, just as regular arguments.
7. Lastly, it was attempted to try and compile the function and its call separately, to check, whether it will (somehow) work. No miracles, it failed, as currently argument labels are not serialized and anyhow put even into the IR.

#### Benchmarks

Up to this moment, only the first prototype was properly benchmarked, and the result was a significant (10-25%) increase in compilation time on tests with an increased amount of functions with many arguments with argument labels.

The benchmarks included tests with many (hundreds to thousands) functions with small or large (hundreds) amount of arguments and many function calls. Every test has a version without argument labels and with argument labels.

The prototype shown no significant decrease in performance on tests without argument labels.

#### Existing problems

There are several problems with the prototypes made, both from the technical and from the design sides. We have mostly described them in the previous parts, but here they all are gathered in one place.

1. The first prototype does not work, if a function under transformation contains variadic arguments.
2. The first prototype is currently not designed to support constructors and methods in classes
3. The second prototype does not support separate compilation of a function declaration and its calls.
4. The error messages in the second prototype are not precisely correct with locations in some cases. 

### Further work

Even though this document is a complete report on the research of argument labels, there are still directions to move before the final decision and, if positive, implementation of the feature into the Kotlin programming language. 

The three main directions can be summarized in the following list:
1. Make further research on specific design choices, ranging from specific syntax used for argument labels to the ways to store them in compilation artifacts. The most straightforward action in this direction will be to conclude a poll on the syntax choice. Apart from that, one can dive deeply into specifics of implementation of argument labels in Swift or Gleam, as the first is being compiled into LLVM, and the second --- to JavaScript, which can be useful for Kotlin/Native and Kotlin/JS respectively
2. Add further improvements to the second prototype. Separate compilation problem can be resolved, small bugs with warnings not showing on can be fixed, additional tests and benchmarks can be done. One can also try writing it with making the new identifier play the role of parameter name instead of argument label.
3. Develop the first prototype to the level of the second one. Despite it being created just to make something work and wasn't tested properly, maybe the idea has more behind it that it seems, if polished a little bit more and optimized. The jumper function can be inlined, the check can be replaced with a flag somewhere during the compilation and so on. And it does not require any further meddling with IR and compilation artifacts.

## Final results

During the work on this issue, many insights and information were gathered from the issues, discussions and other documents, as well as some additional research was concluded, including existing implementations in other languages. Different possible use-cases, benefits and drawbacks were discussed, possible ways of implementations were analyzed, different peculiarities were discovered and recorded. Finally, two prototypes were implemented as to prove the concept and allowing for further experiments and research regarding the feature.

## Additional remarks

Some remarks are not directly related to any of the parts in this document, but still are related to argument labels. Those are described in this part.

### On Swift regarding the argument labels

One may notice, that in Swift texts, for example in the documentation, the functions are being referenced not just by their names, but by their parameter names: "Because the function returns a String value, greet(person:) can be wrapped in a call to the print(\_:separator:terminator:) function"

Considering the fact that overloads by argument labels are possible in Swift, and that for most cases you have to specify the argument labels in a function call, it makes sense to say, that argument labels are the actual part of the function name (of the function _signature_), just being split in multiple words, separated by the values for arguments. 

It can actually stem from the same behaviour for methods in the Objective-C language, with each argument starting with the second is recommended to have a "joining Argument".

### On aliases for functions and arguments

Maybe at some point you would like to rename all arguments of a function, but just for one file, to do something like the following:

```kotlin
fun f(a1: Int, a2: Int, a3: Int) {} //...

alias g(b1, b2, b3) = f(a1, a2, a3) // now g with arguments b1, b2, b3 is the same as f
alias h(c1) = f(a1) // same as h(c1, a2, a3) = f(a1, a2, a3)
alias f(d1, d2, d3) = f(a1, a2, a3) // now f has arguments named d1, d2, d3
alias g(e2) = g(b2) // now g has arguments b1, e2, b3, and still just the call to f
```

Could any of these be implemented? Why would someone need those? Can we do something like "local renaming of an argument"?
