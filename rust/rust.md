# Comprehensive Rust Reference Guide

## 📚 Table of Contents
1. [Basic Syntax & Style](#basic-syntax--style)
2. [Ownership & Memory Management](#ownership--memory-management)
3. [References & Borrowing](#references--borrowing)
4. [Generics](#generics)
5. [Traits](#traits)
6. [Trait Bounds](#trait-bounds)
7. [Generic vs Trait Objects](#generic-vs-trait-objects)
8. [Object Safety](#object-safety)
9. [Data Structures](#data-structures)
10. [Structs & Methods](#structs--methods)
11. [Control Flow](#control-flow)
12. [Collections](#collections)
13. [Strings](#strings)
14. [Modules & Crates](#modules--crates)
15. [Error Handling](#error-handling)
16. [Concurrency & Threading](#concurrency--threading)
17. [Smart Pointers](#smart-pointers)
18. [Closures](#closures)
19. [Attributes](#Attributes)
20. [Macros](#macros)
21. [Database Programming](#database-programming)
22. [Unsafe Code](#unsafe-code)
23. [Tokio](#tokio)
24. [Interview Questions](#interview-questions)

---

## Basic Syntax & Style

### Formatting Rules
- **Indentation**: Use **4 spaces**, not tabs
- **Return values**: The final expression in a function block is the return value (no semicolon needed)
- **Function location**: Rust doesn't care where functions are defined, only that they're in a visible scope

### Conditionals
- **`if` expressions**: Must always have a Boolean condition (be explicit)
- No automatic type coercion to boolean

---

## Ownership & Memory Management

### The Three Rules of Ownership
1. Each value in Rust has an **owner**
2. There can only be **one owner at a time**
3. When the owner goes out of scope, the value is **dropped**

### Memory Model Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    RUST MEMORY MODEL                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  STACK                           HEAP                        │
│  ├─ Known, fixed size            ├─ Unknown/dynamic size    │
│  ├─ Fast access                  ├─ Slower access           │
│  ├─ Automatic management         ├─ Manual management       │
│  └─ Not allocating               └─ Requires allocation     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Stack vs Heap
- **Stack**: Known, fixed size at compile time. Fast. Automatic cleanup.
- **Heap**: Unknown/dynamic size. Requires allocation. Returns a pointer. Less organized.
- **Heap Safety**: Less safe because data is accessible to all threads

### The `drop` Function
- Automatically called when a variable goes out of scope (at closing `}`)
- Returns memory to the allocator
- No manual memory management needed (no garbage collector!)

### Ownership in Action

```rust
fn main() {
    let s = String::from("hello");  // s owns the String
    takes_ownership(s);              // s's value MOVES
                                     // s is no longer valid!

    let x = 5;                       // x owns the i32
    makes_copy(x);                   // i32 is Copy, so x is still valid
                                     // x can still be used

} // x goes out of scope, then s
  // Since s was moved, nothing happens for s

fn takes_ownership(some_string: String) {
    println!("{}", some_string);
} // some_string dropped, memory freed

fn makes_copy(some_integer: i32) {
    println!("{}", some_integer);
} // Nothing special happens (Copy type)
```

### Copy vs Clone

| Feature | Copy | Clone |
|---------|------|-------|
| **How it works** | Implicit bitwise copy | Explicit method call `.clone()` |
| **Syntax** | Automatic | Must call `.clone()` |
| **Performance** | Always cheap (stack) | Can be expensive (heap) |
| **Memory** | Stack only | Stack or Heap |
| **Requirements** | Must also implement Clone | Can exist alone |

#### Types that are `Copy`
- All integer types: `i32`, `u64`, etc.
- Floating point: `f32`, `f64`
- `bool`, `char`
- Tuples and arrays (if all elements are `Copy`)
- References: `&T` (but NOT `&mut T`)

#### Rules for `Copy`
- Must be cheap to copy (stack only)
- Cannot contain heap data (`String`, `Vec`, `Box` are NOT `Copy`)
- Must also implement `Clone`
- All fields must be `Copy`

### What Ownership Solves
- Tracking what code uses heap data
- Minimizing duplicate data on heap
- Cleaning up unused data (no memory leaks)
- No garbage collector needed!

---

## References & Borrowing

### The Rules of References
1. At any given time, you can have **either**:
   - **One mutable reference** (`&mut T`), OR
   - **Any number of immutable references** (`&T`)
2. References must **always be valid** (no dangling references)

### Mutable Reference Restriction
- **Big restriction**: If you have a mutable reference to a value, you can have **no other references** to that value
- **Benefit**: Prevents data races at compile time!

### Data Race Conditions
A data race occurs when:
1. Two or more pointers access the same data at the same time
2. At least one pointer is writing to the data
3. No synchronization mechanism exists

**Rust prevents data races at compile time!**

### Lifetime

References are pointers to data owned by someone else. When you borrow, you're creating a reference:
```rust
fn calculate_length(s: &String) -> usize {
    s.len()
} // s goes out of scope, but doesn't drop the String

let s1 = String::from("hello");
let len = calculate_length(&s1);  // borrowing s1
// s1 is still valid here
```
The & creates a reference, and the function doesn't take ownership, so the original owner keeps the data alive.
Lifetimes are Rust's way of tracking how long references are valid. Every reference has a lifetime - the scope for which that reference is valid. The compiler uses lifetime annotations to ensure references never outlive the data they point to.
Most of the time, lifetimes are implicit and the compiler infers them. But sometimes you need to annotate them explicitly.

Lifetimes prevent dangling references - pointers to memory that's been freed. Here's what they prevent:

```rust
// Compiler prevents this:
let r;
{
    let x = 5;
    r = &x;  // ❌ ERROR: x doesn't live long enough
}
println!("{}", r);  // x is already dropped!
```
The compiler rejects this because the reference would outlive the data it points to.
Here's when you need explicit lifetime annotations:
```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```
The `'a` annotation tells the compiler: 'the returned reference will live as long as the shorter of the two input references.' This lets the compiler verify that wherever we use the result, both inputs are still valid.

#### Non-Lexical Lifetimes (NLL)

Modern Rust is smart about when borrows actually end:

```rust
let mut s = String::from("hello");

let r1 = &s;
let r2 = &s;
println!("{} and {}", r1, r2);
// r1 and r2 are no longer used after this point

let r3 = &mut s;  // This is OK! r1 and r2's borrows have ended!
r3.push_str(" world");
```

**Lifetime Elision Rules:**
You don't always need to write lifetime annotations. The compiler has three elision rules that infer lifetimes:

1) Each input reference gets its own lifetime.
2) If there's exactly one input lifetime, it's assigned to all output lifetimes.
3) If there's a &self or &mut self parameter, its lifetime is assigned to all outputs.

---
**'static means the reference is valid for the entire program duration**
---
**Though Rust often dereferences automatically!**

```rust
let s = String::from("hello");
let r = &s;

println!("{}", r.len());  // Rust automatically dereferences for method calls
```

## Generics

### What are Generics?
Generics allow you to write code that works with multiple types while maintaining type safety and performance. Type parameters are declared using angle brackets `<>`.

**Key Benefit**: Zero runtime cost - compiler creates specialized code for each type (monomorphization).

### Generic Functions

```rust
// Basic generic function
fn print_value<T>(value: T) 
where
    T: std::fmt::Display
{
    println!("Value: {}", value);
}

// Usage
print_value(42);           // Works with i32
print_value("hello");      // Works with &str
print_value(3.14);         // Works with f64
```

### Generic Structs

```rust
// Single type parameter
struct Container<T> {
    value: T,
}

impl<T> Container<T> {
    fn new(value: T) -> Self {
        Container { value }
    }
    
    fn get(&self) -> &T {
        &self.value
    }
}

// Usage
let int_container = Container::new(42);
let string_container = Container::new(String::from("hello"));
```

### Multiple Type Parameters

```rust
struct Pair<T, U> {
    first: T,
    second: U,
}

impl<T, U> Pair<T, U> {
    fn new(first: T, second: U) -> Self {
        Pair { first, second }
    }
    
    fn get_first(&self) -> &T {
        &self.first
    }
    
    fn get_second(&self) -> &U {
        &self.second
    }
}

// Usage - different types for each field
let pair = Pair::new(42, "hello");
let another = Pair::new(String::from("world"), 3.14);
```

### Generic Enums

```rust
// Option-like enum
enum MyOption<T> {
    Some(T),
    None,
}

// Result-like enum
enum MyResult<T, E> {
    Ok(T),
    Err(E),
}

// Usage
let some_number: MyOption<i32> = MyOption::Some(5);
let no_number: MyOption<i32> = MyOption::None;
```

### Generic Methods with Specific Implementations

```rust
struct Point<T> {
    x: T,
    y: T,
}

// Generic implementation for all types
impl<T> Point<T> {
    fn new(x: T, y: T) -> Self {
        Point { x, y }
    }
}

// Specific implementation only for f64
impl Point<f64> {
    fn distance_from_origin(&self) -> f64 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}

// Usage
let int_point = Point::new(5, 10);
let float_point = Point::new(3.0, 4.0);
println!("Distance: {}", float_point.distance_from_origin()); // Only works with f64
```

### Real-World Example: Generic Stack

```rust
struct Stack<T> {
    items: Vec<T>,
}

impl<T> Stack<T> {
    fn new() -> Self {
        Stack { items: Vec::new() }
    }
    
    fn push(&mut self, item: T) {
        self.items.push(item);
    }
    
    fn pop(&mut self) -> Option<T> {
        self.items.pop()
    }
    
    fn is_empty(&self) -> bool {
        self.items.is_empty()
    }
    
    fn size(&self) -> usize {
        self.items.len()
    }
}

// Usage with different types
let mut int_stack = Stack::new();
int_stack.push(1);
int_stack.push(2);
println!("{:?}", int_stack.pop()); // Some(2)

let mut string_stack = Stack::new();
string_stack.push(String::from("hello"));
string_stack.push(String::from("world"));
println!("{:?}", string_stack.pop()); // Some("world")
```

---

## Traits

### What are Traits?
Traits define shared behavior - like interfaces in other languages. They specify a set of methods that types must implement.

### Defining and Implementing Traits

```rust
// Define a trait
trait Describe {
    fn describe(&self) -> String;
}

// Implement for different types
struct Dog {
    name: String,
}

impl Describe for Dog {
    fn describe(&self) -> String {
        format!("A dog named {}", self.name)
    }
}

struct Cat {
    name: String,
}

impl Describe for Cat {
    fn describe(&self) -> String {
        format!("A cat named {}", self.name)
    }
}

// Usage
let dog = Dog { name: String::from("Buddy") };
let cat = Cat { name: String::from("Whiskers") };
println!("{}", dog.describe());
println!("{}", cat.describe());
```

### Default Implementations

```rust
trait Summary {
    fn summarize_author(&self) -> String;
    
    // Default implementation
    fn summarize(&self) -> String {
        format!("(Read more from {}...)", self.summarize_author())
    }
}

struct Article {
    author: String,
    content: String,
}

impl Summary for Article {
    fn summarize_author(&self) -> String {
        self.author.clone()
    }
    // Uses default summarize() implementation
}

let article = Article {
    author: String::from("John"),
    content: String::from("Rust is great!"),
};
println!("{}", article.summarize());
```

---

## Trait Bounds

### Basic Trait Bounds

```rust
// Function that works with any type implementing Display
fn print_it<T: std::fmt::Display>(value: T) {
    println!("{}", value);
}

// Multiple trait bounds using +
fn process<T: Clone + std::fmt::Display>(value: T) {
    let copy = value.clone();
    println!("{}", copy);
}
```

### Where Clauses (Cleaner Syntax)

```rust
// Instead of this:
fn complex<T: Display + Clone, U: Clone + Debug>(t: T, u: U) -> i32 {
    // ...
}

// Use where clause for better readability:
fn complex<T, U>(t: T, u: U) -> i32
where
    T: Display + Clone,
    U: Clone + Debug,
{
    // ...
}
```

### Example: Finding the Largest

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    
    largest
}

// Usage
let numbers = vec![34, 50, 25, 100, 65];
println!("Largest: {}", largest(&numbers));

let chars = vec!['y', 'm', 'a', 'q'];
println!("Largest: {}", largest(&chars));
```

---

## Generic vs Trait Objects

### Static Dispatch

A static dispatch occurs at compile time, where the compiler determines which function to call based on the static type of a variable or expression. With static dispatch, there is no runtime overhead, and the static dispatch approach is widely used to achieve better performance since it enables the compiler to generate a more efficient code without the overhead.

A static dispatch is achieved through the use of generics and traits. When a generic function is called with a concrete type, the compiler generates a specialized version of the function for that type. Traits allow for a form of ad-hoc polymorphism, where different types can implement the same trait and provide their own implementations of its methods.

### Generics (Static Dispatch/ Monomorphization)

**Pros:** Fast, compiler optimizes, can inline  
**Cons:** Code size increases (monomorphization), can't mix types

```rust
trait Animal {
    fn make_sound(&self) -> String;
}

struct Dog;
struct Cat;

impl Animal for Dog {
    fn make_sound(&self) -> String {
        "Woof!".to_string()
    }
}

impl Animal for Cat {
    fn make_sound(&self) -> String {
        "Meow!".to_string()
    }
}

// Generic function - compiler creates separate code for each type
fn animal_sound<T: Animal>(animal: &T) -> String {
    animal.make_sound()
}

// Usage - each type known at compile time
let dog = Dog;
let cat = Cat;
println!("{}", animal_sound(&dog));
println!("{}", animal_sound(&cat));

// ❌ Cannot mix types in a collection
// let animals = vec![Dog, Cat]; // Error!
```
### Dynamic Dispatch

Dynamic dispatch in Rust refers to the process of determining which implementation of a method to call at runtime based on the type of object the method is called on.

The dynamic dispatch is implemented using trait objects, which allow a value of any type that implements a given trait to be treated as a single type. When a method is called on a trait object, Rust uses a vtable to determine which implementation of the method to call.

### Trait Objects (Dynamic Dispatch)

**Pros:** Can mix types, smaller code size, flexible  
**Cons:** Slight runtime overhead (vtable lookup), can't inline

```rust
// Function using trait object
fn animal_sound_dyn(animal: &dyn Animal) -> String {
    animal.make_sound()
}

// ✅ CAN mix types in collections!
let animals: Vec<Box<dyn Animal>> = vec![
    Box::new(Dog),
    Box::new(Cat),
    Box::new(Dog),
];

for animal in animals {
    println!("{}", animal.make_sound());
}
```

### Real-World Example: GUI System

```rust
trait Widget {
    fn draw(&self);
}

struct Button {
    label: String,
}

struct TextBox {
    content: String,
}

impl Widget for Button {
    fn draw(&self) {
        println!("Drawing button: {}", self.label);
    }
}

impl Widget for TextBox {
    fn draw(&self) {
        println!("Drawing textbox: {}", self.content);
    }
}

// With trait objects - can store mixed types!
struct Screen {
    components: Vec<Box<dyn Widget>>,
}

impl Screen {
    fn new() -> Self {
        Screen { components: Vec::new() }
    }
    
    fn add_component(&mut self, component: Box<dyn Widget>) {
        self.components.push(component);
    }
    
    fn render(&self) {
        for component in &self.components {
            component.draw();
        }
    }
}

// Usage
let mut screen = Screen::new();
screen.add_component(Box::new(Button { label: "Submit".to_string() }));
screen.add_component(Box::new(TextBox { content: "Email".to_string() }));
screen.add_component(Box::new(Button { label: "Cancel".to_string() }));
screen.render();
```

### Comparison Table

| Feature | Generic `<T: Trait>` | Trait Object `dyn Trait` |
|---------|---------------------|--------------------------|
| **Dispatch** | Static (compile-time) | Dynamic (runtime) |
| **Performance** | Faster (no vtable) | Slightly slower |
| **Code size** | Larger (copies for each type) | Smaller |
| **Mixed types** | ❌ No | ✅ Yes |
| **Inline-able** | ✅ Yes | ❌ No |
| **Trait bounds** | Multiple traits OK | Limited (object-safe only) |

### When to Use Each

**Use Generics when:**
- Performance is critical
- Types known at compile time
- Working with single type at a time
- Need aggressive compiler optimization

**Use Trait Objects when:**
- Need to store mixed types in collections
- Types determined at runtime
- Building plugin systems
- Code size matters more than speed

---

## Object Safety

### What is Object Safety?
A trait is "object-safe" if it can be used as a trait object (`dyn Trait`). Not all traits can be trait objects! So it means a trait is object safe if it can be used for dynamic dispatch, meaning the compiler can call its methods at runtime through a vtable without needing to know the concrete type implementing the trait

### Rules for Object Safety

#### ❌ Rule 1: No Generic Methods

```rust
trait Container {
    fn add<T>(&mut self, item: T);  // Generic method - NOT object safe!
}

// This won't compile:
// let container: Box<dyn Container> = Box::new(MyContainer);
// Error: trait cannot be made into an object

// ✅ Solution: Make the trait generic instead
trait Container<T> {
    fn add(&mut self, item: T);  // No longer generic method
}

let container: Box<dyn Container<i32>> = Box::new(IntContainer);  // Works!
```

**Why?** The compiler can't create a vtable with infinite generic variations.

#### ❌ Rule 2: No Methods Returning `Self`

```rust
trait Cloneable {
    fn clone_me(&self) -> Self;  // Returns Self - NOT object safe!
}

// This won't compile:
// let obj: Box<dyn Cloneable> = Box::new(MyStruct);
// Error: method returns `Self`

// ✅ Solution: Return a trait object
trait Cloneable {
    fn clone_box(&self) -> Box<dyn Cloneable>;
}

impl Cloneable for MyStruct {
    fn clone_box(&self) -> Box<dyn Cloneable> {
        Box::new(MyStruct { value: self.value })
    }
}

let obj: Box<dyn Cloneable> = Box::new(MyStruct { value: 42 });
let cloned = obj.clone_box();  // Works!
```

**Why?** With type erasure, the compiler doesn't know what concrete type to return.

#### ❌ Rule 3: No Associated Functions (without `self`)

The trait Factory is not object safe because its method fn create() -> Self has no self parameter, making it an associated function rather than a method. In Rust, only trait methods that take self (i.e., &self, &mut self, or self) can be called on trait objects via dynamic dispatch. Associated functions like create do not have a receiver, so there's no way to know which concrete type to create when calling the function on a trait object like Box<dyn Factory>; the compiler would lose track of what Self actually is in that context.​

```rust
trait Factory {
    fn create() -> Self;  // No self parameter - NOT object safe!
}

// This won't compile:
// let factory: Box<dyn Factory> = Box::new(Product);
// Error: associated function has no `self` parameter

// ✅ Solution: Add a self parameter
trait Factory {
    fn create(&self) -> Box<dyn Factory>;  // Now has &self
}

let factory: Box<dyn Factory> = Box::new(ProductFactory);
let new_factory = factory.create();  // Works!
```

**Why?** Associated functions are called on the type itself, but trait objects erase the type.

#### ❌ Rule 4: No `where Self: Sized`

```rust
trait MyTrait {
    fn method(&self) where Self: Sized;  // NOT object safe!
}
```

**Why?** Trait objects are `!Sized` (unsized), which contradicts the `Sized` requirement.

### Object Safety Checklist

A trait is object-safe if ALL methods:
- ✅ Have a `self` parameter (`&self`, `&mut self`, or `self`)
- ✅ Don't return `Self` (unless wrapped in `Box<dyn Trait>`)
- ✅ Aren't generic
- ✅ Don't have `where Self: Sized`

### Real Example: Clone Trait

```rust
// The standard Clone trait is NOT object safe
pub trait Clone {
    fn clone(&self) -> Self;  // Returns Self!
}

// ❌ This doesn't work:
// let obj: Box<dyn Clone> = Box::new(String::from("hello"));

// ✅ Object-safe alternative:
trait CloneBox {
    fn clone_box(&self) -> Box<dyn CloneBox>;
}

impl<T: Clone + 'static> CloneBox for T {
    fn clone_box(&self) -> Box<dyn CloneBox> {
        Box::new(self.clone())
    }
}

struct Dog {
    name: String,
}

impl Clone for Dog {
    fn clone(&self) -> Self {
        Dog { name: self.name.clone() }
    }
}

let dog: Box<dyn CloneBox> = Box::new(Dog { name: "Buddy".to_string() });
let cloned = dog.clone_box();  // Works!
```

---

## Data Structures

### Stack vs Heap Structures

```
Stack-allocated (Copy)        Heap-allocated (Clone required)
├─ Primitives (i32, f64)      ├─ String
├─ Arrays [T; N]              ├─ Vec<T>
├─ Tuples (T, U)              ├─ Box<T>
└─ References &T              ├─ HashMap<K, V>
                              └─ Custom structs with heap data
```

### Choosing Data Structures

#### Use `Vec<T>` when:
- Dynamic array needed
- All values are the same type
- Values stored next to each other in memory

#### Use `HashMap<K, V>` when:
- Associate arbitrary keys with values
- Need a cache
- Want a map with no extra functionality

#### Use `BTreeMap<K, V>` when:
- Need to find smallest/largest key-value pair
- Want to find keys larger/smaller than something
- Need all entries in sorted order
- Want a map sorted by its keys

**Note**: In theory, BST is optimal (log₂n lookups), but in practice, it's inefficient for modern computer architectures due to cache locality issues.

---

## Structs & Methods

### Struct Mutability
- **Important**: The entire struct instance must be mutable
- Rust **does NOT allow** marking only certain fields as mutable

```rust
struct User {
    username: String,
    email: String,
}

let mut user1 = User {  // Entire struct is mutable
    username: String::from("user"),
    email: String::from("user@example.com"),
};

user1.email = String::from("new@example.com");  // ✅ OK
```

### Struct Update Syntax

```rust
let user2 = User {
    email: String::from("another@example.com"),
    ..user1  // Copy remaining fields from user1
};
```

### Methods vs Associated Functions

#### Methods
- Take `self`, `&self`, or `&mut self` as first parameter
- Called on an instance: `instance.method()`

#### Associated Functions
- Don't have `self` as first parameter
- Called on the type: `Type::function()`
- Often used as constructors (commonly named `new`)

```rust
impl Rectangle {
    // Associated function (constructor)
    fn square(size: u32) -> Self {
        Self {
            width: size,
            height: size,
        }
    }
    
    // Method
    fn area(&self) -> u32 {
        self.width * self.height
    }
}

// Usage
let sq = Rectangle::square(10);  // Associated function
let area = sq.area();             // Method
```

### Taking Ownership with `self`
- Using just `self` (not `&self`) takes ownership
- Rare technique
- Used when transforming `self` into something else
- Prevents caller from using original instance after transformation

---

## Control Flow

### Loop Labels
- Use loop labels for nested loops
- Apply `break` or `continue` to specific loops
- **Labels must begin with a single quote** (`'`)

```rust
'outer: loop {
    loop {
        break 'outer;  // Breaks the outer loop
    }
}
```

### `if let` - Concise Control Flow
- Combines `if` and `let` for pattern matching
- Less verbose than full `match` when you only care about one case

```rust
let some_value = Some(3);

// Instead of:
match some_value {
    Some(3) => println!("three"),
    _ => (),
}

// Use:
if let Some(3) = some_value {
    println!("three");
}
```

---

## Collections

### Vectors (`Vec<T>`)

#### Key Properties
- Store variable number of values
- **All values must be same type**
- Values stored **next to each other in memory**

#### Creating Vectors

```rust
let mut v = Vec::new();
let v = vec![1, 2, 3];  // vec! macro
let mut v = vec![100, 32, 57];
```

#### Modifying Vectors

```rust
let mut v = vec![100, 32, 57];
for i in &mut v {
    *i += 50;  // Dereference and modify
}
```

#### Important Memory Consideration
- Adding elements may require reallocation
- If not enough space, allocates new memory and copies old elements
- References to elements become invalid after reallocation
- **Borrowing rules prevent this bug at compile time**

#### Restriction in Loops
```rust
let mut v = vec![1, 2, 3];

for i in &v {
    v.push(4);  // ❌ ERROR: Can't modify while iterating
}
```
The reference held by the `for` loop prevents simultaneous modification.

### Hash Maps

#### Use Cases
- Associate arbitrary keys with arbitrary values
- Implement a cache
- Need a map with no extra functionality

### Hash Sets
- Store unique keys only (no values)
- Similar to `HashMap` but key-only

---

## Strings

### String Types

| Type | Description | Location | Mutable |
|------|-------------|----------|---------|
| `String` | Growable, heap-allocated | Heap | Yes |
| `&str` | String slice (view into UTF-8 bytes) | Can be anywhere | No |

### String Literals
```rust
let s = "Hello, world!";  // Type: &str
```
- String literals are `&str` (immutable references)
- Stored inside the binary
- Point to a specific location in the binary

### Creating Strings

```rust
let s = String::new();
let s = String::from("hello");
let s = "hello".to_string();
```

### Why No Indexing?

```rust
let s = String::from("hello");
let h = s[0];  // ❌ ERROR: Cannot index into String
```

**Reason**: Indexing operations should be O(1), but Rust can't guarantee this with UTF-8 strings. Rust would need to walk through from the beginning to validate character boundaries.

---

## Modules & Crates

### Crate Types

#### Library Crate
- No `main` function
- Doesn't compile to executable
- Defines functionality for other projects
- Example: `rand` crate

**Note**: When Rustaceans say "crate", they usually mean library crate.

#### Binary Crate
- Has `main` function
- Compiles to executable

### Module Declaration

Declare modules in the crate root file:

```rust
mod garden;  // Compiler looks for module code in:
```

1. Inline: `mod garden { ... }`
2. In file: `src/garden.rs`
3. In file: `src/garden/mod.rs`

### Crates.io
- Central Rust package registry
- Publish and share crates
- Search for reusable libraries

---

## Error Handling

# Comparing Option and Result in Rust

This guide provides a comprehensive comparison of Option and Result types for error handling in Rust.

---

### OPTION<T>

**Purpose:** Represents the presence or absence of a value.

**Definition:**
```rust
enum Option<T> {
    Some(T),    // Contains a value
    None,       // No value present
}
```

**When to Use:**
- When a value might not exist, but that's **not an error**
- When something is optional or nullable
- When absence is a normal, expected outcome

**Common Scenarios:**
```rust
// Finding an element (might not exist)
let numbers = vec![1, 2, 3, 4, 5];
let found = numbers.iter().find(|&&x| x == 3);  // Option<&i32>

// Accessing a HashMap (key might not exist)
use std::collections::HashMap;
let mut map = HashMap::new();
map.insert("key", "value");
let value = map.get("key");  // Option<&str>

// First/last element of a collection
let first = numbers.first();  // Option<&i32>

// Parsing without detailed error info
let maybe_number: Option<i32> = "42".parse().ok();
```

**Working with Option:**
```rust
let x: Option<i32> = Some(5);

// Pattern matching
match x {
    Some(value) => println!("Got: {}", value),
    None => println!("Got nothing"),
}

// Using if let
if let Some(value) = x {
    println!("Got: {}", value);
}

// Unwrapping (panics if None)
let value = x.unwrap();

// Providing a default
let value = x.unwrap_or(0);

// Chaining operations
let result = x
    .map(|v| v * 2)
    .filter(|v| v > &5)
    .unwrap_or(0);
```

---

### RESULT<T, E>

**Purpose:** Represents success or failure of an operation, with error details.

**Definition:**
```rust
enum Result<T, E> {
    Ok(T),      // Operation succeeded with value T
    Err(E),     // Operation failed with error E
}
```

**When to Use:**
- When an operation can fail and you need to know **why**
- When you need to propagate detailed error information
- For I/O operations, parsing, validation, or any fallible operation

**Common Scenarios:**
```rust
// File I/O
use std::fs::File;
let file = File::open("data.txt");  // Result<File, std::io::Error>

// Parsing with error details
let number: Result<i32, std::num::ParseIntError> = "42".parse();

// Network operations
// HTTP requests, database queries, etc.

// Custom validation
fn validate_age(age: i32) -> Result<i32, String> {
    if age >= 0 && age <= 120 {
        Ok(age)
    } else {
        Err(String::from("Age must be between 0 and 120"))
    }
}
```

**Working with Result:**
```rust
let result: Result<i32, String> = Ok(42);

// Pattern matching
match result {
    Ok(value) => println!("Success: {}", value),
    Err(error) => println!("Error: {}", error),
}

// Using if let
if let Ok(value) = result {
    println!("Success: {}", value);
}

// Unwrapping (panics if Err)
let value = result.unwrap();

// Better unwrapping with custom panic message
let value = result.expect("Failed to get value");

// Providing a default
let value = result.unwrap_or(0);

// The ? operator (propagates errors)
fn read_file() -> Result<String, std::io::Error> {
    let content = std::fs::read_to_string("file.txt")?;  // Returns early if Err
    Ok(content)
}
```

---

### KEY DIFFERENCES

| Aspect | Option | Result |
|--------|--------|--------|
| **Purpose** | Absence of value | Success or failure |
| **Error Info** | No error information (just None) | Contains error details (type E) |
| **Semantic Meaning** | "Might not exist" | "Might fail" |
| **Use Case** | Optional values, lookups | Operations that can error |
| **Variants** | Some(T) / None | Ok(T) / Err(E) |

---

### CONVERTING BETWEEN THEM

**Option to Result:**
```rust
let opt: Option<i32> = Some(5);

// Convert None to an error
let res: Result<i32, &str> = opt.ok_or("Value was None");

// Lazy error creation
let res: Result<i32, String> = opt.ok_or_else(|| {
    String::from("Computed error message")
});
```

**Result to Option:**
```rust
let res: Result<i32, String> = Ok(5);

// Discard the error, convert to Option
let opt: Option<i32> = res.ok();  // Some(5) if Ok, None if Err

// Keep only the error
let err_opt: Option<String> = res.err();  // None if Ok, Some(error) if Err
```

---

### PRACTICAL EXAMPLE: Comparing Both

```rust
use std::collections::HashMap;

struct UserDatabase {
    users: HashMap<u32, String>,
}

impl UserDatabase {
    // Option: User might not exist, but that's not an error
    fn find_user(&self, id: u32) -> Option<&String> {
        self.users.get(&id)
    }
    
    // Result: Validation can fail with specific reasons
    fn add_user(&mut self, id: u32, name: String) -> Result<(), String> {
        if name.is_empty() {
            return Err(String::from("Name cannot be empty"));
        }
        
        if self.users.contains_key(&id) {
            return Err(String::from("User ID already exists"));
        }
        
        self.users.insert(id, name);
        Ok(())
    }
    
    // Result: Getting a user that MUST exist
    fn get_user(&self, id: u32) -> Result<&String, String> {
        self.users.get(&id)
            .ok_or_else(|| format!("User {} not found", id))
    }
}

fn main() {
    let mut db = UserDatabase {
        users: HashMap::new(),
    };
    
    // Using Result for operations that can fail
    match db.add_user(1, String::from("Alice")) {
        Ok(()) => println!("User added"),
        Err(e) => println!("Error: {}", e),
    }
    
    // Using Option for lookups
    match db.find_user(1) {
        Some(name) => println!("Found: {}", name),
        None => println!("User not found"),
    }
    
    // Using Result when we expect the user to exist
    match db.get_user(1) {
        Ok(name) => println!("User: {}", name),
        Err(e) => println!("Error: {}", e),
    }
}
```

---

### THE ? OPERATOR

Works with both, but behaves differently:

**With Option:**
```rust
fn get_first_char(text: Option<String>) -> Option<char> {
    let t = text?;  // Returns None if text is None
    t.chars().next()
}
```

**With Result:**
```rust
fn read_and_parse() -> Result<i32, Box<dyn std::error::Error>> {
    let content = std::fs::read_to_string("number.txt")?;  // Propagates error
    let number = content.trim().parse()?;  // Propagates parse error
    Ok(number)
}
```

---

### CHOOSING BETWEEN THEM

**Use Option when:**
- A value is genuinely optional
- Absence is not exceptional or unexpected
- You don't need to explain why something is missing
- Examples: first element, HashMap lookup, optional config values

**Use Result when:**
- An operation can fail in multiple ways
- You need to communicate WHY something failed
- Errors should be handled or propagated
- Examples: I/O, parsing, validation, network calls

**Sometimes both make sense:**
```rust
// Option: Item might not exist (normal)
fn find_item(id: u32) -> Option<Item> { ... }

// Result: Item SHOULD exist (error if not)
fn get_item(id: u32) -> Result<Item, NotFoundError> { ... }
```

---

## Concurrency & Threading

### Core Concepts
1. **Creating threads**: Run multiple pieces of code simultaneously
2. **Message-passing**: Channels send messages between threads
3. **Shared-state**: Multiple threads access shared data
4. **`Send` and `Sync` traits**: Extend concurrency guarantees

### Thread Lifecycle
**Important**: When the main thread completes, **all spawned threads are shut down**, whether or not they finished running.

### The `Send` Trait
- **Marker trait**
- Indicates ownership can be **transferred between threads**
- Types implementing `Send`: `String`, `Vec<T>`, most primitives

### The `Sync` Trait
- **Marker trait**
- Indicates it's **safe to reference from multiple threads**
- Types implementing `Sync`: `Arc<T>`, `Mutex<T>`

### Trait Comparison

| Trait | Concept | Typical Example | Opposite Example |
|-------|---------|-----------------|------------------|
| `Send` | Ownership can move across threads | `String`, `Vec<T>` | `Rc<T>` |
| `Sync` | Shared reference `&T` can be used across threads | `Arc<T>`, `Mutex<T>` | `RefCell<T>` |

### Thread Example with Closures

```rust
use std::thread;

let data = vec![1, 2, 3];

// `move` keyword transfers ownership to thread
thread::spawn(move || {
    println!("{:?}", data);
});
```

---

## Smart Pointers

Smart Pointers own or manage resource lifetimes:
Smart pointers like Box<T>, Rc<T>, and Arc<T> primarily focus on managing the ownership and lifetime of the data they point to, often handling memory deallocation or shared ownership.

Lazy Types manage initialization:
OnceCell and similar types are designed to ensure a value is initialized exactly once, on its first access, and then provide access to that initialized value. They are about controlled, single-assignment initialization rather than general resource management.

### `Box<T>` - Heap Allocation
- Stores data on the heap
- Stack stores pointer to heap data

### `Rc<T>` - Reference Counting (Single-threaded)

**Purpose**: Shared ownership in single-threaded scenarios

```rust
use std::rc::Rc;

let a = Rc::new(5);
let b = Rc::clone(&a);  // Increments reference count
let c = Rc::clone(&a);  // Now 3 owners
```

**Why needed?** Allows multiple parts of program to own the same data.

**Example use case**: Doubly-linked list where nodes need to be referenced by multiple other nodes.

### `Arc<T>` - Atomic Reference Counting (Multi-threaded)

**Purpose**: Thread-safe shared ownership

```rust
use std::sync::Arc;

let data = Arc::new(vec![1, 2, 3]);
let data_clone = Arc::clone(&data);

std::thread::spawn(move || {
    println!("{:?}", data_clone);
});
```

**When to use**: When you need `Rc` but across multiple threads.

### `RefCell<T>` - Interior Mutability (Single-threaded)

**Purpose**: Mutable borrows checked at **runtime** instead of compile time

```rust
use std::cell::RefCell;

let x = RefCell::new(5);
*x.borrow_mut() += 1;  // Mutable borrow
```

**Why needed?** Allows mutation through shared references when compiler can't prove safety at compile time.

### `Mutex<T>` - Mutual Exclusion (Multi-threaded)

**Purpose**: Thread-safe interior mutability

```rust
use std::sync::Mutex;

let m = Mutex::new(5);
{
    let mut num = m.lock().unwrap();
    *num = 6;
}  // Lock is released here
```

**Key Properties**:
- Only one thread can hold the lock at a time
- Prevents data races
- Blocks other threads until lock is available

### Atomic Types - Lock-free Concurrency

**Purpose**: Indivisible operations that are thread-safe without locks

Available types: `AtomicBool`, `AtomicIsize`, `AtomicUsize`, `AtomicPtr`, etc.

**Guarantee**: Operations are not susceptible to race conditions or data corruption.


### Introduction

Rust's ownership system is strict by design, but sometimes you need more flexibility. Smart pointers like `Rc`, `RefCell`, `Arc`, and `Mutex` help you work around these restrictions safely.

**Key Question**: Which smart pointer should you use for graphs?

**Quick Answer**:
- Single-threaded graphs: `Rc<RefCell<T>>`
- Multi-threaded graphs: `Arc<Mutex<T>>` or `Arc<RwLock<T>>`

---

### Rc (Reference Counted)

#### What It Does
Enables **multiple ownership** of the same data through reference counting.

#### Key Characteristics
- ✅ Multiple owners can share read access
- ✅ Single-threaded only
- ✅ Automatic cleanup when count reaches zero
- ❌ Immutable by default
- ❌ Small runtime overhead for counting

#### How It Works
```rust
use std::rc::Rc;

let data = Rc::new(vec![1, 2, 3]);
let reference1 = Rc::clone(&data); // Count: 2
let reference2 = Rc::clone(&data); // Count: 3

println!("Reference count: {}", Rc::strong_count(&data)); // 3

// When all references drop, data is freed
drop(reference1); // Count: 2
drop(reference2); // Count: 1
// data goes out of scope, Count: 0, memory freed
```

#### Common Use Cases

##### 1. Graphs with Shared Nodes
```rust
use std::rc::Rc;

struct Node {
    value: i32,
    children: Vec<Rc<Node>>,
}

let shared_child = Rc::new(Node {
    value: 5,
    children: vec![],
});

let parent1 = Node {
    value: 1,
    children: vec![Rc::clone(&shared_child)],
};

let parent2 = Node {
    value: 2,
    children: vec![Rc::clone(&shared_child)],
};

// shared_child has two parents!
```

##### 2. Shared Configuration
```rust
use std::rc::Rc;

struct Config {
    max_connections: u32,
    timeout: u64,
}

let config = Rc::new(Config {
    max_connections: 100,
    timeout: 30,
});

let service1 = Service::new(Rc::clone(&config));
let service2 = Service::new(Rc::clone(&config));
// Both services share the same config
```

##### 3. Caching
```rust
use std::rc::Rc;
use std::collections::HashMap;

let mut cache: HashMap<String, Rc<ExpensiveData>> = HashMap::new();

let data = Rc::new(compute_expensive_data());
cache.insert("key1".to_string(), Rc::clone(&data));
cache.insert("key2".to_string(), Rc::clone(&data));
// Same data cached under multiple keys
```

#### Important Notes
- `Rc::clone()` is cheap - it only increments a counter, doesn't copy data
- Watch out for reference cycles - use `Weak` to break them
- Cannot be sent across threads (not `Send`)

---

### RefCell (Reference Cell)

#### What It Does
Enables **interior mutability** - allows mutation through shared references by moving borrow checking to runtime.

#### Key Characteristics
- ✅ Mutate data through shared reference
- ✅ Enforces borrowing rules at runtime
- ✅ Single-threaded only
- ❌ Panics if borrowing rules violated
- ❌ Slight runtime overhead

#### Borrowing Rules (Enforced at Runtime)
- Many immutable borrows OR one mutable borrow
- Cannot have both at the same time
- Violating rules = **runtime panic**

#### How It Works
```rust
use std::cell::RefCell;

let data = RefCell::new(5);

// Multiple immutable borrows OK
{
    let read1 = data.borrow();
    let read2 = data.borrow();
    println!("{}, {}", read1, read2); // 5, 5
} // Borrows dropped here

// Mutable borrow (exclusive access)
{
    let mut write = data.borrow_mut();
    *write += 10;
} // Borrow dropped here

println!("{}", data.borrow()); // 15
```

#### What Causes Panics
```rust
use std::cell::RefCell;

let data = RefCell::new(5);

let read = data.borrow();
let write = data.borrow_mut(); // 💥 PANIC! Already borrowed immutably

// Or:
let write1 = data.borrow_mut();
let write2 = data.borrow_mut(); // 💥 PANIC! Already borrowed mutably
```

#### Common Use Cases

##### 1. Mocking and Testing
```rust
use std::cell::RefCell;

struct Logger {
    logs: RefCell<Vec<String>>,
}

impl Logger {
    fn log(&self, message: &str) {
        // Can mutate even though &self is immutable
        self.logs.borrow_mut().push(message.to_string());
    }
    
    fn get_logs(&self) -> Vec<String> {
        self.logs.borrow().clone()
    }
}
```

##### 2. Graphs that Need Mutation
```rust
use std::cell::RefCell;

struct Node {
    value: RefCell<i32>,
    visited: RefCell<bool>,
}

impl Node {
    fn mark_visited(&self) {
        *self.visited.borrow_mut() = true;
    }
    
    fn increment_value(&self) {
        *self.value.borrow_mut() += 1;
    }
}
```

##### 3. Lazy Initialization
```rust
use std::cell::RefCell;

struct Cache {
    data: RefCell<Option<Vec<i32>>>,
}

impl Cache {
    fn get_data(&self) -> Vec<i32> {
        let mut data = self.data.borrow_mut();
        if data.is_none() {
            *data = Some(expensive_computation());
        }
        data.as_ref().unwrap().clone()
    }
}
```

##### 4. Observer Pattern
```rust
use std::cell::RefCell;

struct Subject {
    observers: RefCell<Vec<Box<dyn Observer>>>,
}

impl Subject {
    fn notify(&self) {
        for observer in self.observers.borrow().iter() {
            observer.update();
        }
    }
    
    fn add_observer(&self, observer: Box<dyn Observer>) {
        self.observers.borrow_mut().push(observer);
    }
}
```

---

### Combining Rc and RefCell

#### Why Combine Them?
- `Rc`: Multiple ownership
- `RefCell`: Interior mutability
- Together: Multiple owners can all mutate shared data

#### The Power Combo
```rust
use std::rc::Rc;
use std::cell::RefCell;

struct Node {
    value: i32,
    next: Option<Rc<RefCell<Node>>>,
}

let node1 = Rc::new(RefCell::new(Node {
    value: 1,
    next: None,
}));

let node2 = Rc::new(RefCell::new(Node {
    value: 2,
    next: None,
}));

// Multiple ownership
let ref1 = Rc::clone(&node1);
let ref2 = Rc::clone(&node1);

// Mutation through any reference
node1.borrow_mut().value = 10;
node1.borrow_mut().next = Some(Rc::clone(&node2));

ref1.borrow_mut().value = 20; // Same node, different reference
println!("{}", node1.borrow().value); // 20
```

#### Real-World Example: Graph Implementation
```rust
use std::rc::Rc;
use std::cell::RefCell;

type NodeRef = Rc<RefCell<Node>>;

struct Node {
    id: usize,
    value: i32,
    neighbors: Vec<NodeRef>,
}

struct Graph {
    nodes: Vec<NodeRef>,
}

impl Graph {
    fn new() -> Self {
        Graph { nodes: vec![] }
    }
    
    fn add_node(&mut self, id: usize, value: i32) -> NodeRef {
        let node = Rc::new(RefCell::new(Node {
            id,
            value,
            neighbors: vec![],
        }));
        self.nodes.push(Rc::clone(&node));
        node
    }
    
    fn add_edge(&mut self, from: &NodeRef, to: &NodeRef) {
        from.borrow_mut().neighbors.push(Rc::clone(to));
    }
    
    fn dfs(&self, start: &NodeRef) {
        let mut visited = std::collections::HashSet::new();
        self.dfs_helper(start, &mut visited);
    }
    
    fn dfs_helper(&self, node: &NodeRef, visited: &mut std::collections::HashSet<usize>) {
        let node_borrow = node.borrow();
        if visited.contains(&node_borrow.id) {
            return;
        }
        
        visited.insert(node_borrow.id);
        println!("Visiting node {} with value {}", node_borrow.id, node_borrow.value);
        
        for neighbor in &node_borrow.neighbors {
            drop(node_borrow); // Release borrow before recursion
            self.dfs_helper(neighbor, visited);
            return; // Prevent use after drop
        }
    }
}

// Usage
let mut graph = Graph::new();
let node1 = graph.add_node(1, 10);
let node2 = graph.add_node(2, 20);
let node3 = graph.add_node(3, 30);

graph.add_edge(&node1, &node2);
graph.add_edge(&node2, &node3);
graph.add_edge(&node3, &node1); // Creates a cycle!

graph.dfs(&node1);
```

---

### Why Rc<Mutex> Doesn't Work in multithreads

#### The Problem
`Rc<Mutex<T>>` will compile but is a **design mistake** because you're mixing incompatible threading models.

#### Analysis

##### Rc: Single-Threaded
- Not thread-safe
- Fast (no atomic operations)
- **Cannot be sent between threads**
- Not `Send` or `Sync`

##### Mutex: Multi-Threaded
- Provides thread-safe exclusive access
- Overhead for synchronization
- Designed for concurrent access

#### Why It Doesn't Make Sense

```rust
use std::rc::Rc;
use std::sync::Mutex;
use std::thread;

// This compiles...
let data = Rc::new(Mutex::new(5));

// But this DOESN'T compile!
thread::spawn(move || {
    let mut val = data.lock().unwrap();
    *val += 1;
}); // Error: `Rc<Mutex<i32>>` cannot be sent between threads safely
```

**Compiler Error:**
```
error[E0277]: `Rc<Mutex<i32>>` cannot be sent between threads safely
   --> src/main.rs:7:5
    |
7   |     thread::spawn(move || {
    |     ^^^^^^^^^^^^^ `Rc<Mutex<i32>>` cannot be sent between threads safely
```

#### Issues with Rc<Mutex>

1. **Can't actually use across threads** - The whole point of `Mutex` is lost
2. **Unnecessary overhead** - Paying for thread-safety you can't use
3. **Wrong tool for the job** - `RefCell` is better for single-threaded mutation

#### The Right Combinations

```rust
// ✅ Single-threaded: Multiple ownership + mutation
use std::rc::Rc;
use std::cell::RefCell;
let data = Rc::new(RefCell::new(5));

// ✅ Multi-threaded: Thread-safe ownership + exclusive access
use std::sync::{Arc, Mutex};
let data = Arc::new(Mutex::new(5));

// ✅ Multi-threaded: Thread-safe ownership + read-heavy workloads
use std::sync::{Arc, RwLock};
let data = Arc::new(RwLock::new(5));
```

#### Comparison Table

| Combination | Compiles? | Thread-Safe? | Should Use? | Reason |
|-------------|-----------|--------------|-------------|---------|
| `Rc<RefCell<T>>` | ✅ | ❌ | ✅ | Perfect for single-threaded |
| `Arc<Mutex<T>>` | ✅ | ✅ | ✅ | Perfect for multi-threaded |
| `Arc<RwLock<T>>` | ✅ | ✅ | ✅ | Multi-threaded, read-heavy |
| `Rc<Mutex<T>>` | ✅ | ❌ | ❌ | Can't use across threads, unnecessary overhead |
| `Arc<RefCell<T>>` | ✅ | ❌ | ❌ | `RefCell` not thread-safe, causes data races |

#### Analogy
`Rc<Mutex<T>>` is like putting a high-security bank vault lock on a bedroom door in a studio apartment:
- You're living alone (`Rc` = single-threaded)
- You installed an expensive multi-person security system (`Mutex`)
- But you can never have multiple people anyway!
- A simple lock (`RefCell`) would work fine and cost less

---

### Smart Pointer Decision Guide

#### Decision Tree

```
Do you need multiple ownership?
├─ NO → Use `Box<T>` or direct ownership
└─ YES → Do you need mutability?
    ├─ NO → Use `Rc<T>` (single-thread) or `Arc<T>` (multi-thread)
    └─ YES → Do you need thread-safety?
        ├─ NO → Use `Rc<RefCell<T>>`
        └─ YES → Do you need frequent reads?
            ├─ NO → Use `Arc<Mutex<T>>`
            └─ YES → Use `Arc<RwLock<T>>`
```

#### Quick Reference

| Need | Single-Threaded | Multi-Threaded |
|------|----------------|----------------|
| Shared ownership (immutable) | `Rc<T>` | `Arc<T>` |
| Shared ownership + mutation | `Rc<RefCell<T>>` | `Arc<Mutex<T>>` |
| Shared ownership + read-heavy | `Rc<RefCell<T>>` | `Arc<RwLock<T>>` |
| Tree (no cycles) | `Box<T>` | `Box<T>` |

#### Performance Characteristics

| Type | Overhead | Thread-Safe | Runtime Checks |
|------|----------|-------------|----------------|
| `Box<T>` | None | N/A | No |
| `Rc<T>` | Reference counting | No | No |
| `Arc<T>` | Atomic counting | Yes | No |
| `RefCell<T>` | Small | No | Yes (borrow rules) |
| `Mutex<T>` | Synchronization | Yes | Yes (lock contention) |
| `RwLock<T>` | Synchronization | Yes | Yes (lock contention) |

---

## Graph Implementation Examples

### Single-Threaded Graph
```rust
use std::rc::Rc;
use std::cell::RefCell;
use std::collections::HashMap;

type NodeRef = Rc<RefCell<GraphNode>>;

struct GraphNode {
    id: usize,
    value: String,
    neighbors: Vec<NodeRef>,
}

struct Graph {
    nodes: HashMap<usize, NodeRef>,
}

impl Graph {
    fn new() -> Self {
        Graph {
            nodes: HashMap::new(),
        }
    }
    
    fn add_node(&mut self, id: usize, value: String) -> NodeRef {
        let node = Rc::new(RefCell::new(GraphNode {
            id,
            value,
            neighbors: vec![],
        }));
        self.nodes.insert(id, Rc::clone(&node));
        node
    }
    
    fn add_edge(&mut self, from_id: usize, to_id: usize) {
        if let (Some(from), Some(to)) = (self.nodes.get(&from_id), self.nodes.get(&to_id)) {
            from.borrow_mut().neighbors.push(Rc::clone(to));
        }
    }
    
    fn get_node(&self, id: usize) -> Option<NodeRef> {
        self.nodes.get(&id).map(|n| Rc::clone(n))
    }
    
    fn bfs(&self, start_id: usize) {
        use std::collections::{VecDeque, HashSet};
        
        let mut queue = VecDeque::new();
        let mut visited = HashSet::new();
        
        if let Some(start) = self.get_node(start_id) {
            queue.push_back(start);
            visited.insert(start_id);
            
            while let Some(node) = queue.pop_front() {
                let node_borrow = node.borrow();
                println!("Visiting: {} ({})", node_borrow.id, node_borrow.value);
                
                for neighbor in &node_borrow.neighbors {
                    let neighbor_id = neighbor.borrow().id;
                    if !visited.contains(&neighbor_id) {
                        visited.insert(neighbor_id);
                        queue.push_back(Rc::clone(neighbor));
                    }
                }
            }
        }
    }
}
```

### Multi-Threaded Graph
```rust
use std::sync::{Arc, Mutex};
use std::collections::HashMap;
use std::thread;

type NodeRef = Arc<Mutex<GraphNode>>;

struct GraphNode {
    id: usize,
    value: String,
    neighbors: Vec<NodeRef>,
}

struct Graph {
    nodes: Arc<Mutex<HashMap<usize, NodeRef>>>,
}

impl Graph {
    fn new() -> Self {
        Graph {
            nodes: Arc::new(Mutex::new(HashMap::new())),
        }
    }
    
    fn add_node(&self, id: usize, value: String) -> NodeRef {
        let node = Arc::new(Mutex::new(GraphNode {
            id,
            value,
            neighbors: vec![],
        }));
        
        let mut nodes = self.nodes.lock().unwrap();
        nodes.insert(id, Arc::clone(&node));
        node
    }
    
    fn add_edge(&self, from_id: usize, to_id: usize) {
        let nodes = self.nodes.lock().unwrap();
        
        if let (Some(from), Some(to)) = (nodes.get(&from_id), nodes.get(&to_id)) {
            let mut from_node = from.lock().unwrap();
            from_node.neighbors.push(Arc::clone(to));
        }
    }
    
    fn parallel_process(&self) {
        let nodes = self.nodes.lock().unwrap();
        let mut handles = vec![];
        
        for (_, node) in nodes.iter() {
            let node_clone = Arc::clone(node);
            let handle = thread::spawn(move || {
                let node = node_clone.lock().unwrap();
                println!("Processing node {} in thread {:?}", 
                         node.id, thread::current().id());
                // Do some work...
            });
            handles.push(handle);
        }
        
        for handle in handles {
            handle.join().unwrap();
        }
    }
}
```

---

### Best Practices

#### 1. Prefer Simple Ownership
Use smart pointers only when necessary. Start with simple ownership and upgrade when needed.

#### 2. Avoid Reference Cycles
Use `Weak<T>` to break cycles in `Rc`/`Arc`:

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>, // Weak to break cycle
    children: RefCell<Vec<Rc<Node>>>,
}
```

### 3. Keep Borrows Short
```rust
// ❌ Bad: Holding borrow too long
let data = RefCell::new(5);
let borrow = data.borrow();
// ... lots of code ...
println!("{}", borrow);

// ✅ Good: Short-lived borrows
let data = RefCell::new(5);
{
    let borrow = data.borrow();
    println!("{}", borrow);
} // Borrow dropped immediately
```

### 4. Use try_borrow for Safety
```rust
use std::cell::RefCell;

let data = RefCell::new(5);

match data.try_borrow_mut() {
    Ok(mut val) => *val += 1,
    Err(_) => println!("Already borrowed!"),
}
```

### 5. Document Thread Safety
Always document whether your types are thread-safe:

```rust
/// Single-threaded graph structure.
/// Not safe to share across threads.
struct SingleThreadGraph {
    nodes: Vec<Rc<RefCell<Node>>>,
}

/// Thread-safe graph structure.
/// Can be safely shared across threads.
struct ThreadSafeGraph {
    nodes: Vec<Arc<Mutex<Node>>>,
}
```

---

### Common Pitfalls

### 1. RefCell Panics
```rust
let data = RefCell::new(5);
let _read = data.borrow();
let _write = data.borrow_mut(); // 💥 PANIC at runtime
```

### 2. Reference Cycles Memory Leak
```rust
// Memory leak! Nodes never freed
let node1 = Rc::new(RefCell::new(Node::new()));
let node2 = Rc::new(RefCell::new(Node::new()));

node1.borrow_mut().next = Some(Rc::clone(&node2));
node2.borrow_mut().next = Some(Rc::clone(&node1)); // Cycle!
```

### 3. Deadlocks with Mutex
```rust
let data = Arc::new(Mutex::new(5));
let _lock1 = data.lock().unwrap();
let _lock2 = data.lock().unwrap(); // 💥 DEADLOCK
```

---

#### Conclusion

**Key Takeaways:**

1. **`Rc`** = Multiple ownership (single-threaded)
2. **`RefCell`** = Interior mutability (single-threaded)
3. **`Rc<RefCell<T>>`** = Multiple owners can all mutate (single-threaded)
4. **`Arc<Mutex<T>>`** = Thread-safe version of the above
5. **Don't mix threading models** - Use `Rc` with `RefCell`, `Arc` with `Mutex`

**Decision Process:**
1. Start with simple ownership
2. Need multiple owners? → `Rc` or `Arc`
3. Need mutation? → Add `RefCell` or `Mutex`
4. Need threads? → Use `Arc` + `Mutex`, not `Rc` + `RefCell`

---

## LRU Cache Implementation Guide

### Core Structure
- **Doubly-linked list**: Track recency of access
- **Tail**: Most recently used
- **Head**: Least recently used (evict from here)

### Key Trick: Dummy Nodes
Create dummy head and tail nodes and connect them together. This simplifies edge case handling.

### Helper Functions
Consider these for clarity:
1. **`remove(node)`**: Remove node from list
2. **`insert(node)`**: Insert node at tail (most recent)

### Strategy
- **For `get()`**: Remove node, then insert at tail (even if already there)
- **For `put()`**: Insert new/updated node, then remove LRU if capacity exceeded

### Single-threaded Implementation

```rust
use std::rc::Rc;
use std::cell::RefCell;

struct Node {
    key: i32,
    value: i32,
    prev: Option<Rc<RefCell<Node>>>,
    next: Option<Rc<RefCell<Node>>>,
}
```

**Why `Rc<RefCell<Node>>`?**

| Component | Purpose | What it provides |
|-----------|---------|------------------|
| `Rc` | Shared ownership | Multiple nodes reference same node |
| `RefCell` | Interior mutability | Modify through shared references |

#### Why Both Are Needed

**`Rc` solves shared ownership**:
```rust
// Without Rc - doesn't work!
struct Node {
    prev: Option<Box<Node>>,  // ❌ Box owns exclusively
}

// Node A can't be owned by both Node B's prev AND Node C's next
```

**`RefCell` solves mutability**:
```rust
// Without RefCell - doesn't work!
fn remove_node(node: Rc<Node>) {
    node.prev = None;  // ❌ Can't mutate through Rc
}

// With RefCell - works!
fn remove_node(node: Rc<RefCell<Node>>) {
    node.borrow_mut().prev = None;  // ✅ Runtime-checked mutation
}
```

### Multi-threaded Implementation

```rust
use std::sync::{Arc, Mutex};

struct Node {
    key: i32,
    value: i32,
    prev: Option<Arc<Mutex<Node>>>,
    next: Option<Arc<Mutex<Node>>>,
}
```

**Why `Arc<Mutex<Node>>`?**

| Component | Purpose | What it provides |
|-----------|---------|------------------|
| `Arc` | Thread-safe shared ownership | Multiple threads reference same node |
| `Mutex` | Thread-safe interior mutability | Only one thread accesses at a time |

#### Example Implementation

```rust
pub fn remove(&self, node: Arc<Mutex<Node>>) {
    let prev = node.lock().unwrap().prev.take().unwrap();
    let next = node.lock().unwrap().next.take().unwrap();
    
    prev.lock().unwrap().next = Some(Arc::clone(&next));
    next.lock().unwrap().prev = Some(Arc::clone(&prev));
}
```

### Choosing the Right Combination

```
Single-threaded                Multi-threaded
└─ Rc<RefCell<T>>             └─ Arc<Mutex<T>>
   ├─ Shared ownership            ├─ Thread-safe ownership
   └─ Runtime borrow checking     └─ Thread-safe mutation
```

---

## Closures

### What Are Closures?
Anonymous functions that can capture their environment.

### Basic Syntax

```rust
// Regular function
fn add_one(x: i32) -> i32 {
    x + 1
}

// Closure (types can be inferred)
let add_one = |x: i32| x + 1;
let add_one = |x| x + 1;  // Type inferred

// Using it
let result = add_one(5);  // 6
```

### Capturing Variables

```rust
let multiplier = 10; // This is in the enclosing environment

// Closure captures 'multiplier' from surrounding scope
let multiply = |x| x * multiplier; // 'multiplier' is captured!

println!("{}", multiply(5));  // 50
```

**Key difference from functions**: Closures can capture variables from their environment.

### Capture Modes

```rust
let s = String::from("hello");

// 1. Borrows immutably (Fn)
let print = || println!("{}", s);

// 2. Borrows mutably (FnMut)
let mut count = 0;
let mut increment = || count += 1;

// 3. Takes ownership (FnOnce)
let consume = || drop(s);  // Moves 's' into closure
```

### Closure Traits

| Trait | Behavior | Can call multiple times? |
|-------|----------|-------------------------|
| `Fn` | Borrows immutably | ✅ Yes |
| `FnMut` | Borrows mutably | ✅ Yes |
| `FnOnce` | Takes ownership | ❌ No (only once) |
### Closure Capture types

There are two types of closure captures in Rust:

#### Move capture

When a closure moves a variable from its enclosing environment into the closure, it is said to perform a "move capture." This means that the closure takes ownership of the variable and can modify it, but the original variable in the enclosing environment is no longer accessible.

#### Borrow capture 

When a closure borrows a variable from its enclosing environment, it is said to perform a "borrow capture." This means that the closure can access and modify the variable, but the original variable in the enclosing environment remains accessible.

### Common Use Cases

#### With Iterators

```rust
let numbers = vec![1, 2, 3, 4, 5];

// Filter even numbers
let evens: Vec<_> = numbers.iter()
    .filter(|&x| x % 2 == 0)
    .collect();

// Map to double each
let doubled: Vec<_> = numbers.iter()
    .map(|x| x * 2)
    .collect();
```

#### As Callbacks

```rust
fn do_twice<F>(f: F) 
where
    F: Fn(i32) -> i32
{
    println!("{}", f(5));
    println!("{}", f(10));
}

do_twice(|x| x * 2);
// Output: 10, 20
```

#### With Threads

```rust
use std::thread;

let data = vec![1, 2, 3];

// 'move' keyword transfers ownership
thread::spawn(move || {
    println!("{:?}", data);
});
```

---
## Attributes

Attributes in Rust are metadata annotations that provide instructions to the compiler or modify the behavior of code. They use the `#[]` or `#![]` syntax.


### Outer Attributes `#[]`
Apply to the item that follows them:

```rust
#[derive(Debug)]
struct Point { x: i32, y: i32 }

#[test]
fn my_test() {
    assert_eq!(2 + 2, 4);
}
```

### Inner Attributes `#![]`
Apply to the enclosing item (module, crate, or function):

```rust
// At the top of main.rs or lib.rs
#![allow(dead_code)]
#![warn(missing_docs)]

fn main() {
    #![allow(unused_variables)]
    let x = 5;
}
```

### Common Built-in Attributes

#### Conditional Compilation

Control which code gets compiled based on conditions:

```rust
#[cfg(target_os = "linux")]
fn linux_only() { }

#[cfg(target_os = "windows")]
fn windows_only() { }

#[cfg(test)]
mod tests { }

#[cfg(feature = "advanced")]
fn advanced_feature() { }
```

### Testing Attributes

```rust
#[test]
fn test_addition() {
    assert_eq!(2 + 2, 4);
}

#[test]
#[ignore]
fn expensive_test() {
    // Run with: cargo test -- --ignored
}

#[test]
#[should_panic]
fn test_panic() {
    panic!("This should panic");
}

#[test]
#[should_panic(expected = "divide by zero")]
fn test_specific_panic() {
    let _ = 1 / 0;
}
```

### Derive Attribute

Automatically implement traits:

```rust
#[deriv
### Common Derive Attributes

The `#[derive]` syntax in Rust is an attribute that automatically generates implementations of certain traits for your types.


### Display and Debug

```rust
use std::fmt;

struct Person {
    name: String,
    age: u32,
}

// Display - for user-facing output
impl fmt::Display for Person {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{} (age {})", self.name, self.age)
    }
}

// Debug - for debugging (can also use #[derive(Debug)])
impl fmt::Debug for Person {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        f.debug_struct("Person")
            .field("name", &self.name)
            .field("age", &self.age)
            .finish()
    }
}

let person = Person { name: "Alice".to_string(), age: 30 };
println!("{}", person);    // Uses Display
println!("{:?}", person);  // Uses Debug
```

#### `Display` Trait (`std::fmt::Display`)

**Purpose**: Format trait for user-facing output (`{}` in format strings)

**Benefits**:
- Implementing `Display` automatically implements `ToString`
- Enables `.to_string()` method
- Prefer implementing `Display` over implementing `ToString` directly

**Key Difference**:
- `Display`: User-facing output
- `Debug`: Developer-facing output (can be derived with `#[derive(Debug)]`)

### Clone and Copy

```rust
// Clone - explicit duplication
#[derive(Clone)]
struct Data {
    value: String,
}

let data1 = Data { value: "hello".to_string() };
let data2 = data1.clone();  // Explicit clone
println!("{}", data1.value);  // data1 still valid

// Copy - implicit duplication (only for stack-only types)
#[derive(Copy, Clone)]
struct Point {
    x: i32,
    y: i32,
}

let p1 = Point { x: 1, y: 2 };
let p2 = p1;  // Implicit copy
println!("{}", p1.x);  // p1 still valid (was copied, not moved)
```

**Rules for Copy:**
- Must be cheap to copy (stack-only data)
- Cannot contain heap data (no `String`, `Vec`, `Box`)
- Must also implement `Clone`

### PartialEq and Eq

```rust
#[derive(PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

let p1 = Point { x: 1, y: 2 };
let p2 = Point { x: 1, y: 2 };
println!("{}", p1 == p2);  // true

// Custom implementation
impl PartialEq for Person {
    fn eq(&self, other: &Self) -> bool {
        self.name == other.name && self.age == other.age
    }
}
```

### PartialOrd and Ord

```rust
#[derive(PartialEq, PartialOrd)]
struct Score(i32);

let s1 = Score(100);
let s2 = Score(200);
println!("{}", s1 < s2);  // true
```

### Derivable Traits

```rust
// Most common derivable traits
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
struct MyStruct {
    value: i32,
}
```

---

## Macros

### Two Types of Macros

#### 1. Declarative Macros (`macro_rules!`)
- Match patterns in Rust code
- Generate new code using those patterns

```rust
macro_rules! vec {
    ( $( $x:expr ),* ) => {
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}
```

#### 2. Procedural Macros

The syntax `#[...]` is used for attributes in Rust. Attributes are declarative tags that provide additional information to the compiler or alter the behavior of the code they are attached to. They can be placed on various items like functions, structs, enums, modules, and crates.
Built-in Attributes vs. Macro Attributes:

Built-in attributes: Many attributes are built into the Rust language, such as `#[allow(unused_variables)]`,` #[test]`, or `#[cfg(target_os = "linux")]`. These are not macros themselves but are recognized and handled directly by the compiler.
Macro attributes: Rust also supports procedural macros, and one type of procedural macro is an attribute-like macro. These macros define custom attributes that can be used on any item. When the compiler encounters an attribute-like macro, it executes the macro's code at compile time, and that code then generates or transforms other Rust code.

- Generate code at compile time through the syntax tree
- Defined within their own crates
- Invoked through custom attributes

```rust
#[derive(Debug)]  // Procedural macro
struct MyStruct;
```

---

## Database Programming

### Popular Rust Database Libraries

#### Diesel
- Popular ORM (Object-Relational Mapping)
- Type-safe, composable query builder
- Supports: PostgreSQL, MySQL, SQLite

#### Postgres
- Native Rust library for PostgreSQL
- Safe, ergonomic API

#### SQLx
- Unified API for multiple databases
- Supports: PostgreSQL, MySQL, SQLite
- Both sync and async operations
- Type-safe query builder

### Why Rust for Databases?
- High performance
- Memory safety
- Type safety prevents SQL injection
- Zero-cost abstractions

---

## Unsafe Code

### Why Unsafe Code Exists
Rust is designed for safety, but sometimes you need low-level memory operations that the compiler can't verify as safe.

### Managing Unsafe Code

#### 1. Raw Pointers
- Similar to C pointers
- Allow pointer arithmetic and casting

```rust
let x = 5;
let raw = &x as *const i32;

unsafe {
    println!("{}", *raw);
}
```

#### 2. Unsafe Functions
- Marked with `unsafe` keyword
- Developer must manually ensure correct usage

```rust
unsafe fn dangerous() {
    // Low-level operations
}
```

#### 3. Unsafe Blocks
- Mark a block as unsafe
- Must be contained within a safe function
- Compiler ensures overall program behavior is safe

```rust
fn safe_function() {
    unsafe {
        // Unsafe operations here
    }
    // Safe operations continue
}
```

### Safety Guarantees
- Unsafe code is isolated in `unsafe` blocks
- Compiler verifies safe code around unsafe blocks
- Reduces surface area for bugs

---
**Be ready to:**
1. Write a simple HTTP server from scratch
2. Explain why blocking is bad
3. Debug a deadlock scenario
4. Design a rate limiter
5. Implement graceful shutdown
6. Handle errors in spawned tasks
7. Choose between `spawn` and `join!`
8. Explain `Send` and `Sync` bounds

**Questions to ask interviewer:**
1. What's your runtime configuration? (threads, limits)
2. How do you handle backpressure?
3. Do you use `tokio-console` for monitoring?
4. What's your approach to graceful shutdown?
5. How do you test async code?

## Interview Questions
### Explain the Rust ownership model and how it helps manage memory

Rust's ownership model is a set of compile-time rules that manages memory without needing a garbage collector. Every value in Rust has a single owner, and when that owner goes out of scope, the value is automatically cleaned up.

The ownership system has three fundamental rules:

1) Each value has exactly one owner - a variable that's responsible for that data
2) When the owner goes out of scope, the value is dropped - memory is freed automatically
3) Ownership can be transferred (moved) but not duplicated - unless the type implements Copy"

```rust
let s1 = String::from("hello");
let s2 = s1;  // ownership moves to s2
// s1 is no longer valid here
```

Rust also has borrowing, which lets you reference data without taking ownership. You can have either multiple immutable references or one mutable reference at a time - never both. This prevents data races at compile time.
"This model eliminates entire classes of bugs: **no dangling pointers**, **no double frees**, **no data races**. All of this is enforced at compile time with zero runtime overhead, giving you C++ level performance with memory safety guarantees.

## Testing

### The `rustc_test` Framework

**Built-in unit testing system**:
- Define tests with `#[test]` attribute
- Assertion macros: `assert_eq!`, `assert_ne!`
- Easy to write and maintain tests

```rust
#[test]
fn it_works() {
    assert_eq!(2 + 2, 4);
}

#[test]
fn it_fails() {
    panic!("This test will fail");
}
```

---

## Key Methods & Functions

### `collect()`
- Takes anything iterable
- Turns it into a relevant collection
- Type inference determines the collection type
- One of the most powerful methods in the standard library

```rust
let v: Vec<i32> = (0..10).collect();
let s: String = vec!['h', 'i'].into_iter().collect();
let map: HashMap<_, _> = vec![(1, 2), (3, 4)].into_iter().collect();
```

---

## Quick Reference: Common Patterns

### Variable Naming Conventions
Use concise names in closures and iterators:
- `i`, `j` for indices
- `e` for event
- `el` for element
- `x`, `y` for generic values

### Common Iteration Patterns

```rust
let v = vec![1, 2, 3];

// Immutable iteration
for item in &v {
    println!("{}", item);
}

// Mutable iteration
for item in &mut v {
    *item += 1;
}

// Consuming iteration
for item in v {
    // v is moved, can't use v after this
}
```

### Type Annotations with `collect()`

```rust
// Explicit type
let v: Vec<i32> = iter.collect();

// Turbofish syntax
let v = iter.collect::<Vec<i32>>();

// Underscore for partial inference
let v: Vec<_> = iter.collect();
```

---

## Best Practices

### 1. Prefer Generics for Performance
```rust
// Fast - compiler optimizes
fn process<T: Display>(item: T) {
    println!("{}", item);
}
```

### 2. Use Trait Objects for Flexibility
```rust
// Flexible - can store different types
let items: Vec<Box<dyn Display>> = vec![
    Box::new(42),
    Box::new("hello"),
];
```

### 3. Use Where Clauses for Readability
```rust
// More readable
fn complex<T, U>(t: T, u: U)
where
    T: Display + Clone,
    U: Debug + Clone,
{
    // ...
}
```

### 4. Implement Common Traits
```rust
// Makes your types more usable
#[derive(Debug, Clone, PartialEq)]
struct MyType {
    // ...
}
```

### 5. Check Object Safety Early
```rust
// Try to create a trait object to check
let obj: Box<dyn MyTrait> = ...;
// Compiler will tell you if it's not object-safe
```

---

## Memory Safety Guarantees Summary

### What Rust Prevents
✅ Null pointer dereferences  
✅ Data races  
✅ Buffer overflows  
✅ Use-after-free  
✅ Double-free  
✅ Iterator invalidation  

### How Rust Prevents Them
- **Ownership system**: One owner, automatic cleanup
- **Borrowing rules**: Either multiple readers OR one writer
- **Lifetime tracking**: References never outlive their data
- **Compile-time checks**: Catch errors before running
- **No garbage collector needed**: Zero runtime overhead

---

## Additional Resources
- [Claude](https://claude.ai/public/artifacts/609f945f-8704-404d-8713-ee023e643a27)
- [The Rust Book](https://doc.rust-lang.org/book/)
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/)
- [Slices Documentation](https://doc.rust-lang.org/book/ch04-03-slices.html)
- [Struct Ownership](https://doc.rust-lang.org/book/ch05-01-defining-structs.html#ownership-of-struct-data)
- [Threading Example with Arc/Mutex](https://claude.ai/public/artifacts/5eeb9009-5129-4f95-b0be-5fc8d209fc3c)

---

*This guide contains all your original content plus comprehensive generics and traits coverage, organized for easy review and understanding. Use the table of contents to jump to specific sections!*

