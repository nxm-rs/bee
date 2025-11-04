# Unified Client Protocol Design

## Overview

This design consolidates 5 separate protocols into a single unified "Client" protocol stream, dramatically reducing stream overhead and mutex contention while providing a cleaner semantic model for Swarm operations.

## Current Architecture Problems

### Multiple Separate Protocols

**Data Protocols:**
- `pushsync` (1.3.1) - PUT operation: deliver chunk to network
- `retrieval` (1.4.0) - GET operation: fetch chunk from network

**Accounting Protocols:**
- `pricing` (1.0.0) - Announce payment thresholds
- `pseudosettle` (1.0.0) - Lightweight settlement (Payment/PaymentAck)
- `swap` (1.0.0) - Token-based settlement (EmitCheque/Handshake)

### Issues with Current Design

1. **Multiple Stream Overhead**
   - Each protocol requires separate stream establishment
   - 5 different protocol handlers per peer connection
   - libp2p overhead for each stream
   - Header exchange per stream

2. **Mutex Contention in Accounting**
   ```
   PUT chunk → pushsync stream → accounting lock
   GET chunk → retrieval stream → accounting lock
   Settlement → pseudosettle/swap stream → accounting lock
   ```
   - Three separate entry points to accounting mutex
   - Can't atomically combine operations
   - Race conditions in balance updates

3. **Coordination Overhead**
   - Pricing announcements on separate channel
   - Settlement happens independently of data operations
   - Can't easily enforce "pay before receive" semantics

4. **Semantic Mismatch**
   - Fundamentally: GET and PUT operations on Swarm
   - Pricing/settlement are just the payment layer
   - Should be unified, not separate protocols

## Unified Client Protocol Design

### Protocol Structure

```
Client Protocol (v2.0.0)
├── PUT (write chunk to swarm)
│   ├── Request: Chunk + metadata
│   └── Response: Receipt (success/error)
│
├── GET (read chunk from swarm)
│   ├── Request: Address
│   └── Response: Delivery (chunk/error)
│
├── PRICING (announce payment threshold)
│   └── One-way: PaymentThreshold
│
└── SETTLEMENT (accounting operations)
    ├── PAYMENT (pseudosettle-style)
    ├── PAYMENT_ACK (pseudosettle-style)
    ├── EMIT_CHEQUE (swap-style)
    └── HANDSHAKE (swap beneficiary exchange)
```

### Message Type Hierarchy

```
ClientMessage
├── type: ClientMessageType enum
│   ├── PUT = 0
│   ├── GET = 1
│   ├── PUT_RESPONSE = 2
│   ├── GET_RESPONSE = 3
│   ├── PRICING = 4
│   └── SETTLEMENT = 5
│
└── oneof message
    ├── Put
    ├── Get
    ├── PutResponse
    ├── GetResponse
    ├── Pricing
    └── Settlement
        ├── type: SettlementMessageType enum
        │   ├── PAYMENT = 0
        │   ├── PAYMENT_ACK = 1
        │   ├── EMIT_CHEQUE = 2
        │   └── HANDSHAKE = 3
        │
        └── oneof settlement_message
            ├── Payment
            ├── PaymentAck
            ├── EmitCheque
            └── Handshake
```

## Protobuf Definition

### client.proto

