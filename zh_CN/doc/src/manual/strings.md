# [字符串](@id man-strings)

Strings are finite sequences of characters. Of course, the real trouble comes when one asks what
a character is. The characters that English speakers are familiar with are the letters `A`, `B`,
`C`, etc., together with numerals and common punctuation symbols. These characters are standardized
together with a mapping to integer values between 0 and 127 by the [ASCII](https://en.wikipedia.org/wiki/ASCII)
standard. There are, of course, many other characters used in non-English languages, including
variants of the ASCII characters with accents and other modifications, related scripts such as
Cyrillic and Greek, and scripts completely unrelated to ASCII and English, including Arabic, Chinese,
Hebrew, Hindi, Japanese, and Korean. The [Unicode](https://en.wikipedia.org/wiki/Unicode) standard
tackles the complexities of what exactly a character is, and is generally accepted as the definitive
standard addressing this problem. Depending on your needs, you can either ignore these complexities
entirely and just pretend that only ASCII characters exist, or you can write code that can handle
any of the characters or encodings that one may encounter when handling non-ASCII text. Julia
makes dealing with plain ASCII text simple and efficient, and handling Unicode is as simple and
efficient as possible. In particular, you can write C-style string code to process ASCII strings,
and they will work as expected, both in terms of performance and semantics. If such code encounters
non-ASCII text, it will gracefully fail with a clear error message, rather than silently introducing
corrupt results. When this happens, modifying the code to handle non-ASCII data is straightforward.

关于Julia的字符串类型有一些值得注意的高级特性：

  * The built-in concrete type used for strings (and string literals) in Julia is [`String`](@ref).
    This supports the full range of [Unicode](https://en.wikipedia.org/wiki/Unicode) characters via
    the [UTF-8](https://en.wikipedia.org/wiki/UTF-8) encoding. (A [`transcode`](@ref) function is
    provided to convert to/from other Unicode encodings.)
  * All string types are subtypes of the abstract type `AbstractString`, and external packages define
    additional `AbstractString` subtypes (e.g. for other encodings).  If you define a function expecting
    a string argument, you should declare the type as `AbstractString` in order to accept any string
    type.
  * Like C and Java, but unlike most dynamic languages, Julia has a first-class type for representing
    a single character, called [`AbstractChar`](@ref). The built-in [`Char`](@ref) subtype of `AbstractChar`
    is a 32-bit primitive type that can represent any Unicode character (and which is based
    on the UTF-8 encoding).
  * As in Java, strings are immutable: the value of an `AbstractString` object cannot be changed.
    To construct a different string value, you construct a new string from parts of other strings.
  * Conceptually, a string is a *partial function* from indices to characters: for some index values,
    no character value is returned, and instead an exception is thrown. This allows for efficient
    indexing into strings by the byte index of an encoded representation rather than by a character
    index, which cannot be implemented both efficiently and simply for variable-width encodings of
    Unicode strings.

## [Characters](@id man-characters)

A `Char` value represents a single character: it is just a 32-bit primitive type with a special literal
representation and appropriate arithmetic behaviors, and which can be converted
to a numeric value representing a
[Unicode code point](https://en.wikipedia.org/wiki/Code_point).  (Julia packages may define
other subtypes of `AbstractChar`, e.g. to optimize operations for other
[text encodings](https://en.wikipedia.org/wiki/Character_encoding).) Here is how `Char` values are
input and shown:

```jldoctest
julia> 'x'
'x': ASCII/Unicode U+0078 (category Ll: Letter, lowercase)

julia> typeof(ans)
Char
```

您可以轻松地将`Char`转换为其整数值，即Unicode代码：

```jldoctest
julia> Int('x')
120

julia> typeof(ans)
Int64
```

在32位的系统内，[`typeof(ans)`](@ref) 将显示为[`Int32`](@ref)。您可以轻松地将一个整数值转化回为`Char` 。

```jldoctest
julia> Char(120)
'x': ASCII/Unicode U+0078 (category Ll: Letter, lowercase)
```

并非所有的整数值都是有效的Unicode代码点，但是为了性能，`Char`转化时不会检查每个值是否有效。如果你想检查每个转换的值是否为有效值，请使用[`isvalid`](@ref)函数去检查：

```jldoctest
julia> Char(0x110000)
'\U110000': Unicode U+110000 (category In: Invalid, too high)

julia> isvalid(Char, 0x110000)
false
```

截至发稿时，有效的 Unicode 码位为，从`U+00`至`U+d7ff`，以及从`U+e000`至`U+10ffff`。这些尚未被赋予可理解的含义，也不一定是应用程序可解释的，但所有这些值都被认为是有效的Unicode字符。

你可以使用单引号输入任何Unicode字符，`\u` 后跟最多四个十六进制或`\U`后跟最多八个十六进制数字（最长有效值仅需要六个）

```jldoctest
julia> '\u0'
'\0': ASCII/Unicode U+0000 (category Cc: Other, control)

julia> '\u78'
'x': ASCII/Unicode U+0078 (category Ll: Letter, lowercase)

julia> '\u2200'
'∀': Unicode U+2200 (category Sm: Symbol, math)

julia> '\U10ffff'
'\U10ffff': Unicode U+10ffff (category Cn: Other, not assigned)
```

Julia 使用系统默认的区域和语言设置来确定，哪些字符可以被正确显示，哪些需要用`\u`或 `\U` 的转义来显
示。除 Unicode 转义格式之外，还可以使用所有的[C语言转义的输入格式](https://en.wikipedia.org/wiki/C_syntax#Backslash_escapes)：

```jldoctest
julia> Int('\0')
0

julia> Int('\t')
9

julia> Int('\n')
10

julia> Int('\e')
27

julia> Int('\x7f')
127

julia> Int('\177')
127
```

您可以使用`Char` 值进行比较和简单的算术运算：

```jldoctest
julia> 'A' < 'a'
true

julia> 'A' <= 'a' <= 'Z'
false

julia> 'A' <= 'X' <= 'Z'
true

julia> 'x' - 'a'
23

julia> 'A' + 1
'B': ASCII/Unicode U+0042 (category Lu: Letter, uppercase)
```

## 字符串基础知识

字符串文本应该放于双引号或三重双引号中间：

```jldoctest helloworldstring
julia> str = "Hello, world.\n"
"Hello, world.\n"

julia> """Contains "quote" characters"""
"Contains \"quote\" characters"
```

如果想从字符串中提取字符, 可以通过索引字符串的方式来获取：

```jldoctest helloworldstring
julia> str[1]
'H': ASCII/Unicode U+0048 (category Lu: Letter, uppercase)

julia> str[6]
',': ASCII/Unicode U+002c (category Po: Punctuation, other)

julia> str[end]
'\n': ASCII/Unicode U+000a (category Cc: Other, control)
```

包括字符串在内的许多Julia对象可以用整数索引。可以通过[`firstindex(str)`](@ref)来获取第一个元素的索引，通过[`lastindex(str)`](@ref)来获取最后一个元素的索引。关键字`end`可以在索引操作中用作最后一个索引的简写。Julia中的大多数索引都是基于1的：许多整数索引对象的第一个元素索引就是1（正如我们将在下面看到的，但这并不一定意味着在索引`n`处找到最后一个索引元素，其中`n`为字符串长度。）

可以使用[`end`](@ref)像正常值一样执行算术运算和其他运算：

```jldoctest helloworldstring
julia> str[end-1]
'.': ASCII/Unicode U+002e (category Po: Punctuation, other)

julia> str[end÷2]
' ': ASCII/Unicode U+0020 (category Zs: Separator, space)
```

使用小于1或大于`end`的索引会引发错误：

```jldoctest helloworldstring
julia> str[0]
ERROR: BoundsError: attempt to access "Hello, world.\n"
  at index [0]
[...]

julia> str[end+1]
ERROR: BoundsError: attempt to access "Hello, world.\n"
  at index [15]
Stacktrace:
[...]
```

还可以使用范围索引来提取字符串：

```jldoctest helloworldstring
julia> str[4:9]
"lo, wo"
```

注意表达式`str[k]` 和 `str[k:k]` 并没有给出相同的结果

```jldoctest helloworldstring
julia> str[6]
',': ASCII/Unicode U+002c (category Po: Punctuation, other)

julia> str[6:6]
","
```

The former is a single character value of type `Char`, while the latter is a string value that
happens to contain only a single character. In Julia these are very different things.

Range indexing makes a copy of the selected part of the original string.
Alternatively, it is possible to create a view into a string using the type [`SubString`](@ref),
for example:

```jldoctest
julia> str = "long string"
"long string"

julia> substr = SubString(str, 1, 4)
"long"

julia> typeof(substr)
SubString{String}
```

Several standard functions like [`chop`](@ref), [`chomp`](@ref) or [`strip`](@ref)
return a [`SubString`](@ref).

## Unicode编码和UTF-8编码

Julia fully supports Unicode characters and strings. As [discussed above](@ref man-characters), in character
literals, Unicode code points can be represented using Unicode `\u` and `\U` escape sequences,
as well as all the standard C escape sequences. These can likewise be used to write string literals:

```jldoctest unicodestring
julia> s = "\u2200 x \u2203 y"
"∀ x ∃ y"
```

Whether these Unicode characters are displayed as escapes or shown as special characters depends
on your terminal's locale settings and its support for Unicode. String literals are encoded using
the UTF-8 encoding. UTF-8 is a variable-width encoding, meaning that not all characters are encoded
in the same number of bytes. In UTF-8, ASCII characters -- i.e. those with code points less than
0x80 (128) -- are encoded as they are in ASCII, using a single byte, while code points 0x80 and
above are encoded using multiple bytes -- up to four per character. This means that not every
byte index into a UTF-8 string is necessarily a valid index for a character. If you index into
a string at such an invalid byte index, an error is thrown:

```jldoctest unicodestring
julia> s[1]
'∀': Unicode U+2200 (category Sm: Symbol, math)

julia> s[2]
ERROR: StringIndexError("∀ x ∃ y", 2)
[...]

julia> s[3]
ERROR: StringIndexError("∀ x ∃ y", 3)
Stacktrace:
[...]

julia> s[4]
' ': ASCII/Unicode U+0020 (category Zs: Separator, space)
```

In this case, the character `∀` is a three-byte character, so the indices 2 and 3 are invalid
and the next character's index is 4; this next valid index can be computed by [`nextind(s,1)`](@ref),
and the next index after that by `nextind(s,4)` and so on.

Extraction of a substring using range indexing also expects valid byte indices or an error is thrown:

```jldoctest unicodestring
julia> s[1:1]
"∀"

julia> s[1:2]
ERROR: StringIndexError("∀ x ∃ y", 2)
Stacktrace:
[...]

julia> s[1:4]
"∀ "
```

Because of variable-length encodings, the number of characters in a string (given by [`length(s)`](@ref))
is not always the same as the last index. If you iterate through the indices 1 through [`lastindex(s)`](@ref)
and index into `s`, the sequence of characters returned when errors aren't thrown is the sequence
of characters comprising the string `s`. Thus we have the identity that `length(s) <= lastindex(s)`,
since each character in a string must have its own index. The following is an inefficient and
verbose way to iterate through the characters of `s`:

```jldoctest unicodestring
julia> for i = firstindex(s):lastindex(s)
           try
               println(s[i])
           catch
               # ignore the index error
           end
       end
∀

x

∃

y
```

The blank lines actually have spaces on them. Fortunately, the above awkward idiom is unnecessary
for iterating through the characters in a string, since you can just use the string as an iterable
object, no exception handling required:

```jldoctest unicodestring
julia> for c in s
           println(c)
       end
∀

x

∃

y
```

Strings in Julia can contain invalid UTF-8 code unit sequences. This convention allows to
treat any byte sequence as a `String`. In such situations a rule is that when parsing
a sequence of code units from left to right characters are formed by the longest sequence of
8-bit code units that matches the start of one of the following bit patterns
(each `x` can be `0` or `1`):

* `0xxxxxxx`;
* `110xxxxx` `10xxxxxx`;
* `1110xxxx` `10xxxxxx` `10xxxxxx`;
* `11110xxx` `10xxxxxx` `10xxxxxx` `10xxxxxx`;
* `10xxxxxx`;
* `11111xxx`.

In particular this implies that overlong and too high code unit sequences are accepted.
This rule is best explained by an example:

```julia-repl
julia> s = "\xc0\xa0\xe2\x88\xe2|"
"\xc0\xa0\xe2\x88\xe2|"

julia> foreach(display, s)
'\xc0\xa0': [overlong] ASCII/Unicode U+0020 (category Zs: Separator, space)
'\xe2\x88': Malformed UTF-8 (category Ma: Malformed, bad data)
'\xe2': Malformed UTF-8 (category Ma: Malformed, bad data)
'|': ASCII/Unicode U+007c (category Sm: Symbol, math)

julia> isvalid.(collect(s))
4-element BitArray{1}:
 false
 false
 false
  true

julia> s2 = "\xf7\xbf\xbf\xbf"
"\U1fffff"

julia> foreach(display, s2)
'\U1fffff': Unicode U+1fffff (category In: Invalid, too high)
```

We can see that the first two code units in the string `s` form an overlong encoding of
space character. It is invalid, but is accepted in a string as a single character.
The next two code units form a valid start of a three-byte UTF-8 sequence. However, the fifth
code unit `\xe2` is not its valid continuation. Therefore code units 3 and 4 are also
interpreted as malformed characters in this string. Similarly code unit 5 forms a malformed
character because `|` is not a valid continuation to it. Finally the string `s2` contains
one too high code point.

Julia uses the UTF-8 encoding by default, and support for new encodings can be added by packages.
For example, the [LegacyStrings.jl](https://github.com/JuliaArchive/LegacyStrings.jl) package
implements `UTF16String` and `UTF32String` types. Additional discussion of other encodings and
how to implement support for them is beyond the scope of this document for the time being. For
further discussion of UTF-8 encoding issues, see the section below on [byte array literals](@ref man-byte-array-literals).
The [`transcode`](@ref) function is provided to convert data between the various UTF-xx encodings,
primarily for working with external data and libraries.

## 连接

One of the most common and useful string operations is concatenation:

```jldoctest stringconcat
julia> greet = "Hello"
"Hello"

julia> whom = "world"
"world"

julia> string(greet, ", ", whom, ".\n")
"Hello, world.\n"
```

It's important to be aware of potentially dangerous situations such as concatenation of invalid UTF-8 strings.
The resulting string may contain different characters than the input strings,
and its number of characters may be lower than sum of numbers of characters
of the concatenated strings, e.g.:

```julia-repl
julia> a, b = "\xe2\x88", "\x80"
("\xe2\x88", "\x80")

julia> c = a*b
"∀"

julia> collect.([a, b, c])
3-element Array{Array{Char,1},1}:
 ['\xe2\x88']
 ['\x80']
 ['∀']

julia> length.([a, b, c])
3-element Array{Int64,1}:
 1
 1
 1
```

This situation can happen only for invalid UTF-8 strings. For valid UTF-8 strings
concatenation preserves all characters in strings and additivity of string lengths.

Julia also provides [`*`](@ref) for string concatenation:

```jldoctest stringconcat
julia> greet * ", " * whom * ".\n"
"Hello, world.\n"
```

While `*` may seem like a surprising choice to users of languages that provide `+` for string
concatenation, this use of `*` has precedent in mathematics, particularly in abstract algebra.

In mathematics, `+` usually denotes a *commutative* operation, where the order of the operands does
not matter. An example of this is matrix addition, where `A + B == B + A` for any matrices `A` and `B`
that have the same shape. In contrast, `*` typically denotes a *noncommutative* operation, where the
order of the operands *does* matter. An example of this is matrix multiplication, where in general
`A * B != B * A`. As with matrix multiplication, string concatenation is noncommutative:
`greet * whom != whom * greet`. As such, `*` is a more natural choice for an infix string concatenation
operator, consistent with common mathematical use.

More precisely, the set of all finite-length strings *S* together with the string concatenation operator
`*` forms a [free monoid](https://en.wikipedia.org/wiki/Free_monoid) (*S*, `*`). The identity element
of this set is the empty string, `""`. Whenever a free monoid is not commutative, the operation is
typically represented as `\cdot`, `*`, or a similar symbol, rather than `+`, which as stated usually
implies commutativity.

## [Interpolation](@id string-interpolation)

Constructing strings using concatenation can become a bit cumbersome, however. To reduce the need for these
verbose calls to [`string`](@ref) or repeated multiplications, Julia allows interpolation into string literals
using `$`, as in Perl:

```jldoctest stringconcat
julia> "$greet, $whom.\n"
"Hello, world.\n"
```

This is more readable and convenient and equivalent to the above string concatenation -- the system
rewrites this apparent single string literal into a concatenation of string literals with variables.

The shortest complete expression after the `$` is taken as the expression whose value is to be
interpolated into the string. Thus, you can interpolate any expression into a string using parentheses:

```jldoctest
julia> "1 + 2 = $(1 + 2)"
"1 + 2 = 3"
```

Both concatenation and string interpolation call [`string`](@ref) to convert objects into string
form. Most non-`AbstractString` objects are converted to strings closely corresponding to how
they are entered as literal expressions:

```jldoctest
julia> v = [1,2,3]
3-element Array{Int64,1}:
 1
 2
 3

julia> "v: $v"
"v: [1, 2, 3]"
```

[`string`](@ref) is the identity for `AbstractString` and `AbstractChar` values, so these are interpolated
into strings as themselves, unquoted and unescaped:

```jldoctest
julia> c = 'x'
'x': ASCII/Unicode U+0078 (category Ll: Letter, lowercase)

julia> "hi, $c"
"hi, x"
```

To include a literal `$` in a string literal, escape it with a backslash:

```jldoctest
julia> print("I have \$100 in my account.\n")
I have $100 in my account.
```

## Triple-Quoted String Literals

When strings are created using triple-quotes (`"""..."""`) they have some special behavior that
can be useful for creating longer blocks of text.

First, triple-quoted strings are also dedented to the level of the least-indented line.
This is useful for defining strings within code that is indented. For example:

```jldoctest
julia> str = """
           Hello,
           world.
         """
"  Hello,\n  world.\n"
```

In this case the final (empty) line before the closing `"""` sets the indentation level.

The dedentation level is determined as the longest common starting sequence of spaces or
tabs in all lines, excluding the line following the opening `"""` and lines containing
only spaces or tabs (the line containing the closing `"""` is always included).
Then for all lines, excluding the text following the opening `"""`, the common starting
sequence is removed (including lines containing only spaces and tabs if they start with
this sequence), e.g.:
```jldoctest
julia> """    This
         is
           a test"""
"    This\nis\n  a test"
```

Next, if the opening `"""` is followed by a newline,
the newline is stripped from the resulting string.

```julia
"""hello"""
```

等价于

```julia
"""
hello"""
```

但是

```julia
"""

hello"""
```

will contain a literal newline at the beginning.

Stripping of the newline is performed after the dedentation. For example:

```jldoctest
julia> """
         Hello,
         world."""
"Hello,\nworld."
```

Trailing whitespace is left unaltered.

Triple-quoted string literals can contain `"` symbols without escaping.

Note that line breaks in literal strings, whether single- or triple-quoted, result in a newline
(LF) character `\n` in the string, even if your editor uses a carriage return `\r` (CR) or CRLF
combination to end lines. To include a CR in a string, use an explicit escape `\r`; for example,
you can enter the literal string `"a CRLF line ending\r\n"`.

## Common Operations

You can lexicographically compare strings using the standard comparison operators:

```jldoctest
julia> "abracadabra" < "xylophone"
true

julia> "abracadabra" == "xylophone"
false

julia> "Hello, world." != "Goodbye, world."
true

julia> "1 + 2 = 3" == "1 + 2 = $(1 + 2)"
true
```

You can search for the index of a particular character using the [`findfirst`](@ref) function:

```jldoctest
julia> findfirst(isequal('x'), "xylophone")
1

julia> findfirst(isequal('p'), "xylophone")
5

julia> findfirst(isequal('z'), "xylophone")
```

You can start the search for a character at a given offset by using [`findnext`](@ref)
with a third argument:

```jldoctest
julia> findnext(isequal('o'), "xylophone", 1)
4

julia> findnext(isequal('o'), "xylophone", 5)
7

julia> findnext(isequal('o'), "xylophone", 8)
```

You can use the [`occursin`](@ref) function to check if a substring is found within a string:

```jldoctest
julia> occursin("world", "Hello, world.")
true

julia> occursin("o", "Xylophon")
true

julia> occursin("a", "Xylophon")
false

julia> occursin('o', "Xylophon")
true
```

The last example shows that [`occursin`](@ref) can also look for a character literal.

Two other handy string functions are [`repeat`](@ref) and [`join`](@ref):

```jldoctest
julia> repeat(".:Z:.", 10)
".:Z:..:Z:..:Z:..:Z:..:Z:..:Z:..:Z:..:Z:..:Z:..:Z:."

julia> join(["apples", "bananas", "pineapples"], ", ", " and ")
"apples, bananas and pineapples"
```

Some other useful functions include:

  * [`firstindex(str)`](@ref) gives the minimal (byte) index that can be used to index into `str` (always 1 for strings, not necessarily true for other containers).
  * [`lastindex(str)`](@ref) gives the maximal (byte) index that can be used to index into `str`.
  * [`length(str)`](@ref) the number of characters in `str`.
  * [`length(str, i, j)`](@ref) the number of valid character indices in `str` from `i` to `j`.
  * [`ncodeunits(str)`](@ref) number of [code units](https://en.wikipedia.org/wiki/Character_encoding#Terminology) in a string.
  * [`codeunit(str, i)`](@ref) gives the code unit value in the string `str` at index `i`.
  * [`thisind(str, i)`](@ref) given an arbitrary index into a string find the first index of the character into which the index points.
  * [`nextind(str, i, n=1)`](@ref) find the start of the `n`th character starting after index `i`.
  * [`prevind(str, i, n=1)`](@ref) find the start of the `n`th character starting before index `i`.

## [Non-Standard String Literals](@id non-standard-string-literals)

There are situations when you want to construct a string or use string semantics, but the behavior
of the standard string construct is not quite what is needed. For these kinds of situations, Julia
provides [non-standard string literals](@ref). A non-standard string literal looks like a regular
double-quoted string literal, but is immediately prefixed by an identifier, and doesn't behave
quite like a normal string literal.  Regular expressions, byte array literals and version number
literals, as described below, are some examples of non-standard string literals. Other examples
are given in the [Metaprogramming](@ref) section.

## Regular Expressions

Julia has Perl-compatible regular expressions (regexes), as provided by the [PCRE](http://www.pcre.org/)
library. Regular expressions are related to strings in two ways: the obvious connection is that
regular expressions are used to find regular patterns in strings; the other connection is that
regular expressions are themselves input as strings, which are parsed into a state machine that
can be used to efficiently search for patterns in strings. In Julia, regular expressions are input
using non-standard string literals prefixed with various identifiers beginning with `r`. The most
basic regular expression literal without any options turned on just uses `r"..."`:

```jldoctest
julia> r"^\s*(?:#|$)"
r"^\s*(?:#|$)"

julia> typeof(ans)
Regex
```

To check if a regex matches a string, use [`occursin`](@ref):

```jldoctest
julia> occursin(r"^\s*(?:#|$)", "not a comment")
false

julia> occursin(r"^\s*(?:#|$)", "# a comment")
true
```

As one can see here, [`occursin`](@ref) simply returns true or false, indicating whether a
match for the given regex occurs in the string. Commonly, however, one wants to know not
just whether a string matched, but also *how* it matched. To capture this information about
a match, use the [`match`](@ref) function instead:

```jldoctest
julia> match(r"^\s*(?:#|$)", "not a comment")

julia> match(r"^\s*(?:#|$)", "# a comment")
RegexMatch("#")
```

If the regular expression does not match the given string, [`match`](@ref) returns [`nothing`](@ref)
-- a special value that does not print anything at the interactive prompt. Other than not printing,
it is a completely normal value and you can test for it programmatically:

```julia
m = match(r"^\s*(?:#|$)", line)
if m === nothing
    println("not a comment")
else
    println("blank or comment")
end
```

If a regular expression does match, the value returned by [`match`](@ref) is a `RegexMatch`
object. These objects record how the expression matches, including the substring that the pattern
matches and any captured substrings, if there are any. This example only captures the portion
of the substring that matches, but perhaps we want to capture any non-blank text after the comment
character. We could do the following:

```jldoctest
julia> m = match(r"^\s*(?:#\s*(.*?)\s*$|$)", "# a comment ")
RegexMatch("# a comment ", 1="a comment")
```

When calling [`match`](@ref), you have the option to specify an index at which to start the
search. For example:

```jldoctest
julia> m = match(r"[0-9]","aaaa1aaaa2aaaa3",1)
RegexMatch("1")

julia> m = match(r"[0-9]","aaaa1aaaa2aaaa3",6)
RegexMatch("2")

julia> m = match(r"[0-9]","aaaa1aaaa2aaaa3",11)
RegexMatch("3")
```

You can extract the following info from a `RegexMatch` object:

  * the entire substring matched: `m.match`
  * the captured substrings as an array of strings: `m.captures`
  * the offset at which the whole match begins: `m.offset`
  * the offsets of the captured substrings as a vector: `m.offsets`

For when a capture doesn't match, instead of a substring, `m.captures` contains `nothing` in that
position, and `m.offsets` has a zero offset (recall that indices in Julia are 1-based, so a zero
offset into a string is invalid). Here is a pair of somewhat contrived examples:

```jldoctest acdmatch
julia> m = match(r"(a|b)(c)?(d)", "acd")
RegexMatch("acd", 1="a", 2="c", 3="d")

julia> m.match
"acd"

julia> m.captures
3-element Array{Union{Nothing, SubString{String}},1}:
 "a"
 "c"
 "d"

julia> m.offset
1

julia> m.offsets
3-element Array{Int64,1}:
 1
 2
 3

julia> m = match(r"(a|b)(c)?(d)", "ad")
RegexMatch("ad", 1="a", 2=nothing, 3="d")

julia> m.match
"ad"

julia> m.captures
3-element Array{Union{Nothing, SubString{String}},1}:
 "a"
 nothing
 "d"

julia> m.offset
1

julia> m.offsets
3-element Array{Int64,1}:
 1
 0
 2
```

It is convenient to have captures returned as an array so that one can use destructuring syntax
to bind them to local variables:

```jldoctest acdmatch
julia> first, second, third = m.captures; first
"a"
```

Captures can also be accessed by indexing the `RegexMatch` object with the number or name of the
capture group:

```jldoctest
julia> m=match(r"(?<hour>\d+):(?<minute>\d+)","12:45")
RegexMatch("12:45", hour="12", minute="45")

julia> m[:minute]
"45"

julia> m[2]
"45"
```

Captures can be referenced in a substitution string when using [`replace`](@ref) by using `\n`
to refer to the nth capture group and prefixing the substitution string with `s`. Capture group
0 refers to the entire match object. Named capture groups can be referenced in the substitution
with `g<groupname>`. For example:

```jldoctest
julia> replace("first second", r"(\w+) (?<agroup>\w+)" => s"\g<agroup> \1")
"second first"
```

Numbered capture groups can also be referenced as `\g<n>` for disambiguation, as in:

```jldoctest
julia> replace("a", r"." => s"\g<0>1")
"a1"
```

You can modify the behavior of regular expressions by some combination of the flags `i`, `m`,
`s`, and `x` after the closing double quote mark. These flags have the same meaning as they do
in Perl, as explained in this excerpt from the [perlre manpage](http://perldoc.perl.org/perlre.html#Modifiers):

```
i   Do case-insensitive pattern matching.

    If locale matching rules are in effect, the case map is taken
    from the current locale for code points less than 255, and
    from Unicode rules for larger code points. However, matches
    that would cross the Unicode rules/non-Unicode rules boundary
    (ords 255/256) will not succeed.

m   Treat string as multiple lines.  That is, change "^" and "$"
    from matching the start or end of the string to matching the
    start or end of any line anywhere within the string.

s   Treat string as single line.  That is, change "." to match any
    character whatsoever, even a newline, which normally it would
    not match.

    Used together, as r""ms, they let the "." match any character
    whatsoever, while still allowing "^" and "$" to match,
    respectively, just after and just before newlines within the
    string.

x   Tells the regular expression parser to ignore most whitespace
    that is neither backslashed nor within a character class. You
    can use this to break up your regular expression into
    (slightly) more readable parts. The '#' character is also
    treated as a metacharacter introducing a comment, just as in
    ordinary code.
```

For example, the following regex has all three flags turned on:

```jldoctest
julia> r"a+.*b+.*?d$"ism
r"a+.*b+.*?d$"ims

julia> match(r"a+.*b+.*?d$"ism, "Goodbye,\nOh, angry,\nBad world\n")
RegexMatch("angry,\nBad world")
```

The `r"..."` literal is constructed without interpolation and unescaping (except for
quotation mark `"` which still has to be escaped). Here is an example
showing the difference from standard string literals:

```julia-repl
julia> x = 10
10

julia> r"$x"
r"$x"

julia> "$x"
"10"

julia> r"\x"
r"\x"

julia> "\x"
ERROR: syntax: invalid escape sequence
```

Triple-quoted regex strings, of the form `r"""..."""`, are also supported (and may be convenient
for regular expressions containing quotation marks or newlines).

## [Byte Array Literals](@id man-byte-array-literals)

Another useful non-standard string literal is the byte-array string literal: `b"..."`. This
form lets you use string notation to express read only literal byte arrays -- i.e. arrays of
[`UInt8`](@ref) values. The type of those objects is `CodeUnits{UInt8, String}`.
The rules for byte array literals are the following:

  * ASCII characters and ASCII escapes produce a single byte.
  * `\x` and octal escape sequences produce the *byte* corresponding to the escape value.
  * Unicode escape sequences produce a sequence of bytes encoding that code point in UTF-8.

There is some overlap between these rules since the behavior of `\x` and octal escapes less than
0x80 (128) are covered by both of the first two rules, but here these rules agree. Together, these
rules allow one to easily use ASCII characters, arbitrary byte values, and UTF-8 sequences to
produce arrays of bytes. Here is an example using all three:

```jldoctest
julia> b"DATA\xff\u2200"
8-element Base.CodeUnits{UInt8,String}:
 0x44
 0x41
 0x54
 0x41
 0xff
 0xe2
 0x88
 0x80
```

The ASCII string "DATA" corresponds to the bytes 68, 65, 84, 65. `\xff` produces the single byte 255.
The Unicode escape `\u2200` is encoded in UTF-8 as the three bytes 226, 136, 128. Note that the
resulting byte array does not correspond to a valid UTF-8 string:

```jldoctest
julia> isvalid("DATA\xff\u2200")
false
```

As it was mentioned `CodeUnits{UInt8,String}` type behaves like read only array of `UInt8` and
if you need a standard vector you can convert it using `Vector{UInt8}`:

```jldoctest
julia> x = b"123"
3-element Base.CodeUnits{UInt8,String}:
 0x31
 0x32
 0x33

julia> x[1]
0x31

julia> x[1] = 0x32
ERROR: setindex! not defined for Base.CodeUnits{UInt8,String}
[...]

julia> Vector{UInt8}(x)
3-element Array{UInt8,1}:
 0x31
 0x32
 0x33
```

Also observe the significant distinction between `\xff` and `\uff`: the former escape sequence
encodes the *byte 255*, whereas the latter escape sequence represents the *code point 255*, which
is encoded as two bytes in UTF-8:

```jldoctest
julia> b"\xff"
1-element Base.CodeUnits{UInt8,String}:
 0xff

julia> b"\uff"
2-element Base.CodeUnits{UInt8,String}:
 0xc3
 0xbf
```

Character literals use the same behavior.

For code points less than `\u80`, it happens that the
UTF-8 encoding of each code point is just the single byte produced by the corresponding `\x` escape,
so the distinction can safely be ignored. For the escapes `\x80` through `\xff` as compared to
`\u80` through `\uff`, however, there is a major difference: the former escapes all encode single
bytes, which -- unless followed by very specific continuation bytes -- do not form valid UTF-8
data, whereas the latter escapes all represent Unicode code points with two-byte encodings.

If this is all extremely confusing, try reading ["The Absolute Minimum Every
Software Developer Absolutely, Positively Must Know About Unicode and Character
Sets"](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/).
It's an excellent introduction to Unicode and UTF-8, and may help alleviate
some confusion regarding the matter.

## [Version Number Literals](@id man-version-number-literals)

Version numbers can easily be expressed with non-standard string literals of the form [`v"..."`](@ref @v_str).
Version number literals create [`VersionNumber`](@ref) objects which follow the
specifications of [semantic versioning](http://semver.org),
and therefore are composed of major, minor and patch numeric values, followed by pre-release and
build alpha-numeric annotations. For example, `v"0.2.1-rc1+win64"` is broken into major version
`0`, minor version `2`, patch version `1`, pre-release `rc1` and build `win64`. When entering
a version literal, everything except the major version number is optional, therefore e.g.  `v"0.2"`
is equivalent to `v"0.2.0"` (with empty pre-release/build annotations), `v"2"` is equivalent to
`v"2.0.0"`, and so on.

`VersionNumber` objects are mostly useful to easily and correctly compare two (or more) versions.
For example, the constant [`VERSION`](@ref) holds Julia version number as a `VersionNumber` object, and
therefore one can define some version-specific behavior using simple statements as:

```julia
if v"0.2" <= VERSION < v"0.3-"
    # do something specific to 0.2 release series
end
```

Note that in the above example the non-standard version number `v"0.3-"` is used, with a trailing
`-`: this notation is a Julia extension of the standard, and it's used to indicate a version which
is lower than any `0.3` release, including all of its pre-releases. So in the above example the
code would only run with stable `0.2` versions, and exclude such versions as `v"0.3.0-rc1"`. In
order to also allow for unstable (i.e. pre-release) `0.2` versions, the lower bound check should
be modified like this: `v"0.2-" <= VERSION`.

Another non-standard version specification extension allows one to use a trailing `+` to express
an upper limit on build versions, e.g.  `VERSION > v"0.2-rc1+"` can be used to mean any version
above `0.2-rc1` and any of its builds: it will return `false` for version `v"0.2-rc1+win64"` and
`true` for `v"0.2-rc2"`.

It is good practice to use such special versions in comparisons (particularly, the trailing `-`
should always be used on upper bounds unless there's a good reason not to), but they must not
be used as the actual version number of anything, as they are invalid in the semantic versioning
scheme.

Besides being used for the [`VERSION`](@ref) constant, `VersionNumber` objects are widely used
in the `Pkg` module, to specify packages versions and their dependencies.

## [Raw String Literals](@id man-raw-string-literals)

Raw strings without interpolation or unescaping can be expressed with
non-standard string literals of the form `raw"..."`. Raw string literals create
ordinary `String` objects which contain the enclosed contents exactly as
entered with no interpolation or unescaping. This is useful for strings which
contain code or markup in other languages which use `$` or `\` as special
characters.

The exception is that quotation marks still must be escaped, e.g. `raw"\""` is equivalent
to `"\""`.
To make it possible to express all strings, backslashes then also must be escaped, but
only when appearing right before a quote character:

```jldoctest
julia> println(raw"\\ \\\"")
\\ \"
```

Notice that the first two backslashes appear verbatim in the output, since they do not
precede a quote character.
However, the next backslash character escapes the backslash that follows it, and the
last backslash escapes a quote, since these backslashes appear before a quote.