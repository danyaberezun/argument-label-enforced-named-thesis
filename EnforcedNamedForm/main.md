# Enforced Named Arguments Form

## Description

As was stated in the introduction, by the name form of an argument or a named function call we understand an argument passed in the function call with the name of the argument specified. The main idea of the enforced named arguments form feature is to allow developers mark some of the arguments in a function (or perhaps the whole function) as requiring strictly named form.

This will require the following modifications to be implemented:
1. Add a way for a developer to mark an argument or a function (or a class? or a file?) as requiring named form for the function calls (or for specific arguments in these calls).
2. Transform this mark into a property of an argument, a function or any other entity
3. Verify, ensure or check that if an argument is marked as requiring the named form.

One possible example of such marking can be achieved by introducing a new parameter keyword (here --- `enf`), as can be seen on the following listing:

```kotlin
fun callMe(regular: Int, **enf** enforced: Int) {}

callMe(30, 566) // Compilation error
callMe(30, enforced=566) // Compiles
```

### Discussion history

The discussion firstly recorded in the issue [KT-14934](https://youtrack.jetbrains.com/issue/KT-14934/Enforce-parameter-usage-only-in-named-form) on the Kotlin Youtrack. At some point the discussion about argument labels was included, but soon after creation it was separated into a dedicated issue (more about that in its corresponding document).

In the beginning, the idea was to allow developers to specify for certain parameters that they should be provided only with arguments in named form, prohibiting the positional usage of such arguments.

It was stated, that this behaviour will be useful for the parameters, that are not direct input of a function, but rather some options that affect the function behavior, like Boolean flags (for which there is already an [inspection present](https://youtrack.jetbrains.com/issue/KTIJ-1634), and a request for more sophisticated one [exists for three years now](https://youtrack.jetbrains.com/issue/KTIJ-18880/Consider-more-sophisticated-inspection-for-same-looking-arguments-without-corresponding-parameter-names)).

The initial idea is still present in this work, with some additional ideas to think about, including applying the modifier not only to an argument, but to functions and other units (classes, files...)

There were many further discussions both in the issue (and a few other issues) and in the posts at [Kotlin Discussions](https://discuss.kotlinlang.org/t/add-annotation-or-other-mechanism-to-force-named-arguments-for-method/15636) revolving around these features. Several findings and points from them will be present in this document.

### Possible benefits

Adding a feature to a programming language, especially a big one, requires much work and can lead to significant changes and increased support and attention from the language development team, therefore it needs a proper motivation and requests from the language community, which is the case in this situation.

Two common reasons between this feature and Argument Labels are already described in the introduction document, even though we still can briefly mention them here:

1. Introduction of enforced named form can enforce explicit indication of the meaning of the arguments passed, especially when the arguments are literals or objects, constructed in place. Such direct indication of meaning makes the code more self-documenting
2. Changing APIs and libraries under development can benefit from enforcing named form for the arguments that can be affected by changes in future. When all such arguments are passed strictly in named form, they can freely be reordered on the side of the developer without any changes needed from the end user. Apart from that, new parameters with default values could be added to the function, and it would not break the existing calls regardless of place they were added.

More about these reasons can be read back in the introduction document, while here some more specific reasons will be described.

#### Explicit indication of arguments meaning

Even though this benefit is already partly described in the introduction, it can be useful to restate it once more with more examples and references.

If a function has many parameters of the same type, especially when this type is primitive, and, even more, when the arguments are passed as constants or objects created at the call site, it becomes easy to mix up different parameters, ultimately resulting in hard-to-detect bugs.

Easiest to imagine examples of this kind include functions with boolean flags to configure the function’s behavior, or some various numerical parameters being passed to configure or tune function's behaviour.

**Enforcing named arguments form** can be used to solve this problem. The developer can mark the function or some of its parameters as requiring a named form and then everyone who will use it will have to specify the names of arguments, explicitly indicating the meaning of passed arguments.

The problem is not a new thing, so there are already related solutions as boolean literal inspection (solution to [KTIJ-1634](https://youtrack.jetbrains.com/issue/KTIJ-1634/Add-inspection-for-boolean-literals-passed-without-using-named-parameters-feature)) as well as requests for more in-depth inspections ([KTIJ-18880](https://youtrack.jetbrains.com/issue/KTIJ-18880/Consider-more-sophisticated-inspection-for-same-looking-arguments-without-corresponding-parameter-names))

There is even a section in Coding Conventions about it: “Use the named argument syntax when a method takes multiple parameters of the same primitive type, or for parameters of `Boolean` type, unless the meaning of all parameters is clear from the context.” ([Named arguments](https://kotlinlang.org/docs/coding-conventions.html#named-arguments))

Talking about more specific examples, suppose we have the following function declaration:

```kotlin
fun reformat(
    str: String,
    normalizeCase: Boolean = true,
    upperCaseFirstLetter: Boolean = true,
    divideByCamelHumps: Boolean = false,
    wordSeparator: Char = ' ',
) { /*...*/ }
```

Compare two following ways to call it:

```kotlin
reformat("String!", false, false, true, '_')
// VS
reformat(
    "String!",
    normalizeCase = false,
    upperCaseFirstLetter = false,
    divideByCamelHumps = true,
    wordSeparator = '_'
)
```

Even though the second one looks way more verbose, the meaning of the first one is impossible to understand without looking at the function declaration.

Another couple of examples from real code from a Kotlin machine-learning library, https://github.com/Kotlin/kotlindl:

```kotlin                                                                                                                                                 
Conv2D(
        filters = 16,
        kernelSize = intArrayOf(5, 5),
        strides = intArrayOf(1, 1, 1, 1),
        activation = Activations.Tanh,
        kernelInitializer = GlorotNormal(SEED),
        biasInitializer = Zeros(),
        padding = ConvPadding.SAME
    )
```

```kotlin
model.use {
    it.compile(
        optimizer = Adam(),
        loss = Losses.SOFT_MAX_CROSS_ENTROPY_WITH_LOGITS,
        metric = Metrics.ACCURACY
    )

    it.printSummary()

    it.fit(
        dataset = train,
        epochs = 10,
        batchSize = 100
    )

    val accuracy = it.evaluate(dataset = test, batchSize = 100).metrics[Metrics.ACCURACY]

    println("Accuracy: $accuracy")
    it.save(File("model/my_model"), writingMode = WritingMode.OVERRIDE)
}
```

It is quite difficult to find examples of use of these functions without the usage of named form, but if one were to use it, the meaning of the calls and purposes of the arguments would be impossible to guess. So, to prevent it, it makes perfect sense to mark such functions or these parameters as parameters with enforced named argument form.

#### Remark: some statistics about "bad" function calls
 
According to the data, obtained via different sources (including search by regexp on GitHub), functions with multiple primitive arguments are widely present in the global codebase. Some of the examples include Android Development, various UI functions, utilities to work with images or canvases and more. 

Some of the examples are part [of an Android application, responsible for working with a database](https://github.com/GrzegorzDawidek/AndroidZal/blob/e942947fdd01b3191f472cf378e46c6523a93721/app/src/main/java/com/example/sqllitetask/DatabaseHelper.kt), a [simple image utility](https://github.com/2T-T2/ImageUtil/blob/acc3eb444365caf89e014de9747831b0fec9cbe6/src/ImageUtil.kt) and a [small part of a shopping app](https://github.com/jiangyuanyuan/KotlinMall/blob/0e58f238614a4ba2644712ce4190290c9e19bed0/GoodsCenter/src/main/java/com/kotlin/goods/presenter/GoodsDetailPresenter.kt).
 
While those are definitely not examples of clean and good code, the fact is: that people do write like that, and they do use Kotlin, and, probably, some people even depend on their code. 
 
If we are to move to concrete data, GitHub search by file using regular expression, we can get the following numbers:

| Amount of arguments, at least `x` | Files with at least `x` positional arguments | Files with at least `x` literal arguments |
|-----------------------------------|----------------------------------------------|-------------------------------------------|
| 1 | 11400000 | 8700000 |
| 2 | 7900000 | 2000000 |
| 3 | 3600000 | 553000 |
| 4 | 1800000 | 314000 |
| 5 | 745000 | 151000 |
| 6 | 446000 | 89600 |
| 7 | 267000 | 51500 |
 
This does look like a significant portion of the codebase, and may serve as an argument to introduce the Enforced Named Arguments Form.

### Possible drawbacks

## Solutions

### Existing solutions in Kotlin

## Enforcing named form

This part is dedicated to depicting different currently existing ways to implement or work with enforcing the named form of function arguments, split into parts by ways and languages.

## Enforcing named form: Kotlin improvisation

There is a way in Kotlin to make some arguments of a function be always named, but it uses `vararg nothings: Nothig`, which is forbidden (there is a compilation error, which can be suppressed), and it looks weird and takes space for, in a fact, separator.

```kotlin
/* requires passing all arguments by name */
fun f0(vararg nothings: Nothing, arg0: Int, arg1: Int, arg2: Int) {}
f0(arg0 = 0, arg1 = 1, arg2 = 2)    // compiles with named arguments
//f0(0, 1, 2)                       // doesn't compile without each required named argument

/* requires passing some arguments by name */
fun f1(arg0: Int, vararg nothings: Nothing, arg1: Int, arg2: Int) {}
f1(arg0 = 0, arg1 = 1, arg2 = 2)    // compiles with named arguments
f1(0, arg1 = 1, arg2 = 2)           // compiles without optional named argument
//f1(0, 1, arg2 = 2)                // doesn't compile without each required named argument
```

The example is taken from [stackoverflow](https://stackoverflow.com/questions/37394266/how-can-i-force-calls-to-some-constructors-functions-to-use-named-arguments/37394267#37394267), where the question about the possibility of doing this was initially asked.

This method was discussed in issue [KT-12846](https://youtrack.jetbrains.com/issue/KT-27282/Allow-vararg-parameters-of-type-Nothing), and deemed strange and “looks really like a hack”. The issue was closed in favour of the main, [KT-14934](https://youtrack.jetbrains.com/issue/KT-14934/Enforce-parameter-usage-only-in-named-form).

And, as it can be seen from BigCode query, practically no one uses this approach, at least in the queried codebase:

![Vararg of nothings](VarargNothing.png)

## Enforcing named form: Kotlin annotation

At some point, the idea and need for it arose again, and it was proposed to imitate this behaviour using annotations, which [were requested](https://discuss.kotlinlang.org/t/add-annotation-or-other-mechanism-to-force-named-arguments-for-method/15636/2), and, sometime later, [implemented](https://github.com/chao2zhang/RequireNamedArgument).

```kotlin
@NamedArgsOnly
fun buildSomeInstance(param1: Boolean = true, param2: Boolean = false /* so on */)

// Ok
buildSomeInstance(param1=false, param2=true)

// Compilation error
buildSomeInstance(false, true)
```

The problem with this method is that it is an annotation, a mechanism that is (apparently) unreliable and heavily abused to modify the compiler’s behaviour. Moreover, this approach will not work for libraries that want to require consumers to specify the argument names (and want to provide overloads differing only in the names of the arguments; look to the *overload by argument name* section).


### Approaches in other languages

## Enforcing named form (and Argument label): Swift

In Swift both of the features are present and, more than that, named form and argument labels are the default, meaning that to pass an argument in the unnamed form you need to specify it in the function declaration. 

Example from [Swift documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/functions/):

```swift
func fun(argumentLabel parameterName: Type) {}

func greet(person: String, from hometown: String) -> String {
    return "Hello \(person)!  Glad you could visit from \(hometown)."
}
print(greet(person: "Bill", from: "Cupertino"))
// Prints "Hello Bill!  Glad you could visit from Cupertino."
```

There is a long discussion on how and when to use named form (outdated by 8 years, but still why not): [on swift forums](https://forums.swift.org/t/when-to-use-argument-labels-a-new-approach/1289)

### Remark on Swift function arguments

There is a way to pass an argument without specifying an argument label: give it an “empty” label (`func test(_ a: Int) {}`).

All arguments in Swift **still have fixed positions**, and you cannot change their order, despite them being uniquely named.

If you don’t explicitly specify the argument label, it will be implicitly equal to the parameter name.

## Other remarks on function arguments in Kotlin

Named arguments do not have to be passed in specific orders. Named and unnamed arguments can be mixed, but only when the values of all other arguments *can be deduced* from the named ones. That is not always clear. There can be only one variadic argument in a function, but it can be placed at any point of the arguments list, although all arguments after it have to be in a named form. Except for the case, when the last argument is a (lambda) function, which does not have to be named if passed as a trailing lambda



### Possible ways to implement

### Possible technical details

Another counterpoint: IDEs (IntelliJ IDEA) already highlight argument’s names, so why would we need it?

- Not everyone has this IDE,  for example, when you are reviewing a pull request in GitHub. Or Android Studio.

It's worth providing 2 severity levels of message when such parameter is passed in positional form: warning or error. The use case is when you already have a function and you want to make one of its parameters named only. Instead of breaking user code with an error, a warning and a quick fix would gently encourage migrating to that style of argument passing.

There are related issues for position-based destructing for data classes. “If the way to enforce parameter usage in named form is implemented, then it would be logical to extend it all the way to the data-class restructuring. That is, if constructor parameters' usage is somehow enforced to be in named form, then positional restructuring for those parameters shall be disabled, too.”


## Evaluation

### Prototypes implemented

### Implementation results

### Existing problems

### Further work

ENF is pretty solid when talking about the passing of literals, but what to do if someone passes a variable with a self-explanatory name?

Things to consider regarding the interaction with [KT-14934](https://youtrack.jetbrains.com/issue/KT-14934/Enforce-parameter-usage-only-in-named-form) and [KT-9872](https://youtrack.jetbrains.com/issue/KT-9872/Disallow-calling-a-method-with-named-argument):

## Results
