# Kotlin Design Patterns: Simplifying the Prototype Pattern

We use the Prototype Pattern when creating new instances from scratch is more expensive than copying existing ones.

So, instead of instantiating new objects, you can have a prototype from which clones / copies are made.

## Traditional Approach in Java:

In this Java example, `GraphicElement` represents a complex graphic element with complex initialization logic, such as loading textures, calculating geometry, etc.:

```java
interface PrototypeCapable extends Cloneable {  
    PrototypeCapable clone() throws CloneNotSupportedException;  
}  
  
class GraphicElement implements PrototypeCapable {  
    private String color;  
    private List<String> points; // Represents complex geometry  
    private String texture;  
    // Getters and setters ommited.  
  
    public GraphicElement(String color, List<String> points, String texture) {  
        this.color = color;  
        this.points = points;
        this.texture = texture;
        if (this.texture == null) {
            this.texture = "Some complex initialization logic here (loading textures, calculating geometry, etc.)";
        }
}  
  
  @Override  
    public GraphicElement clone() throws CloneNotSupportedException {  
        return (GraphicElement) super.clone();  
    }  
}  
  
// Usage:  
List<String> initialPoints = Arrays.asList("x1", "y1", "x2", "y2");  
GraphicElement originalElement = new GraphicElement("Red", initialPoints, "BrickTexture");  
GraphicElement clonedElement = originalElement.clone();  
clonedElement.setColor("Blue");  
System.out.println("Cloned Element Color: " + clonedElement.getColor()); // Output: Blue
```

In this example, you can see cloning is used to create a new element `clonedElement` based on the existing `originalElement`, instead of instatiating a new object from scratch. Also, the `color` is changed from red to blue on the `clonedElement`.

## Kotlin's Approach:
Kotlin allows for an efficient and concise way to implement cloning of complex objects.

```kotlin
data class GraphicElement(
    val color: String,
    val points: List<String>,
    var texture: String? = null,
) {
    init {
        if (texture == null) {
            texture = "Some complex initialization logic here (loading textures, calculating geometry, etc.)"
        }
    }
}

// Usage:
val initialPoints = listOf("x1", "y1", "x2", "y2")
val originalElement = GraphicElement("Red", initialPoints)
val clonedElement = originalElement.copy(color = "Blue") // Modifying color while cloning
```
In this Kotlin example, `GraphicElement` is a data class used for creating complex graphic elements. The `copy` method simplifies the process of cloning and modifying these elements.
Notice that the `color` for the `clonedElement` can be set / changed earlier, while copying the `originalElement`.


### Kotlin Features Simplifying the Prototype Pattern

1.  **Data Classes**: Offer an inbuilt `copy` method, streamlining the creation of prototype instances.
2.  **`copy` method**: Allows changing specific properties while copying, allowing for more concise and expressive code.

## Final Thougths
Kotlin's approach to the Prototype pattern showcases its efficiency and simplicity. Data classes and their built-in `copy` method make the cloning process straightforward, simplifying the pattern implementation while maintaining functionality.