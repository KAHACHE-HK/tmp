Absolutely! Here’s a robust version with:

- **Error Handling**: Each Redis operation is wrapped and errors are logged.
- **Graceful Shutdown**:  
  - Listens for Ctrl+C (`tokio::signal::ctrl_c()`).  
  - On shutdown, both tasks are notified and exit cleanly.
- **Cancellation & Coordination**:  
  - Uses a broadcast channel to notify all tasks to shut down.  
  - Both the stream reader and the background task listen for shutdown.

---

## Complete Example

**`Cargo.toml`:**

```toml
[dependencies]
redis = { version = "0.25", features = ["tokio-comp"] }
tokio = { version = "1.37", features = ["full"] }
```

---

**`main.rs`:**

```rust
use redis::{aio::Connection, streams::StreamReadReply};
use std::collections::HashMap;
use tokio::{
    sync::broadcast,
    time::{sleep, Duration},
};

#[tokio::main]
async fn main() -> redis::RedisResult<()> {
    // Broadcast channel for shutdown notification
    let (shutdown_tx, _) = broadcast::channel(1);

    // Clone sender for ctrl_c handler
    let shutdown_tx_ctrlc = shutdown_tx.clone();

    // Spawn ctrl_c signal handler
    let ctrlc_handle = tokio::spawn(async move {
        if tokio::signal::ctrl_c().await.is_ok() {
            println!("\nReceived Ctrl+C, shutting down...");
            let _ = shutdown_tx_ctrlc.send(());
        }
    });

    // Start stream read task
    let shutdown_rx1 = shutdown_tx.subscribe();
    let stream_handle = tokio::spawn(stream_reader(shutdown_rx1));

    // Start another background async task
    let shutdown_rx2 = shutdown_tx.subscribe();
    let background_handle = tokio::spawn(other_task(shutdown_rx2));

    // Wait for shutdown signal and tasks to finish
    let _ = ctrlc_handle.await;
    let _ = stream_handle.await;
    let _ = background_handle.await;

    println!("Exited gracefully.");
    Ok(())
}

async fn stream_reader(
    mut shutdown: broadcast::Receiver<()>,
) -> redis::RedisResult<()> {
    let client = match redis::Client::open("redis://:mypassword@localhost:6379/") {
        Ok(c) => c,
        Err(e) => {
            eprintln!("Failed to create Redis client: {e}");
            return Err(e);
        }
    };
    let mut con = match client.get_async_connection().await {
        Ok(c) => c,
        Err(e) => {
            eprintln!("Failed to connect to Redis: {e}");
            return Err(e);
        }
    };

    let stream_keys = vec!["mystream1", "mystream2"];
    let mut last_ids: HashMap<String, String> = stream_keys
        .iter()
        .map(|k| (k.to_string(), "0-0".to_string()))
        .collect();

    loop {
        tokio::select! {
            _ = shutdown.recv() => {
                println!("Stream reader received shutdown signal.");
                break;
            }

            result = xread_streams(&mut con, &stream_keys, &last_ids) => {
                match result {
                    Ok(Some(reply)) => {
                        for stream in reply.keys {
                            let stream_name = &stream.key;
                            for entry in &stream.ids {
                                println!("Stream: {}, ID: {}", stream_name, entry.id);
                                for (k, v) in &entry.map {
                                    println!("  {}: {}", k, v);
                                }
                                last_ids.insert(stream_name.clone(), entry.id.clone());
                            }
                        }
                    }
                    Ok(None) => {
                        // XREAD timed out (block timeout), continue looping
                    }
                    Err(e) => {
                        eprintln!("Redis XREAD error: {e}");
                        sleep(Duration::from_secs(2)).await; // Brief backoff
                    }
                }
            }
        }
    }

    Ok(())
}

async fn xread_streams(
    con: &mut Connection,
    stream_keys: &[&str],
    last_ids: &HashMap<String, String>,
) -> redis::RedisResult<Option<StreamReadReply>> {
    let keys: Vec<&str> = stream_keys.iter().map(|s| *s).collect();
    let ids: Vec<&str> = stream_keys
        .iter()
        .map(|k| last_ids.get(*k).map(|s| s.as_str()).unwrap_or("0-0"))
        .collect();

    // XREAD with BLOCK for up to 5 seconds
    redis::cmd("XREAD")
        .arg("BLOCK")
        .arg(5000)
        .arg("COUNT")
        .arg(10)
        .arg("STREAMS")
        .arg(&keys)
        .arg(&ids)
        .query_async(con)
        .await
}

async fn other_task(
    mut shutdown: broadcast::Receiver<()>,
) {
    loop {
        tokio::select! {
            _ = shutdown.recv() => {
                println!("Other task received shutdown signal.");
                break;
            }
            _ = sleep(Duration::from_secs(3)) => {
                println!("Other async task is running...");
            }
        }
    }
}
```

---

## Features

- **Graceful Shutdown**:  
  - Press Ctrl+C to notify all tasks to exit cleanly.
- **Error Handling**:
  - All Redis operations are handled with clear error reporting.
  - If Redis connection fails at startup, the program exits with an error.
  - Transient Redis errors during streaming are logged; the task retries after a brief pause.
- **Parallel Tasks**:  
  - Both tasks (stream reader + background) run concurrently and exit when shutdown signal is received.

---

**You can now Ctrl+C to exit, and all tasks will shut down cleanly.**  
Let me know if you need further customization!
