# Stage 2 — Topic 3: gRPC (Deep Dive)

## Theory

We covered gRPC's basics in Stage 1 Topic 11. Now we go deep — the internals of Protobuf, how HTTP/2 is used under the hood, all four communication patterns with real code, error handling, interceptors, and exactly how gRPC fits into ShopSphere's internal architecture.

**The core philosophy of gRPC:**
REST asks — *"what resource do you want and what do you want to do to it?"*
gRPC asks — *"what function do you want to call on that remote service?"*

This is **RPC — Remote Procedure Call** — making a call to code on another machine feel like calling a local function. gRPC is Google's open-source implementation of this idea, built for production microservices at massive scale.

**Why Google built it:**
Google runs thousands of microservices making billions of inter-service calls per second. JSON over HTTP/1.1 was too slow, too large, and too loosely typed. They needed something faster, smaller, and with strict contracts enforced at compile time — hence Protobuf + HTTP/2.

---

## Internals — Protocol Buffers (Protobuf)

Protobuf is the serialisation format gRPC uses. Understanding it deeply is what separates a surface-level gRPC answer from a strong one.

### How Protobuf Encodes Data

JSON is text — human readable, self-describing, verbose:
```json
{
  "orderId": "o-789",
  "status": "CONFIRMED",
  "total": 1299.99,
  "itemCount": 3
}
→ ~70 bytes, includes field names in every message
```

Protobuf is binary — compact, schema-dependent, not self-describing:
```
Field 1 (orderId):    type=string, value="o-789"
Field 2 (status):     type=string, value="CONFIRMED"
Field 3 (total):      type=double, value=1299.99
Field 4 (itemCount):  type=int32,  value=3
→ ~25 bytes, field names NOT included — only field numbers
```

**The critical detail — field numbers, not names:**
```protobuf
message OrderResponse {
  string order_id   = 1;   ← field number 1
  string status     = 2;   ← field number 2
  double total      = 3;   ← field number 3
  int32  item_count = 4;   ← field number 4
}
```

The wire format only contains `1: "o-789"`, `2: "CONFIRMED"`, `3: 1299.99`, `4: 3`. The schema (`.proto` file) is needed to decode what field number 1 means. This is why:
- Protobuf messages are 3–10x smaller than JSON
- You can **rename a field** without breaking compatibility — field number stays the same
- You **cannot reuse a field number** — doing so corrupts existing data

### Protobuf Wire Types

Every field is encoded as `(field_number << 3) | wire_type`:

| Wire Type | Meaning | Used For |
|---|---|---|
| 0 | Varint | int32, int64, bool, enum |
| 1 | 64-bit | double, fixed64 |
| 2 | Length-delimited | string, bytes, nested messages, repeated |
| 5 | 32-bit | float, fixed32 |

**Varint encoding — why small integers are tiny:**
```
int32 value 1    → 1 byte  (0x01)
int32 value 300  → 2 bytes (0xAC 0x02)
int32 value 1    vs JSON "itemCount": 1  → 1 byte vs 16 bytes
```

Varint uses variable-length encoding — small numbers use fewer bytes. This is why Protobuf is especially efficient for typical API data where most integers are small.

### Schema Evolution — Forward and Backward Compatibility

This is critical in production microservices — services deploy independently and must remain compatible:

**Safe changes (backward + forward compatible):**
```protobuf
// v1
message OrderRequest {
  string user_id = 1;
  repeated string item_ids = 2;
}

// v2 — added optional field, safe
message OrderRequest {
  string user_id = 1;
  repeated string item_ids = 2;
  string promo_code = 3;       ← new optional field, old clients ignore it
  optional string note = 4;    ← old clients send without it, new clients read it
}
```

**Breaking changes (never do these):**
```protobuf
// NEVER do this:
message OrderRequest {
  string user_id = 1;
  repeated string item_ids = 3;  ← changed field number from 2 to 3 — BREAKS everything
  string promo_code = 2;         ← reused field number 2 — CORRUPTS data
}
```

