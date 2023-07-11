+++
title = "Understanding `@extend`"
+++

`@extend` is a special rule in Sass that makes it easier to write base classes. Where in regular CSS you might have an element with the classes `"button button--error"`, using `@extend` you can reduce duplication by creating a base class `%button` and having `.button--error` extend it. This results in simpler code and smaller bundle sizes.

The implementation of `@extend` is surprisingly complex. It's by far one of the hardest parts of Sass compilation. A naive implementation wouldn't be _too_ hard, but the production implementation of `@extend` has been around [since 2010](https://github.com/sass/ruby-sass/commit/f918cf3c397b1b463752af3f8780028d5dbaea80) and over time has adopted a number of features, constraints, and optimizations. Sass tries its hardest to produce the smallest number of selectors possible, which is a non-trivial problem.

This post goes through a high level overview of the algorithms behind `@extend`. An understanding of Sass is not necessary, but some familiarity with HTML and CSS is expected. 

I'll walk through a high level description of the primitives necessary to implement `@extend`, and then explain how they can be combined together to get the final algorithm. The general outline of this is:
 - [Anatomy of a Selector](#anatomy-of-a-selector)
 - [Anatomy of a Single Extend](#anatomy-of-a-single-extend)
 - [Superselectors](#superselectors)
 - [Specificity](#specificity)
 - [Selector Unification](#selector-unification)
 - [Weave](#weave)
 - [Putting It All Together](#putting-it-all-together)

#### Anatomy of a Selector

The first primitive necessary to understand `@extend` is the CSS selector. Selector lists are composed of a series of complex selectors, which are themselves composed of compound selectors, and which finally are composed of simple selectors.

Simple selectors are the base atoms of a CSS selector. In Sass, they can be either:
 - id `#foo`
 - class `.foo`
 - attribute `[foo]`, with one of 6 operators (`[foo$=bar]`, etc.)
 - type `foo`, with an optional namespace. these are also sometimes called element selectors
 - pseudo (class and element) `:hover`/`::before`
 - universal `*`, with an optional namespace
 - the parent selector `&`
 - placeholder `%foo`

If you're familiar with CSS, the first 6 should already be familiar. The last 2 might only make sense if you're used to Sass. The parent selector, as the name implies, refers to the selector of the parent style rule. A selector will have a parent if it's [nested inside another rule](https://sass-lang.com/documentation/style-rules/#nesting). If the style rule is at the root, then this is `null`. Parent selectors are resolved prior to extension, so they're not necessary to understand `@extend`.

The placeholder selector is special in that it gets removed during compilation and will not show up in the resulting CSS. This is useful when combined with `@extend`, as it allows for the creation of base classes that can be extended but not show up in the CSS.

Compound selectors are composed of 1 or more simple selectors not separated by any other characters. They operate as an "and." For example, `.bar.foo` is a compound selector containing two simple class selectors, `.bar` and `.foo`. This selector will only match elements that have both classes.

Complex selectors are composed of 1 or more compound selectors joined together by combinators. Valid combinators are descendant (` `), next sibling (`+`), child (`>`), and following sibling (`~`). The semantics of these are relevant to some parts of extend, and can be found in the [MDN docs](https://developer.mozilla.org/en-US/docs/Learn/CSS/Building_blocks/Selectors/Combinators).

Finally, selector lists are composed of 1 or more comma separated complex selectors.

A bit more verbose phrasing of this can be found in the [CSS spec](https://drafts.csswg.org/selectors-4/#syntax). Sass, being a superset of CSS, inherits much of its syntax from the CSS spec.

#### Anatomy of a Single Extend

Extension can be thought of as a function that takes three arguments: the extendee, the extender, and the target. The extendee and extender are selector lists. The target is a simple selector.

When we apply extension from the extender to the extendee, we look for all instances of the target and intelligently replace the target with the extender such that the extender takes on the same semantics of the target. 

This phrasing is a bit dense, so let's look at a simple example of an extend:

```scss
a { // extendee
    color: red;
}

b { // extender
    @extend a;  // target
}
```

In this case, our extendee is `a`, our extender is `b`, and our target is `a`.

If we compile this:

```css
a, b {
  color: red;
}
```

We end up with a selector that matches both our extendee `a` and our extender `b`. This is pretty straightforward. Let's see what happens when we try extending a compound selector,

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

<!-- Extension in Sass knows a lot about selectors, and will never generate invalid ones. For example `#foo#bar` -->

#### Superselectors

Our next primitive is the concept of a superselector and a subselector. If you're familiar with [subtyping in other programming languages](https://doc.rust-lang.org/nomicon/subtyping.html), the vocabulary should be a bit familiar.

Selector `A` is a superselector of selector `B` if it matches at least all elements that `B` matches. `B` would then be considered a subselector of `A`.

Let's walk through a couple examples. We'll define a function, `is-superselector` that takes 2 selectors and returns whether the first selector is a superselector of the second.

```scss
is-superselector("a", "a")
// true
```

All selectors are superselectors of themselves. This should be pretty intuitive -- `a` matches all elements that are matched by `a`.

```scss
is-superselector("a.foo", "a")
// false
```

`a.foo` isn't a superselector of `a`, because it only matches `a` elements that have the class `foo`. But if we switch the arguments around,

```scss
is-superselector("a", "a.foo")
// true
```

`a` _is_ a superselector of `a.foo` because it matches all the elements that `a.foo` would.

<!-- todo: i don't think they do get the idea -->
We'll talk about a few more interesting cases before moving on:

```scss
is-superselector("a", "b")
// false
```

Two selectors that have no overlap can never be superselectors or subselectors of the other.

```scss
is-superselector("a b", "a > b")
// true
is-superselector("a b", "a + b")
// false
is-superselector("a b", "a ~ b")
// false
is-superselector("a > b", "a b")
// false
is-superselector("a + b", "a + b")
// true
```

This is where we have to start caring about the semantics of combinators. The interesting case here is that the descendant combinator (` `) is considered a superselector of the next child combinator (`>`) while the inverse isn't true. The other combinators don't have any interesting interactions, though they _are_ able to be superselectors of themselves.


Superselector calculations work on selector lists as well: `a, b` is a superselector of both `a` and `b`.

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

The style the browser chooses depends on the selector's [specificity](https://developer.mozilla.org/en-US/docs/Web/CSS/Specificity). In this case, the id selector `#foo` is the most specific, so the color of the element will be `orange`.

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
<!-- a.foo, .foo, a -->

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

For selectors that Sass generates, things are a bit different. Sass only guarantees that generated selectors will have at least the specificity of the _extender_. That is, if we have an input like

```scss
a {
    color: red;
}

a.foo {
    @extend a;
}
```

where our extender is `a.foo`, Sass guarantees that it will output a selector at least as specific as `a.foo`. Our result is

```css
a, a.foo {
  color: true;
}
```

which is quite similar to our previous example. Where things differ is when our extendee is more specific than our extender.

```scss
a, a.foo {
    color: red;
}

b {
    @extend a;
}
```

Here, our extender `b` has a specificity of `1`. When we execute this code for the most recent version of Sass, we get

```css
a, b, a.foo {
  color: true;
}
```

There's a pretty noticeable ommission of `b.foo` here. Although this affects the semantics of our styles, Sass is free to omit this selector as an optimization as it doesn't violate the guarantee that the generated selector will have a specificity of at least that of the extender.

#### Selector unification

The next primitive we need to introduce is the concept of selector unification.

When we "unify" two selectors, we create a new selector that matches only the elements that are matched by both selectors. This is like taking the "and" of two selectors.

Unification is fallible and will return `null` if it's not possible to represent the "and" of both selectors. For example, `selector-unify("#a", "#b)` will fail because `#a#b` would never match any elements.

As before, we'll walk through a couple examples to get an idea of how this primitive works.

```scss
selector-unify(".a", ".b")
```

This is a pretty simple example. To match both class selectors, we just concatenate them into `.a.b`. Most unifications of simple selectors end up just concatenating the two, unless they're a special case like two ids, two psuedo elements, two type selectors, or `*`.

In the case of `*`, for most combinations the result is just the other selector. You can think of it sort of like `true && X`. Our result is always just `X`. Things get a bit more complex when namespaces are involved, but we won't dive into that here.

Unification of complex selectors is a bit more.... complex :p

I think it's helpful if we revisit the semantics of unification. Our goal is to create one selector that combines 2 selectors, A and B, and which matches the elements matched by _both_ A and B.

If we look at an example like `.a .b`, semantically this is matching all `.b` elements with a `.a` parent. For the selector `.c .d`, the same is true -- we're selecting all `.d` elements with a `.c` parent.

If we want to unify these two selectors, what should the semantics of the resulting selector look like?

We'd want a selector that matches all child elements that are _both_ `.b` and `.d` and that have parents of `.a` and `.c`. Modelling the children is pretty easy -- we just unify the children selectors to get `.b.d`. The parents are a bit harder -- how do you model having both parents `.a` and `.c`. 

The solution is to emit all possible orderings of `.a` and `.c`. Specifically we would emit `.a .c`, `.c .a`, and also `.a.c` to account for the case in which a parent has both classes.

Putting this together, we get the final selector of `.a .c .b.d, .c .a .b.d, .a.c .b.d`. In practice, implementations of Sass omit the `.a.c` selector because it results in _much_ larger selectors for marginal gain.

Let's zoom into this algorithm to see how we'd go about generating this selector.

We start by looking at the last compound selector of both complex selectors and trying to unify them. We refer to this final value as the "base." If unification of the compound bases fails, the entire unification will fail as well.

Let's walk through a 2 examples of this:

```scss
selector-unify("a .foo", "a .bar")
```

Here, unification of the two base selectors succeeds and gives us `.foo.bar`, making our final unified selector `a .foo.bar`.

```scss
selector-unify("a #foo", "a #bar")
```

In this example, `#foo` and `#bar` can't be unified, and so the entire result is `null`.

Once we resolve and unify the bases, we need to unify the parent selectors. We refer the algorithm that does this as "weave."

##### Weave

The goal of `weave` is to generate all possible orderings of the parent selectors. Earlier we looked at a pretty simple example, but weaving can get pretty complex. In particular, we run into increased complexity when we have combinators other than descendant (`>`, `+`, or `~`), we have multiple parent selectors (e.g. `.a .b .c` and `.d .e .f`), or if the parents share a selector either with each other or the base.

Weaving maintains the invariant that the relative ordering of compound selectors within a given complex selector will remain the same. That is, if we are merging `.a .b .c` and `.d .e .f`, `.a` will always come before `.b` and `.d` will always come before `.e`. This should make sense intuitively -- if we swapped the order of `.a` and `.b`, we would be modifying the semantics of the original selector.

We'll start by looking at some examples of these more complex cases, and then we'll take a deeper look at the implementation.

The first complex case is when there are multiple parent selectors. Take this example:

```scss
selector-unify(".a .b .c", ".d .c");
```

Unification gives us `.a .b .d .c, .d .a .b .c`. We put the parent of selector 2 on both sides of selector 1.

Based on our explanation so far, you might imagine that Sass would attempt to interleave the `.d` selector between `.a .b`, producing the parent `.a .d .b`; however, Sass doesn't do this for the same reason it doesn't emit `.a.c` or `.b.c`.

For completeness, if we look at an example where _both_ selectors have multiple parent selectors,

```scss
selector-unify(".a .b .c", ".d .e .c");
```

we get `.a .b .d .e .c, .d .e .a .b .c`. Again, there's no interleaving of the selectors from the two parents. We simply change their ordering.

When the two selectors share a common compound selector, we can often emit a much smaller result. 

The case in which the first parents are the same should be pretty intuitive --

```scss
selector-unify(".a .b .c", ".a .d .c");
```

Here, we can maintain the semantics of both selectors by only permuting the middle compound selector. Our unification result is `.a .b .d .c, .a .d .b .c`. 

If the selectors instead share a middle compound selector, we can do a similar trick.

```scss
selector-unify(".a .b .c", ".e .b .c");
```

Our result is `.a .e .b .c, .e .a .b .c`. We keep `.b .c` the same between both complex selectors, but we permute `.a` and `.e`.

In both cases that we've seen here, not de-duplicating the selector would change the semantics of the unified result from that of the original selectors. In our first case, if Sass had emitted `.a .a .b .d .c`, the additional `.a` would not only be superfluous -- it would be incorrect. The original selectors being unified only required a single `.a` parent.

The next complex case to look at is when we have to worry about combinators.

Before discussing combinators further though, I do want to mention that Sass today is in the process of [changing how it handles invalid combinator sequences](https://github.com/sass/sass/issues/3340). For this reason I will be skipping over talking about parts of the existing algorithm that attempt to handle them gracefully.

Let's go back to the simplest unify example that we've looked at: `selector-unify(".a .c", ".b .c")`. Hopefully you remember what the unified result is -- `.a .b .c, .b .a .c`. This should make sense so far. But what happens if we insert a combinator into the mix?

```scss
selector-unify(".a > .c", ".b .c")
```

How should the result be updated to account for the combinator? 

It helps to look at the two complex selectors separately, `.a .b .c` and `.b .a .c`. For the latter, it's pretty straightforward; we can maintain the semantics by simply adding a combinator between `.a` and `.c`: `.b .a > .c`. 

For the former case, it's harder to reason about. Should the combinator go between the `.a` and `.b` or the `.b` and `.c`. The answer is actually neither -- Sass completely discards the second complex selector. Our final unified result is `.b .a > .c`.

This can be pretty surprising -- why do we throw away the second complex selector? The answer is that it's impossible to model the semantics of `.a > .c` while adding a parent in between. The same is true of all combinators other than descendant:  `>`, `+`, and `~`.

Things get even more complex when both selectors contain combinators other than descendant.

Let's start with the case in which the combinators on both sides are the same,

```scss
selector-unify(".a > .c", ".b > .c")
selector-unify(".a + .c", ".b + .c")
selector-unify(".a ~ .c", ".b ~ .c")
```

The semantics of the combinators here affects how we can unify the two selectors. For `>` and `+`, the only valid unification is one in which the parent of `.c` has both `.a` and `.b`. That is, our total result is `.a.b .c`.

The sibling combinator `~` is a less restrictive. The semantics of the unified selector should be that `.c` is a sibling of `.a` _and_ `.b`. If we express these semantics terms of selectors, our possible parents are `.a ~ .b`, `.b ~ .a`, and `.a.b`. This is very similar to how we do unification without combinators -- just with an additional `~`. Our final result would be `.a ~ .b ~ .c, .b ~ .a ~ .c, .b.a ~ .c`.

Unlike regular unification without combinators, Sass actually does emit the `.b.a` case here. This is because the likelihood of this parent selector occurring in the context of the `~` combinator is much higher than in the general case.

The case in which combinators are not the same is interesting as well:

```scss
selector-unify(".a > .c", ".b ~ .c")
selector-unify(".a > .c", ".b + .c")
selector-unify(".a + .c", ".b ~ .c")
```

For unification involving the `>` combinator, the result is simple. `.a` becomes the parent of the other selector: `.a > .b ~ .c` and `.a > .b + .c`.

When combining `+` and `~`, we have to take into account that both combinators refer to sibling elements, with `~` being a more general version of `+`. Like when there are two `~` combinators, we have to permute both parent selectors. But in the case that one of the combinators is a `+`, we can omit one of the permutations. 

<!-- todo: redundant or just not valid? -->
Our final selector would be `.b ~ .a + .c, .b.a + .c`. Note that we didn't include the `.a ~ .b` variant, because that would be redundant.

#### Putting it All Together

Now that we've covered the primitives of extend, we need to combine them together to get the actual algorithm.

Extension occurs during traversal of the Sass AST. When a selector node is encountered, it is registered in an "extension store." The extension store contains the state of all extensions of the current execution context. Namely, it maintains a mapping from simple selectors to the selector lists containing them and a mapping from simple selectors to the extensions containing them (i.e. the `@extend` rules having the simple selector as an extender). 

The extension store also keeps tracks of source specificity and the original selectors as declared by the source code. This is done in order to maintain the invariant that Sass doesn't elide selectors as declared by users.

When a selector is registered in the extension store, the mapping from simple selectors to the selectors containing them is populated. Then, if we have encountered any `@extend` rules up to this point, we apply them to the selector. We will have to update this selector again if we encounter further `@extend` rules during execution.

The process of applying an extension is where we get to use the primitives we discussed earlier.

#### Addendum

...

##### :not
##### :is/:matches/:where
##### :root
##### :has
##### Interactions with @media
##### !optional

<!-- We start by grouping our parent selectors based on combinators. We want to split sequences at the descendant (` `) selector. For example, `(A B > C D + E ~ > G)` would be grouped into `(A) (B > C) (D + E ~ > G)`.

This will help us in later steps to avoid having combinators in weird positions.

Then we find the longest common subsequence of groups between the two selectors. The equality function our LCS implemenation uses is pretty interesting. When we do traversal and go to determine if two groups are equal, we compute whether the two groups have a superselector/subselector relation. -->


 <!-- finding the base selector of all selector components.  -->

<!-- It's actually complex enough that we're going to take a pretty long detour to explain it. I promise we'll come back to complex selector unification. By the time we do so, we'll be pretty close to fully understanding all the edges of `@extend`. -->

<!-- #### The `:is` selector

I want to start by talking about the `:is` selector. Not in the context of how it's used in `@extend`, but rather how it works in regular CSS. This will give us a vocabulary to talk about some of the more complex parts of extension.

`:is` allows users to specify multiple selectors. For example, `:is(a, b) c` will match `b c` and `a c`. Taking this further, `:is(a, b) :is(c, d)` expands to `a c, a d, b c, b d`.

You might already see the similarities to `@extend`. The rest of this post will be devoted to discussing the manual implementation of expanding these `:is`-like selectors. -->

<!-- #### Weaving

We saw earlier that Sass generates all possible combinations of selectors. The algorithm for generating these combinations is referred to as "weave."  -->

<!-- https://github.com/sass/sass/issues/1807 -->
<!-- https://gist.github.com/nex3/7609394 -->
<!-- https://github.com/sass/sass/blob/main/spec/at-rules/extend.md -->
<!-- https://github.com/sass/sass/blob/main/accepted/extend-specificity.md -->
<!-- https://specificity.keegan.st/ -->




Sass notably makes an optimization by not emitting the `.a.b` parent selector in the unification of `.a .c` and `.b .c`. This is because 