```protobuf
syntax = "proto3";

package swarm.client;

import "common.proto";

option go_package = "pb";

// ClientMessageType defines the type of client message
enum ClientMessageType {
  PUT = 0;           // Write chunk to swarm
  GET = 1;           // Read chunk from swarm
  PUT_RESPONSE = 2;  // Response to PUT request
  GET_RESPONSE = 3;  // Response to GET request
  PRICING = 4;       // Payment threshold announcement
  SETTLEMENT = 5;    // Accounting settlement operations
}

// ClientMessage is the top-level message wrapper for the unified client protocol
message ClientMessage {
  ClientMessageType type = 1;

  oneof message {
    Put put = 2;
    Get get = 3;
    PutResponse put_response = 4;
    GetResponse get_response = 5;
    Pricing pricing = 6;
    Settlement settlement = 7;
  }
}

// Put writes a chunk to the swarm (replaces pushsync Delivery)
message Put {
  swarm.common.Chunk chunk = 1;
}

// PutResponse is the response to a Put request (replaces pushsync Receipt)
message PutResponse {
  bytes chunk_addr = 1;

  oneof result {
    PutSuccess success = 2;
    swarm.common.Error error = 3;
  }
}

// PutSuccess contains the receipt for a successful chunk storage
message PutSuccess {
  bytes signature = 1;
  bytes nonce = 2;
  uint32 storage_radius = 3;
}

// Get requests a chunk from the swarm (replaces retrieval Request)
message Get {
  bytes chunk_addr = 1;
}

// GetResponse is the response to a Get request (replaces retrieval Delivery)
message GetResponse {
  bytes chunk_addr = 1;

  oneof result {
    swarm.common.Chunk chunk = 2;
    swarm.common.Error error = 3;
  }
}

// Pricing announces the payment threshold (replaces pricing AnnouncePaymentThreshold)
// This is typically sent when opening a client connection
message Pricing {
  bytes payment_threshold = 1;
}

// SettlementMessageType defines the type of settlement operation
enum SettlementMessageType {
  PAYMENT = 0;      // Pseudosettle payment
  PAYMENT_ACK = 1;  // Pseudosettle acknowledgment
  EMIT_CHEQUE = 2;  // Swap cheque emission
  HANDSHAKE = 3;    // Swap beneficiary handshake
}

// Settlement handles all accounting operations (replaces pseudosettle + swap)
message Settlement {
  SettlementMessageType type = 1;

  oneof settlement_message {
    Payment payment = 2;
    PaymentAck payment_ack = 3;
    EmitCheque emit_cheque = 4;
    Handshake handshake = 5;
  }
}

// Payment represents a pseudosettle payment
message Payment {
  bytes amount = 1;  // big.Int encoded as bytes
}

// PaymentAck acknowledges a pseudosettle payment
message PaymentAck {
  bytes amount = 1;      // big.Int encoded as bytes
  int64 timestamp = 2;
}

// EmitCheque represents a swap cheque emission
message EmitCheque {
  bytes cheque = 1;  // Serialized cheque
}

// Handshake exchanges swap beneficiary information
message Handshake {
  bytes beneficiary = 1;  // Beneficiary address
}
```

## Benefits of Unified Design

### 1. Single Stream Per Peer

**Before:**
```
Peer A ←→ Peer B
  Stream 1: pushsync
  Stream 2: retrieval
  Stream 3: pricing
  Stream 4: pseudosettle
  Stream 5: swap
```

**After:**
```
Peer A ←→ Peer B
  Stream 1: client (all operations)
```

**Savings:**
- 80% reduction in active streams
- Single header exchange per connection
- Reduced libp2p overhead
- Simpler connection management

### 2. Unified Accounting Lock

**Before:**
```go
// Three separate entry points
func (ps *PushSync) handler() {
    // ...
    ps.accounting.Credit(peer, amount)  // Lock acquisition #1
}

func (s *Service) handler() {  // Retrieval
    // ...
    s.accounting.Debit(peer, amount)    // Lock acquisition #2
}

func (s *Service) handler() {  // Settlement
    // ...
    s.accounting.Settle(peer, amount)   // Lock acquisition #3
}
```

**After:**
```go
// Single entry point with batching opportunity
func (s *ClientService) handler() {
    for {
        msg := readClientMessage()

        // Process message and accumulate accounting changes
        accountingOps := make([]AccountingOp, 0)

        switch msg.Type {
        case PUT:
            // Process PUT, add credit op
            accountingOps = append(accountingOps, CreditOp{...})
        case GET:
            // Process GET, add debit op
            accountingOps = append(accountingOps, DebitOp{...})
        case SETTLEMENT:
            // Process settlement, add settle op
            accountingOps = append(accountingOps, SettleOp{...})
        }

        // Single lock acquisition for batch
        s.accounting.ExecuteBatch(peer, accountingOps)  // Lock once
    }
}
```

**Benefits:**
- **80%+ reduction in lock acquisitions** for typical workloads
- Opportunity for **batching** multiple operations
- **Atomic operations** (e.g., "PUT if balance sufficient")
- Better cache locality

### 3. Better Semantics

**Client's Perspective:**
```go
// Open client connection to peer
client := swarm.NewClient(peer)

// Unified interface
client.Put(chunk)           // Write
chunk := client.Get(addr)   // Read
client.Settle(amount)       // Pay

// No need to manage 5 different protocol streams
```

