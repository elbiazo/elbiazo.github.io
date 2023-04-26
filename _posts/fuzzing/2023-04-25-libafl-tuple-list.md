---
title: LibAFL Tuple List
date: 2023-04-25
categories: [fuzzing]
tags: [libafl]
---

When you are working with LibAFL you will see [tuple_list](https://docs.rs/tuple_list/0.1.3/tuple_list/index.html) everywhere. It is a way to do static dispatch in Rust because it doens't support variadic generics.

You can do static dispatch like below. But problem with that is when you are writing library, you don't want to update the actual source everytime someone wants to add new feature.

```rust
pub enum CorpusType {
    /// [`Wordlist`] wrapper
    Wordlist(Wordlist),

    /// [`DirCorpus`] wrapper
    Dir(DirCorpus),

    /// [`RangeCorpus`] wrapper
    Range(RangeCorpus),

    /// [`HttpMethodsCorpus`] wrapper
    HttpMethods(HttpMethodsCorpus),
}

/// [`Corpus`] implementation for [`CorpusType`] enum
impl Corpus for CorpusType {
    #[instrument(skip_all, level = "trace")]
    fn add(&mut self, value: Data) {
        match self {
            Self::Wordlist(corpus) => corpus.add(value),
            Self::Dir(corpus) => corpus.add(value),
            Self::Range(corpus) => corpus.add(value),
            Self::HttpMethods(corpus) => corpus.add(value),
        }
    }
    ...
```

You can also do dynamic dispatch and do [trait object](https://doc.rust-lang.org/book/ch17-02-trait-objects.html) but that will be done in run time.

## Example

From [tuple_list doc.rs](https://docs.rs/tuple_list/0.1.3/tuple_list/index.html), you can get this example
```rust
// `TupleList` is a helper trait implemented by all tuple lists.
// Its use is optional, but it allows to avoid accidentally
// implementing traits for something other than tuple lists.
use tuple_list::TupleList;
 
// Define trait and implement it for several primitive types.
trait PlusOne {
    fn plus_one(&mut self);
}
impl PlusOne for i32    { fn plus_one(&mut self) { *self += 1; } }
impl PlusOne for bool   { fn plus_one(&mut self) { *self = !*self; } }
impl PlusOne for String { fn plus_one(&mut self) { self.push('1'); } }
 
// Now we have to implement trait for an empty tuple,
// thus defining initial condition.
impl PlusOne for () {
    fn plus_one(&mut self) {}
}
 
// Now we can implement trait for a non-empty tuple list,
// thus defining recursion and supporting tuple lists of arbitrary length.
impl<Head, Tail> PlusOne for (Head, Tail) where
    Head: PlusOne,
    Tail: PlusOne + TupleList,
{
    fn plus_one(&mut self) {
        self.0.plus_one();
        self.1.plus_one();
    }
}
 
// `tuple_list!` as a helper macro used to create
// tuple lists from a list of arguments.
use tuple_list::tuple_list;
 
// Now we can use our trait on tuple lists.
let mut tuple_list = tuple_list!(2, false, String::from("abc"));
tuple_list.plus_one();
 
// `tuple_list!` macro also allows us to unpack tuple lists
let tuple_list!(a, b, c) = tuple_list;
assert_eq!(a, 3);
assert_eq!(b, true);
assert_eq!(&c, "abc1");
```

First you need to define a trait so that each member of tuple list have function you want to call. Then you can call tuplie_list after you impelement (Head, Tail) which calls function one after the other.

## LibAFL

In libafl, you will often see this method so that it can give user more flexibility to add their own implementation. In libafl_qemu, you can see this on `QemuEdgeCoverageHelper`.

If you look at `executor` of `QemuExecutor`
```rust
impl<'a, EM, H, OT, QT, S, Z> Executor<EM, Z> for QemuExecutor<'a, H, OT, QT, S>
where
    EM: UsesState<State = S>,
    H: FnMut(&S::Input) -> ExitKind,
    S: UsesInput,
    OT: ObserversTuple<S>,
    QT: QemuHelperTuple<S>,
    Z: UsesState<State = S>,
{
    fn run_target(
        &mut self,
        fuzzer: &mut Z,
        state: &mut Self::State,
        mgr: &mut EM,
        input: &Self::Input,
    ) -> Result<ExitKind, Error> {
        let emu = Emulator::new_empty();
        if self.first_exec {
            self.hooks.helpers().first_exec_all(self.hooks);
            self.first_exec = false;
        }
        self.hooks.helpers_mut().pre_exec_all(&emu, input);
        let mut exit_kind = self.inner.run_target(fuzzer, state, mgr, input)?;
        self.hooks
            .helpers_mut()
            .post_exec_all(&emu, input, &mut exit_kind);
        Ok(exit_kind)
    }
}
```

they have the code `self.hooks.helperes().first_exec_all` and `self.hooks.helpers_mut().pre_exec_all()`. This gets passed in in our harness kind of like this
```rust

        let mut hooks = QemuHooks::new(
            &emu,
            tuple_list!(
                QemuEdgeCoverageHelper::default(),
            ),
        );

        // Create a QEMU in-process executor
        let executor = QemuExecutor::new(
            &mut hooks,
            &mut harness,
            tuple_list!(edges_observer, time_observer),
            &mut fuzzer,
            &mut state,
            &mut mgr,
        )
        .expect("Failed to create QemuExecutor");
```
You can look at how `QemuEdgeCoverageHelper` is implemented and implement your own. It will pass QemuHelper argument and that can contain anything that gets updated in the qemu harness. We can use this to filter hitcount or add more option for executor