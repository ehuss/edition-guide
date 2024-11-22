# Tail expression temporary scope

🚧 The 2024 Edition has not yet been released and hence this section is still "under construction".
More information may be found in the tracking issue at <https://github.com/rust-lang/rust/issues/123739>.

## Summary

- Temporary values generated in evaluation of the tail expression of a [function] or closure body, or a [block] are now dropped before local variables.

[function]: ../../reference/items/functions.html
[block]: ../../reference/expressions/block-expr.html

## Details

The 2024 Edition changes the drop order of [temporary values] in tail expressions. It often comes as a surprise that, before the 2024 Edition, temporary values in tail expressions are dropped later than the local variable bindings, as in the following example:

[temporary values]: ../../reference/expressions.html#temporaries

```rust,edition2021,compile_fail,E0597
// Before 2024
# use std::cell::RefCell;
fn f() -> usize {
    let c = RefCell::new("..");
    c.borrow().len() // error[E0597]: `c` does not live long enough
}
```

This yields the following error with the 2021 Edition:

```text
error[E0597]: `c` does not live long enough
 --> src/lib.rs:4:5
  |
3 |     let c = RefCell::new("..");
  |         - binding `c` declared here
4 |     c.borrow().len() // error[E0597]: `c` does not live long enough
  |     ^---------
  |     |
  |     borrowed value does not live long enough
  |     a temporary with access to the borrow is created here ...
5 | }
  | -
  | |
  | `c` dropped here while still borrowed
  | ... and the borrow might be used here, when that temporary is dropped and runs the destructor for type `Ref<'_, &str>`
  |
  = note: the temporary is part of an expression at the end of a block;
          consider forcing this temporary to be dropped sooner, before the block's local variables are dropped
help: for example, you could save the expression's value in a new local variable `x` and then make `x` be the expression at the end of the block
  |
4 |     let x = c.borrow().len(); x // error[E0597]: `c` does not live long enough
  |     +++++++                 +++

For more information about this error, try `rustc --explain E0597`.
```

In 2021 the local variable `c` is dropped before the temporary created by `c.borrow()`. The 2024 Edition changes this so that the temporary value `c.borrow()` is dropped first, followed by dropping the local variable `c`, allowing the code to compile as expected.

See the [temporary scope rules] for more information about how temporary scopes are extended. See the [`if let` temporary scope] chapter for a similar change made to `if let` expressions.

[`if let` temporary scope]: temporary-if-let-scope.md
[temporary scope rules]: ../../reference/destructors.html#temporary-scopes

## Migration

Unfortunately, there are no semantics-preserving rewrites to shorten the lifetime for temporary values in tail expressions[^RFC3606]. The [`tail_expr_drop_order`] lint detects if a temporary value with a custom, non-trivial `Drop` destructor is generated in a tail expression. Warnings from this lint will appear when running `cargo fix --edition`, but will otherwise not automatically make any changes. It is recommended to manually inspect the warnings and determine whether or not you need to make any adjustments.

If you want to manually inspect these warnings without performing the edition migration, you can enable the lint with:

```rust
// Add this to the root of your crate to do a manual migration.
#![warn(tail_expr_drop_order)]
```

[^RFC3606]: Details are documented at [RFC 3606](https://github.com/rust-lang/rfcs/pull/3606)

[`tail_expr_drop_order`]: ../../rustc/lints/listing/allowed-by-default.html#tail-expr-drop-order