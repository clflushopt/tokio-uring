# Stream Processor Design for High-Performance Request-Response Services

**Date:** 2025-11-04
**Status:** Design
**Target:** Single-core stream processing pipeline for req-res services

## Overview

This design describes a high-performance stream processor component for building request-response services on tokio-uring. The processor focuses on the single-core execution model, handling the transformation pipeline from incoming byte streams to processed responses.

**Key Goals:**
- Minimum latency (p99/p999) through CPU affinity and cache efficiency
- Maximum throughput via pipelined execution
- Zero-copy I/O where possible
- Pluggable I/O strategies (recv singleshot, multishot, future zero-copy variants)
- Ergonomic API: users implement `tower::Service`, framework provides fast execution

**Explicitly Out of Scope (for now):**
- Multi-core coordination and connection distribution (SO_REUSEPORT setup)
- Main entry point and server lifecycle management
- Shared state between cores

## Architecture

### Core Principle: Three-Stage Pipeline

The processor executes a three-stage pipeline on a single core, with each stage running as an independent tokio task:

```
┌─────────────┐    VecDeque    ┌─────────────┐    VecDeque    ┌─────────────┐
│   Reader    │───────────────>│  Processor  │───────────────>│   Writer    │
│    Task     │   (requests)   │    Task     │  (responses)   │    Task     │
└─────────────┘                └─────────────┘                └─────────────┘
      │                               │                               │
      v                               v                               v
futures::Stream              tower::Service              futures::Sink
```

**Stage 1 - Reader Task:**
- Consumes from `source: impl futures::Stream<Item=Request>`
- Drains requests into bounded `VecDeque<Request>` (ring buffer)
- Backpressure handling: `tokio::task::yield_now()` on full queue (spin with yield)

