# Actix-web Interview Questions - Complete Answers

## Core Rust Concepts

### 1. How does Rust's ownership system affect web application design?

Rust's ownership system fundamentally shapes how you architect web applications:

**State Management**: You can't simply share mutable state across handlers like in other languages. You must explicitly use thread-safe types like `Arc<Mutex<T>>` or `Arc<RwLock<T>>` to share mutable data across request handlers.

**Request Handlers**: Each handler takes ownership or borrows its parameters. Extractors automatically handle borrowing, so you don't need to manually manage lifetimes in most cases.

**Memory Safety**: No garbage collection means predictable performance. The compiler ensures memory safety at compile time, eliminating entire classes of bugs like data races and use-after-free errors.

**Clone vs Reference**: When passing data to async tasks or closures, you often need to clone `Arc` pointers rather than borrowing, since the lifetime of the async task may outlive the current scope.

```rust
// Shared application state
struct AppState {
    counter: Arc<Mutex<i32>>,
}

async fn increment(data: web::Data<AppState>) -> impl Responder {
    let mut counter = data.counter.lock().unwrap();
    *counter += 1;
    format!("Counter: {}", counter)
}
```

### 2. Explain how lifetimes work in request handlers and middleware

**Request Handler Lifetimes**: Most of the time, you don't need to specify lifetimes explicitly because Actix-web's extractors handle them automatically. The framework ensures that extracted data lives long enough for the handler to complete.

**Middleware Lifetimes**: When implementing custom middleware, you need to be careful with lifetimes because middleware wraps the entire request-response cycle. The middleware must not hold references that outlive the request.

**Common Pattern**: Use owned types or `'static` lifetimes when data needs to persist across async boundaries.

```rust
// Extractor with automatic lifetime management
async fn handler(path: web::Path<String>) -> impl Responder {
    // path is borrowed from the request, but you don't need to specify lifetimes
    format!("Path: {}", path)
}

// Custom extractor with explicit lifetimes
impl<'a> FromRequest for CustomExtractor<'a> {
    type Error = Error;
    type Future = Ready<Result<Self, Self::Error>>;

    fn from_request(req: &HttpRequest, payload: &mut Payload) -> Self::Future {
        // Implementation
    }
}
```

### 3. How do you share state across request handlers safely?

Actix-web provides `web::Data<T>` to share application state safely across handlers:

**Thread-Safe Sharing**: Wrap your state in `Arc` for read-only access, or `Arc<Mutex<T>>` / `Arc<RwLock<T>>` for mutable access.

**App Data**: Register state with `App::app_data()` which automatically wraps it in `web::Data`.

