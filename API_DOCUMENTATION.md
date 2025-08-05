# Python-libevent API Documentation

Python-libevent provides Python bindings for the libevent library, enabling high-performance event-driven network programming.

## Table of Contents

1. [Installation](#installation)
2. [Core Classes](#core-classes)
   - [Base](#base)
   - [Event](#event)
   - [Timer](#timer)
   - [Signal](#signal)
3. [Buffer Classes](#buffer-classes)
   - [Buffer](#buffer)
   - [BufferEvent](#bufferevent)
4. [HTTP Classes](#http-classes)
   - [HttpServer](#httpserver)
   - [HttpRequest](#httprequest)
5. [Constants and Flags](#constants-and-flags)
6. [Examples](#examples)
7. [Error Handling](#error-handling)

## Installation

### Prerequisites

- libevent 2.0 or later
- Python 2.7 or Python 3.x
- C compiler (GCC, MSVC, etc.)

### Building from Source

1. Set the `LIBEVENT_ROOT` environment variable:
   ```bash
   export LIBEVENT_ROOT=/path/to/libevent
   ```

2. Build and install:
   ```bash
   python setup.py build
   python setup.py install
   ```

### Windows (MSVC)

```cmd
# Build libevent first
cd libevent-2.0.21-stable
nmake -f Makefile.nmake

# Build python-libevent
cd python-libevent
set LIBEVENT_ROOT=C:\path\to\libevent-2.0.21-stable
python setup.py build
```

## Core Classes

### Base

The `Base` class represents an event base - the central dispatcher for events.

#### Constructor

```python
Base(config=None)
```

**Parameters:**
- `config` (Config, optional): Configuration object for the event base

**Example:**
```python
import libevent

# Create a default event base
base = libevent.Base()
print(f"Using {base.method} backend")

# Create with specific configuration
config = libevent.Config()
base = libevent.Base(config)
```

#### Methods

##### `reinit()`

Reinitialize the event base after a fork.

**Returns:** None

**Example:**
```python
import os
import libevent

base = libevent.Base()
if os.fork() == 0:
    # In child process
    base.reinit()
```

##### `dispatch()`

Run the event loop once, processing events that are ready.

**Returns:** None

**Example:**
```python
base = libevent.Base()
# Add some events...
base.dispatch()  # Process one round of events
```

##### `loop(flags=0)`

Run the event loop until no more events are pending.

**Parameters:**
- `flags` (int, optional): Loop flags (EVLOOP_ONCE, EVLOOP_NONBLOCK)

**Returns:** None

**Example:**
```python
base = libevent.Base()
# Add events...
base.loop()  # Run until no more events
```

##### `loopexit(timeout=None)`

Exit the event loop after the specified timeout.

**Parameters:**
- `timeout` (float, optional): Timeout in seconds

**Returns:** None

**Example:**
```python
base = libevent.Base()
base.loopexit(5.0)  # Exit after 5 seconds
base.loop()
```

##### `loopbreak()`

Exit the event loop immediately.

**Returns:** None

**Example:**
```python
def signal_handler(evt, signum, userdata):
    evt.base.loopbreak()

base = libevent.Base()
signal_evt = libevent.Signal(base, signal.SIGINT, signal_handler)
signal_evt.add()
base.loop()  # Will exit when SIGINT is received
```

##### `got_exit()`

Check if the event loop was exited via `loopexit()`.

**Returns:** bool

##### `got_break()`

Check if the event loop was exited via `loopbreak()`.

**Returns:** bool

##### `priority_init(priorities)`

Initialize event priorities.

**Parameters:**
- `priorities` (int): Number of priority levels

**Returns:** None

#### Properties

- `method` (str, read-only): The kernel event notification mechanism in use
- `features` (int, read-only): Bitmask of features implemented by the backend

### Event

The `Event` class represents a single event that can be monitored.

#### Constructor

```python
Event(base, fd, events, callback, userdata=None)
```

**Parameters:**
- `base` (Base): The event base to use
- `fd` (int): File descriptor to monitor
- `events` (int): Event types to monitor (EV_READ, EV_WRITE, etc.)
- `callback` (callable): Function to call when event triggers
- `userdata` (object, optional): User data to pass to callback

**Example:**
```python
import socket
import libevent

def handle_read(evt, fd, what, userdata):
    if what & libevent.EV_READ:
        data = userdata.recv(1024)
        print(f"Received: {data}")

base = libevent.Base()
sock = socket.socket()
sock.setblocking(False)

evt = libevent.Event(base, sock.fileno(), libevent.EV_READ, handle_read, sock)
```

#### Methods

##### `add(timeout=None)`

Add the event to the event base.

**Parameters:**
- `timeout` (float, optional): Timeout in seconds

**Returns:** None

**Example:**
```python
evt.add()        # Add without timeout
evt.add(5.0)     # Add with 5-second timeout
```

##### `delete()`

Remove the event from the event base.

**Returns:** None

**Example:**
```python
evt.delete()
```

##### `set_priority(priority)`

Set the event priority.

**Parameters:**
- `priority` (int): Priority level (lower numbers = higher priority)

**Returns:** None

#### Properties

- `base` (Base, read-only): The event base this event is assigned to
- `callback` (callable, read-only): The callback function
- `userdata` (object, read-only): The user data
- `fd` (int, read-only): The file descriptor

### Timer

The `Timer` class is a simplified interface for timer events.

#### Constructor

```python
Timer(base, callback, userdata=None)
```

**Parameters:**
- `base` (Base): The event base to use
- `callback` (callable): Function to call when timer fires
- `userdata` (object, optional): User data to pass to callback

**Example:**
```python
import libevent

def timer_callback(timer, userdata):
    print("Timer fired!")
    timer.base.loopbreak()

base = libevent.Base()
timer = libevent.Timer(base, timer_callback)
timer.add(2.0)  # Fire after 2 seconds
base.loop()
```

#### Methods

Inherits all methods from `Event` class.

### Signal

The `Signal` class handles Unix signals.

#### Constructor

```python
Signal(base, signum, callback, userdata=None)
```

**Parameters:**
- `base` (Base): The event base to use
- `signum` (int): Signal number to handle
- `callback` (callable): Function to call when signal is received
- `userdata` (object, optional): User data to pass to callback

**Example:**
```python
import signal
import libevent

def signal_handler(sig_evt, signum, userdata):
    print(f"Received signal {signum}")
    sig_evt.base.loopbreak()

base = libevent.Base()
sig_evt = libevent.Signal(base, signal.SIGINT, signal_handler)
sig_evt.add()
base.loop()
```

#### Methods

Inherits all methods from `Event` class.

## Buffer Classes

### Buffer

The `Buffer` class provides a growable buffer for efficient I/O operations.

#### Constructor

```python
Buffer()
```

**Example:**
```python
import libevent

buffer = libevent.Buffer()
```

#### Methods

##### `enable_locking()`

Enable thread-safe locking for the buffer.

**Returns:** None

**Example:**
```python
buffer = libevent.Buffer()
buffer.enable_locking()
```

##### `lock()` / `unlock()`

Manually lock/unlock the buffer for thread safety.

**Returns:** None

**Example:**
```python
buffer.lock()
try:
    # Perform buffer operations
    buffer.add(b"data")
finally:
    buffer.unlock()

# Or use as context manager
with buffer:
    buffer.add(b"data")
```

##### `get_contiguous_space()`

Get the size of contiguous space available at the end of the buffer.

**Returns:** int - Number of bytes available

##### `expand(size)`

Expand the buffer by the specified number of bytes.

**Parameters:**
- `size` (int): Number of bytes to expand

**Returns:** None

##### `add(data)`

Add data to the end of the buffer.

**Parameters:**
- `data` (bytes or str): Data to add

**Returns:** None

**Example:**
```python
buffer = libevent.Buffer()
buffer.add(b"Hello, ")
buffer.add(b"World!")
```

##### `remove(size)`

Remove data from the beginning of the buffer.

**Parameters:**
- `size` (int): Number of bytes to remove

**Returns:** bytes - The removed data

**Example:**
```python
buffer.add(b"Hello, World!")
data = buffer.remove(5)  # Returns b"Hello"
```

##### `copyout(size)`

Copy data from the beginning of the buffer without removing it.

**Parameters:**
- `size` (int): Number of bytes to copy

**Returns:** bytes - The copied data

##### `remove_buffer(src, size)`

Move data from another buffer to this buffer.

**Parameters:**
- `src` (Buffer): Source buffer
- `size` (int): Number of bytes to move

**Returns:** int - Number of bytes actually moved

##### `readln(style=EVBUFFER_EOL_CRLF_STRICT)`

Read a line from the buffer.

**Parameters:**
- `style` (int, optional): Line ending style

**Returns:** bytes - The line data, or None if no complete line

**Example:**
```python
buffer.add(b"Line 1\r\nLine 2\r\n")
line1 = buffer.readln()  # Returns b"Line 1"
line2 = buffer.readln()  # Returns b"Line 2"
```

##### `add_file(fd, offset, length)`

Add data from a file to the buffer.

**Parameters:**
- `fd` (int): File descriptor
- `offset` (int): Offset in the file
- `length` (int): Number of bytes to read

**Returns:** int - Number of bytes actually added

##### `drain(size)`

Remove data from the buffer without returning it.

**Parameters:**
- `size` (int): Number of bytes to drain

**Returns:** None

##### `write(fd)`

Write buffer contents to a file descriptor.

**Parameters:**
- `fd` (int): File descriptor to write to

**Returns:** int - Number of bytes written

##### `read(fd, size)`

Read data from a file descriptor into the buffer.

**Parameters:**
- `fd` (int): File descriptor to read from
- `size` (int): Maximum number of bytes to read

**Returns:** int - Number of bytes actually read

##### `pullup(size)`

Ensure that at least `size` bytes are contiguous at the start of the buffer.

**Parameters:**
- `size` (int): Number of bytes to make contiguous

**Returns:** bytes - Pointer to the contiguous data

##### `prepend(data)`

Add data to the beginning of the buffer.

**Parameters:**
- `data` (bytes): Data to prepend

**Returns:** None

##### `freeze(at_front)`

Prevent modifications to the front or back of the buffer.

**Parameters:**
- `at_front` (bool): If True, freeze the front; if False, freeze the back

**Returns:** None

##### `unfreeze(at_front)`

Allow modifications to the front or back of the buffer.

**Parameters:**
- `at_front` (bool): If True, unfreeze the front; if False, unfreeze the back

**Returns:** None

##### `defer_callbacks(defer)`

Enable or disable callback deferral.

**Parameters:**
- `defer` (bool): Whether to defer callbacks

**Returns:** None

#### Properties

- `length` (int, read-only): Number of bytes in the buffer

### BufferEvent

The `BufferEvent` class provides buffered I/O with automatic read/write event handling.

#### Constructor

```python
BufferEvent(base, fd=None, options=0)
```

**Parameters:**
- `base` (Base): The event base to use
- `fd` (int, optional): File descriptor for the connection
- `options` (int, optional): BufferEvent options

**Example:**
```python
import socket
import libevent

base = libevent.Base()
sock = socket.socket()
sock.connect(('example.com', 80))

bev = libevent.BufferEvent(base, sock.fileno())
```

#### Methods

##### `lock()` / `unlock()`

Lock/unlock the BufferEvent for thread safety.

**Returns:** None

**Example:**
```python
with bev:  # Uses __enter__/__exit__
    bev.write(b"data")
```

##### `set_callbacks(readcb, writecb, eventcb, userdata=None)`

Set callback functions for read, write, and error events.

**Parameters:**
- `readcb` (callable): Called when data is available to read
- `writecb` (callable): Called when ready to write
- `eventcb` (callable): Called on connection events (errors, EOF, etc.)
- `userdata` (object, optional): User data passed to callbacks

**Example:**
```python
def read_callback(bev, userdata):
    data = bev.read(8192)
    print(f"Received: {data}")

def write_callback(bev, userdata):
    print("Ready to write")

def event_callback(bev, events, userdata):
    if events & libevent.BEV_EVENT_CONNECTED:
        print("Connected")
    elif events & libevent.BEV_EVENT_ERROR:
        print("Error occurred")

bev.set_callbacks(read_callback, write_callback, event_callback)
```

##### `write(data)`

Write data to the output buffer.

**Parameters:**
- `data` (bytes): Data to write

**Returns:** None

**Example:**
```python
bev.write(b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n")
```

##### `read(size=None)`

Read data from the input buffer.

**Parameters:**
- `size` (int, optional): Maximum bytes to read (default: all available)

**Returns:** bytes - The read data

**Example:**
```python
data = bev.read()      # Read all available data
data = bev.read(1024)  # Read up to 1024 bytes
```

##### `enable(events)`

Enable monitoring for specific events.

**Parameters:**
- `events` (int): Event types to enable (EV_READ, EV_WRITE)

**Returns:** None

**Example:**
```python
bev.enable(libevent.EV_READ | libevent.EV_WRITE)
```

##### `disable(events)`

Disable monitoring for specific events.

**Parameters:**
- `events` (int): Event types to disable

**Returns:** None

**Example:**
```python
bev.disable(libevent.EV_WRITE)
```

##### `set_timeouts(timeout_read, timeout_write)`

Set read and write timeouts.

**Parameters:**
- `timeout_read` (float): Read timeout in seconds
- `timeout_write` (float): Write timeout in seconds

**Returns:** None

**Example:**
```python
bev.set_timeouts(30.0, 30.0)  # 30-second timeouts
```

##### `set_watermark(events, low, high)`

Set low and high watermarks for buffering.

**Parameters:**
- `events` (int): Event types (EV_READ or EV_WRITE)
- `low` (int): Low watermark
- `high` (int): High watermark

**Returns:** None

##### `set_ratelimit(bucket)`

Set rate limiting for the BufferEvent.

**Parameters:**
- `bucket` (RateLimitBucket): Rate limit configuration

**Returns:** None

#### Properties

- `base` (Base, read-only): The event base this BufferEvent is assigned to
- `input` (Buffer, read-only): The input buffer
- `output` (Buffer, read-only): The output buffer
- `bucket` (RateLimitBucket, read-only): The rate limit bucket

## HTTP Classes

### HttpServer

The `HttpServer` class provides a simple HTTP server implementation.

#### Constructor

```python
HttpServer(base)
```

**Parameters:**
- `base` (Base): The event base to use

**Example:**
```python
import libevent

base = libevent.Base()
http_server = libevent.HttpServer(base)
```

#### Methods

##### `bind(address, port)`

Bind the HTTP server to a specific address and port.

**Parameters:**
- `address` (str): IP address to bind to
- `port` (int): Port number to bind to

**Returns:** None

**Example:**
```python
http_server.bind('0.0.0.0', 8080)
```

##### `accept(fd)`

Accept connections on a pre-existing socket.

**Parameters:**
- `fd` (int): File descriptor of the listening socket

**Returns:** None

##### `set_max_headers_size(size)`

Set the maximum size for HTTP headers.

**Parameters:**
- `size` (int): Maximum header size in bytes

**Returns:** None

##### `set_max_body_size(size)`

Set the maximum size for HTTP request bodies.

**Parameters:**
- `size` (int): Maximum body size in bytes

**Returns:** None

##### `set_allowed_methods(methods)`

Set which HTTP methods are allowed.

**Parameters:**
- `methods` (int): Bitmask of allowed methods

**Returns:** None

##### `set_callback(path, callback, userdata=None)`

Set a callback for a specific URL path.

**Parameters:**
- `path` (str): URL path to handle
- `callback` (callable): Function to call for requests to this path
- `userdata` (object, optional): User data to pass to callback

**Returns:** None

**Example:**
```python
def handle_root(request, userdata):
    request.send_reply(200, "OK", b"Hello, World!")

http_server.set_callback("/", handle_root)
```

##### `del_callback(path)`

Remove a callback for a specific URL path.

**Parameters:**
- `path` (str): URL path to remove

**Returns:** None

##### `set_generic_callback(callback, userdata=None)`

Set a generic callback for all unmatched requests.

**Parameters:**
- `callback` (callable): Function to call for unmatched requests
- `userdata` (object, optional): User data to pass to callback

**Returns:** None

##### `set_timeout(timeout)`

Set the timeout for HTTP connections.

**Parameters:**
- `timeout` (int): Timeout in seconds

**Returns:** None

### HttpRequest

The `HttpRequest` class represents an HTTP request and provides methods to send responses.

#### Methods

##### `send_error(code, reason)`

Send an HTTP error response.

**Parameters:**
- `code` (int): HTTP status code
- `reason` (str): Reason phrase

**Returns:** None

**Example:**
```python
def handle_request(request, userdata):
    request.send_error(404, "Not Found")
```

##### `send_reply(code, reason, data)`

Send a complete HTTP response.

**Parameters:**
- `code` (int): HTTP status code
- `reason` (str): Reason phrase
- `data` (bytes): Response body

**Returns:** None

**Example:**
```python
def handle_request(request, userdata):
    html = b"<html><body><h1>Hello, World!</h1></body></html>"
    request.send_reply(200, "OK", html)
```

##### `send_reply_start(code, reason)`

Start sending a chunked HTTP response.

**Parameters:**
- `code` (int): HTTP status code
- `reason` (str): Reason phrase

**Returns:** None

##### `send_reply_chunk(data)`

Send a chunk of data in a chunked response.

**Parameters:**
- `data` (bytes): Chunk data

**Returns:** None

##### `send_reply_end()`

End a chunked HTTP response.

**Returns:** None

**Example:**
```python
def handle_request(request, userdata):
    request.send_reply_start(200, "OK")
    request.send_reply_chunk(b"Hello, ")
    request.send_reply_chunk(b"World!")
    request.send_reply_end()
```

#### Properties

- `command` (str, read-only): HTTP method (GET, POST, etc.)
- `uri` (str, read-only): Request URI
- `input_headers` (dict, read-only): Request headers
- `output_headers` (dict): Response headers (can be modified)
- `input_buffer` (Buffer, read-only): Request body buffer
- `output_buffer` (Buffer): Response body buffer
- `remote_host` (str, read-only): Client IP address
- `remote_port` (int, read-only): Client port

### Listener

The `Listener` class provides a connection listener for accepting incoming connections.

#### Constructor

```python
Listener(base, callback, userdata=None, flags=0, backlog=-1, address=None)
```

**Parameters:**
- `base` (Base): The event base to use
- `callback` (callable): Function to call when accepting connections
- `userdata` (object, optional): User data to pass to callback
- `flags` (int, optional): Listener flags
- `backlog` (int, optional): Listen backlog
- `address` (tuple, optional): Address to bind to (host, port)

**Example:**
```python
def accept_callback(listener, fd, address, userdata):
    print(f"New connection from {address}")
    # Handle the new connection

base = libevent.Base()
listener = libevent.Listener(base, accept_callback, address=('0.0.0.0', 8080))
```

#### Methods

##### `enable()`

Enable the listener to accept connections.

**Returns:** None

##### `disable()`

Disable the listener from accepting connections.

**Returns:** None

#### Properties

- `base` (Base, read-only): The event base this listener is assigned to
- `callback` (callable, read-only): The callback function
- `userdata` (object, read-only): The user data
- `fd` (int, read-only): The file descriptor

## Constants and Flags

### Event Types

These constants define the types of events that can be monitored:

- `EV_TIMEOUT`: Timer events
- `EV_READ`: Read events (data available for reading)
- `EV_WRITE`: Write events (ready for writing)
- `EV_SIGNAL`: Signal events
- `EV_PERSIST`: Make the event persistent (don't remove after triggering)
- `EV_ET`: Edge-triggered events (requires backend support)

**Example:**
```python
# Monitor for read events that persist
evt = libevent.Event(base, fd, libevent.EV_READ | libevent.EV_PERSIST, callback)
```

### Backend Features

These constants indicate features supported by the event backend:

- `EV_FEATURE_ET`: Edge-triggered events supported
- `EV_FEATURE_O1`: O(1) operation complexity
- `EV_FEATURE_FDS`: File descriptor events supported

**Example:**
```python
base = libevent.Base()
if base.features & libevent.EV_FEATURE_ET:
    print("Edge-triggered events supported")
```

### BufferEvent Events

These constants are used with BufferEvent callbacks:

- `BEV_EVENT_READING`: Error occurred while reading
- `BEV_EVENT_WRITING`: Error occurred while writing
- `BEV_EVENT_EOF`: End of file reached
- `BEV_EVENT_ERROR`: Generic error occurred
- `BEV_EVENT_TIMEOUT`: Timeout occurred
- `BEV_EVENT_CONNECTED`: Connection established

**Example:**
```python
def event_callback(bev, events, userdata):
    if events & libevent.BEV_EVENT_CONNECTED:
        print("Connected successfully")
    elif events & libevent.BEV_EVENT_ERROR:
        print("Connection error")
    elif events & libevent.BEV_EVENT_EOF:
        print("Connection closed")
```

### BufferEvent Options

These constants control BufferEvent behavior:

- `BEV_OPT_CLOSE_ON_FREE`: Close underlying socket when BufferEvent is freed
- `BEV_OPT_THREADSAFE`: Enable thread-safe operations
- `BEV_OPT_DEFER_CALLBACKS`: Defer callbacks until next event loop iteration
- `BEV_OPT_UNLOCK_CALLBACKS`: Release locks during callback execution

**Example:**
```python
bev = libevent.BufferEvent(base, fd, libevent.BEV_OPT_CLOSE_ON_FREE | libevent.BEV_OPT_THREADSAFE)
```

### Rate Limiting

- `EV_RATE_LIMIT_MAX`: Maximum rate limit value

### Listener Options

These constants control Listener behavior:

- `LEV_OPT_LEAVE_SOCKETS_BLOCKING`: Don't set accepted sockets to non-blocking
- `LEV_OPT_CLOSE_ON_FREE`: Close listening socket when Listener is freed
- `LEV_OPT_CLOSE_ON_EXEC`: Set close-on-exec flag on listening socket
- `LEV_OPT_REUSEABLE`: Set SO_REUSEADDR on listening socket
- `LEV_OPT_THREADSAFE`: Enable thread-safe operations

### Available Methods

The `METHODS` list contains the available event notification mechanisms:

```python
import libevent
print("Available methods:", libevent.METHODS)
# Output might be: ['select', 'poll', 'epoll', 'kqueue']
```

## Utility Functions

### Module-Level Functions

##### `enable_debug_mode()`

Enable debug mode for libevent.

**Returns:** None

##### `socket_get_error(fd)`

Get the socket error for a file descriptor.

**Parameters:**
- `fd` (int): File descriptor

**Returns:** int - Error code

##### `socket_error_to_string(error)`

Convert a socket error code to a string.

**Parameters:**
- `error` (int): Error code

**Returns:** str - Error description

##### `set_log_callback(callback)`

Set a custom logging callback.

**Parameters:**
- `callback` (callable): Logging function

**Returns:** None

##### `set_fatal_callback(callback)`

Set a custom fatal error callback.

**Parameters:**
- `callback` (callable): Fatal error function

**Returns:** None

## Examples

### Basic TCP Server

```python
import socket
import libevent

def handle_client(evt, fd, what, server_socket):
    if what & libevent.EV_READ:
        try:
            client_socket, address = server_socket.accept()
            client_socket.setblocking(False)
            print(f"Connection from {address}")
            
            # Create event for client
            client_evt = libevent.Event(evt.base, client_socket.fileno(), 
                                      libevent.EV_READ, handle_client_data, client_socket)
            client_evt.add()
            
        except socket.error:
            pass

def handle_client_data(evt, fd, what, client_socket):
    if what & libevent.EV_READ:
        try:
            data = client_socket.recv(1024)
            if data:
                print(f"Received: {data.decode('utf-8', errors='ignore')}")
                client_socket.send(b"Echo: " + data)
            else:
                # Client disconnected
                client_socket.close()
                evt.delete()
        except socket.error:
            client_socket.close()
            evt.delete()

def main():
    # Create event base
    base = libevent.Base()
    
    # Create server socket
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind(('localhost', 8080))
    server.listen(5)
    server.setblocking(False)
    
    print("Server listening on localhost:8080")
    
    # Create event for server socket
    server_evt = libevent.Event(base, server.fileno(), libevent.EV_READ | libevent.EV_PERSIST, 
                               handle_client, server)
    server_evt.add()
    
    # Run event loop
    base.loop()

if __name__ == '__main__':
    main()
```

### HTTP Server Example

```python
import libevent

def handle_request(request, userdata):
    # Get request information
    method = request.command
    uri = request.uri
    headers = request.input_headers
    
    print(f"{method} {uri}")
    
    if uri == '/':
        # Serve homepage
        html = b"""
        <html>
        <head><title>Python-libevent HTTP Server</title></head>
        <body>
            <h1>Welcome!</h1>
            <p>This is a simple HTTP server built with python-libevent.</p>
            <a href="/about">About</a>
        </body>
        </html>
        """
        request.output_headers['Content-Type'] = 'text/html'
        request.send_reply(200, "OK", html)
        
    elif uri == '/about':
        # Serve about page
        html = b"""
        <html>
        <head><title>About - Python-libevent HTTP Server</title></head>
        <body>
            <h1>About</h1>
            <p>This server demonstrates the HTTP capabilities of python-libevent.</p>
            <a href="/">Home</a>
        </body>
        </html>
        """
        request.output_headers['Content-Type'] = 'text/html'
        request.send_reply(200, "OK", html)
        
    else:
        # 404 Not Found
        request.send_error(404, "Not Found")

def main():
    base = libevent.Base()
    
    # Create HTTP server
    http_server = libevent.HttpServer(base)
    http_server.bind('0.0.0.0', 8080)
    http_server.set_generic_callback(handle_request)
    
    print("HTTP server running on http://localhost:8080")
    base.loop()

if __name__ == '__main__':
    main()
```

### BufferEvent Client Example

```python
import socket
import libevent

class HTTPClient:
    def __init__(self, base):
        self.base = base
        self.bev = None
        
    def connect(self, host, port):
        # Create socket
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.setblocking(False)
        
        # Create BufferEvent
        self.bev = libevent.BufferEvent(self.base, sock.fileno(), 
                                       libevent.BEV_OPT_CLOSE_ON_FREE)
        self.bev.set_callbacks(self.on_read, self.on_write, self.on_event)
        self.bev.enable(libevent.EV_READ | libevent.EV_WRITE)
        
        # Connect
        try:
            sock.connect((host, port))
        except socket.error as e:
            if e.errno != 115:  # EINPROGRESS
                raise
    
    def on_read(self, bev, userdata):
        data = bev.read()
        print("Received:")
        print(data.decode('utf-8', errors='ignore'))
        
        # Stop the event loop after receiving response
        self.base.loopbreak()
    
    def on_write(self, bev, userdata):
        # Send HTTP request
        request = b"GET / HTTP/1.1\r\nHost: httpbin.org\r\nConnection: close\r\n\r\n"
        bev.write(request)
        bev.disable(libevent.EV_WRITE)
    
    def on_event(self, bev, events, userdata):
        if events & libevent.BEV_EVENT_CONNECTED:
            print("Connected!")
        elif events & libevent.BEV_EVENT_ERROR:
            print("Connection error")
            self.base.loopbreak()
        elif events & libevent.BEV_EVENT_EOF:
            print("Connection closed by server")

def main():
    base = libevent.Base()
    client = HTTPClient(base)
    client.connect('httpbin.org', 80)
    base.loop()

if __name__ == '__main__':
    main()
```

### Timer and Signal Handling

```python
import signal
import libevent

def timer_callback(timer, userdata):
    print(f"Timer fired! Count: {userdata['count']}")
    userdata['count'] += 1
    
    if userdata['count'] < 5:
        # Re-add timer for another 2 seconds
        timer.add(2.0)
    else:
        print("Timer finished")

def signal_callback(sig_evt, signum, userdata):
    print(f"Received signal {signum}")
    sig_evt.base.loopbreak()

def main():
    base = libevent.Base()
    
    # Create timer
    timer_data = {'count': 0}
    timer = libevent.Timer(base, timer_callback, timer_data)
    timer.add(2.0)  # Fire after 2 seconds
    
    # Create signal handler for SIGINT (Ctrl+C)
    sig_evt = libevent.Signal(base, signal.SIGINT, signal_callback)
    sig_evt.add()
    
    print("Timer will fire 5 times every 2 seconds. Press Ctrl+C to exit.")
    base.loop()
    print("Exiting...")

if __name__ == '__main__':
    main()
```

### Buffer Operations

```python
import libevent

def demonstrate_buffer_operations():
    # Create a buffer
    buffer = libevent.Buffer()
    
    # Add data
    buffer.add(b"Hello, ")
    buffer.add(b"World!")
    buffer.add(b"\r\nSecond line\r\n")
    
    print(f"Buffer length: {buffer.length}")
    
    # Read line by line
    line1 = buffer.readln()
    print(f"First line: {line1}")
    
    line2 = buffer.readln()
    print(f"Second line: {line2}")
    
    # Add more data and demonstrate other operations
    buffer.add(b"Some more data for testing")
    
    # Copy data without removing it
    copied = buffer.copyout(10)
    print(f"Copied data: {copied}")
    
    # Remove data from the beginning
    removed = buffer.remove(10)
    print(f"Removed data: {removed}")
    
    print(f"Remaining buffer length: {buffer.length}")
    
    # Drain remaining data
    buffer.drain(buffer.length)
    print(f"Buffer length after drain: {buffer.length}")

if __name__ == '__main__':
    demonstrate_buffer_operations()
```

## Error Handling

### Exception Handling

Python-libevent operations can raise various Python exceptions:

```python
import libevent
import socket

try:
    base = libevent.Base()
    
    # This might raise an exception if the port is already in use
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.bind(('localhost', 80))  # Privileged port
    
except PermissionError:
    print("Permission denied - cannot bind to privileged port")
except OSError as e:
    print(f"OS error: {e}")
except Exception as e:
    print(f"Unexpected error: {e}")
```

### Socket Error Handling

Use the utility functions to handle socket errors:

```python
import libevent
import socket

def handle_socket_error(fd):
    error_code = libevent.socket_get_error(fd)
    if error_code != 0:
        error_msg = libevent.socket_error_to_string(error_code)
        print(f"Socket error {error_code}: {error_msg}")
        return True
    return False

# Example usage in an event callback
def on_event(bev, events, userdata):
    if events & libevent.BEV_EVENT_ERROR:
        if hasattr(bev, 'fd'):
            handle_socket_error(bev.fd)
```

### Custom Logging

Set up custom logging to debug issues:

```python
import libevent

def log_callback(severity, msg):
    severity_names = {
        0: "DEBUG",
        1: "MSG", 
        2: "WARN",
        3: "ERR"
    }
    print(f"[{severity_names.get(severity, 'UNKNOWN')}] {msg}")

# Enable debug mode and set log callback
libevent.enable_debug_mode()
libevent.set_log_callback(log_callback)
```

## Performance Tips

1. **Use persistent events** when monitoring the same file descriptor repeatedly:
   ```python
   evt = libevent.Event(base, fd, libevent.EV_READ | libevent.EV_PERSIST, callback)
   ```

2. **Enable thread safety** only when needed, as it adds overhead:
   ```python
   bev = libevent.BufferEvent(base, fd, libevent.BEV_OPT_THREADSAFE)
   ```

3. **Use BufferEvent** for network I/O instead of raw Events when possible.

4. **Set appropriate watermarks** for BufferEvent to control memory usage:
   ```python
   bev.set_watermark(libevent.EV_READ, 1024, 65536)  # Read between 1KB and 64KB
   ```

5. **Use edge-triggered events** if supported by your platform for better performance:
   ```python
   if base.features & libevent.EV_FEATURE_ET:
       evt = libevent.Event(base, fd, libevent.EV_READ | libevent.EV_ET, callback)
   ```