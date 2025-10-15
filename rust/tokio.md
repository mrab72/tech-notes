# Tokio Runtime - Preparation

## Quick Overview

**Tokio** is an asynchronous runtime for Rust that provides:
- Event-driven, non-blocking I/O
- Task scheduling
- Timers and utilities
- Multi-threaded or single-threaded execution

**Why Kraken cares**: High-performance trading systems need to handle thousands of concurrent connections efficiently. Tokio enables this without blocking threads.

---

## Core Concepts You MUST Know

### 1. Async/Await Basics

**What is async/await?**
- `async fn` returns a `Future` (lazy, doesn't run until polled)
- `await` polls the future until completion
- Futures are state machines compiled by Rust

```rust
// This doesn't run immediately!
async fn fetch_price(symbol: &str) -> f64 {
    // Simulating API call
    tokio::time::sleep(Duration::from_millis(100)).await;
    42.0
}

#[tokio::main]
async fn main() {
    let price = fetch_price("BTC").await; // Now it runs
    println!("Price: {}", price);
}
```

**Key Point**: `async` functions don't do anything until awaited or spawned!

---

### 2. The Tokio Runtime

**What is the runtime?**
The engine that:
- Polls futures to completion
- Schedules tasks across threads
- Manages I/O resources
- Provides timers and utilities

**Three ways to create a runtime:**

```rust
// 1. Macro (most common)
#[tokio::main]
async fn main() {
    // Multi-threaded by default
}

// 2. Builder pattern (more control)
#[tokio::main(flavor = "current_thread")]
async fn main() {
    // Single-threaded
}

// 3. Manual (full control)
fn main() {
    let runtime = tokio::runtime::Runtime::new().unwrap();
    runtime.block_on(async {
        // Your async code
    });
}
```

**Runtime Flavors:**

| Flavor | Threads | Use Case |
|--------|---------|----------|
| `multi_thread` | N (CPU cores) | Production, high concurrency |
| `current_thread` | 1 | Testing, simple apps, lightweight |

---

### 3. Task Spawning

**Why spawn tasks?**
Run multiple operations concurrently without blocking.

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    // Spawn returns a JoinHandle
    let handle1 = tokio::spawn(async {
        println!("Task 1");
        42
    });
    
    let handle2 = tokio::spawn(async {
        println!("Task 2");
        100
    });
    
    // Wait for results
    let result1 = handle1.await.unwrap();
    let result2 = handle2.await.unwrap();
    
    println!("Results: {}, {}", result1, result2);
}
```

**Critical Interview Points:**
- Spawned tasks run **concurrently**, not necessarily in parallel
- Tasks are **lightweight** (not OS threads)
- `JoinHandle` is like a future that resolves to the task's result
- If you drop a `JoinHandle`, the task continues running (detached)

---

### 4. Work Stealing Scheduler

**How Tokio schedules tasks:**

```
Thread 1: [Task A] [Task B] [Task C]
Thread 2: [Task D] [Task E]
Thread 3: [Task F] [Task G] [Task H]

If Thread 2 finishes early, it "steals" tasks from Thread 1 or 3
```

**Key characteristics:**
- Multi-threaded runtime uses work-stealing
- Balances load automatically
- Tasks can migrate between threads
- **Important**: Don't use thread-local storage!

**Interview Question**: "Why work-stealing?"
**Answer**: Maximizes CPU utilization. Idle threads don't stay idle—they help other threads.

---

### 5. Blocking vs Non-Blocking

**The Golden Rule**: NEVER block the Tokio runtime!

```rust
// ❌ BAD: Blocking the runtime
#[tokio::main]
async fn main() {
    std::thread::sleep(Duration::from_secs(1)); // BLOCKS EVERYTHING!
}

// ✅ GOOD: Non-blocking sleep
#[tokio::main]
async fn main() {
    tokio::time::sleep(Duration::from_secs(1)).await; // Yields to other tasks
}
```

**When you MUST do blocking work:**

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    // Spawn blocking work on dedicated thread pool
    let result = task::spawn_blocking(|| {
        // CPU-intensive or blocking I/O
        std::thread::sleep(Duration::from_secs(1));
        expensive_computation()
    }).await.unwrap();
}
```

**Interview Question**: "What happens if you block in a Tokio task?"
**Answer**: You block the entire thread, preventing other tasks from running. In worst case, you can deadlock the runtime.

---

### 6. Channels for Communication