**Cloning**: `web::Data<T>` is cheap to clone (it's just cloning an `Arc`), so each handler gets its own handle to the shared state.

```rust
struct AppState {
    db_pool: Pool<Postgres>,
    api_key: String,
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let state = web::Data::new(AppState {
        db_pool: create_pool().await,
        api_key: "secret".to_string(),
    });

    HttpServer::new(move || {
        App::new()
            .app_data(state.clone())
            .route("/", web::get().to(index))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}

async fn index(data: web::Data<AppState>) -> impl Responder {
    format!("API Key: {}", data.api_key)
}
```

---

## Actix-web Fundamentals

### Application Structure

#### 1. Explain the difference between `App`, `HttpServer`, and `web::scope`

**HttpServer**: The top-level server that manages worker threads and binds to TCP sockets. It creates multiple instances of your application (one per worker thread).

```rust
HttpServer::new(|| {
    App::new()
})
.workers(4) // Number of worker threads
.bind("127.0.0.1:8080")?
.run()
```

**App**: Represents a single application instance with its own configuration, middleware, routes, and state. Each worker thread gets its own `App` instance.

```rust
App::new()
    .app_data(app_state.clone())
    .wrap(middleware::Logger::default())
    .route("/", web::get().to(index))
```

**web::scope**: Creates a group of routes with a common prefix and shared configuration. Useful for organizing related endpoints and applying middleware to specific route groups.

```rust
App::new()
    .service(
        web::scope("/api")
            .wrap(AuthMiddleware)
            .service(
                web::scope("/v1")
                    .route("/users", web::get().to(get_users))
                    .route("/users/{id}", web::get().to(get_user))
            )
    )
```

#### 2. How do you structure a large Actix-web application?

**Modular Organization**:
```
src/
├── main.rs              # Server setup and configuration
├── config.rs            # Configuration management
├── models/              # Data models
│   ├── mod.rs
│   ├── user.rs
│   └── post.rs
├── handlers/            # Request handlers
│   ├── mod.rs
│   ├── user_handlers.rs
│   └── post_handlers.rs
├── services/            # Business logic
│   ├── mod.rs
│   ├── user_service.rs
│   └── post_service.rs
├── middleware/          # Custom middleware
│   ├── mod.rs
│   └── auth.rs
├── routes/              # Route configuration
│   ├── mod.rs
│   ├── user_routes.rs
│   └── post_routes.rs
├── db/                  # Database layer
│   ├── mod.rs
│   └── pool.rs
└── utils/               # Utilities and helpers
    ├── mod.rs
    └── errors.rs
```

**Best Practices**:
- Separate route configuration from handler implementation
- Keep handlers thin, move business logic to services
- Use a dedicated error type for the entire application
- Group related functionality into modules
- Use dependency injection via `web::Data` for testability

```rust
// routes/user_routes.rs
pub fn configure(cfg: &mut web::ServiceConfig) {
    cfg.service(
        web::scope("/users")
            .route("", web::get().to(handlers::get_users))
            .route("", web::post().to(handlers::create_user))
            .route("/{id}", web::get().to(handlers::get_user))
    );
}

// main.rs
App::new()
    .configure(routes::user_routes::configure)
    .configure(routes::post_routes::configure)
```

#### 3. What is the difference between `App::data()` and `App::app_data()`?

**App::data()** (Deprecated): The older method that automatically wraps your data in `web::Data<T>` and `Arc`. It has been deprecated in favor of `app_data()`.

**App::app_data()**: The current recommended method. You have full control over how data is wrapped. You can use `web::Data::new()` for automatic `Arc` wrapping or provide your own wrapper.

```rust
// Modern approach with app_data
let state = web::Data::new(AppState { /* ... */ });

HttpServer::new(move || {
    App::new()
        .app_data(state.clone()) // Explicit wrapping
        .app_data(web::JsonConfig::default().limit(4096)) // Configuration
})

// You can also use different wrapper types
App::new()
    .app_data(web::Data::new(db_pool))
    .app_data(web::PayloadConfig::default())
```

**Key Difference**: `app_data()` is more flexible and allows you to register multiple types of configuration and data, not just application state.

### Request Handling

#### 1. How do extractors work in Actix-web?

Extractors implement the `FromRequest` trait, which allows them to extract data from incoming HTTP requests. They run before your handler and convert raw request data into typed Rust structures.

**Extraction Process**:
1. Request arrives at the server
2. Extractors are executed in the order they appear in the handler signature
3. Each extractor processes the request and produces a value or error
4. If all extractors succeed, the handler is called with the extracted values
5. If any extractor fails, an error response is returned

```rust
// Multiple extractors in one handler
async fn handler(
    path: web::Path<(String, u32)>,    // Extract from URL path
    query: web::Query<SearchParams>,    // Extract from query string
    json: web::Json<CreateUser>,        // Extract and parse JSON body
    req: HttpRequest,                   // Access to raw request
    data: web::Data<AppState>,          // Access to app state
) -> impl Responder {
    // All extractors have succeeded by the time this runs
}
```

**Important**: Extractors consume the request payload, so you can only use one payload extractor (`Json`, `Form`, `Bytes`, etc.) per handler.

#### 2. Explain the difference between `Path`, `Query`, `Json`, and `Form` extractors

**Path**: Extracts dynamic segments from the URL path.

```rust
// Route: /users/{id}/posts/{post_id}
async fn handler(path: web::Path<(u32, u32)>) -> impl Responder {
    let (user_id, post_id) = path.into_inner();
    format!("User: {}, Post: {}", user_id, post_id)
}

// Or with named struct
#[derive(Deserialize)]
struct PathParams {
    id: u32,
    post_id: u32,
}

async fn handler(path: web::Path<PathParams>) -> impl Responder {
    format!("User: {}, Post: {}", path.id, path.post_id)
}
```

**Query**: Extracts data from query string parameters.

```rust
// URL: /search?q=rust&page=2&limit=10
#[derive(Deserialize)]
struct SearchQuery {
    q: String,
    page: Option<u32>,
    limit: Option<u32>,
}

async fn search(query: web::Query<SearchQuery>) -> impl Responder {
    format!("Searching for: {}", query.q)
}
```

**Json**: Extracts and deserializes JSON from the request body. Sets a content-type requirement.

```rust
#[derive(Deserialize)]
struct CreateUser {
    name: String,
    email: String,
}

async fn create_user(json: web::Json<CreateUser>) -> impl Responder {
    format!("Creating user: {}", json.name)
}
```

**Form**: Extracts and deserializes URL-encoded form data from the request body.

```rust
#[derive(Deserialize)]
struct LoginForm {
    username: String,
    password: String,
}

async fn login(form: web::Form<LoginForm>) -> impl Responder {
    format!("Login attempt for: {}", form.username)
}
```

#### 3. How would you implement a custom extractor?

Implement the `FromRequest` trait for your type:

```rust
use actix_web::{FromRequest, HttpRequest, dev::Payload, Error};
use futures::future::{ready, Ready};

struct AuthenticatedUser {
    user_id: u32,
    username: String,
}

impl FromRequest for AuthenticatedUser {
    type Error = Error;
    type Future = Ready<Result<Self, Self::Error>>;

    fn from_request(req: &HttpRequest, _payload: &mut Payload) -> Self::Future {
        // Extract authorization header
        let auth_header = req.headers().get("Authorization");
        
        match auth_header {
            Some(header) => {
                // Parse and validate token (simplified)
                let token = header.to_str().unwrap_or("");
                
                if token.starts_with("Bearer ") {
                    // In reality, you'd validate the JWT token here
                    ready(Ok(AuthenticatedUser {
                        user_id: 1,
                        username: "john_doe".to_string(),
                    }))
                } else {
                    ready(Err(actix_web::error::ErrorUnauthorized("Invalid token")))
                }
            }
            None => ready(Err(actix_web::error::ErrorUnauthorized("Missing authorization header")))
        }
    }
}

// Usage
async fn protected_route(user: AuthenticatedUser) -> impl Responder {
    format!("Hello, {}!", user.username)
}
```

**For async extraction**:

```rust
use actix_web::dev::Payload;
use futures::future::{Future, LocalBoxFuture};
use std::pin::Pin;

impl FromRequest for AuthenticatedUser {
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self, Self::Error>>;

    fn from_request(req: &HttpRequest, _payload: &mut Payload) -> Self::Future {
        let req = req.clone();
        
        Box::pin(async move {
            // Async operations like database queries
            let token = req
                .headers()
                .get("Authorization")
                .and_then(|h| h.to_str().ok())
                .ok_or_else(|| actix_web::error::ErrorUnauthorized("Missing token"))?;
            
            // Validate token asynchronously
            let user = validate_token_async(token).await?;
            
            Ok(user)
        })
    }
}
```

### Async/Await

#### 1. How does async/await work in Actix-web handlers?

Actix-web is built on top of Tokio, an async runtime for Rust. All handlers are async by default:

**Async Execution**: When a request arrives, the handler returns a Future that represents the eventual result. The Tokio runtime schedules this future for execution without blocking the thread.

**Non-blocking I/O**: Async operations like database queries, HTTP requests, and file I/O don't block the thread. Instead, they yield control back to the runtime, allowing other tasks to run.

**Handler Types**: Handlers can return anything that implements `Responder`:

```rust
// Simple async handler
async fn index() -> impl Responder {
    "Hello, world!"
}

// Async handler with I/O operations
async fn fetch_data(pool: web::Data<DbPool>) -> Result<HttpResponse, Error> {
    let users = sqlx::query_as::<_, User>("SELECT * FROM users")
        .fetch_all(pool.get_ref())
        .await?;  // Async database query
    
    Ok(HttpResponse::Ok().json(users))
}

// Multiple async operations
async fn complex_handler() -> Result<HttpResponse, Error> {
    let (result1, result2, result3) = tokio::join!(
        async_operation_1(),
        async_operation_2(),
        async_operation_3(),
    );
    
    Ok(HttpResponse::Ok().json(/* combined results */))
}
```

**Key Points**:
- `.await` points are where the function can be suspended
- The runtime can switch to processing other requests during await points
- This enables handling thousands of concurrent connections with just a few threads

#### 2. Explain the actor model that Actix-web is built on

**Historical Note**: Earlier versions of Actix-web (v2 and below) were heavily based on the Actix actor framework. Since Actix-web v3+, the framework has moved away from actors for most use cases and now primarily uses standard async/await.

**Original Actor Model Concepts**:
- Actors are independent units of computation with their own state
- Actors communicate through message passing
- Each actor processes messages sequentially, eliminating race conditions
- Actors can be distributed across threads

**Modern Actix-web**: While the core framework no longer requires actors, you can still use the Actix actor system for specific use cases:

```rust
use actix::prelude::*;

// Define an actor
struct MyActor {
    count: usize,
}

impl Actor for MyActor {
    type Context = Context<Self>;
}

// Define a message
#[derive(Message)]
#[rtype(result = "usize")]
struct Increment;

// Handle the message
impl Handler<Increment> for MyActor {
    type Result = usize;

    fn handle(&mut self, _msg: Increment, _ctx: &mut Context<Self>) -> Self::Result {
        self.count += 1;
        self.count
    }
}

// Using actors in handlers (if needed)
async fn handler(actor: web::Data<Addr<MyActor>>) -> impl Responder {
    let count = actor.send(Increment).await.unwrap();
    format!("Count: {}", count)
}
```

**When to use Actors**: 
- Managing WebSocket connections
- Background task processing
- Complex stateful services that need isolation
- When you need supervision trees and automatic restart on failure

#### 3. How do you handle blocking operations in async handlers?

Blocking operations can stall the async runtime, so they must be handled carefully:

**Method 1: Use `web::block()` for CPU-intensive or blocking I/O**

```rust
use actix_web::web;

async fn handler() -> Result<HttpResponse, Error> {
    // Run blocking code in a separate thread pool
    let result = web::block(|| {
        // CPU-intensive work
        compute_heavy_calculation()
    }).await?;
    
    Ok(HttpResponse::Ok().json(result))
}

// Database example with synchronous driver
async fn get_user(pool: web::Data<DbPool>) -> Result<HttpResponse, Error> {
    let user = web::block(move || {
        let conn = pool.get()?;
        users::table.find(1).first::<User>(&conn)
    }).await??;  // Note: double ?? for two Result types
    
    Ok(HttpResponse::Ok().json(user))
}
```

**Method 2: Use async-native libraries when possible**

```rust
// Instead of blocking database drivers, use async ones
use sqlx::PgPool;

async fn get_user(pool: web::Data<PgPool>) -> Result<HttpResponse, Error> {
    let user = sqlx::query_as::<_, User>("SELECT * FROM users WHERE id = $1")
        .bind(1)
        .fetch_one(pool.get_ref())
        .await?;
    
    Ok(HttpResponse::Ok().json(user))
}
```

**Method 3: Spawn blocking tasks with Tokio**

```rust
use tokio::task;

async fn handler() -> Result<HttpResponse, Error> {
    let result = task::spawn_blocking(|| {
        // Blocking operation
        std::thread::sleep(std::time::Duration::from_secs(1));
        "Done"
    }).await?;
    
    Ok(HttpResponse::Ok().body(result))
}
```

**Best Practices**:
- Always use async libraries when available (tokio-postgres, reqwest, etc.)
- Use `web::block()` for compatibility with sync libraries
- Never use `std::thread::sleep()` in async code; use `tokio::time::sleep()` instead
- Limit the size of blocking thread pool with `HttpServer::worker_max_blocking_threads()`

---

## State Management

### 1. How do you share application state across handlers?

Use `web::Data<T>` which is Actix-web's wrapper around `Arc<T>`:

```rust
struct AppState {
    app_name: String,
    counter: Arc<Mutex<i32>>,
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // Create shared state
    let app_state = web::Data::new(AppState {
        app_name: "My App".to_string(),
        counter: Arc::new(Mutex::new(0)),
    });

    HttpServer::new(move || {
        App::new()
            .app_data(app_state.clone())  // Clone is cheap (just Arc clone)
            .route("/", web::get().to(index))
            .route("/increment", web::post().to(increment))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}

async fn index(data: web::Data<AppState>) -> impl Responder {
    let counter = data.counter.lock().unwrap();
    format!("{}: Counter is {}", data.app_name, counter)
}

async fn increment(data: web::Data<AppState>) -> impl Responder {
    let mut counter = data.counter.lock().unwrap();
    *counter += 1;
    format!("Incremented to {}", counter)
}
```

**Multiple State Types**:

```rust
struct DbPool(Pool<Postgres>);
struct ApiConfig { api_key: String }

HttpServer::new(move || {
    App::new()
        .app_data(web::Data::new(DbPool(pool.clone())))
        .app_data(web::Data::new(ApiConfig { 
            api_key: env::var("API_KEY").unwrap() 
        }))
})

async fn handler(
    db: web::Data<DbPool>,
    config: web::Data<ApiConfig>,
) -> impl Responder {
    // Use both
}
```

### 2. Explain `web::Data<T>` and when to use `Arc` and `Mutex`

**web::Data<T>**: An application data wrapper that:
- Automatically wraps your data in `Arc` for thread-safe reference counting
- Is cheap to clone (only clones the `Arc` pointer, not the data)
- Provides automatic extraction in handlers
- Ensures data lives as long as the application

```rust
// web::Data<T> is essentially Arc<T> with extractor support
let data = web::Data::new(MyState { /* ... */ });
// Under the hood, this is Arc<MyState>
```

**When to use Arc**: For immutable shared state or read-only data.

```rust
struct Config {
    api_key: String,
    max_connections: usize,
}

let config = web::Data::new(Config {
    api_key: "secret".to_string(),
    max_connections: 100,
});
// No Mutex needed - data is read-only
```

**When to use Mutex**: For mutable shared state that needs exclusive access.

```rust
struct AppState {
    visit_count: Arc<Mutex<u64>>,
}

async fn increment(state: web::Data<AppState>) -> impl Responder {
    let mut count = state.visit_count.lock().unwrap();
    *count += 1;
    format!("Visits: {}", count)
}
```

**When to use RwLock**: For read-heavy workloads where multiple readers can access simultaneously.

```rust
struct Cache {
    data: Arc<RwLock<HashMap<String, String>>>,
}

async fn get_cached(
    cache: web::Data<Cache>,
    key: web::Path<String>,
) -> impl Responder {
    let data = cache.data.read().unwrap();
    match data.get(key.as_str()) {
        Some(value) => HttpResponse::Ok().body(value.clone()),
        None => HttpResponse::NotFound().finish(),
    }
}

async fn set_cached(
    cache: web::Data<Cache>,
    item: web::Json<CacheItem>,
) -> impl Responder {
    let mut data = cache.data.write().unwrap();
    data.insert(item.key.clone(), item.value.clone());
    HttpResponse::Ok().finish()
}
```

**Best Practices**:
- Use `Arc` alone for read-only data
- Use `Arc<Mutex<T>>` for small critical sections
- Use `Arc<RwLock<T>>` for read-heavy scenarios
- Consider using lock-free data structures from `crossbeam` for high-contention scenarios
- Always keep lock scopes small to avoid blocking

### 3. What's the difference between app-level and handler-level state?

**App-level state**: Shared across all handlers in the application. Registered with `app_data()` or `data()`.

```rust
struct GlobalState {
    db_pool: Pool<Postgres>,
    cache: Arc<RwLock<Cache>>,
}

HttpServer::new(move || {
    let global_state = web::Data::new(GlobalState { /* ... */ });
    
    App::new()
        .app_data(global_state.clone())  // Available to all routes
        .service(web::scope("/api/v1").configure(api_config))
        .service(web::scope("/api/v2").configure(api_v2_config))
})
```

**Handler-level state**: Specific to a particular handler or route. Created within the handler or passed as a closure.

```rust
// Handler-level state via closure
fn create_counter_handler() -> impl Fn() -> impl Future<Output = impl Responder> {
    let counter = Arc::new(Mutex::new(0));
    
    move || {
        let counter = counter.clone();
        async move {
            let mut count = counter.lock().unwrap();
            *count += 1;
            format!("Count: {}", count)
        }
    }
}

App::new()
    .route("/counter", web::get().to(create_counter_handler()))
```

**Scope-level state**: Shared within a specific scope but not the entire app.

```rust
App::new()
    .service(
        web::scope("/admin")
            .app_data(web::Data::new(AdminConfig { /* ... */ }))
            .route("/users", web::get().to(admin_users))
            .route("/settings", web::get().to(admin_settings))
    )
    .service(
        web::scope("/public")
            .app_data(web::Data::new(PublicConfig { /* ... */ }))
            .route("/info", web::get().to(public_info))
    )
```

**Use Cases**:
- **App-level**: Database pools, configuration, global caches
- **Scope-level**: Feature-specific configuration, versioned API settings
- **Handler-level**: Per-endpoint metrics, specialized state that doesn't need sharing

---

## Middleware

### 1. How do you create custom middleware in Actix-web?

Implement the `Transform` and `Service` traits:

```rust
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpMessage,
};
use futures::future::LocalBoxFuture;
use std::future::{ready, Ready};

// Middleware factory
pub struct RequestLogger;

impl<S, B> Transform<S, ServiceRequest> for RequestLogger
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = RequestLoggerMiddleware<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(RequestLoggerMiddleware { service }))
    }
}

// Middleware service
pub struct RequestLoggerMiddleware<S> {
    service: S,
}

impl<S, B> Service<ServiceRequest> for RequestLoggerMiddleware<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;

    forward_ready!(service);

    fn call(&self, req: ServiceRequest) -> Self::Future {
        let method = req.method().clone();
        let path = req.path().to_string();
        
        println!("Incoming request: {} {}", method, path);

        let fut = self.service.call(req);

        Box::pin(async move {
            let res = fut.await?;
            println!("Response status: {}", res.status());
            Ok(res)
        })
    }
}

// Usage
App::new()
    .wrap(RequestLogger)
    .route("/", web::get().to(index))
```

**Simpler middleware with `wrap_fn`**:

```rust
use actix_web::dev::Service as _;

App::new()
    .wrap_fn(|req, srv| {
        println!("Request to: {}", req.path());
        let fut = srv.call(req);
        async {
            let res = fut.await?;
            println!("Response: {}", res.status());
            Ok(res)
        }
    })
```

### 2. Explain the middleware chain execution order

Middleware executes in **LIFO order** (Last In, First Out) for requests and **FIFO order** (First In, First Out) for responses:

```rust
App::new()
    .wrap(MiddlewareA)  // Executes 3rd on request, 1st on response
    .wrap(MiddlewareB)  // Executes 2nd on request, 2nd on response
    .wrap(MiddlewareC)  // Executes 1st on request, 3rd on response
    .route("/", web::get().to(handler))
```

**Execution Flow**:
```
Request → MiddlewareC → MiddlewareB → MiddlewareA → Handler
Response ← MiddlewareC ← MiddlewareB ← MiddlewareA ← Handler
```

**Practical Example**:

```rust
App::new()
    .wrap(middleware::Logger::default())           // Logs first, finishes last
    .wrap(middleware::Compress::default())         // Compresses response
    .wrap(AuthenticationMiddleware)                // Checks auth
    .route("/api/users", web::get().to(get_users))
```

Order matters:
1. Logger sees the original request
2. Compress will compress the response before logger logs it
3. Auth runs before the handler, can reject early

**Scope-specific middleware**:

```rust
App::new()
    .wrap(GlobalMiddleware)
    .service(
        web::scope("/admin")
            .wrap(AdminAuthMiddleware)  // Only for /admin routes
            .route("/users", web::get().to(admin_users))
    )
    .service(
        web::scope("/public")
            .route("/info", web::get().to(public_info))
    )
```

### 3. How would you implement authentication middleware?

Here's a complete JWT authentication middleware example:

```rust
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpMessage, HttpResponse,
};
use futures::future::LocalBoxFuture;
use std::future::{ready, Ready};
use jsonwebtoken::{decode, DecodingKey, Validation, Algorithm};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    sub: String,
    exp: usize,
}

pub struct Authentication;

impl<S, B> Transform<S, ServiceRequest> for Authentication
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = AuthenticationMiddleware<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(AuthenticationMiddleware { service }))
    }
}

pub struct AuthenticationMiddleware<S> {
    service: S,
}

impl<S, B> Service<ServiceRequest> for AuthenticationMiddleware<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;

    forward_ready!(service);

    fn call(&self, req: ServiceRequest) -> Self::Future {
        // Extract Authorization header
        let auth_header = req.headers().get("Authorization");
        
        let token = match auth_header {
            Some(header) => {
                match header.to_str() {
                    Ok(h) if h.starts_with("Bearer ") => {
                        h.trim_start_matches("Bearer ").to_string()
                    }
                    _ => {
                        return Box::pin(async {
                            Err(actix_web::error::ErrorUnauthorized("Invalid authorization header"))
                        });
                    }
                }
            }
            None => {
                return Box::pin(async {
                    Err(actix_web::error::ErrorUnauthorized("Missing authorization header"))
                });
            }
        };

        // Validate token
        let secret = "your-secret-key";
        let validation = Validation::new(Algorithm::HS256);
        
        match decode::<Claims>(
            &token,
            &DecodingKey::from_secret(secret.as_ref()),
            &validation,
        ) {
            Ok(token_data) => {
                // Store user info in request extensions
                req.extensions_mut().insert(token_data.claims.sub.clone());
                
                let fut = self.service.call(req);
                Box::pin(async move {
                    let res = fut.await?;
                    Ok(res)
                })
            }
            Err(_) => {
                Box::pin(async {
                    Err(actix_web::error::ErrorUnauthorized("Invalid token"))
                })
            }
        }
    }
}

// Usage
App::new()
    .service(
        web::scope("/api")
            .wrap(Authentication)
            .route("/protected", web::get().to(protected_handler))
    )
    .route("/login", web::post().to(login))

// Access user info in handler
async fn protected_handler(req: HttpRequest) -> impl Responder {
    let extensions = req.extensions();
    let user_id = extensions.get::<String>().unwrap();
    format!("Hello, user {}!", user_id)
}
```

**Alternative: Using extractors for selective auth**:

```rust
use actix_web::{FromRequest, HttpRequest};

struct AuthenticatedUser {
    user_id: String,
}

impl FromRequest for AuthenticatedUser {
    type Error = Error;
    type Future = Ready<Result<Self, Self::Error>>;

    fn from_request(req: &HttpRequest, _: &mut Payload) -> Self::Future {
        // Similar token validation logic
        ready(Ok(AuthenticatedUser {
            user_id: "extracted_from_token".to_string(),
        }))
    }
}

// Only protected if you use the extractor
async fn handler(user: AuthenticatedUser) -> impl Responder {
    format!("Authenticated as {}", user.user_id)
}
```

---

## Performance & Concurrency

### 1. How does Actix-web handle concurrent requests?

Actix-web uses a **multi-threaded async runtime** based on Tokio:

**Worker Threads**: By default, Actix-web creates one worker thread per CPU core. Each worker:
- Runs its own Tokio runtime
- Has its own event loop
- Handles multiple concurrent requests via async/await
- Doesn't share memory with other workers (except through Arc)

```rust
HttpServer::new(|| App::new())
    .workers(4)  // Explicitly set worker count
    .bind("127.0.0.1:8080")?
    .run()
```

**Request Flow**:
1. Connection accepted by a worker's Tokio runtime
2. Request parsed and routed to handler
3. Handler returns a Future
4. Future scheduled on the worker's runtime
5. During `.await` points, the worker can process other requests
6. Response sent when Future completes

**Concurrency Model**:
```
Worker 1: [Request A] [Request B] [Request C] ...
Worker 2: [Request D] [Request E] [Request F] ...
Worker 3: [Request G] [Request H] [Request I] ...
Worker 4: [Request J] [Request K] [Request L] ...
```

Each worker can handle thousands of concurrent connections because:
- Async I/O doesn't block threads
- Small memory footprint per connection
- Efficient task scheduling by Tokio

**Benefits**:
- No need for thread pools for I/O operations
- Minimal context switching
- High throughput with low resource usage
- Automatic load balancing across workers

### 2. Explain worker threads and how to configure them

**Worker Configuration**:

```rust
HttpServer::new(|| App::new())
    .workers(8)                    // Number of worker threads
    .max_connections(25000)        // Max connections per worker
    .max_connection_rate(256)      // Max connections per second per worker
    .client_timeout(5000)          // Timeout in milliseconds
    .client_disconnect(5000)       // Disconnect timeout
    .bind("127.0.0.1:8080")?
    .run()
```

**Default Settings**:
- Workers: Number of logical CPU cores
- Max connections per worker: 25,000
- Connection rate: 256/sec per worker

**Worker Thread Behavior**:
- Each worker is completely independent
- Each has its own `App` instance (from the factory closure)
- State must be cloned or wrapped in `Arc` to share
- Load balancing happens at the OS level (socket sharing)

**Choosing Worker Count**:

```rust
// For CPU-bound workloads (computation heavy)
.workers(num_cpus::get())  // One per core

// For I/O-bound workloads (most web apps)
.workers(num_cpus::get() * 2)  // Can handle more

// For development
.workers(1)  // Easier debugging

// Maximum recommendation
.workers(num_cpus::get() * 4)  // Rarely beneficial to go higher
```

**Blocking Thread Pool**:

```rust
HttpServer::new(|| App::new())
    .worker_max_blocking_threads(512)  // Default is 512
    .bind("127.0.0.1:8080")?
```

This controls the thread pool used by `web::block()` for blocking operations.

**Best Practices**:
- Start with default worker count
- Monitor CPU usage and adjust
- More workers != better performance (diminishing returns)
- Consider connection count and request patterns
- Use load testing to find optimal configuration

### 3. How would you optimize database connection pooling?

**Using SQLx with async connections**:

```rust
use sqlx::postgres::PgPoolOptions;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let pool = PgPoolOptions::new()
        .max_connections(20)           // Total connections in pool
        .min_connections(5)            // Minimum idle connections
        .acquire_timeout(Duration::from_secs(3))  // Wait time for connection
        .idle_timeout(Duration::from_secs(600))   // Close idle connections after
        .max_lifetime(Duration::from_secs(1800))  // Max connection lifetime
        .connect("postgres://user:pass@localhost/db")
        .await
        .expect("Failed to create pool");

    let pool_data = web::Data::new(pool);

    HttpServer::new(move || {
        App::new()
            .app_data(pool_data.clone())
            .route("/users", web::get().to(get_users))
    })
    .workers(4)
    .bind("127.0.0.1:8080")?
    .run()
    .await
}

async fn get_users(pool: web::Data<PgPool>) -> Result<HttpResponse, Error> {
    let users = sqlx::query_as::<_, User>("SELECT * FROM users")
        .fetch_all(pool.get_ref())
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;

    Ok(HttpResponse::Ok().json(users))
}
```

**Optimization Guidelines**:

**1. Pool Size Calculation**:
```rust
// Formula: connections = (worker_threads * concurrent_requests_per_worker) / 4
// For 4 workers, ~1000 concurrent requests per worker:
// connections = (4 * 1000) / 4 = 1000
// In practice, use much lower numbers:

let pool = PgPoolOptions::new()
    .max_connections(20)  // Start conservative
    .min_connections(5)   // Keep some warm
```

**2. Connection Timeouts**:
```rust
let pool = PgPoolOptions::new()
    .acquire_timeout(Duration::from_secs(3))  // Don't wait forever
    .connect_timeout(Duration::from_secs(5))  // Connection establishment
```

**3. Connection Lifecycle**:
```rust
let pool = PgPoolOptions::new()
    .max_lifetime(Duration::from_secs(30 * 60))   // 30 minutes
    .idle_timeout(Duration::from_secs(10 * 60))   // 10 minutes
    .test_before_acquire(true)  // Validate connections
```

**4. Using Diesel with r2d2** (for sync connections):
```rust
use diesel::r2d2::{self, ConnectionManager};

type DbPool = r2d2::Pool<ConnectionManager<PgConnection>>;

fn create_pool() -> DbPool {
    let manager = ConnectionManager::<PgConnection>::new(database_url);
    
    r2d2::Pool::builder()
        .max_size(15)                    // Max connections
        .min_idle(Some(5))               // Min idle
        .connection_timeout(Duration::from_secs(3))
        .idle_timeout(Some(Duration::from_secs(600)))
        .build(manager)
        .expect("Failed to create pool")
}

async fn handler(pool: web::Data<DbPool>) -> Result<HttpResponse, Error> {
    let users = web::block(move || {
        let conn = pool.get()?;
        users::table.load::<User>(&conn)
    })
    .await??;

    Ok(HttpResponse::Ok().json(users))
}
```

**5. Monitoring and Tuning**:
```rust
// Add connection pool metrics
async fn pool_stats(pool: web::Data<PgPool>) -> impl Responder {
    format!(
        "Connections: {} / {}, Idle: {}",
        pool.size(),
        pool.options().get_max_connections(),
        pool.num_idle()
    )
}
```

**Best Practices**:
- Start with `max_connections = num_workers * 3`
- Monitor database connection count
- Use connection pooling at application level, not per-worker
- Set appropriate timeouts to prevent connection exhaustion
- Use async database drivers (SQLx, tokio-postgres) over sync ones
- Test under load to find optimal settings
- Consider database max_connections limit

---

## Practical Scenarios

### 1. How would you implement rate limiting?

**Using middleware with in-memory store**:

```rust
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpResponse,
};
use futures::future::LocalBoxFuture;
use std::collections::HashMap;
use std::future::{ready, Ready};
use std::sync::Arc;
use std::time::{Duration, Instant};
use tokio::sync::RwLock;

#[derive(Clone)]
struct RateLimiter {
    store: Arc<RwLock<HashMap<String, Vec<Instant>>>>,
    max_requests: usize,
    window: Duration,
}

impl RateLimiter {
    fn new(max_requests: usize, window: Duration) -> Self {
        Self {
            store: Arc::new(RwLock::new(HashMap::new())),
            max_requests,
            window,
        }
    }
}

impl<S, B> Transform<S, ServiceRequest> for RateLimiter
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = RateLimiterMiddleware<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(RateLimiterMiddleware {
            service,
            store: self.store.clone(),
            max_requests: self.max_requests,
            window: self.window,
        }))
    }
}

pub struct RateLimiterMiddleware<S> {
    service: S,
    store: Arc<RwLock<HashMap<String, Vec<Instant>>>>,
    max_requests: usize,
    window: Duration,
}

impl<S, B> Service<ServiceRequest> for RateLimiterMiddleware<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;

    forward_ready!(service);

    fn call(&self, req: ServiceRequest) -> Self::Future {
        // Extract client identifier (IP address in this case)
        let client_ip = req
            .connection_info()
            .realip_remote_addr()
            .unwrap_or("unknown")
            .to_string();

        let store = self.store.clone();
        let max_requests = self.max_requests;
        let window = self.window;

        Box::pin(async move {
            let now = Instant::now();
            let mut store = store.write().await;

            // Get or create entry for this client
            let requests = store.entry(client_ip.clone()).or_insert_with(Vec::new);

            // Remove old requests outside the time window
            requests.retain(|&time| now.duration_since(time) < window);

            // Check if rate limit exceeded
            if requests.len() >= max_requests {
                return Ok(req.into_response(
                    HttpResponse::TooManyRequests()
                        .json(serde_json::json!({
                            "error": "Rate limit exceeded"
                        }))
                        .into_body(),
                ));
            }

            // Record this request
            requests.push(now);

            drop(store); // Release lock before calling service

            let res = self.service.call(req).await?;
            Ok(res)
        })
    }
}

// Usage
App::new()
    .wrap(RateLimiter::new(
        100,                            // 100 requests
        Duration::from_secs(60)         // per minute
    ))
    .route("/api", web::get().to(handler))
```

**Using Redis for distributed rate limiting**:

```rust
use redis::AsyncCommands;

async fn check_rate_limit(
    redis: web::Data<redis::Client>,
    client_ip: &str,
) -> Result<bool, Error> {
    let mut con = redis
        .get_async_connection()
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;

    let key = format!("rate_limit:{}", client_ip);
    
    // Increment counter
    let count: i32 = con
        .incr(&key, 1)
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;

    // Set expiry on first request
    if count == 1 {
        con.expire(&key, 60)
            .await
            .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    }

    Ok(count <= 100) // 100 requests per minute
}

async fn handler(
    req: HttpRequest,
    redis: web::Data<redis::Client>,
) -> Result<HttpResponse, Error> {
    let client_ip = req
        .connection_info()
        .realip_remote_addr()
        .unwrap_or("unknown");

    if !check_rate_limit(redis, client_ip).await? {
        return Ok(HttpResponse::TooManyRequests().json(serde_json::json!({
            "error": "Rate limit exceeded"
        })));
    }

    Ok(HttpResponse::Ok().body("Success"))
}
```

**Using token bucket algorithm**:

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

struct TokenBucket {
    tokens: f64,
    capacity: f64,
    refill_rate: f64,
    last_refill: Instant,
}

impl TokenBucket {
    fn new(capacity: f64, refill_rate: f64) -> Self {
        Self {
            tokens: capacity,
            capacity,
            refill_rate,
            last_refill: Instant::now(),
        }
    }

    fn take(&mut self) -> bool {
        self.refill();
        if self.tokens >= 1.0 {
            self.tokens -= 1.0;
            true
        } else {
            false
        }
    }

    fn refill(&mut self) {
        let now = Instant::now();
        let elapsed = now.duration_since(self.last_refill).as_secs_f64();
        self.tokens = (self.tokens + elapsed * self.refill_rate).min(self.capacity);
        self.last_refill = now;
    }
}

struct RateLimitState {
    buckets: Arc<Mutex<HashMap<String, TokenBucket>>>,
}
```

### 2. How do you handle file uploads efficiently?

**Streaming upload with multipart**:

```rust
use actix_multipart::Multipart;
use actix_web::{web, HttpResponse, Error};
use futures::{StreamExt, TryStreamExt};
use std::io::Write;

async fn upload_file(mut payload: Multipart) -> Result<HttpResponse, Error> {
    // Iterate over multipart stream
    while let Ok(Some(mut field)) = payload.try_next().await {
        let content_disposition = field.content_disposition();
        let filename = content_disposition
            .get_filename()
            .ok_or_else(|| actix_web::error::ErrorBadRequest("Missing filename"))?;

        let filepath = format!("./uploads/{}", sanitize_filename::sanitize(filename));
        
        // Create file
        let mut f = web::block(|| std::fs::File::create(filepath))
            .await?
            .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;

        // Stream field data to file
        while let Some(chunk) = field.next().await {
            let data = chunk?;
            f = web::block(move || f.write_all(&data).map(|_| f))
                .await?
                .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
        }
    }

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "status": "success"
    })))
}

