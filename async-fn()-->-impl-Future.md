# Summary
[summary]: #summary

Transition `#[async]` functions from: being written with a `Result` return type
that gets rewritten inside the macro to `impl Future`; to: being written with an
`impl Future` return type directly. Also support `impl Stream`, `Box<Future>`,
`impl StableFuture` etc. (potentially also any `CoerceUnsized` if possible?).

# Motivation
[motivation]: #motivation

## Tie the written signature more directly to the final signature

When browsing the documentation, then moving to the source code it is confusing
to see different signatures. By specifying the expected signature directly this
will be avoided.

## Reduce friction for migrating to `#[async]`

If a library is planning on using `#[async]` in the future then they can prepare
today by declaring all their functions `-> impl Future` and returning explicitly
implemented futures. They will then simply have to add the `#[async]` attribute
and rewrite the body, whereas today they would be forced to change the signature
as well:

```diff
+#[async]
+fn fetch_rust_lang(client: hyper::Client) -> io::Result<String> {
-fn fetch_rust_lang(client: hyper::Client) -> impl Future<Item=String, Error=io::Error> {
```

## Forward compatibility with nominal existential types

```rust
pub trait CountDown {
  type Future<'a>: Future<Item = (), Error = !> + 'a;

  fn start(&mut self, count: Duration) -> Self::Future<'_>;
}

impl CountDown for Timer {
  abstract type Future<'a> = impl Future<Item = (), Error = !> + 'a;

  #[async]
  fn start(&mut self, count: Duration) -> Self::Future<'_> {
    ...
  }
}
```

## Remove necessity for `#[async_stream]` and associated macros.

It's possible to support all return types via just a single macro. This may
still require arguments for boxing or pinning the return type, but it might be
possible to automatically infer this from the return type as well.

## More consistency with other languages

All existing statically typed `async`/`await` supporting languages use this form
of signature. See [Prior Art][prior-art] below for examples.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

## The generators return type is not written down

While this changes the written return type, it doesn't affect what the user is
allowed to return from the generator. This is still forced to be a `Result` of
some kind (although, this could likely be extended to `Try`). This inconsistency
between the functions declared return type and the function body's actual return
is what prompted the original design of the macro.

## `impl Trait` requires specifying all lifetimes

> Because of how impl Trait and trait objects handle lifetimes, this interacts
> very poorly with lifetime elisions. Specifically, the return type only
> captures explicitly called out lifetimes, but the returned generator
> necessarily captures all lifetimes.
>
> So say you have a method like this:
>
> ```rust
> #[async]
> fn foo(&self, arg: &str) -> Result<i32, io::Error>
> ```
>
> You'd have to write it like this:
>
> ```rust
> #[async]
> fn foo<'a, 'b>(&'a self, arg: &'b str) -> impl Future<Item = i32, Error = io::Error> + 'a + 'b
> ```
>
> The alternative would be to make lifetime elisions not work the same way the
> work with regular impl Trait return types, but that seems clearly out to me,
> since the whole idea of this change is to make the return type accurately
> reflect with the function returns.
>
> -- [@withoutboats comment](https://github.com/alexcrichton/futures-await/issues/15#issuecomment-366489968)

I argue this is not a true downside. This is just the current state of `impl
Trait`, if this is a major issue for `#[async]` then it will be a major issue
for other users of `impl Trait` and should be fixed at the source.

It also seems to be closer to the current thinking around lifetimes in
signatures, along with the plan to support explicit elided lifetimes this is
ensuring that reading the function signature will allow you to quickly determine
the lifetime dependencies.

# Rationale and alternatives
[alternatives]: #alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

## More consistency with other languages

[C#](https://msdn.microsoft.com/en-us/library/hh191443(v=vs.110).aspx):
```C#
async Task<int> AccessTheWebAsync()
{
    HttpClient client = new HttpClient();
    string urlContents = await client.GetStringAsync("http://msdn.microsoft.com");
    return urlContents.Length;
}
```

[TypeScript](http://www.typescriptlang.org/play/index.html#src=function%20delay(ms)%20%7B%0D%0A%20%20%20%20return%20new%20Promise(function%20(resolve)%20%7B%0D%0A%20%20%20%20%20%20%20%20setTimeout(resolve%2C%20ms)%3B%0D%0A%20%20%20%20%7D)%3B%0D%0A%7D%0D%0Aasync%20function%20foo()%3A%20Promise%3Cnumber%3E%20%7B%0D%0A%20%20%20%20await%20delay(100)%3B%0D%0A%20%20%20%20return%205%3B%0D%0A%7D):
```TypeScript
async function foo(): Promise<number> {
    await delay(100);
    return 5;
}
```

[Dart](https://www.dartlang.org/articles/language/await-async):
> If the expression being returned has type `T`, the function should have return type `Future<T>` (or a supertype thereof). Otherwise, a static warning is issued.

[Hack](https://docs.hhvm.com/hack/async/introduction):
```hack
async function curl_A(): Awaitable<string> {
  $x = await \HH\Asio\curl_exec("http://example.com/");
  return $x;
}
```

# Unresolved questions
[unresolved]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
