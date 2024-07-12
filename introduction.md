# Introduction

## Abstract

Functions are significant elements of modern programming languages; therefore, interactions with them, be it writing or using them, must be as comfortable and understandable as possible. For the code to be understood from the first read is a valuable quality, and in some cases, it can be achieved by using named form of function arguments or using separate names for use in named forms and inside the function's body. One possible sign of usefulness is that there are languages with these features being built-in.

This work aims to understand the reasons, possible benefits and drawbacks, implications and ways to introduce Argument Labels and Enforced Named Argument Form in the Kotlin programming language. Apart from simply collecting and discussing the information to check the possibility and test the ideas, several prototypes (versions of the kotlin compiler supporting these features) were implemented.

## Descriptions and Definitions

To define, what are the features we will be working with, let us look at the following Kotlin code snippet:


```kotlin
fun drawLine(start: Point, end: Point, width: Int) {
    drawer.moveTo(start)
    drawer.setWidth(width)
    drawer.drawTo(end)
}

drawLine(start=(500, 66), end=(200, 39), width=30)
```

We define three parts we are interested in --- function declaration (on line 1), function body (lines 2 to 4) and function call (line 6).

In function calls, we will be particularly interested in the function calls with named argument syntax, which means the names of arguments are specified in the call, as presented in this example. We often shorten this into "**named form**" or "named arguments form". On the contrary, passing arguments _without_ specifying the names will be called "**positional form**".

In most cases, it is up to the person writing the function call whether to use the named or positional forms. However, in some situations, it might be helpful to restrict the user's choice to the named form only. Currently, Kotlin does not have a dedicated way to do so; therefore, one part of the work, namely, **enforced named form** is focused on introducing a way to do so and analyzing reasons for and consequences of it.

We will also differentiate the role of identifiers being used inside the function body and at the function call site in named form. Those being inside the function body, we will call **parameter names**, and those used at the function call --- **argument labels**. To differentiate these two separate roles from the sole identifier being used for them, we will use the word _argument name_ for it. Like in this snippet, `start`, `end` and `width` are still argument names, as Kotlin does not allow two separate identifiers.

As you may see in the function declaration, parameter names and argument labels are currently the same identifiers, making it impossible to use different, separate identifiers for these two distinct roles. The second part of the work, namely, **argument labels**, focuses on introducing and analyzing the implications of having separate identifiers for argument labels and parameter names.

## Common Reasons

The two ideas described naturally come together, as they are both related to introducing new syntax to the function declaration, more specifically to the declaration of arguments. Therefore, it is natural that they have some reasons to implement applicable to both argument labels and enforced named form.

### Self-documenting code

As stated in the abstract, self-explanatory code, which is easy to read and understand, is essential for the development process. 

To get a better understanding of it, let's move back to the example given in the previous section, the `drawLine` function:

```kotlin
fun drawLine(start: Point, end: Point, width: Int) {
    drawer.moveTo(start)
    drawer.setWidth(width)
    drawer.drawTo(end)
}
```

One of the simplest ways to call it will be the following:

```kotlin
drawLine((500, 66), (200, 39), 30)
```

Here, with positional argument form, the call is easy to write, but the meaning of the arguments is quite hard to understand without seeing the declaration of the function before your eyes. However, if the developer decides to enforce a named form of arguments, then the call turns into:

```kotlin
drawLine(start=(500, 66), end=(200, 39), width=30)
```

, which is now more readable. One reading this code does not need to go to the function declaration. But what if we want slightly different argument labels used in the call place? For example, _so the call can be read as a natural language sentence_?

One simple choice would be to change the names of the arguments, but what if the new names make sense in the call place but not in the function body? Separation of argument labels and parameter names can help with this problem.

```kotlin
drawLine(from=(500, 66), to=(200, 39), withThickness=30)
```

All in all, using the named form (and, therefore, enforcing it in some places) and using argument labels lead to improved code readability and reduced ambiguity, which results in fewer errors.

