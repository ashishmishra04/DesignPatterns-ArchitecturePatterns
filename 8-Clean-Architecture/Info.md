
# Clean Architecture in C# (.NET 8/9)  

**The Ultimate Blueprint for Maintainable, Testable, and Adaptable Systems**

Invented by Robert C. Martin (Uncle Bob) in 2012  
Used by: Most enterprise .NET teams, Miro, Deliveroo, Uber (parts), Shopify, Bank systems

![alt text](image.png)
![alt text](image-1.png)

## Nice Articles
https://blog.ndepend.com/clean-architecture-for-asp-net-core-solution/
https://medium.com/@rudrakshnanavaty/clean-architecture-7c1b3b4cb181
https://www.geeksforgeeks.org/system-design/complete-guide-to-clean-architecture/
## Definition

> **"An architecture that separates the core business rules from external concerns like UI, databases, and frameworks, making the system easy to test, maintain, and evolve."**

Core Idea: Dependencies flow **inward** to the domain.  
The domain knows NOTHING about outer layers (no EF Core, no ASP.NET, no JSON serializers).

| Ring/Layer             | Contains                              | Depends On          |
|------------------------|---------------------------------------|---------------------|
| **Entities**           | Business entities (pure C#)           | Nothing             |
| **Use Cases**          | Business rules, orchestrators         | Entities            |
| **Interfaces/Ports**   | Adapters' contracts                   | Use Cases           |
| **Adapters**           | UI, DB, External services             | Interfaces/Ports    |

**Similar to**: Hexagonal (Ports & Adapters), Onion Architecture, DDD Layers.  
**Different from**: Classic N-Layer (where domain depends on data access).

## Why Use Clean Architecture?

| Benefit                        | Explanation                                                                 |
|--------------------------------|-----------------------------------------------------------------------------|
| **High Testability**           | Core runs without DB/UI → 100% unit testable                                |
| **Framework Independence**     | Swap ASP.NET for Blazor? No core changes                                    |
| **Database Independence**      | EF Core → Dapper → MongoDB? Just new adapter                                |
| **Business-Focused**           | Core code reads like business requirements                                  |
| **Scalability**                | Easy to add microservices, event sourcing, CQRS                             |
| **Longevity**                  | Systems last 10+ years without rot                                          |

**Rule #1**: Dependencies point inward. Outer layers depend on inner.  
**Rule #2**: Nothing in inner layers knows about outer (e.g., no [HttpGet] in use cases).

## Full Real-World Example: E-Commerce Product Catalog (.NET 8)

### Project Structure (Vertical Slices + Clean)

```
ECommerce/
├── src/
│   ├── ECommerce.Api/                     ← Adapters: Web API
│   ├── ECommerce.WebApp/                  ← Adapters: Blazor/Razor (optional)
│   ├── ECommerce.Application/             ← Use Cases, Interfaces, DTOs
│   ├── ECommerce.Domain/                  ← Pure Entities, Value Objects, Domain Events
│   ├── ECommerce.Infrastructure/         ← Adapters: EF Core, Redis, Email, etc.
│   └── ECommerce.Tests/                   ← Unit + Integration
└── ECommerce.sln
```

### 1. Domain Layer (Pure Business Logic)

```csharp
// Domain/Entities/Product.cs
public class Product
{
    public Guid Id { get; private set; }
    public string Name { get; private set; } = "";
    public Money Price { get; private set; } = Money.Zero;
    public int Stock { get; private set; }

    private Product() { } // For serialization/EF

    public static Product Create(string name, Money price, int stock)
    {
        if (string.IsNullOrEmpty(name)) throw new ArgumentException("Name required");
        if (price.Amount <= 0) throw new ArgumentException("Positive price required");

        return new Product { Id = Guid.NewGuid(), Name = name, Price = price, Stock = stock };
    }

    public void UpdatePrice(Money newPrice)
    {
        if (newPrice.Amount <= 0) throw new ArgumentException("Positive price required");
        Price = newPrice;
    }

    public void ReduceStock(int quantity)
    {
        if (quantity > Stock) throw new InvalidOperationException("Insufficient stock");
        Stock -= quantity;
    }
}

// Domain/ValueObjects/Money.cs
public record Money(decimal Amount, string Currency = "USD")
{
    public static Money Zero => new(0);
    // Operators as before...
}
```

### 2. Application Layer (Use Cases + Interfaces)

```csharp
// Application/Products/Commands/CreateProduct/CreateProductCommand.cs
public record CreateProductCommand(string Name, decimal Price, int Stock) : IRequest<Guid>;

// Use Case Handler
public class CreateProductHandler : IRequestHandler<CreateProductCommand, Guid>
{
    private readonly IProductRepository _repository;

    public CreateProductHandler(IProductRepository repository) => _repository = repository;

    public async Task<Guid> Handle(CreateProductCommand request, CancellationToken ct)
    {
        var product = Product.Create(request.Name, new Money(request.Price), request.Stock);
        await _repository.AddAsync(product);
        return product.Id;
    }
}

// Application/Interfaces/IProductRepository.cs  ← Port
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(Guid id);
    Task AddAsync(Product product);
    Task UpdateAsync(Product product);
}

// Application/DTOs/ProductDto.cs  ← For outer layers
public record ProductDto(Guid Id, string Name, decimal Price, int Stock);
```

### 3. Infrastructure Layer (Adapters)

```csharp
// Infrastructure/Persistence/ProductRepository.cs  ← Adapter
public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _context;

    public ProductRepository(AppDbContext context) => _context = context;

    public async Task<Product?> GetByIdAsync(Guid id)
        => await _context.Products.FindAsync(id);

    public async Task AddAsync(Product product)
    {
        await _context.Products.AddAsync(product);
        await _context.SaveChangesAsync();
    }

    public async Task UpdateAsync(Product product)
    {
        _context.Products.Update(product);
        await _context.SaveChangesAsync();
    }
}

// Infrastructure/Email/EmailSender.cs  ← Another Adapter (implements IEmailSender port)
```

### 4. Presentation Layer (API Adapter)

```csharp
// Api/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(
    typeof(CreateProductCommand).Assembly));

builder.Services.AddScoped<IProductRepository, ProductRepository>();

var app = builder.Build();

app.MapPost("/products", async (CreateProductCommand cmd, IMediator mediator) =>
{
    var id = await mediator.Send(cmd);
    return Results.Created($"/products/{id}", id);
});
```

### 5. Real-World Scenario: Scaling the E-Commerce App

**Initial**: Simple monolith with EF Core + SQL Server.

**Evolution**:
- Add caching? New `CachedProductRepository` decorator (implements same port).
- Switch to CosmosDB? New `CosmosProductRepository` adapter.
- Add Blazor UI? New project, same application layer.
- Go microservices? Split to ProductService, use same domain/application.
- Add CQRS/Event Sourcing? Fits perfectly inside application layer.

**Without Clean Arch**: Domain code littered with [Table], DbSet, HttpContext → massive refactoring.

## When NOT to Use Clean Architecture

- Tiny scripts or prototypes
- Simple CRUD apps with no business rules
- Teams unfamiliar with abstractions (start simple)
- Deadlines where over-engineering hurts

In such cases: Use ASP.NET scaffolding + EF Core directly.

## Summary Table

| Aspect                     | Clean Architecture          | Classic N-Layer             |
|----------------------------|-----------------------------|-----------------------------|
| Dependency Direction       | Inward                      | Downward                    |
| Testability                | Excellent (no mocks needed) | Medium (mock DB/UI)         |
| Flexibility                | High (swap anything)        | Low                         |
| Complexity                 | Medium-High                 | Low-Medium                  |
| Best for                   | Enterprise, evolving apps   | Simple apps                 |

## 2025 Best Practices

- Combine with **Vertical Slices** (feature folders > layers).
- Use **MediatR** for use cases.
- Add **FluentValidation** in application layer.
- For large apps: Add DDD (Aggregates, Domain Events).
- Tools: .NET 9, Rider/Resharper for refactoring.