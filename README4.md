*Thinking...*

> **Reviewing Rust Script Revisions**
> 
> I've addressed the user's requests for Rust script improvements. Now, I'm reviewing the code changes, making sure everything functions as expected and aligns with the initial design. It’s crucial I don't introduce any new bugs.
> 
> 
> **Expanding Script Capabilities**
> 
> I'm assessing next steps. The Rust script's core is solid, now it's time to increase flexibility. I'll focus on making configuration adjustable via command-line arguments and config files (TOML or YAML). I'll also investigate logging with log crates. Finally, I should improve `ScriptError` handling.
> 
> 
> 
> **Detailing Further Script Refinements**
> 
> I'm now evaluating advanced features. I should focus on improving configuration flexibility via command-line options and config file loading. Adding robust logging capabilities using crates is also vital. I also have to consider better error handling and SOAP version support. Finally, I must confirm the executables' pathing for file resolution.
> 
>  
> 
> 
> **Outlining Further Improvements**
> 
> I'm now exploring key areas for refinement. Config flexibility is key, focusing on command-line arguments and configuration file loading. I'll integrate robust logging, improve error handling, and address SOAP version compatibility. The executable's pathing also requires attention. Further consideration includes secure handling of sensitive data, such as passwords.
> 
> 
> 
> 
> 
> 
> **Defining Immediate Script Enhancements**
> 
> My focus is now on refining the Rust script for real-world use. I'm prioritizing configuration flexibility, particularly for template/expected paths and logging. Adding a configuration file loader will be helpful too. I also need to address the executable path for file resolution.
> 
>  
>  
> 
> 
Okay, building upon the previous refinements, here are a few more areas where the Rust script could be enhanced for even greater flexibility, robustness, and production-readiness:

**1.  Configuration for Paths (Templates & Expected Responses):**
    *   **Problem:** Currently, `"templates"` and `"expected_responses"` are assumed to be in the Current Working Directory (CWD). This might not always be convenient or reliable, especially if the script is called from different locations.
    *   **Solution:** Add command-line arguments to specify these paths. Alternatively, look for them relative to the executable's location.

2.  **Advanced Logging with `env_logger`:**
    *   **Problem:** `println!` and `eprintln!` are basic. A dedicated logging framework offers levels (DEBUG, INFO, WARN, ERROR), timestamps, module paths, and runtime configuration via environment variables.
    *   **Solution:** Integrate `log` and `env_logger`.

