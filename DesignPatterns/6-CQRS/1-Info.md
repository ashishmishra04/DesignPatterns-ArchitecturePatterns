# CQRS – Command Query Responsibility Segregation  

### Nice Articles
https://medium.com/design-microservices-architecture-with-patterns/cqrs-design-pattern-in-microservices-architectures-5d41e359768c
![alt text](image.png)
https://blog.bytebytego.com/p/a-pattern-every-modern-developer
![alt text](image-1.png)
https://www.codeproject.com/articles/Introduction-to-CQRS#comments-section
![alt text](image-2.png)


## What is CQRS?

> **"Separate the responsibility for reading data (Queries) from writing data (Commands)"**

| Side      | Responsibility                  | Model Type      | Typical Technology         |
|----------|----------------------------------|------------------|-----------------------------|
| **Command** | Create, Update, Delete         | Rich Domain Model, Tasks | EF Core, Events, Outbox    |
| **Query**   | Read, List, Search, Reports    | Simple DTOs       | Dapper, EF Core (read-only), Redis, Elastic |

**CQRS is NOT just "two methods"** — it's an architectural pattern that enables:
- Different models for read vs write
- Independent scaling
- Optimized performance per side
- Cleaner business logic
- Easier implementation of Event Sourcing, Eventual Consistency, etc.

## Why Use CQRS? (Real Benefits)

| Benefit                        | Explanation                                                                 |
|-------------------------------|-----------------------------------------------------------------------------|
| Performance                   | Read model can be heavily denormalized and cached                           |
| Scalability                   | Read and write sides can scale independently                                |
| Simpler business logic        | Commands focus only on validation + state changes                           |
| Better security               | Queries never change data → safer endpoints                                 |
| UI/UX flexibility             | Read model can be tailored exactly to each screen/report                    |
| Enables Event Sourcing        | Natural fit when combined with CQRS                                         |

## When NOT to Use CQRS

- Simple CRUD applications
- Small teams / startups
- Tight deadlines
- You only have basic list/create/edit forms

**Rule of thumb**: Start simple → Add CQRS only when you feel the pain.

## Full Real-World Example: E-Commerce Order System (.NET 8)

### Project Structure (Clean Architecture Style)

```
ECommerce/
├── src/
│   ├── API/                    (Controllers / Minimal APIs)
│   ├── Application/            ← CQRS lives here
│   │   ├── Commands/
│   │   ├── Queries/
│   │   ├── DTOs/
│   │   └── Interfaces/
│   ├── Domain/                 (Entities, Aggregates, Domain Events)
│   └── Infrastructure/         (EF Core, Dapper, MediatR, Outbox)
└── tests/
```

### 1. Domain Model (Write Side)

```csharp
// Domain/Entities/Order.cs
public class Order : AggregateRoot
{
    public Guid Id { get; private set; }
    public string CustomerName { get; private set; } = "";
    public decimal TotalAmount { get; private set; }
    public OrderStatus Status { get; private set; }
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
    private readonly List<OrderItem> _items = new();

    private Order() { } // EF

    public static Order Create(string customerName)
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            Status = OrderStatus.Draft
        };
        order.AddDomainEvent(new OrderCreatedEvent(order.Id, customerName));
        return order;
    }

    public void AddItem(string productName, int quantity, decimal price)
    {
        var item = new OrderItem(Guid.NewGuid(), productName, quantity, price);
        _items.Add(item);
        TotalAmount += quantity * price;

        AddDomainEvent(new OrderItemAddedEvent(Id, productName, quantity));
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Draft)
            throw new InvalidOperationException("Only draft orders can be confirmed");

        Status = OrderStatus.Confirmed;
        AddDomainEvent(new OrderConfirmedEvent(Id));
    }
}

public enum OrderStatus { Draft, Confirmed, Shipped, Cancelled }
```

### 2. Command Side (MediatR + FluentValidation)

Install packages:
```bash
dotnet add package MediatR
dotnet add package MediatR.Extensions.Microsoft.DependencyInjection
dotnet add package FluentValidation.DependencyInjectionExtensions
```

```csharp
// Application/Orders/Commands/CreateOrder/CreateOrderCommand.cs
public record CreateOrderCommand(string CustomerName, List<OrderItemDto> Items) 
    : IRequest<Guid>;

public record OrderItemDto(string ProductName, int Quantity, decimal Price);

// Validator
public class CreateOrderCommandValidator : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderCommandValidator()
    {
        RuleFor(x => x.CustomerName).NotEmpty();
        RuleFor(x => x.Items).NotEmpty();
    }
}

// Handler – This is where business logic lives
public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, Guid>
{
    private readonly IOrderRepository _repository;
    private readonly IUnitOfWork _unitOfWork;

    public CreateOrderCommandHandler(IOrderRepository repository, IUnitOfWork unitOfWork)
    {
        _repository = repository;
        _unitOfWork = unitOfWork;
    }

    public async Task<Guid> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        var order = Order.Create(request.CustomerName);

        foreach (var item in request.Items)
            order.AddItem(item.ProductName, item.Quantity, item.Price);

        order.Confirm(); // business rule: new orders are auto-confirmed

        await _repository.AddAsync(order);
        await _unitOfWork.CommitAsync(ct);

        return order.Id;
    }
}
```

