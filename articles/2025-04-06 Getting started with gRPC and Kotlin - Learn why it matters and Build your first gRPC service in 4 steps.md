
# Getting started with gRPC and Kotlin: Build your first gRPC service in 4 steps

This is the first article in a hands-on series where we'll explore how to build **gRPC** services using **Kotlin**. The series is designed to help you understand not only the technical how-to, but also the design choices behind **Protocol Buffers** and **gRPC**. We'll start simple and gradually introduce more advanced concepts as we go.

Here is what you can expect from the upcoming articles:
- In the next article, we will explore field **presence**, **optional** fields, and how to **evolve your API safely**.
- Then, we will expand our model to cover **repeated fields** and **nested messages**.
- Finally, we will explore gRPC streaming, how to build APIs that go beyond request/response, using **Kotlin coroutines** and **flows**.
    
By the end of the series, you will have a solid understanding of gRPC in Kotlin, and a real working service you can build upon.

If you've worked with REST APIs, you know they've become the standard way to build and expose web services. They're simple, flexible, and have broad tooling support. But when services need to communicate with each other efficiently, especially in a distributed or high-performance system, gRPC is often a better choice.

In this first article, we will talk about what gRPC is, how it fits into a Kotlin ecosystem, and how to build a simple gRPC service and client from scratch. We will also compare it to REST and JSON, discuss its tradeoffs, and walk you through the tools and structure needed to get started.

By the end, you'll understand how gRPC works, why it matters, and how to implement a basic service using Kotlin, Protocol Buffers, and gRPC.

## What is gRPC and Why Use It?

