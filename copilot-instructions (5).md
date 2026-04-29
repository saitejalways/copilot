# GitHub Copilot Instructions - Anti Money Laundering

## Project Structure

This project contains the Anti Money Laundering (AML) domain logic and API for the TPR system.

### Domain Organization

- **Domain Handlers**: Located in `Aca.AntiMoneyLaundering/Domain/{Feature}/`
- **Entities**: Database entities across multiple contexts (KycDomainContext, RiskDomainContext, RegistryEntityContext)
- **Facades**: DTOs/View models for external communication
- **API Controllers**: Located in `Aca.AntiMoneyLaundering.Api/Controllers/`

## Core Domain Concepts

Understanding these key concepts is essential for developing features in this domain:

### Investors

- **Type**: Individual or Business entity
- **Storage**: KYC Domain (PostgreSQL)
- **Lifecycle**:
  - Onboarded into the system
  - Undergo screenings (one-time and/or continuous)
  - Risk ratings are calculated and may be overridden by analysts
  - Can be linked to investment funds and closing periods
- **Compliance Significance**: Every investor must have a complete audit trail and compliance status

### Screenings

- **Two Types**:
  - **One-Time Screenings**: Single evaluation triggered on onboarding or manually
  - **Continuous Screenings**: Recurring evaluations scheduled to catch status changes
- **Storage**: Risk Domain (MongoDB)
- **Results**: Generate matches and non-matches from various screening engines (adverse media, PEP/sanctions, etc.)
- **AI Treatment**: Results are suggestions only - analysts must review and accept/reject
- **Workflow**: Screening → Results → Matches/Issues → Alerts to Analysts → Analyst Decision

### Screening Results & Items of Interest (IoIs)

- **AI Matches**: Indicate potential compliance issues requiring investigation
  - Adverse media matches
  - PEP/Sanctions matches
  - Other regulatory red flags
- **AI Non-Matches**: Negative findings that clear the investor for the screening criteria
- **IoIs**: Alert system that notifies analysts when matches are found
- **Real-Time Notifications**: Use SignalR to push alerts to analysts immediately
- **Analyst Decisions**: Accept risk, escalate, or request more information

### Key Data Contexts

**KYC Domain (PostgreSQL)**:

- Core compliance and investor data
- Legal entities, individuals, contacts
- Risk questionnaires and assessments
- Documents and document templates
- Investment funds and closing periods

**Risk Domain (MongoDB)**:

- Screening records and results
- AI analysis outputs
- Adverse media, sanctions, PEP data
- AI-assisted evaluation records
- Historical screening data

**Registry Domain (MongoDB)**:

- Third-party legal entities
- Individual customer records
- Migration tracking for legacy data

### Critical Data Flow

```
Investor Onboarded → Screening Initiated → AI Engines Run Screenings →
Results Generated (Matches/Non-Matches) → IoIs Created for Matches →
Analyst Notified (SignalR) → Analyst Reviews → Decision Made →
Event Published → Compliance Status Updated
```

### Event Publishing & Audit Trail

- All write operations must publish events via Hangfire background jobs
- Events are logged in EventMessageLog table
- Enables audit trail for regulatory compliance
- Triggers downstream processing and notifications

## Event Publishing and Listening

### Publishing Events

When publishing events, follow these steps:

#### 1. Define Event Class

Events must inherit from `EventMessage<TData>`:

```csharp
using Aca.AntiMoneyLaundering.Domain;

public class ScreeningSummarizedResultEvent : EventMessage<ScreeningSummarizedResultFacade>
{
	public const string DomainName = "screening-summarized-result";

	[SetsRequiredMembers]
	public ScreeningSummarizedResultEvent(ScreeningSummarizedResultFacade data, Guid? clientId, Guid? userId)
	{
		Domain = DomainName;
		Action = EventContext.ActionInsert;
		Data = data;
		ClientId = clientId;
		UserId = userId;
	}
}
```

#### 2. Publish Events Using `PublishEvent` Method

Inject `IAmlEventPublisher` into your handler and use the `PublishEvent` method:

