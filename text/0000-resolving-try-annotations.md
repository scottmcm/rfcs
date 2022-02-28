- Feature Name: `annotated_try_blocks`
- Start Date: 2022-02-22
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Tweak the default behaviour of `?` inside `try{}` blocks to not depend on context,
in order to work better with methods and need type annotations less often.

Also adds a type-annotated syntactic variant for when the conversion behaviour is desired.

The stable behaviour of `?` when *not* in a `try{}` block is untouched.


# Motivation
[motivation]: #motivation

> I do have some mild other concerns about try block -- in particular it is
> frequently necessary in practice to give hints as to the try of a try-block.
>
> ~ [Niko commenting on #70941](https://github.com/rust-lang/rust/issues/70941#issuecomment-612167041)

---

The desugaring of `val?` currently works as follows, per RFC #3058:

```rust
match Try::branch(val) {
    ControlFlow::Continue(v) => v,
    ControlFlow::Break(r) => return FromResidual::from_residual(r),
}
```

Importantly, that's using a trait to create the return value.
And because the argument of the associated function is a generic on the trait,
it depends on inference to determine the correct type to return.

That works great in functions, because Rust's inference trade-offs mean that
the return type of a function is always specified in full.  Thus the `return`
always has complete type context, both to pick the return type as well as,
for `Result`, the exact error type into which to convert the error.

However, once things get more complicated, it stops working as well.  That's even
true before we start adding `try{}` blocks, since closures can hit them too.
(While closures behave like functions in most ways, their return types can be
left for type inference to figure out, and thus might not have full context.)

For example, consider this example of trying to use `Iterator::try_for_each` to
read the `Result`s from the `BufRead::lines` iterator:

```rust
use std::io::{self, BufRead};
pub fn concat_lines(reader: impl BufRead) -> io::Result<String> {
    let mut out = String::new();
    reader.lines().try_for_each(|line| {
        let line = line?; // <-- question mark
        out.push_str(&line);
        Ok(())
    })?; // <-- question mark
    Ok(out)
}
```

<!-- https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=c8171a07eb256392a6e392e0f940b171 -->

Though it looks reasonable, it doesn't compile:

```text
error[E0282]: type annotations needed
 --> src/lib.rs:7:9
  |
7 |         Ok(())
  |         ^^ cannot infer type for type parameter `E` declared on the enum `Result`
  |

error[E0283]: type annotations needed
 --> src/lib.rs:8:7
  |
8 |     })?; // <-- question mark
  |       ^ cannot infer type for type parameter `E`
  |
```

The core of the problem is that there's nothing to constrain the intermediate type
that occurs *between* the two `?`s.  We'd be happy for it to just be the same
`io::Result<_>` as in the other places, but there's nothing saying it *must* be that.
To the compiler, we might want some completely different error type that happens
to support conversion to and from `io::Error`.

The easiest fix here is the annotate the return type of the closure, as follows:

```rust
use std::io::{self, BufRead};
pub fn concat_lines(reader: impl BufRead) -> io::Result<String> {
    let mut out = String::new();
    reader.lines().try_for_each(|line| -> io::Result<()> { // <-- return type
        let line = line?;
        out.push_str(&line);
        Ok(())
    })?;
    Ok(out)
}
```

<!-- https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=a3870699c7df0df06eb720c036d94f0e -->

But it would be nice to have a way to request that "the obvious thing" should happen.

This same kind of problem happens with `try{}` blocks as they were implemented
in nightly at the time of writing of this RFC.  The desugaring of `?` in a `try{}`
block was essentially the same as in a function or closure, differing only in that
it "returns" the value from the block instead of from the enclosing function.

For example, this works great as it the type context available from the return type:

```rust
pub fn adding_a(x: Option<i32>, y: Option<i32>, z: Option<i32>) -> Option<i32> {
    Some(x?.checked_add(y?)?.checked_add(z?)?)
}
```

<!-- https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=0e383fab261dc7a39adf61975a307af3 -->

Suppose, however, that you wanted to do more in the method after the additions,
and thus added a `try{}` block around it:

```rust
#![feature(try_blocks)]
pub fn adding_b(x: Option<i32>, y: Option<i32>, z: Option<i32>) -> i32 {
    try { // pre-RFC version
        x?.checked_add(y?)?.checked_add(z?)?
    }
    .unwrap_or(0)
}
```

<!-- https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=e53dda532b44eecd6d1725143d04f900 -->

That doesn't compile, since a (non-trait) method call required the type be determined:

```text
error[E0282]: type annotations needed
 --> src/lib.rs:3:5
  |
3 | /     try { // pre-RFC version
4 | |         x?.checked_add(y?)?.checked_add(z?)?
5 | |     }
  | |_____^ cannot infer type
  |
  = note: type must be known at this point
```

This is, in a way, more annoying than the `Result` case.  Since at least there,
there's the possibility that one wants the `io::Error` converted into some
`my_special::Error`.  But for `Option`, there's no conversion for `None`.
While it's possible that there's some other type that accepts its residual,
the normal case is definitely that it just stays a `None`.

This RFC proposes using the unannotated `try { ... }` block as the marker to
request a slightly-different `?` desugaring that stays in the same family,
and add a new `try ☃ YourType { ... }` form for when you want conversions.

With that, the `adding_b` example just works.  And the earlier `concat_lines`
problem can be solved simply as

```rust
use std::io::{self, BufRead};
pub fn concat_lines(reader: impl BufRead) -> io::Result<String> {
    let mut out = String::new();
    reader.lines().try_for_each(|line| try { // <-- new version of `try`
        let line = line?;
        out.push_str(&line);
    })?;
    Ok(out)
}
```

(Note that this version also removes an `Ok(())`, as was decided in
[#70941](https://github.com/rust-lang/rust/issues/70941).)


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

<!--
Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.
-->

*Assuming this would go some time after [9.2](https://doc.rust-lang.org/stable/book/ch09-02-recoverable-errors-with-result.html)
in the book, which introduces `Result` and `?` for error handling.*

<!-- https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=24e0f7446f4980fcb1afa0c4b1ab08bf -->

So far all the places we've used `?` it's been fine to just return from the function on an error.  Sometimes, however,
it's nice to do a bunch of fallible operations, but still handle the errors from all of them before leaving the function.

One way to do that is to make a closure an immediately call it (an *IIFE*,
immediately-invoked function expression, to borrow a name from JavaScript):

```rust,edition2021,compile_fail
let pair_result = (||{
    let a = std::fs::read_to_string("hello")?;
    let b = std::fs::read_to_string("world")?;
    Ok((a, b))
})();
```

That's somewhat symbol soup, however.  And even worse, it doesn't actually compile
because it doesn't know what error type to use:
```text
error[E0282]: type annotations needed for `Result<(String, String), E>`
  --> src/lib.rs:28:9
   |
   |     let pair_result = (||{
   |         ----------- consider giving `pair_result` the explicit type `Result<(_, _), E>`, where the type parameter `E` is specified
...
   |         Ok((a, b))
   |         ^^ cannot infer type for type parameter `E` declared on the enum `Result`
```

Why haven't we had this problem before?  Well, when we're writing *functions*
we have to write the return type of the function down explicitly.  The `?` operator
in a function uses that to know to which error type it should convert any error is gets.
But in the closure, the return type is left to be inferred, and there are many possible answers,
so it errors because of the ambiguity.

This can be fixed by using a *try block* instead:

```rust,edition2021
let pair_result = try {
    let a = std::fs::read_to_string("hello")?;
    let b = std::fs::read_to_string("world")?;
    (a, b)
};
```

Here the `?` operator still does essentially the same thing -- either gives the value
from the `Ok` or short-circuits the error from the `Err` -- but with slightly
different details:

- Rather than returning the error from the function, it returns it from the `try` block.
  And thus in this case an error from either `read_to_string` ends up in the `pair_result` local.

- Rather than using the function's return type to decide the error type,
  it keeps using the same family as the type to which the `?` was applied.
  And thus in this case, since `read_to_string` returns `io::Result<String>`,
  it knows to return `io::Result<_>`, which ends up being `io::Result<(String, String)>`.

The trailing expression of the `try` block is automatically wrapped in `Ok(...)`,
so we get to remove that call too.  (Note to RFC readers: this decision is not part of this RFC.
It was previously decided in [#70941](https://github.com/rust-lang/rust/issues/70941).)

This behaviour is what you want in the vast majority of simple cases.  In particular,
it always works for things with just one `?`, so simple things like `try { a? + 1 }`
will do the right thing with minimal syntactic overhead.  It's also common to want
to group a bunch of things with the same error type.  Perhaps it's a bunch of calls
to one library, which all use that library's error type.  Or you want to do
[a bunch of `io` operations](https://github.com/rust-lang/rust/blob/d6f3a4ecb48ead838638e902f2fa4e5f3059779b/compiler/rustc_borrowck/src/nll.rs#L355-L367) which all use `io::Result`.  Additionally, `try` blocks work with
`?`-on-`Option` as well, where error-conversion is never needed, since there is only `None`.

It will fail to compile, however, if not everything shares the same error type.
Suppose we add some formatting operation to the previous example:

```rust,edition2021,compile_fail
let pair_result = try {
    let a = std::fs::read_to_string("hello")?;
    let b = std::fs::read_to_string("world")?;
    let c: i32 = b.parse()?;
    (a, c)
};
```

The compiler won't let us do that:

```text
error[E0308]: mismatched types
  --> src/lib.rs:14:32
   |
   |     let c: i32 = b.parse()?;
   |                           ^ expected struct `std::io::Error`, found struct `ParseIntError`
   = note: expected enum `Result<_, std::io::Error>`
              found enum `Result<_, ParseIntError>`
note: return type inferred to be `Result<_, std::io::Error>` here
  --> src/lib.rs:14:32
   |
   |     let a = std::fs::read_to_string("hello")?;
   |                                             ^
suggestion: annotate the `try` block with a type that supports both
   |
   | let pair_result = try ☃ YourTypeHere {
   |                       ^^^^^^^^^^^^^^
```

We can solve that by, as suggested, telling the `try` block which type we want it to use:

```rust,edition2021
let pair_result = try ☃ Result<_, Box<dyn std::error::Error>> {
    let a = std::fs::read_to_string("hello")?;
    let b = std::fs::read_to_string("world")?;
    let c: i32 = b.parse()?;
    (a, c)
};
```

In this type-annotated form, the `?` will again to error-conversion to the desired type,
the same way it does in functions.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

<!--
This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.
-->

## Grammar

Today on nightly, there's the following production:

*TryBlockExpression*: `try` *BlockExpression*

This RFC extends that to allow an optional type annotation:

*TryBlockExpression*: `try` ( `☃` *Type* )? *BlockExpression*

## Desugaring

Today on nightly, `x?` *inside a `try` block* desugars as follows,
after [RFC 3058](https://rust-lang.github.io/rfcs/3058-try-trait-v2.html):

```rust
match Try::branch(x) {
    ControlFlow::Continue(v) => v,
    ControlFlow::Break(r) => break 'try FromResidual::from_residual(r),
}
```

Where `'try` means the synthetic label added to the innermost enclosing `try` block.
(It's not something that can be mentioned in user code.)

This RFC continues to use that exact desugaring for *annotated* `try` blocks.
That means that any use in nightly today can be switched from `try { ... }`
to `try ☃ _ { ... }` and will continue to do exactly the same thing.

For *un*annotated `try` blocks, however, this RFC changes that desugaring to

```rust
fn make_try_type<T, R: Residual<T>>(r: R) -> <R as Residual<T>>::TryType {
    FromResidual::from_residual(r)
}

match Try::branch(x) {
    ControlFlow::Continue(v) => v,
    ControlFlow::Break(r) => break 'try make_try_type(r),
}
```

This still uses `FromResidual::from_residual` to actually create the value,
but determines the type to return from the argument via the `Residual` trait
rather than depending on having sufficient context to infer it.

## The `Residual` trait

This trait [already exists as unstable](https://doc.rust-lang.org/1.59.0/std/ops/trait.Residual.html),
so feel free to read its rustdoc instead of here, if you prefer.  It was added to support APIs like
[`Iterator::try_find`](https://doc.rust-lang.org/1.59.0/std/iter/trait.Iterator.html#method.try_find)
which also need this "I want a `Try` type from the same 'family', but with a different `Output` type" behaviour.

> ℹ As the author of this RFC, the details of this trait are not the important part of this RFC.
> I propose that, like was done for 3058, the exact details here be left as an unresolved question
> to be finalized after nightly experimentation.
> In particular, it appears that the [naming and structure related to `try_trait_v2`
> is likely to change](https://github.com/rust-lang/rust/issues/84277#issuecomment-1066120333),
> and thus the `Residual` trait will likely change as part of that.  But for now
> this RFC is written following the names used in the previous RFC.

```rust
pub trait Residual<V> {
    type TryType: ops::Try<Output = V, Residual = Self>;
}
```

### Implementations

```rust
impl<T, E> ops::Residual<T> for Result<convert::Infallible, E> {
    type TryType = Result<T, E>;
}

impl<T> ops::Residual<T> for Option<convert::Infallible> {
    type TryType = Option<T>;
}

impl<B, C> ops::Residual<C> for ControlFlow<B, convert::Infallible> {
    type TryType = ControlFlow<B, C>;
}
```


# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

<!--
- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
-->

Questions to be resolved in nightly:
- [ ] How exactly should the trait for this be named and structured?


# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
