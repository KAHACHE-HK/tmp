*Thinking...*

> **Evaluating WASI Capabilities**
> 
> I'm assessing the implications of targeting WASI. File access is restricted without runtime permission, so `include_str!` remains key for self-contained templates. Command-line arguments and stdout/stderr are fine. Networking presents the biggest challenge; WASI lacks a standard API.
> 
> 
> **Reassessing Network Requirements**
> 
> I'm revisiting the networking challenge in WASI. Given the SOAP focus, `reqwest` is crucial. I'll assume a runtime with networking extensions for now, making HTTP calls within the WASM module. Otherwise, I'll need to offload those calls to the host, which adds complexity I'd prefer to avoid.
> 
> 
> 
> **Confirming WASI Assumptions**
> 
> I'm re-evaluating the WASI constraints. I'll use `include_str!` and `clap` as planned. I'll lean on `reqwest` (blocking), assuming a runtime with networking support, and clarify its dependencies. Standard output is fine. I'll drop the use of `current_exe()`. The blocking approach seems best to align with the constraints.
> 
> 
> 
> 
> 
> **Confirming and Adapting**
> 
> I've re-evaluated the constraints. I will stick with `clap` and `include_str!` for self-containment. The networking aspect is still tricky; I'll assume the WASI runtime can handle `reqwest`, using async, perhaps with a WASI feature. I need to adjust the code to use embedded strings instead of filesystem paths for the templates.
> 
> 
> 
> 
> **Simplifying and Re-evaluating**
> 
> I'm simplifying the setup. I'll keep `clap` and embed resources with `include_str!`, and modify `RequestConfig`. Networking with `reqwest` (async) is retained, but I'll clearly document the WASI runtime's networking and async requirements, including `reqwest::blocking::Client` as a simpler WASI option.
> 
>  
>  
> 
> 
> **Streamlining for WASI Simplicity**
> 
> I've revised the WASI strategy, opting for `reqwest::blocking::Client` to sidestep async complexities. This simplifies the setup, removing the need for a separate async runtime within the WASM. I'll continue using `clap` and embedded resources, making this a cleaner WASI target.
> 
>  
> 
> 
> 
Okay, targeting WASI (WebAssembly System Interface) without a browser means we can leverage some system-like capabilities (like command-line arguments, environment variables, and potentially pre-opened file descriptors if the host provides them), but we'll aim for a self-contained WASM by embedding resources.

**Key changes for a WASI target:**

1.  **Resource Embedding:** We will use `include_str!` to embed all templates and expected XML responses directly into the WASM binary. This makes the `.wasm` file self-contained regarding these resources.
2.  **File System:** We will remove the logic for finding `templates_dir` and `expected_responses_dir` on the filesystem.
3.  **Command-Line Arguments (`clap`):** `clap` can still be used with WASI, as WASI runtimes can provide arguments to the WASM module.
4.  **Logging (`env_logger`):** This can also work, as WASI provides stdout/stderr.
5.  **Networking (`reqwest`):**
    *   For WASI, `reqwest` needs specific configuration. We'll use `reqwest::blocking::Client` to simplify things, as it avoids the need for an async runtime like Tokio within the WASI module, which can be complex to set up for WASI.
    *   The WASI runtime (e.g., Wasmtime, Wasmer) **must be configured to provide networking capabilities** (like `wasi-sockets`) for `reqwest` to function. This is a crucial host-side requirement.
6.  **No `tokio` runtime:** Since we're switching to `reqwest::blocking::Client`, we no longer need `#[tokio::main]` or `async/await` for the HTTP operations. The functions will become synchronous.
7.  **`std::process::exit`:** This is available and works as expected in WASI.

Here's the full source code adapted for WASI:

**`Cargo.toml` dependencies:**

Ensure your `Cargo.toml` looks something like this (versions might vary):

```toml
[package]
name = "soap_client_wasi"
version = "0.1.0"
edition = "2021"

[dependencies]
clap = { version = "4.4", features = ["derive"] }
# For WASI, reqwest needs specific features.
# Using the blocking client simplifies WASI by not needing an async runtime in the WASM.
reqwest = { version = "0.11", features = ["blocking", "cookies", "json", "rustls-tls-webpki-roots"], default-features = false }
minijinja = { version = "1.0", features = ["source"] } # "source" feature for loading from strings
serde = { version = "1.0", features = ["derive"] }
xmlunit-rs = "0.2"
thiserror = "1.0"
log = "0.4"
env_logger = "0.10"

# Note: When compiling for wasm32-wasi, the reqwest crate will attempt to use
# WASI-specific networking features (like wasi-sockets).
# The WASI runtime executing this module MUST provide these capabilities.
# rustls-tls-webpki-roots is used for TLS root certificates, which is often simpler for WASI.
```
*   `default-features = false` for `reqwest` is important to selectively enable what's needed.
*   `rustls-tls-webpki-roots` is a good choice for TLS in WASI as it embeds root certificates.
*   `minijinja` needs the `source` feature to load templates from strings effectively when using `extends`.

