---
layout: post
title: Coming soon
date: 2026-01-01 00:00:00
description: Working on it...
tags: tbd
categories: verus
---

Here are some common pitfalls me and/or the LLM stumbled upon when working with Verus verified code generation, and I wanted to share.

# Correct but not proven
When the code works perfectly, but the verifier cannot prove it due to technical limitations or missing information.

## Loop isolation
Look at that code:
```verus
fn func(n: usize) -> (result: Vec<i32>)
    requires
        n <= 1000
    ensures
        result@.len() == n,
        forall|i: int| 0 <= i < result@.len() ==> result[i] == n
{
    let mut res: Vec<i32> = Vec::new();
    let mut i: usize = 0;
    while i < n
        invariant
            0 <= i <= n,
            res@.len() == i,
            forall|j: int| 0 <= j < res@.len() ==> res[j] == n
        decreases n - i
    {
        res.push(n as i32);
        i += 1;
    }
    res
}
```

Seems pretty valid, but it receive the error:
```text
error: invariant not satisfied at end of loop body
  --> zzz.rs:24:13
   |
24 |             forall|j: int| 0 <= j < res@.len() ==> res[j] == n
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

note: recommendation not met: value may be out of range of the target type (use `#[verifier::truncate]` on the cast to silence this warning)
  --> zzz.rs:27:18
   |
27 |         res.push(n as i32);
   |                  ^

verification results:: 1 verified, 1 errors
error: aborting due to 1 previous error
```

Why is that? Verus cannot prove it is safe to case `i` into `i32`, despite knowing `0 <= i <= n`. This is because it erases the knowledge about `n` when entering the loop, for performance reasons. This can be resolved by adding a loop invariant:
```diff
fn func(n: usize) -> (result: Vec<i32>)
    requires
        n <= 1000
    ensures
        result@.len() == n,
        forall|i: int| 0 <= i < result@.len() ==> result[i] == n
{
    let mut res: Vec<i32> = Vec::new();
    let mut i: usize = 0;
    while i < n
        invariant
+            n <= 1000,
            0 <= i <= n,
            res@.len() == i,
            forall|j: int| 0 <= j < res@.len() ==> res[j] == n
        decreases n - i
    {
        res.push(n as i32);
        i += 1;
    }
    res
}
```

Or by disabling loop isolation
```diff
fn func(n: usize) -> (result: Vec<i32>)
    requires
        n <= 1000
    ensures
        result@.len() == n,
        forall|i: int| 0 <= i < result@.len() ==> result[i] == n
{
    let mut res: Vec<i32> = Vec::new();
    let mut i: usize = 0;
+    #[verifier::loop_isolation(false)]
    while i < n
        invariant
            0 <= i <= n,
            res@.len() == i,
            forall|j: int| 0 <= j < res@.len() ==> res[j] == n
        decreases n - i
    {
        res.push(n as i32);
        i += 1;
    }
    res
}
```
