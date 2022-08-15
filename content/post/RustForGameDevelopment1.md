---
title: "Rust For Game Development #1 -- Rock Paper Scissors"
date: 2022-08-11T20:35:23-07:00
draft: false
tags: ["Rust", "Game Development"]
---

## Introduction

Games require two things: 
- Fast code

It needs to be fast enough to render graphics and simulate physics all within one game tick. It needs to be able to support more complex mechanics, AI, and networking (for multiplayer games). It needs to be performant on a large range of devices that may have suboptimal hardware like consoles...
- Safe code

It needs to support unloading and loading game objects frequently. It needs to load files from CDNs, disk, and cache. It needs to be able to not leak memory...

Rust has quickly been gaining popular not only in the software industry as a whole, but in game development for it's ability to fill all of these requirements.

Before jumping into writing a huge open-world MMO, it is important to start with the basics. We will be looking at how to implement one of the most simple games, Rock Paper Scissors.

## Project initialization

To ensure your computer has rust correctly installed, run

```
$ rustc --version
rustc 1.62.1 (e092d0b6b 2022-07-16)
```

If you do not see something similar to this, go visit [the rust lang book](https://doc.rust-lang.org/book/ch01-01-installation.html) and follow the correct instructions for your machine.

Now let's create a new package and call it rock_paper_scissors.

```
$ cargo new rock_paper_scissors
Created binary (application) `rock_paper_scissors` package
```

This will initalize a rust project that comes with many helpful tools for adding dependencies, compiling our packages, and distributing them.

## Project layout

Before writing any code, it is important to think about how the game rules work. Rock paper scissors is a strictly two person game with strictly three moves. There is also strictly three game outcomes. The three moves are:
- Rock
- Paper
- Scissors

The three game outcomes are:
- Player win
- Opponent win
- Tie

It is also important to think about what the logical flow of the game works as well.
There are 4 distinct steps in a successful rock paper scissors game:
1. Reading input from the player.
2. Generating a random input for the opponent.
3. Comparing the moves.
4. Displaying the winner.

Without further ado, let's get to implementing this.

## Implementation

### Reading input from the player

To keep this game as simple as possible, all interactions with this game will be done through the CLI. It is possible to write all the code to parse input, display help text, and so on, but, there is a widely used crate called [clap](https://crates.io/crates/clap). The great thing about this crate, is the easy of use to define the valid input.

To add this to our project, in the cargo.toml, add the clap dependecies.

```toml
[dependencies]
clap = { version = "3.2.14", features = ["derive"] }
```

It is important to add the feature [derive](https://docs.rs/clap/latest/clap/_derive/index.html) to make a CLI parser.


To make a CLI parser, we need to add a data structure to define what input we want, and variables to store it in. This can be done the derive feature added above. Additionally, I will include an optional attribute that reads metadata from cargo.toml. This is a nice to have for any user using the -help flag.

```rust
#[derive(Parser)]
#[clap(author, version, about, long_about = None)]
struct Cli {
    pattern: String,
    seed: Option<u64>
}
```

As seen above, an optional seed can be used to pass in a u64. This will be helpful in seeding the random move generator for our opponent.

With all of this done, we can finally init our parser.

```rust
// Initalize cli parser.
fn main() {
let args = Cli::parse();
}
```

Great, try putting a println! and running this to see it working correctly. It may not be intuitive to "run" this code with arguments, so I will provide a simple snippet on how.

```
$ cargo run -- <move> <optionalseed>
```

There is an issue with our code right now though. The move the user is providing is of type String. It is important to make some sort of data structure to store all of the potential moves and how they interact with one another. Additionally, it will be nice to convert from string to our data structure.

Let's make an enum called Move which will store the rock paper scissor move set.

```rust
#[derive(Debug, PartialEq)]
enum Move {
    Rock,
    Paper,
    Scissors
}
```

While we are at it, let's define the enum for storing game results.

```rust
#[derive(Debug, PartialEq)]
enum GameResult {
    UserWin,
    OpponentWin,
    Tie
}
```


It is important to derive both Debug and PartialEq as we will want to both print to the console the value of the enum (Debug) and compare moves and game results to other moves and game results (PartialEq).

Now that we have an enum to store moves, it is important to define how a string can be converted into a move. This can be done through implementing the from_str trait for our enum.

```rust
impl FromStr for Move {
    type Err = clap::ErrorKind;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        match s.to_ascii_lowercase().as_str() {
            "rock" => Ok(Move::Rock),
            "paper" => Ok(Move::Paper),
            "scissors" => Ok(Move::Scissors),
            _ => Err(clap::ErrorKind::InvalidValue),
        }
    }
}
```

It is up to you, what type of error you want to return.

Now that we can convert strings to a move, it is time to read a move from the user.
A good rusty way to do this, is to pass the string into the from_str method on our enum, and match the result with the desired outcome. If it is a valid move, store the move in a variable. If it is not, let the user know, and exit the program.

```rust
let our_move : Move = match Move::from_str(&args.pattern) {
    Ok(x) => x,
    Err(_) => {
        eprintln!("Invalid move");
        process::exit(1)
    }
};
```

Pattern matching is a powerful feature of rust as it enforces all code paths to be handled. Sometimes it may feel robust, but a successful compile becomes more powerful knowing all code paths are being explicitly handled.

We have now read input from the user!

### Generating a random input for the opponent

There is a crate called [rand](https://crates.io/crates/rand) that provides a way to create randomness in our application.

Just like before, inside cargo.toml, add the following line.

```toml
[dependencies]
. . .
rand = "0.8.5"
```

There is only two use cases we will use rand for: Randomness from a seed and randomness from no seed.

Using pattern matching on the optional seed argument, it is possible to ensure both options are explicitly defined. Our program needs a standard distrubution random number generator. Let's define it from a seed or from entropy.

```rust
let mut rng : StdRng = match args.seed {
    None => StdRng::from_entropy(),
    Some(x) => StdRng::seed_from_u64(x)
};
```

Before we generate a number and convert into a move, it would be redudant to do this explicitly every single time we want a random move. The trait we are implicitly using here is [distribution](https://docs.rs/rand/0.8.5/rand/distributions/index.html). We can implement this trait for type Move which would give us the ability to generate a random move from our random number generator.

```rust
impl Distribution<Move> for Standard {
    fn sample<R: Rng + ?Sized>(&self, rng: &mut R) -> Move {
        match rng.gen_range(0..=2) {
            0 => Move::Rock,
            1 => Move::Paper,
            _ => Move::Scissors,
        }
    }
}
```

Great. Let's generate a random move and display it to our user.
```rust
    // Generate a random move.
    let opponent_move : Move = rng.gen();
    // Let the user know what the move the opponent generated.
    print!("Opponent's move: {:?}. " , opponent_move);
```

Look how nice and clean that looks. By implementing traits on our enums, our core logic is neatly tucked away from our main methods.


### Comparing the moves

As stated above, to write clean modular code, traits are a great option. To compare our enum to itself, the most convenient trait to implement is PartialOrd. This will define the ordering between two moves. To use this for our enum, PartialEq (which we already added), needs to be defined for said enum. 

Let's write out how each move compares to one another.
```rust
impl PartialOrd for Move {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        match (self, other) {
            (Move::Rock, Move::Rock) => Some(Ordering::Equal),
            (Move::Rock, Move::Paper) => Some(Ordering::Less),
            (Move::Rock, Move::Scissors) => Some(Ordering::Greater),
            (Move::Paper, Move::Rock) => Some(Ordering::Greater),
            (Move::Paper, Move::Paper) => Some(Ordering::Equal),
            (Move::Paper, Move::Scissors) => Some(Ordering::Less),
            (Move::Scissors, Move::Rock) => Some(Ordering::Less),
            (Move::Scissors, Move::Paper) => Some(Ordering::Greater),
            (Move::Scissors, Move::Scissors) => Some(Ordering::Equal),
        }
    }
}
```

This allows us to compare moves with operators like '>' and '<'.

I think it looks better to write a method defining the conversion between the two moves' ordering and a game result. This could be possible through implementing a from_move trait for GameResult.

```rust
fn calculate_winner(user: Move, opponent:Move) -> GameResult {
    if user > opponent {
        return GameResult::UserWin
    } else if user == opponent {
        return GameResult::Tie
    } else {
        return GameResult::OpponentWin
    }
}
```

With all of this written, we can play a game of rock paper scissors. All that is left to do is call this function with our player's moves, and display this to the user.

### Displaying the winner

With the calculate_winner method, we can match on the returned value of GameResult.

```rust
    match calculate_winner(our_move, opponent_move) {
            GameResult::UserWin => println!("You win!"),
            GameResult::Tie => println!("Tie"),
            GameResult::OpponentWin => println!("You lose!"),
    }
```

Go try compiling and running this program and seeing if you can beat the computer! If you do not like losing, learn what move the computer will use for a seed.


## Conclusion

Rock paper scissors is a very simple game, but, by writing this out, it shows how some of rust's features can be used to write games. Imagine writing a card game. Having card values in an enum, and traits implemented to define the behavior between them all, the main functions of the game would be much easier to read. Additionally, using pattern matching ensures all code paths are accounted for.

There is a big part of this program missing. Testing. For the curious, if you navigate to the [repository](https://github.com/theseatoad/rock-paper-scissors) for this game, I have written test cases on our traits and gameplay to ensure the correct outcome it generated. 

Now go write the next greatest AAA hit ... or a card game. Either or, rust is a great option.