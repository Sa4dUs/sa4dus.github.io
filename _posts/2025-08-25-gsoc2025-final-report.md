---
title: GSoC 2025. Final Report
categories: [gsoc 2025]
tags: [gsoc, rust, rustc, autodiff]
---

- Contributor: [https://github.com/sa4dus](https://github.com/sa4dus)
- Mentor: [Manuel Drehwald (@ZuseZ4)](https://github.com/ZuseZ4), [Oli Scherer (@oli-obk)](https://github.com/oli-obk)
- Project: [ABI/Layout handling for the automatic differentiation feature](https://summerofcode.withgoogle.com/programs/2025/projects/USQvru7i)
- Organization: [The Rust Foundation](https://summerofcode.withgoogle.com/programs/2025/organizations/the-rust-foundation)

---
## Overview

The original goal of this project was to make sure that `rustc` ABI adjustments applied to functions annotated with the `#[rustc_autodiff]` attribute do not interfere with autodiff functionallity. However, between the submission of the proposal and the start of the coding period, it became clear that moving autodiff to a pipeline based on Rust intriniscs would be a cleaner and more sustainable approach.

This change made the compilation pipeline simpler and less error prone, which improves the robustness and maintanabillity of autodiff in `rustc`.

The project then split into two parts:
1. Migrating autodiff to an intrisics-based pipeline.
2. Making sure ABI adjustments in `rustc` do not break compatibility with Enzyme.

## Part I: Migration to rust intrinsics
### Pull Request
- [PR#142640](https://github.com/rust-lang/rust/pull/142640)

### Issues Closed
- [Issue#143994](https://github.com/rust-lang/rust/issues/143994)
- [Issue#143543](https://github.com/rust-lang/rust/issues/143543)

Since autodiff essentially generates a function body through an LLVM Enzyme call, a lot of the extra machinery in the pipeline was unnecessary.

The solution was to simplify this process by adding an intrinsic that directly lowers to the corresponding autodiff engine (currently Enzyme).

After this change, the autodiff workflow now has only three marked steps:
- macro expansion,
- attribute parsing,
- code generation

There is no need for special autodiff handling in every compiler stage. The only exception is ensuring the source function is not optimized away during monomophization. This is done by forcing its addition to `MonoItems`.

For example, before:
```rust
#[autodiff_reverse(cos_box, Duplicated, Active)]
fn sin(x: &Box<f32>) -> f32 {
    f32::sin(**x)
}
```

was expanded into a more complicated version with dummy bodies and `black_box` calls to keep items alive:
```rust
#[rustc_autodiff]
#[inline(never)]
fn sin(x: &Box<f32>) -> f32 {
    f32::sin(**x)
}
#[rustc_autodiff(Reverse, 1, Duplicated, Active)]
#[inline(never)]
fn cos_box(x: &Box<f32>, dx: &mut Box<f32>, dret: f32) -> f32 {
    unsafe {
        asm!("NOP");
    };
    ::core::hint::black_box(sin(x));
    ::core::hint::black_box((dx, dret));
    ::core::hint::black_box(sin(x))
}
```

Here, some details must be noted:
- The dummy body of the differentiated function is later replaced by an Enzyme call at the LLVM level.
- The dummy body is a bit hacky; although it works, it suggests that there could be a better way to do it. The `::core::hint::black_box(...)` calls are mainly there to mark items as used. The last one ensures the function has the right return type so it does not cause errors during early syntax analysis. `black_box` tells the compiler to be maximally pessimistic, preventing most optimizations. This allow the function to survive until lowering without breaking.

To solve this, we introduced an `autodiff` intrinsic defined as:
```rust
#[rustc_intrinsic]
pub const fn autodiff<F, G, T: crate::marker::Tuple, R>(f: F, df: G, args: T) -> R;
```

With this, we can now expand our previous function:
```rust
#[autodiff(cos_box, Reverse, Duplicated, Active)]
fn sin(x: &Box<f32>) -> f32 {
    f32::sin(**x)
}
```

as:
```rust
#[rustc_autodiff]
fn sin(x: &Box<f32>) -> f32 {
    f32::sin(**x)
}
#[rustc_autodiff(Reverse, 1, Duplicated, Active)]
fn cos_box(x: &Box<f32>, dx: &mut Box<f32>, dret: f32) -> f32 {
    std::intrinsics::autodiff(sin::<>, cos_box::<>, (x, dx, dret))
}
```

Where `autodiff` intrinsic expands to (omiting some llvm details):
```llvm
define internal float @cos_box(ptr %x, ptr %dx, float %dret) {
start:
  %0 = tail call float (...) @__enzyme_autodiffcos_box(ptr @sin, metadata !"enzyme_primal_return", metadata !"enzyme_dup", ptr nonnull %x, ptr nonnull %dx)
  ret float %0
}
```

Enzyme then transforms this `@__enzyme_autodiffcos_box` call into a real function that computes the derivative of `@sin`.

However, there are still some issues we need to handle:

What if source function (`sin` in our example) is unused in the code? In that case, `rustc_monomorphize::collector` will not include it in `MonoItems`. When we later try to retrieve it at codegen, it will not be found, which causes an error.

To prevent this, we must ensure that the function is not optimized away, at least until the differentiated function body has been completed by Enzyme. This must be done without affecting ordinary functions.

In this case, when we encounter an intrinsic during monomorphization and it's our `autodiff` intrinsic, we take the source function from the first argument and explicitly add it to the collected `MonoItems`.

> Note: when differentiating functions with generic parameters, every time we add, resolve, or use an instance of such a function, we must ensure we are using the correct generic parameters. For example, collecting `foo::<f32>` is not the same as collecting `foo::<f64>`, as from monomorphization's point of view, they are completely different functions.

Finally, all related inlining logic has been removed. It is no longer necessary because, at the point where the intrinisc is codegened, inlining has not yet been applied. This reduces the level of indirection in the LLVM IR, generates a much smaller IR, and may even improve runtime performance.

## Part II: Handling ABI Adjustments for Enzyme
### Pull Request
- [PR#142544](https://github.com/rust-lang/rust/pull/142544)

### Issues Closed
- [Issue#144025](https://github.com/rust-lang/rust/issues/144025)

The second part of the project focused on cases where ABI adjustments in `rustc` could cause mismatches in function argument activity when working with Enzyme. This mostly affected types that change the number of arguments after lowering. While slices were already handled correctly, other types required additional work.

We approached the solution by looking at each argument's `BackendRepr`. After some analysis, we identified two problematic cases:

#### 1. `BackendRepr::ScalarPair`

Types with this backend representation are passed to LLVM-IR in a special way (here we always refer to the representation passed to Enzyme, before any optimization).

For these types, the fix is to duplicate the activity. For example:

Suppose we have an argument of type `(f32, f32)` with activity `Dual`. This argument is lowered to LLVM-IR as two separate `f32` values. This creates a mismatch, because the activity count is 1 but the argument count is 2. The solution is to insert another `Dual` activity after the first one. From Enzyme's point of view, we are now differentiating two `f32` values, both with `Dual` activity.

#### 2. **ZST (Zero-Sized Types)**
These types are special because they are not referenced in the function arguments at LLVM level. For now, we simply drop the activity for such arguments. In the future, we will enforce that the activity for a ZST must always be `Const`, since it makes no sense to differentiate with respect to "nothing".

However, because the activity is always dropped anyway, this check does not affect functionality. It is more a matter of ensuring correctness in the source code.

There is still one issue left unresolved: types that use dynamic dispatch. Handling these was deliberately left outside the scope of this project. They will be one of the first problems I plan to address after the end of the coding period.

## Acknowledgments

I would like to thank my mentors, Manuel Drehwald ([@ZuseZ4](https://github.com/ZuseZ4)) and Oli Scherer ([@oli-obk](https://github.com/oli-obk)), for their guidance on both overall direction of the project and the technical details. I also thank The Rust Foundation and Google for supporting this work through GSoC, and the Rust community members who provided useful feedback and discussions.