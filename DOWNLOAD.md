*Thinking...*

> **Planning Rust Implementation**
> 
> I'm starting to map out the Rust program's structure. First, a `cargo new` project. Then, the crucial crates: `reqwest` for fetching the executable, `std::fs` for file operations, and `std::process::Command` to run it. Path manipulation with `std::path` and error handling using `anyhow` will be important.
> 
> **Planning the project**
> 
> Okay, starting the project with `cargo new`. Crates are next: `reqwest`, `std::fs`, `std::process::Command`, and `std::path`. The setup should define configuration constants or env vars for the URL and install location. Now, to the core logic!
> 
> **Planning the check**
> 
> Now I am structuring the core logic step by step. First, the `main` function and handling errors. Then, creating directories. Checking for the executable's existence, and downloading if it is missing. I also must ensure correct permissions and execute it.
> 
> **Refining Download Logic**
> 
> Considering edge cases now. I'll handle the download errors more robustly: network issues, HTTP errors, and writing file problems, all wrapped within the error handling. I am setting up error reporting via `stderr`.
> 
> **Refining structure**
> 
> I'm structuring the `main` function and how it handles errors. A helper function, returning a `Result<i32, Box<dyn std::error::Error>>`, will do the heavy lifting. `main` then handles the result and calls `std::process::exit`. I need a good flow to pass-through of the check's exit code and error reporting.
> 
> **Refining Logic Flow**
> 
> Alright, the `run_check` helper looks solid. It will return a `Result` with the exit code or an error. `main` will handle the error reporting and exit.  Next is clarifying the arguments and deciding on how the `EXECUTABLE_PATH` and `DOWNLOAD_URL` will be specified, which may influence configuration.
> 
> **Refining Code Structure**
> 
> I've solidified the plan for `run_check` to return a result with the intended Consul exit code, while the `main` function handles the error reporting and exit call. I also clarified where `EXECUTABLE_PATH` and `DOWNLOAD_URL` will originate, and decided that configuration should come later.
> 
Okay, let's create a Rust program that performs the same dynamic Consul check: fetching an executable if it's not locally present, ensuring it's executable, and then running it.

This Rust script will:
1.  Define constants for the executable name, installation directory, and download URL.
2.  Check if the executable exists in the specified path.
3.  If not, download it using the `reqwest` crate.
4.  Ensure the downloaded file has execute permissions (for Unix-like systems).
5.  Execute the application using `std::process::Command`.
6.  Pass through the exit code of the executed application to Consul, or exit with a specific code if the wrapper script itself fails.

**1. Project Setup**

If you don't have a Rust project, create one:
```bash
cargo new consul_dynamic_check_rust --bin
cd consul_dynamic_check_rust
```

**2. Add Dependencies to `Cargo.toml`**

You'll need `reqwest` for HTTP requests and `anyhow` for convenient error handling (optional, but recommended).

Open your `Cargo.toml` file and add the following under `[dependencies]`:
```toml
[dependencies]
reqwest = { version = "0.12", features = ["blocking"] } # Use 0.11 if 0.12 is not out or causes issues
anyhow = "1.0"
```
*   We use the `blocking` feature of `reqwest` because Consul checks are typically synchronous scripts.
*   `anyhow` simplifies error propagation.

**3. Rust Code (`src/main.rs`)**