**Deprecating a field safely:**
```protobuf
message OrderRequest {
  string user_id = 1;
  reserved 2;                    ← field 2 is reserved, cannot be reused
  reserved "item_ids";           ← field name also reserved
  repeated OrderItem items = 3;  ← new field at new number
}
```

---

## Internals — How gRPC Uses HTTP/2

gRPC maps RPC calls onto HTTP/2 streams:

```
HTTP/2 Connection
  │
  ├── Stream 1: OrderService.CreateOrder
  │     Headers: POST /OrderService/CreateOrder
  │              content-type: application/grpc
  │     Data:    [length-prefixed protobuf bytes]
  │     Trailers: grpc-status: 0 (OK)
  │
  ├── Stream 3: InventoryService.CheckStock (concurrent)
  │     Headers: POST /InventoryService/CheckStock
  │     Data:    [protobuf bytes]
  │     Trailers: grpc-status: 0
  │
  └── Stream 5: NotificationService.StreamUpdates (server streaming)
        Headers: POST /NotificationService/StreamUpdates
        Data:    [protobuf frame 1]
                 [protobuf frame 2]
                 [protobuf frame 3]  ← server keeps writing
        Trailers: grpc-status: 0    ← only sent when stream ends
```

**gRPC message framing:**
```
Each message is prefixed with a 5-byte header:
  Byte 0:   Compression flag (0 = not compressed, 1 = compressed)
  Bytes 1-4: Message length (big-endian uint32)
  Bytes 5+: Protobuf-encoded message body
```

**gRPC over HTTP/2 URL structure:**
```
POST /{package}.{ServiceName}/{MethodName}
e.g.
POST /com.shopsphere.order.OrderService/CreateOrder
POST /com.shopsphere.inventory.InventoryService/CheckStock
```

This is always POST — gRPC does not use GET, PUT, DELETE. HTTP method semantics are irrelevant — the RPC method name carries the intent.

---

## The Four Communication Patterns — In Depth

### Pattern 1 — Unary RPC

```protobuf
service OrderService {
  rpc CreateOrder (CreateOrderRequest) returns (OrderResponse);
}
```

```
Client                    Server
  |--- CreateOrder req --->|
  |                        | (processes synchronously)
  |<-- OrderResponse ------|
```

**When to use:** Any standard request-response. The most common pattern. Equivalent to a REST POST/GET.

```java
// Server implementation
@Override
public void createOrder(CreateOrderRequest request,
                        StreamObserver<OrderResponse> responseObserver) {
    // Business logic
    Order order = orderService.create(request.getUserId(), request.getItemsList());
    
    OrderResponse response = OrderResponse.newBuilder()
        .setOrderId(order.getId())
        .setStatus(order.getStatus())
        .setTotal(order.getTotal())
        .build();
    
    responseObserver.onNext(response);      // send response
    responseObserver.onCompleted();         // close stream
}

// Client call
OrderResponse response = orderStub.createOrder(
    CreateOrderRequest.newBuilder()
        .setUserId("u-123")
        .addItems(item)
        .build()
);
```

---

### Pattern 2 — Server Streaming RPC

```protobuf
service OrderService {
  rpc StreamOrderUpdates (OrderStatusRequest) returns (stream OrderUpdate);
}
```

```
Client                         Server
  |--- OrderStatusRequest --->|
  |<-- OrderUpdate (1) -------|
  |<-- OrderUpdate (2) -------|
  |<-- OrderUpdate (3) -------|   ← server sends as events occur
  |<-- stream closed  --------|
```

**When to use:** Server needs to push multiple responses over time. Order tracking, live price feeds, log streaming, progress updates.

