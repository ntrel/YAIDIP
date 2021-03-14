# String Interpolation

| Field           | Value                                                             |
|-----------------|-------------------------------------------------------------------|
| DIP:            | xxxx                                                              |
| Review Count:   | 0                                                                 |
| Author:         | Andrei Alexandrescu<br>John Colvin jcolvin@symmetryinvestments.com|
| Implementation: |                                                                   |
| Status:         |                                                                   |

## Abstract

Textual formatting is often achieved either by APIs relying on formatting strings followed by arguments to be formatted (in the style of `printf`, `std.format.format`, and `std.stdio.writefln`), or by interspersing string fragments with arguments (in the style of `std.conv.text` and `std.stdio.writeln`). String interpolation enables embedding the arguments in the string itself. We propose an extremely simple yet powerful approach of lowering interpolated strings into comma-separated lists that works with both *format-string* style and *interspersion* style with no change to any library function. We demonstrate how this approach achieves all major objectives of an interpolated strings feature with a minimal footprint on the language definition and support library.

## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Copyright & License](#copyright--license)

## Rationale

A frequent pattern in programming is to create strings (for the purpose of printing to the console, writing to files, etc) by mixing predefined string fragments with data contained in variables. We identify two distinct approaches to formatting data: *format-string* style and *interspersion* style.

The most straightforward approach to implement that pattern is to intersperse expressions with string fragments in calls to variadic functions:

```d
void f1(string name) {
    writeln("Hello, ", name, "!");          // formats and prints in one go
    string s = text("Hello, ", name, "!");  // formats and returns formatted string
    ...
}
```

A more flexible approach, embodied by the classic `printf` family of C functions and carried over to D standard library functions such as `std.format.format` and `std.stdio.writefln`, is to use *format strings* that provide the string fragments intermixed with conventionally defined *formatting specifiers*. Specialized functions take such format strings followed by the data to be formatted, and replace each formatting directive with suitably formatted data:

```d
void f2(string name) {
    writefln("Hello, %40s!", name);           // formats and prints in one go
    string s = format("Hello, %40s!", name);  // formats and returns formatted string
    ...
}
```

Other examples of the format-string style are SQL prepared statements and string templates for formatting HTML documents. The convention used for format specifiers is defined by the respective APIs:

```d
void f2(string name) {
    htmlOutput("Looking for #{}...", name);
    auto rows = sql("SELECT * FROM t WHERE name = ? AND active = true", name);
    ...
}
```

