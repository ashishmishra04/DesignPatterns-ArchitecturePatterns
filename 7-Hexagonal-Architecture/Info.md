
# Hexagonal Architecture  

**Also known as: Ports and Adapters**  
**The Cleanest, Most Testable, and Future-Proof Pattern in Modern .NET**

Made famous by Alistair Cockburn in 2005  
Used by: Netflix, Uber, Amazon (parts), Miro, Deliveroo, most serious DDD teams

## Core Idea in One Sentence

> **"Keep your business logic completely independent of frameworks, databases, UI, and external services."**

Your domain code should run perfectly with:
- No Entity Framework
- No ASP.NET Core
- No RabbitMQ
- No SQL Server

Just pure C# classes and unit tests.

## Visual Diagram

```
                     ┌────────────────────┐
               ┌────►│   Web API (MVC,    │◄────┐
               │     │  Minimal APIs)     │     │
               │     └────────────────────┘     │
               │                                │
               │     ┌────────────────────┐     │   DRIVING ADAPTERS
               └────►│   GraphQL / gRPC   │◄────┘   (Left side)
                     └────────────────────┘
                             ▲
                             │
                   ┌─────────┴─────────┐
                   │     APPLICATION   │
                   │      (Use Cases)  │
                   └─────────┬─────────┘
                             │
             ┌───────────────┼───────────────┐
             │               │               │
   ┌─────────▼─────┐ ┌───────▼───────┐ ┌─────▼─────────┐
   │   PORTS       │ │   DOMAIN      │ │   PORTS       │
   │ (Interfaces)  │ │   (Core Logic)│ │ (Interfaces)  │
   └───────┬─────────┘ └───────┬───────┘ └─────┬─────────┘
         │                   │               │
         │                   │               │
   ┌─────▼─────┐       ┌─────▼─────┐   ┌─────▼─────┐
   │ SQL Repo  │       │ In-Memory │   │ Message   │
   │ (EF Core) ◄──────►│ Repository◄──►│ Publisher │
   └───────────┘       └───────────┘   └───────────┘
         ▲                   ▲               ▲
         │                   │               │
   DRIVEN ADAPTERS      TEST DOUBLE     EXTERNAL SYSTEMS
      (Right side)
```

## Why Hexagonal > Layered (N-Layer) Architecture?

| Problem in Classic Layers          | Hexagonal Solution                                      |
|------------------------------------|----------------------------------------------------------|
| Domain depends on EF Core          | Domain knows nothing about persistence                   |
| Hard to test business logic        | 100% unit testable without mocks or DB                   |
| Can't swap database easily         | Just plug in new adapter (PostgreSQL → MongoDB)          |
| UI changes break everything        | Web API, CLI, Message Consumer use same core             |
| "God Services" with mixed concerns | Clear separation: Use Cases own the flow                 |

## Full Real-World Example: Online Banking System (.NET 8)

### Final Project Structure (2025 Standard)

```
Banking/
├── src/
│   ├── Banking.Api/                     ← ASP.NET Core Web API (Adapter)
│   ├── Banking.Cli/                      ← Console tool (Adapter)
│   ├── Banking.Application/             ← Use Cases, DTOs, Ports (Application)
│   ├── Banking.Domain/                  ← Pure domain: Entities, Value Objects, Domain Events
│   └── Banking.Infrastructure/         ← All adapters: EF Core, SMTP, RabbitMQ, etc.
└── tests/
    ├── Banking.UnitTests/
    └── Banking.IntegrationTests/
```

### 1. Domain – 100% Pure C# (No dependencies!)

```csharp
// Banking.Domain/Entities/Account.cs
public class Account : AggregateRoot
{
    public Guid Id { get; private set; }
    public string OwnerName { get; private set; } = "";
    public Money Balance { get; private set; } = Money.Zero;

    public void Deposit(Money amount)
    {
        if (amount <= 0) throw new InvalidAmountException();
        Balance += amount;
        RaiseDomainEvent(new MoneyDeposited(Id, amount));
    }

    public void Withdraw(Money amount)
    {
        if (amount <= 0) throw new InvalidAmountException();
        if (Balance < amount) throw new InsufficientFundsException();

        Balance -= amount;
        RaiseDomainEvent(new MoneyWithdrawn(Id, amount));
    }
}

// Banking.Domain/ValueObjects/Money.cs
public record Money(decimal Amount, string Currency = "USD")
{
    public static Money Zero => new(0);
    public static Money operator +(Money a, Money b) => 
        new(a.Amount + b.Amount, a.Currency);
    public static Money operator -(Money a, Money b) => 
        new(a.Amount - b.Amount, a.Currency);
    public static bool operator <(Money a, Money b) => a.Amount < b.Amount;
    // ... more operators
}
```

### 2. Ports – Interfaces Owned by the Core

```csharp
// Banking.Application/Ports/IAccountRepository.cs  ← PRIMARY PORT
public interface IAccountRepository
{
    Task<Account?> GetByIdAsync(Guid id, CancellationToken ct);
    Task AddAsync(Account account, CancellationToken ct);
    Task UpdateAsync(Account account, CancellationToken ct);
}

// Banking.Application/Ports/IEmailService.cs  ← PRIMARY PORT (driven)
public interface IEmailService
{
    Task SendAsync(string to, string subject, string body);
}

// Banking.Application/Ports/IAccountNotification.cs  ← DRIVING PORT (for external systems)
public interface IAccountNotification
{
    Task NotifyAccountCreated(Guid accountId, string ownerName);
}
```

