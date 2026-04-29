# GitHub Copilot Instructions - AML Core Tests

## Overview

This project contains unit tests for the Anti Money Laundering (AML) domain logic. All tests use MSTest framework and follow specific patterns for testing handlers and domain logic.

**Compliance Note**: Since this is a regulatory compliance system, tests must not only verify that code works, but also that:

- Business rules are enforced (e.g., soft delete only, never hard delete)
- Audit trails are maintained (timestamps, user tracking)
- Client data is properly isolated (no data leakage between clients)
- AI results are treated as suggestions (not decisions)
- Compliance workflows are followed (screening → results → alerts → decisions)

## Test Structure

### Directory Structure

Tests should mirror the structure of the code they test:

- `Domain/{Feature}/{HandlerName}Tests.cs` - Tests for handlers in `Aca.AntiMoneyLaundering/Domain/{Feature}/{HandlerName}.cs`

### Test Class Pattern

All test classes should:

1. Inherit from `UnitTest<TSubject>` where `TSubject` is the class being tested
2. Use `[TestClass]` attribute
3. Follow the naming convention: `{ClassName}Tests`

```csharp
[TestClass]
public class EntityReadHandlerTests : UnitTest<EntityReadHandler>
{
	[TestInitialize]
	public Task Setup()
	{
		// Optional setup that runs before each test
		return Task.CompletedTask;
	}

	[TestMethod]
	public async Task GetByIdAsync_ExistingEntity_ReturnsEntity()
	{
		// Arrange
		var entityId = Guid.NewGuid();
		await UpdateKycEntityContextAsync(db =>
		{
			db.Add(new Entity
			{
				Id = entityId,
				ClientId = PrincipalClientGuid,
				Name = "Test Entity",
				IsDeleted = false,
				CreatedBy = PrincipalUserGuid,
				CreatedAt = DateTime.UtcNow
			});
		});

		// Act
		var result = await GetTestSubject().GetByIdAsync(entityId);

		// Assert
		result.Should().NotBeNull();
		result!.Id.Should().Be(entityId);
		result.Name.Should().Be("Test Entity");
	}

	[TestMethod]
	public async Task GetByIdAsync_DeletedEntity_ReturnsNull()
	{
		// Arrange
		var entityId = Guid.NewGuid();
		await UpdateKycEntityContextAsync(db =>
		{
			db.Add(new Entity
			{
				Id = entityId,
				ClientId = PrincipalClientGuid,
				Name = "Deleted Entity",
				IsDeleted = true,
				CreatedBy = PrincipalUserGuid,
				CreatedAt = DateTime.UtcNow
			});
		});

		// Act
		var result = await GetTestSubject().GetByIdAsync(entityId);

		// Assert
		result.Should().BeNull();
	}
}
```

## Test Method Naming

Use descriptive test names that follow the pattern:
`{MethodName}_{Scenario}_{ExpectedOutcome}`

Examples:

- `GetByIdAsync_ExistingEntity_ReturnsEntity`
- `WriteAsync_NewEntity_CreatesSuccessfully`
- `DeleteAsync_NonExistentEntity_ThrowsItemNotFoundException`
- `Query_WithDeletedEntities_ExcludesDeletedItems`
- `BulkDeleteAsync_MultipleEntities_SoftDeletesAll`

## Test Coverage Requirements

### Coverage Standards

- **Goal: 100% line coverage** for all changed lines - this is the target, not optional
- **80% line coverage** is the absolute bare minimum threshold - never settle for this
- Always strive for complete coverage; 80% means you're leaving 20% of your code untested

### Logical Coverage

Beyond line coverage, test all logical branches:

```csharp
// For this method:
public async Task<Entity?> GetByStatusAsync(string status)
{
	var entity = await _context.Entity
		.Where(e => e.Status == status && !e.IsDeleted)
		.FirstOrDefaultAsync();

	return entity != null ? entity : null;
}

// Write these tests:
[TestMethod]
public async Task GetByStatusAsync_EntityWithStatus_ReturnsEntity() { }

[TestMethod]
public async Task GetByStatusAsync_NoEntityWithStatus_ReturnsNull() { }

[TestMethod]
public async Task GetByStatusAsync_DeletedEntityWithStatus_ReturnsNull() { }
```

## Single Responsibility in Tests

**CRITICAL**: Each test method must test ONE specific scenario.