Both approaches have pros and cons. The format-string style observes the important principle of [separating logic from display](https://www.cs.usfca.edu/~parrt/papers/mvc.templates.pdf). This principle is well respected in a variety of programming paradigms, such as Model-View-Controller, web development, and UX design. Localization and internationalization applications and libraries can store all display artifacts in complete separation from program logic and swap them as needed. In a perfect separation model, there is no access to computation in the formatting strings at all, even as much as a simple addition or (in the case of `printf` format strings) even the names of the variables being printed. The disadvantage of the format-string style is that the expressions to be formatted appear lexically *after* the (possibly long) format string, which makes it difficult to follow how format specifiers sync with their respective arguments.

The interspersion style is simple, intuitive, and requires learning no convention. However, creating complex outputs becomes cumbersome due to the syntactic heaviness of alternating string literals and other arguments in comma-separated lists. Also, customized formatting (such as rendering an integral in hexadecimal instead of decimal) is not immediate.

We will demonstrate how both the format-string style and the interspersion style fall short of expectations on three categories of everyday tasks:

- *code generation:* when generating code, the tight integration between the string literal fragments and the expressions to be inserted is essential to the process;
- *scripting:* shell scripts use string interpolation very frequently, to the extent that the mechanics of quoting and interpolation is an essential focus of all shell scripting languages;
- *casual printing, tracing, logging, and debugging:* sometimes, such tasks have more focus on the expressions to be printed, than on formatting paraphernalia.

Such needs are served poorly by either the interspersion approach and the format-string approach. Consider an example of code generation adapted from the implementation of `std.bitmanip.bitfields`:

```d
enum result = text(
    "@property bool ", name, "() @safe pure nothrow @nogc const { return ",
    "(", store, " & ", maskAllElse, ") != 0;}\n",
    "@property void ", name, "(bool v) @safe pure nothrow @nogc { ",
    "if (v) ", store, " |= ", maskAllElse, ";",
    "else ", store, " &= cast(typeof(", store, "))(-1-cast(typeof(", store, "))", maskAllElse, ");}\n"
);
```

(The [original code](https://github.com/dlang/phobos/blob/v2.095.1/std/bitmanip.d#L115) uses string concatenation instead of a call to `text`. We use the latter to simplify the example.) Here, the interspersion mechanics (closing quote, comma, expression, comma, opening quote) distract from following the correctness of the generated code. An approach based on format specifiers would look as follows:

```d
enum result = format(
    "@property bool %s() @safe pure nothrow @nogc const {
        return (%s & %s) != 0;
    }
    @property void %s(bool v) @safe pure nothrow @nogc {
        if (v) %s |= %s;
        else %s &= cast(typeof(%s))(-1-cast(typeof(%s))%s);
    }\n",
    name, store, maskAllElse,
    name, store, maskAllElse, store, store, store, maskAllElse
);
```

This form has less syntactic noise and appears as a format string separated from the expressions involved, for which reason we afforded to reformat it in a shape consistent with the generated code. However, the separation of format from data is clearly an impediment here requiring the reader to mentally track and pair the format specifiers `%s` with the arguments trailing the formatting string. Using positional arguments brings a marginal improvement to the code:

```d
enum result = format(
    "@property bool %1$s() @safe pure nothrow @nogc const {
        return (%2$s & %3$s) != 0;
    }
    @property void %1$s(bool v) @safe pure nothrow @nogc {
        if (v) %2$s |= %3$s;
        else %2$s &= cast(typeof(%2$s))(-1-cast(typeof(%2$s))%3$s);
    }\n",
    name, store, maskAllElse
);
```

Here, the reader only needs to track the correct use of numbers in the format specifiers and match it with the order in the trailing arguments. Correctness of the generated code is still difficult to assess on the generated code, for example the reader must mentally map tedious sequences such as `%3$s` to meaningful names such as `maskAllElse` throughout the code snippet.

By comparison, using the interpolation syntax proposed in this DIP would make the code much easier to follow:

```d
enum result = text(
    i"@property bool $name() @safe pure nothrow @nogc const {
        return ($store & $maskAllElse) != 0;
    }
    @property void $name(bool v) @safe pure nothrow @nogc {
        if (v) $store |= $maskAllElse;
        else $store &= cast(typeof($store))(-1-cast(typeof($store))$maskAllElse);
    }\n"
);
```

The latter form has dramatically less syntactic noise and appears as a single string with expressions inside escaped by `$`. Correctness of the generated code is much easier to assess in the second form as well.

Let us also look at a shell command example. Assume `url` is an URL and `file` is an filename, both preprocessed as escaped shell strings. To download `url` into `file` without risking corrupt files in case of incomplete downloads, the code below first downloads into a temporary file with the extension `.frag` and then atomically renames it to the correct name:

```d
executeShell("wget " ~ url ~ " -O" ~ file ~ ".frag && mv " ~ file ~ ".frag " ~ file);
```

The version using format specifiers is marginally more readable:

```d
// Classic
executeShell("wget %s -O%s.frag && mv %s.frag %s", url, file, file, file);
// Positional
executeShell("wget %1$s -O%2$s.frag && mv %2$s.frag %2$s", url, file);
```

The interpolated form is, again, by far the easiest to follow:

```d
executeShell(i"wget $url -O$file.frag && mv $file.frag $file");
```

Last but not least, there are numerous cases in which casual console output can use interpolated strings to reduce on boilerplate and improve clarity:

```d
writeln("Hello, ", name, ". You are ", age, " years old.");  // interspersion
writefln("Hello, %s. You are %s years old.", name, age);     // format string
writeln(i"Hello, $name. You are $age years old.");         // interpolation
```

### Why Yet Another String Interpolation Proposal?

This DIP derives from, and owes much to, the previous work on string interpolation in the D community. The abundance of such work raises the question why a new proposal is needed.

This DIP is close to the prior work yet different in key aspects as follows:

- Like [Jonathan Marler's Interpolated Strings](http://github.com/dlang/dmd/pull/7988), this DIP expands the interpolated string into an argument list. However, unlike that proposal that automatically passes the expansion as an argument list for `std.typecons.tuple`, we expand into an argument list. We will show how doing so has significant flexibility and efficiency advantages.
- Like [DIP 1027](https://github.com/dlang/DIPs/blob/master/DIPs/rejected/DIP1027.md), this DIP expands the interpolated string into a list. Unlike DIP 1027, which only supports the format-string style, this proposal supports both the interspersion and the format-string style. It also has a simpler syntax and semantics.
- We heed important lessons from [DIP 1036](https://github.com/dlang/DIPs/blob/master/DIPs/DIP1036.md), mainly that while pursuing generality, complexity must be kept under control. In wake of it, we concluded that unification of the format-string style and interspersion approach with a single interpolation syntax is not the appropriate goal. Instead, we recognize the two goals as distinct and propose distinct constructs for them.

We will demonstrate how this DIP achieves all major goals of extant proposals with a radically simpler definition and implementation.

## Related Work

* Interpolated strings have been implemented and well-received in many languages.
For many such examples, see [String Interpolation](https://en.wikipedia.org/wiki/String_interpolation).
* [Jonathan Marler's Interpolated Strings](http://github.com/dlang/dmd/pull/7988) and [DIP1027](https://github.com/dlang/DIPs/blob/master/DIPs/rejected/DIP1027.md), from which this DIP was derived.
* [DIP1036](https://github.com/dlang/DIPs/blob/master/DIPs/DIP1036.md), which is as of the time of this writing in review.
* [C#'s implementation](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/tokens/interpolated#compilation-of-interpolated-strings) which returns a formattable object that user functions can use
* [Javascript's implementation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals) which passes `string[], args...` to a builder function very similarly to this proposal
* Jason Helson submitted a DIP [String Syntax for Compile-Time Sequences](https://github.com/dlang/DIPs/pull/140).

## Description

D strings have several syntactical form, among which a few that follow the pattern of a letter followed by the string proper. Such include `r"WYSIWYG strings"`, `q"[delimited strings]"`, and `q{token strings}`. Our proposal follows the same pattern by introducing `i"interpolated strings"` and `f"interpolated formatting strings"`.

An *interpolated string* is a regular D string prefixed with the letter `i`, as in `i"Hello"`. An *interpolated formatting string* is a regular D string prefixed with the letter `f`, as in `f"world"`. No whitespace is allowed between `i` or `f` and the opening quote. The first expands into the interspersed style, and the second expands into the format-string style. We refer to them as `i`-strings and `f`-strings, respectively.

An `i`-string or an `f`-string may occur only in one of the following contexts:

- in the argument list of a function or constructor call;
- in the argument list of a `mixin`;
- in the argument list of a template instantiation.

In any other context, `i`-string or an `f`-string are ill-formed.

For an example of `i`-strings, the function call expression:

```d
writeln(i"I ate $apples apples and $bananas bananas totalling $(apples + bananas) fruit.")
```

is lowered into:

```d
writeln("I ate ", apples, " apples and ", bananas, " bananas totalling ", apples + bananas, " fruit.")
```

For an example of `f`-strings, the function call expression:

```d
writefln(f"I ate %s$apples apples and %s$bananas bananas totalling %s$(apples + bananas) fruit.")
```

is lowered into:

```d
writefln(f"I ate %s apples and %s bananas totalling %s fruit.", apples, bananas, apples + bananas)
```

The resulting lowered code is subjected to the usual typechecking and has the same semantics as if the lowered code were present in the source.

After introducing an intuition of how interpolated string work, let us formalize the syntax and semantics. Lexically:

```
InterpolatedString:
   i" DoubleQuotedCharacters "
InterpolatedFormattingString:
   f" DoubleQuotedCharacters "
```

The `InterpolatedString` and `InterpolatedFormattingString` appear in the parser grammar as an `InterpolatedExpression`, which is under `PrimaryExpression`.

```
InterpolatedExpression:
   InterpolatedString
   InterpolatedString StringLiterals
   InterpolatedFormattingString
   InterpolatedFormattingString StringLiterals
```

Inside an interpolated string, the character `$` is of particular interest because the interpolated string will use it as an escape. To render `$` verbatim inside an interpolated string, the sequence `$$` shall be used. The contents of the `InterpolatedExpression` must conform to the following grammar, which is identical for `i`-strings and `f`-strings:

```
Elements:
    Element
    Element Elements

Element:
    Character excluding '$'
    '$$'
    '$' Identifier
    '$(' Type ')'
    '$(' Expression ')'
```

In the grammar above `Type` is the nonterminal for types, and `Expression` is the nonterminal for general D expressions.

The `InterpolatedExpression` is lowered to a comma-separated list that consists of the string fragments interspersed with the expressions escaped by `$`. The `InterpolatedFormattingExpression` is lowered to the string literal fragments stitched together, followed by all escaped fragments, in lexical order.

An `f`-string expansion produces a literal string in the first position even if that would be empty: `fun(f"$x")` lowers to `fun("", x)`, not `fun(x)`. In contrast, `i`-string expansion never produces empty string literals: `fun(i"$x")`expands to `fun(x)` and `fun(i"$x$y")`expands to `fun(x, y)`.

Any lexical errors (such as a `$` followed by a space, or unbalanced `(` and `)`) will be reported during parsing. Semantic checking will ensure that interpolated strings occur only in the contexts specified above. Then, other semantic errors will be reported during the typechecking of the lowered code.

This concludes the syntax and semantics of the proposed feature.

Why choose the `$` when many popular languages and libraries (Python, C++20, C#) use `{` and `}` as escape characters? Also, why use `$(` and `)` as opposed to `${` and `}`, as perhaps a bash user may be more familiar with?

One essential use of interpolation, specific to D, is for code generation. In generated D code, curly braces `{` and `}` are abundant. Requiring `{{` and `}}` everywhere in the generated code would have been aggravating. Anecdotal evidence has been collected in the creation of this DIP, which initially attempted to use `{` and `}` for escaping: the examples extracted from `std.bitmanip.bitfields` turned out to have numerous bugs, and be difficult to read when corrected:

```d
// Using `{` and `}` for escaping, similar to Python's f-strings
enum result = text(
    i"@property bool {name}() @safe pure nothrow @nogc const {{
        return ({store} & {maskAllElse}) != 0;
    }}
    @property void {name}(bool v) @safe pure nothrow @nogc {{
        if (v) {store} |= {maskAllElse};
        else {store} &= cast(typeof({store}))(-1-cast(typeof({store})){maskAllElse});
    }}\n"
);
```

In contrast, occurrences of `$` in D code are rare --- indeed more likely to be present in generated code that uses interpolation itself. This makes `$` a disproportionately strong candidate compared to other choices.

The second question --- why not use `${` and `}` instead of `$(` and `)` --- has a simple answer: the elements to group inside the escape sequences are expressions, not statements. There already exists a syntax for grouping expressions, and that's surrounding them with `(` and `)`, which closes the case by invoking the [principle of least astonishment](https://en.wikipedia.org/wiki/Principle_of_least_astonishment).

### Use Cases

Although the proposed feature is deceptively simple, its flexibility affords a multitude of use cases. They can be easily assessed with a current D compiler by typing in the lowered code.

#### Passing arguments to functions

The simplest use case of interpolated strings, and the most likely to be encountered in practice, is as argument to functions such as `writeln`, `text`, `writefln`, or `format`:

```d
void main(string[] args) {
    import std.stdio;
    writeln(i"The program $(args[0]) received $(args.length - 1) arguments.");
    // Lowering: --->
    // writeln("The program ", args[0], " received ", args.length - 1, " arguments.");

    writefln(f"The program %s$(args[0]) received %s$(args.length - 1) arguments.");
    // Lowering: --->
    // writefln("The program %s received %s arguments.", args[0], args.length - 1);

    auto s = sqlExec(f"INSERT INTO runs VALUES(?$(args[0]), ?$(args.length - 1))");
    // Lowering: --->
    // auto s = sqlExec("INSERT INTO runs VALUES(?, ?)", args[0], args.length - 1);
}
```

A function such as `std.stdio.writeln` or `std.stdio.writefln` above has no way to know whether it was called via an interpolated string or with arguments specified as in the corresponding lowering; interpolated strings are purely a call-side mechanism. We consider this a key characteristic of the proposed feature that drastically simplifies its definition and interoperation with existing code.

For `f`-strings, the convention for format specifiers varies with the API --- for example, `printf` and `std.stdio.writefln` use the well-known `%`-prefixed specifiers, whereas SQL traditionally uses `?`. For that reason and to keep complexity to a minimum, `f`-strings do not try to be clever and provide their own convention and translation mechanism; instead, they just concatenate the string literal fragments together just like the user wrote them, and follows them with the interpolated expressions. An `f`-string does not create any text.

#### Saving the result of interpolation as a tuple

Although it may seem limiting to impose that interpolated strings expand in a function call, `tuple` offers an immediate and efficient mechanism for storing the result of interpolation. The following program produces the same output as the previous one:

```d
void main(string[] args) {
    import std.stdio;
    auto t1 = tuple(i"The program $(args[0]) received $(args.length - 1) arguments.");
    // Lowering: --->
    // auto t1 = tuple("The program ", args[0], " received ", args.length - 1, " arguments.");
    writeln(t1.expand);

    auto t2 = tuple(f"The program %s$(args[0]) received %s$(args.length - 1) arguments.");
    // Lowering: --->
    // auto t2 = tuple("The program %s received %s arguments.", args[0], args.length - 1);
    writefln(t2.expand);
}
```

#### Manipulator-style formatting

[C++'s iostreams](http://www.cplusplus.com/reference/iolibrary/) introduced the notion of [*stream manipulators*](https://en.cppreference.com/w/cpp/io/manip) --- special functions interspersed with the data to print, which direct the way data is to be formatted. For example, the C++ `std::dec` and `std::hex` manipulators instruct the formatting engine to format the following integral in decimal and hexadecimal, respectively:

```C++
// C++ code
#include <iostream>
void fun(int x) {
    std::cout << std::dec << x << " in hexadecimal is 0x" << std::hex << x << ".\n";
}
```

The approach is obviously extensible by simply adding new manipulators. Unfortunately, C++ stream manipulators have developed a poor reputation because they are very heavy syntactically --- interspersion is done with the visually prominent `<<` and the user must choose between polluting their namespace and using the `std::` prefix for scope resolution with each manipulator. (These issues and other unrelated ones motivated the introduction of the `<format>` facility in C++20.)

Using interpolation, a manipulators-based approach can be used elegantly and implemented with minimal effort. Consider for example using stream manipulators such as `dec` and `hex` for `writeln` by using an `i`-string:

```d
void fun(int x) {
    writeln(i"$dec$x in hexadecimal is 0x$hex$x.");
    // Lowering: --->
    // writeln(dec, x, " in hexadecimal is 0x", hex, x, ".");
}
```

There is no need for defining, implementing, and memorizing a mini-language of encoded format specifiers --- all formatting can be done with D language expressions. Continuing the example, the library can just as easily define parameterized formatting for floating-point numbers, such as width, precision, and scientific notation:

```d
void fun(double x) {
    writeln(i"$x can be written as $scientific$x and its exact value is $(fixed(20, 10))$x");
    // Lowering: --->
    // writeln(x, " can be written as ", scientific, x, " and its exact value is ", fixed(20, 10), x);
}
```

#### Use in `mixin` declarations and expressions

`i`-strings (but not `f`-strings) are allowed in `mixin` declarations and expressions:

```d
immutable x = "asd", y = 42;
mixin(i"int $x = $y;");
// Lowering --->
// mixin("int ", x, " = ", y, ";");
auto z = mixin(i"$x + 5");
// Lowering --->
// auto z = mixin(x, " + 5");
```

#### Use in the argument list of template instantiations

An interpolated string may be present in the  argument list of a template instantiation. This allows, for example, a parser generator to compose with strings in the grammar definition:

```d
struct Grammar(spec...) { ... }

immutable identifier = "( [ character ]+ )"

alias Calculator = Grammar!(
    i"Expression := Term + Term
    Term := Factor * Factor
    Factor := $identifier | '(' Expression ')' "
);
// Lowering: --->
// alias Calculator = Grammar!(
//     "Expression := Term + Term
//     Term := Factor * Factor
//     Factor := ", identifier, " | '(' Expression ')' "
// );
```

In certain cases, the interpolated string can be passed to `AliasSeq` as well resulting in a sequence of strings interspersed with identifiers:

```d
int x = 42;
alias Q = AliasSeq!(i"I'm interpolating $x here.");  // OK, use Q as a type
auto q = AliasSeq!(i"I'm interpolating $x here.");   // OK, store as AliasSeq object
```

### Limitations and tradeoffs

Users may be confused that they cannot define variables that are interpolated strings:

```d
int x = 42;
auto s = i"Let's interpolate $x!"  // Error, interpolated string not allowed here
```

The remedy is simple and may be suggested by the text of the error message: use a tuple to store the interpolation, or call a function such as `text` or `format` to convert everything to a string:

```d
int x = 42;
auto s = text(i"Let's interpolate $x!");        // OK, store as string
auto t = tuple(i"Let's interpolate $x!");       // OK, store as tuple
alias Q = AliasSeq!(i"Let's interpolate $x!");  // OK, use as type
auto q = AliasSeq!(i"Let's interpolate $x!");   // OK, store as AliasSeq
```

As mentioned, functions or templates are not aware whether they received an interpolated string or a manually written list of arguments. This confers consistency, simplicity, and uniformity to the approach.

It is not possible for an `f`-string to pass the string literal as a template argument and the interpolated expressions as run-time arguments:

```d
void main(string[] args) {
    import std.stdio;
    // No equivalent using interpolation
    writefln!"The program %s received %s arguments."(args[0], args.length - 1);
}
```

`f`-strings do not provide special rules to match format specifiers (such as `%s` in `std.stdio.writefln` or `?` in SQL prepared statements) against arguments:

```d
void fun(int x) {
    writefln(f"Adding %s$x...");  // prints "Adding 42 ..."
    writefln(f"Adding $x...");    // prints "Adding ..."
    sqlExec(f"INSERT INTO t VALUES(?$x)");  // OK
    sqlExec(f"INSERT INTO t VALUES($x?)");  // OK, equivalent
    sqlExec(f"INSERT INTO t$x VALUES(?)");  // OK, equivalent but perverse
    sqlExec(f"INSERT INTO t VALUES($x)");   // Runtime error, missing '?' binding
}
```

We consider that adding special mechanisms to adapt `f`-strings to a variety of formatting conventions is disproportionately complex and adds its own liabilities.

## Breaking Changes and Deprecations

Because `InterpolatedString` and `InterpolatedFormatString` are new tokens, no existing code is broken.

## Copyright & License

Copyright (c) 2021 by the D Language Foundation

Licensed under Creative Commons Zero 1.0

