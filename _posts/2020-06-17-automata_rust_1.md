---
title: "Crafting Cellular Automata in Rust (Part 1)"
layout: blog
---

![automaton]({{ 'assets/blog/automata_rust_1/header.png' | relative_url }}){: class="centered" }

In this series, we will explore cellular automata, both in theory and implementation.
We will begin by building a simple automaton in Rust, and then later iterate on that design in subsequent posts.
In the process, we will discuss various aspects of cellular automata, and how we might generalize our implementation to any automaton.

In this post, we will conceive a simple automaton to model forest fires.
We will then implement this model using our best Rusty practices, as a quick prototype to get our ideas in code.

First, though, what are cellular automata?


## What are cellular automata?

At a basic level, cellular automata are little machines that we set up according to some rules, and observe to see what happens when they run.
While they are fairly simple to describe, massively complex results can arise from them, making them both very useful, and interesting to study.



### Some terminology

Cell
: The basic unit of an automaton.
A cell contains one of a *finite* number of states.
Cells are arranged on a lattice, giving cellular automata their gridded appearance.

Neighborhood
: A finite set of cells near an initial cell.
In an automaton, we define what a "neighborhood" is for *any arbitrary cell*, so our concept of what a "neighborhood" is cannot vary from cell to cell within the automaton.

Transition Function
: A function defining change in the automaton.
The *local transition function* defines a change of state for a single cell, as a function of its neighborhood, while the *global transition function* defines an application of the local transition function to every cell in the automaton, simultaneously.

Step
: A single application of the global transition function, resulting in a new state for every cell in the automaton.


## What can cellular automata be used for?

### Modeling

