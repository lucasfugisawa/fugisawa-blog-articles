# Kotlin + gRPC: Streaming, Deadlines, and Structured Error Handling

In the previous articles of this series, we built a solid foundation: from setting up a gRPC service in Kotlin, to mastering schema design, and writing idiomatic, expressive code with Protobuf and Kotlin DSLs.

Now, we're ready to go beyond the basics into features that power real-time, reactive, and resilient APIs.

In this article, we'll explore:
- How **gRPC streaming** works, and how to use **Kotlin coroutines and Flow** idiomatically.
- How to handle **deadlines and cancellations**.
- How to return and interpret **structured errors** using gRPC status codes and custom metadata.

 These features are essential when building robust and production-grade services.

## When request/response isn't enough

Until now, our gRPC calls have all followed a **unary** pattern, where the client sends a single request and the server replies with a single response.

But gRPC supports more than that. With **streaming**, you can build APIs that push data continuously, handle long-lived connections, or send multiple requests and responses within a single call.

There are three types of gRPC streaming:
- **Server streaming**: the client sends one request and receives a stream of responses.
- **Client streaming**: the client sends a stream of requests and gets a single response.
- **Bidirectional streaming**: both client and server send streams of messages.

Let's see how each one works in practice using **Kotlin's `Flow`** and coroutines.

## Server Streaming: one request, many responses

Imagine we want to support **live updates**: the client subscribes to notes with a certain tag, and the server streams back matching notes as they're created.

First, lets update the our `.proto` file and add that new RPC:
```protobuf
service NoteService {
  rpc StreamNotesByTag (NoteTagFilter) returns (stream Note);
}

message NoteTagFilter {
  string tag = 1;
}
```
Remember that when you update the Protobuf definitions, you also need to regenerate the Kotlin code by running the `generateProto` Gradle task, or the `protoc` command we saw in the first article of this series.

In Kotlin, this generates a suspending `Flow`-based method:
```kotlin
override fun streamNotesByTag(request: NoteTagFilter): Flow<Note> { 
	return flow { // Example: mock stream of notes tagged with 'kotlin'  
        val notes = listOf(
            note { title = "Note 1"; tags += "kotlin" },
            note { title = "Note 2"; tags += "kotlin" }
        } 
        notes.forEach {
            emit(it)
            delay(1000) // simulate async delivery 
	    }
    }
}
```

The client can collect the stream:

```kotlin
val tagFilter = noteTagFilter { tag = "kotlin" }
val responseFlow = stub.streamNotesByTag(tagFilter)

responseFlow.collect { note ->
    println("Received note: ${note.title}")
}
```

This is a powerful pattern for pushing real-time updates to clients, and thanks to Kotlin `Flow`, it integrates naturally with your coroutine pipeline.

## Client Streaming: multiple requests, one response

Let's say a client wants to **send a batch of notes** in a single stream, and the server replies with a summary (e.g., how many were saved).

Let's add the `.proto` definitions for that:

```protobuf
service NoteService {
  rpc CreateNotes (stream Note) returns (NoteBatchSummary);
}

message NoteBatchSummary {
  int32 count = 1;
}
```

The server receives a `Flow<Note>`:
```kotlin
override suspend fun createNotes(requests: Flow<Note>): NoteBatchSummary { 
    var count = 0 
    requests.collect { note ->
        println("Saving note: ${note.title}")
        count++
    } 
    return noteBatchSummary { this.count = count }
}
```

And on the client side, you can send a stream using `flow { ... }`:

```kotlin
val notesFlow = flow {
    emit(note { title = "Streamed Note 1" })
    emit(note { title = "Streamed Note 2" })
} 
val summary = stub.createNotes(notesFlow)

println("Server stored ${summary.count} notes")` 
```
This is great for bulk uploads or scenarios where messages arrive incrementally.

## Bidirectional Streaming: send and receive concurrently

Bidirectional streaming gives you full duplex communication, which is perfect for chat apps, collaborative editing, or real-time sync.

Let's suppose we want a functionality to share notes in real time with other users in the same group.

We'll start by defining the Protobufs for that:
```protobuf
service NoteService {
  rpc NoteCollab(stream Note) returns (stream Note);
}
```

Now, we can use that new definition to implement the logic. To keep this simple, we won't implement a complete logic for collaborative group notes. Instead, we'll implement this RPC to simply echo back each note received.

```kotlin

override fun noteCollab(requests: Flow<Note>): Flow<Note> = flow {
    requests.collect { receivedNote ->
        println("Received from ${receivedNote.title}: ${receivedNote.content}")
        emit(
            note {
                title = "From server: ${receivedNote.title}"
                content = "${receivedNote.content}"
            }
        )
    }
}
```

And on the client:

```kotlin
val noteFlow = flow {
    emit(note { title = "Note 1"; content = "Content 1" })
    emit(note { title = "Note 2"; content = "Content 2" })
} 
val responses = stub.noteCollab(noteFlow)

