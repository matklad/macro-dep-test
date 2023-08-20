# Associated Proc Macro Pattern

It's a common pattern to provide a `foo` crate with trait definitions, and `foo-derive` crate with a
proc-macro derive implementation. Typically, you want `foo` and `foo-derive` to be versioned in
lockstep, because derive crates like to use `#[doc(hidden)]` non-semver-guarded API. Usually, this
is solved by a `derive` feature, which makes `foo` depend on `foo-derive` with `=x.y.z` constraint.

This, however, is problematic for compile times! It means that compilation of `foo-derive` is
sequenced before compilation of `foo`. As `foo-derive` is a derive macro, it needs to parse the Rust
language. Rust is not a small language, so parsing it is fundamentally hard, and requires loads of
code to do correctly. So it takes some time to compile `foo-derive`. What's worse, while normally
Cargo pipelines compilation such that `.rmeta` files are all that's needed to unblock compilation of
dependent crates, for proc macros Cargo really needs to link the whole .so!

The bottom line, while

```toml
foo = { version = "x.y.z", features = ["derive"] }
```

is easy to explain and
works correctly, it could significantly reduce the amount of parallelism available during builds.

On the other hand, while

```toml
foo = { version = "x.y.z" }
foo-derive = { version = "x.y.z" }
```

provides better compilation time, it doesn't constrain `foo` and `foo-derive` to be the same
version.

The pattern in this crate shows how to add that constraint! We can use the following declaration of
dependencies in `foo`'s Cargo.toml:

```toml
[package]
name = "foo"
version = "1.2.3"

[dependencies]
foo-derive = { version = "=1.2.3", optional = true }

[target.'cfg(any())'.dependencies] # <- the trick
foo-derive = { version = "=1.2.3" }

[features]
derive = ["dep:foo-derive"]
```

The trick is a target specific dependency with "impossible" `any()` cfg. This cfg is never true, so
`foo` never actually depends on `foo-derive` (unless the `derive` feature flag is enabled). Non the
less, this platform-specific dependency forces Cargo to include foo-derive into the lockfile, so the
"no two semver-compatible versions of a crate" constraint kicks in.

Eg, if the user's tries to do

```
[dependencies]
foo = "=1.2.3"
foo-derive = "=1.2.2"
```

their build will (correctly) fail.

Crucially, because the cfg is never true, the `foo` crate doesn't _actually_ depend on `foo-derive`,
so it can be independently compiled. Similarly, although every lockfile gets a `foo-derive`, it
isn't actually downloaded unless it is needed elsewhere in the crate graph (the situation is
similar to having windows-specific deps in a lockfile of a linux-only crate).

Note that it's important that target-specific deps, rather features, are used here. With features,
Cargo can look at the entires set of features of the root crate being compiled, deduce the _precise_
set of features that could be activated for dependencies, and prune anything which is guaranteed to
not be needed from Cargo.lock.

With target-specific dependencies (`target.'cfg()'.dependencies` syntax), Cargo has to assume that
each `cfg` _could_ be true, and so it has to conservatively include everything into a lockfile.

## Important Clarification

So, does this actually work? I don't know! I don't think anyone tried this hack at scale, so it
might be the case that something breaks HORRIBLY somewhere. We won't know without trying though :-)
And it seems to work in this little experimient (there's a bunch of `macro-dep-test` and
`macro-dep-test-macros` versions published to crates.io, so you could check for yourself )
