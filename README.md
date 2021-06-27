# pinned-aliasable

Unboxed aliasable values based on `Pin`.

In Rust, it is currently impossible to soundly create unboxed self-referencing, or
externally-referenced, types - see [this
document](https://gist.github.com/Darksonn/1567538f56af1a8038ecc3c664a42462) for details.
However, futures generated by async blocks need to be self-referencing, and this makes them
[technically unsound](https://github.com/rust-lang/rust/issues/63818). So to avoid
miscompilations, Rust has inserted a temporary loophole: No unique references to `!Unpin` types
will actually be annotated with `noalias`, preventing the compiler from making optimizations
based on the pointed-to types being not self-referential.

This is a huge hack. Not all self-referential types are `!Unpin`, and some `!Unpin` types would
actually work with `noalias`. Ultimately, the solution will be to provide an `Aliasable<T>` type
in libcore that prevents any parent containers from being annotated with `noalias`, but
unfortunately that doesn't exist yet.

As a workaround, this crate provides an [`Aliasable<T>`] type that doesn't cause miscompilations
today by being `!Unpin`, and is future-compatible with the hypothetical `Aliasable` type in
libcore. Additionally, to avoid Miri giving errors, it is substituted for a boxed value when
run using it. When `Aliasable` is finally added to the language itself, I will release a new
version of this crate based on it and yank all previous versions.

[`Aliasable<T>`]: https://docs.rs/pinned-aliasable/*/pinned_aliasable/struct.Aliasable.html

## Examples

A pair type:

```rust
use core::pin::Pin;
use core::cell::Cell;

use pinned_aliasable::Aliasable;
use pin_project_lite::pin_project;
use pin_utils::pin_mut;

pin_project! {
    pub struct Pair {
        #[pin]
        inner: Aliasable<PairInner>,
    }
}
struct PairInner {
    value: u64,
    other: Cell<Option<&'static PairInner>>,
}
impl Drop for PairInner {
    fn drop(&mut self) {
        if let Some(other) = self.other.get() {
            other.other.set(None);
        }
    }
}

impl Pair {
    pub fn new(value: u64) -> Self {
        Self {
            inner: Aliasable::new(PairInner {
                value,
                other: Cell::new(None),
            })
        }
    }
    pub fn get(self: Pin<&Self>) -> u64 {
        self.project_ref().inner.get().other.get().unwrap().value
    }
}

pub fn link_up(left: Pin<&Pair>, right: Pin<&Pair>) {
    let left = unsafe { left.project_ref().inner.get_extended() };
    let right = unsafe { right.project_ref().inner.get_extended() };
    left.other.set(Some(right));
    right.other.set(Some(left));
}

fn main() {
    let pair_1 = Pair::new(10);
    let pair_2 = Pair::new(20);
    pin_mut!(pair_1);
    pin_mut!(pair_2);

    link_up(pair_1.as_ref(), pair_2.as_ref());

    println!("Pair 2 value: {}", pair_1.as_ref().get());
    println!("Pair 1 value: {}", pair_2.as_ref().get());
}
```

License: MIT