# Python-libevent Tutorial

This tutorial will guide you through the basics of using python-libevent for event-driven programming.

## Getting Started

### What is libevent?

libevent is a high-performance event notification library that provides a mechanism to execute a callback function when a specific event occurs on a file descriptor or after a timeout has been reached.

### Why Use python-libevent?

- **High Performance**: Uses the most efficient event notification mechanism available on your platform (epoll, kqueue, etc.)
- **Scalable**: Can handle thousands of simultaneous connections
- **Cross-Platform**: Works on Linux, macOS, Windows, and other Unix-like systems
- **Event-Driven**: Perfect for network servers, clients, and other I/O-intensive applications

## Installation

### Step 1: Install libevent

First, you need to install the libevent library itself:

#### Ubuntu/Debian:
```bash
sudo apt-get update
sudo apt-get install libevent-dev
```

#### CentOS/RHEL/Fedora:
```bash
# CentOS/RHEL
sudo yum install libevent-devel

# Fedora
sudo dnf install libevent-devel
```

#### macOS:
```bash
# Using Homebrew
brew install libevent

# Using MacPorts
sudo port install libevent
```

#### Windows:
Download and build libevent from source or use a pre-built package.

### Step 2: Set Environment Variable

Set the `LIBEVENT_ROOT` environment variable to point to your libevent installation:

```bash
# Linux/macOS (add to ~/.bashrc or ~/.zshrc)
export LIBEVENT_ROOT=/usr/local

# Or if installed via package manager
export LIBEVENT_ROOT=/usr
```

### Step 3: Install python-libevent

```bash
# Clone the repository
git clone https://github.com/joachim-bauch/python-libevent.git
cd python-libevent

# Build and install
python setup.py build
python setup.py install
```

## Your First Event-Driven Program

Let's start with a simple timer example:

```python
import libevent

def timer_callback(timer, userdata):
    print("Hello from timer!")
    timer.base.loopbreak()  # Exit the event loop

# Create an event base
base = libevent.Base()
print(f"Using {base.method} backend")

# Create a timer that fires after 2 seconds
timer = libevent.Timer(base, timer_callback)
timer.add(2.0)

# Run the event loop
print("Starting event loop...")
base.loop()
print("Event loop finished!")
```

## Building a Simple Echo Server

Now let's build a more practical example - an echo server:

```python
import socket
import libevent

class EchoServer:
    def __init__(self, host='localhost', port=8080):
        self.host = host
        self.port = port
        self.base = libevent.Base()
        self.clients = {}
        
    def start(self):
        # Create server socket
        server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        server_socket.bind((self.host, self.port))
        server_socket.listen(10)
        server_socket.setblocking(False)
        
        print(f"Echo server listening on {self.host}:{self.port}")
        
        # Create event for accepting connections
        accept_event = libevent.Event(
            self.base, 
            server_socket.fileno(), 
            libevent.EV_READ | libevent.EV_PERSIST,
            self.accept_callback,
            server_socket
        )
        accept_event.add()
        
        # Run the event loop
        try:
            self.base.loop()
        except KeyboardInterrupt:
            print("\nShutting down server...")
        finally:
            server_socket.close()
            
    def accept_callback(self, event, fd, what, server_socket):
        try:
            client_socket, address = server_socket.accept()
            client_socket.setblocking(False)
            print(f"New connection from {address}")
            
            # Create event for client data
            client_event = libevent.Event(
                self.base,
                client_socket.fileno(),
                libevent.EV_READ,
                self.client_callback,
                client_socket
            )
            client_event.add()
            
            # Store client info
            self.clients[client_socket.fileno()] = {
                'socket': client_socket,
                'event': client_event,
                'address': address
            }
            
        except socket.error:
            pass
            
    def client_callback(self, event, fd, what, client_socket):
        try:
            data = client_socket.recv(1024)
            if data:
                print(f"Received from {self.clients[fd]['address']}: {data.decode('utf-8', errors='ignore').strip()}")
                # Echo the data back
                client_socket.send(b"Echo: " + data)
                
                # Re-add the event for more data
                event.add()
            else:
                # Client disconnected
                print(f"Client {self.clients[fd]['address']} disconnected")
                self.cleanup_client(fd)
                
        except socket.error:
            print(f"Error with client {self.clients[fd]['address']}")
            self.cleanup_client(fd)
            
    def cleanup_client(self, fd):
        if fd in self.clients:
            self.clients[fd]['socket'].close()
            del self.clients[fd]

if __name__ == '__main__':
    server = EchoServer()
    server.start()
```

Test your server:
```bash
# In one terminal, run the server
python echo_server.py

# In another terminal, connect with telnet
telnet localhost 8080
```

## Using BufferEvent for Better Performance

BufferEvent provides automatic buffering and is more efficient for network I/O:

