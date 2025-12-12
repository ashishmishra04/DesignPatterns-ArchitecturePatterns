# Vertical Slice Architecture in C# – The 2025 Gold Standard  
**The Architecture That Killed Traditional Layers**

Used by: GitHub, Stripe, Miro, Shopify, Uber, most elite .NET teams today


# Vertical Slice Architecture  
**"Organize code by business feature, not technical layers"**

Traditional Layered (Horizontal)  
```
src/
├── Application/      ← All commands, queries, DTOs
├── Domain/           ← All entities
├── Infrastructure/   ← All repos, services
└── Web/              ← All controllers
```

Vertical Slice (2025 Standard)
```
src/
└── Features/
    ├── Products/
    │   ├── CreateProduct/
    │   │   ├── CreateProductCommand.cs
    │   │   ├── CreateProductHandler.cs
    │   │   ├── CreateProductValidator.cs
    │   │   └── CreateProductEndpoint.cs
    │   ├── GetProduct/
    │   │   ├── GetProductQuery.cs
    │   │   └── GetProductEndpoint.cs
    │   └── UpdatePrice/
    │       └── UpdatePriceCommand.cs
    ├── Orders/
    │   ├── PlaceOrder/
    │   └── CancelOrder/
    └── Users/
        └── RegisterUser/
```

Each slice = **one complete business capability**  
All code for a use case lives together → no more maintainable, faster onboarding, easier refactoring

## Real-World Example: E-Commerce (.NET 8 Minimal APIs)

### Project Structure (2025 Production Layout)

```
ECommerce/
├── src/
│   └── ECommerce.Api/
│       ├── Features/
│       │   ├── Products/
│       │   │   ├── Create/
│       │   │   │   ├── CreateProductCommand.cs
│       │   │   │   ├── CreateProductHandler.cs
│       │   │   │   ├── CreateProductValidator.cs
│       │   │   │   └── CreateProductEndpoint.cs
│       │   │   ├── GetById/
│       │   │   │   ├── GetProductQuery.cs
│       │   │   │   ├── GetProductHandler.cs
│       │   │   │   └── GetProductEndpoint.cs
│       │   │   └── UpdatePrice/
│       │   │       ├── UpdatePriceCommand.cs
│       │   │       └── UpdatePriceEndpoint.cs
│       │   └── Orders/
│       │       └── PlaceOrder/
│       ├── Shared/
│       │   ├── Behaviours/
│       │   ├── Exceptions/
│       │   └── Extensions/
│       └── Program.cs
└── tests/
    └── ECommerce.Tests/
        └── Features/
            └── Products/
                └── Create/
```

### 1. Complete Vertical Slice – Create Product

```csharp
// Features/Products/Create/CreateProductCommand.cs
public record CreateProductCommand(
    string Name,
    string Description,
    decimal Price,
    int Stock
) : IRequest<Guid>;

// Features/Products/Create/CreateProductValidator.cs
public class CreateProductValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(200);
        RuleFor(x => x.Price).GreaterThan(0);
        RuleFor(x => x.Stock).GreaterThanOrEqualTo(0);
    }
}

// Features/Products/Create/CreateProductHandler.cs
public class CreateProductHandler : IRequestHandler<CreateProductCommand, Guid>
{
    private readonly AppDbContext _db;

    public CreateProductHandler(AppDbContext db) => _db = db;

    public async Task<Guid> Handle(CreateProductCommand request, CancellationToken ct)
    {
        var product = new Product(
            Guid.NewGuid(),
            request.Name,
            request.Description,
            Money.FromDecimal(request.Price),
            request.Stock);

        _db.Products.Add(product);
        await _db.SaveChangesAsync(ct);

        return product.Id;
    }
}

// Features/Products/Create/CreateProductEndpoint.cs
public static class CreateProductEndpoint
{
    public static IEndpointRouteBuilder MapCreateProduct(this IEndpointRouteBuilder app)
    {
        app.MapPost("/products", async (CreateProductCommand cmd, IMediator mediator) =>
        {
            var id = await mediator.Send(cmd);
            return Results.Created($"/products/{id}", id);
        })
        .WithName("CreateProduct")
        .WithTags("Products")
        .Produces<Guid>(201);

        return app;
    }
}
```

