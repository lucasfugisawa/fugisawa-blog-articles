

# Kotlin + gRPC: Nesting, Composition, Validations, and Idiomatic Builder DSL

In the first two articles of this series, we built our first gRPC service in Kotlin and explored the essentials of Protobuf schema design from optional and repeated fields to enums, oneof, maps, and how to evolve your schema safely.

In this article, we'    ll level up again and explore how to:
- Compose more **expressive data models** using **nested messages** and **message composition**.
- Write **idiomatic Kotlin code** using the Protobuf-generated **DSL builders**.
- Understand what types of **validations** belong in your schema, and which should be handled at the application level.

Let'    s get into it.

## Designing composite schemas

As your service grows, you'    ll likely need to model increasingly complex data structures. Instead of dumping all fields into a single message, Protobuf encourages a **compositional approach**.

Let'    s say we want to enrich our `Note` with:
- An **author** (id, name, email),
- And **timestamps** (created/updated).
 
 This is a great chance to encapsulate those concerns into separate messages:

```protobuf
message Author {
  string id = 1;
  string name = 2;
  string email = 3;
}

message Timestamps {
  string created_at = 1;
  string updated_at = 2;
}

message Note {
  string id = 1;
  string title = 2;
  optional string content = 3;
  repeated string tags = 4;
  map<string, string> metadata = 5;
  NoteStatus status = 6;
  Author author = 7;
  Timestamps timestamps = 8;
}
```

This brings several benefits:
- Better **modularity**: you can reuse `Author` or `Timestamps` elsewhere.
- Clearer **separation of concerns**.
- Cleaner and more readable `.proto` files.
    
From a Kotlin perspective, this also maps nicely into structured types that you can build and test independently.

## Nesting messages (inline or not)

Sometimes, if a message is only used inside a parent, it can be convenient to nest it directly:

```protobuf
message Note {
  message Attachment {
    oneof source {
      string file_path = 1;
      string url = 2;
    }
  }

  string id = 1;
  string title = 2;
  Attachment attachment = 9;
}
```

This keeps your schema scoped and avoids polluting the top-level namespace with types that are internal to `Note`.

In Kotlin, the generated class will be accessible as `Note.Attachment`, fully usable and composable like any other message.

## Writing idiomatic Kotlin with the builder DSL

The `protobuf-kotlin` library generates **Kotlin-style DSL builders** for your messages. We'    ve used these before, but now we'    ll dive into **more advanced and idiomatic usage patterns**.

You can compose nested messages with the gRPC builders DSL in a very concise and expressive way. Take a look at this example:

```kotlin
val note = note {
    id = UUID.randomUUID().toString()
    title = "My composed note" 
    content = "This note uses nested messages and DSL composition" 
    author = author {
        id = "user-123" name = "Lucas Fugisawa" email = "lucas@fugisawa.com" 
    }
    timestamps = timestamps {
        createdAt = LocalDateTime.now().toString()
        updatedAt = LocalDateTime.now().toString()
    }
}
```
The `note` is composed using a natural composition language enabled by the gRPC builders Kotlin DSLs.

This is clean, readable, and can leverage Kotlin's scope functions (`apply`, `let`, `run`) to make your code more concise and expressive. For example, let'    s say some fields should only be set when they'    re available:

```kotlin
val maybeContent: String? = fetchContent()

val note = note {
    title = "Conditional note" 
    maybeContent?.let { content = it } 
    
    if (includeTags) {
        tags += listOf("grpc", "kotlin")
    } 

    if (metadata.isNotEmpty()) {
        metadataMap.putAll(metadata)
    }
}
```

Repeated and map fields in Protobuf are always initialized (empty by default), and Kotlin gives you a nice collection interface:

```kotlin
note {
    tags += listOf("protos", "modeling")
    metadataMap["source"] = "imported" 
    metadataMap["priority"] = "high" 
}
```
You can also clear them when needed:
```kotlin
note {
    clearTags()
    clearMetadataMap()
}
```
Again, these APIs are idiomatic and safe, and you never need to worry about `null` collections.


## Validation: should I validate on schema or application?

A recurring design question is: *what types of validation should live in the Protobuf schema, and which ones should be handled in the application code?*

The answer is about separating **structural** vs. **business** validation.

### ✅ Schema-level validation: enforce structure

When we want to enforce structure, we handle validations by the schema itself:
- Is a field required or optional? → use `optional`, `oneof`, etc.
- Should only one of multiple fields be set? → use `oneof`.
- Should a field be a list? → use `repeated`.
- Is the value restricted to a fixed set? → use `enum`.

These are enforced during Protobuf message parsing and ensure that the wire format is well-formed.

```protobuf
message Note {
  optional string title = 1;
  oneof location {
    string folder = 2;
    string tag = 3;
  }
}
```

This ensures structural correctness before your business logic even runs.

### ❗ Application-level validation: enforce business rules

These rules depend on your domain logic:
- Title must be at least 5 characters.
- Author email must be valid.
- Updated timestamp must not precede created timestamp.
- Content length must not exceed 500 words.

These should be handled in your server (and possibly client) code:
```kotlin
require(note.title.length >= 5) { "Title must be at least 5 characters" }

require(note.timestamps.updatedAt >= note.timestamps.createdAt) { "updatedAt cannot be before createdAt" }
```
If you try to push business rules into the Protobuf schema, you'    ll quickly hit limitations and possibly hurt your API'    s flexibility.


## Implementation summary: what changed from the previous article?

As I mentioned in the previous article, the source code for the scope of each article will be made available in individual branches at the  [note-service-kotlin-gprc](https://github.com/lucasfugisawa/note-service-kotlin-gprc)  repository, while  `HEAD`  on the  `main`  branch will accommodate the latest working version.

In the article, I highlighted only parts of the code that were relevant to understanding the concepts. So, here is a summary of what changed in the code, which can be checked in detail in the [article3-composition-validation](https://github.com/lucasfugisawa/note-service-kotlin-gprc/tree/article3-composition-validation) repository branch.


### ✅ Protobuf implementation:
- Added `Author` and `Timestamps` message types.
- Included `author` and `timestamps` fields inside `Note`.
- Moved `Attachment` inside `Note` as a nested message using `oneof` for mutually exclusive fields (`url` vs `file_path`).
- Updated `CreateNoteRequest` to allow clients to pass an `Author`.

### ✅ Server-side logic:
- Dynamically sets `timestamps` on note creation using `Clock.System.now()`.
- Calculates and includes word count in `metadataDetails`.
- Composes the full `Note` using nested builders (`author { ... }`, `timestamps { ... }`) idiomatically.

### ✅ Client-side logic:
- Demonstrates how to build nested structures using Kotlin'    s builder DSL (`createNoteRequest { ... }`).
- Uses conditional composition (`tags +=`, `metadata["key"] = value`, etc.).
- Prints nested fields clearly: timestamps, author info, oneof handling for attachments.
    
----------

## Final thoughts

In this article, we covered:
- How to build richer and more structured Protobuf schemas with composition and nesting.
- How to write idiomatic Kotlin using the DSL builders generated by `protobuf-kotlin`.
- How to compose messages conditionally and fluently.
- When to validate at the schema level, and when to leave it to your app logic.

These are foundational skills for writing maintainable, expressive, and production-ready gRPC services.

Next up, we'    ll explore a new dimension: **gRPC Streaming in Kotlin**, including how to build reactive APIs with **coroutines** and **Flows** — perfect for real-time updates and sync.

---

To explore more about Kotlin-related topics, subscribe to my newsletter at  [https://fugisawa.com/](https://fugisawa.com/)  and stay tuned for more insights and updates.