```java
// Server implementation
@Override
public void streamOrderUpdates(OrderStatusRequest request,
                               StreamObserver<OrderUpdate> responseObserver) {
    String orderId = request.getOrderId();
    
    // Subscribe to order events
    orderEventBus.subscribe(orderId, event -> {
        OrderUpdate update = OrderUpdate.newBuilder()
            .setOrderId(orderId)
            .setStatus(event.getStatus())
            .setTimestamp(event.getTimestamp())
            .build();
        
        responseObserver.onNext(update);   // push each update
        
        if (event.isFinal()) {
            responseObserver.onCompleted(); // close when delivered/cancelled
        }
    });
}

// Client consumption
stub.streamOrderUpdates(request, new StreamObserver<OrderUpdate>() {
    @Override
    public void onNext(OrderUpdate update) {
        System.out.println("Status: " + update.getStatus());
    }
    
    @Override
    public void onError(Throwable t) {
        System.err.println("Stream error: " + t.getMessage());
    }
    
    @Override
    public void onCompleted() {
        System.out.println("Order tracking complete");
    }
});
```

---

### Pattern 3 — Client Streaming RPC

```protobuf
service InventoryService {
  rpc BulkUpdateStock (stream StockUpdateRequest) returns (BulkUpdateResponse);
}
```

```
Client                              Server
  |--- StockUpdate (product 1) --->|
  |--- StockUpdate (product 2) --->|
  |--- StockUpdate (product 3) --->|  ← client streams many requests
  |--- stream closed           --->|
  |<-- BulkUpdateResponse      ----|  ← server responds once at the end
```

**When to use:** Client sends a large amount of data — bulk imports, chunked file uploads, batch operations. Instead of one massive request, stream it piece by piece.

```java
// Client streaming
StreamObserver<StockUpdateRequest> requestObserver = 
    stub.bulkUpdateStock(new StreamObserver<BulkUpdateResponse>() {
        @Override
        public void onNext(BulkUpdateResponse response) {
            System.out.println("Updated: " + response.getUpdatedCount());
        }
        @Override
        public void onError(Throwable t) { /* handle */ }
        @Override
        public void onCompleted() { /* done */ }
    });

// Stream 10,000 product stock updates
for (StockUpdate update : stockUpdates) {
    requestObserver.onNext(StockUpdateRequest.newBuilder()
        .setProductId(update.getProductId())
        .setNewStock(update.getStock())
        .build());
}
requestObserver.onCompleted(); // signal end of stream
```

---

### Pattern 4 — Bidirectional Streaming RPC

```protobuf
service ChatService {
  rpc Chat (stream ChatMessage) returns (stream ChatMessage);
}
```

```
Client                    Server
  |--- Message A -------->|
  |<-- Message B  --------|
  |--- Message C -------->|   ← both sides stream simultaneously
  |<-- Message D  --------|
  |--- close stream ------>|
  |<-- close stream -------|
```

**When to use:** Real-time bidirectional communication — live chat, collaborative editing, multiplayer game state sync, real-time trading.

```java
StreamObserver<ChatMessage> requestObserver = 
    stub.chat(new StreamObserver<ChatMessage>() {
        @Override
        public void onNext(ChatMessage message) {
            // Received message from server
            displayMessage(message.getSender(), message.getText());
        }
        @Override
        public void onError(Throwable t) { reconnect(); }
        @Override
        public void onCompleted() { System.out.println("Chat ended"); }
    });

// Send messages
requestObserver.onNext(ChatMessage.newBuilder()
    .setSender("u-123")
    .setText("Hello!")
    .build());
```

---

## gRPC Error Handling

gRPC has its own status codes — separate from HTTP status codes:

| gRPC Status | Code | Meaning |
|---|---|---|
| OK | 0 | Success |
| CANCELLED | 1 | Client cancelled the request |
| UNKNOWN | 2 | Unknown error |
| INVALID_ARGUMENT | 3 | Bad input (like HTTP 400) |
| NOT_FOUND | 5 | Resource not found (like HTTP 404) |
| ALREADY_EXISTS | 6 | Conflict (like HTTP 409) |
| PERMISSION_DENIED | 7 | Authorisation failure (like HTTP 403) |
| RESOURCE_EXHAUSTED | 8 | Rate limited (like HTTP 429) |
| UNAUTHENTICATED | 16 | Not authenticated (like HTTP 401) |
| UNAVAILABLE | 14 | Service down (like HTTP 503) |
| DEADLINE_EXCEEDED | 4 | Timeout |