```rust
use std::fs::{self, File, Permissions};
use std::io::{self, Write};
use std::os::unix::fs::PermissionsExt; // For Unix-specific permissions (chmod)
use std::path::{Path, PathBuf};
use std::process::{Command, exit};
use std::env;

use anyhow::{Context, Result}; // Using anyhow for cleaner error handling

// --- Configuration ---
const EXECUTABLE_NAME: &str = "my_actual_check_app"; // Replace with your executable's name
const INSTALL_DIR: &str = "/opt/consul/checks";     // Or any other suitable directory
const DOWNLOAD_URL: &str = "YOUR_DOWNLOAD_URL_HERE/my_actual_check_app"; // Replace with the actual URL

fn main() {
    // The main function now directly calls the logic and handles the exit code.
    // We print errors to stderr, and the exit code determines Consul's status.
    match run_consul_check_wrapper() {
        Ok(exit_code) => {
            // This is the exit code from the *actual* health check application
            exit(exit_code);
        }
        Err(e) => {
            eprintln!("Error in consul_dynamic_check_rust wrapper: {:?}", e);
            // If the wrapper itself fails (e.g., download, permissions),
            // exit with 2 (Critical for Consul).
            exit(2);
        }
    }
}

fn run_consul_check_wrapper() -> Result<i32> {
    let executable_path = PathBuf::from(INSTALL_DIR).join(EXECUTABLE_NAME);

    println!(
        "Consul Check Wrapper: Ensuring health check executable '{}' is available.",
        executable_path.display()
    );

    // --- 1. Ensure installation directory exists ---
    fs::create_dir_all(INSTALL_DIR)
        .with_context(|| format!("Failed to create installation directory: {}", INSTALL_DIR))?;
    println!("Consul Check Wrapper: Install directory {} ensured.", INSTALL_DIR);

    // --- 2. Check if executable exists, download if not ---
    if !executable_path.exists() {
        println!(
            "Consul Check Wrapper: Executable {} not found. Downloading from {}...",
            executable_path.display(),
            DOWNLOAD_URL
        );

        // Perform the download
        let client = reqwest::blocking::Client::builder()
            .user_agent("ConsulCheckWrapper-Rust/1.0")
            .build()
            .context("Failed to build reqwest client")?;

        let response = client.get(DOWNLOAD_URL).send()
            .with_context(|| format!("Failed to send request to {}", DOWNLOAD_URL))?;

        if !response.status().is_success() {
            let status = response.status();
            let body = response.text().unwrap_or_else(|_| String::from("<no body>"));
            anyhow::bail!(
                "Failed to download executable. Status: {}. URL: {}. Body: {}",
                status, DOWNLOAD_URL, body
            );
        }

        let mut file_content = response.bytes()
            .with_context(|| format!("Failed to read response bytes from {}", DOWNLOAD_URL))?;

        let mut dest_file = File::create(&executable_path)
            .with_context(|| format!("Failed to create file at {}", executable_path.display()))?;

        io::copy(&mut file_content.as_ref(), &mut dest_file)
            .with_context(|| format!("Failed to write downloaded content to {}", executable_path.display()))?;

        println!("Consul Check Wrapper: Download complete: {}", executable_path.display());

        // --- 3. Ensure executable has execute permissions (Unix-specific) ---
        println!(
            "Consul Check Wrapper: Setting execute permissions for {}",
            executable_path.display()
        );
        let metadata = fs::metadata(&executable_path)
            .with_context(|| format!("Failed to get metadata for {}", executable_path.display()))?;
        let mut permissions = metadata.permissions();

        // Set rwxr-xr-x (0755)
        permissions.set_mode(0o755);

        fs::set_permissions(&executable_path, permissions)
            .with_context(|| format!("Failed to set execute permissions for {}", executable_path.display()))?;
        println!("Consul Check Wrapper: Execute permissions set.");

    } else {
        println!("Consul Check Wrapper: Executable {} found.", executable_path.display());
        // Even if it exists, ensure it's executable (e.g. if permissions were changed manually)
        if let Ok(metadata) = fs::metadata(&executable_path) {
            if metadata.permissions().mode() & 0o111 == 0 { // Check if any execute bit is missing
                 println!(
                    "Consul Check Wrapper: Executable {} exists but is not executable. Setting permissions...",
                    executable_path.display()
                );
                let mut permissions = metadata.permissions();
                permissions.set_mode(0o755); // rwxr-xr-x
                fs::set_permissions(&executable_path, permissions)
                    .with_context(|| format!("Failed to set execute permissions for existing {}", executable_path.display()))?;
            }
        }
    }


    // --- 4. Execute the actual health check ---
    println!(
        "Consul Check Wrapper: Running Consul health check: {}",
        executable_path.display()
    );

    // Collect arguments passed to this wrapper, skip the wrapper's own name
    let args_to_forward: Vec<String> = env::args().skip(1).collect();

    let mut command = Command::new(&executable_path);
    if !args_to_forward.is_empty() {
        command.args(&args_to_forward);
        println!("Consul Check Wrapper: Forwarding arguments: {:?}", args_to_forward);
    }


    let status = command.status().with_context(|| {
        format!(
            "Failed to execute the health check application: {}",
            executable_path.display()
        )
    })?;

    // --- 5. Return the exit code for Consul ---
    // Consul interprets exit codes as:
    // 0: Passing
    // 1: Warning
    // Any other: Critical/Failing
    let exit_code = status.code().unwrap_or_else(|| {
        eprintln!(
            "Consul Check Wrapper: Health check process {} terminated by signal.",
            executable_path.display()
        );
        2 // Treat signal termination as critical
    });

    println!(
        "Consul Check Wrapper: Health check {} finished with exit code: {}",
        executable_path.display(), exit_code
    );
    Ok(exit_code)
}
```

