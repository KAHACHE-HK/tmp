*Thinking...*

> **Examining User Needs**
> 
> I'm currently focused on understanding the user's core requirements. Breaking down the request, I've identified the need for a Rust/Wasm interop example. Specifically, I need to figure out which host functionalities should be exposed, prioritizing elements like a full HTTP client. Then, I'll start to design the structure.
> 
> 
> **Planning the Architecture**
> 
> I'm now sketching the architectural blueprint. It's becoming clear that robust HTTP handling is paramount. I'll lean heavily on Rust's `reqwest` crate for the host-side client and build a clear bridge to the Wasm guest. I also recognize that file system access via WASI needs to be integrated. Data transfer complexities will be addressed via linear memory, focusing on a smooth API design.
> 
> 
> **Prioritizing Functionality**
> 
> I'm now working on fleshing out the functionalities to be supported. Thinking through the HTTP client, I will include methods, headers, bodies, and session management via a cookie store, which implies TLS. Also, the file system access through WASI will be mandatory. I plan to rely on `wasmtime` as the runtime and `reqwest` on the host side. The most pressing challenge is how to pass and serialize complex data. I'm leaning toward JSON, but other options are being considered.
> 
> 
> **Mapping Data Structures**
> 
> I'm currently focused on defining the structures for data exchange. I'm designing `HttpRequest` and `HttpResponse` structs in both host and guest. I'll use JSON for serializing and deserializing these structures to facilitate interaction. The `HttpRequest` will contain HTTP-related information, and the `HttpResponse` will mirror the result. The goal is for both the host and the guest to easily serialize and deserialize the shared data using serde.
> 
> 
> **Outlining Guest Implementation**
> 
> I'm now focused on the guest-side implementation details. I'll define `HttpRequest` and `HttpResponse` structs, serializing them with JSON using `serde`. The `extern "C"` functions will handle the host calls, and I'll create a `run_demo()` function demonstrating both WASI file system and HTTP interactions. I plan to include examples for GET and POST requests.
> 
> 
> **Designing Data Exchange**
> 
> I'm currently focused on handling the data transfer, specifically how the host and guest communicate via linear memory. I will include `malloc` and `free` functions for memory management, while using JSON and Serde for data serialization/deserialization. This will facilitate structured interaction between the host and the guest WASM modules.
> 
> 
> **Choosing Serialisation Strategy**
> 
> I'm now focusing on the method for serializing and deserializing request/response data. The plan is to utilize JSON for data exchange, and I'm leaning towards `serde_json` due to its user-friendliness. This makes the interaction between the guest and host seamless. I'm considering other options, like MessagePack, but the readability of JSON makes it preferable for demonstration.
> 
> 
> **Finalizing Host Functions**
> 
> I'm working on the host-side functions, particularly the `host_http_request`. I'll implement the logic to read the request from Wasm memory, use `reqwest` to make the actual HTTP call, and then serialize the response. Most crucially, I'll handle memory allocation within the guest's linear memory to return the response data, and will implement a memory allocator on the guest-side via `malloc` and `free`.
> 
> 
> **Implementing the Host**
> 
> I'm now focusing on implementing the host-side functionality, particularly the interaction with `wasmtime` and `reqwest`. I'll establish the WASI environment, set up the linker to include custom host functions for HTTP requests, and manage the communication with the guest Wasm module. The key challenge here will be seamlessly integrating the host's HTTP calls with the guest module.
> 
> 
> **Decomposing the Requirements**
> 
> I'm currently decomposing the initial request to provide a working example of Rust and Wasm interop. The user needs a complete HTTP client with file system access.  I will use `reqwest` for the host's HTTP client, WASI for filesystem interactions, and `wasmtime` to execute the Wasm code. I'll need to manage the complexities of data exchange.
> 
> 
Of course. Here is a comprehensive example that demonstrates how to expose a more feature-rich environment from a Rust host to a WebAssembly guest.

This example will provide the Wasm module with:

1.  **A Full HTTP Client:** Capable of making GET and POST requests, handling custom headers, and maintaining a persistent cookie store across multiple requests. We will use the powerful `reqwest` library on the host.
2.  **File System Access:** The ability to read and write files within a sandboxed directory, provided securely by the host using the WebAssembly System Interface (WASI).

To manage the complexity of passing data like HTTP requests and responses, we will serialize them into JSON. This is a robust and extensible pattern for host-guest communication.

### Solution Overview

*   **Host Application (Rust):**
    *   Uses `wasmtime` to run the Wasm module.
    *   Uses `wasmtime_wasi` to provide standardized file system access.
    *   Initializes a `reqwest::Client` with a cookie store enabled. This client will be shared by all host calls.
    *   Exposes a single host function, `host_http_request`, which takes a serialized `HttpRequest` struct (as JSON), executes it, and returns a serialized `HttpResponse`.

