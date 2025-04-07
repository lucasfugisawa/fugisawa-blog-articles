

# Kotlin + gRPC: Tooling, CI/CD, and Architectural Practices

In the previous articles of this series, we explored how to build idiomatic gRPC APIs with Kotlin — from schema design using Protobuf to leveraging streams, coroutines, and Kotlin DSL builders. Now it's time to look beyond the application code.

In this article, we'll discuss what it takes to prepare your gRPC service for production. This includes:
- **Observability and traceability**.
- **Security and authentication**.
- **Testing tools and diagnostics**.
- **Best practices for repository structure**.
- **CI/CD automation**.
- And how this fits into a **DDD-oriented architecture**.

This fifth article is more of a bonus and a point of attention that you should focus on if you need to implement gRPC for a production environment. Each topic mentioned here is very broad and it is up to you to delve into each of them as needed. Also, for this reason, I will not cover code changes for the content of this article specifically.

Our focus is on **scalability and maintainability**. Let's explore how to structure your environment so your gRPC APIs thrive in the real world.

## Observability

In REST APIs, it's common to use reverse proxies, access logs, or APM tools for insights. In the gRPC world, you need to be more intentional.

### Interceptors

Interceptors allow you to intercept gRPC calls and inject cross-cutting concerns like logging, metrics, or authentication. They work similarly to middleware in web frameworks.

```kotlin
class LoggingInterceptor : ServerInterceptor {
    override fun <ReqT, RespT> interceptCall(
        call: ServerCall<ReqT, RespT>, headers: Metadata,
        next: ServerCallHandler<ReqT, RespT>
    ): ServerCall.Listener<ReqT> {
        val method = call.methodDescriptor.fullMethodName
        println("[gRPC] Call to method: $method")
        return next.startCall(call, headers)
    }
}
```

You can also hook into libraries like `micrometer` to expose metrics (latency, counters, histograms) to your monitoring tool.

### Distributed tracing and OpenTelemetry

With multiple services interacting via gRPC, tracing request flows is crucial. **OpenTelemetry** is the open standard for collecting traces, metrics, and logs.

```kotlin
val tracer = GlobalOpenTelemetry.getTracer("note-service")
val span = tracer.spanBuilder("createNote").startSpan()
try {
    // request logic
} finally {
    span.end()
}
```

Use OTLP exporters to send data to Jaeger, Zipkin, Grafana Tempo, NewRelic, DataDog etc. and visualize the full call path across services.

If you don't have very complex tracing requirements, I recommend that you use the [OpenTelemetry Java Agent](https://opentelemetry.io/docs/zero-code/java/agent/) instead of instrumenting through code. And if you need some tracing customization, you can use an [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) to process the data before sending it to your monitoring tool. By avoiding instrumentation through code, you end up with a simpler, more scalable, and less intrusive solution to observe what's happening with your application.

----------

## Security: encryption and authentication

By default, gRPC does not enforce TLS or authentication. You must explicitly configure it.

### TLS: encrypting transport

gRPC uses HTTP/2 under the hood, which supports TLS natively. In production, you can use managed certificates (e.g., via ingress or cert-manager). For local development, you can configure it manually:

```kotlin
val creds = TlsServerCredentials.create(certChain, privateKey)
val server = Grpc.newServerBuilderForPort(443, creds)
    .addService(NoteServiceImpl())
    .build()
```

### Authentication: via metadata headers

gRPC supports custom headers via `Metadata`. The common pattern is to pass a token (e.g., JWT) through the `Authorization` header.

```kotlin
class AuthInterceptor : ServerInterceptor {
    override fun <ReqT, RespT> interceptCall(...): ServerCall.Listener<ReqT> {
        val token = headers.get(Metadata.Key.of("Authorization", Metadata.ASCII_STRING_MARSHALLER))
        if (!validateToken(token)) {
            call.close(Status.UNAUTHENTICATED.withDescription("Invalid token"), headers)
            return object : ServerCall.Listener<ReqT>() {}
        }
        return next.startCall(call, headers)
    }
}
```
In the interceptor, validate the token, extract claims, and inject user context for downstream logic.

----------

## Repository Structure: where do `.proto` files go?

In systems with multiple services, `.proto` file sharing becomes a real concern. You typically choose between two options:

### Monorepo (app + proto together)
- Everything in one place; simple for small teams.
- Easy co-evolution of contracts and services.
- Harder to share across languages or services.

### Dedicated proto repository
- Store `.proto` files separately with versions and changelogs.
- Services import generated artifacts or raw definitions.
- Publish as binary artifacts (e.g., JAR, NPM package).
    
This second pattern usually promotes decoupling, better versioning, and clean reuse across the organization.


## Tools for testing gRPC APIs

Testing gRPC locally doesn't have to be difficult. These tools make it straightforward:

### grpcurl

Think of it as `curl` for gRPC. Make calls from the command line:

```
grpcurl -plaintext -d '{"title":"Hello"}' localhost:50051 com.fugisawa.NoteService/CreateNote
```

### Postman

Postman now supports gRPC. You can import your `.proto` files or use server introspection, and test methods via a graphical interface.


## CI/CD: artifact generation and compatibility guarantees

You should automate the generation and publication of your Protobuf contracts.

### Build + publish

In your CI pipeline, generate and publish a reusable artifact:

```bash
./gradlew generateProto build publish
```

This artifact can be published to Maven, GitHub Packages, or Artifactory and consumed by other services.

### Compatibility checks

[buf](https://buf.build/docs/breaking/overview/) is a powerful tool that checks for breaking changes in your `.proto` definitions:

```
buf breaking --against ./previous-version
```

Run this in CI to prevent merges that would break existing consumers.

## Domain models and Protobuf: avoiding contract leakage

The types generated by Protobuf (like `Note`) are transport-level DTOs. They should **not** be used as your domain model.

### Separate domain from transport

Design rich domain types with business semantics:

```
data class Note(val id: NoteId, val title: Title, val content: Content)
```

Then map between transport and domain:

```
fun Note.toProto(): NoteProto = note {
    id = this@toProto.id.value
    title = this@toProto.title.value
    content = this@toProto.content.value
}
```

This keeps your domain clean, testable, decoupled and independent of gRPC or any other transport layer.

## Final thoughts

In this article, we explored:
- How to add observability with interceptors and tracing.
- How to secure gRPC with TLS and token-based authentication.
- How to test APIs with `grpcurl` and Postman.
- How to structure your repositories and CI/CD pipelines.
- How to keep your domain model decoupled from your transport layer.

These practices are essential for building resilient, maintainable, and scalable gRPC APIs in real-world environments.

---

This is the last article in the Kotlin + gRPC series. If you want to talk and exchange experiences on this subject, contact me, and I will be happy to talk.

To explore more about Kotlin-related topics, subscribe to my newsletter at  [https://fugisawa.com/](https://fugisawa.com/)  and stay tuned for more insights and updates.