```csharp
public class ScreeningSummarizedResultWriteHandler(
	KycDomainWriteContext writeContext,
	IAmlEventPublisher eventPublisher,
	ILogger<ScreeningSummarizedResultWriteHandler> logger) : KycDomainWriteHandler(writeContext, logger)
{
	public async Task CreateAsync(ScreeningSummarizedResultFacade facade)
	{
		// Save to database
		var entity = facade.ToEntity();
		EntityContext.ScreeningSummarizedResult.Add(entity);
		await EntityContext.SaveChangesAsync(CancellationToken);

		// Publish event using PublishEvent method
		eventPublisher.PublishEvent(new ScreeningSummarizedResultEvent(
			facade,
			Principal.ClientId,
			Principal.UserGuid));

		logger.LogInformation(
			"Screening summarized result created and event published. Id: {Id}",
			entity.Id);
	}
}
```

**Important**: From application code, always use `IAmlEventPublisher.PublishEvent(...)`. Avoid calling `AmlEventPublisher.TrackAndPublishAsync(...)` directly; it is used as the Hangfire job entry point invoked by `PublishEvent`.

### Listening to Events

When listening to events from other services/modules:

#### 1. Create an Event Listener

Create a listener class that inherits from `IListener`:

```csharp
using Aca.AntiMoneyLaundering.Domain.ScreeningSummarizedResults;
using Aca.Framework.Services.Events;

public class ScreeningSummarizedResultsListener(
	EntityAttributesWriteHandler entityAttributesWriteHandler,
	ILogger<ScreeningSummarizedResultsListener> logger) : IListener
{
	public Task ProcessMessageAsync(EventReceivedDetails eventMessage)
	{
		switch (eventMessage.Domain)
		{
			case ScreeningSummarizedResultEvent.DomainName
				when eventMessage.Action is EventContext.ActionInsert or EventContext.ActionUpdate:
				var screeningResult = eventMessage.ParseMessageData<ScreeningSummarizedResultFacade>();

				if (screeningResult is null)
				{
					logger.LogError("Could not parse ScreeningSummarizedResult from event message data.");
					return Task.CompletedTask;
				}

				logger.LogInformation(
					"Processing screening summarized result event for entity: {EntityId}",
					screeningResult.EntityId);

				return HandleScreeningSummarizedResultUpsertAsync(screeningResult);

			default:
				logger.LogInformation(
					"Ignoring event for domain: {Domain}, action: {Action}",
					eventMessage.Domain,
					eventMessage.Action);
				return Task.CompletedTask;
		}
	}

	private Task HandleScreeningSummarizedResultUpsertAsync(ScreeningSummarizedResultFacade screeningResult) =>
		entityAttributesWriteHandler.UpdateEntityAttributesFromScreeningResultAsync(screeningResult);
}
```

See [ScreeningSummarizedResultsListener.cs](../Aca.AntiMoneyLaundering/Listeners/ScreeningSummarizedResultsListener.cs) for a complete example.

#### 2. Register Listener in EventListenerService

Add the listener to the `EventListenerService` hosted service in `Aca.AntiMoneyLaundering.Api/HostedServices/EventListenerService.cs`:

1. Add your listener handler method to the `ProcessMessageAsync` array:
   ```csharp
   var tasks = new[] {
       HandleWorkConnectEventsAsync(context, eventMessage),
       HandleScreeningSummarizedResultsEventsAsync(context, eventMessage), // Add your handler
       // ... other handlers
   };
   ```

2. Create the handler method:
   ```csharp
   private static async Task<bool> HandleScreeningSummarizedResultsEventsAsync(
       ProcessingContext context,
       EventReceivedDetails eventMessage)
   {
       var listener = context.Services.GetRequiredService<ScreeningSummarizedResultsListener>();
       await listener.ProcessMessageAsync(eventMessage);
       return true;
   }
   ```

#### 3. Update Infrastructure Event Subscription Filter

**CRITICAL**: When listening to a new module or domain, the infrastructure event subscription filter must be updated.

