# Bee Protocol Implementations Analysis

## Overview

This document provides a comprehensive analysis of the protocol implementations in the Bee codebase, focusing on:
- Protocol buffer definitions locations
- Current protocol implementations
- Message handling patterns
- Protocol registration and lifecycle
- Error handling patterns
- Stream management

---

## 1. Protocol Buffer Definitions

### Location
All `.proto` files are located under `pkg/[protocol-name]/pb/`:

| Protocol | Location | File |
|----------|----------|------|
| Handshake | `pkg/p2p/libp2p/internal/handshake/pb/` | `handshake.proto` |
| Headers | `pkg/p2p/libp2p/internal/headers/pb/` | `headers.proto` |
| Hive | `pkg/hive/pb/` | `hive.proto` |
| Pingpong | `pkg/pingpong/pb/` | `pingpong.proto` |
| Pricing | `pkg/pricing/pb/` | `pricing.proto` |
| Pullsync | `pkg/pullsync/pb/` | `pullsync.proto` |
| Pushsync | `pkg/pushsync/pb/` | `pushsync.proto` |
| Retrieval | `pkg/retrieval/pb/` | `retrieval.proto` |
| Pseudosettle | `pkg/settlement/pseudosettle/pb/` | `pseudosettle.proto` |
| Status | `pkg/status/internal/pb/` | `status.proto` |
| Swap | `pkg/settlement/swap/swapprotocol/pb/` | `swap.proto` |

### Current Structure

All `.proto` files use:
- **Syntax**: `proto3`
- **Package naming**: Matches protocol name (e.g., `package hive;`, `package pingpong;`)
- **Go package**: All use `option go_package = "pb";`
- **Code generation**: Uses `protoc-gen-gogo` (gogo/protobuf)

---

## 2. Protocol Implementations

### 2.1 Core Architecture

Each protocol follows a consistent pattern:

```go
// Protocol specification with stream definitions
func (s *Service) Protocol() p2p.ProtocolSpec {
    return p2p.ProtocolSpec{
        Name:    protocolName,
        Version: protocolVersion,
        StreamSpecs: []p2p.StreamSpec{
            {
                Name:    streamName,
                Handler: s.handler,
                Headler: s.headler,  // Optional header handler
            },
        },
        ConnectIn:     s.init,      // Optional connection handler (inbound)
        ConnectOut:    s.init,      // Optional connection handler (outbound)
        DisconnectIn:  s.terminate, // Optional disconnect handler (inbound)
        DisconnectOut: s.terminate, // Optional disconnect handler (outbound)
    }
}
```

### 2.2 Handshake Protocol

**Path**: `/home/user/bee/pkg/p2p/libp2p/internal/handshake/`
**Version**: 14.0.0

**Key Features**:
- Initiates p2p connections
- Exchanges peer metadata (overlay address, underlay addresses, network ID)
- Validates peer signatures and network compatibility
- Supports welcome messages (max 140 characters)
- Handles bee 2.6.0 compatibility mode for single underlay address

**Message Types**:
- `Syn`: Observed underlay address
- `Ack`: Address information, network ID, fullNode flag, nonce, welcome message
- `SynAck`: Combined Syn + Ack response

**Key Methods**:
- `Handshake()`: Initiates handshake (outbound)
- `Handle()`: Handles incoming handshake (inbound)
- `SetWelcomeMessage()`: Updates welcome message

**Error Handling**:
- `ErrNetworkIDIncompatible`: Network ID mismatch
- `ErrInvalidAck`: Invalid signature or address
- `ErrWelcomeMessageLength`: Message exceeds 140 chars
- `ErrPicker`: Kademlia rejection

---

### 2.3 Headers Protocol (Internal)

**Path**: `/home/user/bee/pkg/p2p/libp2p/internal/headers/`

**Purpose**: Exchange headers between peers on every stream

**Message Types**:
- `Headers`: Repeated header key-value pairs
- `Header`: Key-value pair (key: string, value: bytes)

**Key Functions**:
- `sendHeaders()`: Send headers and receive peer headers
- `handleHeaders()`: Receive headers and send response headers
- `headersPBToP2P()`: Convert protobuf to p2p.Headers
- `headersP2PToPB()`: Convert p2p.Headers to protobuf

**Usage**: Every protocol stream exchanges headers first (via `addProtocol` stream handler)