```python
import socket
import libevent

class BufferedEchoServer:
    def __init__(self, host='localhost', port=8080):
        self.host = host
        self.port = port
        self.base = libevent.Base()
        
    def start(self):
        # Create listening socket
        server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        server_socket.bind((self.host, self.port))
        server_socket.listen(10)
        server_socket.setblocking(False)
        
        print(f"Buffered echo server listening on {self.host}:{self.port}")
        
        # Create listener
        listener = libevent.Listener(
            self.base,
            self.accept_callback,
            address=(self.host, self.port)
        )
        
        try:
            self.base.loop()
        except KeyboardInterrupt:
            print("\nShutting down server...")
        finally:
            server_socket.close()
            
    def accept_callback(self, listener, fd, address, userdata):
        print(f"New connection from {address}")
        
        # Create BufferEvent for the client
        bev = libevent.BufferEvent(self.base, fd, libevent.BEV_OPT_CLOSE_ON_FREE)
        bev.set_callbacks(self.read_callback, None, self.event_callback, address)
        bev.enable(libevent.EV_READ)
        
    def read_callback(self, bev, address):
        # Read all available data
        data = bev.read()
        if data:
            print(f"Received from {address}: {data.decode('utf-8', errors='ignore').strip()}")
            # Echo back with a prefix
            bev.write(b"Echo: " + data)
            
    def event_callback(self, bev, events, address):
        if events & libevent.BEV_EVENT_EOF:
            print(f"Client {address} disconnected")
        elif events & libevent.BEV_EVENT_ERROR:
            print(f"Error with client {address}")

if __name__ == '__main__':
    server = BufferedEchoServer()
    server.start()
```

## Building an HTTP Client

Here's how to create a simple HTTP client:

```python
import socket
import libevent

class SimpleHTTPClient:
    def __init__(self):
        self.base = libevent.Base()
        self.response_data = b""
        
    def get(self, url):
        # Parse URL (simplified)
        if url.startswith('http://'):
            url = url[7:]
        
        if '/' in url:
            host, path = url.split('/', 1)
            path = '/' + path
        else:
            host, path = url, '/'
            
        if ':' in host:
            host, port = host.split(':')
            port = int(port)
        else:
            port = 80
            
        print(f"Connecting to {host}:{port}")
        
        # Create socket
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.setblocking(False)
        
        # Create BufferEvent
        self.bev = libevent.BufferEvent(
            self.base, 
            sock.fileno(),
            libevent.BEV_OPT_CLOSE_ON_FREE
        )
        self.bev.set_callbacks(self.on_read, self.on_write, self.on_event)
        self.bev.enable(libevent.EV_READ | libevent.EV_WRITE)
        
        # Store request info
        self.host = host
        self.path = path
        
        # Connect
        try:
            sock.connect((host, port))
        except socket.error as e:
            if e.errno != 115:  # EINPROGRESS
                raise
                
        # Run event loop
        self.base.loop()
        return self.response_data
        
    def on_write(self, bev, userdata):
        # Send HTTP request
        request = f"GET {self.path} HTTP/1.1\r\n"
        request += f"Host: {self.host}\r\n"
        request += "Connection: close\r\n"
        request += "\r\n"
        
        bev.write(request.encode('utf-8'))
        bev.disable(libevent.EV_WRITE)
        
    def on_read(self, bev, userdata):
        data = bev.read()
        self.response_data += data
        
    def on_event(self, bev, events, userdata):
        if events & libevent.BEV_EVENT_CONNECTED:
            print("Connected!")
        elif events & libevent.BEV_EVENT_EOF:
            print("Connection closed by server")
            self.base.loopbreak()
        elif events & libevent.BEV_EVENT_ERROR:
            print("Connection error")
            self.base.loopbreak()

# Usage
if __name__ == '__main__':
    client = SimpleHTTPClient()
    response = client.get('http://httpbin.org/get')
    print("Response:")
    print(response.decode('utf-8', errors='ignore'))
```

## Working with Timers

Timers are useful for implementing timeouts, periodic tasks, and delays:

```python
import libevent
import time

class TimerDemo:
    def __init__(self):
        self.base = libevent.Base()
        self.counter = 0
        
    def run(self):
        # One-shot timer
        oneshot_timer = libevent.Timer(self.base, self.oneshot_callback, "one-shot")
        oneshot_timer.add(1.0)  # Fire after 1 second
        
        # Periodic timer
        self.periodic_timer = libevent.Timer(self.base, self.periodic_callback, "periodic")
        self.periodic_timer.add(2.0)  # First fire after 2 seconds
        
        # Timeout timer
        timeout_timer = libevent.Timer(self.base, self.timeout_callback, "timeout")
        timeout_timer.add(10.0)  # Stop everything after 10 seconds
        
        print("Starting timer demo...")
        self.start_time = time.time()
        self.base.loop()
        
    def oneshot_callback(self, timer, userdata):
        elapsed = time.time() - self.start_time
        print(f"[{elapsed:.1f}s] {userdata} timer fired!")
        
    def periodic_callback(self, timer, userdata):
        elapsed = time.time() - self.start_time
        self.counter += 1
        print(f"[{elapsed:.1f}s] {userdata} timer fired! (count: {self.counter})")
        
        # Re-schedule for another 2 seconds
        timer.add(2.0)
        
    def timeout_callback(self, timer, userdata):
        elapsed = time.time() - self.start_time
        print(f"[{elapsed:.1f}s] {userdata} - stopping demo")
        self.base.loopbreak()

if __name__ == '__main__':
    demo = TimerDemo()
    demo.run()
```