Cellular automata are often used in modeling.
Specifically, when something tends to only change because of local factors, a cellular automaton may be very effective in modeling that process.
Some examples include [forest growth and wildfires](https://www.researchgate.net/publication/253486190_Simulations_of_Forest_Fires_by_the_Cellular_Automata_Model), [bacterial behavior](https://www.microbiologyresearch.org/content/journal/micro/10.1099/mic.0.26134-0), and [land use](https://www.youtube.com/watch?v=iSZJNPm2UPY).



### Cave generation

<video autoplay loop muted class="right">
  <source src="{{ "assets/blog/automata_rust_1/cave.webm" | relative_url }}" type="video/webm">
</video>

A common use case of cellular automata is for level generation in video games, particularly for cave structures, where each "cell" is either a wall tile, or floor tile.
For further reading on this use, I would suggest any one of the [many](https://gamedevelopment.tutsplus.com/tutorials/generate-random-cave-levels-using-cellular-automata--gamedev-9664) [blog](https://jeremykun.com/2012/07/29/the-cellular-automaton-method-for-cave-generation/) [posts](https://pvigier.github.io/2019/06/23/vagabond-dungeon-cave-generation.html) [on](https://codiecollinge.wordpress.com/2012/08/24/simple_2d_cave-like_generation/) [this](http://www.roguebasin.com/index.php?title=Cellular_Automata_Method_for_Generating_Random_Cave-Like_Levels).
A typical implementation for this method is to:

1. Generate some "path" of floor tiles, yielding a general idea of how the level should be laid out.

2. Flip the state on a certain number of random cells.

3. Run the "Voter's" automaton, where each cell becomes the majority state of its neighborhood.

The result of the automaton is an area with smoother walls, giving the impression that it has been generated organically.




## Modeling a forest fire

As stated above, we will be creating and implementing a cellular automaton to model forest fires in this post.
The goal here is not to have a completely accurate model, but to have one that *feels* right; one that seems to act like we imagine a forest fire to act.
We will develop and implement this automaton in parallel, and in the order of components listed above.
That is, we will define what a cell is, define our neighborhood, create a local transition function, and finally implement our global transition function.


### Defining a Cell

First, we need to decide the states of our automaton.
A forest, without a fire, can be simply composed of areas with, and without trees, so we will include the states "tree" and "ground" to denote this.
We would like to have differently sized trees, so we will actually have a number in the range of 1-5 representing each size of tree.
To represent the fire, make two states: "fire" and "smouldering", with the idea that a fire will progress to a smouldering state, which subsequently cannot burn any more.
The fire will progress by burning itself out, so we will also associate a number in the range of 1-5 with the fire state.
As code, we will represent it as:

``` rust
#[derive(Copy, Clone)]
enum ForestState {
    Tree(u8),
    Ground,
    Fire(u8),
    Smouldering
}

use ForestState::*;
```

### What is our neighborhood?

Now that we have defined our states, we need to start talking about how they change into one another in the local transition function.
However, first we need to define our neighborhood.

Since fire spreads via proximity, it makes sense to define our neighborhood based on proximity.
This is an excellent use case for the [Moore Neighborhood](https://en.wikipedia.org/wiki/Moore_neighborhood), which is defined as a single cell, and its eight closest neighbors.
We will define two things in our code for this: a type, and the actual relation of the neighborhood to a single cell.

``` rust
// with layout [nw, n, ne, w, c, e, sw, s, se]
type Neighborhood = [ForestState; 9];

const NEIGHBORHOOD: [(i64, i64); 9] = [
    (-1,  1),
    ( 0,  1),
    ( 1,  1),
    (-1,  0),
    ( 0,  0),
    ( 1,  0),
    (-1, -1),
    ( 0, -1),
    ( 1, -1)
];
```

### Creating a transition function

Now, we can think about how these states should change over time.
A fire spreads via proximity, but we can add some nuance to this process.
With sufficient nearby fire, a tree will burn.
Additionally, larger trees will ignite more easily, so the ability for a tree to catch fire is inversely proportional to its size.
Finally, we also know that a fire will burn longer on a larger tree.
As such, we will establish the following rules for our automaton:

1. A tree cell becomes a fire cell only if there are at least $$ 4 \cdot \text{size}^{-1} $$ nearby fire cells.

2. Any tree cell which catches on fire transfers its associated integer directly.

3. A fire cell always decrements its associated integer by 1, unless it is 1, at which point it becomes smouldering.

4. Smouldering and ground cells do not change.

These rules can be implemented as the local transition function `update()`.

``` rust
fn update(n: Neighborhood) -> ForestState {
    match n[4] {
        Tree(size) =>
            if n.iter().filter(
                    |x| match x { Fire(_) => true, _ => false }
                ).count() as f32 >= 4.0 / (size as f32) {
                Fire(size)
            } else {
                Tree(size)
            },
        Fire(1) => Smouldering,
        Fire(size) => Fire(size - 1),
        state => state
    }
}
```

### Bringing it all together

At this point, we still can't see the forest for the trees, as we have not yet defined a lattice for our cells.
We'll remedy this immediately, before we're logged down with more awful tree jokes.

``` rust
const LENGTH: usize = 30;
const WIDTH: usize = 15;

struct Forest([[ForestState; LENGTH]; WIDTH]);
```

At last, we can implement our global transition function.

For a moment, let's take a step back and think about what we're doing.
We are, at a high level, writing a function that takes a `Forest`, applies our local transition function `update()` onto every neighborhood of the forest, and returns the resulting `Forest`.
For now, let's think of this as an exercise in doing things the "Rusty" way.
That is, we will leverage some abstractions and do this like the closet Haskellers that we are.

What is our goal, then?

We will use an `Iterator` over every `Neighborhood` in the forest to map onto their corresponding `ForestState`.
We will then `collect()` these into a new `Forest`, and return that.
This is so easy, we can even write the function right here.

``` rust
fn step(forest: Forest) -> Forest {
    forest.into_iter().map(update).collect()
}
```


### Implementation details

Unfortunately, we live in a world where not all functions are pure, state exists, and `main` doesn't take and return every electron in the known universe.
As such, we need to write some *imperative* code to implement our beautiful iterators.
Avert your eyes if you must, I know I would.

``` rust
struct ForestIterator {
    forest: Forest,
    x: usize,
    y: usize
}

impl Iterator for ForestIterator {
    type Item = Neighborhood;

    fn next(&mut self) -> Option<Self::Item> {
        let Forest(forest) = self.forest;

        match (self.x, self.y) {
            // Waiting room for inclusive range patterns
            (0..=WIDTH, 0..=LENGTH) if (self.x, self.y) != (WIDTH, LENGTH) => {
                let mut neighborhood = [Ground; 9];

                for (i, (x, y)) in NEIGHBORHOOD.iter().enumerate() {
                    let x_p = x + self.x as i64;
                    let y_p = y + self.y as i64;

                    if x_p >= 0 && x_p < WIDTH as i64
                        && y_p >= 0 && y_p < LENGTH as i64 {
                        neighborhood[i] = forest[x_p as usize][y_p as usize];
                    }
                }

                self.x += 1;
                if self.x >= WIDTH {
                    self.x = 0;
                    self.y += 1;
                }
                Some(neighborhood)
            },
            _ => None
        }
    }
}

impl IntoIterator for Forest {
    type Item = Neighborhood;
    type IntoIter = ForestIterator;

    fn into_iter(self) -> Self::IntoIter {
        ForestIterator {
            forest: self,
            x: 0,
            y: 0
        }
    }
}

use std::iter::FromIterator;

impl FromIterator<ForestState> for Forest {
    fn from_iter<T>(iter: T) -> Self 
    where
        T : IntoIterator<Item = ForestState> {
        let mut forest = [[Ground; LENGTH]; WIDTH];
        let mut x = 0;
        let mut y = 0;

        for cell in iter {
            forest[x][y] = cell;
            x += 1;
            if x >= WIDTH {
                x = 0;
                y += 1;
                if y >= LENGTH {
                    break;
                }
            }
        }

        Forest(forest)
    }
}
```


### Generating and displaying a forest

As a last step before running the automaton, we need to write some utility functions for generating and displaying a forest.
These are relatively simple.

``` rust
extern crate rand;
use rand::{distributions::*, *};

impl Distribution<ForestState> for Standard {
    fn sample<R : Rng + ?Sized>(&self, rng: &mut R) -> ForestState {
        match rng.gen_range(0, 6) {
            0 => Ground,
            x => Tree(x)
        }
    }
}

use std::fmt;

impl fmt::Display for Forest {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        let Forest(forest) = self;

        for row in forest.iter() {
            for cell in row.iter() {
                let ch = match cell {
                    Ground      => ' ',
                    Smouldering => '.',
                    Fire(1..=2) => 'f',
                    Fire(_)     => 'F',
                    Tree(1..=2) => 't',
                    Tree(_)     => 'T'
                };

                write!(f, "{}", ch)?;
            }

            write!(f, "\n")?;
        }

        Ok(())
    }
}
```


### Running the automaton

Finally, we are able to use our automaton.
I added some ANSI escape magic to make it display better in the terminal.

``` rust
use std::{time, thread};

fn main() {
    let mut rng = rand::thread_rng();

    print!("{}[2J", 0x1b as char);
    print!("{}[?25l", 0x1b as char);

    // Grow the forest
    let mut forest: Forest = rng.sample_iter(Standard).take(LENGTH * WIDTH).collect();

    // Light a fire
    forest.0[rng.gen_range(0, WIDTH)][rng.gen_range(0, LENGTH)] = Fire(5);

    // Watch it burn
    loop {
        print!("{}[;H", 0x1b as char);

        println!("{}", forest);
        forest = step(forest);

        thread::sleep(time::Duration::from_millis(100));
    }
}
```

We finally have created our automaton, let's take it for a run!
If you're copying this code, don't worry if your cursor disappears.
Just run `bash -c 'echo -e "\033[?25h"'` and you'll have it back.

<video autoplay loop muted class="centered">
  <source src="{{ "assets/blog/automata_rust_1/final.webm" | relative_url }}" type="video/webm">
</video>

## Now what?

While making this automaton was exciting, and occasionally elegant, it wasn't without its costs.
There were several flaws with our approach to this problem:

1. We buffer the next forest in an iterator, transform that iterator into a forest, and then move that forest back into its original place.
We could skip the expensive iterator conversion if we simple buffered our forest with another forest.

2. Our approach only allows us to implement one concept of a neighborhood for the `Forest`, due to the limitations of `IntoIterator`.
If we wanted to use an automaton to generate the forest before simulating a fire, we would need to define a new data type for the forest.

3. On a broader scope, we make use of almost no optimizations.
There are several ways that automata can be sped up, but our approach is too abstracted to make much use of them.

We will explore an approach that tries to address these issues in the [next post]().

