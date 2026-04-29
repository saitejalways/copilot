# GitHub Copilot Instructions - AML API Tests

## Overview

This project contains API integration tests for the Anti Money Laundering (AML) API endpoints. Tests use Reqnroll (formerly SpecFlow), which is a BDD (Behavior-Driven Development) framework that uses Gherkin syntax.

**Compliance Context**: API tests must verify not only that endpoints work correctly, but also that:

- Multi-tenant client isolation is enforced (users cannot access other clients' data)
- Regulatory workflows are supported (screening, alerting, decision-making)
- Compliance data is protected (audit trails, soft deletes, not hard deletes)
- AI results are presented as suggestions requiring analyst acceptance
- Real-time notifications work for analyst alerts
- All write operations are properly logged and auditable

## Test Structure

### Directory Structure

```
Aca.AntiMoneyLaundering.Api.Tests/
├── Controllers/
│   ├── {ControllerName}/
│   │   ├── Get.feature          # GET endpoint tests
│   │   ├── Post.feature         # POST endpoint tests
│   │   ├── Patch.feature        # PATCH endpoint tests
│   │   ├── Delete.feature       # DELETE endpoint tests
│   │   └── Query.feature        # OData query tests
├── Steps/
│   ├── DataSteps.cs            # Given steps for data setup
│   ├── HttpSteps.cs            # When steps for HTTP requests
│   └── AssertionSteps.cs       # Then steps for assertions
├── Contexts/
│   ├── ServiceContext.cs       # DI container context
│   ├── HttpContext.cs          # HTTP request/response context
│   └── PlaceholderContext.cs   # Dynamic value substitution
└── HostedServices/             # Tests for background services
```

## Feature File Structure

### Basic Template

```gherkin
Feature: {Controller} - {Operation}
    As a user of the API
    I want to {perform action}
    So that {business value}

Background:
    Given I am authenticated as a user with the following details:
        | UserId | ClientId | UserGuid                             | ClientGuid                           |
        | 1      | 1        | face1e55-deb5-50d5-f1ed-0ddba1150ff5 | 294a5c5f-7d35-43d7-a45d-83668bd6babb |

Scenario: {Specific scenario description with expected outcome}
    Given I have the following records in the kyc '{TableName}' table
        | Field1 | Field2 | Field3 |
        | Value1 | Value2 | Value3 |
    When I send a GET request to "/api/v3/{controller}/{id}"
    Then the response status code should be 200
    And the response should contain the following JSON:
        """
        {
            "field1": "Value1",
            "field2": "Value2"
        }
        """
```

### Naming Conventions

#### Feature Names

- Format: `{Controller} - {HttpMethod}` or `{Controller} - {Operation}`
- Examples:
  - `Entity - Get`
  - `Entity - Post`
  - `Entity - Query`
  - `EventService - WorkConnectSync`

#### Scenario Names

Use descriptive names that describe:

1. The operation being performed
2. The expected outcome

Pattern: `{Operation}_{Condition}_ReturnsOr{ExpectedResult}`

Examples:

```gherkin
Scenario: Get entity by ID - existing entity - returns entity
Scenario: Get entity by ID - non-existent entity - returns 404
Scenario: Get entity by ID - deleted entity - returns 404
Scenario: Create entity - valid data - returns 201 with created entity
Scenario: Create entity - invalid data - returns 400 with validation errors
Scenario: Update entity - valid changes - returns 200 with updated entity
Scenario: Delete entity - existing entity - returns 204 and soft deletes
Scenario: Query entities - with filter - returns filtered results
```

## Test Coverage Requirements

### API Endpoint Coverage

For **every API endpoint** (GET, POST, PATCH, PUT, DELETE), create tests for:

1. **Happy Path** (200-level responses)
   - Valid request returns expected result
   - Authentication and authorization succeed

2. **Client Data Isolation** (Multi-tenancy)
   - User cannot access data from other clients
   - Queries are properly filtered by ClientId

3. **Validation Errors** (400-level responses)
   - Missing required fields
   - Invalid data formats
   - Business rule violations

4. **Not Found Cases** (404 responses)
   - Requested resource doesn't exist
   - Soft-deleted resources return 404

5. **Authorization** (403 responses) - if applicable
   - User lacks required permissions

6. **Server Errors** (500-level responses) - if applicable
   - Handle edge cases gracefully

### Coverage Standards

- **Goal: 100% endpoint coverage** for all endpoints - this is the standard we aim for
- **80% endpoint coverage** is the absolute bare minimum threshold - not acceptable as a target
- Every endpoint should have comprehensive test scenarios covering all paths and edge cases

## Single Responsibility in Scenarios

**CRITICAL**: Each scenario must test ONE specific behavior.

```gherkin
# ❌ BAD - Testing multiple things in one scenario
Scenario: Entity operations
    Given I have an entity
    When I get the entity
    Then I should receive it
    When I update the entity
    Then it should be updated
    When I delete the entity
    Then it should be deleted

# ✅ GOOD - Separate scenarios for each operation
Scenario: Get entity - existing entity - returns entity
    Given I have the following records in the kyc 'Entity' table
        | Id                                   | Name          | IsDeleted |
        | 550e8400-e29b-41d4-a716-446655440000 | Test Entity   | false     |
    When I send a GET request to "/api/v3/entity/550e8400-e29b-41d4-a716-446655440000"
    Then the response status code should be 200

Scenario: Update entity - valid changes - returns updated entity
    Given I have the following records in the kyc 'Entity' table
        | Id                                   | Name          | IsDeleted |
        | 550e8400-e29b-41d4-a716-446655440000 | Original Name | false     |
    When I send a PATCH request to "/api/v3/entity/550e8400-e29b-41d4-a716-446655440000" with JSON:
        """
        {
            "name": "Updated Name"
        }
        """
    Then the response status code should be 200

Scenario: Delete entity - existing entity - soft deletes
    Given I have the following records in the kyc 'Entity' table
        | Id                                   | Name          | IsDeleted |
        | 550e8400-e29b-41d4-a716-446655440000 | Test Entity   | false     |
    When I send a DELETE request to "/api/v3/entity/550e8400-e29b-41d4-a716-446655440000"
    Then the response status code should be 204
    And the following records should be in the kyc 'Entity' table
        | Id                                   | IsDeleted |
        | 550e8400-e29b-41d4-a716-446655440000 | true      |
```

## Test Data - GUID Requirements

**CRITICAL**: Always use valid v4 GUIDs in test data tables. Never use placeholder strings, sequential numbers, or invalid formats.

✅ **Valid v4 GUIDs** (used in examples below):
- `550e8400-e29b-41d4-a716-446655440000`
- `294a5c5f-7d35-43d7-a45d-83668bd6babb`
- `face1e55-deb5-50d5-f1ed-0ddba1150ff5`

❌ **Invalid GUIDs to avoid**:
- All zeros: `00000000-0000-0000-0000-000000000000`
- Sequential: `00000000-0000-0000-0000-000000000001`

Generate valid GUIDs: https://www.uuidgenerator.net/version4

## Common Step Patterns

### Given Steps (Data Setup)

```gherkin
# Set authenticated user
Given I am authenticated as a user with the following details:
    | UserId | ClientId | UserGuid                             | ClientGuid                           |
    | 1      | 1        | face1e55-deb5-50d5-f1ed-0ddba1150ff5 | 294a5c5f-7d35-43d7-a45d-83668bd6babb |

# Add data to PostgreSQL (KYC domain)
Given I have the following records in the kyc 'Entity' table
    | Id                                   | ClientId                             | Name        | IsDeleted | CreatedBy                            |
    | 550e8400-e29b-41d4-a716-446655440000 | 294a5c5f-7d35-43d7-a45d-83668bd6babb | Test Entity | false     | face1e55-deb5-50d5-f1ed-0ddba1150ff5 |

# Add data to MongoDB (Risk domain)
Given I have the following records in the 'ScreeningNames' table
    | Id                   | ClientId | FullName    | IsDeleted | CreatedByUserId |
    | 507f1f77bcf86cd799439011 | 1      | John Doe    | false     | 1               |

# Add data to MongoDB (Registry domain)
Given I have the following records in the registry 'ThirdPartyLegalEntities' table
    | Id                   | ClientId | Name            | IsDeleted | CreatedByUserId |
    | 507f1f77bcf86cd799439012 | 1      | Test Company    | false     | 1               |

# Add JSON data for complex objects
Given I have the following JSON records in the kyc 'Entity' table
    | Data                                                                          |
    | {"id": "550e8400-e29b-41d4-a716-446655440000", "name": "Complex", ...}       |
```

### When Steps (HTTP Requests)

```gherkin
# GET requests
When I send a GET request to "/api/v3/entity/550e8400-e29b-41d4-a716-446655440000"

# GET with query parameters
When I send a GET request to "/api/v3/entity?$filter=name eq 'Test'"

# POST requests
When I send a POST request to "/api/v3/entity" with JSON:
    """
    {
        "name": "New Entity",
        "status": "Active"
    }
    """

# PATCH requests
When I send a PATCH request to "/api/v3/entity/550e8400-e29b-41d4-a716-446655440000" with JSON:
    """
    {
        "name": "Updated Name"
    }
    """

# DELETE requests
When I send a DELETE request to "/api/v3/entity/550e8400-e29b-41d4-a716-446655440000"
```

### Then Steps (Assertions)

```gherkin
# Status code assertions
Then the response status code should be 200
Then the response status code should be 201
Then the response status code should be 204
Then the response status code should be 400
Then the response status code should be 404

# JSON response assertions
Then the response should contain the following JSON:
    """
    {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "name": "Test Entity"
    }
    """

# JSON array response assertions
Then the response should contain JSON with an array:
    """
    [
        {
            "id": "550e8400-e29b-41d4-a716-446655440000",
            "name": "Entity 1"
        },
        {
            "id": "550e8400-e29b-41d4-a716-446655440001",
            "name": "Entity 2"
        }
    ]
    """

# Database state assertions
Then the following records should be in the kyc 'Entity' table
    | Id                                   | Name          | IsDeleted |
    | 550e8400-e29b-41d4-a716-446655440000 | Updated Entity | false    |

# Verify data NOT in database
Then the following records should not be in the kyc 'Entity' table
    | Id                                   | IsDeleted |
    | 550e8400-e29b-41d4-a716-446655440000 | false     |
```

## Placeholders and Dynamic Values

Use placeholders for dynamic generated values:

```gherkin
# Using placeholders in Given
Given I have the following records in the kyc 'Entity' table
    | Id               | ClientId         | Name        |
    | {{GeneratedId1}} | {{ClientGuid}}   | Test Entity |

# Reference placeholder in When
When I send a GET request to "/api/v3/entity/{{GeneratedId1}}"

# Assert placeholder in Then
Then the response should contain the following JSON:
    """
    {
        "id": "{{GeneratedId1}}",
        "clientId": "{{ClientGuid}}"
    }
    """
```

Available placeholder patterns:

- `{{GeneratedId1}}`, `{{GeneratedId2}}`, etc. - Auto-generated GUIDs
- `{{ClientGuid}}` - Current principal's client GUID
- `{{UserGuid}}` - Current principal's user GUID

## Testing Multi-Tenancy (Client Isolation)

**CRITICAL**: Always test that users cannot access other clients' data.

```gherkin
Scenario: Get entity - from different client - returns 404
    Given I am authenticated as a user with the following details:
        | UserId | ClientId | UserGuid                             | ClientGuid                           |
        | 1      | 1        | face1e55-deb5-50d5-f1ed-0ddba1150ff5 | 294a5c5f-7d35-43d7-a45d-83668bd6babb |
    And I have the following records in the kyc 'Entity' table
        | Id                                   | ClientId                             | Name               | IsDeleted |
        | 550e8400-e29b-41d4-a716-446655440000 | 9f1c2a10-2b6d-4f6b-9c3e-8d5f7b6a1c2d | Other Client Data  | false     |
    When I send a GET request to "/api/v3/entity/550e8400-e29b-41d4-a716-446655440000"
    Then the response status code should be 404

Scenario: Query entities - includes only user's client data
    Given I am authenticated as a user with the following details:
        | UserId | ClientId | UserGuid                             | ClientGuid                           |
        | 1      | 1        | face1e55-deb5-50d5-f1ed-0ddba1150ff5 | 294a5c5f-7d35-43d7-a45d-83668bd6babb |
    And I have the following records in the kyc 'Entity' table
        | Id                                   | ClientId                             | Name              | IsDeleted |
        | 550e8400-e29b-41d4-a716-446655440001 | 294a5c5f-7d35-43d7-a45d-83668bd6babb | My Client Entity  | false     |
        | 550e8400-e29b-41d4-a716-446655440002 | 9f1c2a10-2b6d-4f6b-9c3e-8d5f7b6a1c2d | Other Client      | false     |
    When I send a GET request to "/api/v3/entity"
    Then the response status code should be 200
    And the JSON response array should have 1 item
```

## Testing Soft Deletes

Verify that deleted records behave correctly:

```gherkin
Scenario: Get entity - deleted entity - returns 404
    Given I have the following records in the kyc 'Entity' table
        | Id                                   | ClientId                             | Name            | IsDeleted |
        | 550e8400-e29b-41d4-a716-446655440000 | 294a5c5f-7d35-43d7-a45d-83668bd6babb | Deleted Entity  | true      |
    When I send a GET request to "/api/v3/entity/550e8400-e29b-41d4-a716-446655440000"
    Then the response status code should be 404

Scenario: Query entities - excludes deleted records
    Given I have the following records in the kyc 'Entity' table
        | Id                                   | ClientId                             | Name     | IsDeleted |
        | 550e8400-e29b-41d4-a716-446655440001 | 294a5c5f-7d35-43d7-a45d-83668bd6babb | Active   | false     |
        | 550e8400-e29b-41d4-a716-446655440002 | 294a5c5f-7d35-43d7-a45d-83668bd6babb | Deleted  | true      |
    When I send a GET request to "/api/v3/entity"
    Then the response status code should be 200
    And the JSON response array should have 1 item

Scenario: Delete entity - sets IsDeleted true
    Given I have the following records in the kyc 'Entity' table
        | Id                                   | ClientId                             | Name        | IsDeleted |
        | 550e8400-e29b-41d4-a716-446655440000 | 294a5c5f-7d35-43d7-a45d-83668bd6babb | To Delete   | false     |
    When I send a DELETE request to "/api/v3/entity/550e8400-e29b-41d4-a716-446655440000"
    Then the response status code should be 204
    And the following records should be in the kyc 'Entity' table
        | Id                                   | IsDeleted |
        | 550e8400-e29b-41d4-a716-446655440000 | true      |
```

## Testing OData Queries

For endpoints that support OData:

```gherkin
Scenario: Query entities - filter by name
    Given I have the following records in the kyc 'Entity' table
        | Id                                   | ClientId                             | Name      | IsDeleted |
        | 550e8400-e29b-41d4-a716-446655440001 | 294a5c5f-7d35-43d7-a45d-83668bd6babb | Alpha     | false     |
        | 550e8400-e29b-41d4-a716-446655440002 | 294a5c5f-7d35-43d7-a45d-83668bd6babb | Beta      | false     |
    When I send a GET request to "/api/v3/entity?$filter=name eq 'Alpha'"
    Then the response status code should be 200
    And the JSON response array should have 1 item

Scenario: Query entities - order by name
    Given I have the following records in the kyc 'Entity' table
        | Id                                   | ClientId                             | Name      | IsDeleted |
        | 550e8400-e29b-41d4-a716-446655440001 | 294a5c5f-7d35-43d7-a45d-83668bd6babb | Zulu      | false     |
        | 550e8400-e29b-41d4-a716-446655440002 | 294a5c5f-7d35-43d7-a45d-83668bd6babb | Alpha     | false     |
    When I send a GET request to "/api/v3/entity?$orderby=name"
    Then the response status code should be 200
    And the first item in the JSON response array should have "name" equal to "Alpha"

Scenario: Query entities - select specific fields
    Given I have the following records in the kyc 'Entity' table
        | Id                                   | ClientId                             | Name      | Status    | IsDeleted |
        | 550e8400-e29b-41d4-a716-446655440001 | 294a5c5f-7d35-43d7-a45d-83668bd6babb | Test      | Active    | false     |
    When I send a GET request to "/api/v3/entity?$select=id,name"
    Then the response status code should be 200
    And the response should only contain fields: "id", "name"
```

## Testing Validation

Test all validation rules:

```gherkin
Scenario: Create entity - missing required field - returns 400
    When I send a POST request to "/api/v3/entity" with JSON:
        """
        {
            "status": "Active"
        }
        """
    Then the response status code should be 400
    And the response should contain validation errors for "name"

Scenario: Create entity - invalid format - returns 400
    When I send a POST request to "/api/v3/entity" with JSON:
        """
        {
            "name": "Test",
            "email": "not-an-email"
        }
        """
    Then the response status code should be 400
    And the response should contain validation errors for "email"

Scenario: Update entity - business rule violation - returns 400
    Given I have the following records in the kyc 'Entity' table
        | Id                                   | ClientId                             | Name      | Status    | IsDeleted |
        | 550e8400-e29b-41d4-a716-446655440000 | 294a5c5f-7d35-43d7-a45d-83668bd6babb | Test      | Closed    | false     |
    When I send a PATCH request to "/api/v3/entity/550e8400-e29b-41d4-a716-446655440000" with JSON:
        """
        {
            "status": "Active"
        }
        """
    Then the response status code should be 400
    And the response should contain validation error "Cannot reopen a closed entity"
```

## Best Practices

1. **Descriptive Scenario Names**: Make it clear what's being tested and what the expected outcome is
2. **Minimal Background**: Only include Background data needed by ALL scenarios
3. **Independent Scenarios**: Each scenario should work standalone
4. **Clear Assertions**: Be specific about expected results
5. **Test Client Isolation**: Always verify multi-tenancy works correctly
6. **Test Soft Deletes**: Verify IsDeleted behavior in all CRUD operations
7. **Test Validation**: Cover all validation rules and error cases
8. **Test OData**: If endpoint supports OData, test $filter, $orderby, $select, etc.

## Anti-Patterns to Avoid

❌ **Don't**: Combine multiple operations in one scenario
❌ **Don't**: Test multiple endpoints in one scenario
❌ **Don't**: Use complex logic in scenarios - keep them simple
❌ **Don't**: Make scenarios dependent on each other
❌ **Don't**: Test log messages or logging behavior
❌ **Don't**: Forget to test negative cases
❌ **Don't**: Skip multi-tenancy tests

✅ **Do**: One scenario per operation/edge case
✅ **Do**: Test one endpoint per feature file (organize by controller/operation)
✅ **Do**: Keep scenarios simple and readable
✅ **Do**: Make scenarios independent
✅ **Do**: Test happy path AND error cases
✅ **Do**: Always test client data isolation

## Example: Complete Feature File

```gherkin
Feature: Entity - Get
    As an API consumer
    I want to retrieve entities by ID
    So that I can view entity details

Background:
    Given I am authenticated as a user with the following details:
        | UserId | ClientId | UserGuid                             | ClientGuid                           |
        | 1      | 1        | face1e55-deb5-50d5-f1ed-0ddba1150ff5 | 294a5c5f-7d35-43d7-a45d-83668bd6babb |

Scenario: Get entity - existing entity - returns entity
    Given I have the following records in the kyc 'Entity' table
        | Id                                   | ClientId                             | Name        | Status  | IsDeleted |
        | 550e8400-e29b-41d4-a716-446655440000 | 294a5c5f-7d35-43d7-a45d-83668bd6babb | Test Entity | Active  | false     |
    When I send a GET request to "/api/v3/entity/550e8400-e29b-41d4-a716-446655440000"
    Then the response status code should be 200
    And the response should contain the following JSON:
        """
        {
            "id": "550e8400-e29b-41d4-a716-446655440000",
            "name": "Test Entity",
            "status": "Active"
        }
        """

Scenario: Get entity - non-existent entity - returns 404
    When I send a GET request to "/api/v3/entity/550e8400-e29b-41d4-a716-446655440099"
    Then the response status code should be 404

Scenario: Get entity - deleted entity - returns 404
    Given I have the following records in the kyc 'Entity' table
        | Id                                   | ClientId                             | Name            | IsDeleted |
        | 550e8400-e29b-41d4-a716-446655440000 | 294a5c5f-7d35-43d7-a45d-83668bd6babb | Deleted Entity  | true      |
    When I send a GET request to "/api/v3/entity/550e8400-e29b-41d4-a716-446655440000"
    Then the response status code should be 404

Scenario: Get entity - from different client - returns 404
    Given I have the following records in the kyc 'Entity' table
        | Id                                   | ClientId                             | Name              | IsDeleted |
        | 550e8400-e29b-41d4-a716-446655440000 | 9f1c2a10-2b6d-4f6b-9c3e-8d5f7b6a1c2d | Other Client Data | false     |
    When I send a GET request to "/api/v3/entity/550e8400-e29b-41d4-a716-446655440000"
    Then the response status code should be 404
```
