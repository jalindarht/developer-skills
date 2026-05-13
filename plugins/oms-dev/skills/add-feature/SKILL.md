---
name: add-feature
description: Scaffold a complete new feature end-to-end — entity, DTO, service, specification, EF configuration, API endpoints, route constants, and DI registration
user_invocable: true
---

# Add Complete Feature

Scaffold all layers for a new feature following the project's Clean Architecture. This orchestrates all the individual skills together.

## Step 1: Gather Feature Requirements

Ask the user for the complete feature description:
1. **Feature name** (e.g., "Coupon Management", "Loyalty Points")
2. **Domain entities** needed (with properties and relationships)
3. **Which entity is the aggregate root?**
4. **API operations** needed (CRUD + custom operations)
5. **Database schema** to use (store, sale, menu, management, or new)
6. **Authorization** — which endpoints are public vs protected?

## Step 2: Plan the Feature

Before writing any code, present a plan to the user listing all files to be created:

```
Feature: [Name]
Schema: [schema]

Files to create:
1. Entity:        Core/Entities/[Entity].cs
2. DTO:           Core/DTO/[Dto].cs
3. Service:       Core/Services/[Feature]/[Service].cs (interface + implementation)
4. Specification: Core/Entities/Specifications/[AggregateRoot]/[Specs].cs
5. EF Config:     Infrastructure/Data/Configurations/[Entity]Config.cs
6. Endpoints:     CentralAPI/Endpoints/[Feature]/[Endpoint].cs + .Request.cs (per operation)

Files to modify:
7. Routes:        CentralAPI/Constants/ApiEndPoints.cs
8. DI:            Core/CoreExtensions.cs
9. DbContext:     Infrastructure/Data/AppDbContext.cs
```

**Wait for user approval before proceeding.**

## Step 3: Create in Order

Execute in this order (respecting layer dependencies):

1. **Entity** (`/add-entity` pattern)
2. **DTO** (`/add-dto` pattern)
3. **EF Configuration** (`/add-ef-configuration` pattern)
4. **DbSet** in AppDbContext
5. **Specification(s)** (`/add-specification` pattern)
6. **Service interface + implementation** (`/add-service` pattern)
7. **DI registration** in CoreExtensions.cs
8. **Route constants** in ApiEndPoints.cs
9. **Endpoint(s) + Request(s)** (`/add-endpoint` pattern)

## Step 4: Architecture Validation

After creating all files, run mental `/architecture-review`:

- [ ] Layer boundaries respected (Core has no upward dependencies)
- [ ] All DI registrations in place
- [ ] All files follow naming conventions
- [ ] Aggregate root pattern correct
- [ ] Soft delete configured
- [ ] All endpoints have Swagger annotations
- [ ] All endpoints have authorization
- [ ] Service returns Result<T> throughout
- [ ] Specifications used for queries

## Step 5: Ask About Standards

If during implementation you notice the feature could benefit from patterns NOT yet in the project (e.g., FluentValidation, MediatR, unit tests), ask:

> "The current project doesn't have [pattern/standard]. For this feature, it would be beneficial because [reason]. Would you like me to introduce it, or should I follow the existing conventions?"

Always default to existing conventions unless the user explicitly wants to introduce new patterns.
