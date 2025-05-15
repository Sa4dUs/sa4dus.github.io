---
title: Prevent ICE in autodiff return activity validation
categories: [rustc]
tags: [rust, rustc, autodiff, internal-compiler-error, error-handling, macro-expansion]     # TAG names should always be lowercase
---

Previously, the compiler did not validate the OUTPUT_ACTIVITY parameters during the macro expansion phase. As a result, invalid configurations would propagate to later stages, causing ICEs such as:

```
error: internal compiler error:
compiler/rustc_codegen_ssa/src/codegen_attrs.rs:935:13:
Invalid input activity Dual for Reverse mode
```

This error originated in the code generation phase (`compiler/rustc_codegen_ssa/src/codegen_attrs.rs`), making it difficult for users to diagnose and correct the issue.

[PR #138231](https://github.com/rust-lang/rust/pull/138231/files#diff-a313c94149bd8e03a482dbf15c3d1b9414ca74c4f2e35f94253649c2be1ceb5c) addresses this by moving the validation of OUTPUT_ACTIVITY to the macro expansion phase, specifically within `compiler/rustc_builtin_macros/src/autodiff.rs`. By performing these checks earlier, the compiler can emit clear and informative error messages without encountering ICEs.

In addition, a new error type was introduced in `compiler/rustc_builtin_macros/src/errors.rs`:
```rust
#[derive(Diagnostic)]
#[diag(builtin_macros_autodiff_ret_activity)]
pub(crate) struct AutoDiffInvalidRetAct {
    #[primary_span]
    pub(crate) span: Span,
    pub(crate) mode: String,
    pub(crate) act: String,
}
```

Before this change, the following code would cause an ICE:
```rust
#[autodiff(df, Forward, Dual, Active)]
fn f(x: f32) -> f32 {
    unimplemented!()
}
```

With the changes from [PR #138231](https://github.com/rust-lang/rust/pull/138231/files#diff-a313c94149bd8e03a482dbf15c3d1b9414ca74c4f2e35f94253649c2be1ceb5c), the compiler now produces a clear error message:
```
error: invalid return activity Active in Forward Mode
  --> $DIR/autodiff_illegal.rs:161:1
   |
LL | #[autodiff(df19, Forward, Dual, Active)]
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: this error originates in the attribute macro `autodiff` (in Nightly builds, run with -Z macro-backtrace for more info)
```

See `tests/ui/autodiff/autodiff_illegal.stderr` to have a clearer vision about autodiff errors.

> **Note on span precision**:  
Currently, error messages related to invalid activity annotations point to the entire `#[autodiff]` attribute, rather than the specific argument causing the issue. This is due to limitations in how spans are passed within the `gen_enzyme_decl` function, which only receives the overall attribute span. While some error types (like unknown activities) can target specific spans, improving span precision for return/input activity errors would require broader changes. A `FIXME` has been added to revisit this in the future.