App::new()
    .route("/upload", web::post().to(upload_file))
```

**With size limits and validation**:

```rust
use actix_web::web::PayloadConfig;

const MAX_SIZE: usize = 10 * 1024 * 1024; // 10 MB

async fn upload_with_limits(mut payload: Multipart) -> Result<HttpResponse, Error> {
    let mut total_size: usize = 0;

    while let Ok(Some(mut field)) = payload.try_next().await {
        let content_disposition = field.content_disposition();
        
        // Validate content type
        let content_type = field.content_type();
        if !content_type.to_string().starts_with("image/") {
            return Err(actix_web::error::ErrorBadRequest("Only images allowed"));
        }

        let filename = content_disposition
            .get_filename()
            .ok_or_else(|| actix_web::error::ErrorBadRequest("Missing filename"))?;

        let filepath = format!("./uploads/{}", sanitize_filename::sanitize(filename));
        let mut f = web::block(|| std::fs::File::create(filepath))
            .await??;

        // Stream with size checking
        while let Some(chunk) = field.next().await {
            let data = chunk?;
            total_size += data.len();

            if total_size > MAX_SIZE {
                return Err(actix_web::error::ErrorBadRequest("File too large"));
            }

            f = web::block(move || f.write_all(&data).map(|_| f)).await??;
        }
    }

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "status": "success",
        "size": total_size
    })))
}