### 3. Application Layer – Use Cases (Orchestration)

```csharp
// Banking.Application/Accounts/Commands/CreateAccount/CreateAccountCommand.cs
public record CreateAccountCommand(string OwnerName, decimal InitialDeposit) 
    : IRequest<Guid>;

// Handler = Use Case
public class CreateAccountHandler : IRequestHandler<CreateAccountCommand, Guid>
{
    private readonly IAccountRepository _repository;
    private readonly IEmailService _emailService;

    public CreateAccountHandler(IAccountRepository repository, IEmailService emailService)
    {
        _repository = repository;
        _emailService = emailService;
    }

    public async Task<Guid> Handle(CreateAccountCommand request, CancellationToken ct)
    {
        var account = new Account();
        account.Deposit(Money.FromDecimal(request.InitialDeposit));

        await _repository.AddAsync(account, ct);
        await _emailService.SendAsync(
            "customer@example.com", 
            "Welcome!", 
            $"Account created with balance {account.Balance}");

        return account.Id;
    }
}
```

### 4. Infrastructure – Adapters (The Only Place with External Dependencies)

```csharp
// Banking.Infrastructure/Persistence/EfAccountRepository.cs  ← ADAPTER
public class EfAccountRepository : IAccountRepository
{
    private readonly BankingDbContext _context;

    public EfAccountRepository(BankingDbContext context) => _context = context;

    public async Task<Account?> GetByIdAsync(Guid id, CancellationToken ct)
        => await _context.Accounts.FindAsync(id);

    public async Task AddAsync(Account account, CancellationToken ct)
    {
        await _context.Accounts.AddAsync(account, ct);
        await _context.SaveChangesAsync(ct);
    }
    // Update not needed – EF tracks changes
}

// Banking.Infrastructure/Email/SmtpEmailService.cs  ← ADAPTER
public class SmtpEmailService : IEmailService
{
    public Task SendAsync(string to, string subject, string body)
    {
        // Use MailKit or SendGrid
        return Task.CompletedTask;
    }
}
```

### 5. API – Driving Adapter (Web)

```csharp
// Banking.Api/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(
    typeof(CreateAccountCommand).Assembly));

builder.Services.AddDbContext<BankingDbContext>(...);
builder.Services.AddScoped<IAccountRepository, EfAccountRepository>();
builder.Services.AddScoped<IEmailService, SmtpEmailService>();

var app = builder.Build();

app.MapPost("/accounts", async (CreateAccountCommand cmd, IMediator m) =>
{
    var id = await m.Send(cmd);
    return Results.Created($"/accounts/{id}", id);
});
```

### 6. CLI Tool – Another Driving Adapter (Same Core!)

```csharp
// Banking.Cli/Program.cs
// Same DI container → same core logic!
services.AddScoped<IAccountRepository, InMemoryAccountRepository>(); // for demo
services.AddScoped<IEmailService, ConsoleEmailService>();

var accountId = await mediator.Send(new CreateAccountCommand("John", 1000));
Console.WriteLine($"Created account: {accountId}");
```

## Testing – The Real Power

```csharp
// Banking.UnitTests/Accounts/CreateAccountTests.cs
[Fact]
public async Task Should_create_account_and_send_email()
{
    // Arrange
    var repo = new InMemoryAccountRepository();
    var emailService = new FakeEmailService();
    var handler = new CreateAccountHandler(repo, emailService);

    // Act
    var accountId = await handler.Handle(
        new CreateAccountCommand("Alice", 500), default);

    // Assert
    var account = await repo.GetByIdAsync(accountId, default);
    account.Should().NotBeNull();
    account!.Balance.Amount.Should().Be(500);
    emailService.SentEmails.Should().Contain(e => e.Contains("Welcome"));
}
```

Zero mocks. Zero database. Lightning fast.

## Summary Table

| Concern                    | Hexagonal (Ports & Adapters)       | Classic N-Layer              |
|---------------------------|-------------------------------------|-------------------------------|
| Domain depends on EF?     | Never                               | Usually yes                   |
| Can run core without DB?  | Yes                                 | No                            |
| Can have multiple UIs?    | Yes (Web, CLI, Desktop, Tests)      | Hard                          |
| Testability               | 10/10                               | 4/10                          |
| Swap database             | Plug new adapter                    | Rewrite repositories          |
| Team independence        | High (core team owns domain)        | Low                           |
| Learning curve            | Medium                              | Low                           |
| Best for                  | Complex, long-lived systems         | Simple CRUD, prototypes       |

## 2025 Best Practices Cheat Sheet

| Layer                  | Recommended Tech (2025)                         |
|-----------------------|--------------------------------------------------|
| API                   | Minimal APIs + Carter or FastEndpoints           |
| CQRS                  | MediatR + Vertical Slice Architecture            |
| Persistence           | EF Core, Dapper, Marten, EventStoreDB            |
| Messaging             | MassTransit or Rebus                             |
| Testing               | xUnit + FluentAssertions + Test doubles in memory|
| Structure             | Vertical Slices (Feature folders) > Layers       |

## Final Verdict

Use Hexagonal Architecture when:
- Your system will live >2 years
- You have complex business rules
- You want bulletproof unit tests
- You might need CLI tools, message consumers, or multiple frontends
- You value clean, maintainable code over "getting it done fast"

Do NOT use it for:
- Simple admin panels
- Hackathon projects
- MVP with 1-week deadline