**[gRPC](https://grpc.io/)** is a framework created by Google for making remote procedure calls (RPC). It allows you to call methods on a service running on another machine as if they were local. Under the hood, it uses HTTP/2 and **[Protocol Buffers (Protobuf)](https://protobuf.dev/)** for communication.

**Protocol Buffers** are a language-neutral, platform-neutral mechanism for serializing structured data. You define your messages and services in a `.proto` file, and code is generated from that file in your target language.

This allows for benefits such as:
- **Strongly typed contracts** between services.
- **Code generation** avoids boilerplate and reduces errors.
- **Binary serialization** for performance and smaller payloads.
- **HTTP/2** enables multiplexing, streaming, and low latency.

### REST vs. gRPC: Tradeoffs

It's important to understand that gRPC is not always "better" than REST. It's different. Choosing between REST and gRPC depends on your context, constraints, and goals.

#### gRPC strengths:
- More efficient data transmission (Protobuf is binary, smaller than JSON).
- Strong typing and codegen reduce runtime errors.
- Supports streaming (one-way and bidirectional).
- Easier evolution with schema compatibility rules.

#### REST strengths:
- Human-readable and debug-friendly (JSON over HTTP).
- Better support for web clients and public APIs.
- Mature ecosystem, including caching, proxies, and tools.

## Our Use Case: A Simple Note Service

In this series, we will build a Note Service — something like a backend for a basic note-taking app. Over time, we will introduce complexity gradually.

Here's what you'll see in future articles:
- Add support for **tags**, **users**, and **note metadata** (nested fields, repeated fields).
- Handle optional fields and versioning concerns as the schema evolves.
- Use **streaming** for features like syncing or watching updates in real-time.
- Learn about Protobuf features like `oneof`, field presence, and default values.

Let's start small: a service to create a new note with a title and content.

### Project structure and tools

To build our Kotlin gRPC service, we'll use:
- **[Kotlin/JVM](https://kotlinlang.org/docs/jvm-get-started.html)** as our programming language on the JVM platform.
- **[Gradle](https://kotlinlang.org/docs/gradle.html) (Kotlin DSL)** — dependency and build management.
- **[Protocol Buffers](https://kotlinlang.org/docs/gradle.html) (`.proto`)** — our API schema.
- **[protoc](https://protobuf.dev/installation/)** — the Protobuf compiler.
- **[gRPC Kotlin](https://grpc.io/docs/languages/kotlin/quickstart/)** — gRPC support for Kotlin and coroutines.

Our folder structure will start with something like this (a very basic Kotlin + Gradle project), but might evolve as we expand our project scope:
```
note-service-kotlin-gprc/
├── build.gradle.kts
├── settings.gradle.kts
├── src/
│   ├── main/
│   │   ├── kotlin/            # Kotlin code (server + client)
│   │   └── proto/             # Protobuf files (.proto)
```
The source code for the scope of each article will be made available in individual branches at the [note-service-kotlin-gprc](https://github.com/lucasfugisawa/note-service-kotlin-gprc) repository, while `HEAD` on the `main` branch will accommodate the latest working version.

The speific branch for this article is [article1-first-service](https://github.com/lucasfugisawa/note-service-kotlin-gprc/tree/article1-first-service).

## Step 1: Writing your first `.proto` file

Let's define the initial structure of our Note Service using Protocol Buffers. This file describes the types and service methods that gRPC will generate for us.

Create a file at `src/main/proto/note_service.proto` with the following content:

```protobuf
syntax = "proto3";

package com.fugisawa.grpc.noteservice;

option java_package = "com.fugisawa.grpc.noteservice";

option java_multiple_files = true;

service NoteService { 
    rpc CreateNote (CreateNoteRequest) returns (Note); 
}

message CreateNoteRequest { 
    string title = 1; 
    string content = 2;
}

message Note { 
    string id = 1; 
    string title = 2; 
    string content = 3; 
}
```

Let's break this down:

### `syntax = "proto3";`

This declares that we're using **[proto3](https://protobuf.dev/programming-guides/proto3/)**, the most recent version of 'the Protocol Buffers language. It simplifies the syntax and sets some defaults (like making fields optional unless marked otherwise).

### `package note;`

This is the **Protobuf package name**. It groups related messages and services under a common namespace to avoid naming conflicts.

### `option java_package` and `java_multiple_files`

These are used by the code generator for JVM languages (like Kotlin or Java):

- `java_package = "com.fugisawa.noteservice"`: sets the package name in the generated Kotlin/Java code.
    
- `java_multiple_files = true`: instead of placing all generated types into a single file, this generates one file per message/service, which is cleaner and more modular.

### The `service` block

This defines the **gRPC service** and its RPC methods. In our case...

```protobuf
service NoteService {
    rpc CreateNote (CreateNoteRequest) returns (Note);
}
```
... defines a service named `NoteService` with one RPC method `CreateNote`, which accepts a message of type `CreateNoteRequest` and returns a `Note`.
    
gRPC will generate both a base server class and a client stub for us, based on this.

### The `message` blocks

These define structured data for the request and response types.
```protobuf
message CreateNoteRequest {
    string title = 1;
    string content = 2;
}
```

This defines the input type for our `CreateNote` method. It has two fields: `title` (a string field with **tag 1**) and `content` (another string field with **tag 2**).
    
And this...
```protobuf
message Note {
    string id = 1;
    string title = 2;
    string content = 3;
}
```
... is the response type, which includes `id` (a string representing the unique note ID, with **tag 1**), `title` (for the note title with **tag 2**) and `content` (for the note content with **tag 3**).

### But what are those numbered tags?

Each field in a Protobuf message has a **unique number**. These numbers are used in the binary encoding of the message, not the field names. They matter a lot:
- **Must not change** once published — they define the wire format.
- **Can't reuse deleted field numbers** — unless you're absolutely sure they won't be interpreted by old clients.
- **Field order doesn't matter** in the source, but tags must be unique.

Protobuf tags must be between 1 and (2<sup>29</sup> - 1), but you should use 1–15 for frequently-used fields, since they encode more efficiently.

## Step 2: Generating Kotlin code from Protobuf definitions

The next step is to generate Kotlin code from our `.proto` file. There are two main ways to do this:

### Option 1: Using Gradle

For compiling our Protobuf definitions to Kotlin code, we can use a Gradle plugin, which makes our life easier. Add the Protobuf plugin and dependencies to your `build.gradle.kts`. Remember we also need to add the Kotlin Coroutines dependency:
```kotlin
plugins { 
    kotlin("jvm") version "1.9.22" 
    id("com.google.protobuf") version "0.9.4" 
}

repositories { 
    mavenCentral() 
}

dependencies { 
    implementation("io.grpc:grpc-kotlin-stub:1.4.1")  
    implementation("io.grpc:grpc-netty-shaded:1.71.0")  
    implementation("io.grpc:grpc-protobuf:1.71.0")
    implementation("com.google.protobuf:protobuf-kotlin:4.30.2")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.10.1")
}

protobuf {  
    protoc {  
        artifact = "com.google.protobuf:protoc:4.30.2"  
    }  
    plugins {  
        create("grpc") {  
            artifact = "io.grpc:protoc-gen-grpc-java:1.71.0"  
        }  
        create("grpckt") {  
            artifact = "io.grpc:protoc-gen-grpc-kotlin:1.4.1:jdk8@jar"  
        }  
    }  
    generateProtoTasks {  
        all().forEach {  
            it.builtins {  
                create("kotlin")  
            }  
            it.plugins {  
                id("grpc")  
                id("grpckt")  
            }  
        } 
    }
}
```
*(Consider using local String variables for dependency version numbers. I didn't do it here for simplicity.)*

Now run:
```bash
./gradlew generateProto
```
This generates Kotlin and gRPC code under `build/generated`.

----------

### Option 2: Manual compilation using `protoc`

You can also generate code manually:
```bash
protoc  
    --proto_path=src/main/proto  
    --kotlin_out=build/generated/source/proto/main/kotlin  
    --grpc-kotlin_out=build/generated/source/proto/main/kotlin  
    note_service.proto
```
You'll need the `protoc` binary and the `protoc-gen-grpc-kotlin` plugin [installed](https://protobuf.dev/installation/) on your system.

## Step 3: Implementing the gRPC Server

Now that we have the Kotlin stub code generated, the next step is to write the backend logic that will handle incoming gRPC calls. This is where we implement the `NoteService`.
```kotlin
package com.fugisawa.noteservice  
  
import com.fugisawa.grpc.noteservice.CreateNoteRequest  
import com.fugisawa.grpc.noteservice.Note  
import com.fugisawa.grpc.noteservice.NoteServiceGrpcKt  
import com.fugisawa.grpc.noteservice.note  
import java.util.*  
  
class NoteServiceImpl : NoteServiceGrpcKt.NoteServiceCoroutineImplBase() {  
    override suspend fun createNote(request: CreateNoteRequest): Note {  
        // Here, you will implement your note creation logic, persist the new note etc.  
        // Let's just generate dummy note, for this example:  val newNoteId = UUID.randomUUID().toString()  
        return note {   
            id = newNoteId  
            title = request.title  
            content = request.content  
        }  
    }  
}
```
In this code, what we are doing is:
- We are extending the `NoteServiceCoroutineImplBase` (generated by the Protobuf compilation).
- We are implementing the `createNote()` function with our own logic for creating notes.

We also need to start our gRPC server, register the service, and block the main thread (which, for simplicity, we'll do in the main application entry point function):

```kotlin
package com.fugisawa.noteservice  
  
import io.grpc.Server  
import io.grpc.ServerBuilder  
  
fun main() {  
    val server: Server = ServerBuilder  
        .forPort(50051)  
        .addService(NoteServiceImpl())  
        .build()  
  
      server.start()  
      println("Server started on port 50051")  
      server.awaitTermination()  
}
```
That's all we need to get a Kotlin gRPC server running.

## Step 4: Creating a gRPC Client in Kotlin

Now, let's now write a client that connects to the server and sends a `CreateNote` request.

Typically, we will not implement a gRPC client for a gRPC service available in the same application. We will do this in this article just to keep things simple and avoid having to create a new application to act as the client.

```kotlin
package com.fugisawa.noteservice  
  
import com.fugisawa.grpc.noteservice.NoteServiceGrpcKt.NoteServiceCoroutineStub  
import com.fugisawa.grpc.noteservice.createNoteRequest  
import io.grpc.ManagedChannelBuilder  
import kotlinx.coroutines.runBlocking  
  
fun main() = runBlocking {  
    val channel =  
        ManagedChannelBuilder  
            .forAddress("localhost", 50051)  
            .usePlaintext()  
            .build()  

    val stub = NoteServiceCoroutineStub(channel)  

    val request = createNoteRequest {  
        title = "My first note"  
        content = "This is my first note created with gRPC!"  
    }  

    val note = stub.createNote(request)  

    println("Note created: ${note.id} - ${note.title}")  
}
```

This small program connects to our gRPC server and calls the `CreateNote` method, just like a real client would. Let's walk through what each part is doing.

We wrap our `main` function in `runBlocking` because gRPC Kotlin uses **suspend functions** for all RPC calls, which means we need a coroutine scope to call them.

```kotlin
fun main() = runBlocking { ... }
```

Then, we create a **gRPC channel** (which is a connection to the server). We are connecting to `localhost:50051`, which is where our server is running. We are using `.usePlaintext()` to disable TLS (which is fine for local development), as we didn't configure TLS for the server.

Then we create a stub. A **stub** is a client-side object that lets you call remote methods as if they were local. We're using the **coroutine-based version**, auto-generated by the gRPC Kotlin plugin. 
```kotlin
val stub = NoteServiceCoroutineStub(channel)
```
This stub exposes `suspend` functions like `createNote()` that we can call directly.

The next thing we do is create a `CreateNoteRequest` object using the **Kotlin DSL builder** generated from the `.proto` file. This DSL-style builder only works because you're using the `protobuf-kotlin` library and configured our Gradle setup to generate `kotlin` output.
```kotlin
val request = createNoteRequest {
    title = "My first note"
    content = "This is my first note created with gRPC!"
}
```
Finally, here is where the actual **RPC call** happens. We call `createNote()` on the stub, passing the request. It suspends while the request is sent and response is received from the server.

```kotlin
val note = stub.createNote(request)
```

Notice how simple and idiomatic calling a gRPC service can be in Kotlin:
- All calls are suspending functions — no callbacks, no threads to manage.
- Messages are constructed using type-safe Kotlin builders.
- Communication is binary, efficient, and strongly typed.

## How to run the Server and the Client

We're building both server and client in the same Gradle project, but typically they would be two separate applications. In this article, we will use them by starting our application with different entry points.
- To run the server, execute `Server.main()`.
- In a second terminal (or a new IntelliJ run configuration), run `Client.main()`.

# Final thoughts

In this article, we covered:
- What gRPC and Protobuf are, and why they matter.
- When to use gRPC vs REST.
- How to define an API using a `.proto` file.
- How to generate Kotlin code using Gradle and `protoc`.
- How to implement and run a gRPC server and client in Kotlin.

This article is a bit long, as it includes an introduction to the main concepts and the setup of an example project. In the upcoming articles, which will be more concise, we'll dig deeper into optional fields, field/argument presence detection, and the subtle but important differences in how Protocol Buffers handle missing or default values. That's where things get interesting and real-world.

--- 

To explore more about Kotlin-related topics, subscribe to my newsletter at  [https://fugisawa.com/](https://fugisawa.com/)  and stay tuned for more insights and updates.
