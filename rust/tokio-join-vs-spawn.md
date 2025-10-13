# Tokio: `join!` vs `spawn` - Simple Guide

## Quick Summary

- **`tokio::join!`** - Run multiple things at once, wait for all results together
- **`tokio::spawn`** - Launch independent background tasks

---

## Example 1: Using `tokio::join!`

### The Code

```rust
use tokio::time::{sleep, Duration};

async fn make_coffee() -> String {
    println!("Starting coffee...");
    sleep(Duration::from_secs(2)).await;
    println!("Coffee done!");
    "☕ Coffee".to_string()
}

async fn make_toast() -> String {
    println!("Starting toast...");
    sleep(Duration::from_secs(1)).await;
    println!("Toast done!");
    "🍞 Toast".to_string()
}

#[tokio::main]
async fn main() {
    let (coffee, toast) = tokio::join!(
        make_coffee(),
        make_toast()
    );
    
    println!("Breakfast ready: {} and {}", coffee, toast);
}
```

### What Happens Step-by-Step

1. Both `make_coffee()` and `make_toast()` start **at the same time**
2. Toast finishes after 1 second (prints "Toast done!")
3. Coffee finishes after 2 seconds (prints "Coffee done!")
4. **Total time: 2 seconds** (not 3!)
5. You get both results back together

### Output

```
Starting coffee...
Starting toast...
Toast done!
Coffee done!
Breakfast ready: ☕ Coffee and 🍞 Toast
```

---

## Example 2: Using `tokio::spawn`

### The Code

```rust
use tokio::time::{sleep, Duration};

async fn download_file(name: &str) {
    println!("Downloading {}...", name);
    sleep(Duration::from_secs(2)).await;
    println!("{} downloaded!", name);
}

#[tokio::main]
async fn main() {
    // Spawn background tasks
    let handle1 = tokio::spawn(async {
        download_file("video.mp4").await;
    });
    
    let handle2 = tokio::spawn(async {
        download_file("music.mp3").await;
    });
    
    println!("Downloads started, doing other work...");
    sleep(Duration::from_secs(1)).await;
    println!("Still doing stuff...");
    
    // Wait for downloads to finish
    handle1.await.unwrap();
    handle2.await.unwrap();
    
    println!("All downloads complete!");
}
```

### What Happens Step-by-Step

1. `handle1` spawns a **new independent task** for video download
2. `handle2` spawns **another independent task** for music download
3. Main function **continues immediately** - doesn't wait
4. Prints "Downloads started..."
5. Does other work for 1 second
6. Finally waits for both handles at the end

### Output

```
Downloads started, doing other work...
Downloading video.mp4...
Downloading music.mp3...
Still doing stuff...
video.mp4 downloaded!
music.mp3 downloaded!
All downloads complete!
```

---

## Visual Comparison

### With `join!` - Everything happens together in one task

```
[Main Task]
  ├─ coffee (2s) ──┐
  └─ toast (1s) ───┤
                   ├─> Both done, continue
```

### With `spawn` - Separate independent tasks

```
[Main Task] ────────────────> continues immediately
   │
   ├─> [Task 1] coffee (2s)
   └─> [Task 2] toast (1s)
```

---

## Detailed Differences

| Feature | `tokio::join!` | `tokio::spawn` |
|---------|----------------|----------------|
| **Creates new task?** | No - runs on same task | Yes - spawns independent task |
| **When does it return?** | After all futures complete | Immediately with `JoinHandle` |
| **Result handling** | Returns tuple of all results | Each handle gives one result |
| **Cancellation** | All cancelled together | Each task independent |
| **Requirements** | Futures can be `!Send` | Requires `Send + 'static` |
| **Overhead** | Zero allocation | Small allocation per task |

---

## When To Use Which?

### Use `tokio::join!` when:

- ✅ You need all results before continuing
- ✅ Tasks are related and should complete together
- ✅ You want structured concurrency
- ✅ Futures don't need to be `Send`
- ✅ Running on the same task is fine

**Example:** Fetching multiple pieces of data for one API response

```rust
let (user, posts, comments) = tokio::join!(
    fetch_user(id),
    fetch_posts(id),
    fetch_comments(id)
);
// Use all three together
```

### Use `tokio::spawn` when:

- ✅ You want fire-and-forget behavior
- ✅ Task should run in the background
- ✅ You don't need the result immediately
- ✅ Task should continue even if parent moves on
- ✅ You want to distribute work across threads

**Example:** Background job processing

```rust
tokio::spawn(async move {
    process_video(video_id).await;
    send_notification(user_id).await;
});
// Continue immediately, don't wait for processing
```

---

## Common Patterns

### Pattern 1: Parallel API Calls with `join!`

```rust
async fn get_dashboard_data(user_id: u64) -> Dashboard {
    let (profile, analytics, notifications) = tokio::join!(
        api::get_profile(user_id),
        api::get_analytics(user_id),
        api::get_notifications(user_id)
    );
    
    Dashboard {
        profile,
        analytics,
        notifications
    }
}
```

### Pattern 2: Background Worker with `spawn`

```rust
fn start_cleanup_worker() -> JoinHandle<()> {
    tokio::spawn(async {
        loop {
            sleep(Duration::from_secs(3600)).await;
            cleanup_old_files().await;
        }
    })
}
```

### Pattern 3: Race Multiple Sources with `select!`

```rust
use tokio::select;

async fn fetch_with_fallback() -> String {
    select! {
        result = fetch_from_primary() => result,
        result = fetch_from_backup() => result,
    }
}
```

---

## Common Mistakes

### ❌ Mistake 1: Sequential instead of parallel

```rust
// SLOW - takes 3 seconds total
let coffee = make_coffee().await;
let toast = make_toast().await;
```

```rust
// FAST - takes 2 seconds total
let (coffee, toast) = tokio::join!(make_coffee(), make_toast());
```

### ❌ Mistake 2: Spawning when join! would work

```rust
// Unnecessary overhead
let h1 = tokio::spawn(fetch_user());
let h2 = tokio::spawn(fetch_posts());
let user = h1.await.unwrap();
let posts = h2.await.unwrap();
```

```rust
// Better - simpler and more efficient
let (user, posts) = tokio::join!(fetch_user(), fetch_posts());
```

### ❌ Mistake 3: Not awaiting spawn handles

```rust
// Task might not complete!
tokio::spawn(async {
    save_to_db(data).await;
});
// Program exits before save finishes
```

```rust
// Better - ensure completion
let handle = tokio::spawn(async {
    save_to_db(data).await;
});
handle.await.unwrap();
```

---

## Summary

- Use `join!` for **related concurrent work** that needs to finish together
- Use `spawn` for **independent background tasks** that run separately
- Both run concurrently, but `join!` is simpler when you need all results immediately

Happy coding! 🦀