responses.collect { note ->
    println("Received: ${note.title} - ${note.content}")
}
```

Each side can send and receive notes independently, and all backed by non-blocking, coroutine-friendly flows.

----------

## Deadlines and cancellations

Every gRPC call can be bounded by a **deadline** — a time limit for how long the client is willing to wait.

In Kotlin, deadlines are passed using **call options**:

```kotlin
val stubWithDeadline = stub.withDeadlineAfter(3, TimeUnit.SECONDS) 
try { 
	val note = stubWithDeadline.createNote(
        createNoteRequest { title = "Timeout?" }
    )
} catch (e: StatusRuntimeException) { 
    if (e.status.code == Status.Code.DEADLINE_EXCEEDED) {
        println("Request timed out!")
    }
}
```

On the server side, when the deadline is exceeded or the client cancels, `collect` or `emit` will throw a cancellation exception.

To handle this cleanly, we just need to catch potential `CancellationException`:

```kotlin
override fun streamNotesByTag(request: NoteTagFilter): Flow<Note> = flow { 
    try { 
        while (true) {
            emit(generateNote())
            delay(1000)
        }
    } catch (e: CancellationException) {
        println("Client cancelled the stream")
    }
}
```

This makes it easier to free up resources when clients disconnect.


## Structured error handling

By default, when something goes wrong in gRPC, you'll get a `StatusRuntimeException` with a gRPC status code. For example:

```kotlin
throw Status.NOT_FOUND
    .withDescription("Note not found")
    .asRuntimeException()
```
The client can catch it and inspect the code and message:

```kotlin
try {
    stub.getNoteById(getNoteRequest { id = "missing" })
} catch (e: StatusRuntimeException) {
    println("Error: ${e.status.code} - ${e.status.description}")
}
```

But for production systems, you'll often want **structured error responses**, with application-level codes, details, or even localized messages. To achieve this, use **gRPC error metadata** with `StatusProto`.

For example, let's define an error details message in `.proto`:

```protobuf
import "google/rpc/status.proto";
import "google/rpc/error_details.proto";` 
```
And, then, build structured errors on the server:

```kotlin
val errorDetail = ErrorInfo.newBuilder()
    .setReason("NOT_FOUND")
    .setDomain("noteservice.fugisawa.com")
    .build() 
val statusProto = com.google.rpc.Status.newBuilder()
    .setCode(Code.NOT_FOUND_VALUE)
    .setMessage("Note not found")
    .addDetails(Any.pack(errorDetail))
    .build() throw StatusProto.toStatusRuntimeException(statusProto)` 
```

And finally extract it on the client:

```kotlin
val status = StatusProto.fromThrowable(e) 
val errorInfo = status?.detailsList
    ?.mapNotNull { it.unpack(ErrorInfo::class.java) }
    ?.firstOrNull()

println("Reason: ${errorInfo?.reason}, Domain: ${errorInfo?.domain}")
```
This approach gives you:
- Machine-readable error codes.
- Human-readable messages.
- Extensibility with metadata.

*The error handling code is not yet available in the provided source code, and should be added soon.*

## Implementation summary: what changed from the previous article?

As in the previous articles, all code is available in individual branches of the [note-service-kotlin-gprc](https://github.com/lucasfugisawa/note-service-kotlin-gprc) repository. For this article, you can check the [`article4-streaming`](https://github.com/lucasfugisawa/note-service-kotlin-gprc/tree/article4-streaming) branch.

Here's a quick summary of what's new:
- **Protobuf schema**:
    - Added `StreamNotesByTag`, `CreateNotes`, and `NoteCollab` RPCs for server, client, and bidirectional streaming.
- **Server implementation**:
    - Implemented all three streaming methods using Kotlin `Flow`.
- **Client implementation**:
    - Demonstrated how to collect streams using `Flow`.

## Final thoughts

In this article, we explored:
- How gRPC streaming works and how to use it idiomatically with Kotlin coroutines.
- How to build real-time and interactive APIs using server, client, and bidirectional streaming.
- How to use deadlines and cancellation to build responsive and efficient services.
- How to model structured, expressive error responses that go beyond raw status codes.

These tools give you the building blocks to write modern, responsive, and production-ready gRPC services in Kotlin.

---

To explore more about Kotlin-related topics, subscribe to my newsletter at  [https://fugisawa.com/](https://fugisawa.com/)  and stay tuned for more insights and updates.
