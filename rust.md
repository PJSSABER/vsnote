#### String \ &str \ literal
ref to :https://doc.rust-lang.org/book/ch04-03-slices.html?highlight=string#string-slices
```Rust
let s = String::from("hello world");  // String
let hello = &s[0..5];   // &str,  immutable reference, do not have ownership!!
let tmp = "Hello, world!";  // tmp is $str, "Hello, world!" is a &'static str literal stored in const variables area
/*
    String: have two parts, one on stack contains {ptr, len, capacity}; one on heap contains contents
    &str: have one part on stack, {ptr, len}
*/

let s1 = s.clone();  // deep copy
let s2 = s;
/*
When we assign s to s2, the String data is copied, meaning we copy the pointer, the length, and the capacity that are on the stack. a shallow copy
After this, Rust considers s as no longer valid. s2 takes the ownership!!!

The mechanics of passing a value to a function are similar to those when assigning a value to a variable. Passing a variable to a function will move or copy, just as assignment does
*/

/*
    stack only data have trait Copy
*/
let x = 5;
let y = x;
println!("x = {}, y = {}", x, y);

/* &String can be treat as a reference to a string slice (&str) because the String type implements the Deref trait to automatically coerce into a string slice when necessary.
*/
```
#### move semantics and ownership

##### reference and borrowing
A reference is like a pointer in that it’s an address we can follow to access the data stored at that address; that data is owned by some other variable. Unlike a pointer, a reference is guaranteed to point to a valid value of a particular type for the life of that reference.
Mutable references have one big restriction: if you have a mutable reference to a value, you can have no other references to that value.
no Dangling References

#### partial move in match expression
& denotes that your pattern expects a reference to an object. Hence & is a part of said pattern: &Foo matches different objects than Foo does.

ref indicates that you want a reference to an unpacked value. It is not matched against: Foo(ref foo) matches the same objects as Foo(foo).

#### trait Self self

self: the instance of the implementing type on which the method is being called. It's similar to this in other programming languages.

Self: When used as a return type within a trait, Self refers to the type that implements the trait. 

```rust
trait MyTrait {
    fn create() -> Self;
}

struct MyStruct;

impl MyTrait for MyStruct {
    fn create() -> Self {
        MyStruct
    }

    fn do_something(&self) {
        
    }
}

fn main() {
    let new_instance = MyStruct::create();
}
```

Traits as Parameters
https://doc.rust-lang.org/book/ch10-02-traits.html#traits-as-parameters

pub fn notify<T: Summary>(item: &T)
pub fn notify(item: &impl Summary)
fn some_function<T: Display + Clone, U: Clone + Debug>(t: &T, u: &U) -> i32 {
return type: fn returns_summarizable() -> impl Summary

#### lifetime
While lifetimes and scopes are often referred to together, they are not the same
apostrophe
using lifetimes requires generics
```rust
foo<'a>
// `foo` has a lifetime parameter `'a`; this lifetime syntax indicates that the lifetime of foo may not exceed that of 'a

```

#### Box
What remains on the stack(also can be on heap) is the pointer to the heap data

 stack        heap
 Box<T> ----> data
                .
                .
                .


#### closure
by default using borrow; (mutable if needed)

why move?
If you want to force the closure to take ownership of the values it uses in the environment even though the body of the closure doesn’t strictly need ownership, you can use the move keyword before the parameter list.
```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    let handle = thread::spawn(|| {
        println!("Here's a vector: {:?}", v);
    });

    drop(v); // oh no!   using move to make code run

    handle.join().unwrap();
}
```

#### concurrency

```rust
mpsc::channel();
//built consumer-producer
```

#### Macro
exclamation mark : !
writing code that writes other code: metaprogramming
1. parameters changeble
2. compile time processing, like implement a trait

Four series:
- declarative macros with macro_rules!
- Custom #[derive] macros that specify code added with the derive attribute used on structs and enums
- Attribute-like macros that define custom attributes usable on any item
- Function-like macros

