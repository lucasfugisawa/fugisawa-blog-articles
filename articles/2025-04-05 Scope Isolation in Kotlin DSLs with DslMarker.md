# Scope Isolation in Kotlin DSLs with `@DslMarker`

In the previous article "Taking Kotlin Builders to the Next Level: A Type-Safe DSL Approach",  we explored how Kotlin's type-safe builders allow you to build expressive and concise DSLs. But as soon as your DSL starts nesting structures, something sneaky can happen: **receiver conflicts**.

Let's take a deeper look at this problem and how Kotlin's `@DslMarker` annotation solves it, making your DSLs safer and more robust.

But if you haven't read that previous article yet, I strongly recommend you to pause here for a while and do it first.

## So what's the problem, in fact?

When you nest multiple DSL blocks, like this...

```kotlin 
val car = car {
    make = "Honda"
    model = "Civic"
    announcementDate = localDate {
        year = 2025
        month = 2
        day = 15
    }
}
```
..., you're creating **nested lambdas with receivers**, which are also **closures**.

In Kotlin, lambdas *are* closures: they carry with them access to everything in the outer scope, including the outer receiver (`CarBuilder`). So inside the `localDate { ... }` block, the compiler sees both `LocalDateBuilder` _and_ `CarBuilder` in scope. That means inside the `localDate { ... }` block, you still have access to the outer `CarBuilder`, even though you're writing in the context of `LocalDateBuilder`. And that can get dangerous fast.

This can lead to subtle and unintended behavior:
```kotlin
announcementDate = localDate {
    make = "Oops" // Refers to CarBuilder, not LocalDateBuilder
}
```
The `make` property is not part of `LocalDateBuilder`, but since the lambda captures the outer receiver (`CarBuilder`), it's still accessible. That’s a classic receiver conflict.

### What if both builders have some properties with the same name?

Let's say, hypothetically, that both `CarBuilder` and `LocalDateBuilder` define a property called `year`. This is _not unrealistic_ — maybe `CarBuilder` tracks the model year, while `LocalDateBuilder` builds the date when that car model was first announced.

```kotlin
class CarBuilder {
    var make: String = "N/A"
    var model: String = "N/A"
    var year: Int = 0 // Model year!
    var announcementDate: LocalDate? = null
    ...
}

class LocalDateBuilder {
    var year: Int = 1970
    var month: Int = 1
    var day: Int = 1
    ...
}
```
Now let's write a nested DSL block:

```kotlin
val car = car {
    make = "Toyota"
    model = "Corolla"
    year = 2023 // OK: CarBuilder.year

    announcementDate = localDate {
        year = 2025 // Is this LocalDateBuilder.year or CarBuilder.year?
    }
}
```
Surprisingly, both `year` properties are accessible inside the `localDate` block. This makes the code **ambiguous and error-prone**, because:

-   If `year = 2025` targets the **outer receiver**, the `LocalDate` may be built with a default year (1970), leading to silent bugs.
-   The code _looks_ correct, and it _compiles_ - but it does an unexpected thing.

If you want to be sure you're writing to the intended scope, you can always disambiguate manually using `this`:

```kotlin
announcementDate = localDate {
    this.year = 2025 // Clearly refers to LocalDateBuilder.year
    this@car.year = 2023 // Explicitly refers to CarBuilder.year
}
```
But of course, this adds verbosity and cognitive overhead. Using `@DslMarker` gives you a safer default: the compiler protects you from accidental access.

## How `@DslMarker` solves this?


By annotating both `CarBuilder` and `LocalDateBuilder` with the same `@CarDsl`, Kotlin will restrict visibility to **only the closest receiver** within each block. That marker tells the compiler: _these receivers belong to the same DSL, don't mix them up_.

First, we need to define a DSL marker annotation:
```kotlin
@DslMarker
@Target(AnnotationTarget.CLASS, AnnotationTarget.TYPE)
annotation class CarDsl
```

Then, we need to annotate our builders using our new DSL marker annotation:
```kotlin
@CarDsl
class CarBuilder {
    var make: String = "N/A"
    var model: String = "N/A"
    var announcementDate: LocalDate? = null

    fun build(): Car = Car(make, model, announcementDate)
}

@CarDsl
class LocalDateBuilder {
    var year: Int = 1970
    var month: Int = 1
    var day: Int = 1

    fun build(): LocalDate = LocalDate.of(year, month, day)
}
```

Now, if we accidentally try this...
```kotlin
announcementDate = localDate {
    year = 2025
    model = "Oops" // ❌ Error: model is from CarBuilder, not in scope here
}
```
..., Kotlin will **refuse to compile**, forcing you to stay inside the intended builder scope. This helps eliminate bugs **before they happen**,

### But… what if I _really_ need to access the outer scope?

While `@DslMarker` intentionally hides outer receivers to avoid accidental misuse, Kotlin still gives you an escape hatch: **you can explicitly reference outer receivers using labeled lambdas**.

This is helpful in advanced use cases where you genuinely need access to both scopes, but want to do it _on purpose_.

Let's say you want to configure both the `LocalDateBuilder` **and** the outer `CarBuilder` inside the same nested block:

```kotlin
val car = car {
    make = "Toyota"
    model = "Corolla"
    year = 2023

    announcementDate = localDate {
        year = 2025
        month = 2
        day = 15

        make = "Oops" // not allowed due to @DslMarker
		// To access the CarBuilder.make, use  the labeled receiver explicitly
        this@car.make = "Toyota (revised)"
    }
}
```

And, if you have nested builders of the same type, which can cause confusion when using the default labels, you can use custom labels to make it clearer:

```kotlin
val car = outerCar@car {
    // ...

    announcementDate = localDate {
       // ...
        this@outerCar.make = "Toyota (revised)"
    }
}
```

#### Here's what's happening:
-   We've labeled the outer `car` block with `outerCar@` (or used the implicit default `@car` label name)
-   Inside `localDate { ... }`, the `CarBuilder` is no longer directly visible (because of `@DslMarker`)
-   But we can still access it with `this@outerCar` (or with the default `@car`) explicitly referring to the outer receiver.

## Final thoughts

Using the Kotlin DSL markers lets you maintain safe defaults (no accidental access), but still gives you full control when needed. It's the best of both worlds:
-   **Safety by default** with `@DslMarker`
-   **Escape hatch** with labeled receivers when necessary

It's a good idea to use this sparingly — reaching across DSL scopes often indicates the logic might be better separated — but it's great to know the tool is there when you need it.

---

To explore more about Kotlin-related topics, subscribe to my newsletter at [https://fugisawa.com/](https://fugisawa.com/) and stay tuned for more insights and updates.