```java
// Server — throwing a gRPC error
@Override
public void getOrder(GetOrderRequest request,
                     StreamObserver<OrderResponse> responseObserver) {
    Order order = orderRepository.findById(request.getOrderId());
    
    if (order == null) {
        responseObserver.onError(
            Status.NOT_FOUND
                .withDescription("Order " + request.getOrderId() + " not found")
                .asRuntimeException()
        );
        return;
    }
    
    if (!order.getUserId().equals(request.getCallerId())) {
        responseObserver.onError(
            Status.PERMISSION_DENIED
                .withDescription("You do not own this order")
                .asRuntimeException()
        );
        return;
    }
    
    responseObserver.onNext(buildResponse(order));
    responseObserver.onCompleted();
}

// Client — handling errors
try {
    OrderResponse response = stub.getOrder(request);
} catch (StatusRuntimeException e) {
    switch (e.getStatus().getCode()) {
        case NOT_FOUND:
            // handle 404 equivalent
            break;
        case PERMISSION_DENIED:
            // handle 403 equivalent
            break;
        case UNAVAILABLE:
            // retry with backoff
            break;
    }
}
```

---

## gRPC Interceptors — Cross-Cutting Concerns

Interceptors in gRPC are the equivalent of Spring filters/middleware — they run on every call without touching business logic:

```java
// Server-side interceptor — JWT authentication for all gRPC calls
public class AuthInterceptor implements ServerInterceptor {
    
    @Override
    public <Req, Resp> ServerCall.Listener<Req> interceptCall(
            ServerCall<Req, Resp> call,
            Metadata headers,
            ServerCallHandler<Req, Resp> next) {
        
        String token = headers.get(
            Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER)
        );
        
        if (token == null || !jwtService.isValid(token)) {
            call.close(Status.UNAUTHENTICATED
                .withDescription("Invalid or missing token"), new Metadata());
            return new ServerCall.Listener<>() {};
        }
        
        // Token valid — proceed to actual handler
        return next.startCall(call, headers);
    }
}

// Register interceptor on server
Server server = ServerBuilder.forPort(9090)
    .addService(ServerInterceptors.intercept(
        new OrderServiceImpl(), new AuthInterceptor(), new LoggingInterceptor()
    ))
    .build();
```

---

## Real-World Example — ShopSphere Internal gRPC Architecture

**Proto definitions across ShopSphere services:**

```protobuf
// order-service/src/main/proto/order.proto
syntax = "proto3";
package com.shopsphere.order;

service OrderService {
  rpc CreateOrder        (CreateOrderRequest)  returns (OrderResponse);
  rpc GetOrder           (GetOrderRequest)     returns (OrderResponse);
  rpc CancelOrder        (CancelOrderRequest)  returns (OrderResponse);
  rpc StreamOrderUpdates (GetOrderRequest)     returns (stream OrderUpdate);
}

// inventory-service/src/main/proto/inventory.proto
service InventoryService {
  rpc CheckStock    (CheckStockRequest)  returns (StockResponse);
  rpc ReserveStock  (ReserveRequest)     returns (ReserveResponse);
  rpc ReleaseStock  (ReleaseRequest)     returns (ReleaseResponse);
  rpc BulkSync      (stream SyncRequest) returns (SyncSummary);
}

// payment-service/src/main/proto/payment.proto
service PaymentService {
  rpc ProcessPayment (PaymentRequest)  returns (PaymentResponse);
  rpc RefundPayment  (RefundRequest)   returns (RefundResponse);
}
```

**Order creation flow using gRPC internally:**