Tokio provides async-aware channels (different from `std::sync::mpsc`).

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(32); // Buffer size 32
    
    // Spawn producer
    tokio::spawn(async move {
        for i in 0..10 {
            tx.send(i).await.unwrap();
        }
    });
    
    // Consumer
    while let Some(value) = rx.recv().await {
        println!("Received: {}", value);
    }
}
```

**Channel Types:**

| Type | Use Case |
|------|----------|
| `mpsc` | Multiple producers, single consumer |
| `oneshot` | Send one value, then close |
| `broadcast` | Multiple consumers, all receive same message |
| `watch` | Latest value, old values dropped |

**For Kraken**: You'd use `mpsc` for order processing pipelines, `broadcast` for market data distribution.

---

### 7. Select! Macro

Wait on multiple futures, act on whichever completes first.

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    tokio::select! {
        _ = sleep(Duration::from_secs(1)) => {
            println!("Timeout!");
        }
        result = fetch_data() => {
            println!("Got data: {:?}", result);
        }
        _ = tokio::signal::ctrl_c() => {
            println!("Shutting down...");
        }
    }
}

async fn fetch_data() -> String {
    sleep(Duration::from_millis(500)).await;
    "data".to_string()
}
```

**Use cases for Kraken:**
- Timeout on API requests
- Graceful shutdown signals
- Racing multiple data sources

---

### 8. Join and Try Join

Run multiple futures concurrently and wait for all to complete.

```rust
use tokio::try_join;

async fn fetch_btc_price() -> Result<f64, String> {
    Ok(50000.0)
}

async fn fetch_eth_price() -> Result<f64, String> {
    Ok(3000.0)
}

#[tokio::main]
async fn main() {
    // All must succeed
    match try_join!(fetch_btc_price(), fetch_eth_price()) {
        Ok((btc, eth)) => println!("BTC: {}, ETH: {}", btc, eth),
        Err(e) => println!("Error: {}", e),
    }
    
    // Or use join! if no Result
    let (btc, eth) = tokio::join!(
        fetch_btc_price(),
        fetch_eth_price()
    );
}
```

**Difference from spawning:**
- `join!` runs on current task
- `spawn` creates new task
- `join!` for CPU-bound, `spawn` for true concurrency

---

## Real-World Kraken Interview Scenarios

### Scenario 1: WebSocket Server

```rust
use tokio::net::{TcpListener, TcpStream};
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:8080").await.unwrap();
    
    loop {
        let (socket, addr) = listener.accept().await.unwrap();
        
        // Spawn a task per connection
        tokio::spawn(async move {
            handle_client(socket).await;
        });
    }
}

async fn handle_client(mut socket: TcpStream) {
    let mut buffer = [0; 1024];
    
    loop {
        match socket.read(&mut buffer).await {
            Ok(0) => break, // Connection closed
            Ok(n) => {
                // Echo back
                socket.write_all(&buffer[..n]).await.unwrap();
            }
            Err(e) => {
                eprintln!("Error: {}", e);
                break;
            }
        }
    }
}
```

**Interview points:**
- One task per connection (lightweight!)
- Non-blocking I/O
- Can handle 10,000+ connections

---

### Scenario 2: Rate Limiter

```rust
use tokio::time::{sleep, Duration, Instant};
use std::sync::Arc;
use tokio::sync::Semaphore;

struct RateLimiter {
    semaphore: Arc<Semaphore>,
    rate: u32,
    interval: Duration,
}

impl RateLimiter {
    fn new(rate: u32, interval: Duration) -> Self {
        RateLimiter {
            semaphore: Arc::new(Semaphore::new(rate as usize)),
            rate,
            interval,
        }
    }
    
    async fn acquire(&self) {
        let _permit = self.semaphore.acquire().await.unwrap();
        
        tokio::spawn({
            let sem = self.semaphore.clone();
            let interval = self.interval;
            async move {
                sleep(interval).await;
                // Permit automatically released when dropped
            }
        });
    }
}

#[tokio::main]
async fn main() {
    let limiter = RateLimiter::new(10, Duration::from_secs(1));
    
    for i in 0..100 {
        limiter.acquire().await;
        println!("Request {}", i);
        // Make API call
    }
}
```

**For Kraken**: Rate limit API requests to exchanges.

---

### Scenario 3: Timeout Pattern

```rust
use tokio::time::{timeout, Duration};

async fn fetch_with_timeout(url: &str) -> Result<String, Box<dyn std::error::Error>> {
    let response = timeout(
        Duration::from_secs(5),
        reqwest::get(url)
    ).await??; // Double ? for timeout error and request error
    
    Ok(response.text().await?)
}

#[tokio::main]
async fn main() {
    match fetch_with_timeout("https://api.kraken.com/prices").await {
        Ok(data) => println!("Success: {}", data),
        Err(e) => eprintln!("Failed: {}", e),
    }
}
```

