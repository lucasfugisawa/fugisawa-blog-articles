# Kotlin Design Patterns: Simplifying the Observer Pattern


The Observer Pattern is a behavioral design pattern where an object (the subject) maintains a list of its dependents (observers), and notifies them automatically of any state changes.

This pattern ensures that multiple objects are notified when certain state changes occur. It's widely used in implementing distributed event handling systems.

The Observer Pattern decouples the subject from its observers and allows for dynamic addition or removal of observers.

## Approaches in Java

Let’s consider a weather station that notifies displays when the temperature changes. Check this **Java** example:
```java
// Observer Interface:
public interface Observer {
    void update(float temperature);
}

// Subject Interface:
public interface Subject {
    void registerObserver(Observer o);
    void removeObserver(Observer o);
    void notifyObservers();
}

// Concrete Subject:
public class WeatherStation implements Subject {
    private float temperature;
    private List<Observer> observers = new ArrayList<>();

    public void setTemperature(float temperature) {
        this.temperature = temperature;
        notifyObservers();
    }

    public void registerObserver(Observer o) {
        observers.add(o);
    }

    public void removeObserver(Observer o) {
        observers.remove(o);
    }

    public void notifyObservers() {
        for (Observer observer : observers) {
            observer.update(temperature);
        }
    }
}

// Concrete Observer:
public class Display implements Observer {
    private String displayId;

    public Display(String id) {
        this.displayId = id;
    }

    public void update(float temperature) {
        System.out.println(displayId + ": Temperature updated: " + temperature);
    }
}

// Client:
public class WeatherApp {
    public static void main(String[] args) {
        WeatherStation station = new WeatherStation();
        Display display1 = new Display("Display 1");
        Display display2 = new Display("Display 2");

        station.registerObserver(display1);
        station.registerObserver(display2);
        station.setTemperature(30f);
    }
}
```
In this **Java** example, `WeatherStation` is the subject, and `Display` is an observer that updates when the temperature changes. Both displays `display1` and `display2` are notified (`update(float temperature)`) when the station temperature changes.

### Functional approach in Java 8+
You can use **Java 8+** functional features to simplify the Observer pattern and achive a very similar approach using functional interfaces and Java's lambda expressions.

Here’s how you can adapt the Weather Station example:
```java
import java.util.ArrayList;
import java.util.List;
import java.util.function.Consumer;

public class WeatherStation {
    private float temperature;
    private List<Consumer<Float>> onTemperatureChangeListeners = new ArrayList<>();

    public void setTemperature(float temperature) {
        this.temperature = temperature;
        notifyTemperatureChange(temperature);
    }

    private void notifyTemperatureChange(float newTemperature) {
        onTemperatureChangeListeners.forEach(listener -> listener.accept(newTemperature));
    }

    public void onTemperatureChange(Consumer<Float> listener) {
        onTemperatureChangeListeners.add(listener);
    }
}

// Usage:
WeatherStation station = new WeatherStation();

// Registering listeners using lambda expressions:
station.onTemperatureChange(temp -> System.out.println("Display 1: Temperature updated to " + temp));
station.onTemperatureChange(temp -> System.out.println("Display 2: Temperature updated to " + temp));

// Simulating a temperature change:
station.setTemperature(30f);
```
In this **Java** example:

-   The `WeatherStation` class maintains a list of `Consumer<Float>` objects, which are functional interfaces in Java that can be used with lambda expressions.
-   The `onTemperatureChange` method allows registering `Consumer` lambda expressions that will be called when the temperature changes.
-   When `setTemperature` is called, it triggers `notifyTemperatureChange`, which executes all registered lambda expressions with the new temperature.

## Kotlin's Approach:
**Kotlin** provides observer delegates feature. `Delegates.observable()` simplifies the observer pattern implementation for objects properties changes:

You can combine observer delegates to observe property changes and higher-order functions to register callbacks.
```kotlin
import kotlin.properties.Delegates

class WeatherStation {
    // Observable property with callbacks:
    var temperature: Float by Delegates.observable(0f) { _, _, newValue ->
            onTemperatureChangeListeners.forEach { it(newValue) }
    }

    // List of callbacks:
    private val onTemperatureChangeListeners = mutableListOf<(Float) -> Unit>()

    // Function to add callbacks:
    fun onTemperatureChange(listener: (Float) -> Unit) {
        onTemperatureChangeListeners.add(listener)
    }
}

// Client:
fun main() {
    val station = WeatherStation()

    // Registering callbacks:
    station.onTemperatureChange { println("Display 1: Temperature updated to $it") }
    station.onTemperatureChange { println("Display 2: Temperature updated to $it") }

    // Simulating temperature change:
    station.temperature = 30f
}
```

In this **Kotlin** implementation:
-   The `temperature` property in `WeatherStation` is an observable property. When it changes, all registered callbacks in `onTemperatureChangeListeners` are invoked.
-   The `onTemperatureChange` method allows registration of lambda expressions (callbacks) that react to temperature changes.
-   Clients register callbacks to `WeatherStation` which get executed whenever the `temperature` property changes.

#### Benefits of This Approach:
-   **Simplicity:** This approach simplifies the observer pattern by eliminating the need for interfaces and concrete observer classes.
-   **Flexibility:** It's easy to add or remove behaviors (callbacks) dynamically at runtime.
-   **Expressiveness:** Leveraging Kotlin's language features results in more readable and maintainable code.

### Kotlin Features Simplifying the Observer Pattern
- **Higher-Order Functions and Lambdas:** Enables concise observer implementation using functions (behavior) as parameters.
- **Delegated Properties (`Delegates.observable()`):** Simplifies property change observation.


## Final thoughts
Kotlin’s `Delegates.observable()` offers a concise and powerful alternative to the traditional observer pattern, especially for simple use cases. For more complex scenarios, the standard implementation is still useful and can be implemented concisely.