3.  **Loading Request Configurations from a File (Conceptual):**
    *   **Problem:** The `request_configs` array is hardcoded. For many requests or dynamic setups, this is inflexible.
    *   **Solution:** Define a structure for request configurations that can be deserialized from a file (e.g., TOML, JSON, YAML) using `serde`. (I'll show the structure and how it would integrate, but skip the full file parsing to keep the example focused).

4.  **Finding Paths Relative to the Executable:**
    *   **Problem:** If you want to bundle templates with your application, you need to find them relative to where the application binary is located, not the CWD.
    *   **Solution:** Use `std::env::current_exe()` and then navigate from there.

5.  **More Granular `SOAPAction` and Content-Type (Minor):**
    *   **Problem:** The script assumes SOAP 1.1 conventions.
    *   **Solution:** This could be made configurable per request if needed, but for now, the current approach is often sufficient if the target service is consistent.

Let's implement improvements #1, #2, and #4, and touch on #3 conceptually.

**Add to `Cargo.toml`:**

```toml
[dependencies]
# ... (clap, reqwest, minijinja, serde, xmlunit-rs, thiserror, tokio) ...
log = "0.4"
env_logger = "0.10" # Or check for the latest version

# No new dependencies for path args or conceptual config file loading beyond serde (already there)
```

**Refactored `src/main.rs`:**

```rust
use clap::Parser;
use minijinja::{path_loader, Environment};
use reqwest::{blocking::Client, header, StatusCode};
use serde::{Deserialize, Serialize}; // Added Deserialize for conceptual config loading
use std::{
    env, // For current_exe
    fs,
    path::{Path, PathBuf}, // Added PathBuf
    process::exit,
};
use thiserror::Error;
use log::{debug, error, info, warn}; // Logging macros

// --- Command Line Argument Parsing ---
#[derive(Parser, Debug)]
#[clap(author, version, about, long_about = None)]
struct Args {
    /// Username for SOAP WS-Security (optional)
    #[clap(long)]
    username: Option<String>,

    /// Password for SOAP WS-Security (optional)
    #[clap(long)]
    password: Option<String>,

    /// Full URL of the SOAP service
    #[clap(long)]
    soap_address: String,

    /// Namespace lookup string used in SOAP XML
    #[clap(long)]
    ns_lookup: String,

    /// Path to the directory containing Jinja templates.
    /// Defaults to "./templates" or "templates" dir alongside executable.
    #[clap(long)]
    templates_dir: Option<PathBuf>,

    /// Path to the directory containing expected XML response files.
    /// Defaults to "./expected_responses" or "expected_responses" dir alongside executable.
    #[clap(long)]
    expected_responses_dir: Option<PathBuf>,

    /// Path to a configuration file defining the sequence of SOAP requests (e.g., requests.toml).
    /// If provided, this will be used instead of the hardcoded request sequence. (Conceptual)
    #[clap(long)]
    requests_config_file: Option<PathBuf>,
}

// --- Custom Error Types ---
#[derive(Debug, Error)]
enum ScriptError {
    #[error("HTTP Request Error: {source}. URL: {url}")]
    ReqwestError {
        #[source]
        source: reqwest::Error,
        url: String,
    },
    #[error("I/O Error: {source}. File path: {path}")]
    IoError {
        #[source]
        source: std::io::Error,
        path: String, // Keep as String for display, PathBuf for operations
    },
    #[error("Template Rendering Error: {source}. Template: {template_name}")]
    TemplateError {
        #[source]
        source: minijinja::Error,
        template_name: String,
    },
    #[error("XML Comparison Failed for request '{request_name}'. Differences found:\n{diff_details}")]
    XmlComparisonFailure {
        request_name: String,
        diff_details: String,
    },
    #[error("SOAP API returned non-200 status: {status_code} for URL: {url}. Response body:\n{body}")]
    SoapApiNon200 {
        status_code: StatusCode,
        url: String,
        body: String,
    },
    #[error("Configuration Error: {0}")]
    ConfigError(String),
}

// --- Template Context Structure ---
#[derive(Serialize, Debug)]
struct TemplateContext<'a> {
    username: &'a Option<String>,
    password: &'a Option<String>,
    ns_lookup: &'a str,
    operation_name: &'a str,
    common_param_value: &'a str,
    request_id: usize,
}

// --- Request Configuration Structure ---
// Adding Deserialize for conceptual loading from a file (e.g., TOML)
#[derive(Serialize, Deserialize, Debug, Clone)] // Added Deserialize and Clone
struct RequestConfig {
    name: String, // Changed to String for ownership if deserialized
    operation_name: String,
    common_param: String,
    template_filename: String,
    expected_response_filename: String,
}

/// Tries to find a path relative to the executable or CWD.
/// Prefers path relative to executable if `dir_name` exists there.
fn resolve_resource_path(arg_path: Option<PathBuf>, dir_name: &str) -> Result<PathBuf, ScriptError> {
    if let Some(path) = arg_path {
        if path.is_dir() {
            return Ok(path);
        } else {
            return Err(ScriptError::ConfigError(format!(
                "Provided path for {} ('{}') does not exist or is not a directory.",
                dir_name,
                path.display()
            )));
        }
    }

    // Try relative to executable
    if let Ok(mut exe_path) = env::current_exe() {
        exe_path.pop(); // Remove executable name, keep parent dir
        let path_near_exe = exe_path.join(dir_name);
        if path_near_exe.is_dir() {
            debug!("Using {} directory found near executable: {}", dir_name, path_near_exe.display());
            return Ok(path_near_exe);
        }
    }

    // Fallback to CWD
    let cwd_path = PathBuf::from(dir_name);
    if cwd_path.is_dir() {
        debug!("Using {} directory found in CWD: {}", dir_name, cwd_path.display());
        return Ok(cwd_path);
    }

    Err(ScriptError::ConfigError(format!(
        "Could not find the '{}' directory. Please provide it via command line argument or ensure it exists near the executable or in the CWD.",
        dir_name
    )))
}


fn main() {
    // Initialize logger. RUST_LOG=info (or debug, trace, warn, error) can control verbosity.
    // Example: RUST_LOG=soap_client_script=debug cargo run -- ...
    env_logger::Builder::from_env(env_logger::Env::default().default_filter_or("info")).init();

    let args = Args::parse();
    info!("Script starting with arguments: {:?}", args);


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
            error!("Script Arguments Used:");
            error!("  SOAP Address: {}", args.soap_address);
            error!("  NS Lookup: {}", args.ns_lookup);
            if let Some(ref user) = args.username {
                error!("  Username: {}", user);
            } else {
                error!("  Username: Not provided");
            }
            error!("  Password: {}", if args.password.is_some() { "Provided (not shown)" } else { "Not provided" });
            if let Some(ref path) = args.templates_dir { error!("  Templates Dir: {}", path.display()); }
            if let Some(ref path) = args.expected_responses_dir { error!("  Expected Resp. Dir: {}", path.display()); }
            if let Some(ref path) = args.requests_config_file { error!("  Requests Config: {}", path.display()); }
            error!("--------------------------------------------------");

            match &e {
                ScriptError::ReqwestError { url, .. } => {
                    error!("Context: HTTP request to URL '{}' failed.", url);
                }
                ScriptError::IoError { path, .. } => {
                    error!("Context: File operation on path '{}' failed.", path);
                }
                ScriptError::TemplateError { template_name, .. } => {
                    error!("Context: Rendering template '{}' failed.", template_name);
                }
                ScriptError::XmlComparisonFailure { request_name, .. } => {
                     error!("Context: XML comparison for '{}' failed.", request_name);
                }
                ScriptError::ConfigError(msg) => {
                    error!("Configuration problem: {}", msg);
                }
                _ => {}
            }
            exit(1);
        }
    }
}

#[tokio::main]
async fn run_soap_operations(args: &Args) -> Result<(), ScriptError> {
    let client = Client::builder()
        .cookie_store(true)
        .timeout(std::time::Duration::from_secs(30))
        .build()
        .map_err(|e| ScriptError::ReqwestError {
            source: e,
            url: args.soap_address.clone(),
        })?;

    let templates_path = resolve_resource_path(args.templates_dir.clone(), "templates")?;
    let expected_responses_path = resolve_resource_path(args.expected_responses_dir.clone(), "expected_responses")?;

    info!("Using templates from: {}", templates_path.display());
    info!("Using expected responses from: {}", expected_responses_path.display());

    let mut template_env = Environment::new();
    template_env.set_loader(path_loader(&templates_path));

    // --- Load Request Configurations ---
    // Conceptually, if args.requests_config_file is Some, load from there.
    // For now, using the hardcoded version.
    let request_configs: Vec<RequestConfig> = if let Some(config_file_path) = &args.requests_config_file {
        info!("Loading request configurations from: {}", config_file_path.display());
        // Example: Load from TOML (would need `toml` crate and error handling)
        // let config_content = fs::read_to_string(config_file_path).map_err(|e| ScriptError::IoError { source: e, path: config_file_path.to_string_lossy().into_owned() })?;
        // toml::from_str(&config_content).map_err(|e| ScriptError::ConfigError(format!("Failed to parse requests config: {}", e)))?
        warn!("Request config file loading is conceptual. Using hardcoded requests for now.");
        get_hardcoded_request_configs() // Fallback or error if file loading is primary
    } else {
        info!("Using hardcoded request configurations.");
        get_hardcoded_request_configs()
    };


    for (idx, config) in request_configs.iter().enumerate() {
        let request_idx = idx + 1;
        info!("\n--- Processing {} (ID: {}) ---", config.name, request_idx);

        process_single_request(
            &client,
            &template_env,
            args,
            config, // Now owned String fields
            request_idx,
            &expected_responses_path, // Pass the resolved path
        )
        .await?;
    }

    Ok(())
}

fn get_hardcoded_request_configs() -> Vec<RequestConfig> {
    vec![
        RequestConfig {
            name: "Request1".to_string(),
            operation_name: "Operation1".to_string(),
            common_param: "shared_value_for_req1".to_string(),
            template_filename: "request1.xml.j2".to_string(),
            expected_response_filename: "expected_response1.xml".to_string(),
        },
        RequestConfig {
            name: "Request2".to_string(),
            operation_name: "Operation2".to_string(),
            common_param: "shared_value_for_req2".to_string(),
            template_filename: "request2.xml.j2".to_string(),
            expected_response_filename: "expected_response2.xml".to_string(),
        },
        RequestConfig {
            name: "Request3".to_string(),
            operation_name: "Operation3".to_string(),
            common_param: "shared_value_for_req3".to_string(),
            template_filename: "request3.xml.j2".to_string(),
            expected_response_filename: "expected_response3.xml".to_string(),
        },
    ]
}

async fn process_single_request(
    client: &Client,
    template_env: &Environment<'_>,
    cli_args: &Args,
    req_config: &RequestConfig, // Now takes owned RequestConfig
    request_id_for_template: usize,
    expected_responses_base_path: &Path, // New argument
) -> Result<(), ScriptError> {
    // --- 1. Render SOAP XML from Template ---
    let tmpl = template_env
        .get_template(&req_config.template_filename) // Use & on String
        .map_err(|e| ScriptError::TemplateError {
            source: e,
            template_name: req_config.template_filename.clone(),
        })?;

    let context = TemplateContext {
        username: &cli_args.username,
        password: &cli_args.password,
        ns_lookup: &cli_args.ns_lookup,
        operation_name: &req_config.operation_name, // Use & on String
        common_param_value: &req_config.common_param, // Use & on String
        request_id: request_id_for_template,
    };

    let soap_xml_payload = tmpl.render(context).map_err(|e| ScriptError::TemplateError {
        source: e,
        template_name: req_config.template_filename.clone(),
    })?;
    debug!("Rendered SOAP XML for {}:\n{}", req_config.name, soap_xml_payload);

    // --- 2. Make SOAP HTTP Request ---
    let soap_action = format!("{}/{}", cli_args.ns_lookup, req_config.operation_name);
    debug!("SOAPAction: {}", soap_action);

    let response = client
        .post(&cli_args.soap_address)
        .header(header::CONTENT_TYPE, "text/xml; charset=utf-8")
        .header("SOAPAction", &soap_action)
        .body(soap_xml_payload.clone())
        .send()
        .await
        .map_err(|e| ScriptError::ReqwestError {
            source: e,
            url: cli_args.soap_address.clone(),
        })?;

    let status = response.status();
    let response_url = response.url().to_string();
    let response_text = response.text().await.map_err(|e| ScriptError::ReqwestError {
        source: e,
        url: response_url.clone(),
    })?;

    info!("Response Status for {}: {}", req_config.name, status);
    if !status.is_success() {
        return Err(ScriptError::SoapApiNon200 {
            status_code: status,
            url: response_url,
            body: response_text,
        });
    }
    debug!("Received Response XML for {}:\n{}", req_config.name, response_text);

    // --- 3. Compare Response XML with Expected XML ---
    // Use the resolved base path
    let expected_response_file_path = expected_responses_base_path.join(&req_config.expected_response_filename);

    let expected_xml = fs::read_to_string(&expected_response_file_path).map_err(|e| { // Pass PathBuf by ref
        ScriptError::IoError {
            source: e,
            path: expected_response_file_path.to_string_lossy().into_owned(),
        }
    })?;

    let comparison = xmlunit_rs::compare(&response_text, &expected_xml)
        .ignore_whitespace(true)
        .ignore_comments(true)
        .normalize_whitespace(true)
        .are_similar();

    if !comparison {
        let diff = xmlunit_rs::Diff::new(&expected_xml, &response_text)
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

**Summary of Key Changes:**

1.  **`Args` Struct Updated:**
    *   Added `templates_dir: Option<PathBuf>` and `expected_responses_dir: Option<PathBuf>`.
    *   Added `requests_config_file: Option<PathBuf>` for conceptual config file loading.
2.  **`resolve_resource_path` Function:**
    *   New helper function to determine the actual path for templates/expected responses.
    *   It prioritizes the command-line argument, then checks relative to the executable, then falls back to the CWD.
    *   Returns `Result<PathBuf, ScriptError>`.
3.  **Logging Integration (`log`, `env_logger`):**
    *   `env_logger` is initialized in `main`.
    *   `println!` and `eprintln!` are replaced with `info!`, `debug!`, `warn!`, `error!`.
    *   You can control log level via `RUST_LOG` environment variable (e.g., `RUST_LOG=info` or `RUST_LOG=my_crate_name=debug`).
4.  **`RequestConfig` Changes:**
    *   Fields changed from `&'a str` to `String` to allow them to be owned, which is necessary if deserializing from a file. Added `#[derive(Clone)]`.
    *   A `get_hardcoded_request_configs()` function now provides the default list.
    *   Conceptual logic in `run_soap_operations` to load from `args.requests_config_file` if provided.
5.  **Path Usage:**
    *   `run_soap_operations` now calls `resolve_resource_path` to get the base paths for templates and expected responses.
    *   `process_single_request` now takes `expected_responses_base_path: &Path` and constructs the full path by joining it with `req_config.expected_response_filename`.
6.  **Error Handling:**
    *   `ScriptError::ConfigError` added for path resolution and conceptual config file issues.
    *   Error messages now use the `log` macros.

**To Use the Logging:**

When you run the script, you can set the `RUST_LOG` environment variable:

*   `RUST_LOG=info ./target/debug/soap_client_script --soap-address ...` (Default level set in code)
*   `RUST_LOG=debug ./target/debug/soap_client_script --soap-address ...` (More verbose)
*   `RUST_LOG=soap_client_script=trace ./target/debug/soap_client_script ...` (Very verbose, specific to your crate)

These changes make the script more configurable in terms of file locations and provide better, more controllable logging, which is invaluable for debugging and monitoring. The conceptual addition for loading request configurations also points towards how you'd make the sequence of operations entirely data-driven.