```csharp
// ❌ BAD - Testing multiple scenarios in one test
[TestMethod]
public async Task WriteAsync_VariousScenarios()
{
	// Test create
	var createResult = await GetTestSubject().WriteAsync(null, new Patch<EntityFacade> { /* ... */ });
	createResult.Should().NotBeNull();

	// Test update
	var updateResult = await GetTestSubject().WriteAsync(createResult.Id, new Patch<EntityFacade> { /* ... */ });
	updateResult.Should().NotBeNull();

	// Test validation
	await Assert.ThrowsExceptionAsync<ValidationException>(() =>
		GetTestSubject().WriteAsync(null, new Patch<EntityFacade> { /* invalid data */ }));
}

// ✅ GOOD - Separate tests for each scenario
[TestMethod]
public async Task WriteAsync_NewEntity_CreatesSuccessfully()
{
	// Arrange
	var facade = new Patch<EntityFacade>
	{
		Value = new EntityFacade { Name = "New Entity" }
	};

	// Act
	var result = await GetTestSubject().WriteAsync(null, facade);

	// Assert
	result.Should().NotBeNull();
	result.SuccessValue.Should().NotBeNull();
	result.SuccessValue.Name.Should().Be("New Entity");
}

[TestMethod]
public async Task WriteAsync_ExistingEntity_UpdatesSuccessfully()
{
	// Arrange
	var entityId = Guid.NewGuid();
	await UpdateKycEntityContextAsync(db =>
	{
		db.Add(new Entity
		{
			Id = entityId,
			ClientId = PrincipalClientGuid,
			Name = "Original Name",
			IsDeleted = false,
			CreatedBy = PrincipalUserGuid,
			CreatedAt = DateTime.UtcNow
		});
	});

	var facade = new Patch<EntityFacade>
	{
		Value = new EntityFacade { Name = "Updated Name" }
	};

	// Act
	var result = await GetTestSubject().WriteAsync(entityId, facade);

	// Assert
	result.SuccessValue.Name.Should().Be("Updated Name");
}

[TestMethod]
public async Task WriteAsync_InvalidData_ThrowsValidationException()
{
	// Arrange
	var facade = new Patch<EntityFacade>
	{
		Value = new EntityFacade { Name = null } // Invalid - name required
	};

	// Act & Assert
	await Assert.ThrowsExceptionAsync<ValidationException>(() =>
		GetTestSubject().WriteAsync(null, facade));
}
```

## Test Both Positive and Negative Cases

For every feature, test:

1. **Happy path** (positive case) - operation succeeds
2. **Edge cases** - boundary conditions, empty collections, etc.
3. **Error cases** (negative cases) - validation failures, not found, etc.

```csharp
// Feature: Delete entity
[TestMethod]
public async Task DeleteAsync_ExistingEntity_SetsIsDeletedTrue() { /* Happy path */ }

[TestMethod]
public async Task DeleteAsync_NonExistentEntity_ThrowsItemNotFoundException() { /* Error case */ }

[TestMethod]
public async Task DeleteAsync_AlreadyDeleted_RemainsDeleted() { /* Edge case */ }
```

## Test Data

### GUID Generation

**CRITICAL**: Always use valid v4 GUIDs in test data. Never use placeholder strings, sequential numbers, or invalid formats.

```csharp
// ✅ GOOD - Valid v4 GUIDs
var entityId = new Guid("550e8400-e29b-41d4-a716-446655440000");
var clientId = new Guid("294a5c5f-7d35-43d7-a45d-83668bd6babb");
var userId = new Guid("face1e55-deb5-50d5-f1ed-0ddba1150ff5");

// ❌ BAD - Invalid/placeholder GUIDs
var entityId = new Guid("00000000-0000-0000-0000-000000000001"); // All zeros/ones not realistic
var clientId = Guid.Empty; // Never use empty GUID
var userId = new Guid(); // Uninitialized
```

Resources for generating valid v4 GUIDs:
- Online: https://www.uuidgenerator.net/version4
- PowerShell: `[guid]::NewGuid()`
- C#: `Guid.NewGuid()`

## Database Context Helpers

### PostgreSQL (KYC Domain)

