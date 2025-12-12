
# 🟦 **1. Mediator Pattern**

**The Mediator Pattern defines an object that centralizes communication between multiple objects, reducing direct dependencies between them.**

Put simply:

> Instead of objects calling each other directly, they communicate through a mediator.

This leads to **loose coupling**, **cleaner code**, and easier **modification/extension**.

---

# 🟩 **2. Why We Need the Mediator Pattern**

We use the Mediator pattern when:

* Many classes/components need to interact with each other
* Direct communication leads to **spaghetti dependencies**
* We want to avoid raising coupling across layers
* We want to promote **single responsibility** & **clean architecture**

Common problems that Mediator solves:

### ❌ Without Mediator

Classes call each other → **tight coupling**, hard to test, chain reactions.

### ✅ With Mediator

Classes send requests/events through the mediator → **centralized, controlled communication**.

---

# 🟧 **3. Mediator Pattern – Simple C# Example (No Libraries)**

### ✨ Example: A Chat Room System

#### **Mediator Interface**

```csharp
public interface IChatMediator
{
    void SendMessage(string message, User user);
    void Register(User user);
}
```

#### **Concrete Mediator**

```csharp
public class ChatMediator : IChatMediator
{
    private readonly List<User> _users = new();

    public void Register(User user)
    {
        _users.Add(user);
    }

    public void SendMessage(string message, User user)
    {
        foreach (var u in _users)
        {
            if (u != user)
            {
                u.Receive(message);
            }
        }
    }
}
```

#### **Colleague Class**

```csharp
public abstract class User
{
    protected IChatMediator _mediator;
    protected string _name;

    protected User(IChatMediator mediator, string name)
    {
        _mediator = mediator;
        _name = name;
    }

    public abstract void Receive(string message);

    public void Send(string message)
    {
        Console.WriteLine($"{_name} sends: {message}");
        _mediator.SendMessage(message, this);
    }
}
```

#### **Concrete Colleague**

```csharp
public class ChatUser : User
{
    public ChatUser(IChatMediator mediator, string name) 
        : base(mediator, name) { }

    public override void Receive(string message)
    {
        Console.WriteLine($"{_name} receives: {message}");
    }
}
```

#### **Usage**

```csharp
var mediator = new ChatMediator();

var u1 = new ChatUser(mediator, "John");
var u2 = new ChatUser(mediator, "Sarah");
var u3 = new ChatUser(mediator, "Mike");

mediator.Register(u1);
mediator.Register(u2);
mediator.Register(u3);

u1.Send("Hello everyone!");
```

---

# 🟦 **4. Mediator Pattern Using MediatR (Industry Standard)**

MediatR is the most popular .NET library for implementing the Mediator pattern, especially in **CQRS** and **Clean Architecture**.

---

## 🟩 **Command Example with MediatR**

### **Install**

```
dotnet add package MediatR.Extensions.Microsoft.DependencyInjection
```

### **Step 1 – Create a Request (Command)**

```csharp
public record CreateOrderCommand(string ProductName, int Quantity) 
    : IRequest<string>;
```

---

### **Step 2 – Create Handler**

```csharp
public class CreateOrderHandler 
    : IRequestHandler<CreateOrderCommand, string>
{
    public Task<string> Handle(CreateOrderCommand request, CancellationToken cancellationToken)
    {
        // Business logic
        return Task.FromResult($"Order created for {request.ProductName} (Qty: {request.Quantity})");
    }
}
```

---

### **Step 3 – Register MediatR in Program.cs**

```csharp
builder.Services.AddMediatR(typeof(Program));
```

---

### **Step 4 – Call via Mediator**

```csharp
public class OrderController : ControllerBase
{
    private readonly IMediator _mediator;

    public OrderController(IMediator mediator)
    {
        _mediator = mediator;
    }

    [HttpPost("create-order")]
    public async Task<IActionResult> CreateOrder(CreateOrderCommand command)
    {
        var result = await _mediator.Send(command);
        return Ok(result);
    }
}
```

---

# 🟧 **5. Real-World Example**

### **CQRS + MediatR in a Microservices Architecture**

In your modernization project:

* Angular frontend
* .NET Core microservices
* Auth0 authentication
* Multiple bounded contexts

Instead of controllers calling services tightly, you use MediatR:

**Controller → Mediator → Handler → Domain Logic → Repository**

This achieves:

✔ No direct service dependency
✔ Centralized cross-cutting logic (logs, caching, validation)
✔ Clean separation of commands, queries, and events
✔ Highly testable code

---

# 🟩 **6. Summary (Interview-Ready)**

| Concept       | Explanation                                    |
| ------------- | ---------------------------------------------- |
| **Purpose**   | Remove direct dependencies between components  |
| **How**       | All communication goes through a mediator      |
| **Benefit**   | Loosely coupled system, easier maintenance     |
| **Use Cases** | CQRS, chat systems, UI controls, microservices |
| **Framework** | MediatR is the most used .NET implementation   |

---

If you want, I can provide:

✅ Mediator vs Observer comparison
✅ Full CQRS template using MediatR
✅ MediatR pipeline behaviors (logging, validation, retry, caching)

Just tell me!
