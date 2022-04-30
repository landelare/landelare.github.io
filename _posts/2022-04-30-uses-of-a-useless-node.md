---
title:  "Uses of a useless node"
excerpt: "Sometimes, doing nothing can be useful."
---

Today's node will appear useless at first (and is also trivial to implement in
BP):

![Do Nothing BP node](/images/k2_donothing.png)

```c++
UFUNCTION(BlueprintCallable,
          Meta = (DevelopmentOnly, CompactNodeTitle = "Do Nothing"))
static void DoNothing()
{
}
```

But sometimes, being explicit about nothing can be useful. Take this example:

![Chained BP branch nodes](/images/k2_chain_with_missing_false.png)

What it does is irrelevant, but at a glance a `False` branch is "obviously"
missing. In the middle of a frenzied debugging session you might think exactly
that and "fix" the problem by adding a similar latent node as the others, which
might appear correct but will break something later.

You could (and should) add a comment there, but in more ~~spaghetti~~complex 
situations it would not be obvious what the comment is referring to, which means
you need to type extra text which makes it even worse...

...or you could just be explicit that you intended to `Do Nothing` here, connect
it to the branch's `False` pin and avoid these issues altogether in a nice,
compact, self-documenting way.

It also has a few other uses:

* Comments can be attached to the node itself instead of a comment box
* These nodes can have breakpoints in BP and in C++ (if you write them in C++)
* You can add debug prints inside without having to replace every instance
* They come in multiple flavors! For example a similar `Should Not Happen` node
(that you can fill with an `ensure()` in C++ and/or `Print String` with a
breakpoint)