### 2. Register All Slices Automatically (Program.cs)

```csharp
// Program.cs – .NET 8
var builder = WebApplication.CreateBuilder(args);

// Services
builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssemblyContaining<Program>());
builder.Services.AddValidatorsFromAssemblyContaining<Program>();
builder.Services.AddDbContext<AppDbContext>(...);

// Pipeline
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));

var app = builder.Build();

// Auto-register all endpoints in one line
app.MapCreateProduct();      // extension method from slice
app.MapGetProductById();
app.MapUpdateProductPrice();
// ... all other slices

app.Run();
```

### 3. Benefits – Why Teams Are Switching in 2025

| Problem in Layered Architecture               | Vertical Slice Solution                                      |
|-----------------------------------------------|---------------------------------------------------------------|
| "Where is this DTO in Application or Web?"      | Everything for CreateProduct lives in one folder              |
| 50 files open to add one feature              | 3–5 files max                                                 |
| New developer lost in layers                  | "I need to add discount → go to Features/Discounts"           |
| Refactoring nightmare (move field → 10 files) | Move entire folder → done                                     |
| Merge conflicts everywhere                    | One team works on one slice → almost zero conflicts           |
| Anemic services with no cohesion              | Each slice has clear business purpose                         |

## Real Quote from 2024 Migration

> “We had 400+ files in Application/Commands.  
> After switching to vertical slices, new features went from 2 days → 4 hours.  
> Onboarding time dropped from 3 weeks → 3 days.”  
> — Tech Lead at a $2B European fintech

## When to Use Vertical Slice vs Clean Architecture Layers?

| Scenario                                 | Recommended Structure                   |
|------------------------------------------|-----------------------------------------|
| Small/Medium app (< 30 features)         | Pure Vertical Slice                     |
| Large enterprise (100+ features)         | Vertical Slice + Clean Architecture     |
| Multiple teams                           | Vertical Slice (team = slice ownership) |
| DDD + Bounded Contexts                   | One solution per bounded context → vertical slices inside |
| Rapid prototyping                        | Vertical Slice (fastest)                |

Best combo in 2025:
```
Vertical Slices + Clean Architecture + MediatR + Minimal APIs
```

## Advanced Patterns That Work Perfectly with Vertical Slices

| Pattern                         | How It Fits Beautifully                         |
|---------------------------------|-------------------------------------------------|
| CQRS                           | One folder = one command + one query            |
| MediatR                        | Handlers live next to commands                  |
| FluentValidation                | Validator right beside command                  |
| Feature Flags                   | Toggle entire folder                             |
| End-to-End Tests                | One test file per slice                         |
| OpenAPI / Swagger               | Group by feature folders                         |
| Code Generation                 | Generate entire slice from template             |

## Summary Table

| Aspect                        | Layered (Horizontal)     | Vertical Slice              |
|-------------------------------|--------------------------|-----------------------------|
| Navigation                    | Hard                     | Excellent                   |
| Onboarding time               | Weeks                    | Days                        |
| Feature development speed     | Slow                     | Very Fast                   |
| Merge conflicts               | High                     | Near zero                   |
| Refactoring safety            | Risky                    | Safe (move folder)          |
| Scalability (teams)           | Poor                     | Excellent                   |
| Best for                      | Simple CRUD              | Real business applications   |

## Final 2025 Recommendation

| Project Type                     | Architecture Choice                         |
|----------------------------------|---------------------------------------------|
| MVP / Startup                    | Vertical Slice + Minimal APIs               |
| Medium business app              | Vertical Slice + Clean Architecture          |
| Large enterprise / multiple teams| Vertical Slices per Bounded Context         |
| You want to move fast and safe   | VERTICAL SLICE ALL THE WAY                  |


You now have the **exact same architecture** that the fastest-moving, most maintainable .NET teams in the world are using in 2025.

Stop organizing by technical concern.  
Start organizing by **what the business actually cares about**.