An additional example, when the introduction of argument labels results in function calls being read as sentences, can be seen here:

```kotlin
// Suppose we have the following declaration using argument labels feature
fun <E> List<E>.max([by] comparator: Comparator<E>, [or] zero: E) {
        E result = zero
        for (item in this) {
                if (comparator.compare(result, item) > 0) {
                        result = item
                }
        }
        return result
}

listOf(1, 2, 3, 4).max(by=naturalOrder, or=0) // pretty understandable
// VS
// Choose names `by` and `or` for function body
fun <E> List<E>.max(by: Comparator<E>, or: E) {
        E result = or // Ouch!
        for (item in this) {
                if (**by.compare**(result, item) > 0) { // Ouch!
                        result = item
                }
        }
        return result
}
// or choose names `comparator` and `zero` for labels
listOf(1, 2, 3, 4).max(**comparator**=naturalOrder, **zero**=0) // meh...
```

Suppose the part of the project you are working on is still in development, so its API is frequently changing. But there are already some uses of the function in the project, and, possibly, not by you, so frequent API changes are not favourable. The worst case is when the function you work on is in some part of a library, and the end users use your library and depend on it.

In this situation, different changes related to the addition/rearranging of function arguments are likely to ruin compatibility, which will require updating every usage, or, if some arguments with the same type were swapped, can even lead to subtle bugs that will be noticed only sometime after. All of this can happen even if there are no significant changes to the parameters used.

Therefore, if we are to change a function, we will have to support both the new version and the old one while marking it as deprecated to maintain compatibility (perhaps binary)

But if the end user were **forced to use a named form**, such changes would less likely affect them. The parameters could be rearranged, or new, probably optional parameters could be added, and the end user would not be disturbed without a reason.

Suppose one can use external **argument labels**. In that case, the developer can even change the internal parameter names as a part of the refactoring process without affecting the end user.

Therefore, the implementation of enforcement of named form of arguments and argument labels can lead to a better experience during function evolution in public APIs.

Another description of a situation, taken directly from the issue discussion:

“Here is an additional use-case. When designing a DSL I might have a function with many params and I know that I will adding more params in the future. If people rely on parameter position, then me adding more parameters may break source compatibility. I was to preserve future source compatibility by forcing all my advanced params to be used in named form only.

The related story is for the last lambda parameters that can be used outside of parenthesis. I also want to avoid it being used positionally in such case. However, "forcing its usage in named form" shall still also allow its usage after the closing parenthesis. This way, if I have something like:

```kotlin
fun myBuilder(param1: T1, param2: T2, ..., paramN: TN, body: () -> Unit)
```

and only named usage if forced like this:

```kotlin
myBuilder(paramI = valI, ...) { ... }
```

I can safely add more named params before `body`, still being sure that old sources compile.”

## In general about Swift

Both features described here are present in the Swift programming language (used for iOS development). The difference is that argument labels and enforced named form is the default behaviour in Swift. Each argument must be passed in the named form, and whether an argument can be passed without specifying the name is decided in the function declaration. Argument labels are defined in the function declaration as just an additional name (so there are two identifiers in place of one).

An example of the Swift code:

```swift
func greet(person: String, from hometown: String) -> String {
    return "Hello \(person)!  Glad you could visit from \(hometown)."
}
print(greet(person: "Bill", from: "Cupertino"))
```

The argumentation for argument labels is: "The use of argument labels can allow a function to be called in an expressive, sentence-like manner, while still providing a function body that’s readable and clear in intent."

One significant difference from Kotlin regarding the features is that despite having enforced named arguments form, Swift arguments in the function call have to follow the same order as in the function declaration. Therefore, the names are being used only as the source of information for the user but not for the compiler. The additional consequence of this fact is that in Swift, two or more parameters can have the same argument label (although the parameter names must be unique).
