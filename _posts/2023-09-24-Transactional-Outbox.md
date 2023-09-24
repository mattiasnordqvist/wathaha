---
layout: post
title: "Transactional Outbox"
date: 2023-09-24 19:52:12 +0200
tags: net7 ef-core sql-server transactional-outbox
---

# Create a transactional outbox with SQL Server and EF Core

Let's create an implementation of https://microservices.io/patterns/data/transactional-outbox.html in SQL Server, .NET and EF Core.

1. Create a table to hold your outbox entites.

```sql
CREATE TABLE [dbo].[TransactionalOutbox](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[CreatedAt] [datetimeoffset](7) NOT NULL,
	[EventTypeName] [varchar](255) NOT NULL,
	[Data] [nvarchar](max) NOT NULL,
PRIMARY KEY CLUSTERED
(
	[Id] ASC
))
```

2. Introduce the `Event` type. An interface could probably do it as well.

```csharp
public abstract record Event;
```

3. Introduce the `IEventSource` interface
   Any entity that can create events, can implement this.

```csharp
public interface IEventSource
{
    IEnumerable<Event> Events { get; }
    void ClearEvents();
}
```

4. Ideally, only AggregateRoots can produce events. You can use a base class like this. Use explicit interface implementation to hide IEventSource-properties from other code that is working with the aggregate.

```csharp
public abstract class AggregateRoot : IEventSource
{
    IEnumerable<Event> IEventSource.Events => _events.ToList();
    void IEventSource.ClearEvents() => _events.Clear();
    private readonly List<Event> _events = new();
    protected void AddEvent(Event @event) => _events.Add(@event);
}
```

5. Introduce outbox-entry. It is ef-cores representation of an item in the TransactionalOutbox. (it maps to the table you created earlier)

```csharp
public class OutboxEntry
{
    public int? Id { get; set; }
    public required DateTimeOffset CreatedAt { get; set; }
    public required string Data { get; set; }
    public required string EventTypeName { get; set; }
}
```

6. Introduce the Transactional Outbox and its dependencies

```csharp

// To help configuration of ef core.
public record TransactionalOutboxEfCoreConfiguration(string Schema, string Table);

// Provides a way to notify the message relay service about new messages in the outbox.
public interface IMessageRelayServiceNotifier
{
    Task Notify();
}

public abstract class TransactionalOutbox
{
    private readonly TransactionalOutboxEfCoreConfiguration _transactionalOutboxEfCoreConfiguration;
    private readonly IMessageRelayServiceNotifier _messageRelayServiceNotifier;
    private readonly ILogger<TransactionalOutbox<TContext>> _logger;
    private readonly JsonSerializerOptions _jsonSerializerOptions;
    private DbContext? _context;
    private bool _hasEventsToNotifyAbout;
    private static bool _modelBuilderConfigured; //  OnModelCreating will only be called once per context type, thats why this is static: https://learn.microsoft.com/en-us/ef/core/modeling/dynamic-model#imodelcachekeyfactory

    public TransactionalOutbox(
        IOptions<TransactionalOutboxEfCoreConfiguration> transactionalOutboxEfCoreConfiguration,
        IMessageRelayServiceNotifier messageRelayServiceNotifier,
        ILogger<TransactionalOutbox> logger,
        JsonSerializerOptions jsonSerializerOptions)
    {
        _transactionalOutboxEfCoreConfiguration = transactionalOutboxEfCoreConfiguration.Value;
        _messageRelayServiceNotifier = messageRelayServiceNotifier;
        _logger = logger;
        _jsonSerializerOptions = jsonSerializerOptions;
    }

    /// <summary>
    /// Notifies the message relay service about new messages in the outbox
    /// </summary>
    private void Notify()
    {
        try
        {
            if (_hasEventsToNotifyAbout)
            {
                _hasEventsToNotifyAbout = false;
                _outstandingEventsNotifier.Notify();
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to notify about outstanding messages");
        }
    }

    /// <summary>
    /// Attaches the outbox to this context. Returns an action to run in OnModelCreating of the context.
    /// </summary>
    public Action<ModelBuilder> Attach(DbContext context)
    {
        if (_context != null)
        {
            throw new Exception("TransactionalOutbox already attached to context");
        }
        _context = context;
        context.SavingChanges += Context_SavingChanges;
        context.SavedChanges += Context_SavedChanges;
        return ConfigureEfCoreModel;
    }

    private void Context_SavingChanges(object? sender, SavingChangesEventArgs e)
    {
        SaveEvents();
    }

    /// <summary>
    /// Saves any entity events from tracked entities implementing IEventSource
    /// to the transactional outbox and clears entities list of events.
    /// Rolls back if SaveChanges fail! (the whole idea of transactional outbox)
    /// </summary>
    private void SaveEvents()
    {
        if (_context == null)
        {
            throw new Exception("TransactionalOutbox not attached to context");
        }
        if (!_modelBuilderConfigured)
        {
            throw new Exception("TransactionalOutbox model not configured");
        }
        var trackedEventSources = _context.ChangeTracker.Entries<IEventSource>();
        var events = trackedEventSources
            .SelectMany(eventsource => eventsource.Entity.Events)
            .Select(@event =>
            new OutboxEntry
            {
                EventTypeName = @event.GetEventName(),
                CreatedAt = SystemTime.Now,
                Data = JsonSerializer.Serialize(@event, @event.GetType()!, _jsonSerializerOptions)
            })
            .ToList();

        _context.Set<OutboxEntry>().AddRange(events);
        foreach (var trackedPart in trackedEventSources)
        {
            trackedPart.Entity.ClearEvents();
        }
        _hasEventsToNotifyAbout = events.Count > 0;
    }

    private void Context_SavedChanges(object? sender, SavedChangesEventArgs e)
    {
        Notify();
    }

    /// <summary>
    /// Configures ef core to ignore IEventSource.Events properties on all entities implementing IEventSource
    /// Configures ef core to use the configured table and schema for the outbox entries
    /// </summary>
    /// <param name="modelBuilder"></param>
    private void ConfigureEfCoreModel(ModelBuilder modelBuilder)
    {
        var entityTypes = modelBuilder.Model.GetEntityTypes()
            .Where(t => typeof(IEventSource).IsAssignableFrom(t.ClrType));
        foreach (var entityType in entityTypes)
        {
            var entityTypeBuilder = modelBuilder.Entity(entityType.ClrType);
            entityTypeBuilder.Ignore(nameof(IEventSource.Events));
        }

        modelBuilder.Entity<OutboxEntry>().ToTable(_transactionalOutboxEfCoreConfiguration.Table, _transactionalOutboxEfCoreConfiguration.Schema);
        _modelBuilderConfigured = true;
    }
}
```

