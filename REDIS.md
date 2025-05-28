*Thinking...*

> **Developing Rust Code**
> 
> I've started building the Rust example. First things first, I'm establishing the Redis connection with password authentication, that's a must. Now, I'm focusing on reading from those streams from the start, since the user wants to avoid consumer groups in this implementation.
> 
> 
> **Analyzing XREAD Results**
> 
> I'm now working through the XREAD result parsing. I've been examining the data structure of the response and how to best handle the nested nature of the stream data. The redis-rs crate's helper functions like `xread_streams_ids` look promising. I'm focusing on extracting stream keys, IDs, and the associated field-value pairs for clear output.
> 
> 
> **Constructing XREAD Commands**
> 
> I'm now immersed in assembling the `XREAD` command using the redis-rs crate. I am ensuring the correct formatting of the keys and IDs arguments. I've been looking at using the `xread_streams_ids` method, as the types seem straightforward to work with. My current goal is to pass the stream keys and starting IDs effectively and confirm that the command builds as expected.
> 
> 
> **Defining Data Structures**
> 
> I'm now focusing on the data structures involved in parsing the `XREAD` response. I'm mapping the expected `redis-rs` types, especially `StreamReadReply` and `StreamId`, to accurately extract the stream keys, IDs, and the corresponding data fields. I'm also preparing to gracefully handle empty or non-existent streams in the output.
> 
> 
Okay, here's a Rust example using the `redis-rs` library to connect to a Redis instance with password authentication, and then read multiple streams from their beginnings using `XREAD` (without consumer groups).

First, make sure you have `redis-rs` and `tokio` (for the async runtime) in your `Cargo.toml`:

```toml
[dependencies]
redis = { version = "0.25.4", features = ["tokio-comp"] }
tokio = { version = "1", features = ["full"] }
# For parsing the stream data easily
serde_json = "1.0"
```

Now, here's the Rust code:

```rust
use redis::AsyncCommands; // Use AsyncCommands for `con.xadd_map` etc.
use redis::streams::{StreamReadOptions, StreamReadReply};
use std::collections::HashMap;

// Configuration
const REDIS_HOST: &str = "127.0.0.1"; // Or your Redis host
const REDIS_PORT: u16 = 6379;
const REDIS_PASSWORD: &str = "your_secure_password"; // CHANGE THIS!

// Stream keys we want to read
const STREAM_KEYS: &[&str] = &["mystream:1", "mystream:2", "anotherstream"];

#[tokio::main]
async fn main() -> redis::RedisResult<()> {
    // 1. Construct the Redis connection URL with authentication
    let redis_url = if REDIS_PASSWORD.is_empty() {
        format!("redis://{}:{}/", REDIS_HOST, REDIS_PORT)
    } else {
        format!(
            "redis://:{}@{}:{}/",
            REDIS_PASSWORD, REDIS_HOST, REDIS_PORT
        )
    };

    // 2. Connect to Redis
    let client = match redis::Client::open(redis_url) {
        Ok(client) => client,
        Err(e) => {
            eprintln!("Failed to create Redis client: {}", e);
            return Err(e);
        }
    };

    let mut con = match client.get_async_connection().await {
        Ok(con) => con,
        Err(e) => {
            eprintln!("Failed to connect to Redis: {}", e);
            eprintln!("Please ensure Redis is running at {}:{} and the password is correct.", REDIS_HOST, REDIS_PORT);
            return Err(e);
        }
    };
    println!("Successfully connected to Redis!");

    // (Optional) Add some sample data to the streams for demonstration
    // In a real scenario, data would already be in the streams or added by other producers.
    println!("\nAdding some sample data to streams...");
    for &key in STREAM_KEYS {
        // Add a couple of entries to each stream
        let mut map1: HashMap<&str, &str> = HashMap::new();
        map1.insert("sensor_id", "S1");
        map1.insert("temperature", "25.5");
        let id1: String = con.xadd_map(key, "*", &map1).await?;
        println!("Added to {}: ID {}, Data: {:?}", key, id1, map1);

        tokio::time::sleep(tokio::time::Duration::from_millis(10)).await; // Ensure different timestamps

        let mut map2: HashMap<&str, &str> = HashMap::new();
        map2.insert("event", "door_opened");
        map2.insert("user_id", "U123");
        let id2: String = con.xadd_map(key, "*", &map2).await?;
        println!("Added to {}: ID {}, Data: {:?}", key, id2, map2);
    }
    println!("Sample data added.");


    // 3. Read from multiple streams from the beginning
    println!("\nReading streams from the beginning...");

    // Prepare the stream keys and IDs for XREAD
    // To read from the beginning, the ID for each stream is "0-0"
    let stream_ids: Vec<&str> = STREAM_KEYS.iter().map(|_| "0-0").collect();

    // Using StreamReadOptions for more clarity if needed, though not strictly necessary for "0-0"
    // let opts = StreamReadOptions::default().count(10); // Example: Read up to 10 entries per stream call

    // The XREAD command can take multiple stream keys and their corresponding start IDs.
    // `redis-rs` provides a convenient `xread_options` method.
    let reply: StreamReadReply = con
        .xread_options(
            STREAM_KEYS,
            &stream_ids,
            // &opts, // Uncomment if you want to use options like COUNT
            &StreamReadOptions::default(), // Default options (no BLOCK, no COUNT limit per call unless specified)
        )
        .await?;

    // 4. Process the reply
    if reply.keys.is_empty() {
        println!("No data found in the specified streams or streams do not exist.");
    } else {
        println!("Data read from streams:");
        for stream_key_reply in reply.keys {
            println!("\n--- Stream: {} ---", stream_key_reply.key);
            if stream_key_reply.ids.is_empty() {
                println!("  (No messages in this stream for this read operation)");
                continue;
            }
            for stream_id_entry in stream_key_reply.ids {
                println!("  ID: {}", stream_id_entry.id);
                // The `map` contains field-value pairs.
                // Values in the map are `redis::Value`, which you might need to convert.
                for (field, value) in stream_id_entry.map.iter() {
                    // Attempt to convert redis::Value to a String for display
                    let value_str = match value {
                        redis::Value::Data(bytes) => String::from_utf8_lossy(bytes).to_string(),
                        redis::Value::Int(i) => i.to_string(),
                        redis::Value::Nil => "Nil".to_string(),
                        _ => format!("{:?}", value), // Fallback for other types
                    };
                    println!("    {}: {}", field, value_str);
                }
            }
        }
    }

    println!("\nFinished reading streams.");
    Ok(())
}
```

