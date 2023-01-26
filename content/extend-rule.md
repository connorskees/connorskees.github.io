+++
title = "Understanding `@extend`"
+++

<!-- date = "1970-01-01" -->

<!-- https://gist.github.com/nex3/7609394 -->

#### Anatomy of a Selector

The first primitive we want to work with is the selector. Selectors are composed of a series of complex selectors, which are themselves composed of compound selectors, which finally are composed of simple selectors.

Simple selectors are the base atoms of a selector. They can be either
 - id `#foo`
 - class `.foo`
 - attribute `[foo]`, with one of 6 operators
 - type `foo`
 - pseudo (class and element) `:hover`/`::before`
 - universal `*`, with an optional namespace
 - the parent selector `&`
 - placeholder `%foo`

If you're familiar with CSS, the first 6 should look pretty familiar. The last 2 might only make sense if you're used to Sass. The parent selector, as the name implies, refers to the selector of the parent style rule. If the style rule is at the root, then this is `null`. Parent selectors are resolved prior to extension, so they aren't super important for discussing `@extend`.

The placeholder selector is special in that it gets removed during compilation and will not show up in the resulting CSS. This is useful when combined with `@extend`, as it allows for the creation of base classes that can be extended but not show up in the CSS.

Compound selectors are composed of 1 or more simple selectors not separated by any other characters. They operate as an "and." For example, `.bar.foo` is a compound selector containing two simple class selectors, `.bar` and `.foo`. This selector will only match elements that have both classes.

Complex selectors are composed of 1 or more compound selectors joined together by combinators. Valid combinators are descendant (` `), next sibling (`+`), child (`>`), and following sibling (`~`). The semantics of these aren't relevant to extend, but they can be found in the [MDN docs](https://developer.mozilla.org/en-US/docs/Learn/CSS/Building_blocks/Selectors/Combinators).

Finally, selector lists are composed of 1 or more comma separated complex selectors. A bit more verbose phrasing of this can be found in the [CSS spec](https://drafts.csswg.org/selectors-4/#syntax). Sass, largely being a superset of CSS, inherits much of its syntax from the CSS spec.

#### Anatomy of a Single Extend

Extension has three arguments: the extendee, the extender, and the target. The extendee and extender are selector lists. The target is a simple selector. When we apply extension from the extender to the extendee, we look for all instances of the target and intelligently replace the target with the extender such that the extender takes on the same semantics of the target. 

Let's look at a simple example of an extend,

```scss
a {
    color: red;
}

b {
    @extend a;
}
```

In this case, our extendee is `a`, our extender is `b`, and our target is `a`.

If we compile this,

```css
a, b {
  color: red;
}
```

we end up with a selector that matches both our extendee `a` and our extender `b`. This is pretty straightforward. Let's see what happens when we try extending a compound selector,

```scss
a:hover {
    color: red;
}

b {
    @extend a;
}
```

Here, our extendee is `a:hover`, our extender is `b`, and our target is `a`.

Our result is

```css
a:hover, b:hover {
  color: red;
}
```

which translates the semantics of `a:hover` to our extender `b`.

Things get a bit more interesting if we try extending complex selectors,

```scss
a a {
    color: red;
}

b {
    @extend a;
}
```

Here, our extendee is `a a`, our extender is `b`, and our target is `a`. This example is interesting because now we're extending a selector that has multiple instances of our target. How does Sass handle this?

```css
a a, b a, a b, b b {
  color: red;
}
```

A combinatorial explosion. We generate all possible combinations of `a` and `b`. If we try making our extendee the character `a` repeated 15 times, we end up with over a megabyte in just selectors for our output. But, it does accurately translate the semantics of every instance of our target `a` to our extender `b`.

Let's look at an example of extending a selector list to see what we mean by "intelligently replaces." This will be the last example we look at.

```scss
a, b {
    color: red;
}

b {
    @extend a;
}
```

Here, our extendee is `a, b`, our extender is `b`, and our target is `a`. This case is interesting because our extendee already has the semantics for our extender that we would want. How does Sass handle this?

```css
a, b {
  color: red;
}
```

Sass is smart and doesn't generate any redundant selectors here. Though, the way this is happening under the hood is that the redundant selector _is_ generated, it's just trimmed out during a separate pass. We'll discuss this later on.

Extension in Sass knows a lot about selectors, and will never generate invalid ones. For example `#foo#bar`


##### Specificity

The next primitive we need to introduce is selector specificity. Consider the following CSS,

```css
* {
    color: red;
}

a {
    color: green;
}

#foo {
    color: orange;
}

.bar {
    color: blue;
}
```

When we apply this stylesheet to the HTML `<a id="foo" class="bar" ... />`, what should the color be?

We can look at the [CSS specificity docs](https://developer.mozilla.org/en-US/docs/Web/CSS/Specificity) on MDN to see. In this case, the id selector `#foo` is the most specific, so the color of the element will be `orange`.


When Sass executes `@extend`, it will not generate selectors that have a lower specificity than the selector being extended. That is, say we have a Sass stylesheet like this,

```scss
a.foo {
    color: red;
}

.a {
    @extend .foo;
}
```


#### Superselectors


#### Selector unification