// Configure payload limits
App::new()
    .app_data(PayloadConfig::new(MAX_SIZE))
    .route("/upload", web::post().to(upload_with_limits))
```

**Upload to S3 (streaming)**:

```rust
use aws_sdk_s3::{Client, primitives::ByteStream};

async fn upload_to_s3(
    mut payload: Multipart,
    s3_client: web::Data<Client>,
) -> Result<HttpResponse, Error> {
    while let Ok(Some(mut field)) = payload.try_next().await {
        let filename = field
            .content_disposition()
            .get_filename()
            .ok_or_else(|| actix_web::error::ErrorBadRequest("Missing filename"))?;

        // Collect chunks into bytes
        let mut file_data = Vec::new();
        while let Some(chunk) = field.next().await {
            let data = chunk?;
            file_data.extend_from_slice(&data);
        }

        // Upload to S3
        let byte_stream = ByteStream::from(file_data);
        s3_client
            .put_object()
            .bucket("my-bucket")
            .key(filename)
            .body(byte_stream)
            .send()
            .await
            .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    }

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "status": "uploaded to S3"
    })))
}
```

**Best practices for file uploads**:
- Always set payload size limits
- Stream large files instead of loading into memory
- Validate file types and extensions
- Sanitize filenames
- Use virus scanning for user uploads
- Store files outside web root
- Consider using cloud storage (S3, GCS) for scalability
- Implement progress tracking for large uploads

### 3. Explain how to implement WebSocket connections

**Basic WebSocket server**:

```rust
use actix::{Actor, StreamHandler, AsyncContext, ActorContext};
use actix_web::{web, App, Error, HttpRequest, HttpResponse, HttpServer};
use actix_web_actors::ws;