### 3. Query Side (Read Model – Optimized!)

```csharp
// Application/Orders/Queries/GetOrderById/GetOrderByIdQuery.cs
public record GetOrderByIdQuery(Guid Id) : IRequest<OrderDetailDto?>;

public record OrderDetailDto(
    Guid Id,
    string CustomerName,
    decimal TotalAmount,
    string Status,
    DateTime CreatedAt,
    List<OrderItemDto> Items);

// Handler – Uses Dapper or EF Read-Only Context
public class GetOrderByIdQueryHandler : IRequestHandler<GetOrderByIdQuery, OrderDetailDto?>
{
    private readonly IOrderReadRepository _readRepository;

    public GetOrderByIdQueryHandler(IOrderReadRepository readRepository)
    {
        _readRepository = readRepository;
    }

    public async Task<OrderDetailDto?> Handle(GetOrderByIdQuery request, CancellationToken ct)
    {
        return await _readRepository.GetOrderDetailAsync(request.Id, ct);
    }
}
```

### 4. Read Model (Denormalized Table)

```sql
-- Optimized for reading – created by event handlers or projection
CREATE TABLE OrderReadModel (
    Id UNIQUEIDENTIFIER PRIMARY KEY,
    CustomerName NVARCHAR(200),
    TotalAmount DECIMAL(18,2),
    Status NVARCHAR(50),
    CreatedAt DATETIME2,
    ItemCount INT
    -- No need for separate OrderItems table if you embed JSON or use view
);
```

### 5. Event Handler → Updates Read Model (Eventual Consistency)

```csharp
// Application/Orders/Events/OrderConfirmedEventHandler.cs
public class OrderConfirmedEventHandler : INotificationHandler<OrderConfirmedEvent>
{
    private readonly IOrderReadRepository _readRepo;

    public async Task Handle(OrderConfirmedEvent @event, CancellationToken ct)
    {
        await _readRepo.UpdateOrderStatusAsync(@event.OrderId, "Confirmed", ct);
    }
}
```

### 6. API Controller (Minimal API Style – .NET 8)

```csharp
// API/Program.cs or Controllers/OrdersController.cs
app.MapPost("/orders", async (CreateOrderCommand cmd, IMediator mediator) =>
{
    var orderId = await mediator.Send(cmd);
    return Results.Created($"/orders/{orderId}", orderId);
});

app.MapGet("/orders/{id:guid}", async (Guid id, IMediator mediator) =>
{
    var query = new GetOrderByIdQuery(id);
    var result = await mediator.Send(query);
    return result is not null ? Results.Ok(result) : Results.NotFound();
});
```

## Advanced: CQRS + Event Sourcing

```
Write Side (Command) → Append Events → Event Store
                                   ↓
                           Projectors → Read Models (SQL, Redis, Elastic)
```

Perfect for audit, temporal queries ("what was order status yesterday?"), debugging.

## MediatR Pipeline Behaviors (Logging, Validation, Transactions)

```csharp
public class TransactionBehavior<TRequest, TResponse> 
    : IPipelineBehavior<TRequest, TResponse> where TRequest : IRequest<TResponse>
{
    private readonly IUnitOfWork _uow;
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (request is not ICommand) return await next();

        await _uow.BeginTransactionAsync();
        try
        {
            var response = await next();
            await _uow.CommitTransactionAsync();
            return response;
        }
        catch
        {
            await _uow.RollbackTransactionAsync();
            throw;
        }
    }
}
```

## Summary Table

| Feature                     | Traditional CRUD       | CQRS (Simple)             | CQRS + Event Sourcing     |
|----------------------------|-----------------------|----------------------------|----------------------------|
| Complexity                 | Low                   | Medium                     | High                       |
| Performance                | Medium                | High (read side)           | Very High                  |
| Scalability                | Low                   | High                       | Very High                  |
| Audit / History            | Hard                  | Possible                   | Native                     |
| Learning Curve             | Low                   | Medium                     | Steep                      |
| Best for                   | Simple apps           | Growing apps, reporting    | Finance, audit-heavy apps  |

## Final Recommendation (2025 Best Practices)

| Scenario                                | Recommended Approach                               |
|----------------------------------------|-----------------------------------------------------|
| Simple blog, admin panel               | Just use EF Core + Controllers                      |
| Medium app with reporting              | CQRS with MediatR + separate read models            |
| High-traffic e-commerce                | Full CQRS + separate read DB + caching              |
| Banking, trading, legal systems needing audit | CQRS + Event Sourcing                               |

**Start simple. Add CQRS only when you need it. Never over-engineer from day one.**