*   **Guest Module (Rust compiled to `wasm32-wasi`):**
    *   Uses `std::fs` to interact with the filesystem (WASI makes this possible).
    *   Defines `HttpRequest` and `HttpResponse` structs that mirror the host's.
    *   Imports the `host_http_request` function.
    *   Exports memory management functions (`malloc`, `free`) so the host can write response data back into the guest's memory.
    *   Exports a `run_demo` function for the host to call, which demonstrates both file I/O and making HTTP requests.

---

### Project Structure

We'll use a Cargo workspace to manage the two crates.

```
/rust-wasm-host-example
├── Cargo.toml
├── guest/
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs
└── host/
    ├── Cargo.toml
    └── src/
        └── main.rs
```

### 1. Workspace `Cargo.toml`

Create this file at the root of the project.

```toml
[workspace]
members = [
    "host",
    "guest",
]
```

### 2. The Guest Wasm Module (`guest/`)

#### `guest/Cargo.toml`

```toml
[package]
name = "guest"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"] # Critical for creating a Wasm module

[dependencies]
# For serializing/deserializing our request/response structs
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

#### `guest/src/lib.rs`

This code defines the data structures, imports the host function, exports memory utilities, and contains the main logic to be executed.

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::fs;

// Define the data structures for communication. These must match the host's definition.
#[derive(Serialize, Deserialize, Debug)]
struct HttpRequest {
    pub method: String,
    pub url: String,
    pub headers: HashMap<String, String>,
    pub body: Option<Vec<u8>>,
}

#[derive(Serialize, Deserialize, Debug)]
struct HttpResponse {
    pub status_code: u16,
    pub headers: HashMap<String, String>,
    pub body: Vec<u8>,
}

// Import the host function that we will call from Wasm.
#[link(wasm_import_module = "host")]
extern "C" {
    /// Takes a pointer and length of a serialized HttpRequest,
    /// returns a packed u64 with the pointer and length of the serialized HttpResponse.
    fn host_http_request(ptr: *const u8, len: usize) -> u64;
}

// Export memory allocation functions so the host can write data into our memory.
#[no_mangle]
pub extern "C" fn malloc(size: usize) -> *mut u8 {
    let layout = std::alloc::Layout::from_size_align(size, 1).unwrap();
    unsafe { std::alloc::alloc(layout) }
}

#[no_mangle]
pub extern "C" fn free(ptr: *mut u8, size: usize) {
    let layout = std::alloc::Layout::from_size_align(size, 1).unwrap();
    unsafe { std::alloc::dealloc(ptr, layout) };
}

/// A helper function to call the host, handling serialization and deserialization.
fn make_http_request(req: HttpRequest) -> Result<HttpResponse, String> {
    // 1. Serialize the request struct to a JSON string.
    let req_bytes = serde_json::to_vec(&req).map_err(|e| e.to_string())?;

    // 2. Call the imported host function.
    let packed_result = unsafe { host_http_request(req_bytes.as_ptr(), req_bytes.len()) };

    // 3. Unpack the pointer and length from the u64 return value.
    let ptr = (packed_result >> 32) as *mut u8;
    let len = (packed_result & 0xFFFFFFFF) as usize;

    // 4. Read the response bytes from memory.
    let resp_bytes = unsafe { std::slice::from_raw_parts(ptr, len) };

    // 5. Deserialize the bytes into our HttpResponse struct.
    let http_resp: HttpResponse = serde_json::from_slice(resp_bytes).map_err(|e| e.to_string())?;

    // 6. Free the memory that the host allocated for the response.
    unsafe { free(ptr, len) };

    Ok(http_resp)
}

/// The main entry point of our Wasm module, called by the host.
#[no_mangle]
pub extern "C" fn run_demo() {
    println!("[guest] Running demo...");

    // --- 1. File System Demo (via WASI) ---
    println!("\n[guest] --- File System Demo ---");
    let content = "Hello from Wasm! This file was written by a sandboxed WebAssembly module.";
    // The host will map a directory to `/`, so we can write here.
    fs::write("/demo.txt", content).expect("Failed to write file");
    println!("[guest] Wrote to /demo.txt");

    let read_content = fs::read_to_string("/demo.txt").expect("Failed to read file");
    println!("[guest] Read from /demo.txt: \"{}\"", read_content);

    // --- 2. HTTP Client Demo ---
    println!("\n[guest] --- HTTP Client Demo ---");

    // Example 1: GET request to set a cookie
    println!("\n[guest] Making GET request to httpbin.org to set a cookie...");
    let get_req = HttpRequest {
        method: "GET".to_string(),
        url: "https://httpbin.org/cookies/set/session/12345".to_string(),
        headers: HashMap::new(),
        body: None,
    };

    match make_http_request(get_req) {
        Ok(resp) => {
            println!("[guest] GET Response Status: {}", resp.status_code);
            // Note: The cookie is stored in the host's client, not visible here.
        }
        Err(e) => eprintln!("[guest] GET Error: {}", e),
    }

    // Example 2: GET request to check if the cookie is sent back
    println!("\n[guest] Making GET request to see if the cookie is returned...");
    let check_cookie_req = HttpRequest {
        method: "GET".to_string(),
        url: "https://httpbin.org/cookies".to_string(),
        headers: HashMap::new(),
        body: None,
    };

    match make_http_request(check_cookie_req) {
        Ok(resp) => {
            println!("[guest] Cookie Check Response Status: {}", resp.status_code);
            let body_str = String::from_utf8_lossy(&resp.body);
            println!("[guest] Cookie Check Response Body: {}", body_str);
        }
        Err(e) => eprintln!("[guest] Cookie Check Error: {}", e),
    }

    // Example 3: POST request with custom headers and a JSON body
    println!("\n[guest] Making POST request with a JSON body...");
    let mut post_headers = HashMap::new();
    post_headers.insert("X-Custom-Header".to_string(), "Wasm-is-cool".to_string());

    let post_body = serde_json::json!({
        "source": "WebAssembly",
        "value": 42
    });

    let post_req = HttpRequest {
        method: "POST".to_string(),
        url: "https://httpbin.org/post".to_string(),
        headers: post_headers,
        body: Some(post_body.to_string().into_bytes()),
    };

    match make_http_request(post_req) {
        Ok(resp) => {
            println!("[guest] POST Response Status: {}", resp.status_code);
            let body_str = String::from_utf8_lossy(&resp.body);
            println!("[guest] POST Response Body (first 300 chars): {:.300}", body_str);
        }
        Err(e) => eprintln!("[guest] POST Error: {}", e),
    }
}
```

