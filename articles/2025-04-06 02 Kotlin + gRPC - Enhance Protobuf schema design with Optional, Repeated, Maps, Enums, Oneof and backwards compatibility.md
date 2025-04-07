# Kotlin + gRPC: Enhance Protobuf schema design with Optional, Repeated, Maps, Enums, Oneof and backwards compatibility

In the previous (first) article of this series, we introduced gRPC and Protocol Buffers by building a minimal Kotlin gRPC service: a simple NoteService that allowed creating and retrieving notes.

Now that we've covered the fundamentals and the codebase setup, it's time to dig a bit deeper into the backbone of any gRPC application: your schema.

Protobuf is a schema-first system. Every API you design begins with a `.proto` file, and that file _is_ your contract. The way you design your fields (how you think about presence, defaults, repetition, optionality, maps, enums, and even deletions) will determine how flexible and resilient your API is as it evolves. *Other more architecture-level aspects (such as security, decoupling the domain model from the transport model, etc.) also affect your gRPC API resilience. However, we'll look at these later.*

This article focuses on the essential building blocks of schema design:
- Understanding field presence and default values.
- Using `optional`, `repeated`, `map`, `enum`, and `oneof` effectively.
- Applying best practices for evolving your Protobuf definitions without breaking clients.

Along the way, we'll evolve our NoteService use case with new requirements, applying these concepts in a real-world context.

## Evolving our use case: Notes with metadata and updates

In the previous article, a note had only a `title` and `content`, and a unique `id` assigned on creation. That was fine as a starting point, but real-world applications rarely stay that simple.

Let's say our team now needs the following:

- Notes should support optional **tags**, stored as a list.
- Users should be able to attach **custom metadata** (like a map of key/value labels).
- A note can optionally be marked as **archived**, and later we may add more status values.
- We want to support **partial updates**, where clients can update only specific fields.
- Notes may support **multiple attachment types**, but only one per note.
 
This isn't just scope creep — it's a chance to apply schema design best practices and show how to evolve your API safely and clearly.

## Understanding field presence and default values

One of the first challenges developers face when using Protobuf is grasping how it handles missing vs. default values.

In Protobuf, every field has a default value based on its type. For example:
- `int32` defaults to `0`
- `string` defaults to `""`
- `bool` defaults to `false`

But here's the tricky part: **unless you explicitly tell Protobuf to track field presence, you cannot distinguish between "unset" and "default"**.

Imagine this field:
```protobuf
string content = 2;
```
If a client omits this field entirely when calling a RPC, and another client sends an empty string (`""`), you won't be able to tell them apart. This matters in scenarios like PATCH operations, where *"clear the content"* (or change it to zero) and *"leave it untouched"* are very different instructions. To fix this, we have a few tools.

### Using `optional` in proto3 (since v3.15) to enable presence tracking

