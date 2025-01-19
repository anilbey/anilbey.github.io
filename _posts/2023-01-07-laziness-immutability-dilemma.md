---
title: Laziness-immutability dilemma
date: 2023-01-07 21:34:55 +/-TTTT
categories: [programming languages]
tags: [rust, programming-languages, functional-programming]     # TAG names should always be lowercase
---

I had never thought laziness and immutability could be conflicting concepts until I started programming in Rust.
I learned that there is a laziness-immutability dilemma not only in Rust but also in other languages. I will explain the dilemma in this post.

## The problem

I was solving the Advent of Code 2022 challenges to learn Rust. The Advent of Code questions have a good coverage of various programming concepts and thus good for learning a new language.

In one of the challenges I was trying to read some input data line by line in a lazy fashion using this code.

```rust

    let input = std::fs::read_to_string("input.txt").unwrap();
    let lines = input.lines();
    let mut line_iter = lines.peekable();

    while line_iter.peek().is_some() {
        // Process the line
    }
```

Here although the code was working, I was not happy with it because of the use of `mut` keyword.
I did not put the `mut` keyword at first, but the compiler was complaining about it. I wanted an immutable iterator that just "peeks" into the value in a read-only fashion and I wanted to declare that iterator as immutable to guarantee there will be no side-effects caused by that iterator.

> In Rust, all variables and references are immutable unless otherwise specified.
{: .prompt-info }



The `peek` method was supposed to just "peek" into the iterator and not to consume it - as its name suggests.
In the documentation however, it is stated that the `peek` method mutably borrows the `self` argument.

Here is the implementation of the `peek` method from its [source](https://doc.rust-lang.org/stable/src/core/iter/adapters/peekable.rs.html#214):

```rust
#[inline]
#[stable(feature = "rust1", since = "1.0.0")]
pub fn peek(&mut self) -> Option<&I::Item> {
    let iter = &mut self.iter;
    self.peeked.get_or_insert_with(|| iter.next()).as_ref()
}
```

I was so surprised to see it was implemented to mutate the struct. Moreover, the [documentation](https://doc.rust-lang.org/stable/std/iter/struct.Peekable.html#method.peek) of the `peek` method says the following.

> Returns a reference to the next() value without advancing the iterator.

It says, without advancing the iterator. Meaning without modifying. In other words, meaning immutability. Yet it takes the self argument as mutable.

If I were to implement a peek function, I would have implemented it immutably. I was thinking about what made them implement a peek function to mutate the struct. Besides, in the documentation, I noticed there exists a `peek_mut` method that made me wonder even more about the overall design decisions.

I asked this StackOverflow question, namely [Why does std::iter::Peekable::peek mutably borrow the self argument?](https://stackoverflow.com/questions/74841526/why-does-stditerpeekablepeek-mutably-borrow-the-self-argument) to get some answers and I got an enlightening answer from user [finomnis](https://stackoverflow.com/users/2902833/finomnis).


## The Explanation

Short answer
: you need to sacrifice immutability for the sake of laziness.


Long answer
: let's see how exactly laziness is preventing us from using immutable references by reading the source code. Check [finomnis's answer](https://stackoverflow.com/a/74841610/1935611) for the full explanation.

Below are the relevant parts of the source code to the `peek` method.

```rust
pub struct Peekable<I: Iterator> {
    iter: I,
    /// Remember a peeked value, even if it was None.
    peeked: Option<Option<I::Item>>,
}
```
_Notice that `Peeakable` struct has a field to store the peeked value_

```rust
impl<I: Iterator> Iterator for Peekable<I> {
    // ...

    fn next(&mut self) -> Option<I::Item> {
        match self.peeked.take() {
            Some(v) => v,
            None => self.iter.next(),
        }
    }

    // ...
}
```
_Implementation of the `next` method_


```rust
impl<I: Iterator> Peekable<I> {
    // ...

    pub fn peek(&mut self) -> Option<&I::Item> {
        let iter = &mut self.iter;
        self.peeked.get_or_insert_with(|| iter.next()).as_ref()
    }

    // ...
}
```
_Implementation of the peek method_


What happens here is that when `peek` gets called, it checks if `self.peeked` contains a value.

* If yes, it returns that value.

* If no, it calls the `next` method (hence **mutates** the iterator) to get the next value and stores it in `self.peeked` for later use.

So independent of the number of times it is called in a sequence (without explicitly calling `next` in between), peek only iterates the underlying iterator once. Call it once or 20 times, it only iterates the underlying iterator once.

When `next` gets called, it also checks if the `self.peeked` contains a value.

* If yes, it returns that value and removes that value from `self.peeked` (notice that the iterator is not advanced). This means, the first time `next` is called after `peek`, it returns the previously retrieved peeked value. The second time it gets called (without another `peek` method call in the meantime), it just iterates the underlying iterator.

* If no, it calls the `next` method of the underlying iterator to get the next value.

So if the user never uses `peek`, `next` just iterates the underlying iterator the number of times it is called.

When `peek` gets called before `next`, since `peek` already did a single iteration, `next` does not do another. This is how laziness is achieved. This is the reason why `peek` requires a mutable reference.

I mentioned the `peek_mut` at the beginning of the post.
Both `peek` and `peek_mut` take a mutable reference to the `self` argument.
The only difference between `peek` and `peek_mut` is their [return values](https://doc.rust-lang.org/stable/std/iter/struct.Peekable.html#method.peek_mut).

* `peek` returns a reference to the `next()`
* `peek_mut` returns a mutable reference to the `next()`


## Beyond Rust

This issue might have gone unnoticed or could potentially cause more trouble in another programming language.
Rust's design decision on explicitly declaring the mutability information clearly pinpoints to the issue. This is one aspect I really like in Rust. It teaches you core concepts. It forces you to think about the design decisions and the trade-offs. Once you learn to think about these concepts, you use them in any language you write.

> Rust's borrowing rules enforce you to have either a single mutable reference or multiple immutable references to a value at a lifetime.
{: .prompt-info }

After having practiced these borrowing rules, I reckon people will think twice or thrice before designing a system that allows multiple mutable references to a value at a lifetime.