See [infrastructure/.github/copilot-instructions.md](../../infrastructure/.github/copilot-instructions.md) for instructions on updating `infrastructure/bin/build.ts` with the new subscription filter.

#### 4. Infrastructure Deployment Workflow

After creating a listener and updating the infrastructure subscription filter, follow this deployment workflow:

1. **Include infrastructure changes in your PR**: Ensure `infrastructure/bin/build.ts` has been updated with the subscription filter
2. **After PR is merged** (NOT before):
   - Run the infrastructure deploy GitHub workflow against the branch your PR was merged into
3. **When the Release Monitor ticket is created in Jira**:
   - Add a note that **an infrastructure deploy is required** for this release
4. **After deployment**:
   - Verify the subscription filter policy is updated in AWS EventBridge console

### Event Best Practices

- **Domain constant**: Define a `DomainName` constant in your event class for consistency
- **Event structure**: Always include required metadata (ClientId, UserId) for audit trail
- **Error handling**: Always check if `ParseMessageData<T>()` returns `null` and log errors
- **Logging**: Log when events are processed and when they're ignored
- **Idempotency**: Ensure event handlers are idempotent (can be called multiple times safely)
- **Null checks**: Use `is null` pattern instead of `== null`
- **Publishing only**: Only use `PublishEvent`, never `TrackAndPublishAsync` (reserved for tests)

## Handler Implementation Guidelines

### Handler Base Classes

Choose the appropriate base class based on your data access needs:

#### For PostgreSQL (KYC Domain)

- `KycDomainHandler(KycDomainReadContext)` - Read-only operations
- `KycDomainWriteHandler(KycDomainWriteContext, ILogger)` - Write operations

#### For MongoDB (Risk Domain)

- `RiskDomainHandler(RiskDomainReadContext)` - Read-only operations
- `RiskDomainWriteHandler(RiskDomainWriteContext)` - Write operations

#### For MongoDB (Registry Domain)

- `RegistryDomainHandler(RegistryEntityContext)` - Read-only operations
- `RegistryDomainWriteHandler(RegistryEntityContext)` - Write operations

### Handler Structure

Use primary constructors with proper parameter ordering:

```csharp
// Read Handler Example
public class EntityReadHandler(KycDomainReadContext readContext) : KycDomainHandler(readContext)
{
	public IQueryable<EntityFacade> Query()
		=> EntityContext.Entity
			.Where(e => !e.IsDeleted)
			.ProjectToType<EntityFacade>();

	public async Task<EntityFacade?> GetByIdAsync(Guid id)
		=> await EntityContext.Entity
			.Where(e => e.Id == id && !e.IsDeleted)
			.ProjectTo<EntityFacade>()
			.FirstOrDefaultAsync(CancellationToken);
}

// Write Handler Example
public class EntityWriteHandler(
	KycDomainWriteContext writeContext,
	ILogger<EntityWriteHandler> logger) : KycDomainWriteHandler(writeContext, logger)
{
	public async Task<HandlerWriteResult<EntityFacade>> WriteAsync(Guid? id, Patch<EntityFacade> item)
	{
		var mapper = id.HasValue
			? await EntityContext.GetPatchMapperAsync<Entity>(id.Value, CancellationToken)
			: EntityContext.GetNewMapper<Entity>();

		// Map properties from facade to entity
		mapper.Map(i => i.Name, db => db.Name);
		mapper.Map(i => i.Status, db => db.Status);
		mapper.Map(i => i.IsDeleted, db => db.IsDeleted);

		// Set audit fields
		if (mapper.IsNew)
		{
			mapper.Target.CreatedBy = Principal.UserGuid;
			mapper.Target.CreatedAt = NowUtc;
		}

		mapper.Target.ModifiedBy = Principal.UserGuid;
		mapper.Target.ModifiedAt = NowUtc;

		// Save and return
		await EntityContext.SaveChangesAsync(CancellationToken);
		return new HandlerWriteResult<EntityFacade>
		{
			SuccessValue = mapper.Target.Adapt<EntityFacade>()
		};
	}
}
```

