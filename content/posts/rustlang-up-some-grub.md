+++
title = "Rustlang Up Some Grub at The Ten Top"
date = "2019-11-09"
author = "Blaine"
cover = ""
keywords = ["rustlang", "gamedev"]
description = "\"The Ten Top\" is the working title of a game that I'm writing in Rust using the [Amethyst](https://fixme.com) game development framework. My goals with the project are to make a fun casual simulation game in the vein of Game Dev Story and Overcooked."
showFullContent = false
+++

"The Ten Top" is the working title of a game that I'm writing in Rust using the [Amethyst](https://fixme.com) game development
framework. My goals with the project are to make a fun casual simulation game in the vein of Game Dev Story and Overcooked.

The object of the game is to make enough money to:

- Cover operating costs of the restaurant every month
- Pay the worker(s)
- Eventually sell the restaurant and retire

A core mechanic of the game is taking orders from customers (e.g "one hot dog, please") and asking workers to prepare the given
dish, then give it back to the customer.

To deliver a hot dog, a specific set of conditions need to be met:

- A hot dog bun needs to exist
- A hot dog link needs to exist
- A hot dog link needs to be cooked to create a cooked hot dog link
- A cooked hot dog link needs to be combined with a hotdog bun on a plate to create a servable hot dog

These steps result in a (bland) hot dog. A first attempt to code this logic might might include a lot of
hotdog-specific code ("Does a hot dog bun exist", "Does a hot dog link exist?", etc).

```rust
// This is rusty pseudocode!

// Get a collection of foods
let hot_dog_buns : Vec<Ingredient> = (&ingredients)
    .join()
    .filter(|| ingredient.type == Ingredients::HotDogBun )
    .collect();

let cooked_hot_dog_links : Vec<Ingredient> = (&ingredients)
    .join()
    .filter(|| ingredient.type == Ingredients::CookedHotDogLink )
    .collect();

if hot_dog_buns.len() > 1 {
    // A hot dog bun exists
    if cooked_hot_dog_links.len() > 1 {
        // A cooked hot dog link exists
        // OK, that's everything.
        // A hot dog can be made!
        make_hot_dog();
    } else {
        // Find a hot dog link and cook it
        // Try again next tick!
        cook_hot_dog_link();
    }
} else {
    // Waiting for a hot dog bun to exist!
    // Try again next tick?
}
```

If a hot dog bun and a cooked hot dog link, everything exists to make a hot dog. An issue is that "The Ten Top"
sells other kinds of food! Modelling all those cases could get tedious very quickly. Modelling this data with a dependency graph
helps to make the logic more managable. In the Rust ecosystem, [`petgraph`](https://fixme.com) is a popular tool for working
with dependency graphs.

To model this problem, dishes, ingredients, and actions can be treated as nodes in a directed graph:

```rust
use petgraph::{
    dot::{Config, Dot},
    graphmap::GraphMap,
    Directed,
    Direction
};

// Dishes represent completed orders that can be served to a customer
#[derive(Debug, PartialEq, Eq, Copy, Clone, PartialOrd, Ord, Hash)]
enum Dishes {
    HotDog
}

// Ingredients are base components that can be cooked
// or combined to create one or more dishes
#[derive(Debug, PartialEq, Eq, Copy, Clone, PartialOrd, Ord, Hash)]
enum Ingredients {
    HotDogBun,
    HotDogLink,
    HotDogLinkCooked
}

// Actions represent tasks to created cooked variants of ingredients
#[derive(Debug, PartialEq, Eq, Copy, Clone, PartialOrd, Ord, Hash)]
enum Actions {
    CookIngredient
}

// Each graph node needs to have a specific type.
// We can use a nested enum to model this
#[derive(Debug, PartialEq, Eq, Copy, Clone, PartialOrd, Ord, Hash)]
enum Food {
    Dishes(Dishes),
    Ingredients(Ingredients),
    Actions(Actions)
}

// Create an empty, mutable graph.
let mut graph = GraphMap::new();

// Add nodes for each action
graph.add_node(Food::Actions(Actions::CookIngredient));

// Add nodes for each hot dog ingredient
graph.add_node(Food::Ingredients(Ingredients::HotDogBun));
graph.add_node(Food::Ingredients(Ingredients::HotDogLink));
graph.add_node(Food::Ingredients(Ingredients::HotDogLinkCooked));

// Add a node for the hot dog dish
graph.add_node(Food::Dishes(Dishes::HotDog));

// A hot dog link needs to be cooked to make a cooked hot dog link
graph.add_edge(
    Food::Actions(Actions::CookIngredient),
    Food::Ingredients(Ingredients::HotDogLinkCooked),
    1.,
);

// A hot dog link is required to make a cooked hot dog link
graph.add_edge(
    Food::Ingredients(Ingredients::HotDogLink),
    Food::Ingredients(Ingredients::HotDogLinkCooked),
    1.,
);

// A cooked hot dog link is needed to make a hot dog
graph.add_edge(
    Food::Ingredients(Ingredients::HotDogLinkCooked),
    Food::Dishes(Dishes::HotDog),
    1.,
);

// A hot dog bun is needed to make a hot dog
graph.add_edge(
    Food::Ingredients(Ingredients::HotDogBun),
    Food::Dishes(Dishes::HotDog),
    1.,
);

// After adding nodes to the graph and creating edges between
// nodes, `petgraph` can be used to print a directed graph in GraphViz format

println!(
    "{:?}",
    Dot::with_config(&cookbook.graph, &[Config::EdgeNoLabel])
);
```

When run, this code prints:

```rust
digraph {
    0 [label="Dishes(HotDog)"]
    1 [label="Ingredients(HotDogBun)"]
    2 [label="Ingredients(HotDogLink)"]
    3 [label="Ingredients(HotDogLinkCooked)"]
    4 [label="Actions(Cook)"]
    3 -> 0
    1 -> 0
    2 -> 3
    4 -> 3
}
```

The graph can be represented visually:

![graphviz](https://user-images.githubusercontent.com/2590422/67796146-dd17cf00-fa55-11e9-95ea-272d88ec50b8.png)

Now that information about making hot dogs in in the graph, we can use the graph to find requirements for a given item.

```rust
// Ingredients needed to make a hot dog
let food_node = Food::Dishes(Dishes::HotDog);

let hot_dog_ingredients = graph
    .neigbhors_directed(food_node, Direction::Incoming);

for node in hot_dog_ingredients {
    dbg!(node);
}

// Prints
// node = Ingredients::HotDogLinkCooked
// node = Ingredients::HotDogBun
```

Once a vector of (what is the graph word for this??) exists, we can walk through each step.  
For example, if a cooked hot dog link exists, it can be used to make a hot dog. If not, we need to tell a worker to cook a hot dog.

Looking at neighbor nodes of a cooked hot dog link, we can get a sense of what goes into a cooked hotdog.

```rust
// Ingredients needed to make a hot dog
let food_node = Food::Ingredients(Ingredients::CookedHotDogLink);

let cooked_hot_dog_requirements = graph
    .neigbhors_directed(food_node, Direction::Incoming)
    .collect();

for node in cooked_hot_dog_requirements {
    dbg!(node);
}

// Prints
// node = Ingredients::HotDogLink
// node = Actions::HotDogBun
```
