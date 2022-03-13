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

We will also sometimes use the following formulation, especially for mutating algorithms:
```kotlin
class ListNode<T>(var value: T, var next: ListNode<T>?)
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

This `List` is actually _infinite_ because references in Rust are non-nullable, to mitigate this we implement this type algebraically : 
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

Another way to implement a `List<T>` is to use the `Option<T>` monad in Rust.

```rust
struct List<T> {
    data: T,
    next: Option<Box<List<T>>>
}
```

We use `Box<T>` because using a generic `Deref` is unsized and `Option` requires a sized type argument.

Of course, `Box<T>` is a sized implementation of `Deref` but that would require the following constraints :
```rust
struct List<T, R> where R : Deref<Target = List<T>> + Sized {
    data: T,
    next: Option<Box<List<T>>>
}
```

The low-level implementation certainly requires a lot more work due to the lack of the garbage collector!

## Equality of Lists

::: {.exercise}
For list of type $A$, suppose we can test if any two elements $x$, $y$ $\in$ $A$ are equal,
define an algorithm to test if two lists are identical.
:::

:::{.solution}
First we define a recursive algorithm with HLFP : 

```fsharp
let rec Equals a b elementEquals = 
    match a with 
    | [] -> match b with 
        | [] -> true 
        | head::tail -> false 
    | aHead::aTail -> match b with 
        | [] -> false 
        | bHead::bTail -> 
            (elementEquals aHead bHead) && (Equals aTail bTail elementEquals)
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
                | bHead::bTail -> 
                    Loop aTail bTail (result && (elementEquals aHead bHead))
    Loop a b true
```

:::

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
The iterative version is (expectedly) less pretty. 

The iterative Rust version is what you would get after applying the Tail-Call Opitimization to the tail-recursive formulation.

## Length

Length is easily computed recursively :

```fsharp
let rec Length list = 
    match list with 
    | [] -> 0 
    | head::tail -> 1 + Length tail
```

This consume $O(n)$ stack space though. 

Tail-recursive version :

```fsharp
let Length list = 
    let rec Loop list answer = 
        match list with 
        | [] -> answer 
        | head::tail -> Loop tail (answer + 1)
    Loop list 0
```

Iterative version :

```rust
fn <T> length(list: &List<T>) -> usize {
    let mut answer = 0usize;
    let mut current = list.next.as_ref();
    while current.is_some() {
        answer += 1;
        current = current.unwrap().next.as_ref();
    }
    return answer;
}
```

Observe that we are only performing _borrows_ and there are no moves. The default semantics in Rust are move semantics, so we explicitly use `.as_ref()` to make sure we're only borrowing.

## Indices 

To get the element at the $i$th index we perform the following computation:

```fsharp
exception ListIndexException
let rec ElementAt list index = 
    match list with 
    | [] -> raise ListIndexException
    | head::tail -> if index = 0 then head else ElementAt tail (index - 1)
```

Any index $i$ is either the $0$th index of a list or the $i-1$th index of the sublist.

Conveniently for us, this code is already tail-recursive.

Iterative version:

```rust
fn index<'a, T>(list: Option<&'a List<'a, T>>, index: usize) -> Result<&'a T, String> {
    if let None = list {
        return Err(String::from("Indexing undefined for empty lists"));
    }
    let mut current = list;
    let mut i = index;
    while current.is_some() {
        if i == 0 {
            return Ok(&current.unwrap().data);
        }
        i -= 1;
        current = current.unwrap().next;
    }
    // got to none
    // irrespective of i it is an out of bounds
    return Err(format!("Index {} is out of bounds", i));
}
```

Here we have used the generic-lifetime-bound implementation in Rust. The reason for doing this is simple : we need to ensure that the lifetime of the returned reference live alongside the list.
This is easier to do using parameterised lifetimes than using `Box<T>`. 

We use the `Result<T, E>` type to signify that the computation may fail. 

::: {.exercise} 
In the iterative `index(list, index)` algorithm, what is the behavior when `list` is empty? 
What is the behavior when `index` is out of the bound or negative?
:::

::: {.solution}
For the iterative Rust variant, we have used the `struct` definition based on `Option`, as such, truely empty lists are impossible. 
To represent empty lists, we set `Option<&List<'a, T>> = None`. This case is accounted for and erred over.

If `index` is out of bounds, we get an `Err` variant in the `Result`. 
:::

We will now use only functional and Kotlin code except when the imperative needs to be studied extensively.

The reason is that Rust's type system will get in our way so it is better to satisfy the type system later.

## Last and Init

To compute the last element : 

```fsharp
exception UndefinedException
let rec Last list = 
    match list with 
    | [] -> raise UndefinedException
    | head::tail -> 
        match tail with 
        | [] -> head
        | _::subTail -> Last subTail
```

This function is already tail-recursive so we need not care any further :D

In Kotlin :
```kotlin
fun<T> last(list: ListNode<T>): T {
    var current = list 
    var next = current.next
    while (next != null) {
        next = list.next
        current = current.next
    }   
    return current.value
}
```

We use the `ListNode<T>` formulation since this algorithm is rather mutation heavy.

To compute the init of a list (the sublist before the last element)

```fsharp
exception UndefinedException
let rec Init list =
    match list with 
    | [] -> raise UndefinedException
    | head::tail -> 
        match tail with 
        | [] -> []
        | _::_ -> head::(Init tail)
```

We will return to an iterative version later after covering how to reverse a list.

## Reverse Index 

