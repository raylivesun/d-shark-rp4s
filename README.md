# D Shark rp4s
Interpolation Expression Sequence
Contents [hide]

    IES Literals
        Double Quoted IES Literals
        Wysiwyg IES Literals
        Token IES Literals
    Expression Translation
    core.interpolation Types
        InterpolationHeader and InterpolationFooter
        InterpolatedLiteral
        InterpolatedExpression
    Accepting and Processing IES
    Converting to a String

Interpolation Expression Sequences (IES) are expressions that intersperse string literal data and values. An interpolation expression is written like a string, but can contain values that are passed directly to a function that is able to accept them. It is transformed into a Sequence of expressions that can be overloaded or handled by template functions.
IES Literals

InterpolationExpressionSequence:
    InterpolatedDoubleQuotedLiteral
    InterpolatedWysiwygLiteral
    InterpolatedTokenLiteral

An Interpolation Expression sequence can be either wysiwyg quoted, double quoted, or a token literal. Only double quoted literals can have escapes in them.

Unlike string literals, IES literals cannot have a suffix defining the width of the character type for the string expressions.
Double Quoted IES Literals

InterpolatedDoubleQuotedLiteral:
    i" InterpolatedDoubleQuotedCharactersopt "

InterpolatedDoubleQuotedCharacters:
    InterpolatedDoubleQuotedCharacter
    InterpolatedDoubleQuotedCharacter InterpolatedDoubleQuotedCharacters

InterpolatedDoubleQuotedCharacter:
    DoubleQuotedCharacter
    InterpolationEscapeSequence
    InterpolationExpression

InterpolationEscapeSequence:
    EscapeSequence
    \$

InterpolationExpression:
    $( AssignExpression )

Like DoubleQuotedString, double-quoted IES literals can have escape characters in them. Added to the normal escapes is the ability to escape a literal $

A $ followed by any character other than a left parenthesis is treated as a literal $ in the expression, there is no need to escape it.

The expression inside an InterpolationExpression is a full D expression, and escapes are not needed inside that part of the expression.
Wysiwyg IES Literals

InterpolatedWysiwygLiteral:
    i` InterpolatedWysiwygCharactersopt `

InterpolatedWysiwygCharacters:
    InterpolatedWysiwygCharacter
    InterpolatedWysiwygCharacter InterpolatedWysiwygCharacters

InterpolatedWysiwygCharacter:
    WysiwygCharacter
    InterpolationExpression

Wysiwyg ("what you see is what you get") IES literals are defined like WysiwygString strings, but only support backquote syntax. No escapes are recognized inside these literals.
Token IES Literals

InterpolatedTokenLiteral:
    iq{ InterpolatedTokenStringTokensopt }

InterpolatedTokenStringTokens:
    InterpolatedTokenStringToken
    InterpolatedTokenStringToken InterpolatedTokenStringTokens

InterpolatedTokenStringToken:
    InterpolatedTokenNoBraces
    { InterpolatedTokenStringTokensopt }

InterpolatedTokenNoBraces:
    TokenNoBraces
    InterpolationExpression

Like TokenString, IES Token literals must contain only valid D tokens, with the exception of InterpolationExpression. No escapes are recognized.
Expression Translation

When the lexer encounters an Interpolation Expression Sequence, the token is translated into a sequence of expressions, which replaces the single token. The sequence always starts with the expression core.interpolation.InterpolationHeader() and always ends with core.interpolation.InterpolationFooter()

Each part str of the token which is literal string data is translated into the expression core.interpolation.InterpolatedLiteral!(str)

Each part $(expr) of the token which is an InterpolationExpression is translated into the sequence core.interpolation.InterpolatedExpression!(expr), expr.

// simple version of std.typecons.Tuple
struct Tuple(T...) { T value; }
Tuple!T tuple(T...)(T value) { return Tuple!T(value); }

import core.interpolation;
string name = "John Doe";
auto items = tuple(i"Hello, $(name), how are you?");
assert(items == tuple(
    InterpolationHeader(),                       // denotes the start of an IES
    InterpolatedLiteral!("Hello, ")(),           // literal string data
    InterpolatedExpression!("name")(),           // expression literal data
    name,                                        // expression passed directly
    InterpolatedLiteral!(", how are you?")(),    // literal string data
    InterpolationFooter()));                     // denotes the end of an IES

core.interpolation Types

Types defined in core.interpolation need not be imported to use IES. These are automatically imported when an IES is used. The types are defined so as to make it easy to introspect the IES for processing at compile-time.
InterpolationHeader and InterpolationFooter

The InterpolationHeader and InterpolationFooter type are empty structs that allow easy overloading of functions to handle IES. They also can be used to understand which parts of a expression list were passed via IES.

These types have a toString definition that returns an empty string, to allow for processing by functions which intend to convert IES to text, such as std.stdio.writeln or std.conv.text.
InterpolatedLiteral

The InterpolatedLiteral type is an empty struct that provides compile-time access to the string literal portion of an IES. This type also provides a toString member function which returns the part of the sequence that this value replaced.
InterpolatedExpression

The InterpolatedExpression type is an empty struct that provides compile-time access to the literal that was used to form the following expression. It provides a toString member function which returns the empty string. It also has an enum expression member, which is equal to the template parameter used to construct the type.

string name = "John Doe";
auto ies = i"Hello, $(name)";
static assert(is(typeof(ies[0]) == InterpolationHeader));
static assert(ies[1].toString() == "Hello, ");
static assert(ies[2].expression == "name");
assert(ies[3] == name);
static assert(is(typeof(ies[4]) == InterpolationFooter));

Accepting and Processing IES

The recommended mechanism to accept IES is to provide a variadic template function to match the various parameters inside the sequence, surrounded by explicit InterpolationHeader and InterpolationFooter parameters.

void processIES(Sequence...)(InterpolationHeader, Sequence data, InterpolationFooter)
{
    // process `data` here
}

An IES can also contain types as interpolation expressions. This can be used by passing to a variadic template parameter.

template processIESAtCompileTime(InterpolationHeader header, Sequence...)
{
    static assert(Sequence[$-1] == InterpolationFooter());
}

alias result = processIES!i"Here is a type: $(int)";

Converting to a String

In many cases, it is desirable to convert an IES to a string. The Phobos function std.conv.text can convert the IES to a string for use in any context where a string is needed, for instance to assign to a string variable, or call a function that accepts a string.

import std.conv : text;
string name = "John Doe";

string badgreeting = i"Hello, $(name)"; // Error
string greeting = i"Hello, $(name)".text; // OK
assert(greeting == "Hello, John Doe");

It is highly recommended for library authors who wish to accept IES to provide an overload that accepts them, rather than rely on std.conv, as this incurs unnecessary allocations. This is especially important where certain types of injection attacks are possible from malicious user-provided data.
