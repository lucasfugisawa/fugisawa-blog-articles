# Kotlin Design Patterns: Simplifying the Decorator Pattern

The Decorator Pattern is a flexible alternative to subclassing for extending functionality. It allows behavior to be added to individual objects without affecting the other objects from the same class.

It is particularly useful when changes are needed during runtime. Also, it's useful when subclassing would result in an exponential rise of new classes.

The pattern involves an interface, concrete implementations of this interface, and decorator classes that 'wrap' the original classes. The decorators implement the same interface and forward calls to the wrapped object, optionally adding their own behavior.

## Traditional Approach in Java:

Let's consider a `UserRepository` interface and its implementation. We'll also have a `UsernameValidator` as a decorator to add validation logic.
```java
// Interface:
interface UserRepository {  
    String get(String userName);  
    void set(String userName, String user);  
}  
  
// Concrete Implementation:
class UserRepositoryImpl implements UserRepository {  
    private Map<String, String> userList = new HashMap<>();  
  
    @Override  
    public String get(String userName) {  
        return userList.get(userName);  
    }  
  
    @Override  
    public void set(String userName, String user) {  
        userList.put(userName, user);  
    }  
}  
  
// Decorator:
class UsernameValidator implements UserRepository {  
    private UserRepository repository;  
    private static final int MAX_NAME_LENGTH = 20;  
    private static final int MIN_NAME_LENGTH = 4;  
  
    public UsernameValidator(UserRepository repository) {  
        this.repository = repository;  
    }  
  
    @Override  
    public String get(String userName) {  
        return repository.get(userName);  
    }  
  
    @Override  
    public void set(String userName, String user) {  
        if (userName.length() < MIN_NAME_LENGTH || userName.length() > MAX_NAME_LENGTH) {  
            throw new IllegalArgumentException("User name is not of valid length");  
        }  
        repository.set(userName, user);  
    }  
}
```
In this Java example, `UsernameValidator` adds a validation layer to `UserRepositoryImpl` without modifying its original code.

Notice that the method `get(String userName)` simply delegates to the target `repository`, while `set(String userName, String user)` first decorates (adds new functionality) and then delegates to the target `repository`.

This pattern allows `UsernameValidator` to control how and when it calls into the composed `UserRepository`, providing a clear structure for adding or modifying behavior.

## Kotlin's Approach:

In Kotlin, we actually have two usual implementations. The first one uses composition and is similar to the Java solution. The second one uses Kotlin class delegation features that allow a simpler and more concise solution.

### Decorating by Composition (like the Java solution)
Decoration by composition involves explicitly composing a class with another class it wishes to decorate, rather than using class delegation. Here's the Kotlin implementation of the UserRepository example using decoration by composition:
```kotlin
// Interface:
interface UserRepository {
    fun get(userName: String): String?
    fun set(userName: String, user: String)
}

// Concrete Implementation:
class UserRepositoryImpl : UserRepository {
    private val userList = mutableMapOf<String, String>()
    override fun get(userName: String) = userList[userName]

    override fun set(userName: String, user: String) {
        userList[userName] = user
    }
}

// Decorator using Composition:
class UsernameValidator(private val repository: UserRepository) : UserRepository {

    override fun get(userName: String): String? {
        return repository.get(userName)
    }

    override fun set(userName: String, user: String) {
        require(userName.length in MIN_NAME_LENGTH..MAX_NAME_LENGTH) {
            "User name is not of valid length"
        }
        repository.set(userName, user)
    }

    companion object {
        private const val MAX_NAME_LENGTH = 20
        private const val MIN_NAME_LENGTH = 4
    }
}
```
In this composition-based approach, the `UsernameValidator` class explicitly contains a `UserRepository` instance (`repository`) and uses this instance to perform the actual work. It overrides the `set` method to add validation logic before delegating the setting of the user to the composed `UserRepository` object.

### Decorating by Delegation (using Kotlin class delegation)
Kotlin's features, such as class delegation, make implementing the Decorator Pattern simpler and more concise.

Using the same `UserRepository` example:
```kotlin
// Interface:
interface UserRepository {
    fun get(userName: String): String?
    fun set(userName: String, user: String)
}

// Concrete Implementation:
class UserRepositoryImpl : UserRepository {
    private val userList = mutableMapOf<String, String>
    override fun get(userName: String) = userList[userName]

    override fun set(userName: String, user: String) {
        userList[userName] = user
    }
}

// Decorator with Class Delegation:
class UsernameValidator(private val repository: UserRepository) : UserRepository by repository {

    override fun set(userName: String, user: String) {
        require(userName.length in MIN_NAME_LENGTH..MAX_NAME_LENGTH) {
            "User name is not of valid length"
        }
        repository.set(userName, user)
    }

    companion object {
        private const val MAX_NAME_LENGTH = 20
        private const val MIN_NAME_LENGTH = 4
    }
}
```

In Kotlin, the `UsernameValidator` class uses class delegation (`by repository`) to implement `UserRepository`. This delegates all calls to the provided `UserRepository` instance, except for the overridden `set` method.

### Kotlin Features Simplifying the Decorator Pattern

1.  **Class Delegation:** Kotlin simplifies the Decorator Pattern by allowing delegation of the interface's implementation to another object. This removes the need for explicit forwarding of undecorated methods calls.

## Final Thougths
In Kotlin, the Decorator Pattern is simplified through features like class delegation, enhancing readability and reducing boilerplate code. Understanding these features allows Kotlin developers to implement design patterns in a more efficient and effective manner.
