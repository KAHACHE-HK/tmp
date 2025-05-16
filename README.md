*Thinking...*

> **Planning Rust Integration**
> 
> I need to develop a Rust script to seamlessly interface with a SOAP API. First, I'll map out the API endpoints and data structures. Then, I'll explore the best crates for handling SOAP interactions and XML parsing in Rust. This will be a crucial foundation.
> 
> 
> **Mapping, Planning, and Implementation**
> 
> I've now defined the core requirements: parsing command-line arguments with `clap`, managing cookie sessions using `reqwest`, crafting and templating SOAP requests. The final piece is choosing an appropriate templating engine for XML, and it looks like `minijinja` is the frontrunner, with a close eye on error handling.
> 
> <br>
> <br>
> 
> 
> **Structuring the Implementation**
> 
> I've solidified the architecture: command-line arguments with `clap`, a cookie-aware `reqwest` setup, and `minijinja` for XML templating. Now, I'll integrate the XML comparison using `xmlunit-rs` and meticulously refine the error handling, including the unusual exit code for non-200 responses. The goal is robust and accurate functionality.
> 
> <br>
> <br>
> 
> 
> **Refining Implementation Strategy**
> 
> I've now refined the command-line argument definitions with `clap`, ensuring proper help text is generated. I have also confirmed which crates to use, and have a clear structure and a plan. The cookie jar will be handled by `reqwest`. I will be careful about the unusual exit condition for non-200 responses.
> <br>
> <br>
> 
> 
> **Defining Command-Line Interface**
> 
> I'm currently creating the command-line interface with `clap`. Defining arguments like username, password, SOAP address, and the namespace lookup, along with implementing a help flag, is my current task. I need to ensure the help text is informative and easy to use.
> 
> <br>
> <br>
> 
> 
> **Outlining Implementation Steps**
> 
> I've reviewed all requirements and defined the major steps, libraries, and architecture, including how to handle the non-standard exit codes. I have a clear understanding of the template and directory structures and a concrete plan.
> 
> <br>
> <br>
> 
> 
> **Detailing the SOAP Request**
> 
> I've outlined the SOAP request, emphasizing the mandatory `soap_address` and `ns_lookup`.  I'm also detailing the need for a cookie jar, considering HTTP response codes (non-200 status), and, using `xmlunit-rs`, a robust XML comparison, with error handling and the unusual exit code. I have created an error type using `thiserror`.
> 
> <br>
> <br>
> 
> 
> **Reviewing the Implementation**
> 
> I've now fully detailed the structure, main steps, and error handling using a custom `AppError` type for context. The `main` function will orchestrate, and `run` will manage the requests. The implementation will use `clap`, `reqwest`, `minijinja`, and `xmlunit-rs` for a robust and reliable result.
> 
> <br>
> <br>
> 
> 
> **Analyzing the Implementation Steps**
> 
> I've reviewed all requirements and created a detailed plan. It includes detailed libraries and architecture, with a well defined `main` and `run` functions. Now, I'm focused on refining XML comparison and handling non-200 responses properly.
> 
> <br>
> <br>
> 
> 
> **Revisiting Implementation Steps**
> 
> I've thoroughly reviewed the entire design, including error handling, and refined the `AppError` enum to encompass sufficient context for debugging. The next step is implementing the `run` and `main` functions.
> 
> <br>
> <br>
> 
> 
> **Re-evaluating the Plan**
> 
> I'm rethinking the structure: main function with argument parsing and run function for the SOAP interactions. The error handling will be structured with custom error types using `thiserror`, which allows better error context. Also, I'll refine the plan further for robustness.
> 
> <br>
> <br>
> 
> 
> **Reviewing Implementation Details**
> 
> I'm now reviewing all the design decisions, including error handling, SOAPAction header consideration, and the specific files required for the templates and comparison. The overall approach seems solid. I am confident about this implementation.
> 
> 
> **Analyzing the SOAP Request**
> 
> I'm now detailing the SOAP request with attention to `soap_address`, and `ns_lookup`. A cookie jar is crucial, so I'll manage HTTP response codes and, with `xmlunit-rs`, conduct robust XML comparisons. Proper error handling using custom `AppError`s is also essential.
> <br>
> <br>
> 
> 
> **Reviewing the Design**
> 
> I've reviewed all requirements, library choices, and error handling. With clear steps defined, now I'll test the planned structure, paying special attention to template composition and XML comparison with the goal of a robust implementation.
> <br>
> <br>
> 
> 
Okay, I will provide a Rust script that fulfills your requirements. This script will use `clap` for command-line argument parsing, `reqwest` for HTTP requests with cookie management, `minijinja` for XML templating with composition, and `xmlunit-rs` for robust XML comparison.