---

### 2.4 Hive Protocol

**Path**: `/home/user/bee/pkg/hive/`
**Version**: 1.1.0

**Purpose**: Discovery protocol for peer gossip

**Message Types**:
- `Peers`: Repeated BzzAddress entries
- `BzzAddress`: Overlay, underlay, signature, nonce

**Key Features**:
- Rate limiting (4 * MaxBins burst, 1 minute window)
- Batch processing (max 30 peers per message)
- Concurrent peer validation with semaphore
- Ping-based peer reachability check (15 second timeout)
- Filters advertisable underlays based on CIDR settings

**Key Methods**:
- `BroadcastPeers()`: Send peers to specific address
- `peersHandler()`: Handle incoming peer list
- `checkAndAddPeers()`: Validate and add peers asynchronously

---

### 2.5 Pingpong Protocol

**Path**: `/home/user/bee/pkg/pingpong/`
**Version**: 1.0.0

**Purpose**: Measure round-trip time

**Message Types**:
- `Ping`: Greeting string
- `Pong`: Response string

**Key Methods**:
- `Ping()`: Send ping, receive pong
- `handler()`: Echo incoming ping as pong

**Simple bidirectional exchange pattern**

---

### 2.6 Pricing Protocol

**Path**: `/home/user/bee/pkg/pricing/`
**Version**: 1.0.0

**Purpose**: Exchange payment threshold announcements

**Message Types**:
- `AnnouncePaymentThreshold`: Payment threshold (bytes)

**Key Features**:
- Validates threshold >= minimum threshold
- Triggered on connection (ConnectIn/ConnectOut)
- Notifies observer of payment threshold updates

**Key Methods**:
- `AnnouncePaymentThreshold()`: Send threshold
- `handler()`: Receive and validate threshold

---

### 2.7 Pullsync Protocol

**Path**: `/home/user/bee/pkg/pullsync/`
**Version**: 1.4.0

**Purpose**: Synchronize chunks from upstream peers

**Message Types**:
- `Syn`: Empty synchronization message
- `Ack`: Cursors, epoch
- `Get`: Bin number, start position
- `Chunk`: Address, batch ID, stamp hash
- `Offer`: Topmost position, repeated chunks
- `Want`: Bitvector selection
- `Delivery`: Address, data, stamp

**Key Features**:
- Cursor-based synchronization
- Rate limiting (250 chunks/sec per peer)
- Singleflight for duplicate requests
- Multiple streams (pullsync, cursors)
- Postage stamp validation

**Key Methods**:
- `Sync()`: Synchronize batch starting at position
- `GetCursors()`: Retrieve upstream cursors
- `handler()`: Handle sync requests
- `cursorHandler()`: Handle cursor requests

---

### 2.8 Pushsync Protocol

**Path**: `/home/user/bee/pkg/pushsync/`
**Version**: 1.3.1

**Purpose**: Push chunks toward their closest neighbor

**Message Types**:
- `Delivery`: Address, data, stamp
- `Receipt`: Address, signature, nonce, error string, storage radius

**Key Features**:
- Forwarding to closest peer
- Receipt validation
- Accounting for payment
- Tracing support
- Shallow receipt tolerance (with jitter backoff)
- Skip list for failed peers
- Stabilization check before push

**Key Methods**:
- `PushChunkToClosest()`: Push chunk to network
- `handler()`: Receive push and forward/store
- `pushToClosest()`: Route chunk to closest peer

**Error Handling**:
- `ErrNoPush`: Could not push chunk
- `ErrOutOfDepthStoring`: Outside neighborhood
- `ErrWarmup`: Node warmup incomplete
- `ErrShallowReceipt`: Receipt insufficient

---

### 2.9 Retrieval Protocol

**Path**: `/home/user/bee/pkg/retrieval/`
**Version**: 1.4.0

**Purpose**: Retrieve chunks from network using forwarding-kademlia

**Message Types**:
- `Request`: Chunk address
- `Delivery`: Data, stamp, error string

**Key Features**:
- Timeout-based request with preemptive interval (5 sec)
- Forwarding along kademlia paths
- Caching for forwarders
- Accounting for requests
- Skip list for failed peers
- Singleflight for duplicate requests

**Key Methods**:
- `RetrieveChunk()`: Retrieve from network
- `handler()`: Handle retrieval requests