**Stage 2 - Processor Task:**
- Drains from request queue
- Invokes `service.call(request).await` (user's `tower::Service` implementation)
- Pushes responses into bounded `VecDeque<Response>`
- Backpressure handling: `tokio::task::yield_now()` on full queue

**Stage 3 - Writer Task:**
- Drains from response queue
- Sends to `sink: impl futures::Sink<Response>`
- Handles `poll_flush` for buffering strategies (Nagle-like)
- Backpressure handling: `tokio::task::yield_now()` on sink not ready

**Concurrency Model:**
- All three tasks spawned via `tokio::spawn_local` (single-threaded executor)
- Cooperative multitasking allows pipeline stages to overlap
- No locks or atomics needed (single core, VecDeque with spin-yield)

### Type System

**User-Facing API:**

Users implement the standard `tower::Service` trait:

```rust
use tower::Service;
use std::future::Future;

impl Service<Request> for MyHandler {
    type Response = Response;
    type Error = MyError;
    type Future = impl Future<Output = Result<Self::Response, Self::Error>>;

    fn call(&mut self, req: Request) -> Self::Future {
        async move {
            // Business logic here
            Ok(response)
        }
    }
}
```

This gives users access to the entire Tower ecosystem (middleware, utilities, documentation).

**Framework Core Types:**

```rust
/// The three-stage processor that wires up the pipeline
pub struct Processor<Src, Svc, Snk> {
    source: Src,        // impl futures::Stream<Item=Request>
    service: Svc,       // impl tower::Service<Request, Response=Response>
    sink: Snk,          // impl futures::Sink<Response>
    config: ProcessorConfig,
}

/// Configuration for processor behavior
pub struct ProcessorConfig {
    request_queue_size: usize,   // Reader -> Processor queue capacity
    response_queue_size: usize,  // Processor -> Writer queue capacity
}

impl<Src, Svc, Snk> Processor<Src, Svc, Snk>
where
    Src: futures::Stream + Unpin,
    Svc: tower::Service<Src::Item> + Clone,
    Snk: futures::Sink<Svc::Response> + Unpin,
{
    pub fn new(source: Src, service: Svc, sink: Snk, config: ProcessorConfig) -> Self {
        Self { source, service, sink, config }
    }

    /// Run the processor (spawns three tasks, returns on completion/error)
    pub async fn run(self) -> Result<(), ProcessorError> {
        // Spawn reader, processor, writer tasks
        // Join all three
    }
}
```

**Adapter Types:**

The framework provides adapters to integrate with tokio-uring and tokio-util:

```rust
/// Wraps a tokio_uring::net::TcpStream with a codec to produce Stream<Item=Request>
pub struct StreamSource<S, C> {
    stream: S,          // tokio_uring::net::TcpStream or similar
    codec: C,           // Decoder<Item=Request>
    buffer: BytesMut,   // Read buffer
}

impl<S, C> futures::Stream for StreamSource<S, C>
where
    S: /* io-uring read operations */,
    C: Decoder<Item=Request>,
{
    type Item = Result<Request, CodecError>;

    fn poll_next(&mut self, cx: &mut Context) -> Poll<Option<Self::Item>> {
        // Read from stream, decode frames, yield requests
    }
}

/// Wraps a tokio_uring::net::TcpStream with a codec to provide Sink<Item=Response>
pub struct SinkAdapter<S, C> {
    stream: S,          // tokio_uring::net::TcpStream or similar
    codec: C,           // Encoder<Item=Response>
    buffer: BytesMut,   // Write buffer
}

impl<S, C> futures::Sink<Response> for SinkAdapter<S, C>
where
    S: /* io-uring write operations */,
    C: Encoder<Item=Response>,
{
    type Error = CodecError;

    fn poll_send(&mut self, cx: &mut Context, item: Response) -> Poll<Result<(), Self::Error>> {
        // Encode response, write to stream
    }

    fn poll_flush(&mut self, cx: &mut Context) -> Poll<Result<(), Self::Error>> {
        // Flush buffered data
    }
}
```

## Composition and Middleware

### Decorator Pattern for Source/Sink

Since sources and sinks implement `futures::Stream` and `futures::Sink`, we can use standard Rust combinator/wrapper patterns:

**Deadline Wrapper:**
```rust
pub struct DeadlineSource<S> {
    inner: S,
    timeout: Duration,
}

impl<S: Stream> Stream for DeadlineSource<S> {
    type Item = Result<S::Item, DeadlineError>;

    fn poll_next(&mut self, cx: &mut Context) -> Poll<Option<Self::Item>> {
        // Wrap inner poll_next with tokio::time::timeout
    }
}
```

**Buffered Sink (Nagle-like):**
```rust
pub struct BufferedSink<S: Sink> {
    inner: S,
    buffer: Vec<S::Item>,
    max_size: usize,           // Flush when buffer reaches this size
    flush_timer: Interval,      // Flush when timer fires
}

impl<S: Sink> Sink for BufferedSink<S> {
    type Item = S::Item;
    type Error = S::Error;

    fn poll_send(&mut self, cx: &mut Context, item: Self::Item) -> Poll<Result<(), Self::Error>> {
        self.buffer.push(item);

        // Flush if: (1) buffer full OR (2) timer expired
        if self.buffer.len() >= self.max_size || self.flush_timer.poll_tick(cx).is_ready() {
            self.poll_flush(cx)
        } else {
            Poll::Ready(Ok(()))
        }
    }

    fn poll_flush(&mut self, cx: &mut Context) -> Poll<Result<(), Self::Error>> {
        // Drain buffer to inner sink
        while let Some(item) = self.buffer.pop() {
            ready!(self.inner.poll_send(cx, item))?;
        }
        self.inner.poll_flush(cx)
    }
}
```

**SQE Limiter:**
```rust
pub struct SqeLimitedSource<S> {
    inner: S,
    max_outstanding: usize,
    current_outstanding: usize,
}

// Tracks outstanding io-uring submissions, applies backpressure when limit reached
```

### Composition API

Provide extension traits for ergonomic chaining:

```rust
pub trait SourceExt: Stream {
    fn with_deadline(self, timeout: Duration) -> DeadlineSource<Self>;
    fn with_sqe_limit(self, limit: usize) -> SqeLimitedSource<Self>;
}

pub trait SinkExt: Sink {
    fn with_buffer(self, size: usize, max_delay: Duration) -> BufferedSink<Self>;
}

// Usage:
let source = StreamSource::new(tcp_stream, codec)
    .with_deadline(Duration::from_millis(100))
    .with_sqe_limit(64);

let sink = SinkAdapter::new(tcp_stream, codec)
    .with_buffer(1024, Duration::from_micros(50));
```

## Runtime I/O Strategy Selection

Support multiple io-uring strategies, selectable at runtime based on kernel capabilities:

```rust
pub enum IoStrategy {
    RecvSingleShot,    // Standard single-shot recv operations
    RecvMultiShot,     // Multishot recv (kernel 6.0+)
    // Future: ZeroCopyRecv, ZeroCopySend
}

pub struct IoStrategyDetector;

impl IoStrategyDetector {
    /// Probe kernel for io-uring capabilities
    pub fn detect() -> IoStrategy {
        // Check kernel version, probe io-uring features
        // Return best available strategy
    }
}

/// Factory to create sources with appropriate strategy
pub fn create_tcp_source<C>(
    stream: tokio_uring::net::TcpStream,
    codec: C,
    strategy: IoStrategy,
) -> Box<dyn Stream<Item = Result<C::Item, CodecError>>>
where
    C: Decoder + 'static,
{
    match strategy {
        IoStrategy::RecvSingleShot => Box::new(SingleShotSource::new(stream, codec)),
        IoStrategy::RecvMultiShot => Box::new(MultiShotSource::new(stream, codec)),
    }
}
```

This allows the framework to adapt to available kernel features without user code changes.

## Memory Management

### Per-Core Allocation Strategy

Since the processor runs on a single core with ordered request/response processing, we can use efficient allocation strategies:

**Bump Allocator for Request/Response Lifecycles:**

```rust
/// Per-core arena for request/response allocation
pub struct ProcessorArena {
    bump: bumpalo::Bump,
}

impl ProcessorArena {
    /// Reset arena after batch of requests processed
    pub fn reset(&mut self) {
        self.bump.reset();
    }
}
```

**Usage Pattern:**
1. Reader task allocates parsed requests from arena
2. Requests flow through pipeline
3. After responses written, bulk-reset arena
4. No individual deallocations needed

**Benefits:**
- Cache-friendly sequential allocation
- Zero deallocation overhead during processing
- Bulk reset is O(1)
- Works well with ordered stream processing

**Integration with futures::Stream:**

Arena allocation is compatible with ownership-based Stream/Sink model:
- Parse bytes → allocate Request from arena
- Process → Response allocated from arena
- Serialize → write bytes, Response drops
- Periodic batch reset when safe (e.g., every N requests or when queues drain)

## Zero-Copy Considerations (Future)

**Current Approach:** Pragmatic copy-on-parse/serialize
- io-uring recv → bytes in buffer
- Parse → allocate Request from arena (copy)
- Service processes Request → produces Response
- Serialize Response → write buffer (copy)
- io-uring send → kernel sends buffer

**Future Zero-Copy Path:**

When kernel support and requirements justify:
1. **Registered buffer pools:** Pre-register buffers with io-uring
2. **Buffer handles:** Stream yields buffer IDs instead of owned data
3. **Custom traits:** Define lending/streaming variants that don't require ownership
4. **Selective use:** Apply only to hot paths, keep futures::Stream for control plane

**Migration Strategy:**
- Start with futures::Stream/Sink for ecosystem compatibility
- Measure allocation overhead in real workloads
- If bottleneck identified, add zero-copy variants as alternative implementations
- Keep both paths: zero-copy for data plane, futures traits for composability

## Error Handling

**Error Categories:**

1. **I/O Errors:** Socket read/write failures, timeouts
2. **Codec Errors:** Malformed frames, protocol violations
3. **Service Errors:** User handler errors (from `tower::Service::Error`)
4. **Pipeline Errors:** Queue overflow (shouldn't happen with backpressure), task panics

**Error Propagation:**

```rust
pub enum ProcessorError {
    SourceError(Box<dyn Error + Send>),
    ServiceError(Box<dyn Error + Send>),
    SinkError(Box<dyn Error + Send>),
    PipelineShutdown,
}

impl Processor {
    pub async fn run(self) -> Result<(), ProcessorError> {
        // Any stage error causes graceful shutdown
        // Drain queues, close connections, return error
    }
}
```

**Graceful Shutdown:**
1. On error in any stage, signal other stages to stop
2. Drain queues (process pending requests)
3. Flush sink (send pending responses)
4. Return error to caller

## Performance Characteristics

**Expected Properties:**

- **Latency:** Low p99/p999 due to:
  - CPU affinity (no context switches across cores)
  - Minimal allocations (bump allocator)
  - Spin-yield backpressure (no blocking syscalls)
  - Pipeline parallelism (stages overlap)

- **Throughput:** Maximized by:
  - Pipeline parallelism (reader/processor/writer concurrent)
  - io-uring batch submissions/completions
  - Efficient memory layout (sequential arena allocations)
  - Nagle-like buffering on sink (reduced syscalls)

- **Resource Usage:**
  - Fixed memory per core (queue sizes + arena capacity)
  - No dynamic allocations in hot path
  - Bounded SQE usage via limiter decorators

**Tuning Knobs:**
- Queue sizes (reader→processor, processor→writer)
- Arena capacity and reset frequency
- Buffering parameters (size, timeout)
- SQE limits

## Dependencies

**Required Crates:**
- `tower` (v0.4+): Service trait
- `futures` (v0.3+): Stream/Sink traits
- `tokio-uring` (current): Runtime and io-uring operations
- `bytes` (v1.0+): Efficient byte buffer management
- `bumpalo` (v3.0+): Bump allocator for arenas

**Optional/Future:**
- Codec abstractions (custom or from `tokio-util`)
- Metrics/tracing integration

## Example Usage

```rust
use tower::Service;
use futures::{Stream, Sink};

// User implements their service
struct EchoService;

impl Service<Request> for EchoService {
    type Response = Response;
    type Error = Infallible;
    type Future = Ready<Result<Response, Infallible>>;

    fn call(&mut self, req: Request) -> Self::Future {
        ready(Ok(Response::echo(req)))
    }
}

// Framework wires up the pipeline
let tcp_stream = /* tokio_uring::net::TcpStream */;
let strategy = IoStrategyDetector::detect();

let source = create_tcp_source(tcp_stream.clone(), MyCodec, strategy)
    .with_deadline(Duration::from_millis(100))
    .with_sqe_limit(64);

let sink = create_tcp_sink(tcp_stream, MyCodec)
    .with_buffer(1024, Duration::from_micros(50));

let processor = Processor::new(
    source,
    EchoService,
    sink,
    ProcessorConfig {
        request_queue_size: 256,
        response_queue_size: 256,
    },
);

// Run on a single core
processor.run().await?;
```

## Open Questions

1. **Arena reset frequency:** Should reset be time-based, count-based, or both?
2. **Codec abstraction:** Use tokio-util's codec traits or define custom?
3. **Error recovery:** Should processor support per-request error recovery, or fail-fast?
4. **Metrics:** How to integrate observability without sacrificing performance?
5. **Connection lifecycle:** How does processor signal when source exhausted (connection closed)?

## Future Extensions

1. **Advanced io-uring features:**
   - Zero-copy send/recv when kernel support matures
   - Fixed file descriptors for reduced overhead
   - Registered buffers for kernel-userspace efficiency

2. **Additional decorators:**
   - Rate limiting (token bucket per source)
   - Circuit breaker (fail fast on service errors)
   - Retry logic (with exponential backoff)

3. **Multi-protocol support:**
   - HTTP/1.1, HTTP/2, gRPC codec implementations
   - Pluggable framing strategies

4. **Observability:**
   - Per-stage latency metrics
   - Queue depth monitoring
   - Allocation tracking

## Summary

This design provides a high-performance, ergonomic stream processor for single-core request-response services:

- **User simplicity:** Implement `tower::Service`, framework handles execution
- **Composability:** futures::Stream/Sink enable ecosystem integration
- **Performance:** Three-stage pipeline, arena allocation, spin-yield backpressure
- **Flexibility:** Runtime io-uring strategy selection, decorator-based composition
- **Future-proof:** Clear path to zero-copy optimizations when needed

The design prioritizes pragmatic performance (arena allocation, pipelining) over premature zero-copy optimization, while keeping the door open for future enhancements.
