+++
title = "Sass Compilation"
+++


Sass is a programming language that compiles to CSS. It adds a number of additional features to CSS, like functions, loops, conditionals, and variables. 


#### Lexing and Parsing

Sass is a highly context-sensitive language. It inherits this partially from CSS, where you have to know whether you're parsing a style or a selector, but because of some of Sass's additional features, there's a huge amount of ambiguity.

Here's an example of where you need context in CSS parsing 

```css
a:hover {
    b:hover
}
```

We know `a:hover` is a selector because we're at the root of the stylesheet. In CSS, selectors can't be nested. We also know that `b:hover` is a style declaration, because we're inside a style rule.