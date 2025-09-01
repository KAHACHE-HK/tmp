use std::process::Command;
use std::string::String;

fn main() {
    println!("--- Running a command that should succeed ---");

    // We will use 'git --version' as our example command.
    // This works on Windows, macOS, and Linux if git is installed.
    let result = Command::new("git")
        .arg("--version")
        .output(); // This gets status, stdout, and stderr all at once.

    // The result is a Result<Output, Error>, so we handle potential spawn errors.
    match result {
        Ok(output) => {
            // Now we have the `Output` struct. We can check the status.
            println!("Process finished with status: {}", output.status);

            // Use an `if` to check if the command was successful.
            if output.status.success() {
                // `output.stdout` is a Vec<u8>, so we convert it to a string.
                // `String::from_utf8_lossy` is a safe way to do this, as it
                // replaces invalid UTF-8 sequences instead of panicking.
                let stdout_str = String::from_utf8_lossy(&output.stdout);
                
                println!("Command was successful!");
                println!("Stdout:\n{}", stdout_str);
            } else {
                // If the command failed, the reason is often in stderr.
                let stderr_str = String::from_utf8_lossy(&output.stderr);

                println!("Command failed!");
                println!("Stderr:\n{}", stderr_str);
            }
        }
        Err(e) => {
            // This error happens if the command could not even be started.
            // For example, if 'git' is not installed or not in the system's PATH.
            println!("Error executing command: {}", e);
        }
    }

    println!("\n--- Running a command that will fail ---");

    // Let's try a command that doesn't exist to see the error handling.
    let failing_result = Command::new("command_that_does_not_exist").output();

    if let Err(e) = failing_result {
        println!("Correctly failed to spawn the process: {}", e);
    }
}
