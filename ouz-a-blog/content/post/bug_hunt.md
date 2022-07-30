---
title: "The Bug"
date: 2022-07-30T15:41:58+03:00
---

## Introduction

Before we begin I will assume you know the basics of [Rust](https://www.rust-lang.org/) and some basic computer science stuff, I will try to explain everything else. 

This is a story of a bug that broke the Rust compiler so hard it required two months to fix, it produced all manners of crashes, bugs that produced their crashes and horrors beyond reason.

Like all good engineering stories, this one starts with a problem. 

To understand what this problem is we need a super quick intro to the Rust compiler.

### Super quick intro

When you build your Rust project, the Rust compiler does a lot of things to make your code, safe, secure, and fast, it starts this by first checking your command line argument like did you run `cargo run --release` or just `cargo run`, which alters checks and optimizations done by the compiler.

Then we get to [lexing](https://en.wikipedia.org/wiki/Lexical_analysis) which produces tokens, these tokens get into parser which produces [Abstract Syntax Tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree) next AST is converted into [High-Level Intermediate Representation](https://en.wikipedia.org/wiki/Intermediate_representation), `HIR` is a compiler-friendly version of your Rust program. 

`HIR` is used to do *type-inference*, *trait solving*, and *type checking*. After doing those `HIR` is lowered to [Mid-level Intermediate Representation](https://rustc-dev-guide.rust-lang.org/overview.html#mir-lowering) this is where many optimizations and borrow checking happens (yes that borrow checker).

Because our problem resides within `MIR` I won't give any more details about the rest of the Rust compiler, if you want to read more about it click [here](https://rustc-dev-guide.rust-lang.org/overview.html).

## The Problem
Now that we had our super quick intro to the Rust compiler we can explore the problem.


> The MIR uses `Place` to refer to memory locations. A `Place` consists of a base, either a static or a local, and a series of projections starting from that base. With one exception, these projections are merely refinements of the base: they specify a field, an array element, or the portion of memory corresponding to a given enum variant. However, the `Deref` projection is unique in that the target of the  `Place` refers to a completely different region of memory than the base. As a result, several analyses must iterate over each place to see whether that place contains a `Deref`"

In code, this translates into:
```rust    
let mut a = (17,13);
let mut b = (99, &mut a);
let x = &mut (*b.1).0;

// Given the code above emitted MIR would be this.

let mut _0: ();
let mut _1: (i32, i32);
let mut _3: &mut (i32, i32);

// ...
_4 = &mut ((*(_2.1: &mut (i32, i32))).0: i32); 
_5 = &mut ((*(_2.1: &mut (i32, i32))).1: i32);
```

This is the result for a basic case, the more complex this gets longer the `Deref` chain becomes, which results in more complexity to be solved by MIR analyses.

### Solution

What's the solution? Remove `Deref` from projections, but just removing the whole thing would cause chaos, instead, we remove it in small baby steps. 

This is where I started my work. One day I asked [oli-obk](https://github.com/oli-obk) what should I work on, he pointed me to this problem and said we just basically need to do this:

```rust
let x = (*a.b).c
```

into

```rust
let tmp = a.b;
let x = (*tmp).c
```

Which I (with the mentoring of oli-obk) [did](https://github.com/rust-lang/rust/pull/95649). I created a new `MIR` optimization named `Derefer` pass that does the above-mentioned conversion.

Sounds like a happy ending doesn't it, unfortunately, it isn't, `MIR` has a lot of optimization passes and I can't just do the conversion without any side effects, so the plan was to pull `Derefer` early in the passes and deal with a single pass at a time.

Everything went relatively well until I met with `elaborate_drops`.

### Derefer
This is the explanation for the main body of `deref_separator`.

We need to learn a few terms before reading the code.

- `Body` : The lowered representation of a single function.
- `Rvalue`: Expressions that produce value
- `Operand`: The argument to an rvalue
- `Locals`: Memory locations allocated on the stack(temp values, local variables, function arguments)


```rust
// This function visit every `Place` within a MIR body
fn visit_place(&mut self, place: &mut Place<'tcx>,
    cntxt: PlaceContext, loc: Location){
    if !place.projection.is_empty()
        && cntxt != PlaceContext::NonUse(VarDebugInfo)
        && place.projection[1..].contains(&ProjectionElem::Deref)
```
3. Projections shouldn't be empty...
5. Since the problem only starts when `Derefer` is not at the start of the projection list, we skip non-problematic ones.
---------------
```rust
let mut place_local = place.local;
let mut last_len = 0;
let mut last_deref_idx = 0;
let mut prev_temp: Option<Local> = None;
    for (idx, elem) in place.projection[0..].iter().enumerate() {
        if *elem == ProjectionElem::Deref {
            last_deref_idx = idx;
        }
    }
```
1. We need to have a starting local.
2. We start copying from the beginning so this will start at 0 (this is going to make more sense later)
3. Position of the last `Deref` in the slice of projections.
4. We have to destroy the last temp we created, this is `None` for now
------------
`patcher` here refers to the struct called `MirPatch` which can add, new `Locals` new `Statements` etc.

```rust
    for (idx, (p_ref, p_elem)) in place.iter_projections().enumerate() {
        if !p_ref.projection.is_empty() && p_elem == ProjectionElem::Deref {
            let ty = p_ref.ty(&self.local_decls, self.tcx).ty;
            let temp = self.patcher.new_local_with_info(
                ty,
                self.local_decls[p_ref.local].source_info.span,
                Some(Box::new(LocalInfo::DerefTemp)),
```
1. `idx` will be used to compare with `last_deref_idx`
2. We only care for `Deref`
4. `temp` is created with `MirPatch`, it will be used for derefing
7. We mark this as `DerefTemp`, so we can retrieve this information when needed (it will be needed in other mir-opts)
-----
```rust
self.patcher.add_statement(loc, StatementKind::StrorageLive(temp));
let deref_place = Place::from(place_local)
.project_deeper(&p_ref.projection[last_len..], self.tcx);               
```
1. We mark the temp we just created as living. `StorageLive` and `StorageDead` mark the live range of a local
2. These last 2 lines are quite important, I will be explaining them with example code below.
```rust
fn main () {
     let mut a = (42, 43);
     let mut b = (99, &mut a);
     let mut c = (11, &mut b);
     let mut d = (13, &mut c);
     let x = &mut (*d.1).1.1.1;
}
```
Given this code, `deref_place` and `last_len` will have these values.
```rust
projections[Field(field[1], &mut (i32, &mut (i32, &mut (i32, i32)))]
last_len = 0
// in the next loop
projections[Deref,Field(field[1], &mut (i32, &mut (i32, i32)))]
last_len = 1
// in the last loop
projections[Deref, Field(field[1], &mut (i32, i32))]
last_len = 3
```
As you can see, we cut away `&` with each loop creating a new temp that holds the next referenced value.

---
```rust
self.patcher.add_assign(loc, Place::From(temp),
    Rvalue::CopyForDeref(deref_place),);
place_local = temp;
last_len = p_ref.projection.len();
```
1. We assign the newly created `deref_place` to our `temp` using, `Rvalue::CopyForDeref` which is basically `Copy` but let us differentiate it from normal `Copy`.
2. In the next loop `deref_place` will be created from this temps local value.
3. We update last_len as explained above.
----
```rust
if idx == last_deref_idx {
    let temp_place = Place::from(temp)
        .project_deeper(&place.projection[idx..], self.tcx);
    *place = temp_place;
}
```
1. Remember `last_deref_idx` ? this is where we use it.
2. For the example given above temp place's projections are `    Deref,
    Field(
        field[1],
        i32,
    ),` as you can see final projection is clean, meaning we successfully transformed it from this
       
       `_8 = &mut ((*((*((*(_6.1: &mut (i32, &mut (i32, &mut (i32, i32))))).1: &mut (i32, &mut (i32, i32)))).1: &mut (i32, i32))).1: i32);`
    to this `_8 = &mut ((*_12).1: i32); `

3. We swap the original place, with our final value.

---
```rust
if let Some(prev_temp) = prev_temp {
    self.patcher.add_statement(loc, StatementKind::StorageDead(prev_temp));
}
prev_temp = Some(temp)
```
1. Since we have to eliminate previously created temps we do it here
4. Since we are at the end of our loop we assign, temp as `prev_temp` to be destroyed in the next loop.




## The Bug
Now we know how `Derefer` works it's easier to explain what went wrong with it.

After I pulled `Derefer` before `elaborate_drops` compiler started complaining that I was moving things that shouldn't be moved, normally this is a serious error but remember we are creating temporary value with `Derefer` and we are only copying very specific things, so I created a new [Rvalue::CopyForDeref](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/mir/enum.Rvalue.html#variant.CopyForDeref), which lets us bypass `Move` error caused by the `elaborate_drops`.

With that out of the way I thought, that should be it we shouldn't have any more problems with this I hit compile, and everything is going well compiler compiles itself :D.
Because we are dealing with a very complex topic, being able to compile is not a good measure of anything, thankfully brilliant [RalfJung](https://github.com/RalfJung) came up with a tool called `Miri` which catches bugs that would be impossible to catch otherwise. 

So I start compiling `Miri`(it takes 20 minutes), and almost at the end the whole thing crashes, normally when you are dealing with the compiler, *Internal Compiler Errors* or `ICE` are things you see every day and they give quite good information, an error message, backtrace for your program, etc. However, this one was different.

```rust
Caused by:
process didn't exit successfully: ...
warning: build failed, waiting for other jobs to finish...
error: `"/Users/ouz/Documents/GitHub/rust/build/aarch64-apple-darwin/stage0/bin/cargo" "check" "--release" "--manifest-path" "/var/folders/8f/tlj_q5zj6v9g71vp_q32pb7r0000gn/T/xargo.bBvDKyqBOkYA/Cargo.toml" "--target" "aarch64-apple-darwin" "-p" "std"` failed with exit code: Some(101)
fatal error: failed to run xargo
Build completed unsuccessfully in 0:06:48
```
What? So you are telling me the compiler compiled a few million lines of Rust code without any errors, and now it's giving me an error that doesn't make any sense because there is nothing to make sense of how do you create `MRE` from this error how do you debug this?

Don't get me wrong I love this, it's like doing a very difficult quest chain in a game. My curiosity was the fuel and `println!` was my weapon I started hacking the compiler printing everything, trying to break the compiler to see hopefully meaningful error. I spent a few weeks doing this although I didn't manage to solve it I read quite a bit of code and learned a lot which is a good thing.

We were baffled by the situation, how can you solve a problem you don't know? My `println!` game got quite good I was comparing everything but it wasn't enough I had to narrow down the situation. 

As they say, desperate times call for desperate measures, I created a monstrosity called [miri_hunter](https://github.com/ouz-a/rust/commit/095476f6df12bf4f70653d06e905c38b920c4474#diff-a5d4ecb27e5c0f52acc5f4a682626b916874a3628752c9df61a4af0ef8fa60ce). (This is the clean version)

At the time `MIR` passes were like this `miri_hunter -> deref_separator -> elaborate_drops`. 

```rust
let mut og_bod = body.clone();
let mut clon = og_bod.clone();
let derefer = Derefer {};
let elab = ElaborateDrops {};
let call_g = AddCallGuards::CriticalCallEdges;
// derefer before elaborate
derefer.run_pass(tcx, &mut og_bod);
call_g.run_pass(tcx, &mut og_bod);
elab.run_pass(tcx, &mut og_bod);
// derefer after elaborate
call_g.run_pass(tcx, &mut clon);
elab.run_pass(tcx, &mut clon);
derefer.run_pass(tcx, &mut clon);
```
- We clone two bodies.
- For `og_bod` we run -> `Derefer -> elab` 
- For `clon` we run -> `elab -> Derefer` 

We do this to see how putting `Derefer` changes `Body`. But you can't just compare two `MIR` bodies they are huge structs. You can with `format!` it's cursed and you should never debug your Rust code like this :skull:.

So with that, we compile the compiler waiting to see anything, after a few crashes a few days and the genius of oli-obk we were able kinda to find the problem.

I won't go into too much detail about how `elaborate_drops` works as that would require another blog post, but we need to know a tiny about it to understand the next part.

At the beginning of `elaborate_drops` there is this line
```rust
let move_data = match MoveData::gather_moves(body, tcx, param_env) {
```        
that gathers `Moves` from `Body` well amount of moves `og_bod` and `clon` had looked different. You could be asking yourself right now but didn't you deal with that before when you added the new `Rvalue::CopyForDeref`, I asked the same question and spent the next [two weeks](https://twitter.com/o_uz_a/status/1528002287876067329) trying to build a `MRE`.

I finally managed to create a `MRE`, then I wielded `println!` again and went forth printing every single thing trying to find where this difference was coming from, I then found out that more things were *Dropped* in `og_bod` compared to clon, somehow and somewhere temp values created by `Derefer` was causing weird things.

This too took a long time at this point I was 2 months into trying to solve this issue I was seeing `xargo` errors in my dreams but in reality, I couldn't see a solution nor the light at the end of the tunnel. 

Remember that scene from Two Towers, where Helm's Deep was almost taken by Uruk-hai but at the last minute Gandalf comes with the sun rising behind his back with thousands of Rohirrim with him takes Uruk-hai by storm and Rohan wins the war.

Something similar to that happened, one-morning oli-obk said he had an idea how to solve this problem, and as he explained and showed me how to solve this I was in disbelief he wrote down the basics of [un_derefer](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir_dataflow/un_derefer/struct.UnDerefer.html) hit compile and it runs without any errors. I was in disbelief, although this wasn't the perfect solution and we couldn't find why temp values caused errors this was the only way forward.

In the end, I created the [PR](https://github.com/rust-lang/rust/pull/98145), after weeks of hard work and thousands of `println!` we solved it and it got merged.
Also, I opened my first [major change proposal](https://github.com/rust-lang/compiler-team/issues/532) for this `Derefer` project.

### End
I'm a self-taught programmer, I couldn't even imagine contributing to a compiler a few years ago let alone, doing work like this, however, Rust is special, it has an awesome community. 

I learned Rust because I was frustrated with C++ which made making my game way harder than it should be, I ported my roguelike to Rust, read the book twice, and read the rustonomicon(best educational material Rust has imo), as I interacted with the community more I started contributing to a [crate](https://github.com/Ralith/hecs) I was using, I started to fell in love with open source software.
I liked coding and solving challenging problems, and this helped people. Perfect.

At the start of 2022, I re-rolled to college and started a master's in Philosophy, and started contributing to the Rust compiler itself. It was quite challenging, I started small and solved a few `ICE` issues but I wanted to do something bigger so as above mentioned I asked oli-obk what to work on and he showed me this `Deref` and he became my mentor :heart:.

I loved this work and wanted to do this for a living but my professional life was a mess... Thankfully Rust Foundation opened [Community Grants Program](https://foundation.rust-lang.org/news/2022-03-31-cgp-is-open-announcement/) it would give 1000$ a month for a year for selected fellows, I applied, and a few months later they announced I was selected as a fellow.

Although it was a bit scary, at the start of July I quit college, dropped everything I do decide to become a full-time rustc contributor. In near future, I am going to write more blog posts, and record videos about Rust compiler and educational stuff about how to get into contributing to it.

You can support the work I do on [GitHub](https://github.com/sponsors/ouz-a) or [Patreon](https://www.patreon.com/ouz_a).