**Project Structure:**

It's recommended to organize your project as follows:

```
soap_client_project/
├── Cargo.toml
├── src/
│   └── main.rs
├── templates/
│   ├── base.xml.j2         # Base SOAP envelope template
│   ├── request1.xml.j2     # Template for the first SOAP request
│   ├── request2.xml.j2     # Template for the second SOAP request
│   └── request3.xml.j2     # Template for the third SOAP request
└── expected_responses/
    ├── expected_response1.xml # Expected XML for the first response
    ├── expected_response2.xml # Expected XML for the second response
    └── expected_response3.xml # Expected XML for the third response
```

**1. `Cargo.toml`:**

Add the following dependencies to your `Cargo.toml` file:

```toml
[package]
name = "soap_client_script"
version = "0.1.0"
edition = "2021"

[dependencies]
clap = { version = "4.4", features = ["derive"] }
reqwest = { version = "0.11", features = ["blocking", "cookies", "json", "rustls-tls"], default-features = false }
minijinja = { version = "1.0", features = ["loader"] }
serde = { version = "1.0", features = ["derive"] } # For minijinja context if complex data is used
xmlunit-rs = "0.2"
thiserror = "1.0"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] } # reqwest::blocking might still need a runtime
```
*Note: `rustls-tls` is chosen for broader compatibility. You can use `native-tls` if preferred.*
*`tokio` is included because `reqwest::blocking` still relies on a Tokio runtime internally.*

**2. Example Template Files:**

Create the `templates` directory and add the following files:

**`templates/base.xml.j2`:**
This template defines the basic SOAP envelope structure and includes a block for WS-Security if username/password are provided, and a block for the body content.

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:ns="{{ ns_lookup }}">
   <soapenv:Header>
      {% if username and password %}
      <wsse:Security xmlns:wsse="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd" soapenv:mustUnderstand="1">
         <wsse:UsernameToken wsu:Id="UsernameToken-{{ request_id }}" xmlns:wsu="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd">
            <wsse:Username>{{ username }}</wsse:Username>
            <wsse:Password Type="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-username-token-profile-1.0#PasswordText">{{ password }}</wsse:Password>
         </wsse:UsernameToken>
      </wsse:Security>
      {% endif %}
   </soapenv:Header>
   <soapenv:Body>
      {% block body_content %}
      <!-- Default body content if not overridden -->
      {% endblock %}
   </soapenv:Body>
</soapenv:Envelope>
```

**`templates/request1.xml.j2`:**
This extends the base template and provides specific content for the first request.

```xml
{% extends "base.xml.j2" %}

{% block body_content %}
   <ns:{{ operation_name }}>
      <ns:Parameter1>ValueForRequest1</ns:Parameter1>
      <ns:CommonParameter>{{ common_param_value }}</ns:CommonParameter>
   </ns:{{ operation_name }}>
{% endblock %}
```

**`templates/request2.xml.j2`:**
Similar to `request1.xml.j2`, but for the second request.

```xml
{% extends "base.xml.j2" %}

{% block body_content %}
   <ns:{{ operation_name }}>
      <ns:ParameterX>AnotherValueForRequest2</ns:ParameterX>
      <ns:CommonParameter>{{ common_param_value }}</ns:CommonParameter>
   </ns:{{ operation_name }}>
{% endblock %}
```

**`templates/request3.xml.j2`:**
And for the third request.

```xml
{% extends "base.xml.j2" %}

{% block body_content %}
   <ns:{{ operation_name }}>
      <ns:InputData>DataForRequest3</ns:InputData>
      <ns:CommonParameter>{{ common_param_value }}</ns:CommonParameter>
      <ns:FinalFlag>true</ns:FinalFlag>
   </ns:{{ operation_name }}>
{% endblock %}
```

**3. Example Expected Response Files:**

Create the `expected_responses` directory. You'll need to populate these with the actual XML you expect back from your SOAP service for each request. For demonstration, here are placeholders:

**`expected_responses/expected_response1.xml`:**

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:ns="YOUR_NS_LOOKUP_VALUE">
   <soapenv:Body>
      <ns:Operation1Response>
         <ns:Result>Success for request 1</ns:Result>
         <!-- Add more specific expected XML structure here -->
      </ns:Operation1Response>
   </soapenv:Body>
</soapenv:Envelope>
```
*(Remember to replace `YOUR_NS_LOOKUP_VALUE` with the actual namespace you use)*