**Protocol Clarity:**
- GET/PUT are industry-standard terminology
- Pricing + Settlement clearly grouped as "accounting"
- Matches user mental model

### 4. Atomic Operations

**New Capabilities Enabled:**

```go
// Pay-and-fetch atomically
msg := &ClientMessage{
    Type: SETTLEMENT_AND_GET,  // Could be added later
    Settlement: &Settlement{...},
    Get: &Get{...},
}

// Ensures payment before data delivery
// Prevents race conditions
```

**Without unified protocol:**
- Must send settlement on one stream
- Then request chunk on another stream
- Peer could deliver before payment arrives
- Coordination overhead

### 5. Pricing at Connection Time

```go
// When peer connects, immediately announce pricing
func (s *ClientService) ConnectIn(peer p2p.Peer) {
    threshold := s.pricer.GetPaymentThreshold()

    s.sendMessage(&ClientMessage{
        Type: PRICING,
        Pricing: &Pricing{
            PaymentThreshold: threshold,
        },
    })
}

// Client knows pricing before first operation
// No separate pricing protocol stream needed
```

## Implementation Strategy

### Phase 1: Protocol Definition (Week 1)

1. Create `pkg/p2p/protobuf/common/common.proto`
2. Create `pkg/client/pb/client.proto`
3. Generate Go code
4. Review and refine message structures

### Phase 2: Unified Handler (Week 2)

```go
// pkg/client/client.go
type Service struct {
    // Consolidated dependencies
    accounting  accounting.Interface
    pricer      pricer.Interface
    storer      storage.ChunkStore
    topology    topology.Driver
    // ... other fields
}

func (s *Service) Protocol() p2p.ProtocolSpec {
    return p2p.ProtocolSpec{
        Name:    "client",
        Version: "2.0.0",
        StreamSpecs: []p2p.StreamSpec{
            {
                Name:    "client",
                Handler: s.handler,
            },
        },
    }
}

func (s *Service) handler(ctx context.Context, peer p2p.Peer, stream p2p.Stream) error {
    w, r := protobuf.NewWriterAndReader(stream)

    // Single accounting lock for entire session
    accumulatedOps := make([]AccountingOp, 0, 16)
    defer func() {
        if len(accumulatedOps) > 0 {
            s.accounting.ExecuteBatch(peer.Address, accumulatedOps)
        }
    }()

    for {
        var msg pb.ClientMessage
        if err := r.ReadMsgWithContext(ctx, &msg); err != nil {
            if errors.Is(err, io.EOF) {
                break
            }
            return err
        }

        switch msg.Type {
        case pb.PUT:
            s.handlePut(ctx, peer, w, msg.GetPut(), &accumulatedOps)
        case pb.GET:
            s.handleGet(ctx, peer, w, msg.GetGet(), &accumulatedOps)
        case pb.PRICING:
            s.handlePricing(ctx, peer, msg.GetPricing())
        case pb.SETTLEMENT:
            s.handleSettlement(ctx, peer, w, msg.GetSettlement(), &accumulatedOps)
        default:
            return fmt.Errorf("unknown message type: %v", msg.Type)
        }
    }

    return nil
}
```

### Phase 3: Migration Path (Week 3)

**Backward Compatibility:**

```go
// Register both old and new protocols
func (n *Node) setupProtocols() {
    // Legacy protocols (v1)
    n.p2p.AddProtocol(pushsync.Protocol())
    n.p2p.AddProtocol(retrieval.Protocol())
    n.p2p.AddProtocol(pricing.Protocol())
    n.p2p.AddProtocol(pseudosettle.Protocol())
    n.p2p.AddProtocol(swap.Protocol())

    // New unified protocol (v2)
    n.p2p.AddProtocol(client.Protocol())
}

// Client prefers v2, falls back to v1
func (s *Syncer) fetchChunk(peer swarm.Address, addr swarm.Address) {
    if s.supportsClientProtocol(peer) {
        return s.clientGet(peer, addr)
    } else {
        return s.legacyRetrievalGet(peer, addr)
    }
}
```

### Phase 4: Testing (Week 4)

