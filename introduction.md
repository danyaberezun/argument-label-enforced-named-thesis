# Introduction

## Abstract

Functions are significant elements of modern programming languages, therefore interactions with them, being it writing or using them must be as comfortable and understandable as possible. Being able to read the code and understanding what it does right away is a significantly useful quality, and in some cases, it can be achieved by using named form of function arguments or using separate names for use in named forms and inside the function's body. One possible sign of usefullness is that there are languages, with these features being built-in.

The purpose of this work is to understand the reasons, possible benefits and drawbacks, implications and ways to introduce Argument Labels and Enforced Named Argument Form into Kotlin programming language. Apart from simply collecting and discussing the information, to check the possibility and test the ideas several prototypes (versions of kotlin compiler, supporting these features) were implemented.

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

In function calls we will be particularly interested in the function calls with named argument syntax, that means, with the names of arguments being specified in the call, as presented in this example. We will often shorten this into simply "**named form**" or "named arguments form". On the contrary, the way of passing arguments _without_ specifying the names will be called "positional form"

Currently, in most cases it is up to the person writing the function call, whether to use the named or positional forms. However, it might be useful to, in some situations, restrict the user's choice to the named form only. Currently Kotlin does not have a dedicated way to do so, therefore, one part of the work, namely, **enforced named form** is focused on introducing such way and analyzing reasons for and consequences of it.

Apart from that, we will also differentiate the role of identifiers being used inside the function body and at the function call site in named form. Those being inside the function body we will call **parameter names** and those used at the function call --- **argument labels**.

As you may see in the function declaration, currently parameter names and argument labels are the same identifier, making it impossible to use different, separate identifiers for these two different roles. The second part of the work, namely, **argument labels** is focused on introducing and analyzing the implications of possibility to have separate identifiers for argument labels and parameter names.