**Explanation:**

1.  **Constants:** `EXECUTABLE_NAME`, `INSTALL_DIR`, and `DOWNLOAD_URL` are defined at the top. **Remember to replace `YOUR_DOWNLOAD_URL_HERE`**.
2.  **`main` and `run_consul_check_wrapper`:**
    *   `main` now simply calls `run_consul_check_wrapper` and then uses `std::process::exit()` with the result.
    *   `run_consul_check_wrapper` contains all the logic and returns a `Result<i32>`, where `i32` is the intended exit code for Consul. Errors are propagated using `anyhow`.
3.  **Path Construction:** `PathBuf::from(INSTALL_DIR).join(EXECUTABLE_NAME)` creates the full path to the executable in a platform-aware way.
4.  **Directory Creation:** `fs::create_dir_all(INSTALL_DIR)` creates the directory and any necessary parent directories. The `?` operator propagates errors.
5.  **Download Logic:**
    *   It checks if `executable_path.exists()`.
    *   If not, `reqwest::blocking::Client` is used to make a GET request.
    *   A `User-Agent` is set, which is good practice.
    *   `response.status().is_success()` checks for HTTP errors (2xx status codes).
    *   `response.bytes()` gets the response body as raw bytes.
    *   `File::create()` and `io::copy()` are used to write the downloaded content to the file.
    *   `.context(...)` from `anyhow` is used to add descriptive messages to errors.
6.  **Permissions (Unix):**
    *   `std::os::unix::fs::PermissionsExt` provides the `set_mode` method.
    *   `fs::metadata(&executable_path)?.permissions()` gets the current permissions.
    *   `permissions.set_mode(0o755)` sets the permissions to `rwxr-xr-x` (owner can read/write/execute, group and others can read/execute). This is a common setting for executables.
    *   The script also re-checks and sets permissions if the file exists but isn't executable.
7.  **Execution:**
    *   `env::args().skip(1).collect()` gathers any command-line arguments passed to this Rust wrapper script (excluding the script's own name) and forwards them to the actual health check application.
    *   `Command::new(&executable_path).args(&args_to_forward).status()?` executes the command.
    *   `status.code()` gives an `Option<i32>`.
        *   If `Some(code)`, that's the exit code.
        *   If `None` (e.g., process killed by a signal), we default to `2` (Critical).
8.  **Exit Code:** The script explicitly calls `std::process::exit()` with the determined code. This is crucial for Consul to correctly interpret the check's status.

**4. Build the Rust Program**

```bash
cargo build
# For a production/release build (recommended for deployment):
cargo build --release
```
The compiled binary will be in `target/debug/consul_dynamic_check_rust` or `target/release/consul_dynamic_check_rust`.

**5. How to Use with Consul**

1.  **Copy the Binary:** Copy the compiled Rust binary (e.g., `target/release/consul_dynamic_check_rust`) to a suitable location on your Consul nodes (e.g., `/usr/local/bin/`).
2.  **Ensure it's Executable:** The build process usually makes it executable, but you can double-check with `chmod +x /usr/local/bin/consul_dynamic_check_rust`.
3.  **Configure Consul Health Check:**
    This is the same as with the Bash/Python scripts. Define a "script" check in your Consul service definition or as a standalone agent check, pointing to your compiled Rust binary.

    Example Service Definition (JSON):
    ```json
    {
      "service": {
        "name": "my-monitored-app",
        "port": 80,
        "checks": [
          {
            "id": "my-app-rust-check",
            "name": "My Application Rust Dynamic Check",
            "args": ["/usr/local/bin/consul_dynamic_check_rust"],
            // You can pass arguments to your *actual* check app here,
            // they will be forwarded by the Rust wrapper:
            // "args": ["/usr/local/bin/consul_dynamic_check_rust", "--check-url", "http://localhost/health"],
            "interval": "15s",
            "timeout": "8s"
          }
        ]
      }
    }
    ```

4.  **Enable Script Checks in Consul Agent:**
    Ensure your Consul agent configuration allows script checks (e.g., `enable_script_checks = true` or `enable_local_script_checks = true` in the agent's configuration).

This Rust version provides a more robust and potentially more performant solution, especially if you have complex logic or prefer a compiled language for your operational tools. Remember to thoroughly test the download URL and the behavior of your actual health check application.