1. Unit tests for message serialization
2. Handler tests for each message type
3. Accounting batch tests
4. Integration tests with mixed v1/v2 peers
5. Performance benchmarks

## Performance Impact Analysis

### Stream Reduction

**Before:**
- 5 protocols × N peers = 5N potential streams
- Average case: ~2-3 active streams per peer (pushsync + retrieval + periodic settlement)
- 100 peers = 200-300 active streams

**After:**
- 1 protocol × N peers = N streams
- Always 1 active stream per peer
- 100 peers = 100 active streams
- **60-66% reduction in active streams**

### Lock Contention

**Metric: Lock acquisitions per 1000 operations**

Current (worst case):
```
1000 PUT operations:
  - 1000 pushsync → 1000 accounting locks
  - 1000 credits to different peers → 1000 lock acquisitions
Total: 1000 lock acquisitions
```

Unified (with batching):
```
1000 PUT operations:
  - 100 batches (10 ops per batch)
  - 100 accounting locks
Total: 100 lock acquisitions
```

**90% reduction in lock acquisitions** (with batching)

Even without batching:
- Operations to same peer on same stream
- Can batch within stream processing loop
- **Still ~50-70% reduction**

### Memory Usage

**Before:**
```
Per peer:
  - pushsync handler goroutine + stack: ~8KB
  - retrieval handler goroutine + stack: ~8KB
  - pricing handler goroutine + stack: ~8KB
  - pseudosettle handler goroutine + stack: ~8KB
  - swap handler goroutine + stack: ~8KB
Total per peer: ~40KB

100 peers: 4MB
```

**After:**
```
Per peer:
  - client handler goroutine + stack: ~8KB
Total per peer: ~8KB

100 peers: 800KB
```

**80% reduction in handler memory usage**

### CPU Usage

**Saved per Operation:**
- Header exchange: ~50-100µs per stream (libp2p overhead)
- Stream multiplexer overhead: ~10-20µs per message
- Lock contention wait time: Variable, 100-1000µs under load

**Estimate:**
- Light load: 5-10% CPU reduction
- Heavy load: 20-30% CPU reduction (due to lock contention)

## Migration Timeline

### Phase 1: Development (Weeks 1-4)
- Implement unified client protocol
- Maintain backward compatibility with v1 protocols
- Testing and benchmarking

### Phase 2: Alpha Release (Week 5-6)
- Client protocol v2.0.0 available
- Prefer v1 by default
- Opt-in flag: `--enable-unified-client`

### Phase 3: Beta Release (Week 7-10)
- Client protocol v2.0.0 preferred
- Fall back to v1 for incompatible peers
- Monitor metrics

### Phase 4: v1 Deprecation (Week 11-12)
- Announce v1 removal date
- Require v2 support for new peers
- Keep v1 for existing connections

### Phase 5: v1 Removal (Network Milestone)
- Remove v1 protocol handlers
- Unified client protocol only
- Simplified codebase

## Error Handling Improvements

### Structured Errors

```protobuf
// In common.proto
enum ErrorCode {
  UNKNOWN = 0;
  INVALID_CHUNK = 1;
  NOT_FOUND = 2;
  RATE_LIMIT = 3;
  OVERDRAFT = 4;
  OUT_OF_RADIUS = 5;
  INVALID_STAMP = 6;
  SHALLOW_RECEIPT = 7;
  // Settlement errors
  INVALID_CHEQUE = 100;
  INSUFFICIENT_FUNDS = 101;
  INVALID_BENEFICIARY = 102;
}

message Error {
  ErrorCode code = 1;
  string message = 2;
  bytes details = 3;  // Optional error-specific details
}
```

### Error Handling Example

