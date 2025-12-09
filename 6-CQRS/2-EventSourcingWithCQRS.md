# Event Sourcing + CQRS  

**The Ultimate Combination for Auditable, Scalable & Resilient Systems**

Real-world production-ready example in .NET 8

## What is Event Sourcing?

> **"Store the state of your application as an immutable sequence of events, not as the current state."**

Instead of this (traditional):
```sql
UPDATE Orders SET Status = 'Shipped', Total = 1250.00 WHERE Id = 123
```

You do this (Event Sourcing):
```json
[
  { "event": "OrderCreated",       "customer": "John", "total": 1200 },
  { "event": "DiscountApplied",   "discount": 50 },
  { "event": "OrderConfirmed" },
  { "event": "OrderShipped",      "tracking": "1Z999AA101" }
]
```

The **current state** is always derived by replaying events.

## Why Combine Event Sourcing with CQRS?

| Benefit                         | CQRS Alone                 | CQRS + Event Sourcing                                 |
|---------------------------|----------------------------|--------------------------------------------------------|
| Full audit trail          | No                         | Yes – every change is an event                         |
| Temporal queries         | Impossible                 | "What did this order look like on Dec 1st?"            |
| Debugging / Replay        | Hard                       | Replay events → reproduce bugs exactly                 |
| Multiple read models      | Yes                        | Natural – project events into many shapes              |
| Event-driven architecture | Possible                   | Native – events are first-class citizens               |
| Eventual consistency      | Manual                     | Built-in                                               |

Used by: Git, Banking Systems, Booking.com, Shopify (parts), Uber (trip history), EventStoreDB users

## Full Real-World Example: Online Book Store (.NET 8)

### Project Structure (Clean + Vertical Slice)

```
BookStore/
├── src/
│   ├── API/
│   ├── Application/               ← Commands, Queries, Events
│   ├── Domain/                     ← Aggregates, Domain Events
│   ├── Infrastructure/
│   │   ├── Persistence/            ← EventStoreDB / PostgreSQL
│   │   └── ReadModels/             ← Projections (PostgreSQL, Redis, Elastic)
│   └── Tests/
```

### 1. Domain Events (Immutable Facts)

```csharp
// Domain/Events/BookEvents.cs
public interface IDomainEvent { }

public record BookAddedToInventory(
    Guid BookId,
    string Title,
    string Author,
    decimal Price,
    int Quantity) : IDomainEvent;

public record BookReserved(
    Guid BookId,
    Guid ReservationId,
    Guid CustomerId,
    int Quantity) : IDomainEvent;

public record BookReservationConfirmed(Guid ReservationId) : IDomainEvent;

public record BookReservationCancelled(
    Guid ReservationId,
    string Reason) : IDomainEvent;

public record BookRemovedFromInventory(Guid BookId, int Quantity) : IDomainEvent;
```

### 2. Aggregate Root – Reconstituted from Events

```csharp
// Domain/Aggregates/BookInventory.cs
public class BookInventory : AggregateRoot
{
    public Guid BookId { get; private set; }
    public string Title { get; private set; } = "";
    public int AvailableStock { get; private set; }

    // Apply methods – MUST be parameterless and modify state
    private void Apply(BookAddedToInventory e)
    {
        BookId = e.BookId;
        Title = e.Title;
        AvailableStock += e.Quantity;
    }

    private void Apply(BookRemovedFromInventory e)
    {
        AvailableStock -= e.Quantity;
    }

    // Command methods
    public void AddStock(string title, string author, decimal price, int quantity)
    {
        if (quantity <= 0) throw new InvalidOperationException();

        RaiseDomainEvent(new BookAddedToInventory(
            BookId == Guid.Empty ? Guid.NewGuid() : BookId,
            title, author, price, quantity));
    }

    public void ReserveStock(Guid reservationId, Guid customerId, int quantity)
    {
        if (AvailableStock < quantity)
            throw new InvalidOperationException("Not enough stock");

        RaiseDomainEvent(new BookReserved(BookId, reservationId, customerId, quantity));
        RaiseDomainEvent(new BookRemovedFromInventory(BookId, quantity));
    }

    // Called by infrastructure after loading events
    public void LoadFromHistory(IEnumerable<IDomainEvent> history)
    {
        foreach (var e in history)
        {
            When(e);
        }
    }

    private void When(IDomainEvent @event)
    {
        switch (@event)
        {
            case BookAddedToInventory e:        Apply(e); break;
            case BookRemovedFromInventory e:    Apply(e); break;
            case BookReserved e:                Apply(e); break;
            // ... more
        }
    }
}
```