**`expected_responses/expected_response2.xml`:**

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:ns="YOUR_NS_LOOKUP_VALUE">
   <soapenv:Body>
      <ns:Operation2Response>
         <ns:Status>Completed for request 2</ns:Status>
      </ns:Operation2Response>
   </soapenv:Body>
</soapenv:Envelope>
```

**`expected_responses/expected_response3.xml`:**

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:ns="YOUR_NS_LOOKUP_VALUE">
   <soapenv:Body>
      <ns:Operation3Response>
         <ns:Message>Final ack for request 3</ns:Message>
      </ns:Operation3Response>
   </soapenv:Body>
</soapenv:Envelope>
```

**4. `src/main.rs` (The Rust Script):**

```rust
use clap::Parser;
use minijinja::{path_loader, Environment};
use reqwest::{blocking::Client, header};
use serde::Serialize; // Only if using complex structs for context, basic types are fine
use std::{fs, path::Path, process::exit};
use thiserror::Error;

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

    /// Full URL of the SOAP service (e.g., http://localhost:8080/services/MyService)
    #[clap(long)]
    soap_address: String,

    /// Namespace lookup string used in SOAP XML (e.g., http://example.com/myservice)
    #[clap(long)]
    ns_lookup: String,
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
        path: String,
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
        // expected: String, // Could include full XMLs for very verbose logging
        // actual: String,
    },
    #[error("SOAP API returned non-200 status: {status_code} for URL: {url}. Response body:\n{body}")]
    SoapApiNon200 {
        status_code: reqwest::StatusCode,
        url: String,
        body: String,
    },
}

// --- Template Context Structure ---
// Using serde::Serialize allows passing complex structs to minijinja.
// For simple key-value, minijinja::context! macro is often enough.
#[derive(Serialize, Debug)]
struct TemplateContext<'a> {
    username: &'a Option<String>,
    password: &'a Option<String>,
    ns_lookup: &'a str,
    operation_name: &'a str,
    common_param_value: &'a str, // Example of a shared parameter
    request_id: usize, // For unique IDs in XML if needed
    // Add other request-specific parameters here
    // param_for_req1: Option<&'a str>,
    // param_for_req2: Option<&'a str>,
}

fn main() {
    let args = Args::parse();

    // The `run` function contains the core logic and returns a Result.
    // `main` handles the outcome, logging, and exit codes.
    match run_soap_operations(&args) {
        Ok(()) => {
            println!("All SOAP requests and comparisons completed successfully.");
            exit(0); // Successful completion of all operations
        }
        Err(ScriptError::SoapApiNon200 { status_code, url, body }) => {
            // Special case: non-200 HTTP response exits with 0 as per requirement
            eprintln!(
                "SOAP API returned a non-200 status code: {} for URL: {}",
                status_code, url
            );
            eprintln!("Response body:\n{}", body);
            eprintln!("Exiting with status 0 as per requirement for non-200 responses.");
            exit(0);
        }
        Err(e) => {
            // All other errors exit with 1
            eprintln!("--------------------------------------------------");
            eprintln!("Critical Script Error Occurred:");
            eprintln!("Error Type: {}", e);
            eprintln!("--------------------------------------------------");
            eprintln!("Script Arguments Used:");
            eprintln!("  SOAP Address: {}", args.soap_address);
            eprintln!("  NS Lookup: {}", args.ns_lookup);
            if let Some(ref user) = args.username {
                eprintln!("  Username: {}", user);
            } else {
                eprintln!("  Username: Not provided");
            }
            eprintln!("  Password: {}", if args.password.is_some() { "Provided (not shown)" } else { "Not provided" });
            eprintln!("--------------------------------------------------");

            // Log specific context based on error type if useful
            match e {
                ScriptError::ReqwestError { url, .. } => {
                    eprintln!("Context: HTTP request to URL '{}' failed.", url);
                }
                ScriptError::IoError { path, .. } => {
                    eprintln!("Context: File operation on path '{}' failed.", path);
                }
                ScriptError::TemplateError { template_name, .. } => {
                    eprintln!("Context: Rendering template '{}' failed.", template_name);
                }
                ScriptError::XmlComparisonFailure { request_name, .. } => {
                     eprintln!("Context: XML comparison for '{}' failed.", request_name);
                }
                _ => {} // Already handled by the general error message
            }
            exit(1);
        }
    }
}

#[tokio::main] // reqwest::blocking needs a tokio runtime
async fn run_soap_operations(args: &Args) -> Result<(), ScriptError> {
    // --- Initialize HTTP Client with Cookie Store ---
    let client = Client::builder()
        .cookie_store(true) // Enable cookie jar
        .timeout(std::time::Duration::from_secs(30)) // Set a timeout
        .build()
        .map_err(|e| ScriptError::ReqwestError {
            source: e,
            url: args.soap_address.clone(),
        })?;

    // --- Initialize Minijinja Environment ---
    // Assumes templates are in a "./templates" directory relative to executable
    let mut env = Environment::new();
    env.set_loader(path_loader("templates"));

    // --- Define request-specific details ---
    // In a real scenario, these might come from config or be more dynamic
    let request_configs = [
        ("request1", "Operation1", "shared_value_123"),
        ("request2", "Operation2", "shared_value_123"),
        ("request3", "Operation3", "shared_value_123"),
    ];

    for (idx, (request_name_base, operation_name, common_param)) in request_configs.iter().enumerate() {
        let request_idx = idx + 1;
        let current_request_name = format!("{}{}", request_name_base, ""); // No suffix like _call needed if base is unique

        println!("\n--- Processing Request {} ---", current_request_name);

        // --- 1. Render SOAP XML from Template ---
        let template_file_name = format!("{}.xml.j2", request_name_base);
        let tmpl = env.get_template(&template_file_name).map_err(|e| {
            ScriptError::TemplateError {
                source: e,
                template_name: template_file_name.clone(),
            }
        })?;

        let context = TemplateContext {
            username: &args.username,
            password: &args.password,
            ns_lookup: &args.ns_lookup,
            operation_name,
            common_param_value: common_param,
            request_id: request_idx,
            // Add more specific params if template needs them, e.g.
            // param_for_req1: if request_idx == 1 { Some("data1") } else { None },
        };

        let soap_xml_payload = tmpl.render(&context).map_err(|e| ScriptError::TemplateError {
            source: e,
            template_name: template_file_name.clone(),
        })?;
        println!("Rendered SOAP XML for {}:\n{}", current_request_name, soap_xml_payload);


        // --- 2. Make SOAP HTTP Request ---
        // Construct SOAPAction header (common in SOAP 1.1)
        // For SOAP 1.2, Content-Type might be application/soap+xml and action is a parameter of Content-Type
        // This example assumes SOAP 1.1 style or a server that infers action.
        // Adjust if your service requires a specific SOAPAction format.
        let soap_action = format!("{}/{}", args.ns_lookup, operation_name);
        println!("SOAPAction: {}", soap_action);

        let response = client
            .post(&args.soap_address)
            .header(header::CONTENT_TYPE, "text/xml; charset=utf-8") // Common for SOAP 1.1
            // .header(header::CONTENT_TYPE, "application/soap+xml; charset=utf-8") // Common for SOAP 1.2
            .header("SOAPAction", &soap_action) // SOAPAction for SOAP 1.1
            .body(soap_xml_payload.clone()) // Clone payload for potential logging on error
            .send()
            .map_err(|e| ScriptError::ReqwestError {
                source: e,
                url: args.soap_address.clone(),
            })?;

        let status = response.status();
        let response_url = response.url().to_string();
        let response_text = response.text().map_err(|e| ScriptError::ReqwestError {
            source: e,
            url: response_url.clone(),
        })?;

        println!("Response Status for {}: {}", current_request_name, status);
        if !status.is_success() {
            // This error will be caught by main and lead to exit(0)
            return Err(ScriptError::SoapApiNon200 {
                status_code: status,
                url: response_url,
                body: response_text,
            });
        }
        println!("Received Response XML for {}:\n{}", current_request_name, response_text);

        // --- 3. Compare Response XML with Expected XML ---
        let expected_response_file_path_str = format!("expected_responses/expected_{}.xml", request_name_base);
        let expected_response_file_path = Path::new(&expected_response_file_path_str);

        let expected_xml = fs::read_to_string(expected_response_file_path).map_err(|e| {
            ScriptError::IoError {
                source: e,
                path: expected_response_file_path_str.clone(),
            }
        })?;

        // Using xmlunit-rs for comparison
        // Configure the comparison: ignore whitespace, comments.
        // You can also normalize whitespace if needed.
        let comparison = xmlunit_rs::compare(&response_text, &expected_xml)
            .ignore_whitespace(true)
            .ignore_comments(true)
            .normalize_whitespace(true) // Normalizes text nodes before comparison
            .are_similar();

        if !comparison {
            // For more detailed diff, xmlunit-rs's assert_xml_eq! macro is good for tests,
            // but programmatically getting a diff string requires more work or using its diff object.
            // Here, we'll just indicate failure.
            // To get some diff details:
            let diff = xmlunit_rs::Diff::new(&expected_xml, &response_text)
                .ignore_whitespace(true)
                .ignore_comments(true)
                .normalize_whitespace(true);
            let diff_description = diff.to_string(); // Provides a description of the first difference

            return Err(ScriptError::XmlComparisonFailure {
                request_name: current_request_name.to_string(),
                diff_details: diff_description,
                // expected: expected_xml,
                // actual: response_text,
            });
        }

        println!("XML Comparison for {} successful.", current_request_name);
    }

    Ok(())
}

```