```go
func (s *Service) handlePut(ctx context.Context, peer p2p.Peer, w protobuf.Writer, put *pb.Put, ops *[]AccountingOp) {
    // Validate chunk
    chunk, err := validateChunk(put.GetChunk())
    if err != nil {
        s.sendError(w, pb.ErrorCode_INVALID_CHUNK, err)
        return
    }

    // Check accounting
    balance, err := s.accounting.Balance(peer.Address)
    if err != nil {
        s.sendError(w, pb.ErrorCode_OVERDRAFT, err)
        return
    }

    // Store chunk
    receipt, err := s.storer.Put(ctx, chunk)
    if err != nil {
        if errors.Is(err, storage.ErrOutOfRadius) {
            s.sendError(w, pb.ErrorCode_OUT_OF_RADIUS, err)
        } else {
            s.sendError(w, pb.ErrorCode_UNKNOWN, err)
        }
        return
    }

    // Send success
    s.sendSuccess(w, receipt)

    // Accumulate accounting operation
    *ops = append(*ops, CreditOp{
        Peer:   peer.Address,
        Amount: calculatePrice(chunk),
    })
}

func (s *Service) sendError(w protobuf.Writer, code pb.ErrorCode, err error) {
    w.WriteMsgWithContext(ctx, &pb.ClientMessage{
        Type: pb.PUT_RESPONSE,
        Message: &pb.ClientMessage_PutResponse{
            PutResponse: &pb.PutResponse{
                Result: &pb.PutResponse_Error{
                    Error: &common.Error{
                        Code:    code,
                        Message: err.Error(),
                    },
                },
            },
        },
    })
}
```

## Metrics and Observability

### New Metrics

```go
// Unified protocol metrics
var (
    clientMessageTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "client_messages_total",
            Help: "Total number of client messages by type",
        },
        []string{"type"},
    )

    clientMessageDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "client_message_duration_seconds",
            Help: "Duration of client message processing by type",
        },
        []string{"type"},
    )

    accountingBatchSize = prometheus.NewHistogram(
        prometheus.HistogramOpts{
            Name: "accounting_batch_size",
            Help: "Number of operations in accounting batches",
        },
    )

    accountingLockWaitTime = prometheus.NewHistogram(
        prometheus.HistogramOpts{
            Name: "accounting_lock_wait_seconds",
            Help: "Time spent waiting for accounting lock",
        },
    )
)
```

### Dashboard Queries

```promql
# Message rate by type
rate(client_messages_total[5m])

# P99 latency by operation
histogram_quantile(0.99,
  rate(client_message_duration_seconds_bucket[5m])
)

# Average batch size (indicator of contention reduction)
rate(accounting_batch_size_sum[5m]) /
rate(accounting_batch_size_count[5m])

# Lock contention improvement
rate(accounting_lock_wait_seconds_sum[5m])
```

## Security Considerations

### Rate Limiting

```go
// Per-peer rate limiter for each message type
type ClientService struct {
    rateLimiters map[swarm.Address]*MessageRateLimiter
}

type MessageRateLimiter struct {
    put        *rate.Limiter  // e.g., 100 PUT/sec
    get        *rate.Limiter  // e.g., 1000 GET/sec
    settlement *rate.Limiter  // e.g., 10 settlement/sec
}

func (s *Service) checkRateLimit(peer swarm.Address, msgType pb.ClientMessageType) error {
    limiter := s.getRateLimiter(peer)

    switch msgType {
    case pb.PUT:
        if !limiter.put.Allow() {
            return p2p.NewBlockPeerError(time.Minute, "PUT rate limit exceeded")
        }
    case pb.GET:
        if !limiter.get.Allow() {
            return p2p.NewBlockPeerError(time.Minute, "GET rate limit exceeded")
        }
    // ...
    }
    return nil
}
```

### Message Size Limits

```go
const (
    maxChunkSize     = 4096 + 128  // Data + metadata
    maxChequeSize    = 512
    maxMessageSize   = 8192         // Conservative limit
)

func (s *Service) validateMessageSize(msg *pb.ClientMessage) error {
    size := proto.Size(msg)
    if size > maxMessageSize {
        return fmt.Errorf("message too large: %d > %d", size, maxMessageSize)
    }
    return nil
}
```

## Summary

The unified Client protocol provides:

✅ **80% reduction in active streams** (5 protocols → 1)
✅ **90% reduction in lock acquisitions** (with batching)
✅ **80% reduction in handler memory** (40KB → 8KB per peer)
✅ **20-30% CPU reduction** under load (reduced lock contention)
✅ **Better semantics** (GET/PUT instead of pushsync/retrieval)
✅ **Atomic operations** (pay-and-fetch, etc.)
✅ **Simpler codebase** (one handler instead of five)
✅ **Foundation for future optimizations** (stream reuse, pipelining)

This is a significant architectural improvement that addresses fundamental design issues while maintaining backward compatibility during migration.