**`src/main.rs`:**

```rust
use clap::Parser;
use minijinja::{Environment, Source}; // Added Source
use reqwest::{blocking::Client, header, StatusCode}; // Using blocking Client
use serde::{Deserialize, Serialize};
use std::{process::exit, string::String as StdString}; // Renamed String to avoid conflict if any
use thiserror::Error;
use log::{debug, error, info, warn};

// --- Command Line Argument Parsing ---
#[derive(Parser, Debug)]
#[clap(author, version, about, long_about = None)]
struct Args {
    /// Username for SOAP WS-Security (optional)
    #[clap(long)]
    username: Option<StdString>,

    /// Password for SOAP WS-Security (optional)
    #[clap(long)]
    password: Option<StdString>,

    /// Full URL of the SOAP service
    #[clap(long)]
    soap_address: StdString,

    /// Namespace lookup string used in SOAP XML
    #[clap(long)]
    ns_lookup: StdString,

    // Paths for templates/expected_responses are removed as they will be embedded.
    // Conceptual requests_config_file is also removed for this WASI example
    // to focus on self-contained aspect.
}

// --- Custom Error Types ---
#[derive(Debug, Error)]
enum ScriptError {
    #[error("HTTP Request Error: {source}. URL: {url}")]
    ReqwestError {
        #[source]
        source: reqwest::Error,
        url: StdString,
    },
    // IoError is less likely with embedded resources, but kept for completeness if any other IO happens.
    #[error("I/O Error: {source}. Path: {path}")]
    IoError {
        #[source]
        source: std::io::Error,
        path: StdString,
    },
    #[error("Template Rendering Error: {source}. Template: {template_name}")]
    TemplateError {
        #[source]
        source: minijinja::Error,
        template_name: StdString,
    },
    #[error("XML Comparison Failed for request '{request_name}'. Differences found:\n{diff_details}")]
    XmlComparisonFailure {
        request_name: StdString,
        diff_details: StdString,
    },
    #[error("SOAP API returned non-200 status: {status_code} for URL: {url}. Response body:\n{body}")]
    SoapApiNon200 {
        status_code: StatusCode,
        url: StdString,
        body: StdString,
    },
    #[error("Configuration Error: {0}")]
    ConfigError(StdString), // For any other config issues
}

// --- Template Context Structure ---
#[derive(Serialize, Debug)]
struct TemplateContext<'a> {
    username: &'a Option<StdString>,
    password: &'a Option<StdString>,
    ns_lookup: &'a str,
    operation_name: &'a str,
    common_param_value: &'a str,
    request_id: usize,
}

// --- Request Configuration Structure (with embedded content) ---
#[derive(Serialize, Deserialize, Debug, Clone)]
struct RequestConfig {
    name: StdString,
    operation_name: StdString,
    common_param: StdString,
    // Filenames are now identifiers for templates added to Minijinja env
    template_name_id: &'static str, // e.g., "request1.xml.j2"
    // Content is embedded directly
    expected_response_content: &'static str,
}

// --- Embedded Resources ---
const BASE_TEMPLATE_CONTENT: &str = include_str!("../templates/base.xml.j2");
const REQUEST1_TEMPLATE_CONTENT: &str = include_str!("../templates/request1.xml.j2");
const REQUEST1_EXPECTED_RESPONSE: &str = include_str!("../expected_responses/expected_response1.xml");

const REQUEST2_TEMPLATE_CONTENT: &str = include_str!("../templates/request2.xml.j2");
const REQUEST2_EXPECTED_RESPONSE: &str = include_str!("../expected_responses/expected_response2.xml");

const REQUEST3_TEMPLATE_CONTENT: &str = include_str!("../templates/request3.xml.j2");
const REQUEST3_EXPECTED_RESPONSE: &str = include_str!("../expected_responses/expected_response3.xml");


fn main() {
    env_logger::Builder::from_env(env_logger::Env::default().default_filter_or("info")).init();

    let args = Args::parse();
    info!("WASI Script starting with arguments: {:?}", args);

    // The run function is now synchronous
    match run_soap_operations(&args) {
        Ok(()) => {
            info!("All SOAP requests and comparisons completed successfully.");
            exit(0);
        }
        Err(ScriptError::SoapApiNon200 { status_code, url, body }) => {
            error!(
                "SOAP API returned a non-200 status code: {} for URL: {}",
                status_code, url
            );
            error!("Response body:\n{}", body);
            info!("Exiting with status 0 as per requirement for non-200 responses.");
            exit(0);
        }
        Err(e) => {
            error!("--------------------------------------------------");
            error!("Critical Script Error Occurred:");
            error!("Error Type: {}", e);
            error!("--------------------------------------------------");
            // Log arguments and other context as before
            exit(1);
        }
    }
}

// This function is no longer async
fn run_soap_operations(args: &Args) -> Result<(), ScriptError> {
    info!("Attempting to create reqwest blocking client for WASI.");
    let client = Client::builder()
        .cookie_store(true) // Cookie jar should work with the blocking client
        .timeout(std::time::Duration::from_secs(30))
        .build()
        .map_err(|e| {
            error!("Failed to build reqwest client: {}", e);
            ScriptError::ReqwestError {
                source: e,
                url: args.soap_address.clone(),
            }
        })?;
    info!("Reqwest blocking client created.");


    let mut template_env = Environment::new();
    let mut source = Source::new();
    // Add all embedded templates to the source
    source.add_template("base.xml.j2", BASE_TEMPLATE_CONTENT)
        .map_err(|e| ScriptError::TemplateError { source: e, template_name: "base.xml.j2".into()})?;
    source.add_template("request1.xml.j2", REQUEST1_TEMPLATE_CONTENT)
        .map_err(|e| ScriptError::TemplateError { source: e, template_name: "request1.xml.j2".into()})?;
    source.add_template("request2.xml.j2", REQUEST2_TEMPLATE_CONTENT)
        .map_err(|e| ScriptError::TemplateError { source: e, template_name: "request2.xml.j2".into()})?;
    source.add_template("request3.xml.j2", REQUEST3_TEMPLATE_CONTENT)
        .map_err(|e| ScriptError::TemplateError { source: e, template_name: "request3.xml.j2".into()})?;
    template_env.set_source(source);
    info!("Minijinja environment configured with embedded templates.");


    let request_configs = vec![
        RequestConfig {
            name: "Request1".to_string(),
            operation_name: "Operation1".to_string(),
            common_param: "shared_value_for_req1".to_string(),
            template_name_id: "request1.xml.j2", // This is the key for get_template
            expected_response_content: REQUEST1_EXPECTED_RESPONSE,
        },
        RequestConfig {
            name: "Request2".to_string(),
            operation_name: "Operation2".to_string(),
            common_param: "shared_value_for_req2".to_string(),
            template_name_id: "request2.xml.j2",
            expected_response_content: REQUEST2_EXPECTED_RESPONSE,
        },
        RequestConfig {
            name: "Request3".to_string(),
            operation_name: "Operation3".to_string(),
            common_param: "shared_value_for_req3".to_string(),
            template_name_id: "request3.xml.j2",
            expected_response_content: REQUEST3_EXPECTED_RESPONSE,
        },
    ];

    for (idx, config) in request_configs.iter().enumerate() {
        let request_idx = idx + 1;
        info!("\n--- Processing {} (ID: {}) ---", config.name, request_idx);

        // Call the synchronous version
        process_single_request(&client, &template_env, args, config, request_idx)?;
    }

    Ok(())
}

// This function is no longer async
fn process_single_request(
    client: &Client, // reqwest::blocking::Client
    template_env: &Environment<'_>,
    cli_args: &Args,
    req_config: &RequestConfig,
    request_id_for_template: usize,
) -> Result<(), ScriptError> {
    // --- 1. Render SOAP XML from Template ---
    let tmpl = template_env
        .get_template(req_config.template_name_id)
        .map_err(|e| ScriptError::TemplateError {
            source: e,
            template_name: req_config.template_name_id.to_string(),
        })?;

    let context = TemplateContext {
        username: &cli_args.username,
        password: &cli_args.password,
        ns_lookup: &cli_args.ns_lookup,
        operation_name: &req_config.operation_name,
        common_param_value: &req_config.common_param,
        request_id: request_id_for_template,
    };

    let soap_xml_payload = tmpl.render(context).map_err(|e| ScriptError::TemplateError {
        source: e,
        template_name: req_config.template_name_id.to_string(),
    })?;
    debug!("Rendered SOAP XML for {}:\n{}", req_config.name, soap_xml_payload);

    // --- 2. Make SOAP HTTP Request (synchronous) ---
    let soap_action = format!("{}/{}", cli_args.ns_lookup, req_config.operation_name);
    debug!("SOAPAction: {}", soap_action);
    info!("Sending HTTP request to: {} for action: {}", cli_args.soap_address, soap_action);


    let response = client
        .post(&cli_args.soap_address)
        .header(header::CONTENT_TYPE, "text/xml; charset=utf-8")
        .header("SOAPAction", &soap_action)
        .body(soap_xml_payload.clone()) // Clone for potential logging
        .send() // This is a blocking call
        .map_err(|e| {
            error!("HTTP send error for {}: {}", req_config.name, e);
            ScriptError::ReqwestError {
                source: e,
                url: cli_args.soap_address.clone(),
            }
        })?;

    let status = response.status();
    let response_url = response.url().to_string(); // Still useful for logging
    info!("Received response for {} with status: {}", req_config.name, status);

    // Get response text (blocking)
    let response_text = response.text().map_err(|e| {
        error!("Failed to get response text for {}: {}", req_config.name, e);
        ScriptError::ReqwestError {
            source: e,
            url: response_url.clone(),
        }
    })?;


    if !status.is_success() {
        warn!("Non-200 status for {}: {}. Body: {}", req_config.name, status, response_text);
        return Err(ScriptError::SoapApiNon200 {
            status_code: status,
            url: response_url,
            body: response_text,
        });
    }
    debug!("Received Response XML for {}:\n{}", req_config.name, response_text);

    // --- 3. Compare Response XML with Embedded Expected XML ---
    let expected_xml = req_config.expected_response_content;

    let comparison = xmlunit_rs::compare(&response_text, expected_xml)
        .ignore_whitespace(true)
        .ignore_comments(true)
        .normalize_whitespace(true)
        .are_similar();

    if !comparison {
        let diff = xmlunit_rs::Diff::new(expected_xml, &response_text)
            .ignore_whitespace(true)
            .ignore_comments(true)
            .normalize_whitespace(true);
        let diff_description = diff.to_string();
        warn!("XML Comparison Failed for {}. Differences:\n{}", req_config.name, diff_description);
        return Err(ScriptError::XmlComparisonFailure {
            request_name: req_config.name.clone(),
            diff_details: diff_description,
        });
    }

    info!("XML Comparison for {} successful.", req_config.name);
    Ok(())
}
```