---

### 2.10 Pseudosettle Protocol

**Path**: `/home/user/bee/pkg/settlement/pseudosettle/`
**Version**: 1.0.0

**Purpose**: Pseudo-settlement for bandwidth accounting

**Message Types**:
- `Payment`: Amount (bytes)
- `PaymentAck`: Amount, timestamp

**Key Features**:
- Payment allowance calculation (time-based)
- Peer-specific locking
- State persistence
- Light node handling
- Timestamp validation
- Multiple error conditions for mismatches

**Key Methods**:
- `handler()`: Handle incoming payment
- `init()`: Connect peer
- `terminate()`: Disconnect peer

**Error Handling**:
- `ErrSettlementTooSoon`: Too frequent settlement
- `ErrTimeOutOfSyncAlleged`: Decreasing timestamps
- `ErrTimeOutOfSyncRecent`: Timestamp drift > 2s
- `ErrTimeOutOfSyncInterval`: Interval drift > 3s

---

### 2.11 Status Protocol

**Path**: `/home/user/bee/pkg/status/`
**Version**: 1.1.3

**Purpose**: Query node status snapshot

**Message Types**:
- `Get`: Empty request
- `Snapshot`: Node state including:
  - ReserveSize, PullsyncRate, StorageRadius
  - ConnectedPeers, NeighborhoodSize
  - BeeMode, BatchCommitment, IsReachable
  - ReserveSizeWithinRadius, LastSyncedBlock, CommittedDepth
  - Custom metrics map

**Key Methods**:
- `LocalSnapshot()`: Build current status
- `handler()`: Handle status requests
- Metrics encoding/decoding for custom metrics

---

### 2.12 Swap Protocol

**Path**: `/home/user/bee/pkg/settlement/swap/swapprotocol/`
**Version**: 1.0.0

**Purpose**: Settlement protocol for cheque exchange

**Message Types**:
- `EmitCheque`: Serialized cheque (JSON)
- `Handshake`: Beneficiary address

**Key Features**:
- Beneficiary exchange on connection
- Cheque signature validation
- Exchange rate negotiation via headers
- Deduction value handling
- Header-based protocol extension

**Key Methods**:
- `EmitCheque()`: Send signed cheque
- `handler()`: Receive cheque
- `headler()`: Provide exchange rates in response headers
- `init()`: Handle connection handshake

---

## 3. Message Serialization/Deserialization

### Core Utilities

**Path**: `/home/user/bee/pkg/p2p/protobuf/`

```go
// Reader and Writer wrappers
type Reader struct {
    ggio.Reader
}

type Writer struct {
    ggio.Writer
}

// Key methods
func (r Reader) ReadMsgWithContext(ctx context.Context, msg proto.Message) error
func (w Writer) WriteMsgWithContext(ctx context.Context, msg proto.Message) error

// Utility functions
func NewReader(r io.Reader) Reader
func NewWriter(w io.Writer) Writer
func NewWriterAndReader(s p2p.Stream) (Writer, Reader)
func ReadMessages(r io.Reader, newMessage func() Message) ([]Message, error)
```

### Serialization Details

- **Format**: Delimited protobuf messages (gogo/protobuf)
- **Max message size**: 128 KB (delimitedReaderMaxSize)
- **Codec**: Uses gogo protobuf v3 with gzip file descriptors

### Usage Pattern

```go
// In protocol handlers
w, r := protobuf.NewWriterAndReader(stream)

// Write message
var msg pb.MessageType
err := w.WriteMsgWithContext(ctx, &msg)

// Read message
var response pb.ResponseType
err := r.ReadMsgWithContext(ctx, &response)
```

---

## 4. Stream Handling

### Stream Lifecycle

1. **Creation**: `p2p.Streamer.NewStream(ctx, address, headers, protocol, version, stream)`
2. **Header Exchange**: Automatic (via AddProtocol)
3. **Handler Execution**: Protocol-specific handler
4. **Closure**: `stream.FullClose()` (graceful) or `stream.Reset()` (abort)

### Stream Interface

```go
type Stream interface {
    io.ReadWriter
    io.Closer
    ResponseHeaders() Headers
    Headers() Headers
    FullClose() error
    Reset() error
}
```

### Key Features