**Critical for trading systems**: Always timeout external calls!

---

### Scenario 4: Graceful Shutdown

```rust
use tokio::signal;
use tokio::sync::broadcast;

#[tokio::main]
async fn main() {
    let (shutdown_tx, _) = broadcast::channel(1);
    
    // Spawn workers
    for i in 0..3 {
        let mut shutdown_rx = shutdown_tx.subscribe();
        tokio::spawn(async move {
            loop {
                tokio::select! {
                    _ = shutdown_rx.recv() => {
                        println!("Worker {} shutting down", i);
                        break;
                    }
                    _ = tokio::time::sleep(Duration::from_secs(1)) => {
                        println!("Worker {} working...", i);
                    }
                }
            }
        });
    }
    
    // Wait for Ctrl+C
    signal::ctrl_c().await.unwrap();
    println!("Shutdown signal received!");
    
    // Broadcast shutdown
    let _ = shutdown_tx.send(());
    
    // Give workers time to cleanup
    tokio::time::sleep(Duration::from_secs(1)).await;
    println!("Shutdown complete");
}
```

**For Kraken**: Critical for safely shutting down trading systems without losing orders.

---

## Performance & Best Practices

### 1. CPU-Bound vs I/O-Bound

**I/O-Bound** (network, disk): Use Tokio
```rust
// ✅ Perfect for Tokio
async fn fetch_market_data() {
    let response = reqwest::get("https://api.example.com").await;
}
```

**CPU-Bound** (computation): Use `spawn_blocking` or Rayon
```rust
// ❌ Bad: Blocks Tokio runtime
async fn calculate_risk() {
    let result = heavy_computation(); // Blocks!
}

// ✅ Good: Offload to thread pool
async fn calculate_risk() {
    let result = tokio::task::spawn_blocking(|| {
        heavy_computation()
    }).await.unwrap();
}
```

---

### 2. Task Size

**Too small**: Overhead of scheduling > work done
```rust
// ❌ Wasteful
for i in 0..1000 {
    tokio::spawn(async move {
        println!("{}", i); // Tiny work
    });
}
```

**Too large**: Blocks other tasks
```rust
// ❌ One task hogs runtime
tokio::spawn(async {
    for i in 0..1000000 {
        // Long-running loop without yield points
        heavy_work();
    }
});
```

**Just right**: Balance work with yield points
```rust
// ✅ Yields periodically
tokio::spawn(async {
    for i in 0..1000000 {
        heavy_work();
        if i % 1000 == 0 {
            tokio::task::yield_now().await; // Let other tasks run
        }
    }
});
```

---

### 3. Avoid Shared Mutable State

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

// Works, but lock contention can be a bottleneck
let counter = Arc::new(Mutex::new(0));

for _ in 0..100 {
    let counter = counter.clone();
    tokio::spawn(async move {
        let mut count = counter.lock().await;
        *count += 1;
    });
}

// Better: Use channels for communication
let (tx, mut rx) = tokio::sync::mpsc::channel(100);

for _ in 0..100 {
    let tx = tx.clone();
    tokio::spawn(async move {
        tx.send(1).await.unwrap();
    });
}

// Aggregator task
tokio::spawn(async move {
    let mut total = 0;
    while let Some(val) = rx.recv().await {
        total += val;
    }
});
```

**Principle**: "Share by communicating, don't communicate by sharing"

---

### 4. The `Send` Bound

**Key concept**: Tasks must be `Send` because they can move between threads.

```rust
use std::rc::Rc; // Not Send!

#[tokio::main]
async fn main() {
    let data = Rc::new(42);
    
    // ❌ Compile error: Rc is not Send
    tokio::spawn(async move {
        println!("{}", data);
    });
}