**To compile the guest:**

```bash
# Add the wasm32-wasi target if you haven't already
rustup target add wasm32-wasi

# Navigate to the guest directory and build
cd guest
cargo build --target wasm32-wasi --release
cd ..
```

### 3. The Host Application (`host/`)

#### `host/Cargo.toml`

```toml
[package]
name = "host"
version = "0.1.0"
edition = "2021"

[dependencies]
# The Wasm runtime
wasmtime = "19.0.1"
# WASI support for wasmtime
wasmtime-wasi = "19.0.1"

# The HTTP client
reqwest = { version = "0.11", features = ["blocking", "cookies", "json"] }

# For easy error handling
anyhow = "1.0"

# For serialization, matching the guest
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

#### `host/src/main.rs`

This host program sets up the environment, defines the `host_http_request` function, and runs the Wasm module.

```rust
use anyhow::Result;
use wasmtime::*;
use wasmtime_wasi::{WasiCtx, WasiCtxBuilder};
use std::collections::HashMap;
use serde::{Deserialize, Serialize};

// These structs must match the guest's definition.
#[derive(Serialize, Deserialize, Debug)]
struct HttpRequest {
    pub method: String,
    pub url: String,
    pub headers: HashMap<String, String>,
    pub body: Option<Vec<u8>>,
}

#[derive(Serialize, Deserialize, Debug)]
struct HttpResponse {
    pub status_code: u16,
    pub headers: HashMap<String, String>,
    pub body: Vec<u8>,
}

// Host state holds the WASI context and the HTTP client.
struct HostState {
    wasi: WasiCtx,
    // The reqwest client is stored in the host state to share the cookie
    // store and connection pool between guest calls.
    http_client: reqwest::blocking::Client,
}