### Key Handler Properties

Available in all handler base classes:

- `Principal` - Current user principal (AcaPrincipal)
- `EntityContext` - The database context
- `CancellationToken` - Cancellation token for async operations
- `NowUtc` - Current UTC time (via GetNowUtc delegate)
- `GetNewGuid()` - Generate new GUID (via GetNewGuid delegate)

For Write Handlers:

- `Logger` - ILogger instance for logging

## Data Access Patterns

### Entity Framework (PostgreSQL/KYC Domain)

#### Querying

```csharp
// Simple query
public IQueryable<EntityFacade> Query()
	=> EntityContext.Entity
		.Where(e => !e.IsDeleted)
		.Where(e => e.ClientId == Principal.ClientGuid)
		.ProjectToType<EntityFacade>();

// With includes
public async Task<EntityDetailFacade?> GetDetailAsync(Guid id)
	=> await EntityContext.Entity
		.Include(e => e.Individuals)
		.Include(e => e.Contacts)
		.Where(e => e.Id == id && !e.IsDeleted)
		.ProjectTo<EntityDetailFacade>()
		.FirstOrDefaultAsync(CancellationToken);
```

#### Updating

```csharp
// Using PatchMapper for updates
public async Task<HandlerWriteResult<EntityFacade>> UpdateAsync(Guid id, Patch<EntityFacade> item)
{
	var mapper = await EntityContext.GetPatchMapperAsync<Entity>(id, CancellationToken);

	mapper.Map(i => i.Name, db => db.Name);
	mapper.Map(i => i.Description, db => db.Description);

	mapper.Target.ModifiedBy = Principal.UserGuid;
	mapper.Target.ModifiedAt = NowUtc;

	await EntityContext.SaveChangesAsync(CancellationToken);
	return HandlerWriteResult.Success(mapper.Target.Adapt<EntityFacade>());
}

// Bulk update with ExecuteUpdateAsync
public async Task<List<EntityFacade>> BulkSoftDeleteAsync(IReadOnlyList<Guid> ids)
{
	await EntityContext.Entity
		.Where(e => ids.Contains(e.Id))
		.ExecuteUpdateAsync(updates => updates
			.SetProperty(e => e.IsDeleted, true)
			.SetProperty(e => e.ModifiedBy, Principal.UserGuid)
			.SetProperty(e => e.ModifiedAt, NowUtc),
			CancellationToken);

	// Fetch updated entities
	var updated = await EntityContext.Entity
		.Where(e => ids.Contains(e.Id))
		.ToListAsync(CancellationToken);

	return updated.Adapt<List<EntityFacade>>();
}
```

### MongoDB (Risk/Registry Domains)

#### Querying

```csharp
// Simple MongoDB query
public async Task<IEnumerable<EventLogFacade>> QueryAsync()
	=> await EntityContext.EventLogs
		.Find(e => !e.IsDeleted && e.ClientId == Principal.ClientId)
		.Project(e => e.Adapt<EventLogFacade>())
		.ToListAsync(CancellationToken);

// With aggregation
public async Task<List<EntitySummary>> GetSummariesAsync()
{
	var result = await EntityContext.EventLogs
		.Aggregate()
		.Match(x => !x.IsDeleted)
		.Group(x => x.EntityId,
			g => new EntitySummary
			{
				EntityId = g.Key,
				Count = g.Count()
			})
		.ToListAsync(CancellationToken);

	return result;
}
```

#### Updating

```csharp
// MongoDB update
public async Task UpdateScreeningNameAsync(string id, string newStatus)
{
	var update = Builders<ScreeningNameEntity>.Update
		.Set(x => x.Status, newStatus)
		.Set(x => x.LastModifiedDate, NowUtc)
		.Set(x => x.LastModifiedByUserId, Principal.UserId);

	await EntityContext.ScreeningNames.UpdateOneAsync(
		x => x.Id == id && x.ClientId == Principal.ClientId,
		update,
		new UpdateOptions { IsUpsert = false },
		CancellationToken);
}
```