// ✅ Use Arc instead
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let data = Arc::new(42);
    
    tokio::spawn(async move {
        println!("{}", data);
    });
}
```

**Interview Question**: "Why does this fail to compile?"
**Answer**: `tokio::spawn` requires `Send` because tasks can move between threads in the work-stealing scheduler. `Rc` is not thread-safe.

---

## Common Interview Questions

### Q1: "What's the difference between async/await in Rust vs JavaScript?"

**Answer:**
- **Rust**: Futures are lazy (zero-cost until polled), no built-in runtime
- **JavaScript**: Promises are eager (start immediately), runtime built into V8
- **Rust**: Compile-time state machines, no garbage collection
- **JavaScript**: Runtime overhead, GC pauses

### Q2: "How many threads does Tokio use by default?"

**Answer:** 
- Multi-threaded runtime: Number of CPU cores
- Can configure with `Runtime::builder().worker_threads(N)`
- Current-thread runtime: 1 thread

### Q3: "What happens if you `.await` inside a `spawn_blocking`?"

**Answer:**
```rust
// ❌ This works but is wasteful
tokio::task::spawn_blocking(|| {
    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        // Async code in blocking context... why?
    });
});
```
You're wasting a blocking thread to run async code. Just use `tokio::spawn` instead!

### Q4: "Explain the difference between `Mutex` from `std::sync` vs `tokio::sync`"

**Answer:**

| Feature | `std::sync::Mutex` | `tokio::sync::Mutex` |
|---------|-------------------|---------------------|
| Blocking | Yes (OS-level) | No (async-aware) |
| Use in async | ❌ Blocks runtime | ✅ Yields to scheduler |
| Performance | Faster for short locks | Slower due to async overhead |
| Use case | `spawn_blocking` | Async tasks |

```rust
// ❌ Bad: Blocks runtime
use std::sync::Mutex;

tokio::spawn(async {
    let data = Mutex::new(0);
    let _guard = data.lock().unwrap(); // BLOCKS!
});

// ✅ Good: Async-aware
use tokio::sync::Mutex;

tokio::spawn(async {
    let data = Mutex::new(0);
    let _guard = data.lock().await; // Yields!
});
```

### Q5: "How would you handle 100,000 concurrent connections?"

**Answer:**
```rust
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("0.0.0.0:8080").await.unwrap();
    
    // Set ulimit properly: ulimit -n 100000
    loop {
        let (socket, _) = listener.accept().await.unwrap();
        tokio::spawn(handle_connection(socket));
    }
}
```

**Key points:**
- Each connection = 1 task (~2KB overhead)
- Tasks are NOT threads (can have millions)
- Tune OS limits (`ulimit`)
- Use connection pooling if connecting outbound
- Monitor memory and file descriptors

---

## Kraken-Specific Scenarios

### Order Matching Engine

```rust
use tokio::sync::mpsc;

struct Order {
    id: u64,
    price: f64,
    quantity: f64,
}

struct OrderBook {
    buy_orders: Vec<Order>,
    sell_orders: Vec<Order>,
}

#[tokio::main]
async fn main() {
    let (order_tx, mut order_rx) = mpsc::channel::<Order>(1000);
    let (match_tx, mut match_rx) = mpsc::channel::<(Order, Order)>(100);
    
    // Order matching task
    tokio::spawn(async move {
        let mut order_book = OrderBook {
            buy_orders: vec![],
            sell_orders: vec![],
        };
        
        while let Some(order) = order_rx.recv().await {
            // Match orders (simplified)
            if let Some(matched) = find_match(&mut order_book, &order) {
                match_tx.send((order, matched)).await.unwrap();
            }
        }
    });
    
    // Settlement task
    tokio::spawn(async move {
        while let Some((buy, sell)) = match_rx.recv().await {
            println!("Matched: Buy {} @ {} with Sell {} @ {}", 
                     buy.id, buy.price, sell.id, sell.price);
            // Execute trade
        }
    });
    
    // Simulate incoming orders
    for i in 0..100 {
        order_tx.send(Order {
            id: i,
            price: 50000.0 + (i as f64),
            quantity: 0.1,
        }).await.unwrap();
    }
    
    tokio::time::sleep(Duration::from_secs(1)).await;
}

fn find_match(book: &mut OrderBook, order: &Order) -> Option<Order> {
    // Simplified matching logic
    None
}
```

---

### WebSocket Market Data Feed

```rust
use tokio_tungstenite::{connect_async, tungstenite::Message};
use futures_util::{StreamExt, SinkExt};

