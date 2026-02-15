Best Article : https://medium.com/@satishios25/grpc-communication-between-micro-services-in-node-js-express-5a5c31ff8b6b

gRPC uses http 2 protocol . 

1. It runs over HTTP/2
2. HTTP/2 runs over TCP
3. TCP performs the 3-way handshake
4. TLS (if enabled) runs on top of TCP

So connection setup still includes:

1. TCP 3-way handshake (SYN → SYN-ACK → ACK)
2. TLS handshake (if using HTTPS)
3. HTTP/2 connection establishment

2️⃣ Is gRPC stateless?

This is where things differ from traditional HTTP/1.1.
Traditional HTTP (like Axios calls)

When you use axios:

axios.get("http://service-b/api")


Most of the time:

1. Each request is independent
2. Often a new TCP connection is opened (unless keep-alive is enabled)
3. HTTP/1.1 is request-response only
4. No multiplexing

Even with keep-alive:

1. Requests are still sequential over a connection
2. Head-of-line blocking exists

gRPC (HTTP/2)

gRPC uses HTTP/2 features:
✅ Persistent single TCP connection
✅ Multiplexed streams over one connection
✅ Binary framing
✅ Header compression (HPACK)
✅ Full duplex streaming

So:

1. It establishes TCP once.
2. Keeps the connection open.
3. Sends multiple requests simultaneously over the same TCP connection.

3️⃣ Does it do 3-way handshake every time?

No — only once per connection.

After the initial TCP connection:
1. Multiple RPC calls reuse the same connection.
2. No repeated handshakes.
3. No repeated TLS negotiation.
4. No connection setup cost per request.

This is one of the biggest performance gains.

4️⃣ How is gRPC better than normal HTTP + Axios between microservices?

🔹 1. Performance
Axios (HTTP/1.1)
    Text-based (JSON)
    Larger payload size
    Parsing cost
    Possible new TCP connections
    No multiplexing

gRPC
    Binary protocol (Protocol Buffers)
    Smaller payload size
    Faster serialization/deserialization
    Multiplexed requests
    Single persistent connection

🔹 2. Streaming Support

HTTP/1.1:
    Request → Response only

gRPC supports:
    Unary (normal request-response)
    Server streaming
    Client streaming
    Bidirectional streaming

🔹 3. Strong Contract (Proto Files)

With REST + Axios:
    No enforced schema
    You rely on documentation
    Runtime errors possible

With gRPC:
    .proto file defines:
    Message types
    Service contracts
    Strong typing
    Auto-generated client/server code
    Compile-time validation

🔹 4. Multiplexing (Very Important)

HTTP/1.1 problem:
    If one request is slow → others wait (head-of-line blocking)

HTTP/2 (used by gRPC):
    Multiple streams inside one TCP connection
    Requests processed in parallel

5️⃣ Is gRPC Stateful?

Transport layer: Uses persistent connection (stateful TCP session)
Application layer: Still stateless like REST unless you build session logic

6️⃣ When gRPC is Actually Better

gRPC shines in:
    Internal microservice communication
    High throughput systems
    Low latency systems
    Real-time streaming
    Strong contract-based systems
    Polyglot environments (Node, Go, Java, Python, etc.)

🏗️ How Proto Files Are Shared in Real Projects

Pattern 1 — Shared Proto Repository (Very Common)

/proto-repo
   degree.proto
   processor.proto

Each service imports proto as dependency (git submodule or npm package).

This ensures:
    All services use same contract
    Version control is centralized

Pattern 2 — Publish Proto as NPM Package

@company/protos

Each microservice installs:
    npm install @company/protos

🏗️ How to implement retry logic in gRPC ?

In Node.js (@grpc/grpc-js), retries work only if:
    You enable them via channel options
    The server returns retryable status codes
    Service config allows retry

Important retryable status codes:
    UNAVAILABLE
    DEADLINE_EXCEEDED
    RESOURCE_EXHAUSTED

🚀 3️⃣ Easiest Way: Manual Retry Wrapper (Recommended for Learning)

Let’s say your main service calls:

``` 
degreeClient.find({ id: 1 }, callback) 
```

You can wrap it like this:

``` 
function findWithRetry(client, request, retries = 3, delay = 500) {
  return new Promise((resolve, reject) => {
    function attempt(n) {
      client.find(request, (err, response) => {
        if (!err) {
          return resolve(response);
        }

        if (n === 0) {
          return reject(err);
        }

        console.log(`Retrying... attempts left: ${n}`);
        setTimeout(() => attempt(n - 1), delay);
      });
    }

    attempt(retries);
  });
}

 ```

Use it like:

```
    const result = await findWithRetry(degreeClient, { id: 1 });
```