As of version 3.15, proto3 reintroduces the `optional` keyword to track whether a [scalar](https://squidfunk.github.io/protobluff/guide/scalar-types/) (i.e., numbers, enums and strings) field was explicitly set.

```protobuf
message Note {
  string id = 1;
  string title = 2;
  optional string content = 3;
}
```
This enables Kotlin code like:

```kotlin
if (note.hasContent()) { 
	// Client explicitly set the field  val content = note.content
} else { 
	// Field was not set 
}
```
This is particularly useful for implementing update operations where presence matters — for instance, distinguishing between *"remove the content"* (or change it to zero) and *"leave it unchanged"*.

### Using wrapper types to enable presence tracking for scalars

Protobuf < 3.15 doesn't support presence tracking on primitive fields (like `int32`, `bool`, etc.) by default. To track presence for those, you can use **Google's wrapper types**, which wrap a primitive inside a message.

Here's how you define them:

```protobuf
import "google/protobuf/wrappers.proto";

message NoteMetadata {
  google.protobuf.Int32Value word_count = 1;
}
```

This introduces a layer of indirection: the `word_count` is now a message with a single field, `value`. But this indirection lets you test for presence:

```kotlin
if (metadata.hasWordCount()) { 
	val count = metadata.wordCount.value
}
```
You can do this for all scalar types: `StringValue`, `Int32Value`, `BoolValue`, and so on.

Because optional was reintroduced to Protobuf 3.15+, wrappers are generally obsolete, and you shouldn't need them unless you're using messages that already use them. It's a good idea to check the [Proto Best Practices](https://protobuf.dev/best-practices/) when designing Protobuf APIs.

## Repeated fields: modeling lists

Let's say we want to support tags for a note. This is a perfect use case for `repeated` fields:

```protobuf
message Note {
  repeated string tags = 4;
}
```

Repeated fields always default to empty lists. On the Kotlin side, you'll work with them as immutable lists:

```kotlin
val tags: List<String> = note.tagsList
```

You can use the Kotlin builder DSL to set them:

```kotlin
val note = note {
    tags += "kotlin" 
    tags += "grpc" 
}
```
Repeated fields are safe to evolve, so you can add them any time without breaking clients.

## Using `map` for representing arbitrary key-value metadata

Let's say we want to support storing user-defined metadata on a note. Think of labels like `"project" -> "gRPC"` or `"priority" -> "high"`.

```protobuf
message Note {
  map<string, string> metadata = 5;
}
``` 

This defines a map field that becomes a `Map<String, String>` in Kotlin:

```kotlin
val metadata: Map<String, String> = note.metadataMap
```

Under the hood, a `map` is just a `repeated` field of key/value pairs, but Protobuf provides syntactic sugar for it.

Some caveats you should considering when using maps are:
- Map keys must be scalars (string, int, etc.).
- You can't track presence of individual keys — you either have the key or you don't.
- Map order is not guaranteed.

## Modeling finite sets with `enum`: 

Let's model a note's status. We might want to distinguish between active and archived notes:

```protobuf
enum NoteStatus {
  UNKNOWN = 0;
  ACTIVE = 1;
  ARCHIVED = 2;
}

message Note {
  NoteStatus status = 6;
}
```

Two rules here are crucial:
- Always define `0` as a meaningful default, usually `UNKNOWN`.
- Avoid changing enum numeric values, since clients may have them hardcoded.

In Kotlin, Protobuf generates an enum with a fallback for unrecognized values:
```kotlin
when (note.status) {
    NoteStatus.ACTIVE -> ...
    NoteStatus.ARCHIVED -> ...
    NoteStatus.UNRECOGNIZED -> ...
}
```

This is critical for **forward compatibility**: if a newer server adds a new status (`DELETED = 3`), older clients won't crash — they'll get `UNRECOGNIZED`.

## Handling mutually exclusive fields with `oneof`:

Let's say users can attach either a file or a URL to a note, but never both.

```protobuf
message Attachment {
  oneof source {
    string file_path = 1;
    string url = 2;
  }
}
```

This enforces exclusivity: only one of the fields can be set at a time.

Kotlin gives you a `.sourceCase` field to inspect:

```kotlin
when (attachment.sourceCase) {
    Attachment.SourceCase.FILE_PATH -> ...
    Attachment.SourceCase.URL -> ...
    Attachment.SourceCase.SOURCE_NOT_SET -> ...
}
```

This is a powerful modeling tool when you want a field to be _either-or_, such as *“use X or use Y, but not both”*.


## Safe schema evolution

As your service evolves, you'll need to change your schema, but not all changes are safe. Protobuf allows some flexibility, but you must follow a few rules to avoid breaking existing clients or corrupting data.

###  Safe changes

- ✅ **Adding new fields (with new tags)**: This is safe and common. Clients that don't know the new field simply ignore it. Just be sure to use a unique tag that hasn't been used before.
- ✅ **Adding new enum values (with new tags)**: This is also safe. Clients that don't recognize the new value will treat it as `UNRECOGNIZED`, allowing fallback handling.
- ⚠️ **Changing field names**: The name of the field doesn't affect the wire format. Only the tag matters. However, generated code will change, which may break client code unless they recompile. Safe for the protocol, but not always for your code.
- ⚠️ **Changing default behavior**: While proto3 does not support custom default values in `.proto` files, you can change how your app behaves when a field is unset. This is safe as long as you don't rely on transmitted defaults.

### Breaking changes

- 🚫 **Removing fields**: Old clients may still send these fields. If the tag is not handled properly, the data may be dropped or misinterpreted. Always **[reserve](https://protobuf.dev/programming-guides/proto3/#reserved)** the tag and field name to avoid future reuse.
- 🚫 **Changing field tags**: Tags define the wire format. Changing a tag effectively creates a new field, breaking compatibility with all existing clients and servers.
- 🚫 **Changing field types**: Even similar types like `int32` to `int64` are encoded differently and may not deserialize correctly. Never change a field type without also changing its tag.
- 🚫 **Reusing tag numbers**: A reused tag can be misinterpreted as an old field. This is dangerous and can silently corrupt data. Always mark removed tags as `reserved`.

When breaking changes are truly necessary, consider versioning your messages (e.g., `NoteV2`) and methods (`CreateNoteV2`) to introduce changes without affecting existing consumers.


## Type Mapping Reference

Here is a quick summary of how Protobuf types map to Kotlin:

| Protobuf Type                      | Kotlin Type                  | Notes                                         |
|------------------------------------|-------------------------------|-----------------------------------------------|
| `string`                           | `String`                      | Defaults to `""`                              |
| `int32`, `int64`, `uint32`...      | `Int`, `Long`, `UInt`, etc.   | Defaults to 0                                 |
| `bool`                             | `Boolean`                     | Defaults to false                             |
| `optional string`                  | `String` + `.hasField()`      | Presence tracked explicitly                   |
| `google.protobuf.StringValue`      | `String?` + `.hasField()`     | Wrapper, allows presence detection            |
| `repeated string`                  | `List<String>`                | Always present, default = emptyList           |
| `map<string, string>`              | `Map<String, String>`         | Always present, default = emptyMap            |
| `enum`                             | `Enum`                        | Has `UNRECOGNIZED` fallback                   |
| `oneof`                            | Field + `.case` enum          | Only one field set at a time                  |


## Implementation summary: what changed from the previous article?

As I mentioned in the previous article, the source code for the scope of each article will be made available in individual branches at the  [note-service-kotlin-gprc](https://github.com/lucasfugisawa/note-service-kotlin-gprc)  repository, while  `HEAD`  on the  `main`  branch will accommodate the latest working version.

In the article, I highlighted only parts of the code that were relevant to understanding the concepts. So, here is a summary of what changed in the code, which can be checked in detail in the [article2-schema-design](https://github.com/lucasfugisawa/note-service-kotlin-gprc/tree/article2-schema-design) repository branch.

- **Protobuf schema**:
    - Introduced `optional` keyword for tracking field presence (e.g., `content`).
    - Added a `repeated string tags` field to allow multiple tags per note.
    - Added `map<string, string> metadata` to store arbitrary key-value annotations.
    - Introduced a `NoteStatus` enum with values `UNKNOWN`, `ACTIVE`, and `ARCHIVED`.
    - Added a `NoteMetadata` message that includes `word_count`, using a wrapper type (`Int32Value`).
    - Introduced a `oneof` field in `Attachment` to allow attaching either a URL or file path — but never both.
        
- **Server implementation**:
    - Now checks for presence using `hasContent()`.
    - Dynamically computes and returns word count in `metadataDetails`.
    - Maps all new fields from the request into the response.
        
- **Client implementation**:
    - Demonstrates how to build a request with optional fields, tags, metadata, attachment, and enum values.
    - Shows how to check presence, handle `oneof`, and interpret server responses idiomatically in Kotlin.
 
## Final thoughts

Designing Protobuf schemas is more than just writing `.proto` files. It's about thinking deeply about how your API will evolve, how your data behaves over the wire, and how clients will interpret what they receive.

In this article, we've explored:
- Why field presence matters, and how to track it.
- How to use optional fields, wrapper types, repeated values, maps, and enums effectively.
- How to model exclusive choices using `oneof`.
- How to evolve your schema safely, without breaking existing clients.
 
These tools and techniques are foundational for any team using gRPC in production. Schema design _is_ API design, and mastering it gives you a long-term edge in building reliable services.

---

To explore more about Kotlin-related topics, subscribe to my newsletter at  [https://fugisawa.com/](https://fugisawa.com/)  and stay tuned for more insights and updates.
