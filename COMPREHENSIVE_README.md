# Python-libevent: Complete Documentation

Welcome to the comprehensive documentation for python-libevent, Python bindings for the high-performance libevent library.

## 📚 Documentation Overview

This documentation package includes:

- **[API_DOCUMENTATION.md](API_DOCUMENTATION.md)** - Complete API reference with all classes, methods, and constants
- **[TUTORIAL.md](TUTORIAL.md)** - Step-by-step tutorial for getting started
- **[Original README](README.markdown)** - Basic project information

## 🚀 Quick Start

### Installation

1. Install libevent development libraries:
   ```bash
   # Ubuntu/Debian
   sudo apt-get install libevent-dev
   
   # CentOS/RHEL
   sudo yum install libevent-devel
   
   # macOS
   brew install libevent
   ```

2. Set environment variable:
   ```bash
   export LIBEVENT_ROOT=/usr/local  # or /usr for package manager installs
   ```

3. Build and install python-libevent:
   ```bash
   python setup.py build
   python setup.py install
   ```

### Hello World Example

```python
import libevent

def timer_callback(timer, userdata):
    print("Hello, World!")
    timer.base.loopbreak()

base = libevent.Base()
timer = libevent.Timer(base, timer_callback)
timer.add(2.0)  # Fire after 2 seconds
base.loop()
```

## 📖 Core Concepts

### Event Base
The central dispatcher that manages all events:
```python
base = libevent.Base()
print(f"Using {base.method} backend")  # epoll, kqueue, select, etc.
```

### Events
Monitor file descriptors for I/O readiness:
```python
event = libevent.Event(base, fd, libevent.EV_READ, callback, userdata)
event.add()
```

### BufferEvent
High-level buffered I/O for network programming:
```python
bev = libevent.BufferEvent(base, socket_fd)
bev.set_callbacks(read_cb, write_cb, event_cb)
bev.enable(libevent.EV_READ | libevent.EV_WRITE)
```

### Timers
Execute callbacks after specified delays:
```python
timer = libevent.Timer(base, callback, userdata)
timer.add(5.0)  # Fire after 5 seconds
```

## 🏗️ Architecture

Python-libevent provides Python bindings for libevent's core functionality:

```
┌─────────────────────────────────────────┐
│              Python Application         │
├─────────────────────────────────────────┤
│           python-libevent               │
│  ┌─────────┐ ┌──────────┐ ┌───────────┐ │
│  │  Base   │ │  Event   │ │   Timer   │ │
│  └─────────┘ └──────────┘ └───────────┘ │
│  ┌─────────┐ ┌──────────┐ ┌───────────┐ │
│  │ Buffer  │ │BufferEvt │ │HttpServer │ │
│  └─────────┘ └──────────┘ └───────────┘ │
├─────────────────────────────────────────┤
│              libevent (C)               │
│     ┌─────────────────────────────┐     │
│     │    Platform Event Loop      │     │
│     │  (epoll/kqueue/select/...)  │     │
│     └─────────────────────────────┘     │
└─────────────────────────────────────────┘
```

## 🌟 Key Features

- **High Performance**: Uses the most efficient event mechanism available (epoll, kqueue, etc.)
- **Cross-Platform**: Works on Linux, macOS, Windows, and other Unix-like systems
- **Scalable**: Handle thousands of concurrent connections
- **Non-Blocking**: All I/O operations are non-blocking
- **Thread-Safe**: Optional thread-safety for multi-threaded applications

## 📋 API Reference

### Core Classes

| Class | Purpose | Key Methods |
|-------|---------|-------------|
| `Base` | Event dispatcher | `loop()`, `loopbreak()`, `dispatch()` |
| `Event` | File descriptor events | `add()`, `delete()` |
| `Timer` | Timer events | `add(timeout)` |
| `Signal` | Unix signal handling | `add()` |
| `Buffer` | Growable byte buffer | `add()`, `remove()`, `readln()` |
| `BufferEvent` | Buffered I/O | `read()`, `write()`, `set_callbacks()` |
| `HttpServer` | HTTP server | `bind()`, `set_callback()` |
| `Listener` | Connection listener | `enable()`, `disable()` |

### Constants

| Category | Constants |
|----------|-----------|
| Event Types | `EV_READ`, `EV_WRITE`, `EV_SIGNAL`, `EV_TIMEOUT`, `EV_PERSIST` |
| Backend Features | `EV_FEATURE_ET`, `EV_FEATURE_O1`, `EV_FEATURE_FDS` |
| BufferEvent Events | `BEV_EVENT_CONNECTED`, `BEV_EVENT_EOF`, `BEV_EVENT_ERROR` |
| BufferEvent Options | `BEV_OPT_CLOSE_ON_FREE`, `BEV_OPT_THREADSAFE` |
| Listener Options | `LEV_OPT_REUSEABLE`, `LEV_OPT_CLOSE_ON_FREE` |

