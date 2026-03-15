# IRCord HTTP API Server

C++20 HTTP/1.1 server library for ircord-server. Provides REST API endpoints for external integrations, web dashboards, and client-server communication.

## Features

| Feature | Description |
|---------|-------------|
| **HTTP/1.1** | Keep-alive connections, chunked transfer |
| **REST Routing** | Path parameters (`/users/{id}`), wildcards |
| **Middleware** | CORS, rate limiting, authentication chains |
| **JSON** | Built-in nlohmann/json integration |
| **Auth** | Bearer tokens, API key management |
| **Thread-safe** | Asio strand-based concurrency |

## Quick Start

```cpp
#include <ircord/api/server.hpp>
#include <boost/asio.hpp>

int main() {
    boost::asio::io_context io;
    
    // Configure server
    ircord::api::ServerConfig config;
    config.port = 8080;
    config.api_keys = {ircord::api::ApiKeyManager::generate_key()};
    
    // Create server
    auto server = std::make_shared<ircord::api::Server>(io, config);
    
    // Define routes
    server->get("/health", [](const auto& req) {
        return ircord::api::Response::ok({{"status", "ok"}});
    });
    
    server->post("/api/v1/messages", [](const auto& req) {
        auto json = req.json_body();
        // Process message...
        return ircord::api::Response::created({{"id", 123}});
    });
    
    // Start
    server->start();
    io.run();
    return 0;
}
```

## Installation

### As Git Submodule (Recommended)

```bash
cd ircord-server
git submodule add https://github.com/hittoSepi/ircord-server-api.git
git submodule update --init
```

### CMake Integration

```cmake
add_subdirectory(ircord-server-api)

target_link_libraries(your-target PRIVATE ircord-server-api)
```

### Dependencies

- Boost ≥1.82 (system, beast)
- nlohmann/json ≥3.11
- spdlog ≥1.12
- OpenSSL (for HMAC auth)

## API Reference

### Server

```cpp
// Construction
Server(io_context, config)

// Lifecycle
void start()          // Start listening
void stop()           // Graceful shutdown
bool is_running()

// Routes
void get(path, handler)
void post(path, handler)
void put(path, handler)
void del(path, handler)
void patch(path, handler)

// Middleware
void use(middleware)

// Components
Router& router()
ApiKeyManager& api_keys()
```

### Request

```cpp
HttpMethod method()                    // GET, POST, etc.
std::string path()                     // Full path with query
std::string target_path()              // Path without query
QueryParams query()                    // Parsed query params
std::optional<std::string> query_param(key)

Headers headers()
std::optional<std::string> header(key)
std::optional<std::string> bearer_token()

std::string body()
json json_body()                       // Throws if invalid
bool has_json_body()

std::string remote_addr()
```

### Response

```cpp
Response& status(int code)
Response& header(key, value)
Response& content_type(type)
Response& body(text)
Response& json(data)

// Factory methods
static Response ok(data)
static Response created(data)
static Response no_content()
static Response bad_request(msg)
static Response unauthorized(msg)
static Response not_found(msg)
static Response error(msg, code)
```

### Middleware

```cpp
// CORS headers
server.use(ircord::api::middleware::cors("*"));

// Rate limiting (requests per minute)
server.use(ircord::api::middleware::rate_limit(60));

// API key authentication
server.use(ircord::api::middleware::api_key_auth({"key1", "key2"}));

// Require JSON Content-Type
server.use(ircord::api::middleware::require_json());

// Request logging
server.use(ircord::api::middleware::logger());

// Custom middleware
server.use([](Request& req, Response& res) -> bool {
    // Return true to continue, false to reject
    return true;
});
```

## Configuration

```cpp
struct ServerConfig {
    uint16_t port = 8080;
    std::string bind_address = "0.0.0.0";
    bool cors_enabled = true;
    bool rate_limit_enabled = true;
    int rate_limit_requests = 60;
    std::vector<std::string> api_keys;
};
```

## Routing

```cpp
// Static routes
server.get("/api/v1/users", list_users);

// Path parameters
server.get("/api/v1/users/{id}", get_user);
server.del("/api/v1/users/{id}", delete_user);

// Access parameters in handler
auto handler = [](const Request& req) -> Response {
    // req is passed to handler, but params are in router match
    // Access via: router.match(req.method(), req.target_path()).params
    return Response::ok();
};
```

## Example: Bug Report API

```cpp
server->post("/api/v1/bug-reports", [](const auto& req) {
    // Validate JSON
    if (!req.has_json_body()) {
        return ircord::api::Response::bad_request("JSON required");
    }
    
    auto body = req.json_body();
    
    // Validate fields
    if (!body.contains("description")) {
        return ircord::api::Response::bad_request("Missing description");
    }
    
    std::string desc = body["description"];
    if (desc.length() < 10) {
        return ircord::api::Response::bad_request("Description too short");
    }
    
    // Store and respond
    int id = store_bug_report(desc);
    return ircord::api::Response::created({
        {"id", id},
        {"message", "Bug report submitted"}
    });
});
```

## Testing

```bash
# Build tests
cd ircord-server-api
mkdir build && cd build
cmake .. -DIRCORD_API_BUILD_TESTS=ON
cmake --build .

# Run tests
ctest --output-on-failure
```

## Manual Testing

```bash
# Health check
curl http://localhost:8080/health

# With authentication
curl http://localhost:8080/api/v1/data \
  -H "Authorization: Bearer your-api-key"

# POST JSON
curl -X POST http://localhost:8080/api/v1/bug-reports \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-api-key" \
  -d '{"description": "Something broke"}'
```

## Integration with IRCord

See [INTEGRATION.md](INTEGRATION.md) for detailed instructions on integrating with ircord-server.

## Architecture

```
┌─────────────────────────────────────────┐
│           IRCord Server                  │
│                                          │
│  ┌─────────────┐    ┌─────────────────┐ │
│  │ IRCord TLS  │    │ HTTP API Server │ │
│  │ Protocol    │    │ (this library)  │ │
│  │ Port 6697   │    │ Port 8080       │ │
│  └─────────────┘    └─────────────────┘ │
│         │                    │           │
│         └──────┬─────────────┘           │
│                │                         │
│         ┌──────▼──────┐                  │
│         │ Server Core │                  │
│         │ - Channels  │                  │
│         │ - Users     │                  │
│         │ - Messages  │                  │
│         └─────────────┘                  │
└─────────────────────────────────────────┘
```

## License

MIT License - See LICENSE file
