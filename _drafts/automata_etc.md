---
title: "Automata Etc (not for publishing)"
layout: blog
---




The natural way to solve this problem in Rust might be to just use `Iterator`.
In fact, let's use our struct `Cave` above as an example.
If I just implement `IntoIterator` and `FromIterator` as detailed below

{% highlight rust %}
impl IntoIterator<Item = MooreNeighborhood<Tile>> for Cave {
    // Implementation is left as an exercise for the reader ;)
}

impl FromIterator<Tile> for Cave {
    // See above
}
{% endhighlight %}

We can then write the beautifully simple line of code below to generate our next generation.

{% highlight rust %}
cave.into_iter().map(|neighborhood| neighborhood.majority()).collect<Cave>()
{% endhighlight %}

If I were throwing together a weekend project, this is certainly how I would solve this problem.
However, there are some problems with this approach:

- It internally builds an iterator for the new generation before collecting it into a Cave.
This is terribly slow.

- It doesn't offer any flexibility for the neighborhood.
We chose to use the [Moore neighborhood](https://en.wikipedia.org/wiki/Moore_neighborhood) for this automaton.
What if, later, we wanted a more nuanced neighborhood for a more specific problem on the same data structure?


# Design

We can immediately alleviate


Should `Automaton` be a trait or a structure?

If it is a trait, then it should be parameterized by types describing the step function, and the neighborhood.
If these were simply functions to implement for the trait, users would only be able to implement one automaton per data structure.
This is too restrictive, as I have already outlined scenarios where the same data structure should be iteratively operated on by several automata.


