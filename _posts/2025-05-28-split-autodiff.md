---
title: Split autodiff into autodiff_forward and autodiff_reverse
categories: [rustc]
tags: [rust, rustc, autodiff, error-handling, macro-expansion]     # TAG names should always be lowercase
---

PR #140697 introduces a new improvement for the autodiff feature: splitting the existing #[autodiff(...)] procedural macro into two specialized macros—#[autodiff_forward(...)] and #[autodiff_reverse(...)].

The original unified #[autodiff(NAME, MODE, INPUT_ACTIVITIES, OUTPUT_ACTIVITIES)] macro posed several issues:
    1. Forward and reverse modes require different parameters.
    2. While both modes use shadow arguments, their initialization logic differs.
    3. Splitting the macros simplifies documentation and makes it a bit easier, as it's already done at https://enzyme.mit.edu/rust.


Each macro variant now routes through its own adapter, by passing the `mode: DiffMode` at macro expansion (`compiler/rustc_builtin_macros/src/autodiff.rs`)  to `extend` and create two different functions: `expand_forward` and `expand_reverse`:
```rust
pub(crate) fn expand_forward(...) -> Vec<Annotatable> {
    expand_with_mode(..., DiffMode::Forward)
}

pub(crate) fn expand_reverse(...) -> Vec<Annotatable> {
    expand_with_mode(..., DiffMode::Reverse)
}
```

Internally, both expand to the canonical #[rustc_autodiff(...)] form, where the mode is injected manually via:
```rust
let mode_symbol = match mode {
    DiffMode::Forward => sym::Forward,
    DiffMode::Reverse => sym::Reverse,
    _ => unreachable!("Unsupported mode"),
};
ts.insert(0, TokenTree::Token(Token::new(TokenKind::Ident(mode_symbol, false.into()), Span::default()), Spacing::Joint));
```