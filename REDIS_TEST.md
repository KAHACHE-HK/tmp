Absolutely! You **can run [miniredis](https://github.com/alicebob/miniredis) as a local Go binary** in your integration tests _without Docker_.  
The steps are:

1. **Install `miniredis`** (as a binary on your machine).
2. **Spawn `miniredis` as a subprocess** in your test.
3. **Connect your Rust code to the subprocess** using the port it listens on.

---

## Step 1: Install `miniredis`

If you have Go installed:

```sh
go install github.com/alicebob/miniredis/v2@latest
```
This puts `miniredis` in your `$GOPATH/bin` (often `~/go/bin/miniredis`).

---

## Step 2: Integration Test Example

Here’s a **Rust integration test** that spawns and manages a local `miniredis` binary.

Save as `tests/miniredis_spawn.rs`:

```rust
use redis::AsyncCommands;
use std::process::{Command, Child, Stdio};
use std::thread;
use std::time::Duration;
use std::net::{TcpListener, SocketAddr};
use tokio;

fn find_free_port() -> u16 {
    TcpListener::bind("127.0.0.1:0")
        .unwrap()
        .local_addr()
        .unwrap()
        .port()
}

// Helper to spawn miniredis on a random free port
fn spawn_miniredis(port: u16) -> Child {
    let port_arg = format!("-p={}", port);
    Command::new("miniredis")
        .arg(&port_arg)
        .stdout(Stdio::null())
        .stderr(Stdio::null())
        .spawn()
        .expect("failed to start miniredis (is it installed and in your $PATH?)")
}

#[tokio::test]
async fn test_miniredis_spawn() {
    // 1. Find an available port & launch miniredis
    let port = find_free_port();
    let mut child = spawn_miniredis(port);

    // 2. Wait briefly for miniredis to start
    thread::sleep(Duration::from_millis(300));

    // 3. Connect to miniredis
    let url = format!("redis://127.0.0.1:{}/", port);
    let client = redis::Client::open(url).unwrap();
    let mut con = client.get_async_connection().await.unwrap();

    // 4. Test stream commands
    let _: String = redis::cmd("XADD")
        .arg("mystream1")
        .arg("*")
        .arg("field1")
        .arg("value1")
        .query_async(&mut con)
        .await
        .unwrap();

    let reply: redis::streams::StreamReadReply = redis::cmd("XREAD")
        .arg("COUNT").arg(10)
        .arg("STREAMS")
        .arg("mystream1")
        .arg("0-0")
        .query_async(&mut con)
        .await
        .unwrap();

    assert_eq!(reply.keys.len(), 1);
    let stream = &reply.keys[0];
    assert_eq!(stream.key, "mystream1");
    assert_eq!(stream.ids.len(), 1);
    let entry = &stream.ids[0];
    assert_eq!(entry.map.get("field1"), Some(&"value1".into()));

    // 5. Clean up miniredis
    let _ = child.kill();
}
```

---

## Step 3: Run the Test

```sh
cargo test --test miniredis_spawn
```

**If you get an error about `miniredis` not being found:**
- Make sure `miniredis` is in your `$PATH`, or use the full path in `Command::new("/full/path/to/miniredis")`.

---

## Notes

- **No password support** in miniredis, so skip AUTH tests.
- If you want to capture logs, change `Stdio::null()` to `Stdio::inherit()`.
- You can make setup/teardown more robust by waiting for the port to be open, etc.
- You can use this approach for **any Rust Redis integration test**.

---

## Summary Table

| Step                  | Command / Action                                   | Purpose                         |
|-----------------------|----------------------------------------------------|---------------------------------|
| Install miniredis     | `go install github.com/alicebob/miniredis/v2@latest` | Get the binary                  |
| Write test code       | See example above                                  | Spawns and tests against Redis  |
| Run test              | `cargo test --test miniredis_spawn`                | Executes your integration test  |

---

**Let me know if you want a utility function for waiting on the port, or more advanced examples!**