```csharp
// Add data to context
await UpdateKycEntityContextAsync(db =>
{
	db.Add(new Entity { /* ... */ });
	db.AddRange(new[] { entity1, entity2, entity3 });
});

// Verify data in context
UpdateEntityContext(db =>
{
	var entity = db.Entity.Find(entityId);
	entity.Should().NotBeNull();
	entity.IsDeleted.Should().BeTrue();
});
```

### MongoDB (Risk/Registry Domains)

```csharp
// Add data to MongoDB
await UpdateEntityContextAsync(async db =>
{
	await db.ScreeningNames.InsertOneAsync(new ScreeningNameEntity { /* ... */ });
	await db.ScreeningNames.InsertManyAsync(new[] { name1, name2, name3 });
});

// Verify data in MongoDB
UpdateEntityContext(db =>
{
	var screeningName = db.ScreeningNames.Find(x => x.Id == id).FirstOrDefault();
	screeningName.Should().NotBeNull();
	screeningName.IsDeleted.Should().BeTrue();
});
```

## Available Test Helpers

### Principal Context

Tests automatically run with a mock principal:

- `PrincipalUserId` - User ID (int, default: 1)
- `PrincipalUserGuid` - User GUID
- `PrincipalClientId` - Client ID (int, default: 1)
- `PrincipalClientGuid` - Client GUID
- `MockPrincipal` - Full AcaPrincipal object

### Getting Test Subject

```csharp
// Get instance of class being tested
var handler = GetTestSubject();
```

### Mocking Dependencies

When handlers require complex dependencies, add them in the `AddServices` method:

```csharp
[TestClass]
public class EntityWriteHandlerTests : UnitTest<EntityWriteHandler>
{
	protected override void AddServices(IServiceCollection services)
	{
		base.AddServices(services);

		// Add required handlers/services
		services.AddScoped<ReferenceDataReadHandler>();
		services.AddScoped<ValidationService>();

		// Add mocked services
		var mockEmailService = new Mock<IEmailService>();
		services.AddScoped(_ => mockEmailService.Object);

		// Add HTTP mock responses
		MockHttpHandler.AddResponse(
			new Uri("https://api.example.com/endpoint"),
			HttpStatusCode.OK,
			new { Success = true });
	}
}
```

## Assertions

Use FluentAssertions for readable assertions:

```csharp
// Basic assertions
result.Should().NotBeNull();
result.Should().BeNull();
result.Should().Be(expectedValue);
result.Should().BeEquivalentTo(expected);

// String assertions
name.Should().Be("Expected Name");
name.Should().Contain("partial");
name.Should().StartWith("prefix");
name.Should().NotBeNullOrWhiteSpace();

// Collection assertions
list.Should().HaveCount(3);
list.Should().NotBeEmpty();
list.Should().Contain(item);
list.Should().OnlyContain(x => !x.IsDeleted);
list.Should().BeInAscendingOrder(x => x.Name);

// Boolean assertions
isDeleted.Should().BeTrue();
isActive.Should().BeFalse();

// Exception assertions
await Assert.ThrowsExceptionAsync<ItemNotFoundException>(() =>
	handler.GetAsync(nonExistentId));

// Or with FluentAssertions
var act = () => handler.GetAsync(nonExistentId);
await act.Should().ThrowAsync<ItemNotFoundException>();
```

## Testing Patterns by Handler Type

### Read Handler Tests

```csharp
[TestMethod]
public async Task Query_WithData_ReturnsFiltered()
{
	// Arrange - Create test data
	await UpdateKycEntityContextAsync(db =>
	{
		db.AddRange(
			new Entity { Id = Guid.NewGuid(), ClientId = PrincipalClientGuid, IsDeleted = false, Name = "Active" },
			new Entity { Id = Guid.NewGuid(), ClientId = PrincipalClientGuid, IsDeleted = true, Name = "Deleted" },
			new Entity { Id = Guid.NewGuid(), ClientId = Guid.NewGuid(), IsDeleted = false, Name = "Other Client" }
		);
	});

	// Act
	var results = await GetTestSubject().Query().ToListAsync();

	// Assert
	results.Should().HaveCount(1);
	results[0].Name.Should().Be("Active");
}
```

### Write Handler Tests

