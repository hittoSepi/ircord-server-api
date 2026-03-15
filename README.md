# IRCord HTTP API Server

HTTP API server library for ircord-server. Provides REST API for external integrations.

**Stack:** C++20 · Boost.Beast · Boost.Asio · JSON

## Quick Start

```cpp
#include <ircord/api/server.hpp>
#include <boost/asio.hpp>

namespace asio = boost::asio;

int main() {
    asio::io_context io_context;
    
    // Create server on port 8080
    ircord::api::ServerConfig config;
    config.port = 8080;
    config.api_keys = {"your-api-key"};
    
    auto server = std::make_shared<ircord::api::Server>(io_context, config);
    
    // Register routes
    server->get("/health", [](const auto& req) {
        return ircord::api::Response::ok({{"status", "ok"}});
    });
    
    server->post("/api/v1/bug-reports", [](const auto& req) {
        auto body = req.json_body();
        auto description = body["description"];
        // Store bug report...
        return ircord::api::Response::created({{"id", 1}});
    });
    
    // Start server
    server->start();
    
    io_context.run();
    return 0;
}
```

## Features

- **HTTP/1.1** with keep-alive support
- **RESTful routing** with path parameters (`/users/{id}`)
- **Middleware chain** (CORS, rate limiting, auth)
- **JSON request/response** handling
- **API key authentication**
- **Thread-safe** design

## Building

```bash
cd ircord-server-api
mkdir build && cd build
cmake ..
cmake --build .
```

## API Reference

### Server

```cpp
Server(io_context, config)     // Constructor
start()                        // Start listening
stop()                         // Stop server
get(path, handler)             // Register GET route
post(path, handler)            // Register POST route
put(path, handler)             // Register PUT route
del(path, handler)             // Register DELETE route
use(middleware)                // Add middleware
```

### Request

```cpp
method()        // HttpMethod
path()          // Full path
target_path()   // Path without query
query()         // Query parameters
headers()       // All headers
body()          // Raw body
json_body()     // Parsed JSON
bearer_token()  // Auth token
```

### Response

```cpp
status(code)        // Set status
header(k, v)        // Add header
body(text)          // Set body
json(obj)           // Set JSON body

// Helpers
Response::ok(data)
Response::created(data)
Response::bad_request(msg)
Response::not_found(msg)
Response::error(msg, code)
```

## Configuration

```toml
[http_api]
enabled = true
port = 8080
bind_address = "0.0.0.0"
api_keys = ["dev-key-change-in-production"]
cors_enabled = true
rate_limit_enabled = true
rate_limit_requests = 60
```

## Middleware

```cpp
// CORS
server.use(ircord::api::middleware::cors("*"));

// Rate limiting
server.use(ircord::api::middleware::rate_limit(60));

// API key auth
server.use(ircord::api::middleware::api_key_auth({"key1", "key2"}));

// Custom middleware
server.use([](Request& req, Response& res) -> bool {
    // Return true to continue, false to stop
    return true;
});
```

## License

MIT