```java
// OrderService calls InventoryService and PaymentService via gRPC
@Service
public class OrderCreationService {
    
    private final InventoryServiceGrpc.InventoryServiceBlockingStub inventoryStub;
    private final PaymentServiceGrpc.PaymentServiceBlockingStub paymentStub;
    
    public Order createOrder(CreateOrderRequest request) {
        
        // 1. Check stock via gRPC — synchronous, blocking
        StockResponse stockCheck = inventoryStub
            .withDeadlineAfter(500, TimeUnit.MILLISECONDS)  // 500ms timeout
            .checkStock(CheckStockRequest.newBuilder()
                .addAllProductIds(request.getItemsList()
                    .stream().map(Item::getProductId).toList())
                .build());
        
        if (!stockCheck.getAllAvailable()) {
            throw new OutOfStockException(stockCheck.getUnavailableProductsList());
        }
        
        // 2. Reserve stock
        inventoryStub.reserveStock(ReserveRequest.newBuilder()
            .setOrderId(tempOrderId)
            .addAllItems(request.getItemsList())
            .build());
        
        // 3. Process payment via gRPC
        PaymentResponse payment = paymentStub
            .withDeadlineAfter(3000, TimeUnit.MILLISECONDS)
            .processPayment(PaymentRequest.newBuilder()
                .setUserId(request.getUserId())
                .setAmount(calculateTotal(request.getItemsList()))
                .setPaymentMethodId(request.getPaymentMethodId())
                .build());
        
        if (!payment.getSuccess()) {
            inventoryStub.releaseStock(...); // compensate
            throw new PaymentFailedException(payment.getFailureReason());
        }
        
        // 4. Save order, publish events, return response
        return saveAndPublish(request, payment);
    }
}
```

---

## Interview Q&A

**Q: Why would you choose gRPC over REST for internal microservice communication?**
gRPC uses Protobuf binary encoding which is 3–10x smaller and 5–10x faster to serialise than JSON. It runs on HTTP/2 which multiplexes multiple concurrent calls over one connection. The strongly typed proto contract is enforced at compile time — breaking changes are caught before deployment, not at runtime. Native support for four communication patterns including streaming makes it versatile for real-time use cases. The only cost is loss of human readability and browser incompatibility, which do not matter for internal service-to-service calls.

**Q: What is Protobuf and how does it achieve smaller message sizes than JSON?**
Protobuf is a binary serialisation format where the schema is defined in a `.proto` file. Instead of including field names in every message like JSON does, Protobuf uses compact field numbers. Small integers are encoded using variable-length encoding consuming as little as one byte. The combination of binary encoding, field numbers instead of names, and varint compression results in messages 3–10x smaller than equivalent JSON, with significantly faster serialisation and deserialisation.

**Q: How does gRPC handle backward compatibility as services evolve?**
Protobuf's field number system enables safe evolution. Adding new optional fields with new field numbers is always backward compatible — old clients ignore unknown fields, new clients handle missing fields gracefully. You must never change a field's type or reuse a field number — doing so corrupts data in old clients. Removed fields should be marked `reserved` to prevent accidental reuse. This allows independent deployment of services without coordinated upgrades, which is essential in microservices.

**Q: What is deadline propagation in gRPC and why does it matter?**
A deadline is an absolute point in time by which a call must complete — set on the client with `withDeadlineAfter()`. Critically, deadlines propagate through the call chain — if the API Gateway gives the Order Service 2 seconds, the Order Service should pass a proportionally smaller deadline to the Payment Service. Without propagation, a slow downstream service can hold resources indefinitely even after the original caller has given up and moved on. Proper deadline propagation prevents resource leaks and cascading slowdowns across the entire call graph.

**Q: What is the difference between a gRPC deadline and a timeout?**
A timeout is a duration — "wait 500ms". A deadline is an absolute timestamp — "this entire operation must complete by 10:30:00.500". Deadlines are better in distributed systems because they propagate naturally across service calls — each downstream service knows exactly how much time remains for the entire operation, not just its individual slice. A timeout resets at each hop — a 500ms timeout at each of five services could take 2.5 seconds total while the original caller expected 500ms.

---

Say **"next"** when ready for Topic 4 — GraphQL (Deep Dive).