```csharp
[TestMethod]
public async Task WriteAsync_NewEntity_SetsAuditFields()
{
	// Arrange
	var facade = new Patch<EntityFacade>
	{
		Value = new EntityFacade { Name = "Test Entity" }
	};

	// Act
	var result = await GetTestSubject().WriteAsync(null, facade);

	// Assert
	result.SuccessValue.Should().NotBeNull();

	UpdateEntityContext(db =>
	{
		var entity = db.Entity.Find(result.SuccessValue.Id);
		entity.CreatedBy.Should().Be(PrincipalUserGuid);
		entity.ModifiedBy.Should().Be(PrincipalUserGuid);
		entity.CreatedAt.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromSeconds(5));
		entity.ModifiedAt.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromSeconds(5));
	});
}
```

### Delete Handler Tests

```csharp
[TestMethod]
public async Task DeleteAsync_ExistingEntity_SoftDeletes()
{
	// Arrange
	var entityId = Guid.NewGuid();
	await UpdateKycEntityContextAsync(db =>
	{
		db.Add(new Entity
		{
			Id = entityId,
			ClientId = PrincipalClientGuid,
			Name = "To Delete",
			IsDeleted = false,
			CreatedBy = PrincipalUserGuid,
			CreatedAt = DateTime.UtcNow
		});
	});

	// Act
	var result = await GetTestSubject().DeleteAsync(entityId);

	// Assert
	result.Should().BeTrue();

	UpdateEntityContext(db =>
	{
		var entity = db.Entity.Find(entityId);
		entity.IsDeleted.Should().BeTrue();
		entity.ModifiedBy.Should().Be(PrincipalUserGuid);
		entity.ModifiedAt.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromSeconds(5));
	});
}
```

## Performance Considerations

Keep tests fast:

- Use in-memory databases (automatically configured)
- Avoid unnecessary Thread.Sleep() calls
- Use minimal test data
- Clean up is automatic between tests

## Common Patterns

### Testing Conditional Logic

```csharp
// For every if/else or switch, create separate tests
[TestMethod]
public async Task ProcessEntity_WhenActive_ProcessesSuccessfully() { }

[TestMethod]
public async Task ProcessEntity_WhenInactive_SkipsProcessing() { }

[TestMethod]
public async Task ProcessEntity_WhenDeleted_ThrowsException() { }
```

### Testing List Operations

```csharp
[TestMethod]
public async Task Query_WithNoData_ReturnsEmptyList()
{
	// Act
	var results = await GetTestSubject().Query().ToListAsync();

	// Assert
	results.Should().BeEmpty();
}

[TestMethod]
public async Task Query_WithMultipleItems_ReturnsFiltered()
{
	// Arrange - add multiple items
	// Act
	// Assert - verify correct items returned
}
```

### Testing Ordering

```csharp
[TestMethod]
public async Task Query_OrdersByName_ReturnsCorrectOrder()
{
	// Arrange
	await UpdateKycEntityContextAsync(db =>
	{
		db.AddRange(
			new Entity { Name = "Zulu" },
			new Entity { Name = "Alpha" },
			new Entity { Name = "Bravo" }
		);
	});

	// Act
	var results = await GetTestSubject().Query().ToListAsync();

	// Assert
	results.Should().BeInAscendingOrder(x => x.Name);
	results[0].Name.Should().Be("Alpha");
	results[1].Name.Should().Be("Bravo");
	results[2].Name.Should().Be("Zulu");
}
```

## Best Practices

1. **Arrange-Act-Assert Pattern**: Always structure tests clearly
2. **One Assert Per Concept**: Multiple assertions are OK if testing the same concept
3. **Descriptive Names**: Test names should read like documentation
4. **Test Independence**: Tests should not depend on each other
5. **Minimal Setup**: Only create data needed for the specific test
6. **Clean Tests**: Tests should be as simple as possible
7. **Fast Execution**: Keep tests running quickly

## Anti-Patterns to Avoid

❌ **Don't**: Test multiple scenarios in one method
❌ **Don't**: Use production database connections
❌ **Don't**: Create test dependencies (TestMethod1 must run before TestMethod2)
❌ **Don't**: Test framework/library code (e.g., testing that EF works)
❌ **Don't**: Test log messages or logging behavior
❌ **Don't**: Use random data that could cause flaky tests
❌ **Don't**: Skip negative test cases

✅ **Do**: Test one scenario per method
✅ **Do**: Use in-memory databases (automatic)
✅ **Do**: Make tests independent
✅ **Do**: Test your business logic
✅ **Do**: Use predictable test data
✅ **Do**: Test both success and failure paths
