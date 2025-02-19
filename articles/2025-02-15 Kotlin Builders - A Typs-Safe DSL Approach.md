You might recall my [Simplifying the Builder Pattern](https://dev.to/lucasfugisawa/kotlin-design-patterns-simplifying-the-builder-pattern-3e7g) article about using Kotlin Data Classes as a simpler version of the Builder pattern. You saw how named parameters and default values removed much of the ceremony you see in a traditional Builder. Now, I want to guide you further.

In this article, we will explore Kotlin type-safe builders (DSLs). You will learn how to create code for object construction that feels more expressive. Think of it like reading small sentences.

If you want to build objects in a clear and flexible way, this piece will walk you step by step. You will see how to write DSLs in Kotlin. Then you will create objects with code like `car { ... }`, which is both concise and readable.

Take what you already know about Kotlin and extend it. Let's begin and see how to go beyond data classes while keeping the same clarity.

## What are function types with a receiver?

Before we build our DSL, we need a quick look at one Kotlin feature: **function types with a receiver**.

A function type with a receiver allows you to define a block of code that acts as if it's inside an object's scope. That means you can access the object's methods and properties without extra qualifiers. Here's a simple example:

```kolin
class Printer {
    fun show(message: String) = println(message)
}

// Function type with a receiver: Printer.() -> Unit
fun usePrinter(block: Printer.() -> Unit) {
    val printer = Printer()
    printer.block()
}

fun main() {
    usePrinter {
        show("Hello from inside the Printer scope")
    }
}
```

Inside `usePrinter { ... }`, you call `show()` directly. That's because the block is defined as `Printer.() -> Unit`, so it behaves like a method on `Printer`. We'll use this idea to write our DSL for building objects.

## Building on data classes with a type-safe builder

In my [earlier article](https://dev.to/lucasfugisawa/kotlin-design-patterns-simplifying-the-builder-pattern-3e7g), we had a `Car` data class with properties like `make`, `model`, and `year`. Now we will replace `year` with `announcementDate` of type `LocalDate`. We will create a DSL so you can write `car { ... }` blocks to build `Car` objects. Let's start with a simple `Car`:

```kotlin
data class Car(
    val make: String = "N/A",
    val model: String = "N/A",
    val announcementDate: LocalDate? = null,
)
``` 
A type-safe builder, or a DSL, will allow us to create cars like this:
```kotlin
val honda = car {
    make = "Honda"
    model = "Civic"
    announcementDate = localDate {
        year = 2025
        month = 2
        day = 15
    }
}
```

Notice the nested `localDate { ... }`. That's a mini-DSL for creating a `LocalDate`. Let's walk through the steps.

### Step 1: Write a builder class for LocalDate

Define a builder for `LocalDate`. Store `year`, `month`, and `day`, then build a `LocalDate`:

```kotlin
class LocalDateBuilder {
    var year: Int = 1970
    var month: Int = 1
    var day: Int = 1

    fun build(): LocalDate = LocalDate.of(year, month, day)
}

fun localDate(block: LocalDateBuilder.() -> Unit): LocalDate {
    val builder = LocalDateBuilder()
    builder.block()
    return builder.build()
}
```

This lets you call `localDate { ... }` and set the date parts in a small block.

Inside the `LocalDateBuilder`, each property (`year`, `month`, `day`) is a mutable field. You set them in the block passed to `localDate`. The `build()` function then transforms those fields into a `LocalDate`. 

This pattern is flexible. You could add validation or adjust default values if you need stricter control.

### Step 2: Write a builder class for Car

Next, create a builder for `Car`. You will keep `make`, `model`, and `announcementDate` as mutable fields:

```kotlin
class CarBuilder {
    var make: String = "N/A"
    var model: String = "N/A"
    var announcementDate: LocalDate? = null

    fun build(): Car = Car(
        make = make,
        model = model,
        announcementDate = announcementDate
    )
}
```
The `CarBuilder` holds properties that match the data class fields. In your `car` block, you assign values to `make`, `model`, or `announcementDate`. 

When you call `build()`, it creates a `Car` object with whatever state you defined. This separation lets you add checks or transformations if you want more than just copying values.

### Step 3: Write the DSL entry function

Lastly, define a helper function to create a `CarBuilder`, run the block, and return the final `Car`:

```kotlin
fun car(block: CarBuilder.() -> Unit): Car {
    val builder = CarBuilder()
    builder.block()
    return builder.build()
}
```
You can now write:
```kotlin
val civic = car {
    make = "Honda"
    model = "Civic"
}
```
The `car(block: CarBuilder.() -> Unit)` function is the entry point to your DSL. It creates a `CarBuilder`, then runs the given block in that builder's scope. Finally, it calls `build()` to return the finished `Car`. 

You can return different implementations if needed. For example, you could add logic to decide if certain fields are valid or required.

## Comparing approaches: traditional builders, data classes, and DSLs

Let's compare how you might build a `Car` with three different patterns:

### 1. Traditional builder

This often appears in Java or Kotlin code that mimics the same style.

```kotlin
class Car private constructor(
    val make: String,
    val model: String,
    val announcementDate: LocalDate
) {
    class Builder {
        private var make: String = "N/A"
        private var model: String = "N/A"
        private var announcementDate: LocalDate = LocalDate.of(1970, 1, 1)

        fun withMake(make: String) = apply { this.make = make }
        fun withModel(model: String) = apply { this.model = model }
        fun withAnnouncementDate(date: LocalDate) = apply { this.announcementDate = date }

        fun build(): Car = Car(make, model, announcementDate)
    }
}

// Usage
val carWithBuilder = Car.Builder()
    .withMake("Honda")
    .withModel("Civic")
    .withAnnouncementDate(LocalDate.of(2025, 2, 15))
    .build()
```

This pattern is clear about each property you set, but it requires a separate builder class and methods like `withMake`, `withModel`, etc.

### 2. Kotlin data classes with named parameters

Kotlin's data classes can replace many builder use cases thanks to named parameters and default arguments.


```kotlin
data class Car(
    val make: String = "N/A",
    val model: String = "N/A",
    val announcementDate: LocalDate = LocalDate.of(1970, 1, 1)
)

val carWithDataClass = Car(
    make = "Honda",
    model = "Civic",
    announcementDate = LocalDate.of(2025, 2, 15)
)
```
You don't need extra builder methods. You can skip arguments you don't need, thanks to default values.

### 3. DSL (type-safe builder)

When your object or configuration gets deeper, you might want a more expressive way to nest properties. DSLs allow you to build objects within code blocks, making them easy to read or extend.

```kotlin
val carWithDSL = car {
    make = "Honda"
    model = "Civic"
    announcementDate = localDate {
        year = 2025
        month = 2
        day = 15
    }
}
```

Each block runs with the appropriate builder scope. You don’t repeat object names, and you can nest DSL calls for related objects like `LocalDate`.

When your object or configuration becomes more complex, you might want a more expressive way to nest properties. DSLs let you build objects in code blocks, making them easier to read and extend. Each block runs in the correct builder scope. You don’t repeat object names, and you can nest DSL calls for related objects like `LocalDate`. You also gain extra flexibility because a DSL block is actual Kotlin code. That means you can do more than simple assignments. For example, you can perform validations, logging, or other conditional logic inside the builder:

```kotlin
// Suppose you have a function that fetches or reads car data from some file.
// This can be a placeholder for actual file IO and parsing.
// For example, you might parse JSON into a CarData object.
fun loadCarData(path: String): CarData = CarData(
    make = "Nissan",
    model = "Leaf",
    isElectric = true,
)

// A simple data class to represent the loaded car data.
data class CarData(
    val make: String,
    val model: String,
    val isElectric: Boolean,
)

// DSL-based creation of a Car with integrated IO/data loading.
val carWithDSL = car {
    println("Starting the car creation process...")

    val config = loadCarData("car_config.json")

    make = config.make
    model = config.model

    if (config.isElectric) {
        announcementDate = localDate {
            year = 2025
            month = 3
            day = 1
        }
    } else {
        announcementDate = localDate {
            year = 2025
            month = 3
            day = 31
        }
    }

    require(model.isNotBlank()) { "Model must not be blank." }

    println("Car creation block finished. Building the Car object now.")
}
```

## Kotlin's features enabling DSLs / type-safe builders

-   **Named and Default Parameters**:  These let you call constructors in a flexible way. They also reduce the need for multiple constructors.
-   **Function Types with a Receiver**: This feature drives DSL creation. You define blocks that operate in the context of a specific object. That means you access properties and methods directly, which improves readability.

### Pros of the DSLs

-   **Natural Nesting:** You can nest objects that relate to each other. For example, a `car { engine { ... } }` DSL can clarify how parts fit together. This nesting helps you keep code organized and shows logical groupings right inside the builder.
-   **Readability:** Each block reads like a small sentence about the object you are constructing. Instead of calling many methods in a row, you see a structured layout. Future readers can follow your logic like reading a short story about how the object is built.
-   **Less Boilerplate:** You don’t repeat the same method calls or property assignments. You only define your builder classes once. Then you write short blocks that do the configuration. This saves time, especially as your objects grow in complexity.
-   **Type-Safe:** The DSL ensures you only set valid properties. The compiler checks everything. You also get IDE support, like auto-completion. This reduces mistakes and helps you see what options are available at a glance.

### Use cases examples

1.  **User profile setup**  
    You can nest parts of a user profile: 
    ```kotlin
    val profile = userProfile { 
	    name = "Alice" 
	    age = 30 
	    address { 
		    street = "Main Street" 
		    city = "Springfield" 
		} 
	}
    ```
2.  **UI configuration**  
    You might build a layout tree in a natural way: 
    ```kotlin
    val mainPanel = panel { 
	    panel { 
		    field { 
			    label = "Email"
			    type = TEXT
			}
	    }
	    button { label = "OK" } 
	    button { label = "Cancel" } 
	}
    ```
3.  **Emails or reports**  
    You can add sections or attachments in blocks:
    ```kotlin
    val weeklyReportEmail = email { 
	    subject = "Weekly Report" 
	    to("manager@company.com") 
	    to("team@company.com") 
	    body = "Here is our weekly progress..." 
	}    
    ```
4.  **Complex file generation**  
    For XML or JSON, DSLs let you organize nested tags and fields with clarity.
    ```kotlin
    val document = xml("catalog") { 
	    element("book") { 
		    element("title") { } 
		    element("author") { } 
		} 
		element("book") { 
			element("title") { } 
			element("author") { } 
		} 
	}
    ```

This article was originally posted to my Lucas Fugisawa on Kotlin blog, at: https://fugisawa.com/taking-kotlin-builders-to-the-next-level-a-type-safe-dsl-approach/

In each of these scenarios, DSLs help you avoid confusion when building complex structures.


## Final thoughts

Type-safe builders (DSLs) bring a new layer of readability to your code. You maintain the simplicity of data classes, then add a more expressive syntax for your objects. You write `car { ... }` blocks and set properties in a natural way. Test them in your next project, and see how they simplify your object creation logic.

---
To explore more about Kotlin-related topics, subscribe to my newsletter at  [https://fugisawa.com/](https://fugisawa.com/) and stay tuned for more insights and updates.
