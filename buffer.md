Some of the arguments and examples were taken from YouTrack issues (links for issues mentioned are present), and some from different discussions on Swift and Kotlin forums. One such source is [this post on Kotlin forums](https://discuss.kotlinlang.org/t/kotlin-internal-and-external-parameter-name-propose/7906)

Suppose the part of the project you are working on is still in development, so its API is still frequently changing. But there are already some uses of the function in the project, and, possibly, not by you, so frequent changes of API are not really favourable. The worst case is when the function you work on is in some part of a library, and the end users use your library and depend on it.
 
In this situation, different changes, related to the addition/rearranging of function arguments are likely to ruin compatibility, which will require updating every usage, or, if some arguments with the same type were swapped, can even lead to subtle bugs, that will be noticed only sometime after. And all it can happen even if there were not actually any meaningful changes to the parameters that were used.
 
But if the end user were **forced to use a named form**, such changes would less likely affect them. The parameters could be rearranged, or there could be new, probably optional parameters added, and the end user would not be disturbed without a reason.
 
If one is able external **argument labels**, then they can even change the internal parameter names ****as a part of the refactoring process, and the end user will not have to do anything after this kind of update.
 
Therefore, the implementation of enforcement of named form of arguments and argument labels can lead to a better experience during function evolution in public APIs.
 
Another description of a situation, taken directly from an issue discussion:
 
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

**** 

# Current state

Here I will describe, whether and how these ideas are implemented in different languages, what is the widespread of the problems and how the solutions are currently tackled in Kotlin.

# Things to consider

What possible areas and places could these ideas affect, and how should they be considered?

ENF is pretty solid when talking about the passing of literals, but what to do if someone passes a variable with a self-explanatory name?

Another counterpoint: IDEs (IntelliJ IDEA) already highlight argument’s names, so why would we need it?

- Not everyone has this IDE,  for example, when you are reviewing a pull request in GitHub. Or Android Studio.

It's worth providing 2 severity levels of message when such parameter is passed in positional form: warning or error. The use case is when you already have a function and you want to make one of its parameters named only. Instead of breaking user code with an error, a warning and a quick fix would gently encourage migrating to that style of argument passing.

There are related issues for position-based destructing for data classes. “If the way to enforce parameter usage in named form is implemented, then it would be logical to extend it all the way to the data-class restructuring. That is, if constructor parameters' usage is somehow enforced to be in named form, then positional restructuring for those parameters shall be disabled, too.”

Things to consider regarding the interaction with [KT-14934](https://youtrack.jetbrains.com/issue/KT-14934/Enforce-parameter-usage-only-in-named-form) and [KT-9872](https://youtrack.jetbrains.com/issue/KT-9872/Disallow-calling-a-method-with-named-argument):

- how to specify that a parameter doesn't have an external name and cannot be provided in a named form (Swift: `func foo(_ parameterName)`)
- how to specify that a parameter does have a different external name, but doesn't have to be provided in a named form only
- how to require a parameter to be provided in a named form only without duplicating its external name
- how to make a parameter unnamed internally ([KT-8112](https://youtrack.jetbrains.com/issue/KT-8112/Provide-syntax-for-nameless-parameters)) without spelling its external name if it can be inferred from e.g. a supertype method.

One may say that argument labels can be easily replaced by Kotlin rich type system (and using type aliases)

Counter: I don’t think that it is very useful to create a type for every place where you are going to use a primitive, especially when there are no other logic or limitations on the type. Also, this approach will not work when both parameters are already of some non-primitive type with some logic. Further thoughts: [reply on forum](https://discuss.kotlinlang.org/t/kotlin-internal-and-external-parameter-name-propose/7906/12)