- **Headers Exchange**: Every stream exchanges headers first
  - Initiator sends headers
  - Responder receives, optionally modifies (via Headler)
  - Responder sends response headers
  - Both parties can access headers via `Headers()` / `ResponseHeaders()`

- **Context Management**: 
  - Handlers receive context
  - Headers exchange uses timeout (default 10s)
  - Message reads/writes can timeout via context

- **Error Handling**:
  - `Reset()`: Immediate abort (used on errors)
  - `FullClose()`: Graceful closure (used on success)
  - Ensures proper stream cleanup and EOF handling

---

## 5. Protocol Registration & Lifecycle

### Registration Pattern

```go
// In main setup
p2pService := libp2p.New(...)

// Add protocols
p2pService.AddProtocol(hiveService.Protocol())
p2pService.AddProtocol(pingpongService.Protocol())
// ... etc

// AddProtocol implementation in libp2p:
// 1. Creates stream handlers for each stream in protocol
// 2. Registers with libp2p using semantic version matching
// 3. Sets up protocol lifecycle callbacks (Connect/Disconnect)
// 4. Stores protocol spec for querying
```

### Protocol Lifecycle Callbacks

**On Inbound Connection**:
1. Headers exchanged
2. Tracing context extracted
3. Handler executed
4. ConnectIn callback invoked (first time)

**On Outbound Connection**:
1. Stream created to peer
2. Headers exchanged
3. Handler executed
4. ConnectOut callback invoked (first time)

**On Disconnection**:
1. DisconnectIn called (inbound sessions)
2. DisconnectOut called (outbound sessions)

### Error Handling in AddProtocol

```go
// Special error types handled:
- DisconnectError: Causes peer disconnect
- BlockPeerError: Blocks peer for duration (0 = infinite)
- IncompatibleStreamError: Stream version mismatch
- ErrUnexpected: Increments unexpected request counter
- ErrReset: Increments reset counter
```

---

## 6. Current Error Handling Patterns

### Error Types

**Global P2P Errors** (`pkg/p2p/error.go`):
```go
ErrPeerNotFound
ErrAlreadyConnected
ErrDialLightNode
ErrPeerBlocklisted
```

**Special Error Wrappers**:
- `DisconnectError`: Disconnect peer gracefully
- `BlockPeerError`: Disconnect and blocklist for duration
- `ConnectionBackoffError`: Exponential backoff for retries
- `IncompatibleStreamError`: Stream version incompatibility
- `ChunkDeliveryError`: Chunk delivery failure wrapper

### Protocol-Specific Errors

Each protocol defines its own error variables and constants. Common patterns:

```go
// Handshake
ErrNetworkIDIncompatible
ErrInvalidAck
ErrWelcomeMessageLength
ErrPicker

// Pricing
ErrThresholdTooLow

// Pullsync
ErrUnsolicitedChunk

// Pushsync
ErrNoPush
ErrOutOfDepthStoring
ErrWarmup
ErrShallowReceipt

// Hive
ErrRateLimitExceeded
```

### Error Handling in Handlers

**Common Pattern**:
```go
defer func() {
    if err != nil {
        _ = stream.Reset()  // Abort on error
    } else {
        _ = stream.FullClose()  // Graceful close on success
    }
}()

// Handler logic
var msg pb.MessageType
if err := r.ReadMsgWithContext(ctx, &msg); err != nil {
    return fmt.Errorf("read message: %w", err)
}
```

---

## 7. Protocol-Specific Patterns

### Bidirectional Protocols

**Pingpong**: Ping → Pong (symmetric)
```go
// Client sends ping, reads pong
w.WriteMsgWithContext(ctx, &pb.Ping{Greeting: msg})
r.ReadMsgWithContext(ctx, &pong)

// Server reads ping, sends pong
r.ReadMsgWithContext(ctx, &ping)
w.WriteMsgWithContext(ctx, &pb.Pong{Response: response})
```

### Request/Response Protocols

**Retrieval**: Request → Delivery
**Pullsync**: Get → Offer, Want → Delivery
**Pricing**: Announce (connection-triggered)

### Streaming Protocols

**Hive**: Peers (async broadcast)
**Pushsync**: Delivery → Receipt (error in receipt string)
**Pullsync**: Multiple messages (Syn/Ack, Get/Offer, Want/Delivery)

### Header-Based Metadata

