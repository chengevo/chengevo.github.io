+++
title = "Lifetime in Rust"
publishDate = 2024-11-29T15:27:00+08:00
draft = false
+++

Lifetime is a complex and often confusing concept in Rust. When discussing lifetimes, we usually refer to the lifetime of  references. However, the concept of lifetime is not restricted to references. Variables and data in Rust also have lifetimes.  We’ll discuss the lifetime of variables later. For now, let’s start with the lifetime of data and references. [Rust RFC 2094: Non-lexical Lifetimes](https://rust-lang.github.io/rfcs/2094-nll.html) provides precise definitions for these two types of lifetimes:

> 1.  The lifetime of a reference, corresponding to the span of time in which that reference is used.
> 2.  The lifetime of a value, corresponding to the span of time before that value gets freed (or, put another way, before the destructor for the value runs).

To distinguish the two, the lifetime of a value will be referred to as the\*scope of the value\*. The lifetime of a reference cannot outlive the scope of the value it refers to. Let’s see an example:

```rust { linenos=true, linenostart=1, anchorlinenos=true, lineanchors=org-coderef--9c6c47 }
fn main() {
    let mut v = vec![1, 2, 3];
    let len = v.len();
    v.push(4);
}
```

-   line [2](#org-coderef--9c6c47-2) creates a value, which is a vector. The scope of this value lasts from line [2](#org-coderef--9c6c47-2) to line [4](#org-coderef--9c6c47-4).
-   line [3](#org-coderef--9c6c47-3) creates a reference to the value `v` (`len()` method takes an immutable reference). The lifetime of this reference is confined to this line.


## The Lifetime of a Variable {#the-lifetime-of-a-variable}

If a variable is bound to a reference, that variable also has a lifetime, defined as the span of code in which it can safely be dereferenced[^fn:1]. Let's see an example:

<a id="code-snippet--lifetime-variable"></a>
```rust { linenos=true, linenostart=1, anchorlinenos=true, lineanchors=org-coderef--362c91 }
fn main() {
    let mut a;          // 'a
    let b = vec![1];
    let c = vec![2];

    a = &b;             // 'b
    a = &c;             // 'c
    println!("{a:?}");
}
```

1.  `&b` creates a reference with a lifetime of `'b`
2.  `&c` creates a reference with a lifetime of `'c`
3.  the variable `a` is bound to a reference to a `Vec<i32>`. It's lifetime, `'a` spans from line [3](#org-coderef--9c6c47-3) to line [8](#org-coderef--362c91-8), which is the union of `'b` and `'c`.


## Variable, value and reference {#variable-value-and-reference}

The book "The Rust Programming Language" provides an example to introduce the concept of lifetimes:

```rust { linenos=true, linenostart=1, anchorlinenos=true, lineanchors=org-coderef--d0315f }
fn main() {
    let r;                // ---------+-- 'a
                          //          |
    {                     //          |
        let x = 5;        // -+-- 'b  |
        r = &x;           //  |       |
    }                     // -+       |
                          //          |
    println!("r: {r}");   //          |
}                         // ---------+
```

This is how the book explains this code block:

> Here, we’ve annotated the lifetime of r with 'a and the lifetime of x with 'b. As you can see, the inner 'b block is much smaller than the outer 'a lifetime block. At compile time, Rust compares the size of the two lifetimes and sees that r has a lifetime of 'a but that it refers to memory with a lifetime of 'b. The program is rejected because 'b is shorter than 'a: the subject of the reference doesn’t live as long as the reference.

This explanation mixes three concepts:

-   lifetime of a refernce: `&x`
-   lifetime of a variable: `'a`
-   the scope of a value: `'b`

A clearer explanation might be:

-   variable `r` binds to a reference to a `i32`, andits lifetime spans from line 2 (when it's initialized) to line [3](#org-coderef--9c6c47-3)
-   at line [[out_scope_value] [(a)], a reference to `x` is created and assigned to variable `r`. The lifetime of this reference `'b` , is confined to this line because at line [4](#org-coderef--9c6c47-4) the inner scope ends, invalidating the reference.
-   the compiler rejects the code at line [3](#org-coderef--9c6c47-3) because it tries to access an invalid reference


## Assignment Operator Invalidates a Reference {#assignment-operator-invalidates-a-reference}

```rust
fn main() {
    let mut v = Box::new(1);
    let a = &v;
    v = Box::new(2);
    println!("{v:?}");
}
```

-   variable `a` contains a reference to a `Box<i32>`
-   at line 4, the variable `v` is assigned a new value, the assignment runs the destructor of the value previously pointed to by this variable[^fn:2]. Because `a` still contains a reference to the old `Box<i32>`, this reference is invalidated.


### View Lifetime as Data Flow {#view-lifetime-as-data-flow}

In the book _Rust for Rustaceans[^fn:3]_, Jon Gjengset proposed a different approach to think about lifetimes:

> It(theborrow checker) does this by tracing the path back to where 'a starts—where the reference was taken—from the point of use and checking that there are no conflicting uses along that path.

```rust { linenos=true, linenostart=1, anchorlinenos=true, lineanchors=org-coderef--734273 }
fn main() {
    let mut v = Box::new(-1);
    let mut a = &v;             // 'a

    for i in 0..10 {
        println!("{a}");        // 'a
        v = Box::new(i);
        a = &v;                 // 'a
    }

    println!("{a}");            // 'a
}
```

-   the liftime `'a` starts at line [3](#org-coderef--734273-3) when a reference to `v` is created. It spans from line [3](#org-coderef--734273-3) to line [6](#org-coderef--734273-6).
-   `'a` ends at line [7](#org-coderef--734273-7) because the [assignment operator](#assignment-operator-invalidates-a-reference) destroyed the Box `a` refers to
-   `'a` starts again when variable `a` is updated at line [8](#org-coderef--734273-8)
    -   if the code loops back to line [6](#org-coderef--734273-6), `a` still has a valid reference. The lifetime `'a` will end at line [6](#org-coderef--734273-6) again and begin the next create-then-destroy cycle.
    -   if the code continues to the final print statement, `a` still has a valid reference . The lifetime `'a` now spans from line [8](#org-coderef--734273-8) to [11](#org-coderef--734273-11).


## The Scope of a Lifetime {#the-scope-of-a-lifetime}

The lifetime of a reference lasts only for those portions of the function in which the reference may later be used[^fn:4], instead of encompassing the entire scope of the variable, which usually stretches to the end of the code block. Consider the following code:

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    let a = &mut v;                 // 'a
    a.push(4);                      // 'a
    println!("len: {}", v.len());
}
```

At line 3, we take a mutable reference to `v`. The lifetime  `'a` lasts from line 3 to line 4. Because this reference is last used in line 4,  the compiler considers the lifetime to end here instead of extending to line 6.

Understanding the scope of a reference allows us to refine the borrowing rules :

1.  **At runtime**, you can have either one mutable reference or any number of immutable references to access a value
2.  there can be multiple mutable references existing in the same block, as long as their lifetimes don't overlap.
3.  a mutable reference and immutable references can exist in the same block only if the lifetimes of the immutable references do not overlap with the lifetime of the mutable reference

Let's explore more examples.


### Examples Violating Borrowing Rules {#examples-violating-borrowing-rules}

```rust
let mut s = String::from("hello");
let r1 = &s;
let r2 = &s;
let r3 = &mut s;

// scope of mutable reference overlaps with immutable references
println!("{r1}, {r2}, {r3}");
```

```rust
let mut v = vec![1, 2, 3, 4];
let first = &v[0];
// incorrect, mutable borrow used here, starts and ends here
v.push(5);
println!("{:?}", first);
```


### Examples Adhering to Borrowing Rules {#examples-adhering-to-borrowing-rules}

```rust
let mut s = String::from("hello");
let r1 = &s;
let r2 = &s;
println!("{r1}, {r2}"); // lifetime of r1 and r2 ends here

let r3 = &mut s;  // lifetime of r3 starts here
println!("{r3}"); // lifetime of  r3 ends here, not overlapping with immutable references
```


## Multiple generic lifetimes {#multiple-generic-lifetimes}

There are times when multiple generic lifetimes are necessary, either in a function or in a struct. Consider this function, which takes a string slice `s` and a delimiter `d`. This function aims to find the first occurrence of `d`, and return the part of `s` that appears before the delimiter. There are two things worth noting:

1.  the delimiter is also a string slice because it may contain multiple characters
2.  right now this function only takes one generic lifetime `'a`, however,this will raise problems, which we will discuss later.

<!--listend-->

```rust
fn before_delimiter<'a>(s: &'a str, d: &'a str) -> &'a str {
    let pos = s.find(d).unwrap();
    &s[0..pos]
}
```

Assume you write another function `before_char`, which works like `before_delimiter`. The only difference is that it returns the part of thestring slice before a character.

```rust
fn before_char(s: &str, c: char) -> &str {
    let d = c.to_string();
    before_delimiter(s, &d)
}

fn main() {
    let s = "hello world".to_string();
    let c = 'o';
    let before = before_char(&s, c);
    assert_eq!(before, "hell");
}
```

Sadly the compiler will reject the function `before_char`:

> cannot return value referencing local variable `d`

The reason for this error is that, in the function `before_delimiter`, the return value and two parameters have the same lifetime. When this function is called, the generic lifetime is substituted with a concrete lifetime, which is the smaller lifetime of `s` and `d`. In this example, `before_delimiter` is called inside `before_char`, and the smaller lifetime is the lifetime of the string reference converted from `c`: `&c.to_string()`. However, this lifetime is associated with a variable local to function `before_char()`.

The solution to this problem is to make the parameters in `before_delimiter` have two distinct lifetimes, and the return value should be assigned the same lifetime as `s`:

```rust
fn before_delimiter<'a, 'b>(s: &'a str, d: &'b str) -> &'a str {
    let pos = s.find(d).unwrap();
    &s[0..pos]
}
```

This example teaches us that when assigning generic lifetimes to parameters and return values, we should think about which value the returned reference refers to. In `before_delimiter`, the returned reference and the parameter `s` should point to the same value in  memory, which means they should share the same lifetime. With that in mind, let's look at another example.


## A reference to a reference {#a-reference-to-a-reference}

```rust
// ERROR, require lifetime annotation
fn strtok(s: &mut &str, delimiter: char) -> str {
    if let Some(pos) = s.find(delimiter) {
        let pre = &s[0..pos];
        *s = &s[(pos+1)..];
        pre
    } else {
        let pre = *s;
        *s = "";
        pre
    }
}

fn main() {
    let str = "hello world hello Rust".to_string();
    let mut remainder = str.as_str();
    let mut words = vec![];

    while remainder != "" {
        let before = strtok(&mut remainder, ' ');
        words.push(before);
    }

    assert_eq!(words, ["hello", "world", "hello", "Rust"]);
}
```

What `strtok` does is try to find the delimiter in a string slice. If found, it returns the part of the string slice before the first occurrence of the delimiter and makes the parameter point to what's left in the string slice. For example, if you pass in `"hello world"` and `'o'` , this function should return "hell", and its parameter should point to `" world"` . The code won't compile for now because we need to add lifetime annotations to `strtok`. But before that, let's think about what `&mut &str` means.


### &amp;mut &amp;str {#and-mut-and-str}

The type `&mut &str` is a mutable reference to a shared string slice. Let's look at the following example.

```rust
fn main() {
    let s = "hello world".to_string();
    let mut a = s.as_str();
    let b = &mut a;
    foo(b);
    assert_eq!(a, "hello Rust");
}

fn foo(s: &mut &str) {
    *s = &"hello Rust";
}
```

1.  variables such as `a` and `b` are just named locations in the stack.
2.  `a` stores a reference to a shared string slice; you can make `a` hold another reference because `a` is mutable.
3.  `b` also stores a reference, in this case it's a reference to `a`. However, because `b` isn't mutable, you cannot make `b` hold another reference.


### Add lifetime annotation {#add-lifetime-annotation}

There are two lifetimes in the parameter `s`: `&'a mut &'b str`. Lifetime `'b` denotes  how long the reference to the string slice lasts, and lifetime `'a` denotes how long the reference `s` will exist. Since the return value and the reference to the string slice(`'b` associated with) point to the same value in the heap,  the return value should have the  lifetime `'b`. Because lifetime `'a` isn't needed in the function, we can leave this lifetime annotation empty. Now the function signature of `strtok` should look like this:

```rust
fn strtok(s: &mut &'a str, delimiter: char) -> 'a str { //.. }
```

[^fn:1]: [As you can see, the lifetime 'p is part of the type of the variable p. It indicates the portions of the control-flow graph where p can safely be dereferenced](https://rust-lang.github.io/rfcs/2094-nll.html#what-is-a-lifetime-and-how-does-it-interact-with-the-borrow-checker)
[^fn:2]: <https://doc.rust-lang.org/reference/destructors.html>
[^fn:3]: <https://rust-for-rustaceans.com/>
[^fn:4]: <https://rust-lang.github.io/rfcs/2094-nll.html#the-rough-outline-of-our-solution>