**Explanation:**

1.  **Dependencies:**
    *   `redis`: The core `redis-rs` library. The `tokio-comp` feature is for `async/await` support with Tokio.
    *   `tokio`: Used as the asynchronous runtime.
    *   `serde_json`: (Commented out in the `Cargo.toml` above, but useful if your stream values are JSON strings that you want to parse into Rust structs). For this example, we're just printing fields as strings.

2.  **Configuration:**
    *   `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`: Constants to configure your Redis connection. **Remember to change `REDIS_PASSWORD`!** If you don't have a password, leave it empty.
    *   `STREAM_KEYS`: An array of stream keys you want to read.

3.  **Connection URL:**
    *   The code constructs a Redis connection URL in the format `redis://:[password]@[hostname]:[port]/`. If the password is empty, it omits the password part.

4.  **Connecting:**
    *   `redis::Client::open()` creates a client.
    *   `client.get_async_connection().await` establishes the actual connection. Error handling is included.

5.  **(Optional) Adding Sample Data:**
    *   The section with `con.xadd_map(...)` is there to ensure the streams have some data when you run the example. In a real application, producers would be adding data.
    *   `"*"` as the ID in `xadd_map` tells Redis to auto-generate the ID.

6.  **Preparing for `XREAD`:**
    *   `stream_ids`: We create a vector of IDs. For reading from the beginning of each stream, the ID is `"0-0"`. This vector must have the same number of elements as `STREAM_KEYS`.
    *   `StreamReadOptions`: While not strictly necessary for just reading from `0-0` with no `COUNT` or `BLOCK`, it's good to be aware of. You could use it to specify `COUNT` (max messages per stream per call) or `BLOCK` (to wait for new messages). For this example, we use `StreamReadOptions::default()`.

7.  **Executing `XREAD`:**
    *   `con.xread_options(STREAM_KEYS, &stream_ids, &StreamReadOptions::default()).await?` is the call that performs the `XREAD` operation.
    *   It takes the slice of stream keys, a slice of corresponding start IDs, and options.
    *   The result is parsed into a `StreamReadReply` struct.

8.  **Processing the Reply (`StreamReadReply`):**
    *   `StreamReadReply` has a field `keys` which is a `Vec<redis::streams::StreamKey>`.
    *   Each `redis::streams::StreamKey` (renamed to `stream_key_reply` in the loop for clarity) contains:
        *   `key`: The name of the stream.
        *   `ids`: A `Vec<redis::streams::StreamId>`, representing the messages read from that stream.
    *   Each `redis::streams::StreamId` (renamed to `stream_id_entry`) contains:
        *   `id`: The message ID (e.g., "1625075790000-0").
        *   `map`: A `HashMap<String, redis::Value>` where keys are field names and values are the corresponding `redis::Value`s.
    *   The code iterates through this structure and prints the stream name, message ID, and field-value pairs.
    *   `redis::Value` is an enum representing different Redis data types. We attempt a simple conversion to string for display.

**Before Running:**

*   Make sure your Redis server is running.
*   If it's password-protected, ensure the `requirepass` directive in your `redis.conf` is set and matches `REDIS_PASSWORD`.
*   Update `REDIS_PASSWORD` in the code. If your Redis has no password, set `REDIS_PASSWORD = "";`.

This example provides a solid foundation for reading multiple streams from their beginning with authentication.
