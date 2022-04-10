---
layout: post
title: 어려웠던 'already borrowed' 이슈
---

RefCell 다루면서 'already borrowed' 발생하는 이슈를 여러번 겪게 되는데 그중에서 찾기 어려웠던 케이스를 남겨봅니다.

```rust
#[derive(Clone)]
struct B {
    value: i32,
}

#[derive(Clone)]
struct A {
    b: Rc<RefCell<B>>,
}
```

이런 형태의 구조체를 많이 다루고 있습니다. 구조체 내부에서 `Rc<RefCell<T>>`의 형태로 공유된 메모리를 다루고 있는 경우입니다. 여기서 B의 일종의 getter로써 아래와 같은 함수를 정의했습니다.

```rust
fn get_b_mut(a: &A) -> RefMut<'_, B> {
    a.b.borrow_mut()
}
```

이런 상황에서 아래의 코드에서 어떤 문제가 발생할까요?

```rust
fn foo(a: &A, value: i32) {
    assert_eq!(get_b_mut(a).value, value);
}


fn main() {
    let a = A {
        b: Rc::new(RefCell::new(B {
            value: 42,
        }))
    };

    foo(&a, get_b_mut(&a).value);
}
```

제 기존의 직관과 다르게 `foo`를 호출하면 'already borrowed'가 발생하게 됩니다. `foo`의 두번째 인자에 대한 expression을 계산할때 borrow가 일어나지만, 그 (i32타입의) 값을 복사를 통해 전달하고 나면 foo가 호출되는 시점에서는 borrow가 종료되어야 하는게 아닐까요? 막연히 이런 식으로 생각했던것 같습니다.
하지만 실제 일어나는 일은 이런것 같습니다. `get_b_mut()` 함수를 통해 생성되는 `RefMut` 타입의 (임시) 값은 foo함수 호출이 포함되어 있는 전체 statement가 종료되는 시점에서 소멸됩니다. [여기](https://doc.rust-lang.org/reference/destructors.html#temporary-scopes)에서 그 조건이 나와있는데, 생각해보면 납득이 갑니다. 예를 들어 만약에 생성된 임시 값을 reference로 함수 인자로 전달한다고 하면, 그 값이 함수 호출 이후까지는 살이 있는것이 보장되어야 할테니까 그래야 할것 같아요.

참조: 
[Rust Playground 링크](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=7da8a9b4a6a257097f5d53de260fbf3f)
