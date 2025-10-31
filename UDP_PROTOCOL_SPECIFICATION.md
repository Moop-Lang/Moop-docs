# Moop Inter-Layer UDP Protocol Specification

## Overview

The Moop architecture uses UDP communication **only between L3 (Turchin Actor/Controller) and L4 (Rio Prototypes)**. Layers L0-L2 are bare metal implementations without UDP functionality to maintain lightweight operation on constrained hardware.

## Architecture Context

### Layer Responsibilities
- **L5**: Moop Natural Language - Human-readable syntax
- **L4**: Rio Prototypes - Object-oriented abstractions (**UDP enabled**)
- **L3**: Turchin Actor/Controller - Message-passing concurrency (**UDP enabled**)
- **L2**: Prigogine Functional Model - Mathematical bridge layer (**bare metal**)
- **L1**: McCarthy Reversible Gates - Homoiconic foundation (**bare metal**)
- **L0**: RISC-V Assembly - Hardware target (**bare metal**)

### UDP Interface Rationale
- **L3↔L4 UDP**: Enables modular communication between actor system and prototype system
- **L0-L2 Bare Metal**: Optimized for size and speed on constrained hardware
- **Modularity**: UDP allows L3 and L4 to be developed and tested independently

## Port Assignments

| Layer | Port | Direction | Purpose |
|-------|------|-----------|---------|
| L3 Actor/Controller | 4001 | Listen | Receives prototype instantiation requests |
| L4 Rio Prototypes | 6001 | Listen | Receives actor messages and method calls |

## Message Format

All UDP messages use JSON format with UTF-8 encoding.

### Base Message Structure
```json
{
  "type": "message_type",
  "timestamp": 1234567890,
  "id": "unique_message_id"
}
```

## L3 → L4 Communication

### Actor Request Message
```json
{
  "type": "actor_request",
  "timestamp": 1234567890,
  "id": "msg_001",
  "actor_id": "display_controller",
  "message": "render_output",
  "data": "Hello from L3 Actor!",
  "priority": "normal"
}
```

### Method Call Message
```json
{
  "type": "method_call",
  "timestamp": 1234567890,
  "id": "msg_002",
  "actor_id": "calculator_actor",
  "prototype": "Calculator",
  "method": "add",
  "args": [5, 3],
  "context": {}
}
```

### Event Notification Message
```json
{
  "type": "event_notification",
  "timestamp": 1234567890,
  "id": "msg_003",
  "actor_id": "system_monitor",
  "event": "memory_pressure",
  "data": {
    "level": "warning",
    "usage": 0.85
  }
}
```

## L4 → L3 Communication

### Prototype Instantiation Request
```json
{
  "type": "prototype_instantiation",
  "timestamp": 1234567890,
  "id": "msg_004",
  "prototype": "Calculator",
  "method": "add",
  "args": [5, 3],
  "requester": "user_interface"
}
```

### Response Message
```json
{
  "type": "response",
  "timestamp": 1234567890,
  "id": "msg_005",
  "request_id": "msg_002",
  "status": "success",
  "result": 8,
  "error": null
}
```

### Object State Update
```json
{
  "type": "state_update",
  "timestamp": 1234567890,
  "id": "msg_006",
  "prototype": "Calculator",
  "object_id": "calc_001",
  "state": {
    "last_result": 8,
    "operation_count": 42
  }
}
```

## Message Types Reference

### L3 → L4 Message Types
- `actor_request` - Actor sending message to prototype system
- `method_call` - Direct method invocation on prototype
- `event_notification` - System event from actor layer
- `heartbeat` - Keep-alive message
- `shutdown` - Graceful shutdown request

### L4 → L3 Message Types
- `prototype_instantiation` - Request to create new actor for prototype
- `response` - Response to previous request
- `state_update` - Object state change notification
- `error` - Error condition notification
- `ready` - System ready notification

## Error Handling

### Error Response Format
```json
{
  "type": "error",
  "timestamp": 1234567890,
  "id": "msg_007",
  "request_id": "msg_002",
  "error_code": "PROTOTYPE_NOT_FOUND",
  "error_message": "Calculator prototype not available",
  "details": {
    "available_prototypes": ["String", "Number", "List"]
  }
}
```

### Error Codes
- `PROTOTYPE_NOT_FOUND` - Requested prototype doesn't exist
- `METHOD_NOT_FOUND` - Method not available on prototype
- `INVALID_ARGS` - Method arguments invalid
- `ACTOR_NOT_FOUND` - Target actor doesn't exist
- `TIMEOUT` - Operation timed out
- `SYSTEM_OVERLOAD` - System under heavy load

## Performance Considerations

### Message Size Limits
- Maximum message size: 1024 bytes
- Recommended size: < 512 bytes for optimal performance
- Large data should be chunked or referenced

### Timeout Values
- Standard request timeout: 2 seconds
- Heartbeat interval: 30 seconds
- Connection timeout: 5 seconds

### Reliability
- UDP is unreliable; implement application-level acknowledgments for critical messages
- Use message IDs for correlation and duplicate detection
- Implement exponential backoff for retries

## Security Considerations

### Local Network Only
- All communication on localhost (127.0.0.1)
- No external network access required
- Firewall rules should block external UDP access to these ports

### Message Validation
- Validate JSON structure before processing
- Sanitize all string inputs
- Verify message size limits
- Check for malformed timestamps and IDs

## Implementation Guidelines

### Socket Management
- Use non-blocking UDP sockets
- Implement connection pooling for efficiency
- Handle socket errors gracefully
- Close sockets properly on shutdown

### Message Processing
- Parse JSON safely with error handling
- Validate message types and required fields
- Log all communication for debugging
- Implement message queuing for high throughput

### Testing
- Use mock servers for unit testing
- Test timeout scenarios
- Verify error handling paths
- Performance test under load

## Example Implementation

### L3 Actor System (Python)
```python
import socket
import json
import time

class L3ActorSystem:
    def __init__(self):
        self.socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        self.socket.bind(('127.0.0.1', 4001))
        
    def send_to_l4(self, message):
        data = json.dumps(message).encode('utf-8')
        self.socket.sendto(data, ('127.0.0.1', 6001))
        
    def listen(self):
        while True:
            data, addr = self.socket.recvfrom(1024)
            message = json.loads(data.decode('utf-8'))
            self.handle_message(message)
```

### L4 Rio Prototype System (Python)
```python
import socket
import json

class L4RioPrototypes:
    def __init__(self):
        self.socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        self.socket.bind(('127.0.0.1', 6001))
        
    def send_to_l3(self, message):
        data = json.dumps(message).encode('utf-8')
        self.socket.sendto(data, ('127.0.0.1', 4001))
        
    def listen(self):
        while True:
            data, addr = self.socket.recvfrom(1024)
            message = json.loads(data.decode('utf-8'))
            self.handle_message(message)
```

## Future Enhancements

### Potential Improvements
- Message compression for large payloads
- Encryption for sensitive data
- Message priority queuing
- Load balancing across multiple L4 instances
- Monitoring and metrics collection

### Backwards Compatibility
- Version field in messages for protocol evolution
- Graceful degradation for unknown message types
- Migration path for protocol updates

## References

- [Moop Architecture Documentation](ARCHITECTURE.md)
- [Turchin Actor Model](docs/philosophy/)
- [Rio Prototype System](docs/specifications/)
- [UDP Protocol RFC 768](https://tools.ietf.org/html/rfc768)

---

**Note**: This specification covers only the L3↔L4 UDP interface. Layers L0-L2 use direct function calls and memory operations for maximum performance on bare metal systems. 