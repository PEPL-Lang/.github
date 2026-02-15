# Capability System

PEPL spaces interact with the outside world through **capabilities** — declared dependencies on host-provided services. The space declares what it needs; the host decides whether to grant access and mediates all I/O.

## Capability Categories

| Category | Capabilities | Description |
|----------|-------------|-------------|
| **Platform** | `display`, `keyboard_or_touch`, `voice_input` | Input/output modalities. Informational in Phase 0 — declaring them documents the space's modality requirements for future platform-aware scheduling. No runtime effect. |
| **I/O** | `http`, `storage`, `notifications` | Host-mediated external I/O |
| **Sensors** | `location`, `camera`, `accelerometer` | Device sensor access |
| **System** | `clipboard`, `share` | OS-level integration |

## Capability Declaration

```pepl
space WeatherTracker {
  capabilities {
    required: [display, keyboard_or_touch, http, storage]
    optional: [location, notifications, voice_input]
  }

  // Capabilities are available as stdlib-like modules
  // Required capabilities are always available inside the space
  // Optional capabilities are checked with core.capability()

  action refresh_weather() {
    if core.capability("location") {
      let coords = location.current()?
      let weather = http.get("https://api.weather.dev/v1?lat=${coords.lat}&lon=${coords.lon}")?
      set forecast = weather.body
    } else {
      let weather = http.get("https://api.weather.dev/v1?city=${city_name}")?
      set forecast = weather.body
    }
  }
}
```

## Host-Mediated I/O Model

All capability calls are **host-mediated**. PEPL code never performs I/O directly:

```
PEPL calls http.get(url) → yields to host
  → Host performs HTTP request
  → Host records response as event (for deterministic replay)
  → Host resumes PEPL execution with response value
  
On replay: Host returns recorded response (no live request)
```

This preserves PEPL's determinism guarantee:
- **First execution**: Host performs live I/O, records result as event
- **Replay/audit**: Host returns recorded result — same inputs, same outputs
- **Offline**: Host can return cached/recorded results or error

## Capability Modules

Capability-gated modules become available when declared in the `capabilities` block. They are NOT part of the core stdlib — they require host support.

### `http` module (requires `http` capability)

| Function | Signature | Description |
|---|---|---|
| `http.get` | `(url: string, options?: HttpOptions) -> Result<HttpResponse, HttpError>` | GET request — yields to host |
| `http.post` | `(url: string, body: string, options?: HttpOptions) -> Result<HttpResponse, HttpError>` | POST request — yields to host |
| `http.put` | `(url: string, body: string, options?: HttpOptions) -> Result<HttpResponse, HttpError>` | PUT request — yields to host |
| `http.patch` | `(url: string, body: string, options?: HttpOptions) -> Result<HttpResponse, HttpError>` | PATCH request — yields to host |
| `http.delete` | `(url: string, options?: HttpOptions) -> Result<HttpResponse, HttpError>` | DELETE request — yields to host |

```pepl
// HttpOptions type (all fields optional)
// {
//   headers: list<{ key: string, value: string }>,
//   timeout: number,       // milliseconds, default: 30000
//   content_type: string   // default: "application/json"
// }

// HttpResponse type
// { status: number, body: string, headers: list<{ key: string, value: string }> }

// HttpError type
// { code: string, message: string }
// Codes: "network_error", "timeout", "blocked_by_host", "offline", "cors_error"
```

### Credential References

Spaces MUST NOT contain secrets (API keys, tokens, passwords) in source code. Instead, spaces declare required credentials in the `credentials {}` block, and the host injects their values at runtime:

```pepl
space WeatherDashboard {
  credentials {
    api_key: string   // Host prompts user to configure this before first use
  }

  capabilities {
    required: [display, keyboard_or_touch, http]
  }

  // ... state, views ...

  action fetch_data() {
    let response = http.get(
      "https://api.example.com/data",
      { headers: [{ key: "Authorization", value: "Bearer ${api_key}" }] }
    )
    match response {
      Ok(data) -> {
        let parsed = json.parse(data.body)
        match parsed {
          Ok(d) -> { set items = d }
          Err(e) -> { set error_message = e.message }
        }
      }
      Err(error) -> { set error_message = error.message }
    }
  }
}
```

Credential names declared in the `credentials {}` block become read-only bindings within the space. They are:
- **Compile-time declared** — the compiler knows which credentials a space requires
- **Runtime injected** — the host provides values when the space executes
- **Never stored** — credentials never appear in PEPL source, event log, or WASM binary
- **Required** — if the host cannot provide a declared credential, the space does not start

The host application manages credential storage (e.g., encrypted settings, OS keychain). When a space declares credentials, the host prompts the user to configure them before the space can run.

### CORS Proxy (Phase 0 — Web)

In Phase 0 (browser), all `http` calls are routed through a **host-side proxy**:

```
PEPL calls http.get(url) → yields to host
  → Host forwards request through backend relay (avoids CORS)
  → Backend relay performs HTTP request
  → Response returned to host → recorded as event → returned to PEPL
```

The proxy is transparent to PEPL code — spaces use the same `http.get()` API regardless of platform. The host decides the routing strategy. On non-browser platforms (Phase 1+), the host MAY perform HTTP requests directly without a relay.

### `storage` module (requires `storage` capability)

| Function | Signature | Description |
|---|---|---|
| `storage.get` | `(key: string) -> Result<string, StorageError>` | Read from persistent storage — yields to host |
| `storage.set` | `(key: string, value: string) -> Result<nil, StorageError>` | Write to persistent storage — yields to host |
| `storage.delete` | `(key: string) -> Result<nil, StorageError>` | Delete from persistent storage — yields to host |
| `storage.keys` | `() -> Result<list<string>, StorageError>` | List all keys — yields to host |

### `location` module (requires `location` capability)

| Function | Signature | Description |
|---|---|---|
| `location.current` | `() -> Result<{ lat: number, lon: number }, LocationError>` | Current position — yields to host |

### `notifications` module (requires `notifications` capability)

| Function | Signature | Description |
|---|---|---|
| `notifications.send` | `(title: string, body: string) -> Result<nil, NotificationError>` | Send a notification — yields to host |