### 3. Event Store Repository (EventStoreDB or Marten or Custom)

```csharp
// Infrastructure/Persistence/IEventStore.cs
public interface IEventStore
{
    Task AppendEventsAsync(Guid aggregateId, IEnumerable<IDomainEvent> events, int expectedVersion);
    Task<IReadOnlyList<IDomainEvent>> GetEventsAsync(Guid aggregateId);
}

// Implementation with EventStoreDB (recommended)
public class EventStoreDBRepository<T> : IRepository<T> where T : AggregateRoot
{
    private readonly IEventStoreDBConnection _connection;

    public async Task<T> GetByIdAsync(Guid id)
    {
        var events = await _connection.ReadStreamForwardAsync($"book-{id}");
        var aggregate = Activator.CreateInstance<T>();
        var domainEvents = events.Select(e => e.Data.Deserialize<IDomainEvent>()).ToList();
        
        aggregate.LoadFromHistory(domainEvents);
        return aggregate;
    }

    public async Task SaveAsync(T aggregate)
    {
        var newEvents = aggregate.GetUncommittedEvents();
        var expectedVersion = aggregate.Version - newEvents.Count;

        ;

        await _connection.AppendToStreamAsync(
            streamName: $"book-{aggregate.Id}",
            expectedVersion: expectedVersion,
            eventData: newEvents.Select(e => new EventData(
                Uuid.NewUuid(),
                e.GetType().Name,
                JsonSerializer.Serialize(e)
            ))
        );

        aggregate.ClearUncommittedEvents();
    }
}
```

### 4. Command Handler with CQRS + Event Sourcing

```csharp
// Application/Books/Commands/ReserveBook/ReserveBookCommand.cs
public record ReserveBookCommand(
    Guid BookId,
    Guid ReservationId,
    Guid CustomerId,
    int Quantity) : IRequest<Guid>;

public class ReserveBookCommandHandler : IRequestHandler<ReserveBookCommand, Guid>
{
    private readonly IRepository<BookInventory> _repository;

    public async Task<Guid> Handle(ReserveBookCommand command, CancellationToken ct)
    {
        var book = await _repository.GetByIdAsync(command.BookId);

        book.ReserveStock(command.ReservationId, command.CustomerId, command.Quantity);

        await _repository.SaveAsync(book);

        return command.ReservationId;
    }
}
```

### 5. Projections – Building Read Models (The Magic!)

```csharp
// Infrastructure/Projections/BookInventoryProjector.cs
public class BookInventoryProjector : IProjector
{
    private readonly IDbConnection _db;

    public async Task Project(object @event)
    {
        switch (@event)
        {
            case BookAddedToInventory e:
                await _db.ExecuteAsync(
                    @"INSERT INTO ReadModel_BookInventory (BookId, Title, Author, Price, AvailableStock)
                      VALUES (@BookId, @Title, @Author, @Price, @Quantity)
                      ON CONFLICT (BookId) DO UPDATE SET AvailableStock = AvailableStock + @Quantity",
                    e);
                break;

            case BookRemovedFromInventory e:
                await _db.ExecuteAsync(
                    "UPDATE ReadModel_BookInventory SET AvailableStock = AvailableStock - @Quantity WHERE BookId = @BookId",
                    new { e.BookId, e.Quantity });
                break;
        }
    }
}
```

### 6. Query Side – Super Fast Reads