**To Compile for WASI:**

1.  **Install the WASI target:**
    ```bash
    rustup target add wasm32-wasi
    ```
2.  **Compile:**
    ```bash
    cargo build --target wasm32-wasi
    ```
    The output `.wasm` file will be in `target/wasm32-wasi/debug/your_crate_name.wasm` (or `release` if you add `--release`).

**To Run with a WASI Runtime (e.g., Wasmtime):**

1.  **Install Wasmtime:** (Follow instructions on [wasmtime.dev](https://wasmtime.dev/))
2.  **Run the WASM module:**
    ```bash
    wasmtime run \
      --env RUST_LOG=info \ # Set log level
      target/wasm32-wasi/debug/soap_client_wasi.wasm -- \ # Note the -- before app args
      --soap-address "YOUR_SOAP_URL" \
      --ns-lookup "YOUR_NAMESPACE" \
      # --username "user" \ (optional)
      # --password "pass" (optional)
    ```
    *   **Crucially, for networking to work, Wasmtime (or your chosen WASI runtime) must be run with networking permissions/features enabled.** For Wasmtime, this often means enabling experimental features like `wasi-sockets` or `wasi-http` when they become more stable and integrated. The exact flags for Wasmtime can change, so refer to its documentation. For example, you might need flags like `--tcplisten` or specific wasi common flags. As of early 2024, `wasi-sockets` is still somewhat experimental.
    *   If your SOAP service is on `localhost`, you might need specific flags like `--allow-all-network` (use with caution) or more granular permissions.

This version is now self-contained regarding its template and expected response files and uses a blocking HTTP client, which simplifies the execution model within a standard WASI environment. The main external dependency is that the WASI runtime itself must provide the necessary networking capabilities.
