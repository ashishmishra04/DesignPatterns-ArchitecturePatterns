
# Factory Pattern

## Definition

The **Factory Pattern** is a **creational design pattern** that provides an interface for creating instances of a class, while letting subclasses (or concrete factory classes) decide which class to instantiate. It promotes loose coupling by eliminating the need to bind application-specific classes into the code directly.

## Must Read
https://refactoring.guru/design-patterns/factory-method
![alt text](image.png)
The Factory Method pattern suggests that you replace direct object construction calls (using the new operator) with calls to a special factory method. Don’t worry: the objects are still created via the new operator, but it’s being called from within the factory method. Objects returned by a factory method are often referred to as products.



Instead of calling a constructor directly:
```java
new ConcreteProduct()
```
you call a factory method:
```java
Product p = factory.createProduct();
```

There are several variants:
- **Simple Factory** (not a GoF pattern, but commonly used)
- **Factory Method** (GoF pattern)
- **Abstract Factory** (GoF pattern – creates families of related objects)

Here's a clear, practical comparison between **Simple Factory**, **Factory Method**, and **Abstract Factory** — the three most common creational patterns in OOP (especially in C#).

| Feature                  | Simple Factory                          | Factory Method                                | Abstract Factory                                   |
|--------------------------|------------------------------------------|-----------------------------------------------|-----------------------------------------------------|
| Is it a real GoF pattern?| No (it's a common idiom)                | Yes (GoF pattern)                             | Yes (GoF pattern)                                   |
| Where is the creation logic? | In one static class/method              | In a virtual/abstract method in a base class  | In a family of factory classes/interfaces           |
| How many product types?  | Usually one hierarchy                   | One hierarchy (but many concrete creators)    | Multiple related hierarchies (families of products) |
| Extensibility (open/closed) | Not open for extension                  | Open for extension (add new creator classes)  | Open for extension (add new factory classes)        |
| Adding a new product     | Modify the factory (violates OCP)       | Add new creator class (no modification)       | Add new concrete factory class (no modification)    |
| Best when                | Simple cases, quick prototyping         | You expect new types in the future            | You have families of related objects (e.g. UI themes) |

### 1. Simple Factory (Not a GoF pattern, but very common)

```csharp
public enum DatabaseType { SqlServer, MySql, PostgreSql }

public static class DatabaseFactory
{
    public static IDatabase Create(DatabaseType type)
    {
        return type switch
        {
            DatabaseType.SqlServer   => new SqlServerDatabase(),
            DatabaseType.MySql       => new MySqlDatabase(),
            DatabaseType.PostgreSql  => new PostgreSqlDatabase(),
            _ => throw new ArgumentException("Unknown type")
        };
    }
}

// Usage
IDatabase db = DatabaseFactory.Create(DatabaseType.SqlServer);
```

**Pros**: Super simple, easy to understand  
**Cons**: Not extensible — adding a new database requires changing the factory (violates Open/Closed Principle)

### 2. Factory Method (GoF Pattern)

You define an abstract creator with a virtual factory method. Concrete creators override it.

```csharp
public abstract class Dialog
{
    public void Render() 
    {
        IButton button = CreateButton();  // Factory Method
        button.Render();
    }

    public abstract IButton CreateButton(); // This is the Factory Method
}

public class WindowsDialog : Dialog
{
    public override IButton CreateButton() => new WindowsButton();
}

public class WebDialog : Dialog
{
    public override IButton CreateButton() => new HtmlButton();
}

// Usage – no switch/if needed!
Dialog dialog = Runtime.IsWindows() ? new WindowsDialog() : new WebDialog();
dialog.Render();
```

**Pros**: Adding a MacDialog + MacButton requires only new classes — no existing code changes  
**Cons**: Many small classes (one creator per product type)

### 3. Abstract Factory (GoF Pattern)

When you have **families** of related products (e.g. Windows UI vs Mac UI vs Web UI).

```csharp
public interface IGuiFactory
{
    IButton CreateButton();
    ICheckbox CreateCheckbox();
}

public class WindowsFactory : IGuiFactory
{
    public IButton CreateButton() => new WindowsButton();
    public ICheckbox CreateCheckbox() => new WindowsCheckbox();
}

public class MacFactory : IGuiFactory
{
    public IButton CreateButton() => new MacButton();
    public ICheckbox CreateCheckbox() => new MacCheckbox();
}

// Usage
IGuiFactory factory = Runtime.IsMac() ? new MacFactory() : new WindowsFactory();
IButton button = factory.CreateButton();
ICheckbox check = factory.CreateCheckbox();
```

**Pros**: Guarantees consistency — you can't mix Windows button with Mac checkbox  
**Cons**: Adding a new product (e.g. Scrollbar) requires changing all factories

### Quick Decision Table

| Scenario                                          | Use This Pattern       |
|---------------------------------------------------|------------------------|
| Just need to create objects based on a string/enum| **Simple Factory**     |
| You will add new product types later              | **Factory Method**     |
| You have multiple families (themes, platforms)    | **Abstract Factory**   |
| You want maximum flexibility and OCP compliance   | Factory Method or Abstract Factory |
| Learning/prototyping/small app                    | Simple Factory         |

### Real-World Analogy

| Pattern            | Analogy                                      |
|--------------------|-----------------------------------------------|
| Simple Factory     | A single restaurant menu with a switch for food |
| Factory Method     | Each franchise (McDonald's, KFC) has its own kitchen making its own burger |
| Abstract Factory   | A car manufacturer (Toyota) that makes entire families: engine + transmission + interior |

### Summary (One-liner)

- **Simple Factory** → one static method with `switch/if`  
- **Factory Method** → one product hierarchy, many creators (subclasses decide)  
- **Abstract Factory** → families of products, one factory per family
