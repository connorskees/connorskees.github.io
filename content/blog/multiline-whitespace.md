+++
title = "Multiline Statements in Whitespace-Sensitive Programming Languages"
+++

The [most requested Sass feature of all time](https://github.com/sass/sass/issues?q=sort%3Acomments-desc) is issue 216: [Multiline Expressions in Indented Syntax](https://github.com/sass/sass/issues/216).

As a side project, I've been going through the Sass feature request backlog and implementing the most requested features. The first of these is [support for first-class mixins](https://github.com/sass/sass/issues/626). Now that this feature has been merged and released, I'm taking a look at 216. 

My expectation here is that the actual implementation of the parsing will not be that complex. The hard part is coming up with a syntax that works and is intuitive for users. To this end, I'm interested in comparing how other whitespace-sensitive programming languages handle multiline statements and expressions.

Many languages, for example, allow a trailing `\` on a line to allow one to make it multiline. But some don't. 

The languages I will be looking at are:
- python
- ruby
- julia
- yaml
- haml
- nim
- sass

I'm interested in the following contexts:

 - support for `\` at the end of a line
 - `if` statement and `while` loop conditions
 - arithmetic -- e.g. 1 + 1 split over multiple lines, with the line being at different places
 - function declaration args
 - function call args
 - expressions inside parentheses -- e.g. (1) or (1+1)
 - array literals and expressions inside square brackets (`[]`)
 - string contents
 - string interpolation -- e.g. in python f strings, `f"{}"`
 - variable assignment, e.g. a = 1
 - imports -- e.g. `from math import ceil`
 - comments
 - dictionary literals and expressions inside curly braces `{}`
 - ternaries

Which is quite a lot. I think the most interesting cases will be those involving expressions inside square or curly brackets and parentheses.


###### The first case is `\` at the end of a line. 

```
>>> 1 + \
... 1
```

The following languages support such a construct:
 - python


###### Multiline If Expression:

Python:

```
>>> if 1 +
... 1:
```

The following languages support such a construct:
- a

The following languages do not:
 - python

###### Multiline While Condition:

```
>>> while 1 +
... 1:
```

###### Multiline Arithmetic:

```
>>> 1 +
... 1
```

###### Function Declaration Arguments:

```
def foo(
    a
): pass
```
```
def foo(a,
    b
): pass
```
```
def foo(
    a,
    b,
): pass
```

###### Expressions Inside Parentheses:

```
>>> (
...   1
... )
```

```
>>> (1 +
... 1)
```

###### Expressions Inside Square Brackets:

```
>>> [
...   1
... ]
```

```
>>> [1 +
... 1]
```

```
>>> [1 +
... 1,
... 2
... ]
```

###### String Contents:

```
>>> "
... "
```

###### Variable Assignment:

```
>>> a =
... 1
```