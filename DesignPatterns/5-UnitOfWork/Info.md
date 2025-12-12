# Unit of Work (UoW) Pattern in C# (.NET 8)

## Definition

The **Unit of Work** pattern maintains a list of objects affected by a business transaction and coordinates the writing out of changes (and concurrency resolution) at the end of that transaction.

> “Maintains a list of objects affected by a business transaction and coordinates the writing out of changes and the resolution of concurrency problems.”  
> — Martin Fowler

It works perfectly together with the **Repository Pattern**.

| Pattern           | Responsibility                                      |
|-------------------|-------------------------------------------------------|
| Repository        | Encapsulates data access for a single entity type   |
| Unit of Work      | Coordinates multiple repositories and commits changes atomically |

## Why Do We Need Unit of Work?

| Benefit                             | Explanation                                                                 |
|-------------------------------------|-----------------------------------------------------------------------------|
| Atomic transactions                | All changes succeed or all fail                                             |
| Single point of Save               | Call `SaveChanges()` once per business operation instead of per repository |
| Better performance                 | One database round-trip instead of many                                     |
| Clear transaction boundaries       | Business methods clearly define what belongs to one logical operation      |
| Easier testing                     | Can mock `IUnitOfWork` and verify that `CompleteAsync()` was called once   |

## Real-World Analogy

Think of placing an order in an e-commerce site:

1. Create `Order`
2. Add `OrderItems`
3. Update product stock (`Inventory`)
4. Create `Payment`
5. Send confirmation email

All 5 operations must succeed together.  
This entire flow is **one Unit of Work**.

## Full Example (.NET 8 + EF Core)

### 1. IUnitOfWork Interface
```csharp
// Application/Common/IUnitOfWork.cs
public interface IUnitOfWork : IDisposable
{
    IProductRepository Products { get; }
    IOrderRepository   Orders   { get; }
    // Add other repositories here...

    Task<int> CompleteAsync();        // Calls SaveChangesAsync()
    Task BeginTransactionAsync();
    Task CommitTransactionAsync();
    Task RollbackTransactionAsync();
}
```

### 2. Concrete UnitOfWork with EF Core
```csharp
// Infrastructure/Data/UnitOfWork.cs
public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;
    
    public IProductRepository Products { get; private set; }
    public IOrderRepository   Orders   { get; private set; }

    public UnitOfWork(AppDbContext context)
    {
        _context = context;
        Products = new ProductRepository(_context);
        Orders   = new OrderRepository(_context);
    }

    public async Task<int> CompleteAsync()
        => await _context.SaveChangesAsync();

    public async Task BeginTransactionAsync()
        => await _context.Database.BeginTransactionAsync();

    public async Task CommitTransactionAsync()
        => await _context.Database.CommitTransactionAsync();

    public async Task RollbackTransactionAsync()
        => await _context.Database.RollbackTransactionAsync();

    public void Dispose()
        => _context.Dispose();
}
```

### 3. Register in Program.cs
```csharp
// Program.cs (.NET 8)
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
```

### 4. Real-World Usage – Place Order Service
```csharp
// Application/Orders/Commands/PlaceOrderCommandHandler.cs
public class PlaceOrderCommandHandler
{
    private readonly IUnitOfWork _uow;

    public PlaceOrderCommandHandler(IUnitOfWork uow)
    {
        _uow = uow;
    }

    public async Task<Order> Handle(PlaceOrderCommand command)
    {
        await _uow.BeginTransactionAsync();

        try
        {
            // 1. Create order
            var order = new Order(command.CustomerId);
            await _uow.Orders.AddAsync(order);

            // 2. Reduce stock for each product
            foreach (var item in command.Items)
            {
                var product = await _uow.Products.GetByIdAsync(item.ProductId);
                product?.DecreaseStock(item.Quantity);
            }

            // 3. One single save for everything
            await _uow.CompleteAsync();

            await _uow.CommitTransactionAsync();

            // 4. Fire domain event or send email here (after commit)
            return order;
        }
        catch
        {
            await _uow.RollbackTransactionAsync();
            throw;
        }
    }
}
```

### 5. Generic Repository + Unit of Work (Popular Approach)
Many developers prefer this clean style:

```csharp
public interface IGenericRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id);
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(T entity);
}

public class GenericRepository<T> : IGenericRepository<T> where T : class
{
    protected readonly AppDbContext _context;
    public GenericRepository(AppDbContext context) => _context = context;
    // ... methods that use _context.Set<T>()
}

public interface IUnitOfWork : IDisposable
{
    IGenericRepository<Product> Products { get; }
    IGenericRepository<Order>   Orders   { get; }
    Task<int> SaveChangesAsync();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;
    public IGenericRepository<Product> Products => new GenericRepository<Product>(_context);
    public IGenericRepository<Order>   Orders   => new GenericRepository<Order>(_context);

    public UnitOfWork(AppDbContext context) => _context = context;

    public async Task<int> SaveChangesAsync() => await _context.SaveChangesAsync();
    public void Dispose() => _context.Dispose();
}
```

## EF Core DbContext Is Already a Unit of Work!

Yes! `DbContext` already implements:
- Change tracking
- Identity map
- Unit of Work (via `SaveChanges()`)

So why create another UoW?

| Reason                                | Explanation                                                                 |
|---------------------------------------|-----------------------------------------------------------------------------|
| Abstraction                           | Hide EF Core from upper layers (clean architecture)                         |
| Multiple DbContexts                   | Coordinate saves across several contexts in one transaction                  |
| Testability                           | Easy to mock `IUnitOfWork` without EF dependencies                          |
| Explicit transaction boundaries       | Makes intent clear in business code |

## When NOT to Implement Unit of Work

- Simple CRUD apps
- Only one `DbContext`
- You are okay coupling your domain/services to EF Core
- Rapid prototyping

In such cases, just inject `AppDbContext` directly.

## Summary Table

| Feature                         | Repository Only          | Repository + Unit of Work             | Direct DbContext       |
|-------------------------------|--------------------------|---------------------------------------|------------------------|
| Separation of concerns        | Good                     | Excellent                             | Low                    |
| Testability                   | Good                     | Excellent                             | Needs InMemory DB      |
| Transaction control           | Manual per repo          | Centralized                           | Direct                 |
| Performance                   | Multiple SaveChanges     | One SaveChanges per business op       | One SaveChanges        |
| Complexity                    | Medium                   | Medium-High                           | Low                    |
| Best for                      | Medium apps              | Complex enterprise / clean architecture | Simple apps            |

## Final Recommendation (2025 Best Practice)

```csharp
// Most clean architecture projects in 2025 use:
public interface IUnitOfWork
{
    Task<int> CommitAsync(CancellationToken ct = default);
}

// And implement it with DbContext directly:
public class ApplicationDbContext : DbContext, IUnitOfWork { ... }
```

This gives you the abstraction benefits with almost zero boilerplate.

Happy coding!
```

Save this as `UnitOfWorkPattern.md` — perfect companion to your previous Repository Pattern note!
```