## Signal Handling

Handle Unix signals gracefully:

```python
import signal
import libevent

class SignalDemo:
    def __init__(self):
        self.base = libevent.Base()
        self.running = True
        
    def run(self):
        # Handle SIGINT (Ctrl+C)
        sigint_handler = libevent.Signal(self.base, signal.SIGINT, self.signal_callback, "SIGINT")
        sigint_handler.add()
        
        # Handle SIGTERM
        sigterm_handler = libevent.Signal(self.base, signal.SIGTERM, self.signal_callback, "SIGTERM")
        sigterm_handler.add()
        
        # Periodic timer to show we're alive
        timer = libevent.Timer(self.base, self.heartbeat, None)
        timer.add(1.0)
        
        print("Signal demo running. Press Ctrl+C to exit gracefully.")
        self.base.loop()
        print("Clean shutdown completed.")
        
    def signal_callback(self, sig_event, signum, signal_name):
        print(f"\nReceived {signal_name} (signal {signum})")
        print("Performing graceful shutdown...")
        self.running = False
        sig_event.base.loopbreak()
        
    def heartbeat(self, timer, userdata):
        if self.running:
            print(".", end="", flush=True)
            timer.add(1.0)

if __name__ == '__main__':
    demo = SignalDemo()
    demo.run()
```

## Best Practices

### 1. Always Use Non-Blocking Sockets

```python
import socket
import libevent

# Good
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.setblocking(False)

# Bad - will block the event loop
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
# sock.setblocking(False)  # Missing!
```

### 2. Handle Exceptions in Callbacks

```python
def safe_callback(event, fd, what, userdata):
    try:
        # Your callback logic here
        handle_event(fd, what, userdata)
    except Exception as e:
        print(f"Error in callback: {e}")
        # Clean up resources
        cleanup_resources(fd)
```

### 3. Use Persistent Events When Appropriate

```python
# Good - for events that should trigger multiple times
accept_event = libevent.Event(
    base, server_fd, 
    libevent.EV_READ | libevent.EV_PERSIST,  # Note EV_PERSIST
    accept_callback, server_socket
)

# Bad - for one-time events, don't use EV_PERSIST
client_event = libevent.Event(
    base, client_fd,
    libevent.EV_READ,  # No EV_PERSIST for one-time reads
    client_callback, client_socket
)
```

### 4. Clean Up Resources

```python
class Server:
    def __init__(self):
        self.base = libevent.Base()
        self.events = []
        self.sockets = []
        
    def cleanup(self):
        # Delete events
        for event in self.events:
            event.delete()
        self.events.clear()
        
        # Close sockets
        for sock in self.sockets:
            sock.close()
        self.sockets.clear()
        
    def __del__(self):
        self.cleanup()
```

### 5. Use BufferEvent for Network I/O

BufferEvent is generally more efficient and easier to use than raw Events for network operations:

```python
# Preferred for network I/O
bev = libevent.BufferEvent(base, fd)
bev.set_callbacks(read_cb, write_cb, event_cb)
bev.enable(libevent.EV_READ | libevent.EV_WRITE)

# Use raw Events for file I/O or special cases
event = libevent.Event(base, fd, libevent.EV_READ, callback)
```

## Debugging Tips

### Enable Debug Mode

```python
import libevent

# Enable debug output
libevent.enable_debug_mode()

# Set custom log callback
def debug_log(severity, message):
    levels = {0: "DEBUG", 1: "MSG", 2: "WARN", 3: "ERR"}
    print(f"[{levels.get(severity, 'UNK')}] {message}")

libevent.set_log_callback(debug_log)
```

### Check Backend Features

```python
base = libevent.Base()
print(f"Backend: {base.method}")
print(f"Features: {base.features}")

if base.features & libevent.EV_FEATURE_ET:
    print("Edge-triggered events supported")
if base.features & libevent.EV_FEATURE_O1:
    print("O(1) backend")
```

### Monitor Event Loop Status

```python
def monitor_loop(base):
    print(f"Loop exited: {base.got_exit()}")
    print(f"Loop broken: {base.got_break()}")
```

## Next Steps

Now that you understand the basics, you can:

1. Build more complex servers and clients
2. Integrate with web frameworks
3. Add SSL/TLS support
4. Implement custom protocols
5. Build high-performance network applications

For more advanced usage, refer to the complete API documentation and explore the examples in the samples directory.