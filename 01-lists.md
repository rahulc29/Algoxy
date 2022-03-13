# Lists 

Firstly, we present a few perspectives to think about lists :
## Perspectives

### High-level Declarative Perspective

We'd pick a functional language for this, and this is where algebraic types come in :

```fsharp
type List<'a> = Empty | Node of 'a * List<'a>
```

The `|` operator represents disjoint union and `*` represents cartesian product.

### High-level Imperative Perspective

Let's say we're in a garbage collected language with references and inheritance. 

We can easily do this in a language like Kotlin :

```kotlin
sealed class List<T> {
    class Empty<T> : List<T>()
    class Node<T>(val data: T, val next: List<T>): List<T>()
}
```

This is not as pretty as the Declarative Perspective but at least there's pattern matching!

```kotlin
when (list) {
    is List.Node<Int> -> list.data
    is List.Empty<*> -> null
} // return type of when is `Int?`
```

### Low-level Imperative Perspective

Let us start by implementing a simple list imperatively, in Rust. 

```rust 
struct List<T> {
    data: T,
    next: Box<List<T>>
}
```

This code is not particularly generic, making it for more generic requires the use of lifetimes :
```rust
struct List<T, 'a> {
    data: T,
    next: &'a List<T, 'a>
}
```
Also, we have currently used one single lifetime `'a` for all nodes in our list. 
We might want to change it later.

Do note that this `List` is immutable. This is intentional : by default, everything in Rust is immutable.

We can make it mutable as follows :

```rust
struct List<T, 'a> {
    data: T,
    next: &mut 'a List<T, 'a>
}
```

We can also implement this type algebraically : 
```rust
enum List<T, 'a> {
    Empty,
    Node {
        data: T,
        next: &'a List<T, 'a>
    }
}
```

Also, we are currently constrained in using only one kind of reference (native `&` and `&mut`). In reality, we would want our `List` to be _polymorphic_ over all possible kinds of references.

We can do this with Rust's `Deref` trait. 

```rust
enum List<T> {
    Empty,
    Node {
        data: T,
        next: dyn Deref<Target = List<T>>
    }
}
```

The low-level implementation certainly requires a lot more work due to the lack of the garbage collector!

## Equality of Lists

>For list of type $A$, suppose we can test if any two elements $x$, $y$ $\in$ $A$ are equal,
define an algorithm to test if two lists are identical.

First we define a recursive algorithm with HLFP : 

```fsharp
let rec Equals a b elementEquals = 
    match a with 
    | [] -> match b with 
        | [] -> true 
        | head::tail -> false 
    | aHead::aTail -> match b with 
        | [] -> false 
        | bHead::bTail -> (elementEquals aHead bHead) && (Equals aTail bTail elementEquals)
```

This algorithm is elegant and readable but suffers from consuming $O(n)$ stack space. 

Let us make it _tail-recursive_!

```fsharp
let Equals a b elementEquals = 
    let rec Loop a b result = 
        if !result then result else 
            match a with 
            | [] -> match b with 
                | [] -> true 
                | head::tail -> false 
            | aHead::aTail -> match b with 
                | [] -> false 
                | bHead::bTail -> Loop aTail bTail (result && (elementEquals aHead bHead))
    Loop a b true
```

To summarise, we check if the heads are equal, if yes, we recursively do the same for the tails.

In Rust we might do this :

```rust
fn <T> equals(a: &List<T>, b: &List<T>) -> bool where T : Eq {
    match a {
        Empty => {
            match b {
                Empty => true 
                Node(_, _) => false
            }
        }
        Node(a_head, a_tail) => {
            match b {
                Empty => false 
                Node(b_head, b_tail) => (b_head == a_head) && equals(a_tail, b_tail)
            }
        }
    }
}
```

Rust allows mutations, and we can exploit that to make an iterative version 
```rust
fn <T> equals(a: &List<T>, b: &List<T>) -> bool where T : Eq {
    let mut result = true;
    let mut a_eff = a;
    let mut b_eff = b;
    while (result) {
        match a_eff {
            Empty => match b_eff {
                Empty => {
                    result = true;
                    break;
                }
                Node(_, _) => {
                    result = false;
                }
            }
            Node(a_head, a_tail) => {
                match b_eff {
                    Empty => {
                        result = false;
                    }
                    Node(b_head, b_tail) => {
                        if (a_head != b_head) {
                            result = false;
                            break;
                        }
                        a_eff = a_tail;
                        b_eff = b_tail;
                    }
                }
            }
        }
    }
    return result;
}
```
