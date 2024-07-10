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


#### Interface implementation

There is a problem related to the implementation of generic interfaces for more specific cases: usually, it would be more meaningful to use different names, as can be seen in the following example from the related issue: [KT-59531](https://youtrack.jetbrains.com/issue/KT-59531/Add-a-way-to-make-parameter-names-of-interface-functions-non-stable)

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

Another example of this problem:

Suppose we have an interface:

```kotlin
@FunctionalInterface
interface Fn2<A, B, R> : BiFunction<A, B, R>, (A, B) -> R {
    @JvmDefault
    override operator fun invoke(p1: A, p2: B): R {
        ...
```

And then there is an implementation

And then there is an implementation

```kotlin
object: Fn2<Int,Int,Int> {
    override fun invokeEx(accum: Int, i: Int): Int =
    accum + i
}
```

It is meaningless to use old names in a new context, so we introduced new ones and got a warning:  `Warning:(598, 76) Kotlin: The corresponding parameter in the supertype 'Fn2' is named 'a'. This may cause problems when calling this function with named arguments.`

Maybe there is some way to do something with it using argument labels? Special labels, or making it possible to change names of outer/inner labels during overwriting? Or maybe, as it was suggested in the related [stackoverflow question](https://stackoverflow.com/questions/50672203/kotlin-explicitly-unnamed-function-arguments), use an unnamed parameter in the interface definition? 

Another related example of it can be seen in the following issue: [KT-9872](https://youtrack.jetbrains.com/issue/KT-9872/Disallow-calling-a-method-with-named-argument) 

## Unused arguments

Highly related to the previous topic, [KT-8112](https://youtrack.jetbrains.com/issue/KT-8112/Provide-syntax-for-nameless-parameters), also known as nameless parameters. Can be implemented using empty labels/empty parameter names.

Some wish to have the ability to have different functions with the same name and signature, which differ only in the way the arguments are labelled. As mentioned in issue [KT-43016](https://youtrack.jetbrains.com/issue/KT-43016/Support-method-overloads-on-parameter-names-like-Swift), it states that there is already a way to do this, especially in Kotlin/Native, generating C interoperability stubs.

And it seems that there actually is something like that, at least [in Kotlin specification](https://kotlinlang.org/spec/overload-resolution.html#call-with-named-parameters)

Even `@Suppress("conflicting_overloads")` annotation is present.

## Reserved syntax

Parameter modifiers are already in the same place where Swift has argument labels.

```kotlin
send(vararg message: String)
send(noinline message: () -> String)
send(crossinline message: () -> String)
```

# Ways to implement

Different ideas on where and how these ideas can be implemented, concerning the Kotlin compiler. 

## Argument labels: where and how

Probably not the same way as Swift does, because it will end up colliding with parameter modifiers. One proposed idea is to use `as` as a keyword for it:

```kotlin
fun send(message: String,
                 port: ValidPort,
                 securityType: SecurityType,
                 withDelayBeforeSending: Delay = Delay.DEFAULT_DELAY as delay) { //...
```

Another way is to use brackets:

```kotlin
fun send(message: String,
                 port: ValidPort,
                 securityType: SecurityType,
                 [withDelayBeforeSending] delay: Delay = Delay.DEFAULT_DELAY) { //...
```


### Possible drawbacks

One may say that argument labels can be easily replaced by Kotlin rich type system (and using type aliases)

Counter: I don’t think that it is very useful to create a type for every place where you are going to use a primitive, especially when there are no other logic or limitations on the type. Also, this approach will not work when both parameters are already of some non-primitive type with some logic. Further thoughts: [reply on forum](https://discuss.kotlinlang.org/t/kotlin-internal-and-external-parameter-name-propose/7906/12)

## Solutions

### Existing solutions in Kotlin

### Approaches in other languages

### Possible ways to implement

### Possible technical details

## Evaluation

### Prototypes implemented

### Implementation results

### Existing problems

### Further work

- how to specify that a parameter doesn't have an external name and cannot be provided in a named form (Swift: `func foo(_ parameterName)`)
- how to specify that a parameter does have a different external name, but doesn't have to be provided in a named form only
- how to require a parameter to be provided in a named form only without duplicating its external name
- how to make a parameter unnamed internally ([KT-8112](https://youtrack.jetbrains.com/issue/KT-8112/Provide-syntax-for-nameless-parameters)) without spelling its external name if it can be inferred from e.g. a supertype method.


## Results