## Client Filtering

Always apply client filtering to ensure multi-tenancy:

### PostgreSQL

```csharp
// Client filtering is built into queries
public IQueryable<EntityFacade> Query()
	=> EntityContext.Entity
		.Where(e => e.ClientId == Principal.ClientGuid) // Client filter
		.Where(e => !e.IsDeleted)
		.ProjectToType<EntityFacade>();

// Or use extension method
public IQueryable<EntityFacade> Query()
	=> EntityContext.Entity
		.ApplyFilterFromClient(Principal.ClientGuid)
		.Where(e => !e.IsDeleted)
		.ProjectToType<EntityFacade>();
```

### MongoDB

```csharp
// Apply client filter in find/match operations
public async Task<ScreeningNameEntity?> GetAsync(string id)
	=> await EntityContext.ScreeningNames
		.Find(x => x.Id == id && x.ClientId == Principal.ClientId)
		.FirstOrDefaultAsync(CancellationToken);

// Or use extension methods
public IEnumerable<EventLogFacade> Query()
	=> EntityContext.EventLogs
		.ApplyClientFilter(Principal.ClientId)
		.ProjectToType<EventLogFacade>()
		.ToList();
```

## Mapping

Use Mapster for entity-to-facade mapping:

```csharp
// Simple mapping
var facade = entity.Adapt<EntityFacade>();

// Collection mapping
var facades = entities.Adapt<List<EntityFacade>>();

// LINQ projection (more efficient)
var query = EntityContext.Entity
	.Where(e => !e.IsDeleted)
	.ProjectToType<EntityFacade>(); // or .ProjectTo<EntityFacade>()
```

## Error Handling

Use appropriate exception types:

```csharp
// Item not found
if (entity == null)
{
	throw new ItemNotFoundException($"Entity with id '{id}' was not found.");
}

// Validation errors
if (string.IsNullOrWhiteSpace(request.Name))
{
	throw new ValidationException("Name is required.");
}

// General errors - log and throw
try
{
	// Operation
}
catch (Exception ex)
{
	logger.LogError(ex, "Failed to process entity {EntityId}", id);
	throw;
}
```

## Logging

Use structured logging with appropriate log levels:

```csharp
// Information
logger.LogInformation("Processing entity {EntityId} for client {ClientId}", entityId, Principal.ClientId);

// Warning
logger.LogWarning("Entity {EntityId} not found, skipping update", entityId);

// Error — always pass the exception object as the first argument
logger.LogError(ex, "Failed to update entity {EntityId}", entityId);
```

## SonarQube Rules

### MongoDB/EF Core Specific

- **S2971**: Avoid unnecessary `.ToList()` calls when using LINQ to Entities or MongoDB Driver
  - Severity: Suggestion in `Source/AntiMoneyLaundering/.editorconfig`
  - Use `.AsEnumerable()` instead if you need to switch to LINQ to Objects

## Best Practices Specific to This Project

1. **Separate Read and Write Operations**
   - Create separate `*ReadHandler` and `*WriteHandler` classes
   - Read handlers use read-only contexts
   - Write handlers use writable contexts

2. **Use Primary Constructors**
   - Prefer primary constructor syntax for dependency injection
   - Keep dependencies minimal and focused

3. **Never Hard Delete**
   - Always use soft deletes (`IsDeleted = true`)
   - Never cascade delete

4. **Publish Events for Writes**
   - All insert/update/delete operations should publish events
   - Use the `PublishEvent` method from `IAmlEventPublisher`
   - Never use `TrackAndPublishAsync` (reserved for unit tests only)
   - See [Event Publishing and Listening](#event-publishing-and-listening) for detailed examples

5. **Follow SOLID Principles**
   - Each method does one thing (SRP)
   - Extract common logic to avoid duplication (DRY)
   - Handlers contain business logic and orchestration

6. **Security**
   - Always apply client filtering
   - Use `Principal` for current user context
   - Never expose internal IDs in APIs
