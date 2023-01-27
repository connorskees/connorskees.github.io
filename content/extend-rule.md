+++
title = "Understanding `@extend`"
+++

<!-- date = "1970-01-01" -->

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

Complex selectors are composed of 1 or more compound selectors joined together by combinators. Valid combinators are descendant (` `), next sibling (`+`), child (`>`), and following sibling (`~`). The semantics of these are relevant to some parts of extend, and can be found in the [MDN docs](https://developer.mozilla.org/en-US/docs/Learn/CSS/Building_blocks/Selectors/Combinators).

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

Let's look at an example of extending a selector list to see what we mean by "intelligently replaces." This will be the last example we look at for now.

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

<!-- todo: below paragraph is bad. "smart" needs a better word. is the trimming part true? -->
Sass is smart and doesn't generate any redundant selectors here. Though, the way this is happening under the hood is that the redundant selector _is_ generated, it's just trimmed out during a separate pass. We'll discuss this later on.

Extension in Sass knows a lot about selectors, and will never generate invalid ones. For example `#foo#bar`

#### Superselectors

Our next primitive is the concept of a superselector and a subselector. If you're familiar with [subtyping in other programming languages](https://doc.rust-lang.org/nomicon/subtyping.html), the vocabulary should be a bit familiar.

Selector `A` is a superselector of selector `B` if it matches at least all elements that `B` matches. `B` would then be considered a subselector of `A`.

Let's walk through a couple examples. We'll define a function, `is-superselector` that takes 2 selectors and returns whether the first selector is a superselector of the second.

```scss
// true
is-superselector("a", "a")
```

All selectors are superselectors of themselves. This should be pretty intuitive -- `a` matches all elements that are matches by `a`.

```scss
// false
is-superselector("a.foo", "a")
```

`a.foo` isn't a superselector of `a`, because it only matches `a` elements that have the class `foo`. But if we switch the arguments around,

```scss
// true
is-superselector("a", "a.foo")
```

`a` _is_ a superselector of `a.foo` because it matches all the elements that `a.foo` would.

At this point you probably get the idea. We'll talk about a few interesting cases before moving on:

```scss
// false
is-superselector("a", "b")
```

Two selectors that have no overlap can never be superselectors or subselectors of the other.

```scss
// true
is-superselector("a b", "a > b")
// false
is-superselector("a b", "a + b")
// false
is-superselector("a b", "a ~ b")
// false
is-superselector("a > b", "a b")
```

With complex selectors, the semantics of the combinators come into play. The interesting case here is that the descendant combinator (` `) is considered a superselector of the next child combinator (`>`) while the inverse isn't true. The other combinators don't have any interesting interactions.


Superselector calculations work on selector lists as well: `a, b` is a superselector of `a` and `b`.

The universal selector (`*`) is a superselector of everything.

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

The exact algorithm is this (quoting [the spec](https://www.w3.org/TR/selectors-3/#specificity)):

> A selector's specificity is calculated as follows:
>
> - count the number of ID selectors in the selector (= a)
> - count the number of class selectors, attributes selectors, and pseudo-classes in the selector (= b)
> - count the number of type selectors and pseudo-elements in the selector (= c)
> - ignore the universal selector
>
> Concatenating the three numbers a-b-c (in a number system with a large base) gives the specificity.

This makes more sense if we walk through some examples. Take the selector `#foo.bar:hover::before *`. The specificity is calculated like this:

Our `a` value is the number of ID selectors, of which there is 1.  
We have 1 class selector and 1 pseudo class selector, so our `b` value is 2.  
There is 1 pseudo element selector, so our `c` value is 1.  

If we concatenate these 3 numbers together, we get a specificity value of 121.

The spec walks through a number of other examples:

```css
*               /* a=0 b=0 c=0 -> specificity =   0 */
LI              /* a=0 b=0 c=1 -> specificity =   1 */
UL LI           /* a=0 b=0 c=2 -> specificity =   2 */
UL OL+LI        /* a=0 b=0 c=3 -> specificity =   3 */
H1 + *[REL=up]  /* a=0 b=1 c=1 -> specificity =  11 */
UL OL LI.red    /* a=0 b=1 c=3 -> specificity =  13 */
LI.red.level    /* a=0 b=2 c=1 -> specificity =  21 */
#x34y           /* a=1 b=0 c=0 -> specificity = 100 */
#s12:not(FOO)   /* a=1 b=0 c=1 -> specificity = 101 */
```

An interesting bit here is that the `:not(..)` pseudo class takes on the specificity of the selector that it contains and itself does not have any bearing on the calculation.

Sass makes some pretty strict guarantees about selector specificity.

First, it differentiates between user provided selectors and selectors that Sass generates through executing `@extend`.

Sass will never alter the specificity of user provided selectors. For example, it we take a stylesheet like this:

<!-- arg, target, extender -->
<!-- a.foo, .foo, .a -->

```scss
a.foo {
    color: red;
}

a {
    @extend .foo;
}
```

What should the resulting selector be? If we were to ignore specificity, we'd probably expect the result to just be `a`. This is because `a` is a superselector of `a.foo`, so the semantics of `a, a.foo` would be the same as `a`. 

However, the specificity of the selector plays into the semantics. If we were to treat `a.foo` as redundant and only emit `a`, that would leave us with a lower specificity than what the programmer originally wrote.

The solution here is to emit `a, a.foo` which maintains the semantics and specificity of both the original selector as written and the extension. 

#### Selector unification

<!-- https://gist.github.com/nex3/7609394 -->
<!-- https://github.com/sass/sass/blob/main/spec/at-rules/extend.md -->
<!-- https://github.com/sass/sass/blob/main/accepted/extend-specificity.md -->
