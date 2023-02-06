+++
title = "Why Sass Compilation is Hard"
+++
<!-- date = "1970-01-01" -->


Sass is a programming language that compiles to CSS. It adds a number of additional features to CSS, like functions, loops, conditionals, and variables. 

#### What's a Compiler

Sass does a little of everything -- it's a compiler, a transpiler, and an interpreter. For the purposes of this article, I'll be referring to a program that translates Sass to CSS as a "Sass compiler," as this is how the official implementations refer to themselves.

#### Lexing and Parsing

Sass is a highly context-sensitive language. It inherits this partially from CSS, where you have to know whether you're parsing a style or a selector, but because of some of Sass's additional features, there's a huge amount of ambiguity.

Here's an example of where you need context in CSS parsing 

```css
a:hover {
    b:hover
}
```

We know `a:hover` is a selector because we're at the root of the stylesheet. In CSS, selectors can't be nested. We also know that `b:hover` is a style declaration, because we're inside a style rule.

In Sass, this isn't too much worse. We can use the same heuristic to know that `a:hover` is a selector, and then for `b:hover` all we have to do is parse until we encounter a semicolon or a curly brace. If we encounter a closing brace or a semicolon, it's a style, and if we encounter and open brace it's a selector. 

A bit more complex than regular CSS, but really not that bad. Now let's introduce another Sass feature: nested declarations. In Sass you can write style declarations like this,

```scss
a {
    -webkit: {
        foo: red;
        bar: blue;
        baz: green;
    }
}
```

and get CSS that looks like 

```css
a {
    -webkit-foo: red;
    -webkit-bar: blue;
    -webkit-baz: green;
}
```

You can see where it starts to get a bit more complex, but this also isn't so bad. We just have to check if our potential style or selector (`-webkit`) in this case ends with a colon. But that's not where this feature ends -- let's look at another example:

```scss
a {
    -webkit: a {
        foo: red;
        bar: blue;
        baz: green;
    }
}
```

which gives us

```css
a {
  -webkit: a;
  -webkit-foo: red;
  -webkit-bar: blue;
  -webkit-baz: green;
}
```

This complicates things even more,


#### `@extend` is complicated

I have a pretty in-depth explanation of `@extend` in Sass [already](understanding-extend) which describes how `@extend` works and should give an overview of why the implementation itself is challenging.

Right now I want to talk about the implications of `@extend` in incremental compilation. At any point in time, 

<!-- `@extend` is complex, and can at times be challenging to explain. We can start with an example:

```scss
%button-base {
    color: red;
    border: 1px solid black;
    
    &:hover {
        color: blue;
    }
}

.button {
    @extend %button-base;
    background-color: orange;
}

.button-error {
    @extend %button-base;
    background-color: red;
}
```

gets compiled into

```css
.button-error, .button {
  color: red;
  border: 1px solid black;
}
.button-error:hover, .button:hover {
  color: blue;
}

.button {
  background-color: orange;
}

.button-error {
  background-color: red;
}
``` -->


string operations are hard
 - \a - foo