**Swap Protocol**: Exchange rates via headers
```go
func (s *Service) headler(receivedHeaders p2p.Headers, peerAddress swarm.Address) p2p.Headers {
    exchangeRate, deduction, _ := s.priceOracle.CurrentRates()
    return swap.SettlementResponseHeaders(exchangeRate, deduction)
}
```

---

## 8. Key Data Structures

### Protocol Definition
```go
type ProtocolSpec struct {
    Name          string
    Version       string
    StreamSpecs   []StreamSpec
    ConnectIn     func(context.Context, Peer) error
    ConnectOut    func(context.Context, Peer) error
    DisconnectIn  func(Peer) error
    DisconnectOut func(Peer) error
}

type StreamSpec struct {
    Name    string
    Handler HandlerFunc
    Headler HeadlerFunc
}
```

### Peer Information
```go
type Peer struct {
    Address         swarm.Address
    FullNode        bool
    EthereumAddress []byte
}
```

### Headers
```go
type Headers map[string][]byte
```

---

## 9. Testing Patterns

### Stream Testing

**Tool**: `pkg/p2p/streamtest/`

```go
// Create test recorder
recorder := streamtest.New(
    streamtest.WithProtocols(service.Protocol()),
    streamtest.WithBaseAddr(baseAddr),
)

// Use recorder as streamer
client := protocolService.New(recorder, logger, ...)

// Perform operations
client.SomeOperation(ctx)

// Check recorded messages
records, _ := recorder.Records(addr, "protocol", "1.0.0", "stream")
record := records[0]
messages, _ := protobuf.ReadMessages(record.In(), func() protobuf.Message { ... })
```

---

## 10. Summary of Protocol Patterns

### Required Components

For each protocol, implement:

1. **Service struct** with:
   - `streamer p2p.Streamer` or `p2p.StreamerDisconnecter`
   - Protocol-specific state
   - Logger
   - Metrics

2. **Protocol() method** returning `p2p.ProtocolSpec`:
   - Name, Version
   - StreamSpecs with handlers
   - Optional ConnectIn/Out, DisconnectIn/Out

3. **Handler** (HandlerFunc signature):
   - `func(ctx context.Context, peer p2p.Peer, stream p2p.Stream) error`
   - Read/write messages using protobuf.Reader/Writer
   - Return error on failure, nil on success

4. **.proto file** defining message types

5. **Generated code** (via protoc-gen-gogo):
   - Located in `pb/` subdirectory
   - Auto-imported by convention

### Common Methods

- `NewStream()`: Create outbound stream
- `handler()`: Inbound stream handler
- Message exchange via Reader/Writer
- Graceful close with `stream.FullClose()`
- Error abort with `stream.Reset()`

---

## 11. SWIP-27 Implications

SWIP-27 requires "strongly typed protocol buffer messages". Current state:

**Strengths**:
- All messages already use protocol buffers (gogo)
- Type-safe message handling via generated code
- Proper serialization/deserialization

**Current Limitations**:
- Raw bytes used in some messages (e.g., Address, Signature, Data)
- Error strings in response messages (Retrieval, Pushsync)
- Limited type information in headers (map[string][]byte)
- Some protocols mix concerns (e.g., Delivery + error handling)

**Potential Enhancements**:
- Define specific error types in proto instead of strings
- Create separate error response message types
- Use strongly-typed address types
- Define common reusable types (Address, Signature, etc.)
- Separate concerns in message designs

---

## Files Summary

**Core P2P Framework**:
- `/home/user/bee/pkg/p2p/p2p.go`: Interface definitions
- `/home/user/bee/pkg/p2p/error.go`: Error types
- `/home/user/bee/pkg/p2p/protobuf/protobuf.go`: Reader/Writer utilities
- `/home/user/bee/pkg/p2p/libp2p/libp2p.go`: Protocol registration (AddProtocol)
- `/home/user/bee/pkg/p2p/libp2p/stream.go`: Stream implementation
- `/home/user/bee/pkg/p2p/libp2p/headers.go`: Header exchange

**Protocols** (11 total, each with):
- `[protocol]/[protocol].go`: Service implementation
- `[protocol]/pb/[protocol].proto`: Message definitions
- `[protocol]/pb/[protocol].pb.go`: Generated code
- `[protocol]/metrics.go`: Prometheus metrics
- `[protocol]/[protocol]_test.go`: Tests