// WebSocket actor
struct MyWebSocket;

impl Actor for MyWebSocket {
    type Context = ws::WebsocketContext<Self>;

    fn started(&mut self, ctx: &mut Self::Context) {
        println!("WebSocket connection started");
    }

    fn stopped(&mut self, ctx: &mut Self::Context) {
        println!("WebSocket connection closed");
    }
}

// Handle WebSocket messages
impl StreamHandler<Result<ws::Message, ws::ProtocolError>> for MyWebSocket {
    fn handle(&mut self, msg: Result<ws::Message, ws::ProtocolError>, ctx: &mut Self::Context) {
        match msg {
            Ok(ws::Message::Ping(msg)) => ctx.pong(&msg),
            Ok(ws::Message::Text(text)) => {
                println!("Received: {}", text);
                ctx.text(format!("Echo: {}", text));
            }
            Ok(ws::Message::Binary(bin)) => ctx.binary(bin),
            Ok(ws::Message::Close(reason)) => {
                ctx.close(reason);
                ctx.stop();
            }
            _ => (),
        }
    }
}

// WebSocket route handler
async fn ws_index(req: HttpRequest, stream: web::Payload) -> Result<HttpResponse, Error> {
    ws::start(MyWebSocket {}, &req, stream)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/ws", web::get().to(ws_index))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

**WebSocket chat server with rooms**:

```rust
use actix::prelude::*;
use actix_web_actors::ws;
use std::collections::{HashMap, HashSet};

// Chat server actor
#[derive(Message)]
#[rtype(result = "()")]
struct Connect {
    addr: Recipient<ServerMessage>,
    room: String,
}

#[derive(Message)]
#[rtype(result = "()")]
struct Disconnect {
    id: usize,
}

#[derive(Message)]
#[rtype(result = "()")]
struct ClientMessage {
    id: usize,
    msg: String,
    room: String,
}

#[derive(Message, Clone)]
#[rtype(result = "()")]
struct ServerMessage(String);

struct ChatServer {
    sessions: HashMap<usize, Recipient<ServerMessage>>,
    rooms: HashMap<String, HashSet<usize>>,
    next_id: usize,
}

impl ChatServer {
    fn new() -> Self {
        ChatServer {
            sessions: HashMap::new(),
            rooms: HashMap::new(),
            next_id: 0,
        }
    }

    fn send_message(&self, room: &str, message: &str, skip_id: usize) {
        if let Some(sessions) = self.rooms.get(room) {
            for id in sessions {
                if *id != skip_id {
                    if let Some(addr) = self.sessions.get(id) {
                        let _ = addr.do_send(ServerMessage(message.to_owned()));
                    }
                }
            }
        }
    }
}

impl Actor for ChatServer {
    type Context = Context<Self>;
}

impl Handler<Connect> for ChatServer {
    type Result = ();

    fn handle(&mut self, msg: Connect, _: &mut Context<Self>) {
        let id = self.next_id;
        self.next_id += 1;

        self.sessions.insert(id, msg.addr);
        
        self.rooms
            .entry(msg.room.clone())
            .or_insert_with(HashSet::new)
            .insert(id);

        self.send_message(&msg.room, &format!("User {} joined", id), 0);
    }
}

impl Handler<Disconnect> for ChatServer {
    type Result = ();

    fn handle(&mut self, msg: Disconnect, _: &mut Context<Self>) {
        self.sessions.remove(&msg.id);
        
        for (room, sessions) in &mut self.rooms {
            if sessions.remove(&msg.id) {
                self.send_message(room, &format!("User {} left", msg.id), 0);
            }
        }
    }
}

impl Handler<ClientMessage> for ChatServer {
    type Result = ();

    fn handle(&mut self, msg: ClientMessage, _: &mut Context<Self>) {
        self.send_message(&msg.room, &msg.msg, msg.id);
    }
}

// WebSocket session
struct WsSession {
    id: usize,
    room: String,
    server: Addr<ChatServer>,
}

impl Actor for WsSession {
    type Context = ws::WebsocketContext<Self>;

    fn started(&mut self, ctx: &mut Self::Context) {
        let addr = ctx.address();
        self.server
            .send(Connect {
                addr: addr.recipient(),
                room: self.room.clone(),
            })
            .into_actor(self)
            .then(|res, _, ctx| {
                match res {
                    Ok(_) => (),
                    Err(_) => ctx.stop(),
                }
                fut::ready(())
            })
            .wait(ctx);
    }

    fn stopping(&mut self, _: &mut Self::Context) -> Running {
        self.server.do_send(Disconnect { id: self.id });
        Running::Stop
    }
}

impl Handler<ServerMessage> for WsSession {
    type Result = ();

    fn handle(&mut self, msg: ServerMessage, ctx: &mut Self::Context) {
        ctx.text(msg.0);
    }
}

impl StreamHandler<Result<ws::Message, ws::ProtocolError>> for WsSession {
    fn handle(&mut self, msg: Result<ws::Message, ws::ProtocolError>, ctx: &mut Self::Context) {
        match msg {
            Ok(ws::Message::Text(text)) => {
                self.server.do_send(ClientMessage {
                    id: self.id,
                    msg: text.to_string(),
                    room: self.room.clone(),
                });
            }
            Ok(ws::Message::Close(reason)) => {
                ctx.close(reason);
                ctx.stop();
            }
            _ => (),
        }
    }
}
```

**WebSocket with authentication**:

```rust
async fn ws_route(
    req: HttpRequest,
    stream: web::Payload,
    srv: web::Data<Addr<ChatServer>>,
) -> Result<HttpResponse, Error> {
    // Validate token before upgrading connection
    let token = req
        .headers()
        .get("Authorization")
        .and_then(|h| h.to_str().ok())
        .ok_or_else(|| actix_web::error::ErrorUnauthorized("Missing token"))?;

    if !validate_token(token) {
        return Err(actix_web::error::ErrorUnauthorized("Invalid token"));
    }

    // Extract room from query params
    let room = req
        .query_string()
        .split('=')
        .nth(1)
        .unwrap_or("general")
        .to_string();

    let ws = WsSession {
        id: 0,
        room,
        server: srv.get_ref().clone(),
    };

    ws::start(ws, &req, stream)
}
```

### 4. How would you structure a REST API with proper error handling and validation?

**Complete REST API structure**:

```rust
// errors.rs - Custom error types
use actix_web::{error::ResponseError, HttpResponse};
use derive_more::{Display, Error};
use serde::Serialize;

#[derive(Debug, Display, Error)]
pub enum ApiError {
    #[display(fmt = "Bad request: {}", message)]
    BadRequest { message: String },
    
    #[display(fmt = "Not found: {}", message)]
    NotFound { message: String },
    
    #[display(fmt = "Unauthorized")]
    Unauthorized,
    
    #[display(fmt = "Internal server error")]
    InternalServerError,
    
    #[display(fmt = "Database error: {}", _0)]
    DatabaseError(String),
}

#[derive(Serialize)]
struct ErrorResponse {
    error: String,
    message: String,
}

impl ResponseError for ApiError {
    fn error_response(&self) -> HttpResponse {
        let error_response = ErrorResponse {
            error: self.to_string(),
            message: match self {
                ApiError::BadRequest { message } => message.clone(),
                ApiError::NotFound { message } => message.clone(),
                ApiError::Unauthorized => "Authentication required".to_string(),
                ApiError::InternalServerError => "An error occurred".to_string(),
                ApiError::DatabaseError(msg) => msg.clone(),
            },
        };

        match self {
            ApiError::BadRequest { .. } => HttpResponse::BadRequest().json(error_response),
            ApiError::NotFound { .. } => HttpResponse::NotFound().json(error_response),
            ApiError::Unauthorized => HttpResponse::Unauthorized().json(error_response),
            ApiError::InternalServerError => HttpResponse::InternalServerError().json(error_response),
            ApiError::DatabaseError(_) => HttpResponse::InternalServerError().json(error_response),
        }
    }
}

// Convert from other error types
impl From<sqlx::Error> for ApiError {
    fn from(err: sqlx::Error) -> Self {
        ApiError::DatabaseError(err.to_string())
    }
}

// models.rs - Data models with validation
use serde::{Deserialize, Serialize};
use validator::Validate;

#[derive(Debug, Serialize, Deserialize, sqlx::FromRow)]
pub struct User {
    pub id: i32,
    pub username: String,
    pub email: String,
    pub created_at: chrono::DateTime<chrono::Utc>,
}

#[derive(Debug, Deserialize, Validate)]
pub struct CreateUserRequest {
    #[validate(length(min = 3, max = 50))]
    pub username: String,
    
    #[validate(email)]
    pub email: String,
    
    #[validate(length(min = 8))]
    pub password: String,
}

#[derive(Debug, Deserialize, Validate)]
pub struct UpdateUserRequest {
    #[validate(length(min = 3, max = 50))]
    pub username: Option<String>,
    
    #[validate(email)]
    pub email: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct UserResponse {
    pub id: i32,
    pub username: String,
    pub email: String,
    pub created_at: String,
}

impl From<User> for UserResponse {
    fn from(user: User) -> Self {
        UserResponse {
            id: user.id,
            username: user.username,
            email: user.email,
            created_at: user.created_at.to_rfc3339(),
        }
    }
}

// handlers.rs - Request handlers with validation
use actix_web::{web, HttpResponse};
use validator::Validate;

pub async fn create_user(
    pool: web::Data<PgPool>,
    user_data: web::Json<CreateUserRequest>,
) -> Result<HttpResponse, ApiError> {
    // Validate input
    user_data.validate().map_err(|e| ApiError::BadRequest {
        message: format!("Validation error: {}", e),
    })?;

    // Hash password
    let password_hash = bcrypt::hash(&user_data.password, 10)
        .map_err(|_| ApiError::InternalServerError)?;

    // Insert into database
    let user = sqlx::query_as::<_, User>(
        "INSERT INTO users (username, email, password_hash) 
         VALUES ($1, $2, $3) 
         RETURNING id, username, email, created_at"
    )
    .bind(&user_data.username)
    .bind(&user_data.email)
    .bind(&password_hash)
    .fetch_one(pool.get_ref())
    .await?;

    Ok(HttpResponse::Created().json(UserResponse::from(user)))
}

pub async fn get_user(
    pool: web::Data<PgPool>,
    user_id: web::Path<i32>,
) -> Result<HttpResponse, ApiError> {
    let user = sqlx::query_as::<_, User>(
        "SELECT id, username, email, created_at FROM users WHERE id = $1"
    )
    .bind(user_id.into_inner())
    .fetch_optional(pool.get_ref())
    .await?
    .ok_or_else(|| ApiError::NotFound {
        message: "User not found".to_string(),
    })?;

    Ok(HttpResponse::Ok().json(UserResponse::from(user)))
}

pub async fn update_user(
    pool: web::Data<PgPool>,
    user_id: web::Path<i32>,
    update_data: web::Json<UpdateUserRequest>,
) -> Result<HttpResponse, ApiError> {
    // Validate input
    update_data.validate().map_err(|e| ApiError::BadRequest {
        message: format!("Validation error: {}", e),
    })?;

    let user_id = user_id.into_inner();

    // Check if user exists
    let exists: (bool,) = sqlx::query_as(
        "SELECT EXISTS(SELECT 1 FROM users WHERE id = $1)"
    )
    .bind(user_id)
    .fetch_one(pool.get_ref())
    .await?;

    if !exists.0 {
        return Err(ApiError::NotFound {
            message: "User not found".to_string(),
        });
    }

    // Build dynamic update query
    let mut query = String::from("UPDATE users SET ");
    let mut updates = Vec::new();
    let mut param_count = 1;

    if let Some(ref username) = update_data.username {
        updates.push(format!("username = ${}", param_count));
        param_count += 1;
    }

    if let Some(ref email) = update_data.email {
        updates.push(format!("email = ${}", param_count));
        param_count += 1;
    }

    if updates.is_empty() {
        return Err(ApiError::BadRequest {
            message: "No fields to update".to_string(),
        });
    }

    query.push_str(&updates.join(", "));
    query.push_str(&format!(" WHERE id = ${} RETURNING id, username, email, created_at", param_count));

    let mut query_builder = sqlx::query_as::<_, User>(&query);

    if let Some(ref username) = update_data.username {
        query_builder = query_builder.bind(username);
    }
    if let Some(ref email) = update_data.email {
        query_builder = query_builder.bind(email);
    }
    query_builder = query_builder.bind(user_id);

    let user = query_builder.fetch_one(pool.get_ref()).await?;

    Ok(HttpResponse::Ok().json(UserResponse::from(user)))
}

pub async fn delete_user(
    pool: web::Data<PgPool>,
    user_id: web::Path<i32>,
) -> Result<HttpResponse, ApiError> {
    let result = sqlx::query("DELETE FROM users WHERE id = $1")
        .bind(user_id.into_inner())
        .execute(pool.get_ref())
        .await?;

    if result.rows_affected() == 0 {
        return Err(ApiError::NotFound {
            message: "User not found".to_string(),
        });
    }

    Ok(HttpResponse::NoContent().finish())
}

pub async fn list_users(
    pool: web::Data<PgPool>,
    query: web::Query<PaginationParams>,
) -> Result<HttpResponse, ApiError> {
    let limit = query.limit.unwrap_or(10).min(100);
    let offset = query.offset.unwrap_or(0);

    let users = sqlx::query_as::<_, User>(
        "SELECT id, username, email, created_at 
         FROM users 
         ORDER BY created_at DESC 
         LIMIT $1 OFFSET $2"
    )
    .bind(limit as i64)
    .bind(offset as i64)
    .fetch_all(pool.get_ref())
    .await?;

    let total: (i64,) = sqlx::query_as("SELECT COUNT(*) FROM users")
        .fetch_one(pool.get_ref())
        .await?;

    let response = PaginatedResponse {
        data: users.into_iter().map(UserResponse::from).collect(),
        total: total.0,
        limit,
        offset,
    };

    Ok(HttpResponse::Ok().json(response))
}

#[derive(Deserialize)]
pub struct PaginationParams {
    pub limit: Option<i32>,
    pub offset: Option<i32>,
}

#[derive(Serialize)]
pub struct PaginatedResponse<T> {
    pub data: Vec<T>,
    pub total: i64,
    pub limit: i32,
    pub offset: i32,
}

// routes.rs - Route configuration
pub fn configure(cfg: &mut web::ServiceConfig) {
    cfg.service(
        web::scope("/api/v1")
            .service(
                web::scope("/users")
                    .route("", web::get().to(list_users))
                    .route("", web::post().to(create_user))
                    .route("/{id}", web::get().to(get_user))
                    .route("/{id}", web::put().to(update_user))
                    .route("/{id}", web::delete().to(delete_user))
            )
    );
}

// main.rs
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    env_logger::init_from_env(env_logger::Env::new().default_filter_or("info"));

    let database_url = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");

    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await
        .expect("Failed to create pool");

    let pool_data = web::Data::new(pool);

    HttpServer::new(move || {
        App::new()
            .app_data(pool_data.clone())
            .wrap(middleware::Logger::default())
            .wrap(middleware::Compress::default())
            .configure(routes::configure)
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

**Key Features**:
- Custom error types with proper HTTP responses
- Input validation using `validator` crate
- Structured error handling with meaningful messages
- Pagination support
- RESTful route design
- Response DTOs separate from database models
- Database connection pooling
- Middleware for logging and compression

---

## Summary

This document covers the essential topics for Actix-web interviews:
- Rust fundamentals (ownership, lifetimes, error handling)
- Actix-web core concepts (app structure, extractors, async/await)
- State management and sharing data safely
- Custom middleware implementation
- Performance optimization and concurrency
- Real-world scenarios (rate limiting, file uploads, WebSockets, REST APIs)

For interview preparation, focus on:
1. Understanding async/await and how it enables concurrency
2. Proper state management with Arc/Mutex/RwLock
3. Error handling patterns
4. Middleware execution order
5. Performance tuning (workers, connection pooling)
6. Security considerations (authentication, validation, rate limiting)