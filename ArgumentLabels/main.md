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

## Ways to implement

After basic introduction and the vision behind why this feature could be implemented (or not), we should move to the discussion about what exactly needs to be implemented, what are the things to be conserned of and how the feature can actually be implemented.

### The formal task

By "supporting argument labels feature" we will denote the presence of the following changes to the Kotlin compiler:

1. The ability and syntax to have two identifiers for an argument in the function, method and constructor declaration. One of these identifiers will serve as an **argument label**, an external name, and the other will serve as a **parameter name**, an internal name. If only one identifier is specified, it should serve as both.
2. The parameter name should be added to the scope of the function body, with the corresponding behaviour being similar to the existing argument names. This parameter name should not be visible from the outside of the function body.
3. The argument label should be used during the mapping of function call arguments to the parameters. The labels should not be visible inside the function body. It is still has to be possible to use different order of named arguments from the one they were specified in the declaration.

### "What if"s

### Things to consider

- how to specify that a parameter doesn't have an external name and cannot be provided in a named form (Swift: `func foo(_ parameterName)`)
- how to specify that a parameter does have a different external name, but doesn't have to be provided in a named form only
- how to require a parameter to be provided in a named form only without duplicating its external name
- how to make a parameter unnamed internally ([KT-8112](https://youtrack.jetbrains.com/issue/KT-8112/Provide-syntax-for-nameless-parameters)) without spelling its external name if it can be inferred from e.g. a supertype method.


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

Parameter modifiers are already in the same place where Swift has argument labels.

```kotlin
send(vararg message: String)
send(noinline message: () -> String)
send(crossinline message: () -> String)
```

### Existing implementations

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

## Results
