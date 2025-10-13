# Rust Smart Pointers Guide: Rc, RefCell, and Their Combinations

## Table of Contents
- [Introduction](#introduction)
- [Rc (Reference Counted)](#rc-reference-counted)
- [RefCell (Reference Cell)](#refcell-reference-cell)
- [Combining Rc and RefCell](#combining-rc-and-refcell)
- [Why Rc<Mutex> Doesn't Work](#why-rcmutex-doesnt-work)
- [Smart Pointer Decision Guide](#smart-pointer-decision-guide)
- [Graph Implementation Examples](#graph-implementation-examples)

---

## Introduction

Rust's ownership system is strict by design, but sometimes you need more flexibility. Smart pointers like `Rc`, `RefCell`, `Arc`, and `Mutex` help you work around these restrictions safely.

**Key Question**: Which smart pointer should you use for graphs?

**Quick Answer**:
- Single-threaded graphs: `Rc<RefCell<T>>`
- Multi-threaded graphs: `Arc<Mutex<T>>` or `Arc<RwLock<T>>`

---

## Rc (Reference Counted)

### What It Does
Enables **multiple ownership** of the same data through reference counting.

### Key Characteristics
- ✅ Multiple owners can share read access
- ✅ Single-threaded only
- ✅ Automatic cleanup when count reaches zero
- ❌ Immutable by default
- ❌ Small runtime overhead for counting

### How It Works
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

### Common Use Cases

#### 1. Graphs with Shared Nodes
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

#### 2. Shared Configuration
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

#### 3. Caching
```rust
use std::rc::Rc;
use std::collections::HashMap;

let mut cache: HashMap<String, Rc<ExpensiveData>> = HashMap::new();

let data = Rc::new(compute_expensive_data());
cache.insert("key1".to_string(), Rc::clone(&data));
cache.insert("key2".to_string(), Rc::clone(&data));
// Same data cached under multiple keys
```

### Important Notes
- `Rc::clone()` is cheap - it only increments a counter, doesn't copy data
- Watch out for reference cycles - use `Weak` to break them
- Cannot be sent across threads (not `Send`)

---

## RefCell (Reference Cell)

### What It Does
Enables **interior mutability** - allows mutation through shared references by moving borrow checking to runtime.

### Key Characteristics
- ✅ Mutate data through shared reference
- ✅ Enforces borrowing rules at runtime
- ✅ Single-threaded only
- ❌ Panics if borrowing rules violated
- ❌ Slight runtime overhead

### Borrowing Rules (Enforced at Runtime)
- Many immutable borrows OR one mutable borrow
- Cannot have both at the same time
- Violating rules = **runtime panic**

### How It Works
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

### What Causes Panics
```rust
use std::cell::RefCell;

let data = RefCell::new(5);

let read = data.borrow();
let write = data.borrow_mut(); // 💥 PANIC! Already borrowed immutably

// Or:
let write1 = data.borrow_mut();
let write2 = data.borrow_mut(); // 💥 PANIC! Already borrowed mutably
```

### Common Use Cases

#### 1. Mocking and Testing
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

#### 2. Graphs that Need Mutation
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

#### 3. Lazy Initialization
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

#### 4. Observer Pattern
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

## Combining Rc and RefCell

### Why Combine Them?
- `Rc`: Multiple ownership
- `RefCell`: Interior mutability
- Together: Multiple owners can all mutate shared data

### The Power Combo
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

### Real-World Example: Graph Implementation
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

## Why Rc<Mutex> Doesn't Work

### The Problem
`Rc<Mutex<T>>` will compile but is a **design mistake** because you're mixing incompatible threading models.

### Analysis

#### Rc: Single-Threaded
- Not thread-safe
- Fast (no atomic operations)
- **Cannot be sent between threads**
- Not `Send` or `Sync`

#### Mutex: Multi-Threaded
- Provides thread-safe exclusive access
- Overhead for synchronization
- Designed for concurrent access

### Why It Doesn't Make Sense

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

### Issues with Rc<Mutex>

1. **Can't actually use across threads** - The whole point of `Mutex` is lost
2. **Unnecessary overhead** - Paying for thread-safety you can't use
3. **Wrong tool for the job** - `RefCell` is better for single-threaded mutation

### The Right Combinations

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

### Comparison Table

| Combination | Compiles? | Thread-Safe? | Should Use? | Reason |
|-------------|-----------|--------------|-------------|---------|
| `Rc<RefCell<T>>` | ✅ | ❌ | ✅ | Perfect for single-threaded |
| `Arc<Mutex<T>>` | ✅ | ✅ | ✅ | Perfect for multi-threaded |
| `Arc<RwLock<T>>` | ✅ | ✅ | ✅ | Multi-threaded, read-heavy |
| `Rc<Mutex<T>>` | ✅ | ❌ | ❌ | Can't use across threads, unnecessary overhead |
| `Arc<RefCell<T>>` | ✅ | ❌ | ❌ | `RefCell` not thread-safe, causes data races |

### Analogy
`Rc<Mutex<T>>` is like putting a high-security bank vault lock on a bedroom door in a studio apartment:
- You're living alone (`Rc` = single-threaded)
- You installed an expensive multi-person security system (`Mutex`)
- But you can never have multiple people anyway!
- A simple lock (`RefCell`) would work fine and cost less

---

## Smart Pointer Decision Guide

### Decision Tree

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

### Quick Reference

| Need | Single-Threaded | Multi-Threaded |
|------|----------------|----------------|
| Shared ownership (immutable) | `Rc<T>` | `Arc<T>` |
| Shared ownership + mutation | `Rc<RefCell<T>>` | `Arc<Mutex<T>>` |
| Shared ownership + read-heavy | `Rc<RefCell<T>>` | `Arc<RwLock<T>>` |
| Tree (no cycles) | `Box<T>` | `Box<T>` |

### Performance Characteristics

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

## Best Practices

### 1. Prefer Simple Ownership
Use smart pointers only when necessary. Start with simple ownership and upgrade when needed.

### 2. Avoid Reference Cycles
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

## Common Pitfalls

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

## Conclusion

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

**Remember:** These tools exist because some problems (like graphs with cycles) cannot be expressed with simple ownership. Use them wisely, and always prefer the simplest solution that works.