#[tokio::main]
async fn main() {
    let (ws_stream, _) = connect_async("wss://ws.kraken.com/")
        .await
        .expect("Failed to connect");
    
    let (mut write, mut read) = ws_stream.split();
    
    // Subscribe to BTC/USD ticker
    let subscribe_msg = r#"{"event":"subscribe","pair":["BTC/USD"],"subscription":{"name":"ticker"}}"#;
    write.send(Message::Text(subscribe_msg.to_string())).await.unwrap();
    
    // Process incoming messages
    while let Some(msg) = read.next().await {
        match msg {
            Ok(Message::Text(text)) => {
                println!("Received: {}", text);
                // Parse and process market data
            }
            Ok(Message::Close(_)) => break,
            Err(e) => {
                eprintln!("Error: {}", e);
                break;
            }
            _ => {}
        }
    }
}
```

---

### High-Performance HTTP Server

```rust
use axum::{
    routing::get,
    Router,
    Json,
};
use serde::Serialize;
use std::net::SocketAddr;

#[derive(Serialize)]
struct Price {
    symbol: String,
    price: f64,
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/price/:symbol", get(get_price));
    
    let addr = SocketAddr::from(([0, 0, 0, 0], 3000));
    
    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .unwrap();
}

async fn get_price(
    axum::extract::Path(symbol): axum::extract::Path<String>
) -> Json<Price> {
    // Fetch from cache or database
    Json(Price {
        symbol,
        price: 50000.0,
    })
}
```

---

## Debugging & Monitoring

### 1. Enable Tokio Console

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full", "tracing"] }
console-subscriber = "0.1"
```

```rust
#[tokio::main]
async fn main() {
    console_subscriber::init();
    // Your code
}
```

Run with: `tokio-console`

### 2. Tracing

```rust
use tracing::{info, warn, error};

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt::init();
    
    info!("Starting server");
    
    match risky_operation().await {
        Ok(_) => info!("Success"),
        Err(e) => error!("Failed: {}", e),
    }
}
```

### 3. Metrics

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;

struct Metrics {
    requests: AtomicU64,
    errors: AtomicU64,
}

let metrics = Arc::new(Metrics {
    requests: AtomicU64::new(0),
    errors: AtomicU64::new(0),
});

// In your handler
metrics.requests.fetch_add(1, Ordering::Relaxed);
```

---

## Rapid-Fire Review

**Must know cold:**
1. `async`/`await` - lazy futures, state machines
2. `tokio::spawn` - concurrent tasks, returns `JoinHandle`
3. `select!` - wait on multiple futures
4. `join!`/`try_join!` - wait for all futures
5. Never block the runtime - use `spawn_blocking`
6. Channels: `mpsc`, `oneshot`, `broadcast`, `watch`
7. `Arc` for shared data across tasks
8. `tokio::sync::Mutex` for async, `std::sync::Mutex` for blocking
9. Work-stealing scheduler - tasks migrate between threads
10. Tasks must be `Send` - use `Arc` not `Rc`

**Common mistakes:**
1. Blocking in async context
2. Using `std::sync::Mutex` in async code
3. Not handling errors in spawned tasks
4. Forgetting to `.await`
5. Creating too many small tasks
6. Not using channels for inter-task communication

**Performance tips:**
1. Batch small operations
2. Use `spawn_blocking` for CPU work
3. Prefer channels over shared `Mutex`
4. Use `tokio::time::sleep` not `std::thread::sleep`
5. Profile with `tokio-console`

---

## Final Interview Prep

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

---

## Quick Code Snippets to Memorize

### Basic Server Template
```rust
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("0.0.0.0:8080").await.unwrap();
    loop {
        let (socket, _) = listener.accept().await.unwrap();
        tokio::spawn(async move {
            handle_client(socket).await;
        });
    }
}
```

### Error Handling Template
```rust
async fn fallible_operation() -> Result<String, Box<dyn std::error::Error>> {
    let data = tokio::fs::read_to_string("file.txt").await?;
    Ok(data)
}

#[tokio::main]
async fn main() {
    match fallible_operation().await {
        Ok(data) => println!("Success: {}", data),
        Err(e) => eprintln!("Error: {}", e),
    }
}
```

### Timeout Template
```rust
use tokio::time::{timeout, Duration};

async fn with_timeout() -> Result<String, Box<dyn std::error::Error>> {
    Ok(timeout(Duration::from_secs(5), fetch_data()).await??)
}
```

### Graceful Shutdown Template
```rust
tokio::select! {
    _ = signal::ctrl_c() => {
        println!("Shutting down...");
    }
    result = do_work() => {
        println!("Work completed: {:?}", result);
    }
}
```

---

## Good Luck!

**Remember:**
- Tokio is about **concurrency**, not parallelism (though it enables both)
- Tasks are **cheap** - spawn liberally for I/O
- **Never block** the runtime - use `spawn_blocking`
- **Channels** are your friend for task communication
- **Profile** with `tokio-console` when in doubt

You've got this! 🚀
