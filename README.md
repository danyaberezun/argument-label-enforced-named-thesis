# Kotlin Features Research: Enforced Named Argument Form and Argument Labels

This repository is dedicated for my (Mark Ipatov) project, done as a bachelors thesis, regarding the benefits, use and implementation details of two features: **Enforced Named Argument Form** and **Argument Labels**.

Current work on the documents is in progress. Aggregation and structuring of existing information from other documents is being done.

## Structure of the repository

As these features are heavily interconnected, basic descriptions for them and general reasons to implement them are presented in the [introduction](introduction.md) file.

More specific information regarding the features, discussions and existing prototypes is placed in their corresponding directories:

* Argument Labels: [Directory](ArgumentLabels/)
    * [Main document](ArgumentLabels/main.md)
    * [Prototype using jumper function (sugar)](https://github.com/MarkTheHopeful/kotlin/tree/argument-label-proto)
    * [Prorotype using additional field in structures](https://github.com/MarkTheHopeful/kotlin/tree/argument-label-proto-2)
    * [Data gathered about the interface overrides with different parameter names](argument-names-override.md)
* Enforced Named Argument Form: [Directory](EnforcedNamedForm/)
    * [Main document](EnforcedNamedForm/main.md)

Some additional findings not directly related to either of the features are gathered in the [corresponding document](additional-findings.md).

## Sources
There were many sources for discussions, ideas and code snippents, for the reason of sources and possible further ideas for discussions, they are collected here.

* [Discussion about internal and external names](https://discuss.kotlinlang.org/t/kotlin-internal-and-external-parameter-name-propose/7906)
* [Another discussion on the internal parameter names](https://discuss.kotlinlang.org/t/internal-function-parameter-name/17634)
* [Issue KT-34895 (Internal and External names) on the Kotlin youtrack, one of the starting points](https://youtrack.jetbrains.com/issue/KT-34895/Internal-and-external-name-for-a-parameter-aka-Argument-Label)
* [Issue KT-59531 (Non-stable parameter names of interface functions), regarding the overrides with different names](https://youtrack.jetbrains.com/issue/KT-59531/Add-a-way-to-make-parameter-names-of-interface-functions-non-stable)
* [Stackoverflow discussion on unnamed function arguments](https://stackoverflow.com/questions/50672203/kotlin-explicitly-unnamed-function-arguments)
* [Issue KT-9872 (disallow calling a method with named argument), to allow overrides with different names without problems](https://youtrack.jetbrains.com/issue/KT-9872/Disallow-calling-a-method-with-named-argument)
* [Issue KT-8112 (provide syntax for nameless parameters), to suppress warning and IDE quickfix](https://youtrack.jetbrains.com/issue/KT-8112/Provide-syntax-for-nameless-parameters)
* [Issue KTIJ-10594 ("Parameter is never used" quickfix), how this quickfix can render the code invalid](https://youtrack.jetbrains.com/issue/KTIJ-10594)
* [Issue KT-14934 (Enforce parameter usage only in named form), one of the starting points](https://youtrack.jetbrains.com/issue/KT-14934/Enforce-parameter-usage-only-in-named-form)
* Kotlin/Compose team design discussion 2023 (available by request)