## 🔧 Common Use Cases

### TCP Server
```python
import socket
import libevent

def handle_client(evt, fd, what, server_socket):
    client_socket, address = server_socket.accept()
    client_socket.setblocking(False)
    # Handle client...

base = libevent.Base()
server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.bind(('localhost', 8080))
server.listen(5)
server.setblocking(False)

evt = libevent.Event(base, server.fileno(), 
                    libevent.EV_READ | libevent.EV_PERSIST, 
                    handle_client, server)
evt.add()
base.loop()
```

### HTTP Server
```python
import libevent

def handle_request(request, userdata):
    if request.uri == '/':
        request.send_reply(200, "OK", b"Hello, World!")
    else:
        request.send_error(404, "Not Found")

base = libevent.Base()
http_server = libevent.HttpServer(base)
http_server.bind('0.0.0.0', 8080)
http_server.set_generic_callback(handle_request)
base.loop()
```

### Periodic Timer
```python
import libevent

def periodic_task(timer, userdata):
    print("Periodic task executed!")
    timer.add(5.0)  # Re-schedule for 5 seconds

base = libevent.Base()
timer = libevent.Timer(base, periodic_task)
timer.add(5.0)
base.loop()
```

## 🎯 Best Practices

1. **Always use non-blocking sockets**:
   ```python
   sock.setblocking(False)
   ```

2. **Handle exceptions in callbacks**:
   ```python
   def safe_callback(evt, fd, what, userdata):
       try:
           # Your logic here
           pass
       except Exception as e:
           print(f"Error: {e}")
   ```

3. **Use persistent events for repeated triggers**:
   ```python
   evt = libevent.Event(base, fd, libevent.EV_READ | libevent.EV_PERSIST, callback)
   ```

4. **Prefer BufferEvent for network I/O**:
   ```python
   bev = libevent.BufferEvent(base, fd)
   bev.set_callbacks(read_cb, write_cb, event_cb)
   ```

5. **Clean up resources properly**:
   ```python
   evt.delete()
   socket.close()
   ```

## 🐛 Debugging

### Enable Debug Mode
```python
import libevent

libevent.enable_debug_mode()

def log_callback(severity, message):
    print(f"[{severity}] {message}")

libevent.set_log_callback(log_callback)
```

### Check Backend Information
```python
base = libevent.Base()
print(f"Backend: {base.method}")
print(f"Features: {base.features}")
print(f"Available methods: {libevent.METHODS}")
```

## 🔍 Troubleshooting

### Common Issues

1. **"LIBEVENT_ROOT not set"**
   - Set the environment variable: `export LIBEVENT_ROOT=/usr/local`

2. **"Cannot find libevent headers"**
   - Install development packages: `sudo apt-get install libevent-dev`

3. **"Socket operation would block"**
   - Use non-blocking sockets: `sock.setblocking(False)`

4. **"Event loop not responding"**
   - Check for blocking operations in callbacks
   - Use `base.loopbreak()` to exit cleanly

### Performance Issues

1. **High CPU usage**
   - Check for busy loops in callbacks
   - Use appropriate timeouts

2. **Memory leaks**
   - Always call `event.delete()`
   - Close sockets properly

3. **Poor scalability**
   - Use BufferEvent instead of raw Events
   - Enable edge-triggered events if supported

## 📊 Performance Comparison

| Backend | Platform | Performance | Features |
|---------|----------|-------------|----------|
| epoll | Linux | Excellent | Edge-triggered, O(1) |
| kqueue | BSD/macOS | Excellent | Edge-triggered, O(1) |
| select | All | Good | Level-triggered, O(n) |
| poll | Unix | Good | Level-triggered, O(n) |

## 🤝 Contributing

1. Read the existing code and documentation
2. Write tests for new features
3. Follow the existing code style
4. Update documentation as needed
5. Submit pull requests with clear descriptions

## 📄 License

This project is licensed under the GNU Lesser General Public License (LGPL) v2.1.

## 🔗 Related Projects

- [libevent](https://libevent.org/) - The underlying C library
- [Twisted](https://twistedmatrix.com/) - Python event-driven networking engine
- [asyncio](https://docs.python.org/3/library/asyncio.html) - Python's built-in async I/O
- [Tornado](https://www.tornadoweb.org/) - Python web framework and async networking library

## 📞 Support

- **Documentation**: See [API_DOCUMENTATION.md](API_DOCUMENTATION.md) and [TUTORIAL.md](TUTORIAL.md)
- **Examples**: Check the `samples/` directory
- **Issues**: Report bugs and feature requests on the project repository
- **Community**: Join discussions about event-driven programming in Python

---

**Happy event-driven programming with python-libevent!** 🎉