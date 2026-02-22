# RakNet Protocol Specification

This document describes the RakNet protocol implementation used in this project.

## Overview

RakNet is a reliable UDP-based networking protocol designed for real-time multiplayer games. It provides:

- Reliable and ordered packet delivery
- Automatic packet fragmentation and reassembly
- Connection management
- Bandwidth optimization
- Low latency

## Connection Flow

### 1. Initial Handshake

```
Client                          Server
  |                               |
  |------ 0x08 ------------------>|  Open Connection Request 1
  |       (4 bytes)               |
  |                               |
  |<----- 0x1A -------------------|  Open Connection Reply 1
  |       (Cookie: port XOR)      |
  |                               |
  |------ 0xA2 ------------------>|  Open Connection Request 2
  |       (4 bytes)               |
  |                               |
  |<----- 0x19 -------------------|  Open Connection Reply 2
  |       (Connection accepted)   |
  |                               |
```

### 2. SA-MP Authentication (Optional)

```
Client                          Server
  |                               |
  |------ 0x88 ------------------>|  Auth Request
  |                               |
  |<----- E3:00 ------------------|  Challenge
  |       (25 bytes)              |
  |                               |
  |------ 0x22 ------------------>|  Login Data
  |       (48 bytes)              |
  |                               |
  |<----- E3:01 ------------------|  Auth Accept
  |       (3 bytes)               |
  |                               |
```

### 3. Game Connection

```
Client                          Server
  |                               |
  |------ 0x8A ------------------>|  Join Request
  |                               |
  |<----- E5 ---------------------|  Player Sync
  |<----- E3:07 ------------------|  Spawn List
  |<----- E3:21 ------------------|  Game Entry Complete
  |                               |
  |<----- Game RPCs --------------|  InitGame, SetSpawnInfo, etc.
  |                               |
```

## Packet Structure

### Data Packet (0x84-0x8D)

```
+--------+--------+--------+--------+
| Packet |   Sequence Number (24)   |
|   ID   |        (Little Endian)   |
+--------+--------+--------+--------+
|                                   |
|     Encapsulated Packets...       |
|                                   |
+-----------------------------------+
```

### Encapsulated Packet

```
+--------+--------+--------+
| Flags  | Length (16 BE)  |
+--------+--------+--------+
|  Message Index (24 LE)  |  (if Reliable)
+--------+--------+--------+
|   Order Index (24 LE)   |  (if Ordered)
+--------+--------+--------+
| Channel|                 |  (if Ordered)
+--------+                 |
|                          |
|      Payload...          |
|                          |
+--------------------------+
```

### Flags Byte

```
Bit 7-5: Reliability Type
  000 = Unreliable
  001 = Unreliable Sequenced
  010 = Reliable
  011 = Reliable Ordered
  100 = Reliable Sequenced

Bit 4: Has Split
Bit 3-0: Reserved
```

### ACK Packet (0xC0)

```
+--------+--------+--------+
| 0xC0   | Count (16 LE)   |
+--------+--------+--------+
| Record |                 |
| Type   |  Sequence (24)  |
+--------+--------+--------+
|     ... more records     |
+--------------------------+
```

### NACK Packet (0xA0)

Same structure as ACK, but with packet ID 0xA0.

## Reliability Types

### Unreliable (0)

- No delivery guarantee
- No ordering guarantee
- Lowest overhead
- Use for: Position updates, voice data

### Unreliable Sequenced (1)

- No delivery guarantee
- Only latest packet is processed
- Use for: Frequent state updates

### Reliable (2)

- Guaranteed delivery
- No ordering guarantee
- Use for: Important events

### Reliable Ordered (3)

- Guaranteed delivery
- Guaranteed order
- Most commonly used
- Use for: Game events, RPCs

### Reliable Sequenced (4)

- Guaranteed delivery
- Only latest packet is processed
- Use for: State synchronization

## Session Management

### Session States

1. **UNCONNECTED** - No connection established
2. **HANDSHAKE_SENT** - Waiting for handshake completion
3. **CONNECTING** - Connection in progress
4. **CONNECTED** - Connection established
5. **IN_GAME** - Game session active

### Session Counters

Each session maintains:

- **SequenceNumber**: Increments for each datagram sent
- **MessageIndex**: Increments for each reliable packet
- **OrderIndex[channel]**: Increments for each ordered packet per channel

These counters MUST be monotonically increasing and NEVER reset during the session lifetime.

## MTU (Maximum Transmission Unit)

Default MTU: **576 bytes**

This is the maximum size of a single UDP packet. Larger payloads must be split into multiple packets.

Safe payload sizes:
- Reliable Ordered: 501 bytes
- Reliable: 505 bytes

## Timing

### Intervals

- **ACK Send**: 50ms
- **Keepalive**: 5 seconds
- **Timeout**: 30 seconds
- **Retry Delay**: 100ms (exponential backoff)

### Retransmission

Packets are retransmitted if:
1. No ACK received within timeout
2. NACK received
3. Maximum 5 retries before disconnection

## Error Handling

### Packet Loss

- Detected via sequence number gaps
- NACK sent for missing packets
- Automatic retransmission

### Out of Order

- Packets buffered until in-order
- OrderIndex used for reordering
- Per-channel ordering

### Duplicate Packets

- Detected via sequence number
- Silently discarded
- ACK still sent

## Performance Considerations

### Bandwidth Optimization

1. **Batching**: Multiple small packets combined into one datagram
2. **Compression**: Optional payload compression
3. **Selective ACK**: Only ACK received packets
4. **Delayed ACK**: Batch ACKs to reduce overhead

### Latency Optimization

1. **Immediate Send**: Critical packets sent immediately
2. **Priority Queue**: High-priority packets sent first
3. **Congestion Control**: Adaptive send rate

## Security

### Port Obfuscation

The 0x1A packet uses XOR encoding for the client port:
```
encoded_hi = (port >> 8) ^ 0x82
encoded_lo = (port & 0xFF) ^ 0x93
```

### Session Validation

- Each session has unique sequence numbers
- Packets from wrong session are rejected
- Timeout for inactive sessions

## Debugging

### Packet Logging

Enable detailed logging:
```go
logger.SetLevel(logger.LevelDebug)
```

### Common Issues

1. **Counter Reset**: Ensure counters never reset
2. **Wrong Reliability**: Use Reliable Ordered for most game packets
3. **MTU Exceeded**: Split large packets
4. **ACK Timeout**: Check network latency

## References

- [RakNet Documentation](http://www.jenkinssoftware.com/)
- [SA-MP Protocol](https://sampwiki.blast.hk/)
- [UDP Best Practices](https://gafferongames.com/post/udp_vs_tcp/)

## Version History

- **v1.0.0** - Initial implementation
  - Basic RakNet protocol
  - Reliable ordered delivery
  - Session management
  - ACK/NACK system
