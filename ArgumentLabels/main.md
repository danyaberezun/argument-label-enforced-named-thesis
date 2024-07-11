# Argument Labels

## Description

As was stated in the introduction, the main idea of Argument Labels is to separate the identifier, used in the function calls with named arguments (**argument label**) and the identifier, used inside the body of the function (**parameter name**).

Such separation would require several things to be implemented, including:
1. A way two specify two separate identifiers (parameter name and argument label) for an argument in function declaration
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

Firstly discussed in the same issue as Enforced Named Argument Form, it is currently moved to a separate issue: [KT-34895](https://youtrack.jetbrains.com/issue/KT-34895/Internal-and-external-name-for-a-parameter-aka-Argument-Label). 

At that point the idea is to allow developers to specify two names for an argument in their functions: external and internal. The external name acts as a use-site name, the one that is seen during the function call using the named form of arguments, and the internal acts as a parameter name, which is used in the body of a function.

As one can see, the idea is being preserved in this work, with the external names being argument labels, and internal names being parameter names.

There were many further discussions both in the issue and in the posts at [Kotlin Discussions](discuss.kotlinlang.org) revolving around these features. Several findings and points from them will be present in this document.

### Possible benefits

Adding a feature to a programming language, especially a big one, requires much work and can lead to significant changes and increased support and attention from the language development team, therefore it needs a proper motivation and requests from the language community, which is the case in this situation.

Two common reasons between this feature and Enforcing of Named Argument Form are already described in the introduction document, even though we still can briefly mention them here:

1. Introduction of argument labels can make the function calls look like sentences of natural language, which makes the code more self-documenting
2. Changing APIs and libraries under development can benefit from argument labels, as with them one may be able to change the internal name of an argument without the need to change anything in the call sites.

More about these reasons can be read back in the introduction document, while here some more specific reasons will be described.

#### Different meanings for arguments inside and outside

Sometimes the information that a developer needs to know about an argument differs from information that an end user needs to know. Not necessary that some information should be forgotten, but sometimes different sides need to have accents on different properties of the arguments.

One possible example of this situation can be found here, in the situation where it is quite difficult to come up with a meaningful name, that will reflect both sides of the value.

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

It can be seen, that in this example, if we choose to have the parameter named "observeOn", its meaning will be unclear at the point it is being used, but if we choose the name "scheduler", then the purpose of the parameter is being unclear for the one using the function.


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

And indeed, if the names of a parameter in a supertype and subtype differ, it becomes difficult to pass the parameter in the named form. Using the named from the supertype will only be possible when the value is currently in the type of supertype, and same with subtype:

```kotlin
val a: WidgetVisitor<Unit, Context> = WidgetVisitorImpl(Context())
val b: WidgetVisitorImpl = WidgetVisitorImpl(Context())
    
a.visit(Widget(), data=Context())
a.visit(Widget(), context=Context()) // Does not compile
b.visit(Widget(), data=Context()) // Does not compile
b.visit(Widget(), context=Context())
```

The second and the third calls, in fact, produce the "Cannot find a parameter with this name: context/data" error.

This whole situation can be avoided by introducing argument labels: if one uses argument labels, they will only have to keep the argument labels being the same across the sub/supertypes, while being able to have different parameter names, depending on the implementation:

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

This way, there won't be any problems with calls using named argument form, as they both will use the `data` label, but in the implementation it will be possible to use `context` instead of meaningless in the new context name `data`. 

Remark: this is not the only possible way to solve this problem. There were several proposed, like [having an special name for unnamed arguments](https://stackoverflow.com/questions/50672203/kotlin-explicitly-unnamed-function-arguments) or [disallowing named form for some arguments](https://youtrack.jetbrains.com/issue/KT-9872/Disallow-calling-a-method-with-named-argument), which shows that the problem itself is something people encounter during their work.

#### Unused arguments

Highly related to the previous topic, [some ask for syntax for nameless parameters](https://youtrack.jetbrains.com/issue/KT-8112/Provide-syntax-for-nameless-parameters). In the original issue, the motivation is to use "empty" name for unused parameters, catch statemets and situations, where a function accepts a function with specific amount of parameters, as seen in the example in [issue KTIJ-10594](https://youtrack.jetbrains.com/issue/KTIJ-10594).

With argument labels introduced, empty names can be implemented as either empty (that means, set to a special value, like `_` in Rust, for example) argument labels, if one wants to disallow using named form for a parameter (same method, as in Swift), or empty parameter name, to indicate that it is not being used inside the function body. 

#### Overload by parameter names

Some wish to have the ability to have different functions with the same name and signature, which differ only in the way the arguments are labelled. As mentioned in the issue [KT-43016](https://youtrack.jetbrains.com/issue/KT-43016/Support-method-overloads-on-parameter-names-like-Swift), Swift does have such overload, and, when compiling to Kotlin/Native on iOS with specific annotations, it does work. However, one may think that it is strange to have a feature specific to only one platform. 

Argument labels, again, can be used to provide such overload, if included in the resolution process. However, it may be quite difficult to implement from the point of Java Virtual Machine.

### Possible drawbacks

Along with the possible benefits, one may argue that such feature should not be introduced into Kotlin. Even though it was not a purpose of this document to elaborate on these points, there are still some worth noting.

#### Existance of type aliases

One may say that argument labels can be easily replaced by Kotlin rich type system (and using type aliases). They are already present in the language, they can pass additional information, and one can even make some kind of a "parameter name overload" by using different type aliases.

The counterargument to this point is that it can turn into creating a new type for every place instead of using primitives, especially in cases where there is no additional logic or limitations on the value passed. Further thoughts can be found in this [reply on forum](https://discuss.kotlinlang.org/t/kotlin-internal-and-external-parameter-name-propose/7906/12).

#### Increased confusion in function declaration

With the increased size of function declaration it may be harder to read them. Two identifiers, modifier keyword and a type is already four "words" for just one parameter. And it also can be too easy to mix up which is the argument label and which is the parameter name.

Apart from that, having different names in the function body and in call place can actually be confusing for somebody reading the call and then jumping inside the function body just to find that the argument they were looking at is called something completely different, and the only place with the link between these two is the function declaration.

However, if one is going to read function body, they will have to read the function declaration, at least to know what the argument types are, for example.

#### Meaningless labels

Even though it is good that calls may read like language sentences, by further look it becomes difficult to parse the meaning behind the argument with name like `by` or `using`. The type can become the source of information at that point, but then the purpose of argument labels is lost. Full discussion can be found starting at [this reply](https://discuss.kotlinlang.org/t/internal-function-parameter-name/17634/5).

One can say in response, that the meaning of such arguments can be understood from the context, such as names of values passed, other arguments, name of the function and other. In addition, it is up to user to give names that makes sense. 

#### Remark: perhaps we should change the approach to arguments (Argument Objects)?

There is an idea to use argument objects for situations, where many arguments are passed in function. Argument object in this situation is a simple data class, which contains some arguments of the original function (configuration, for example), and can easily (with some special syntax) be passed in the function call as well as unpacked on the function side without significant overload.

It is stated that such idea could provide the ABI compatibility for a function with the ability to change the names and order of arguments, by having them in the argument object.

More discussions on this point can be found in one of the related documents...

## Ways to implement

After basic introduction and the vision behind why this feature could be implemented (or not), we should move to the discussion about what exactly needs to be implemented, what are the things to be conserned of and how the feature can actually be implemented.

### The formal task

By "supporting argument labels feature" we will denote the presence of the following changes to the Kotlin compiler:

1. The ability and syntax to have two identifiers for an argument in the function, method and constructor declaration. One of these identifiers will serve as an **argument label**, an external name, and the other will serve as a **parameter name**, an internal name. If only one identifier is specified, it should serve as both.
2. The parameter name should be added to the scope of the function body, with the corresponding behaviour being similar to the existing argument names. This parameter name should not be visible from the outside of the function body.
3. The argument label should be used during the mapping of function call arguments to the parameters. The labels should not be visible inside the function body. It is still has to be possible to use different order of named arguments from the one they were specified in the declaration.

#### Remark: what about lambdas?

We have not mentioned lambdas in the first point. Why?

Turns out, it is currently impossible to use named form of arguments for calls of functional types (such as lambdas). Therefore, if named form is impossible, it is meaningless to add argument labels for lambdas.

Another approach would be to allow named form for lambdas, but it will require dealing with harder parsing stage due to the destructuring declaration.

### Things to consider

#### Argument Label Syntax

To begin with, we can look at the Swift syntax for argument labels, as it is the most widespread language with this feature supported:

```swift
func greet(person: String, from hometown: String) -> String {
    return "Hello \(person)!  Glad you could visit from \(hometown)."
}
print(greet(person: "Bill", from: "Cupertino"))
```

Here, the argument label `from` is introduced by simply adding an identifier before the parameter name `hometown`. Look how the identifier `person` serves as both an argument label and a parameter name.

In the past some argued that the Swift approach is not possible to use in Kotlin, as there are already parameter modifiers in this place:

```kotlin
send(vararg message: String)
send(noinline message: () -> String)
send(crossinline message: () -> String)
```

However, it appears that those three are the only parameter modifiers present, and that they are parameter keywords (or weak keywords), which means that they are keywords when it is applicable, and if not, they are identifiers. That, in fact, allows us to use the same syntax for Argument labels in Kotlin as in Swift:

```kotlin
fun send(message: String,
         withDelayBeforeSending delay: Delay = Delay.DEFAULT_DELAY) { //... }
```

Even though, there were several other approaches proposed, one with using the `as` keyword to specify the parameter name (with argument label taking the place before the type specification):

```kotlin
fun send(message: String,
         withDelayBeforeSending: Delay = Delay.DEFAULT_DELAY as delay) { //... }
```

And the other using brackets for specifying argument labels, while placing them in the same place as in Swift:

```kotlin
fun send(message: String,
                 port: ValidPort,
                 securityType: SecurityType,
                 [withDelayBeforeSending] delay: Delay = Delay.DEFAULT_DELAY) { //... }
```

It may actually be worth to conclude a poll or a research on which syntax will be more understandable to users, as we have noticed different opinions on this question, regardless the Swift option being the only one already existing in programming languages.

#### Unnamed parameters

We have already menioned the unnamed parameters during the benefits part. Other languages seem to agree on this question: Rust, Swift and Python all use `_` as a special name, marking the parameter as "unused" or "unnamed".

The case with labels can be solved just like that --- one marks the label as `_`, and we then prohibit using it for named arguments (during the argument-parameter mapping, for example).

If we want to mark the internal, parameter names as unused or empty, one will have to, probably, work on preventing them from going into the scope completely. But still, one can freely specify a proper argument label and specify the parameter name as "empty".

#### Argument labels and inheritance

Another point worth noting is how argument labels should behave with inheritance. Should they be inherited and kept the same on overrides, or the user could freely change them? Should an argument "inherit" the label without restating it in function declaration (for example, when we want to specify different parameter name)? What about the other way (specifying only argument label, with parameter name being inherited from the ancestor)?

There was not much discussion on these questions, but a few point could be made, when looking at these questions from the point of already discussed possible benefits and issues:

1. It seems beneficial in most cases to insist on keeping the same argument labels during inheritance while allowing the user to change the parameter names for the purpose of better fitting to the internal context of the function body.
2. Perhaps it could be useful to actually inherit the label from the ancestor, if we are to require it being the same across all ancestors-inheritors. However, how should it be marked? Currently the position on this in syntax that "if only one identifier is specified, it should serve as both an argument label and a parameter name". 
3. Maybe the point 1. should be changed in regard of the special, "empty" names, or in situations, when the call through the original, "super-class" is unadvised or prohibited for some reason. Perhaps we want to prohibit using a parameter in named form in the original function, while allowing such behaviour in some of its inheritors.


#### Remark: set of questions from the original issue
- how to specify that a parameter doesn't have an external name and cannot be provided in a named form (Swift: `func foo(_ parameterName)`)
- how to specify that a parameter does have a different external name, but doesn't have to be provided in a named form only
- how to require a parameter to be provided in a named form only without duplicating its external name
- how to make a parameter unnamed internally ([KT-8112](https://youtrack.jetbrains.com/issue/KT-8112/Provide-syntax-for-nameless-parameters)) without spelling its external name if it can be inferred from e.g. a supertype method.

The first one is discussed in the part about unnamed arguments, the second and third are solved by actual separation of argument labels and enforced named form into two separate features. The fourth one is a little bit harder, but some discussions on the point were made in the "Argument labels and inheritance" part.

### Existing solutions

Before moving to the possible implementation details of our own, we should first try to look at how these features are emulated in Kotlin, if they are, and how they are implemented in other programming languages.

#### Imitations in Kotlin

Surprisingly, there is no actual solution for introducing argument labels in Kotlin present. Perhaps some are providing two functions, one (the external) with the argument labels, and with the sole purpose of it to pass the arguments into the internal function, with the parameter name. Even though, such pattern is not widespread by any means.

#### About named form in other languages

Currently some of the popular programming languages do not even have the regular named form for passing arguments in a function call, like Java or C++. Still there are some which do have, and among them there is a few that have support for argument labels.

#### Approach in Swift

We have already seen how Swift supports argument labels from the point of syntax, but let's restate it once more:

```swift
func greet(person: String, from town: String) -> String {
    return "Hi \(person) from \(town)."
}
print(greet(person: "Bill", from: "Cupertino"))
```

As one can see, in Swift it is possible to specify one identifier, which in this case will serve as both argument label and parameter name, but it is also possible to specify them separately. 

Two important thing to notice is that the named form in Swift is enforced by default, but despite this, the arguments in function invokation have to be specified in the exact order as in the declaration.

Another thing in relation to the inheritance (or, in this case, implementation of a "protocol") can be seen from the following example:

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

This example of code does not compile due to the following error:

```swift
error: instance method 'consume(message:)' has different argument labels from those required by protocol 'Consumer' ('consume(input:)')
```

That means, instances, overrides and implementations should have the same argument labels as their ancestors. However, only the argument labels have to be the same, parameter names can be changed freely, as shown by the following example:

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

On the implementation side, argument labels are implemented as a proper second name for an argument. The field is present along with the parameter name in the corresponding nodes in syntax tree. It is being checked in the function calls resolution to match the one used in the call, but, due to the arguments having to match the order of the declaration, no intellectual resolution and mapping is being done there.

#### Argument Labels in Gleam

Another language related to our investigation is [Gleam](https://gleam.run/), a new programming language from the creators of Rust. 

Even though it does not forces named form of arguments in any kind, it does introduce labelled arguments.

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

1. The labbeled arguments can be used in any order, with the only limitation is that the labelled arguments have to be after the positional ones.
2. If a label is not specified for an argument, then it can be only used in positional form. 

To understand the second part more, here is the call of the `calculate` function, which does not compile:

```gleam
calculate(value: 1, add: 2, multiply: 3)
```

It produces the following error:

```
Unexpected label: You have already supplied all the labelled arguments that this constructor accepts.
```

Even though the 1. is something we already have in Kotlin, the approach by 2. does not seem feasible in our situation, as we have to support the old Kotlin syntax, and, if we are to adopt this part of Gleam likeness, all previous named form usages would be rendered invalid.

### Specific implementation ideas

Now after we moved through the existing implementation, we need to think about which can we actually use for prototype or actual implementation, in regard to their benefits and drawbacks.

#### Syntactic sugar: jumper function

One of the more primitive approaches is to implement this feature as a syntactic sugar via splitting the initial function with argument labels into two.

First, we do the parsing as usual, but with allowance for two identifiers for a parameter by a slight change in the related function. Second, during the transformation of the parsed code source tree (PSI or lightTree) into FIR, we check whether a function has argument labels in any of its arguments. And then, is those are present, instead of regular processing of such functions, we generate two new functions as follows:

1. The first one will be denoted as the external one. It will have the name of the original function, and the parameter names will be set to the function's argument labels, but the body will be replaced with the call to the internal function, with all the received parameters being simply passed to the internal call

2. The second one will be denoted as the internal one. Its name will be the original name with some internal suffix added, hiding it from the user. Its parameters will be the parameter names of the original function, and its body will be the body of the original function.

These functions will be generated during the desugaring stage, and then the compilation will follow as usual. The original function will not be generated in this case (therefore, there will be only two new functions, the internal one and the external one). If a function does not contain argument labels, its transformation will not be affected in any way.

The main disadvantage of this approach is that it creates additional functions, therefore decreasing both the compilation and execution times, among with increased size of the resulting artifact. 

The advantage is that being syntactic sugar, this approach can be much easier to implement than the first one. Apart from that, by generating the functions on early stages, we do not have to worry about platform compatibility, as the backend receives these functions as regular functions like they were written by a user, despite having the new feature in the syntax.

There are more additional things to worry about in this approach, however: 

1. Variadic arguments have to be handled in a special way (add a spread operation), which additionaly increases the overhead.
2. For each method in a class, there will be additional method generated, so we will have to keep track of all the method modifiers.
3. "Internal" methods will still be accesible through reflections
4. Primary constructors will also require additional handling, for example, the primary constructor could be just delegating the behaviour to the secondary. Still, that would require additional thoughts.

#### New field for argument label

As an argument label is, essentially, just an additional name for a function argument, it naturally comes that one of the possible ways to implement this feature is to simply add this new field to related structures, just like it is done in the Swift programming language. All structures holding the `ValueParameter` on various levels, from parser and PSI to FIR and, probably, IR, will be affected by this addition. Apart from that, some additional changes would need to be done during the resolution stages (specifically, argument-parameter mapping), as the new names will have to be considered during either the call using the named form or inside the function body.

The main disadvantage of this approach is that it requires adding changes to many different classes and, in the worst-case scenario, checking every existing creation of related classes in the vast code base of the Kotlin compiler, along with passing the new name down to platform-dependent code. Such work can be overwhelming for the first prototype to implement, and the amount of work required is not easy to estimate.

Nevertheless, on the side of advantages, this method is expected to be pretty robust, as it just adds a new name without any additional duplication. Apart from that, it should work in many different cases initially not discussed but logically related, such as class constructors and methods.

### Possible technical details

In this section different details and moments are collected, that should be noted during the implementation.

#### How to split argument name?

If we are introducing a new identifier, we need to decide, what do we do with the old one, and what --- with the new one. There, naturally, are two possible choices:

1. Leave the existing identifier as a parameter name, and use the new one as an argument label. 

2. Make the new identifier a parameter name, and leave the old one being only an argument label.

In the first case we will have to change the places related to the arguments to parameter mapping , and, possibly, overload resolution (more about it in the following part). Apart from that, we will have to understand, how to put the new name in the compilation artifacts to allow separate compilation of files (again, more about it later).

In the second case, however, we will have to change which names are being passed into various scopes (mainly -- function body scope).

We also need to think, which name will become the member, if used in the primary constructor of a class. Still, it makes sense to declare that the parameter name is the member of the class, when the argument labels used only in the constructor call.

#### Separated compilation

If we want to use argument labels in a library, which then will be compiled and used in a different project, we will have to somehow pass the argument labels to the compilation artifacts.

Currently, Kotlin already passes argument names of parameters (at least to .class files or metadata), but if we are to introduce argument names as a new identifier, we will have to somehow add it into IR and serialization/deserialization processes.

However, the sugar-approach or making the new identifier a parameter name can help avoiding this work.

#### Interoperability with Java

Kotlin has to support interoperability with Java, that means, Kotlin functions can be called from the Java code and vice versa.

How this is related to our work? In fact, not much, as Java do not have any named argument form at all. All the arguments can only be passed in positional form, and this is, probably, due to the fact that Java compilers do not have to always save the argument names to the compiled artifacts. The latter also restricts using named form of Java functions from Kotlin code.

#### Not-JVM backends

We have not really discussed or looked at other backends for Kotlin apart form the JVM one. Still, if the feature is implemented on the sugar level, there should not be such problem. Still, something can probably be thought about the Kotlin/Native platform especially, since it is indeed close to Swift, which has the feature in question.

#### Inheritance

We need to keep in mind, how the labels are being preserved during the interface implementation/function overrides. How different approaches and choices can affect the results? Do we enforce or recommend having the same argument labels or parameter names?

#### Overload resolution

Related to the previous problem and the fact, that in some cases argument names are used in the candidate resolution. Should we change this behaviour so that argument labels are used in this process? Should we introduce any other changes to it?

#### Related diagnostics

The end user of the compiler should be provided with meaningful error message in various new cases related to the usage of argument labels. Do we allow same argument labels? Probably not. Same parameter names? Definitely not. Intersections between argument labels and parameter names? Why not?

Should we warn the user when it tries to use parameter name in the named argument call or directly throw an error? Probably the latter, otherwise we must prohibit intersections between labels and names and/or enforce the same order as in the declaration.

What about usage of labels instead of parameter names inside the function body?

## Evaluation

### Prototypes implemented

### Implementation results

### Existing problems

### Further work

## Results

## Remarks

### On Swift regarding the argument labels

One may notice, that in Swift texts, for example in the documentation, the functions are being referenced not just by their names, but by their parameter names: "Because the function returns a String value, greet(person:) can be wrapped in a call to the print(\_:separator:terminator:) function"

Considering the fact that overloads by argument labels are possible in Swift, and that for most cases you have to specify the argument labels in a function call, it makes sense to say, that argument labels are the actual part of the function name (of the function _signature_), just being split in multiple words, separated by the values for arguments. 

It can actually stem from the same behaviour for methods in the Objective-C language, with each argument starting with the second is recommended to have a "joining Argument".
