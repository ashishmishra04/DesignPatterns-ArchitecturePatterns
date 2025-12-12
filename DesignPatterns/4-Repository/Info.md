# Repository Pattern in C#


# Repository Pattern in C# – Complete Guide

## Definition

The **Repository Pattern** is a design pattern that mediates between the domain and data mapping layers using a collection-like interface for accessing domain objects. It abstracts the data access logic and hides the details of how data is stored or retrieved (EF Core, Dapper, ADO.NET, etc.) from the business logic.

> "A Repository mediates between the domain and data mapping layers using a collection-like interface for accessing domain entities."  
> — Martin Fowler, *Patterns of Enterprise Application Architecture*

## Why Do We Need the Repository Pattern?

| Benefit                          | Explanation |
|----------------------------------|-----------|
| **Separation of Concerns**       | Business logic doesn't depend on Entity Framework or any specific ORM |
| **Testability**                  | Easy to mock repositories in unit tests (no real database needed) |
| **Persistence Ignorance**        | Domain entities don't need to know about EF Core or DbContext |
| **Centralized Data Access Logic**| CRUD operations and queries are in one place |
| **Swap Data Providers Easily**   | Can switch from EF Core to Dapper, MongoDB, etc. with minimal changes |
| **Cleaner Business Logic**       | Services/Controllers work with repositories instead of DbContext directly |

> Note: In simple CRUD apps with EF Core, some developers skip the repository pattern and use `DbContext` directly (it's already a Unit of Work + Repository). The pattern shines in **complex, long-lived, or highly testable** applications.

## Sample Code (C# .NET 8)

### 1. Domain Entity
```csharp
// Domain/Entities/Product.cs
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public string Category { get; set; } = string.Empty;
}
```

### 2. Generic Repository Interfaces
```csharp
// Repositories/IRepository.cs
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id);
    Task<IReadOnlyList<T>> ListAllAsync();
    Task<IReadOnlyList<T>> ListAsync(ISpecification<T> spec);
    Task<T> AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(T entity);
    Task<int> CountAsync(ISpecification<T> spec);
}

// Repositories/IProductRepository.cs
public interface IProductRepository : IRepository<Product>
{
    Task<IReadOnlyList<Product>> GetProductsByCategoryAsync(string category);
    Task<Product?> GetProductWithCategoryAsync(int id);
}
```

### 3. Concrete Repository (EF Core)
```csharp
// Repositories/EfRepository.cs
public class EfRepository<T> : IRepository<T> where T : class
{
    protected readonly AppDbContext _context;

    public EfRepository(AppDbContext context)
    {
        _context = context;
    }

    public async Task<T?> GetByIdAsync(int id)
        => await _context.Set<T>().FindAsync(id);

    public async Task<IReadOnlyList<T>> ListAllAsync()
        => await _context.Set<T>().ToListAsync();

    public async Task<T> AddAsync(T entity)
    {
        _context.Set<T>().Add(entity);
        await _context.SaveChangesAsync();
        return entity;
    }

    // ... other methods
}

// Repositories/ProductRepository.cs
public class ProductRepository : EfRepository<Product>, IProductRepository
{
    public ProductRepository(AppDbContext context) : base(context) { }

    public async Task<IReadOnlyList<Product>> GetProductsByCategoryAsync(string category)
    {
        return await _context.Products
            .Where(p => p.Category == category)
            .ToListAsync();
    }

    public async Task<Product?> GetProductWithCategoryAsync(int id)
    {
        return await _context.Products
            .FirstOrDefaultAsync(p => p.Id == id);
    }
}
```

### 4. Register in Program.cs (.NET 8 Minimal API style)
```csharp
// Program.cs
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>));
builder.Services.AddScoped<IProductRepository, ProductRepository>();
```

### 5. Usage in Service / Controller
```csharp
// Services/ProductService.cs
public class ProductService
{
    private readonly IProductRepository _productRepository;

    public ProductService(IProductRepository productRepository)
    {
        _productRepository = productRepository;
    }

    public async Task<IReadOnlyList<Product>> GetExpensiveLaptopsAsync()
    {
        var products = await _productRepository.GetProductsByCategoryAsync("Laptop");
        return products.Where(p => p.Price > 1000).ToList();
    }
}
```

## Real-World Example: E-Commerce Application

```
Project Structure:
├── ECommerce.Web          (Controllers / Minimal APIs)
├── ECommerce.Application  (Services, DTOs, Interfaces)
├── ECommerce.Domain       (Entities, Value Objects)
├── ECommerce.Infrastructure (EF Core DbContext, Repositories, External APIs)
└── ECommerce.Tests        (Unit + Integration Tests)
```

**Scenario**: You start with Entity Framework Core + SQL Server.

Later, business requires:
- Some products to be read from an external microservice (REST/GraphQL)
- Cache frequently accessed products in Redis
- Audit all product price changes

**With Repository Pattern**:
- Create `CachedProductRepository` decorator
- Create `ExternalProductRepository` for external source
- Create `AuditableProductRepository` decorator
- Swap or chain them via DI without changing any service/controller code

**Without Repository Pattern**:
- You'd have `DbContext.Products.Where(...)` scattered everywhere → massive refactoring

## When NOT to Use Repository Pattern

- Very simple CRUD apps
- You are 100% sure you'll always use EF Core
- Rapid prototyping / MVP
- Team is small and prefers simplicity

In such cases, `DbContext` + extension methods may be sufficient.

## Summary

| Aspect                  | With Repository Pattern       | Without (Direct DbContext) |
|------------------------|-------------------------------|----------------------------|
| Testability            | Excellent (easy mocking)      | Hard (need InMemoryDb)     |
| Flexibility            | High                          | Low                        |
| Code Duplication       | Low (centralized queries)     | High                       |
| Learning Curve         | Slightly higher               | Lower                      |
| Best for               | Complex, enterprise apps      | Simple CRUD apps           |

The Repository Pattern is a powerful tool for building **maintainable, testable, and flexible** data access layers in C# applications.
