---
title: GSoC 2026. Final Report
categories: [gsoc 2026]
tags: [gsoc, rust, rustc, offload]
---

- Contributor: [Marcelo Dominguez (@Sa4dUs)](https://github.com/sa4dus)
- Mentor: [Manuel Drehwald (@ZuseZ4)](https://github.com/ZuseZ4)
- Project: [A Frontend for Safe GPU Offloading in Rust](https://summerofcode.withgoogle.com/programs/2026/projects/eF3fkjrN)
- Organization: [The Rust Foundation](https://summerofcode.withgoogle.com/programs/2025/organizations/the-rust-foundation)

---

## Overview

The `offload` feature lets the Rust compiler generate code to be executed for GPU devices. Before this project, the only way to use the feature was to call the `offload` intrinsic directly. This required internal compiler features and unsafe code.

The code below is the minimum required to run a simple kernel before this project (pseudocode, NVIDIA and no-std assumed for simplicity):

```rust (pseudocode)
#![allow(internal_features)]
#![feature(abi_gpu_kernel)]
#![feature(rustc_attrs)]
#![feature(link_llvm_intrinsics)]
#![feature(core_intrinsics)]
#![cfg_attr(target_arch = "nvptx64", feature(stdarch_nvptx))]

#[cfg(target_os = "linux")]
unsafe extern "C" {
    pub fn foo(y: *mut f64, x: &[f64]);
}

#[cfg(not(target_os = "linux"))]
#[unsafe(no_mangle)]
#[inline(never)]
#[rustc_offload_kernel]
pub extern "gpu-kernel" fn foo(y: *mut f64, x: &[f64]) {
    // manual indexing using arch specific llvm intrinsics
    let i = unsafe {
        (_block_idx_x() * _block_dim_x() + _thread_idx_x()) as usize;
    };

    if i < END {
        // body
    }
}

fn main() {
    unsafe {
        core::intrinsics::offload::<_, _, ()>(
            foo,
            [BLOCKS, 1, 1],
            [THREADS_PER_BLOCK, 1, 1],
            0,
            0,
            (
                // mutable args need to be passed as UNSAFE raw pointers
                y.as_mut_ptr(),
                &x
            ),
        );
    }
}
```

The user had to do the following:
- Enable internal compiler features with `#![feature(...)]` attributes.
- Pass mutable arguments as raw pointers.
- Write manual indexing with architecture-specific intrinsics.

The goal of this project was to build a safe and ergonomic frontend for the intrinsic. The frontend provides the following:
- the `offload_kernel` attribute macro,
- the `offload!` procedural macro,
- the `Region` and `PartitioningStrategy` abstractions for mutable arguments,
- type-checking of intrinsic calls in the host.

With the frontend, the same result is achieved with the code below:

```rust
#[offload_kernel]
fn foo(mut y: Region<f64, Linear1D>, x: &[f64]) {
    if let Some(v) = y.get_mut() {
        // body
    }
}

fn main() {
    offload! {
        kernel = foo,
        // fields with default values can be omitted
        args = (y, x), // mutable args can be passed safely
    };
}
```

The frontend provides the following benefits:
- No internal compiler features are required.
- The interface is more concise.
- Manual indexing is not required.
- Calling kernels is safe.
- Writing kernels is mostly safe.

## Pull Requests

Most of the work was done in the `rust-lang/rust` repository.

### `offload_kernel` macro [#156642](https://github.com/rust-lang/rust/pull/156642)


This PR adds the `#[offload_kernel]` attribute macro. The macro marks a function as a kernel. It expands the kernel definition into a host-side declaration and a device-side definition.

### Expose device selection [#158032](https://github.com/rust-lang/rust/pull/158032)

This PR exposes device selection to the user. The number of available devices is obtained with `omp_get_num_devices`. The `offload` intrinsic now accepts an argument for device selection.

### Safe mutable arguments with `Region` and `PartitioningStrategy` [#158076](https://github.com/rust-lang/rust/pull/158076)

This PR makes mutable arguments safe to use. `Region` is now a lang item and is mapped to slices. A `PartitioningStrategy` partitions the data into disjoint memory regions so each thread receives its own. This prevents mutable aliasing between threads.

### Type-check for `offload` intrinsic calls [#158693](https://github.com/rust-lang/rust/pull/158693)

Before this PR, argument and return types were checked through trait bounds on the intrinsic declaration, which was not sufficient to ensure kernels were being called properly. This PR moves these checks to the type-checking phase. The compiler now validates argument and return types during type checking.

### Guard flags in type checking to prevent performance regressions [#160454](https://github.com/rust-lang/rust/pull/160454)

This PR fixes a performance regression introduced by [PR #158693](https://github.com/rust-lang/rust/pull/158693). Guard flags ensure that the offload type-checking logic runs only when necessary.

### Support for generics in `offload` and removal of `no_mangle` [#159566](https://github.com/rust-lang/rust/pull/159566)

This PR adds support for generic kernels. It implements a third pass that collects the kernel instantiations required by `offload`. It also enforces consistent mangling of offload functions between host and device. This removes the need for the `no_mangle` attribute.

### The `offload!` procedural function-like macro [#161055](https://github.com/rust-lang/rust/pull/161055)

This PR adds the `offload!` procedural macro. The macro expands to the intrinsic call with the kernel and its arguments. Fields with default values can be omitted.

> Note: Required docs for the `offload!` macro and three-pass compilation where added to the rustc-dev-guide in [PR #2969](https://github.com/rust-lang/rustc-dev-guide/pull/2969)

## Issues Closed

- [Issue #150985](https://github.com/rust-lang/rust/issues/150985): `std::offload` requires mangled names.

## Design Discussion

Most of the frontend design was discussed in [PR #8](https://github.com/ZuseZ4/rust_perf/pull/8) of the `ZuseZ4/rust_perf` repository, a draft implementation of the offload frontend.

> Note: Part of the work developed during this project contributed to the following academic paper: https://arxiv.org/abs/2608.13759.

## Acknowledgments
I thank my mentor, Manuel Drehwald, for his support and guidance since I was first interested in GSoC 2025. I also thank The Rust Foundation and Google for supporting this work through GSoC.