fn main() -> Result<()> {
    // --- 1. Engine and Store Setup ---
    let engine = Engine::default();

    // --- 2. WASI and Host State Setup ---
    // Create a directory for the guest to access.
    let dir = tempfile::Builder::new().prefix("wasm_fs_").tempdir()?;
    println!("[host] Granting Wasm access to temporary directory: {:?}", dir.path());

    let wasi = WasiCtxBuilder::new()
        .inherit_stdio()
        .preopened_dir(dir.path(), "/")? // Grant access to `dir` and map it to `/` in the guest
        .build();

    let host_state = HostState {
        wasi,
        http_client: reqwest::blocking::Client::builder()
            .cookie_store(true) // Enable the cookie store
            .build()?,
    };
    let mut store = Store::new(&engine, host_state);

    // --- 3. Linker and Host Function Setup ---
    let mut linker = Linker::new(&engine);
    // Add all WASI functions to the linker
    wasmtime_wasi::add_to_linker(&mut linker, |state: &mut HostState| &mut state.wasi)?;

    // Add our custom HTTP function to the linker
    linker.func_wrap(
        "host",
        "host_http_request",
        |mut caller: Caller<'_, HostState>, ptr: i32, len: i32| -> u64 {
            // --- Reading from Wasm memory ---
            let mem = caller.get_export("memory").unwrap().into_memory().unwrap();
            let req_bytes = &mem.data(&caller)[ptr as usize..(ptr + len) as usize];
            let http_req: HttpRequest = serde_json::from_slice(req_bytes).unwrap();

            println!("[host] Received HTTP request from Wasm: {:?}", http_req.method);

            // --- Performing the HTTP request ---
            let client = &caller.data().http_client;
            let mut req_builder = match http_req.method.as_str() {
                "GET" => client.get(&http_req.url),
                "POST" => client.post(&http_req.url),
                _ => panic!("Unsupported method"), // In a real app, return an error
            };

            for (key, val) in http_req.headers {
                req_builder = req_builder.header(&key, &val);
            }
            if let Some(body) = http_req.body {
                req_builder = req_builder.body(body);
            }

            let resp = req_builder.send().unwrap();

            // --- Building the response ---
            let status = resp.status().as_u16();
            let mut headers = HashMap::new();
            for (key, val) in resp.headers().iter() {
                headers.insert(key.to_string(), val.to_str().unwrap_or("").to_string());
            }
            let body = resp.bytes().unwrap().to_vec();

            let http_resp = HttpResponse {
                status_code: status,
                headers,
                body,
            };
            let resp_bytes = serde_json::to_vec(&http_resp).unwrap();

            // --- Writing the response back to Wasm memory ---
            // 1. Get the `malloc` and `memory` exports from the guest.
            let malloc = caller.get_export("malloc").unwrap().into_func().unwrap();
            let memory = caller.get_export("memory").unwrap().into_memory().unwrap();

            // 2. Allocate memory in the guest for the response.
            let resp_len = resp_bytes.len() as i32;
            let mut results = [Val::I32(0)];
            malloc.call(&mut caller, &[Val::I32(resp_len)], &mut results).unwrap();
            let resp_ptr = results[0].unwrap_i32();

            // 3. Write the response bytes into the allocated memory.
            memory.write(&mut caller, resp_ptr as usize, &resp_bytes).unwrap();

            // 4. Pack the pointer and length into a u64 and return it.
            ((resp_ptr as u64) << 32) | (resp_len as u64)
        },
    )?;

    // --- 4. Module Instantiation and Execution ---
    let module = Module::from_file(&engine, "../guest/target/wasm32-wasi/release/guest.wasm")?;
    let instance = linker.instantiate(&mut store, &module)?;

    // Get the `run_demo` function and call it.
    let run_demo = instance.get_typed_func::<(), ()>(&mut store, "run_demo")?;
    println!("[host] Calling `run_demo` in Wasm module...\n");
    run_demo.call(&mut store, ())?;
    println!("\n[host] Wasm execution finished.");

    Ok(())
}
```

### 4. Running the Example

From the root directory (`rust-wasm-host-example`), run the host application:

```bash
# Ensure the guest is built first
cd guest
cargo build --target wasm32-wasi --release
cd ..

# Run the host
cargo run --manifest-path host/Cargo.toml
```

#### Expected Output

You will see output from both the host and the guest, showing that the guest successfully wrote and read a file, and then made a series of HTTP requests through the host's client, correctly demonstrating the persistent cookie store.

```
[host] Granting Wasm access to temporary directory: "/tmp/wasm_fs_xxxxxx"
[host] Calling `run_demo` in Wasm module...

[guest] Running demo...

[guest] --- File System Demo ---
[guest] Wrote to /demo.txt
[guest] Read from /demo.txt: "Hello from Wasm! This file was written by a sandboxed WebAssembly module."

[guest] --- HTTP Client Demo ---

[guest] Making GET request to httpbin.org to set a cookie...
[host] Received HTTP request from Wasm: "GET"
[guest] GET Response Status: 200

[guest] Making GET request to see if the cookie is returned...
[host] Received HTTP request from Wasm: "GET"
[guest] Cookie Check Response Status: 200
[guest] Cookie Check Response Body: {
  "cookies": {
    "session": "12345"
  }
}

[guest] Making POST request with a JSON body...
[host] Received HTTP request from Wasm: "POST"
[guest] POST Response Status: 200
[guest] POST Response Body (first 300 chars): {
  "args": {},
  "data": "{\"source\":\"WebAssembly\",\"value\":42}",
  "files": {},
  "form": {},
  "headers": {
    "Accept": "*/*",
    "Content-Length": "33",
    "Host": "httpbin.org",
    "User-Agent": "reqwest/0.11.27",
    "X-Amzn-Trace-Id": "Root=...",
    "X-Custom-Header": "Wasm-is-cool"
  },
...

[host] Wasm execution finished.
```