**Explanation:**

1.  **`Args` struct (clap):** Defines command-line arguments. `soap_address` and `ns_lookup` are mandatory by default. Optional `username` and `password`. Clap automatically generates `--help`.
2.  **`ScriptError` enum (thiserror):** Custom error types for different failure scenarios, capturing relevant context.
3.  **`TemplateContext` struct (serde):** Defines the data passed to Minijinja templates.
4.  **`main` function:**
    *   Parses arguments.
    *   Calls `run_soap_operations`.
    *   Handles the `Result` from `run_soap_operations`:
        *   `Ok(())`: Prints success and exits with `0`.
        *   `Err(ScriptError::SoapApiNon200)`: Logs details and exits with `0` (as per specific requirement).
        *   `Err(other_errors)`: Logs detailed error information (including script arguments and contextual info) to `stderr` and exits with `1`.
5.  **`run_soap_operations` function (async with `tokio::main`):**
    *   **HTTP Client:** Initializes `reqwest::blocking::Client` with `cookie_store(true)` to enable the cookie jar.
    *   **Minijinja Environment:** Sets up Minijinja to load templates from the `./templates/` directory.
    *   **Request Loop (3 times):**
        *   Determines template name, operation name, and expected response file for the current iteration.
        *   **Render XML:** Creates a `TemplateContext` and renders the appropriate `.xml.j2` template.
        *   **SOAP Request:**
            *   Sends a POST request to `soap_address`.
            *   Sets `Content-Type` (e.g., `text/xml` for SOAP 1.1 or `application/soap+xml` for SOAP 1.2).
            *   Sets a `SOAPAction` header, which is often `namespace/OperationName`.
            *   Sends the rendered XML as the body.
        *   **Response Handling:**
            *   Checks the HTTP status code. If not 200-299 (i.e., `!status.is_success()`), it returns `ScriptError::SoapApiNon200`. This will be handled by `main` to exit with `0`.
            *   Reads the response body as text.
        *   **XML Comparison:**
            *   Reads the corresponding expected XML file from `./expected_responses/`.
            *   Uses `xmlunit-rs::compare` with `ignore_whitespace(true)`, `ignore_comments(true)`, and `normalize_whitespace(true)` to compare the received XML with the expected XML.
            *   If they are not similar, it returns `ScriptError::XmlComparisonFailure` with details about the difference.
    *   If all requests and comparisons succeed, it returns `Ok(())`.