7. Subclass TransactionalOutbox

```csharp
public class MyTransactionalOutbox : TransactionalOutbox
{
    public MyTransactionalOutbox(TransactionalOutboxOptions options, IMessageRelayServiceNotifier messageRelayServiceNotifier, ILogger<TransactionalOutbox> logger) : base(options, messageRelayServiceNotifier, logger) { }
}
```

8. Attach transactionaloutbox to yourt db context

```csharp

public class MyDbContext
{

    private readonly Action<ModelBuilder> _transactionalOutboxModelBuilder;

    public MyDbContext(MyTransactionalOutbox transactionalOutbox, DbContextOptions<MyDbContext> options) : base(options)
    {
        _transactionalOutboxModelBuilder = transactionalOutbox.Attach(this);
    }

    // maybe some code.

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        _transactionalOutboxModelBuilder(modelBuilder);

        // your configuration ...
    }
}
```

9. Introduce event names and some strategy to resolve event names from event types

```csharp
[AttributeUsage(AttributeTargets.Class)]
public class EventNameAttribute : Attribute
{
    public EventNameAttribute(string name)
    {
        Name = name;
    }

    public string Name { get; }
}

public static class EventExtensions
{
    // These methods are used by transactional outbox when serializing events.
    public static string GetEventName(this Event @event) => @event.GetType().GetEventName();
    public static string GetEventName(this Type eventType)
    {
        if (!eventType.IsAssignableTo(typeof(Event)))
        {
            throw new Exception($"Type {eventType.FullName} is not an Event");
        }

        return eventType.GetCustomAttribute<EventNameAttribute>()?.Name ?? (eventType.Name.EndsWith("Event") ? eventType.Name.Replace("Event", "") : throw new Exception("Cant figure out event name"));
    }
}
```

10. Register services

```csharp
public static class ServiceCollectionExtensions
{
    public static void RegisterTransactionalOutbox(this IServiceCollection serviceCollection, Action<TransactionalOutboxOptions> configure)
    {
        serviceCollection.Configure(configure);

        // register the transactionalOutbox
        serviceCollection.AddScoped<MyTransactionalOutbox>();

        // register your message relay service notifier. We don't have any yet, so maybe just create some no-op implementation
        serviceCollection.AddScoped<IMessageRelayServiceNotifier, NoOpMessageRelayServiceNotifier>();

        // If you noticed, transactional outbox needs to know how to serialize messages. For now, I just inject JsonSerializerOptions, so be sure to register that as well. Maybe we should add some other interface here, so that you can serialize however you like.
    }
}
```

Dont forget to call `RegisterTransactionalOutbox()` from your startup.

11. Declare an Event type and create events somewhere in your domain

```csharp

[EventName("ProjectCreated")]
public record ProjectCreatedEvent(CustomerId CustomerId, ProjectId ProjectId) : Event;

public class Project : AggregateRoot // <-- Aggregate root introduced earlier
{
    public static Project Create(CustomerId customerId, string projectDescription)
    {
        var project = new Project()
        {
            Id = ProjectId.Create(),
            CustomerId = customerId,
            ProjectDescription = projectDescription,
        };
        // Call AddEvent inherited from AggregateRoot
        project.AddEvent(new ProjectCreatedEvent(customerId, project.Id));
        return project;
    }
}
```

12. SaveChanges like you always have, and find the message in your transactional outbox.

```csharp
_projectRepository.Add(Project.Create(
    customerId: existingCustomer.Id,
    projectDescription: request.CustomerDiscoveredEvent.Project.ProjectDescription
));
await _unitOfWork.CommitAsync(cancellationToken);
```

# Next up!

Message relay service to publish events from the transactional outbox
