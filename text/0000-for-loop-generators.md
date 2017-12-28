- Feature Name: for_loop_generators
- Start Date:
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Expand the `Generator` trait and extend `for` loop syntax to work with
generators rather than iterators.

# Motivation
[motivation]: #motivation

It's likely that Rust will end up with some form of generators, either via the
generators RFC or maybe through some kind of effects RFC. However the
generators currently on Rust nightly have two problems: Firstly, it's possible
to call `.resume` on a generator which has already returned. If we instead used
the type system to rule this out we could avoid a runtime check and make
generators failure-proof. Secondly, there's no way to pass an argument back to
the generator when resuming it. This RFC solves both these problems by
extending the `for` loop syntax in a natural way.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## The `Generator` and `IntoGenerator` traits

Rust has a built-in trait, `Generator` used for generators. It's definition is:

```rust
trait Generator {
    type Yield;
    type Resume;
    type Return;

    fn resume(self, resume: Self::Resume) -> GeneratorState<Self>;
}

// where
enum GeneratorState<G: Generator> {
    Yield(G, G::Yield),
    Return(G::Return),
}
```

Note that this `resume` method moves its argument, rather than taking it as a
mutable reference.

As of this RFC being implemented, `for` loops now take an argument which
implements `IntoGenerator` rather than `IntoIterator`. The `IntoGenerator` trait
is defined as:

```rust
trait IntoGenerator {
    type Yield;
    type Resume;
    type Return;
    type IntoGen: Generator<Yield = Self::Yield, Resume = Self::Resume, Return = Self::Return>,

    fn into_gen(self) -> Self::IntoGen;
}
```

For the sake of backwards-compatibility there is a blanket impl which converts
iterators into generators.

```rust
struct IterIntoGen(T);

impl<T: IntoIterator> IntoGenerator for T {
    type Yield = T::Item;
    type Resume = ();
    type Return = ();
    type IntoGen = IterIntoGen<T>;

    fn into_gen(self) -> IterIntoGen<T> {
        IterIntoGen(self.into_iter())
    }
}

impl<T: Iterator> Generator for IterIntoGen<T> {
    type Yield = T::Item;
    type Resume = ();
    type Return = ();

    fn resume(self, resume: ()) -> GeneratorState<IterIntoGen<T>> {
        match self.next() {
            Some(x) => GeneratorState::Yield(self, x),
            None => GeneratorState::Return(()),
        }
    }
}
```

## Extended `continue`

The first change towards integrating `for` loops with generators is that we
allow `continue` to take an argument. The argument to `continue` is the value
that the generator is resumed with. As with `break` and `return`, omitting the
argument on `continue` is syntactic sugar for `continue ()`. If a generator
resumes with anything other than `()` then `continue` is mandatory at the end
of the `for` loop block. For example

```rust
// gen resumes with `u32` ..
let gen: impl Generator<Resume = u32, ... >;
for x in gen {
    ...
    if ... {
        // .. so all branches must end with a continue
        continue 123;
    } else {
        // .. so all branches must end with a continue
        continue 456;
    }
}
```

Note that all of the above it backwards-compatible with today's iterators.

## `break` and `Return`

As of this RFC, `for` loops can `break` with a value. The type that they
`break` with must match the type that the `for` loop would evaluate to without
breaking. If a `for` loop reaches the end of the generator without breaking,
and it has no `then` clause (revealed soon), then it returns the value that the
generator returned.

```rust
// gen returns `char` ..
let gen: impl Generator<Return = char, ...>;
let x: char = for x in gen {
    ...
    // .. so we can (optionally) break with a `char`
    if .. {
        break 'a'
    }
}
```

## `then` clauses

For loops can have an optional additional `then` clause which executes if the
for loop reaches the end of the generator without breaking. The value returned
by this clause becomes the value of the entire `for` expression.

```rust
let x: bool = for i in 0..10 {
    ...
} then {
    true
}
```

The `then` clause can also take as an argument the value that the generator
returned. This is mandatory when the generator returns anything other than
`()`.

```rust
let gen: impl Generator<Return = String, ...>;
let x: usize = for x in gen {
    ...
} then s {
    s.len()
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[alternatives]: #alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
