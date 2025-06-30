# 预先导入模块增加

## 概述

- `TryInto`，`TryFrom` 和 `FromIterator` 特性现在已经是预定义的一部分。
- 这可能会使调用 trait 方法变得模糊，这可能会导致一些代码无法编译。

## Details

[标准库的预先导入模块(prelude)](https://doc.rust-lang.org/stable/std/prelude/index.html)包含了每个模块中自动导入的所有内容。它包含一些常用项，如 `Option`，`Vec`，`drop`，和 `Clone`。

Rust 编译器优先考虑手动导入的项，而不是 prelude 中的项，以确保添加的 prelude 中的项不会破坏任何现有代码。例如，如果你有一个名为 `example` 的 crate 或模块，其中包含一个 `pub struct Option;`，
那么使用 `use example::*;` 将使 `Option` 明确地引用来自 `example`，而不是来自标准库。

然而，向 prelude 中添加一个 _trait_ 可能会以一种微妙的方式破坏现有的代码。举个例子，调用来自 `MyTryInto` 的 `x.try_into()`，如果在编译时也导入了 `std` 的 `TryInto`，可能会编译失败，因为 `try_into` 的调用变得模糊，它既可能来自 `MyTryInto` 也可能来自标准库。这就是为什么我们还没有在预定义中添加 `TryInto`，有很多代码会因此而中断。

作为解决方案，Rust 2021 将使用一个新的 prelude。它与当前版本完全相同，只是增加了三个新内容：

- [`std::convert::TryInto`](https://doc.rust-lang.org/stable/std/convert/trait.TryInto.html)
- [`std::convert::TryFrom`](https://doc.rust-lang.org/stable/std/convert/trait.TryFrom.html)
- [`std::iter::FromIterator`](https://doc.rust-lang.org/stable/std/iter/trait.FromIterator.html)

跟踪问题可以在[这里](https://github.com/rust-lang/rust/issues/85684)找到。

## 代码迁移

As a part of the 2021 edition a migration lint, `rust_2021_prelude_collisions`, has been added in order to aid in automatic migration of Rust 2018 codebases to Rust 2021.

In order to migrate your code to be Rust 2021 Edition compatible, run:

```sh
cargo fix --edition
```

The lint detects cases where functions or methods are called that have the same name as the methods defined in one of the new prelude traits. In some cases, it may rewrite your calls in various ways to ensure that you continue to call the same function you did before.

If you'd like to migrate your code manually or better understand what `cargo fix` is doing, below we've outlined the situations where a migration is needed along with a counter example of when it's not needed.

### Migration needed

#### Conflicting trait methods

When two traits that are in scope have the same method name, it is ambiguous which trait method should be used. For example:

```rust
trait MyTrait<A> {
  // This name is the same as the `from_iter` method on the `FromIterator` trait from `std`.  
  fn from_iter(x: Option<A>);
}

impl<T> MyTrait<()> for Vec<T> {
  fn from_iter(_: Option<()>) {}
}

fn main() {
  // Vec<T> implements both `std::iter::FromIterator` and `MyTrait` 
  // If both traits are in scope (as would be the case in Rust 2021),
  // then it becomes ambiguous which `from_iter` method to call
  <Vec<i32>>::from_iter(None);
}
```

We can fix this by using fully qualified syntax:

```rust,ignore
fn main() {
  // Now it is clear which trait method we're referring to
  <Vec<i32> as MyTrait<()>>::from_iter(None);
}
```

#### Inherent methods on `dyn Trait` objects

Some users invoke methods on a `dyn Trait` value where the method name overlaps with a new prelude trait:

```rust
mod submodule {
  pub trait MyTrait {
    // This has the same name as `TryInto::try_into`
    fn try_into(&self) -> Result<u32, ()>;
  }
}

// `MyTrait` isn't in scope here and can only be referred to through the path `submodule::MyTrait`
fn bar(f: Box<dyn submodule::MyTrait>) {
  // If `std::convert::TryInto` is in scope (as would be the case in Rust 2021),
  // then it becomes ambiguous which `try_into` method to call
  f.try_into();
}
```

Unlike with static dispatch methods, calling a trait method on a trait object does not require that the trait be in scope. The code above works 
as long as there is no trait in scope with a conflicting method name. When the `TryInto` trait is in scope (which is the case in Rust 2021),
this causes an ambiguity. Should the call be to `MyTrait::try_into` or `std::convert::TryInto::try_into`?

In these cases, we can fix this by adding an additional dereferences or otherwise clarify the type of the method receiver. This ensures that 
the `dyn Trait` method is chosen, versus the methods from the prelude trait. For example, turning `f.try_into()` above into `(&*f).try_into()` 
ensures that we're calling `try_into` on the `dyn MyTrait` which can only refer to the `MyTrait::try_into` method.

### No migration needed

####  Inherent methods

Many types define their own inherent methods with the same name as a trait method. For instance, below the struct `MyStruct` implements `from_iter` which shares the same name with the method from the trait `FromIterator` found in the standard library:

```rust
use std::iter::IntoIterator;

struct MyStruct {
  data: Vec<u32>
}

impl MyStruct {
  // This has the same name as `std::iter::FromIterator::from_iter`
  fn from_iter(iter: impl IntoIterator<Item = u32>) -> Self {
    Self {
      data: iter.into_iter().collect()
    }
  }
}

impl std::iter::FromIterator<u32> for MyStruct {
    fn from_iter<I: IntoIterator<Item = u32>>(iter: I) -> Self {
      Self {
        data: iter.into_iter().collect()
      }
    }
}
```

Inherent methods always take precedent over trait methods so there's no need for any migration.

### Implementation Reference

The lint needs to take a couple of factors into account when determining whether or not introducing 2021 Edition to a codebase will cause a name resolution collision (thus breaking the code after changing edition). These factors include:

- Is the call a [fully-qualified call] or does it use [dot-call method syntax]?
  - This will affect how the name is resolved due to auto-reference and auto-dereferencing on method call syntax. Manually dereferencing/referencing will allow specifying priority in the case of dot-call method syntax, while fully-qualified call requires specification of the type and the trait name in the method path (e.g. `<Type as Trait>::method`)
- Is this an [inherent method] or [a trait method]?
  - Inherent methods that take `self` will take priority over `TryInto::try_into` as inherent methods take priority over trait methods, but inherent methods that take `&self` or `&mut self` won't take priority due to requiring a auto-reference (while `TryInto::try_into` does not, as it takes `self`)
- Is the origin of this method from `core`/`std`? (As the traits can't have a collision with themselves)
- Does the given type implement the trait it could have a collision against?
- Is the method being called via dynamic dispatch? (i.e. is the `self` type `dyn Trait`)
  - If so, trait imports don't affect resolution, and no migration lint needs to occur

[fully-qualified call]: https://doc.rust-lang.org/reference/expressions/call-expr.html#disambiguating-function-calls
[dot-call method syntax]: https://doc.rust-lang.org/reference/expressions/method-call-expr.html
[inherent method]: https://doc.rust-lang.org/reference/items/implementations.html#inherent-implementations
[a trait method]: https://doc.rust-lang.org/reference/items/implementations.html#trait-implementations