6.  **Error Logging:** The `main` function's error handling block is designed to print comprehensive error messages to `stderr`, including the error type, a general message, the script arguments used, and some context-specific information based on the error variant.
7.  **Template Composition:** The `.j2` templates use `{% extends "base.xml.j2" %}` and `{% block body_content %}` for composition, allowing shared SOAP envelope structure.

**To Compile and Run:**

1.  Save the Rust code as `src/main.rs`.
2.  Create the `templates` and `expected_responses` directories and populate them with your XML files as shown above.
3.  Ensure you have Rust installed.
4.  Navigate to your `soap_client_project` directory in the terminal.
5.  Build the project: `cargo build`
6.  Run the executable (e.g., on Linux/macOS):
    `./target/debug/soap_client_script --soap-address "YOUR_SOAP_URL" --ns-lookup "YOUR_NAMESPACE" --username "user" --password "pass"`
    Or on Windows:
    `.\target\debug\soap_client_script.exe --soap-address "YOUR_SOAP_URL" --ns-lookup "YOUR_NAMESPACE" --username "user" --password "pass"`

    To see help:
    `./target/debug/soap_client_script --help`

Remember to replace `"YOUR_SOAP_URL"`, `"YOUR_NAMESPACE"`, and the contents of `expected_responses/*.xml` with your actual service details and expected outputs. The example templates for requests also contain placeholders like `Operation1` and parameters that you'll need to adjust.
