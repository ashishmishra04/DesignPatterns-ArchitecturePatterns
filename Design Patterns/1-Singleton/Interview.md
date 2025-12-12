**Singleton Pattern interview questions with clear answers**. 

---

## **1. What is the Singleton Pattern?**

A Singleton ensures a class has **only one instance** during the lifetime of an application and provides a **global point of access** to that instance. The class controls its own creation, usually with a private constructor.

---

## **2. Why do we use the Singleton Pattern?**

It’s used when a shared resource must be instantiated **once** and reused everywhere.
Examples include configuration manager, logger, cache, connection pool, or feature toggles.

---

## **3. How do you implement a Singleton in C#?**

**Without `Lazy<T>` — thread-safe double-checked locking:**

```csharp
public sealed class Singleton
{
    private static Singleton _instance;
    private static readonly object _lock = new object();

    private Singleton() { }

    public static Singleton Instance
    {
        get
        {
            if (_instance == null)
            {
                lock (_lock)
                {
                    if (_instance == null)
                        _instance = new Singleton();
                }
            }
            return _instance;
        }
    }
}
```

---

## **4. What is double-checked locking? Why do we need it?**

It’s a pattern where we check for `null` **twice**:

1. Once before acquiring the lock
2. Once inside the lock

This ensures **thread safety** while avoiding unnecessary locking, which improves performance.

---

## **5. Why is the constructor private in a Singleton?**

To prevent outside code from creating instances with `new`.
The class itself must control all instance creation.

---

## **6. How do you break a Singleton?**

A few ways can break it:

* **Reflection** (can invoke the private constructor)
* **Serialization** (creates a new object on deserialization)
* **Cloning**
* **Multi-threading issues** in poorly written Singletons

---

## **7. How do you prevent breaking a Singleton?**

* Seal the class
* Use double-checked locking or `Lazy<T>` for thread safety
* For serialization, implement `ISerializable` and return the same instance in `GetObjectData`
* Avoid exposing state that allows duplication

---

## **8. Is Singleton an anti-pattern?**

It can be—when misused.

Issues:

* Hidden global state
* Tight coupling
* Harder to test
* Concurrency smells

Use it only for truly global objects—configuration, logging, scheduler, etc.

---

## **9. Is Singleton the same as static class?**

No.

**Singleton**

* Allows lazy initialization
* Implements interfaces
* Can be passed as a dependency
* Supports inheritance (unless sealed)
* Can maintain state

**Static class**

* Cannot be instantiated
* Only holds static methods
* No interfaces or DI

Singletons are object-oriented; static classes are not.

---

## **10. How do you create a Singleton using eager initialization?**

This one creates the instance immediately:

```csharp
public sealed class EagerSingleton
{
    private static readonly EagerSingleton _instance = new EagerSingleton();

    private EagerSingleton() { }

    public static EagerSingleton Instance => _instance;
}
```
### Lazy vs Eager
https://dev.to/albertbennett/lazy-vs-eager-initialization-abn

Used when the instance is cheap and always needed.

---

## **11. What are real-world use cases of Singleton in .NET?**

A few strong examples:

* Logging service
* Configuration manager
* Caching layer
* Database connection pool
* Scheduler or job manager
* Feature toggles
* Authentication providers

These naturally should exist only once.

---

## **12. What is the difference between Singleton and Dependency Injection (DI)?**

DI gives you **controlled lifecycle** such as Singleton, Scoped, Transient.
But DI is flexible and testable.

A pure Singleton hardcodes its lifecycle, which reduces testability.
Modern .NET prefers **DI-based Singleton services** rather than manual Singleton classes.

---

## **13. Why is the Singleton class sealed?**

The class is sealed to ensure:

1️⃣ No subclass can create multiple instances

Allowing inheritance means a subclass could create new instances, violating the Singleton’s purpose.

2️⃣ Prevent modification of the Singleton behavior

A derived class could override methods or change constructor behavior, breaking the controlled instantiation process.

3️⃣ Ensures thread safety and predictable behavior

Singleton relies on a controlled initialization path. Inheritance introduces complexity that can create unintended instances or race conditions.

4️⃣ Design clarity

Sealing the class explicitly communicates that the class is not designed for extension — it is a standalone global object.

---