```csharp
// Application/Books/Queries/GetBookAvailability.cs
public record GetBookAvailabilityQuery(Guid BookId) : IRequest<BookAvailabilityDto?>;

public record BookAvailabilityDto(Guid BookId, string Title, decimal Price, int AvailableStock, bool CanReserve);

public class GetBookAvailabilityHandler : IRequestHandler<GetBookAvailabilityQuery, BookAvailabilityDto?>
{
    private readonly IDbConnection _db;

    public async Task<BookAvailabilityDto?> Handle(GetBookAvailabilityQuery request, CancellationToken ct)
    {
        return await _db.QueryFirstOrDefaultAsync<BookAvailabilityDto>(
            @"SELECT BookId, Title, Price, AvailableStock,
                     CASE WHEN AvailableStock > 0 THEN true ELSE false END as CanReserve
              FROM ReadModel_BookInventory 
              WHERE BookId = @BookId", new { request.BookId });
    }
}
```

### 7. Run Projection on Startup (Catch-up + Live)

```csharp
// Program.cs
var subscription = eventStoreConnection.SubscribeToAllAsync(
    fromPosition: Position.End, // or last checkpoint
    eventAppeared: async (sub, resolvedEvent) =>
    {
        var domainEvent = resolvedEvent.Event.Data.Deserialize<IDomainEvent>();
        await projector.Project(domainEvent);
    });
```

## Snapshots (Performance Optimization)

```csharp
public class BookInventoryMemento
{
    public Guid BookId { get; set; }
    public string Title { get; set; }
    public int AvailableStock { get; set; }
    public int Version { get; set; }
}

// Save snapshot every 50 events
if (aggregate.Version % 50 == 0)
    await snapshotStore.SaveSnapshotAsync(aggregate.Id, aggregate.CreateMemento());
```

## Benefits You Get for Free

| Feature                     | How You Get It Automatically                     |
|----------------------------|---------------------------------------------------|
| 100% audit trail           | Every event stored forever                        |
| Time travel queries        | Replay events up to any date                      |
| Debug production bugs      | Replay exact event stream                         |
| Multiple UIs               | Project same events into different shapes         |
| Analytics / BI             | Events → Data warehouse                           |
| Compensating actions       | Add "ReservationCancelled" instead of DELETE      |

## Tools & Libraries (2025 Recommendations)

| Purpose                    | Recommended Tool (2025)               |
|---------------------------|----------------------------------------|
| Event Store               | EventStoreDB (best) or Marten (PostgreSQL) |
| Message Bus               | MassTransit or Brighter                |
| CQRS Framework            | MediatR + Pipelines                     |
| Projections               | Marten, EventStoreDB Projections, or custom |
| Outbox / Transactional    | Marten or EF Core + Outbox Pattern      |
| Testing                   | Write event assertions (`ExpectEvent<T>()`)     |

## When NOT to Use Event Sourcing

| Scenario                          | Better Alternative                     |
|----------------------------------|-----------------------------------------|
| Simple blog, todo app            | Just use EF Core + CRUD                 |
| Team new to DDD/CQRS             | Start with simple CQRS first            |
| Need immediate consistency       | Traditional DB with transactions        |
| Regulatory forbids event storage | Use traditional audit tables            |

## Summary Table

| Feature                        | Traditional DB      | CQRS Only           | CQRS + Event Sourcing         |
|-------------------------------|---------------------|---------------------|-------------------------------|
| Complexity                    | Low                 | Medium              | High                          |
| Auditability                  | Limited             | Medium              | Perfect                       |
| Scalability                   | Medium              | High                | Very High                     |
| Debugging                     | Hard                | Medium              | Perfect                       |
| Learning Curve                | Low                 | Medium              | Steep                         |
| Best for                      | Simple apps         | Reporting apps      | Finance, Booking, Audit-heavy |

**Final Verdict (2025)**

Use **Event Sourcing + CQRS** when you need:
- Perfect audit trail
- Temporal queries
- Multiple read models
- Event-driven microservices
- Ability to replay/rebuild state

Otherwise: Start with **simple CQRS**, add Event Sourcing later when you feel the pain.