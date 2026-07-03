---
name: "zoran"
version: "4.7"
runtime: "required .NET 10 / C# 14 / EF Core 10 / self-hosted Microsoft SQL Server 2022"
status: "production baseline — fixed SQL Server 2022 compatibility level 160"
summary: "Generate clear C# for domain-heavy applications: explicit domain concepts, immutable values, behaviour-led aggregates, deliberate boundaries, and EF Core mappings that respect the model."
---

> **First run only — verify, then delete this block.**
>
> Run the following against the target database and confirm that `compatibility_level` is `160`:
>
> ```sql
> SELECT name, compatibility_level
> FROM sys.databases
> WHERE name = DB_NAME();
> ```
>
> This skill assumes self-hosted SQL Server 2022 at compatibility level 160.

# Zoran-Inspired C# Domain-Modelling Skill

Generate C# that makes important business rules explicit. Prefer named domain concepts, immutable values, small cohesive aggregates, explicit application boundaries, and behaviour over data flags.

This is an inspiration guide, not a claim that every rule is Zoran Horvat's personal practice or that one architecture suits every application. It favours correctness and clarity in domain-heavy systems over ceremony, novelty, and stylistic purity.

## 1. Fixed platform and decision order

Generate for **.NET 10, C# 14, EF Core 10, and self-hosted SQL Server 2022 at compatibility level 160**. Use the modern features available in this baseline when they improve clarity. Do not generate lower-version alternatives.

Configure the SQL Server provider at the fixed level:

```csharp
options.UseSqlServer(
    connectionString,
    sql => sql.UseCompatibilityLevel(160));
```

### Production safety gate

A feature marked preview in the applicable Microsoft documentation is never a default in production-targeted code. Introduce it only when the task explicitly confirms feature-specific operational approval. Do not infer approval from the fixed platform baseline, compatibility level, or the presence of related APIs.

### Dependency gate

When changing packages, use the current stable version compatible with the fixed platform. Stay on EF Core 10, avoid prereleases without explicit approval, keep direct `Microsoft.EntityFrameworkCore.*` references version-aligned, verify extension compatibility, and do not upgrade packages incidentally.

Apply rules in this order:

1. Explicit user requirements, security requirements, and operational constraints. A preview feature still requires explicit feature-specific operational approval.
2. Correctness, data integrity, clear public contracts, and observability.
3. The established conventions in the code being changed.
4. This skill.

Do not invent business rules. Represent a rule only when the task, tests, schema, surrounding code, or an explicit requirement establishes it.

When repository files, database configuration, or tests are not available, state assumptions. Never claim to have inspected or executed anything that was not available.

## 2. Core principles

### Scope and proportionality

Use this skill proportionately. For a CRUD-heavy feature or read-side projection with no material domain invariant, state transition, historical commitment, or cross-record consistency rule, prefer straightforward entities, DTO projections, and local validation; do not introduce value-object identifiers, aggregates, `ServiceResult<T>`, or separate configuration classes merely to satisfy a pattern. Retain the platform, security, data-integrity, boundary, and testing rules that remain relevant.

Escalate to richer domain modelling when named concepts prevent material mistakes, behaviour must preserve an invariant, a business transition needs an explicit policy, or historical terms and concurrent updates matter.

### 2.1 Give important concepts names

Use a named domain type when a primitive carries a meaningful invariant, unit, currency, time-zone interpretation, lifecycle, or material risk of accidental interchange.

Good candidates include identifiers, money, percentages, quantities, ISO codes, time periods, account references, and externally visible handles.

Do not wrap every primitive mechanically. Primitives remain suitable for:

- transport DTO fields and JSON payloads;
- private implementation locals with no independent domain meaning;
- database projections used only for reading;
- simple values whose name and validation add no useful distinction.

A public HTTP request may carry `Guid accountId`; the application boundary should convert it to `AccountId` before domain behaviour runs.

### 2.2 Enforce invariants at construction and change

Required domain values must be constructed through code that enforces their invariant. Do not expose a post-construction `IsValid`, `Validate`, or `Verify` ritual that callers can forget.

Use exceptions for programmer errors or impossible/corrupted state. At a user-facing boundary, translate invalid input into a typed rejection with a useful code and message.

### 2.3 Values are immutable; aggregates control mutation

Use immutable value objects. Use sealed classes for EF-tracked aggregate roots and child entities; expose controlled mutation methods and private setters.

Do not use records as ordinary EF-tracked entities. Record value equality is appropriate for values; EF Core admits only one CLR instance for a given key and requires entity-navigation collections to use reference equality. A record's value equality can therefore collapse distinct entity instances in an equality-based navigation collection and conflicts with EF's entity-identity model. Use a sealed class for an entity and a sealed record for an immutable value.

### 2.4 Keep boundaries explicit

Keep HTTP, JSON, EF Core, logging, dependency injection, and external I/O out of the domain model. Application services coordinate those concerns; domain objects decide business rules.

Public contracts expose DTOs and public result types, not aggregate types or value objects. A domain type may be `public` because Infrastructure needs to map it across assemblies; CLR accessibility does not make it a transport contract.

### 2.5 Prefer the smallest faithful model

An enum is suitable when state is merely a label. Use a state object or hierarchy only when variants carry distinct data, behaviour, transitions, or capabilities. Do not manufacture a hierarchy merely to avoid an enum.

## 3. Layer responsibilities

| Layer | Owns | Must not expose or contain |
|---|---|---|
| Transport | HTTP DTOs, JSON shape, status codes, authentication context, OpenAPI metadata | EF entities or domain types as wire contracts |
| Application | Use-case orchestration, authorisation decisions, loading aggregates, result mapping | Duplicated business rules or direct JSON policy |
| Domain | Value objects, aggregates, business rules, in-memory domain events | `DbContext`, controllers, service location, HTTP, JSON attributes |
| Infrastructure | EF Core mapping, focused data access ports, external clients, durable message delivery | Business decisions that belong to an aggregate |

A feature may be a class library with a small public surface, but do not force every feature into exactly one or two interfaces. Expose only the contracts a consumer genuinely needs.

## 4. Value objects

### 4.1 Default shape

For a simple non-nullable identifier that wraps a primitive or `Guid`, use one validating constructor. Do not add `TryCreate`, parsing APIs, persistence-only factories, or a separate EF converter class unless the type has genuinely more complex validation or repeated mapping needs.

Use a sealed record with explicit get-only properties. Avoid positional records when a property needs validation, normalisation, or a different public name.

```csharp
using System;

namespace Payments.Domain;

public sealed record AccountId
{
    public Guid Value { get; }

    public AccountId(Guid value)
    {
        ArgumentOutOfRangeException.ThrowIfEqual(value, Guid.Empty);
        Value = value;
    }

    public static AccountId New() => new(Guid.NewGuid());
}
```

Do not add a `FromStorage` method. EF Core conversion must construct the same valid domain object as every other caller. Corrupt persisted data should fail the value object's ordinary invariant rather than enter the domain through a weaker path.

Do not use a constrained `record struct` for an identifier or other value object whose default value is invalid: `default(T)` bypasses constructor validation. Use a sealed record class unless profiling establishes a material need for a struct.

### 4.2 Protect record integrity

A `with` expression copies a record and then applies the requested `init` or `set` changes. Therefore, a public `init` property can bypass cross-field validation performed by a constructor.

For a value object with one or more invariants:

- expose get-only properties;
- validate the complete proposed state in the constructor;
- provide a named replacement method when replacement is meaningful;
- do not rely on a later `Validate` or `Verify` call.

```csharp
using System;

namespace Scheduling.Domain;

public sealed record DateRange
{
    public DateOnly StartsOn { get; }
    public DateOnly EndsOn { get; }

    public DateRange(DateOnly startsOn, DateOnly endsOn)
    {
        if (endsOn < startsOn)
        {
            throw new ArgumentException(
                "End date cannot precede start date.",
                nameof(endsOn));
        }

        StartsOn = startsOn;
        EndsOn = endsOn;
    }

    public DateRange MoveTo(DateOnly startsOn, DateOnly endsOn) =>
        new(startsOn, endsOn);
}
```

`range with { EndsOn = ... }` cannot alter either property because neither has an `init` or `set` accessor. A positional record is acceptable only when it has no validation, normalisation, or cross-field invariant.

### 4.3 Money, time, identifiers, and calendars

- Model money as an amount plus currency. Do not pass bare `decimal` amounts across a meaningful domain boundary. A general `Money` type should not acquire a non-negative rule, a fixed scale, or an invoice-only rounding rule unless every valid use shares it; credits, refunds, adjustments, prices, and settlement amounts may require different constraints. Put those constraints on the relevant operation or a more specific type.
- A three-character currency string is not proof that a value is an ISO 4217 code. Validate only the syntax you can prove, or validate membership against a deliberately versioned approved-code source when that is a requirement. Do not describe a length-only check as ISO validation.
- Use `DateOnly` for dates without a time or zone and `TimeOnly` for a time-of-day. Use `DateTimeOffset` for an instant; normalise to UTC where that is a domain invariant.
- Use `TimeProvider` or a timestamp supplied by the application service for current time when deterministic testing or auditability matters.
- Use `ISOWeek` only where the business specification actually uses ISO 8601 weeks. Do not substitute ISO rules for fiscal or locale-specific calendars.
- Represent absence at a boundary with an optional value. Do not use an `Empty` sentinel inside the domain.
- Avoid implicit conversions between a value object and its primitive. Make conversion explicit in mappings and adapters.

### 4.4 Construction guidance

Use the constructor for a simple non-nullable value object. Use `TryParse` or `TryCreate` only for genuinely fallible parsing or conversion, particularly when a caller needs to avoid exception flow. Use a typed result when expected rejection needs a code, message, or field association.

For one or two validations, use clear guard clauses. Do not introduce a custom functional pipeline merely to make validation look abstract.

## 5. Aggregates and entities

Use a sealed class for an EF-tracked aggregate root or entity. The aggregate owns state transitions and mutations. Application services load the aggregate, invoke behaviour, persist it, and translate the outcome.

```csharp
using System;

namespace Subscriptions.Domain;

public sealed record SubscriptionId
{
    public Guid Value { get; }

    public SubscriptionId(Guid value)
    {
        ArgumentOutOfRangeException.ThrowIfEqual(value, Guid.Empty);
        Value = value;
    }

    public static SubscriptionId New() => new(Guid.NewGuid());
}

public enum SubscriptionStatus
{
    Active,
    Cancelled
}

public sealed class Subscription
{
    private Subscription() { }

    public Subscription(SubscriptionId id, DateTimeOffset startedAt)
    {
        ArgumentNullException.ThrowIfNull(id);

        Id = id;
        StartedAt = startedAt;
        Status = SubscriptionStatus.Active;
    }

    public SubscriptionId Id { get; private set; } = null!;
    public DateTimeOffset StartedAt { get; private set; }
    public SubscriptionStatus Status { get; private set; }
    public DateTimeOffset? CancelledAt { get; private set; }

    public void Cancel(DateTimeOffset cancelledAt)
    {
        if (cancelledAt < StartedAt)
        {
            throw new ArgumentOutOfRangeException(
                nameof(cancelledAt),
                "Cancellation cannot precede the subscription start.");
        }

        if (Status is SubscriptionStatus.Cancelled)
            return; // Explicit replay policy after input validation.

        Status = SubscriptionStatus.Cancelled;
        CancelledAt = cancelledAt;
    }
}
```

### 5.1 Relationship references and navigations

EF Core relationships are defined by foreign keys; navigation properties are optional. A `Subscription` that does not need to traverse a `Customer` or `SubscriptionPlan` can retain only its value-object foreign keys and configure the relationship explicitly:

```csharp
builder.HasOne<Customer>()
    .WithMany()
    .HasForeignKey(subscription => subscription.CustomerId)
    .IsRequired();

builder.HasOne<SubscriptionPlan>()
    .WithMany()
    .HasForeignKey(subscription => subscription.PlanId)
    .IsRequired();
```

Add a private-set navigation only when the aggregate must traverse or deliberately manipulate that relationship in memory, or when callers need `Include`/`ThenInclude` for that navigation:

```csharp
public Customer Customer { get; private set; } = null!;
public SubscriptionPlan Plan { get; private set; } = null!;
```

Navigations are a persistence-facing convenience, not a prerequisite for `HasOne<TPrincipal>().WithMany().HasForeignKey(...)`. Keep them out of domain behaviour unless traversal itself is part of a business rule. A foreign-key-only relationship cannot be traversed with `Include`, and relationship fix-up has no navigation property to update.

When a reference points to an aggregate in another bounded context or external service, keep it as a scalar value-object identifier and do not configure an EF relationship to a principal that is absent from the model. Its existence and authorisation are application-boundary concerns.

Use a private `List<T>` and expose a read-only view for child entities. Do not return the mutable list itself. EF Core can use a conventionally named backing field for a read-only collection navigation, but cover materialisation and relationship fix-up with an integration test. Use `private set` for state changed by aggregate methods. Do not create public setters merely to satisfy persistence.

An aggregate root is a write boundary, even when a child has its own key and an ordinary EF entity mapping. Load and mutate a child through its root for commands; a child table may still be queried directly for a read projection. A public `DbSet<TChild>` is not a licence for application commands to bypass aggregate behaviour.

Use a domain foreign-key property when the child must reason about, report on, authorise against, or correlate its parent. A shadow foreign key is suitable only when that association is purely persistence metadata. When using one deliberately, configure its type, conversion, index, and column name explicitly rather than relying on a string literal and convention.

Each behaviour method must validate the whole proposed state before mutation where fields constrain one another. Use named methods that describe the decision or transition; do not expose generic `Set...` methods for business state. A temporal transition must receive the facts needed to prove it is valid—such as an assessment time, due date, billing boundary, or prior event—and validate chronological order and permitted status. Do not provide `MarkOverdue()` or `SetCurrentPeriodEnd(...)` merely because a caller needs a setter when the actual policy has not been specified.

Whether a repeated command is a no-op, rejection, or a separate auditable action is a business decision. State that policy explicitly.

### 5.2 Historical terms and commitments

When an aggregate creates a commitment whose terms must remain historically true, model those terms deliberately: reference an immutable or versioned record, or store an immutable snapshot. Typical terms include price, tariff, tax treatment, billing cadence, entitlement, exchange rate, and contractual wording.

When an aggregate creates a commitment, decide explicitly whether it refers to current reference data or preserves a versioned or snapshotted set of terms. A completed or issued business record must retain the values, conditions, and authority that were effective when it was created. Where an exceptional adjustment is permitted, model it as a separately named and authorised operation with its reason or source; do not admit arbitrary changes through an ordinary command.

Apply this only where the business meaning is a commitment or historical fact rather than a live view of current reference data.

### 5.3 C# 14 `field`-backed properties

Use the C# 14 contextual keyword `field` when one property needs simple validation or normalisation at assignment and a named backing field would add no clarity. It is appropriate for a local, single-property rule; cross-property invariants and state transitions belong in constructors or aggregate methods.

```csharp
using System;

public sealed class Customer
{
    public Customer(string customerReference) =>
        CustomerReference = customerReference;

    public string CustomerReference
    {
        get;
        private set => field = RequireCustomerReference(value);
    }

    private static string RequireCustomerReference(string value) =>
        !string.IsNullOrWhiteSpace(value)
            ? value.Trim()
            : throw new ArgumentException(
                "Customer reference cannot be empty.",
                nameof(value));
}
```

Do not use `field` merely because it is available. Avoid a member named `field` in a type that uses field-backed properties.

## 6. State modelling and pattern matching

| Situation | Preferred representation |
|---|---|
| Simple label with no variant-specific fields or behaviour | `enum` |
| Finite states with variant-specific data or transitions | abstract base type with sealed variants |
| Rules that vary independently of stored state | strategy or policy interface |
| Boolean flags that create invalid combinations | richer type, enum, or state hierarchy |

C# 14 does not provide compiler-enforced closed discriminated unions. A base record with sealed variants is a convention, not proof that no other subtype exists. For an important switch, include an explicit unexpected-case arm and test every known variant.

Use switch expressions, tuple patterns, property patterns, and guards when they make a decision table clearer. Do not replace a short guard clause with a switch merely to use pattern matching.

## 7. Public contracts and application services

Public interfaces must mention only public types. Transport DTOs may use primitives because they are boundary representations. Map and validate them immediately before domain behaviour begins. Do not serialise aggregates or internal domain states directly.

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

namespace Transfers.Contracts;

public sealed record RequestTransferDto(
    Guid FromAccountId,
    Guid ToAccountId,
    decimal Amount,
    string CurrencyCode);

public sealed record TransferDto(
    Guid TransferId,
    Guid FromAccountId,
    Guid ToAccountId,
    decimal Amount,
    string CurrencyCode,
    DateTimeOffset RequestedAt);

public sealed record ProblemDto(string Code, string Message);

public abstract record ServiceResult<T>
{
    private ServiceResult() { }

    public sealed record Success(T Value) : ServiceResult<T>;
    public sealed record Rejected(ProblemDto Problem) : ServiceResult<T>;
    public sealed record Conflict(ProblemDto Problem) : ServiceResult<T>;
}

public interface IMoneyTransferService
{
    Task<ServiceResult<TransferDto>> RequestAsync(
        RequestTransferDto request,
        CancellationToken cancellationToken = default);
}
```

At an application boundary:

1. validate transport shape and authorisation context;
2. convert primitives to domain types;
3. load other aggregates only when their current state is needed for the rule;
4. invoke domain behaviour;
5. persist within the appropriate unit of work;
6. map to a stable public result.

An interface, DTOs, and entity mappings are not a completed use case. Unless the task explicitly asks for contracts only, implement the command handler or application service, its persistence boundary, and focused tests. Do not leave a public command contract with no code that validates input, loads state, invokes the aggregate, saves, and maps the outcome.

Return `Rejected` for expected business-rule failures and `Conflict` for an expected stale-write outcome. Catch `DbUpdateConcurrencyException` only to produce a meaningful conflict result. A unique-key violation is provider-specific and is not a `DbUpdateConcurrencyException`; where a unique rule is part of the public contract, name the database constraint or index deterministically and translate only that known violation to the appropriate `Conflict` or `Rejected` result. Do not turn every `DbUpdateException` into a client error. Let database outages, timeouts, programming defects, and other unexpected technical faults reach the host exception boundary.

A SQL Server `rowversion` detects stale tracked writes, but it is not automatically a client-side conditional-update protocol. Where clients edit a representation and stale overwrites matter, deliberately expose and require an opaque revision or HTTP ETag, or state why server-side conflict detection alone is sufficient.

Log technical faults, audit events, and security-relevant decisions through the host's structured logging policy. Do not log secrets or treat every expected rejection as an error.

## 8. HTTP, validation, OpenAPI, and JSON

Write small mappings explicitly. A local mapping method should construct every DTO field deliberately rather than expose a domain object or rely on convention-heavy automatic mapping. Do not introduce a mapping library by default.

Use `System.Text.Json` unless an established project requirement needs another serializer. Configure JSON policy at the transport edge.

### 8.1 Minimal API validation

For a .NET 10 Minimal API that uses DataAnnotations for request-shape rules, reference `Microsoft.Extensions.Validation` and call `AddValidation()`.

```csharp
builder.Services.AddValidation();
```

Use it for required fields, length, syntax, and simple ranges on transport DTOs. Keep business validation in domain construction and aggregate behaviour. Do not put DataAnnotations on domain models merely to make an endpoint reject a request.

### 8.2 Strict JSON

For a controlled inbound JSON contract, prefer `JsonSerializerOptions.Strict`. It rejects duplicate and unmapped members, honours nullable annotations and required constructor parameters, and uses case-sensitive property matching.

`JsonSerializerOptions.Strict` is a shared read-only preset. Create a copy before customising it. Do not place JSON attributes or converters on internal domain types merely to expose them accidentally.

Use a JSON converter for a value object only when that value object is intentionally a public wire type. Otherwise use a transport DTO with primitives and map at the boundary.

### 8.3 OpenAPI

When built-in OpenAPI generation is enabled in ASP.NET Core .NET 10, generated documents default to OpenAPI 3.1. Treat the document as a public contract: choose its version deliberately and test meaningful request, response, and status-code changes.

## 9. EF Core 10 mapping

### 9.1 Persistence boundary

Keep EF Core mapping outside the domain model. Prefer Fluent API for domain models that should not reference EF Core.

For a production model, use one `IEntityTypeConfiguration<T>` class per aggregate root or ordinary entity by default. Keep `DbContext.OnModelCreating` as a small registration point:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder) =>
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(BillingDbContext).Assembly);
```

Configure owned and complex members inside the configuration class for their owner. An owned type is scoped to its ownership path, so it is not generally an independently registered `IEntityTypeConfiguration<T>`.

Inline `OnModelCreating` code remains reasonable for a genuinely tiny model or a focused documentation example, but should not be the production default.

Configure the provider, connection string, and `UseCompatibilityLevel(160)` in the composition root, normally through `AddDbContext`. The `UseSqlServer` overload without a connection string is valid for a context whose connection is supplied later, but it is not a normal replacement for host configuration: a connection must be set before the context uses the database. Do not put a placeholder provider configuration in `OnConfiguring` and imply that the context is runnable on its own.

Do not wrap `DbContext` in a generic repository that only re-exposes `DbSet` methods. Use `DbContext` directly in straightforward application services. Add a focused repository or query gateway only when it creates a real boundary, such as a provider-independent read model, a complex query policy, or an external persistence port.

### 9.2 Choose the appropriate mapping

| Domain shape | EF Core mapping default |
|---|---|
| One domain value stored in one column | `ValueConverter` |
| Small identity-less value object stored with its owner, with no navigation | `ComplexProperty` |
| Ownership needing a collection/table, relationship, navigation, or entity semantics | owned entity mapping |
| Object with independent identity or lifecycle | entity type |

`ComplexProperty` is not a universal replacement for owned entities. Complex types have no identity and do not support navigations. A complex type mapped to JSON can contain a structural collection, but that collection shares the JSON document's lifecycle; do not use it to simulate relationships.

**Nested-owned limitation.** In EF Core 10, `ComplexProperty` is exposed by `EntityTypeBuilder<T>`, not by the `OwnedNavigationBuilder<TOwner, TDependent>` supplied to an `OwnsOne` or `OwnsMany` callback. A complex type therefore cannot be configured beneath an owned entity with that Fluent API. The limitation is unrelated to records, get-only properties, `IReadOnlyCollection<T>`, or record value equality. An explicit generic `OwnsMany` overload still supplies an `OwnedNavigationBuilder`; it does not alter this limitation.

When a value beneath an owned entity would otherwise be a complex property, choose deliberately:

| Requirement | Mapping choice | Consequence |
|---|---|---|
| Keep the line owned and store the nested value in relational columns | Use `OwnsOne` for the nested value | The nested value becomes an owned entity in EF's model, with ownership identity and entity-style tracking. It normally remains table-split into the line table. |
| Keep the nested value as a true EF complex property | Map the containing line as a normal dependent entity, then configure `ComplexProperty` on its `EntityTypeBuilder<InvoiceLine>` | The line now has ordinary entity and relationship semantics; the relational mapping change is intentional. |
| Treat the nested value as one self-contained persisted unit | Redesign it as a single-column converted value or self-contained JSON document | This can weaken relational querying and database constraints; use it only where the component's lifecycle and query needs fit. |

The first option is usually the smallest change when the containing line is already owned. For example:

```csharp
builder.OwnsMany(invoice => invoice.Lines, line =>
{
    line.OwnsOne(invoiceLine => invoiceLine.Amount, money =>
    {
        money.Property(value => value.Amount)
            .HasColumnName("LineAmount")
            .HasPrecision(19, 4);

        money.Property(value => value.Currency)
            .HasColumnName("LineCurrency")
            .HasMaxLength(3);
    });
});
```

This makes `Money` an owned entity in EF's model. Cover the mapping and replacement behaviour with an integration test. Choose the normal-entity or converted/document alternative only where its semantic and operational trade-offs fit the domain.

Choose explicit column names that are unique and intelligible within the physical table. `TotalAmount` is not invalid merely because it matches its parent complex property name: the parent is not itself a scalar column. However, `TotalAmount_Amount` or another distinctive name usually makes the schema easier to read. In EF Core 10, accidental duplicate complex-property column names are uniquified rather than silently sharing a column; intentionally shared columns require explicit configuration. Do not rely on generated suffixes as a naming convention.

Use optional complex types only when absence is meaningful. For an optional complex type flattened into relational columns, EF Core must distinguish absence from an all-null instance: give it at least one required nested property, or configure a persistence-only shadow discriminator. Prefer a required nested property where it expresses a real invariant.

A non-nullable complex property represents a required persisted value, not a licence to invent a zero-value merely to silence nullable warnings. For a calculated value such as `TotalAmount`, prefer deriving it from the aggregate's authoritative state and leaving it unmapped. If it must be persisted as a denormalised value, assign it in every public constructor and state-transition method that establishes or changes the inputs. `= null!;` is acceptable only for EF Core's private materialisation path; it is not evidence that the domain invariant has been established. Initialise with `new Money(0, Currency.USD)` only when that exact value is unambiguously valid for every new aggregate instance.

### 9.3 Persisted entity inheritance

Persisted inheritance is exceptional, not a default. Use it only for a closed entity hierarchy whose concrete types have genuinely distinct persisted state and must be loaded or related to polymorphically.

TPH is EF Core's default inheritance mapping. When it is required, configure an explicit shadow discriminator with stable storage codes. Do not duplicate that code as a mutable domain property. Do not call `UseTphMappingStrategy()` merely to restate EF Core's default.

```csharp
builder.HasDiscriminator<string>("ProductKind")
    .HasValue<Material>("material")
    .HasValue<Service>("service");
```

This entity-hierarchy discriminator is distinct from the technical shadow discriminator that can make an optional flattened complex type representable. Test persistence and base-type materialisation for every mapped concrete subtype.

### 9.4 Scalar value objects

Map a scalar value object with an inline converter inside its `IEntityTypeConfiguration<T>` by default:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Subscriptions.Domain;

namespace Subscriptions.Infrastructure;

internal sealed class SubscriptionConfiguration
    : IEntityTypeConfiguration<Subscription>
{
    public void Configure(EntityTypeBuilder<Subscription> builder)
    {
        builder.HasKey(subscription => subscription.Id);

        builder.Property(subscription => subscription.Id)
            .HasConversion(
                id => id.Value,
                value => new SubscriptionId(value))
            .HasColumnType("uniqueidentifier");

        builder.Property(subscription => subscription.Status)
            .HasConversion<string>()
            .HasMaxLength(16);

        builder.Property<byte[]>("Version")
            .IsRowVersion();
    }
}
```

Use the domain `Id` as the primary key by default. Do not add a separate shadow surrogate key or rename `Id` to `PublicId` unless the domain or physical model requires distinct identities.

Create a dedicated `ValueConverter` class only when the same conversion is reused, it needs mapping hints, or the conversion itself is substantial. A converter must call the value object's normal validating constructor; do not create persistence-only construction paths.

EF Core can bind a suitable parameterised constructor to mapped scalar properties. Use a private parameterless constructor only when constructor binding is not suitable for the mapped entity. Do not add empty constructors and public setters by reflex.

An immutable converted class with correct value equality normally needs no custom `ValueComparer`. A mutable converted reference value or collection needs deliberate equality and snapshot semantics.

### 9.5 Keys, relationships, nullability, and decimals

Use a unique index when a value must only be unique. Use an alternate key, with `HasPrincipalKey` where needed, when a dependent relationship must reference that business key. A relationship's principal key must be a primary or alternate key; a unique index alone is not sufficient.

Configure required uniqueness and referential integrity deliberately. Application pre-checks do not prevent races. When a unique rule is user-visible, give the index or constraint a stable database name and decide how the application maps its known violation.

Choose `DeleteBehavior` deliberately for every relationship whose dependents have material business, audit, or retention value. Required relationships conventionally cascade; do not let that default delete invoices, payments, ledger entries, or audit records merely because they are dependent in the object graph. Use `Restrict` or `NoAction`, plus an explicit domain lifecycle such as voiding or archival, unless deletion is an established retention rule. Test the chosen delete behaviour against SQL Server.

Keep nullable reference types enabled. C# nullability affects EF Core's required/optional conventions, so review resulting warnings and column nullability whenever a property changes.

Persist economically meaningful decimals—money, rates, quantities, meter readings, and settlement values—with an explicit SQL Server precision and scale chosen from the required range and resolution. `HasPrecision` does not replace domain-side range, rounding, or currency-minor-unit rules.

For read-only queries, project directly to DTOs and use `AsNoTracking()` where change tracking is unnecessary. Avoid loading an aggregate merely to compute a view model.

### 9.6 Context lifetime, concurrency, and external delivery

Use `AddDbContext` for the normal request-scoped unit of work. Use `IDbContextFactory<TContext>` when a background process or component needs independent short-lived contexts. Never share a `DbContext` concurrently.

Do not introduce context pooling as a default optimisation. Pooled contexts are reused, so tenant, request, and manually changed ADO.NET connection state must be reset correctly before reuse.

Pass `CancellationToken` through EF Core and other I/O calls.

One `SaveChangesAsync()` call on a relational provider is transactional by default. Create an explicit transaction only when a unit of work spans multiple saves, raw SQL, or other database operations that must succeed or fail together. A retry policy does not replace idempotency or a coherent transaction boundary.

Where lost updates matter, configure optimistic concurrency and translate `DbUpdateConcurrencyException` into `ServiceResult<T>.Conflict`. Do not silently retry a business command unless the command is demonstrably idempotent.

Raise domain events in memory as part of the domain operation. Publish external messages only after durable persistence succeeds. Use an outbox when reliable delivery across the database/message boundary matters.

## 10. EF Core 10 query and update patterns

### 10.1 Read-side queries

For projections requiring optional related data, prefer .NET 10 `LeftJoin` over a hand-written `GroupJoin` plus `DefaultIfEmpty` sequence when it improves readability. Keep joins in read models, not aggregate mutation code.

Project only fields required by the caller, bound result sets, and use a pagination strategy stable for the access pattern. Avoid lazy loading by default: it obscures database round-trips and makes N+1 queries easy to introduce.

### 10.2 Named query filters

Use named EF Core 10 query filters when independent global concerns, such as soft deletion and multi-tenancy, must be separately disabled.

```csharp
modelBuilder.Entity<Invoice>()
    .HasQueryFilter("SoftDeletion", invoice => !invoice.IsDeleted)
    .HasQueryFilter("Tenant", invoice => invoice.TenantId == _tenantId);

var includingDeleted = await db.Invoices
    .IgnoreQueryFilters(["SoftDeletion"])
    .ToListAsync(cancellationToken);
```

A query filter is not the sole authorisation mechanism. Bypass a named filter only in an explicitly authorised application path.

### 10.3 Bulk updates

Use `ExecuteUpdateAsync` only when bypassing aggregate methods, ordinary tracking, and in-memory domain events is intentional. Require an explicit predicate, an audit decision, and a concurrency strategy where relevant. Do not use it for ordinary aggregate commands.

`ExecuteUpdateAsync` executes immediately and does not synchronise EF Core's change tracker. Do not mix it with tracked changes in the same context without reloading or clearing tracked entities. Multiple bulk operations are not atomic unless enclosed in an explicit transaction.

EF Core 10 supports `ExecuteUpdateAsync` within relational JSON mapped as a complex type. It does not provide the same support for owned-entity JSON mappings.

### 10.4 Raw SQL and parameterised collections

Prefer interpolated `FromSql` for values. Use `FromSqlRaw` only for trusted SQL fragments or rigorously whitelisted identifiers; never concatenate external input into SQL.

EF Core 10 translates parameterised collection queries such as `ids.Contains(entity.Id)` using multiple scalar parameters by default. Do not alter the translation mode without execution-plan evidence for the relevant workload.

## 11. SQL Server 2022 rules

### 11.1 Provider configuration

The fixed platform uses `UseCompatibilityLevel(160)`. The top-of-file verification is a reminder that server version and database compatibility level are separate settings.

Do not add `EnableRetryOnFailure()` automatically. It is a host resilience decision that depends on deployment topology, transaction boundaries, and command idempotency.

### 11.2 Compatibility-level scope

Compatibility level governs database-scoped Transact-SQL and query-processing behaviour; it does not make later engine features available. Generate only features supported by both self-hosted SQL Server 2022 and compatibility level 160.

At level 160, parameter-sensitive plan optimisation and cardinality-estimation feedback are enabled by default. Treat this as a reason to inspect Query Store and representative execution plans when performance matters, not as a reason to add speculative query hints or provider translation overrides.

### 11.3 JSON and time

SQL Server 2022 provides textual JSON support at this level. JSON documents are stored in `nvarchar` or `varchar` columns and can be validated, queried, transformed, and modified with `ISJSON`, `JSON_VALUE`, `JSON_QUERY`, `JSON_MODIFY`, and `OPENJSON`.

Under `UseCompatibilityLevel(160)`, EF Core 10 maps JSON-backed complex types, owned JSON components, and primitive collections to `nvarchar(max)`, not to a native `json` column. `ToJson()` is permissible only for a self-contained component that is normally read and written together and does not need relational joins, constraints, reporting, or an independent lifecycle. Test representative generated SQL, reads, writes, updates, and queries against the target database.

Do not generate later-engine JSON facilities, including the native `json` type, `CREATE JSON INDEX`, `JSON_CONTAINS`, JSON aggregate functions, the `JSON_VALUE ... RETURNING` form, or the `json` type's `modify()` method. For frequently filtered or sorted JSON paths, consider an indexed computed column based on `JSON_VALUE` only after execution-plan evidence demonstrates the need. Where JSON can be written outside EF Core's mapped model, enforce validity with an appropriate `ISJSON` check constraint or equivalent database-side control.

For a `datetime` or `datetime2` column whose documented meaning is a UTC instant, use a tested mapping policy that preserves the UTC convention on write and restores `DateTimeKind.Utc` on read. Do not use `DateTime.SpecifyKind` to relabel an unknown or local wall-clock value as UTC. Prefer `DateTimeOffset` for new instant-bearing fields.

## 12. Tests

Use the established test framework and assertion library. Do not impose a framework, mocking library, or test style merely because this skill mentions one.

When using MSTest 4, prefer `Assert.Throws` or `Assert.ThrowsExactly` and their async counterparts. Do not use `[ExpectedException]`, `Assert.ThrowsException`, or `Assert.ThrowsExceptionAsync`.

Test at the relevant levels:

1. **Domain tests**: value-object invariants, aggregate mutations, state transitions, and edge cases.
2. **Application tests**: rejection mapping, authorisation decisions, concurrency outcomes, clocks, and external ports.
3. **Infrastructure tests**: EF Core mappings, query translation, transactions, and the actual SQL Server provider where behaviour matters.

Instantiate aggregates in domain tests; do not mock their behaviour. Fake or mock external boundaries only when it makes a test narrower and clearer.

Use a test against the actual SQL Server 2022 compatibility-level-160 target for provider-specific mappings and important queries. For JSON, verify representative schema, generated SQL, writes, updates, and queries. For published HTTP APIs, test important request/response payloads, status codes, and OpenAPI output.

For an aggregate with converted keys, complex values, private collection fields, concurrency tokens, or non-default delete behaviour, include a provider integration test that builds the model, initialises an isolated SQL Server test schema, round-trips a fresh context, verifies relationship fix-up/materialisation, and proves the expected concurrency and deletion outcomes. Isolate write tests with transactions or independent databases; do not substitute an in-memory provider for SQL Server-specific mapping tests.

Test names should describe behaviour. Arrange/Act/Assert comments are optional; use them only when they make a long test clearer.

## 13. Organisation, accessibility, and C# 14 extensions

Prefer feature-oriented organisation. Use one significant public or domain type per file by default, but keep short related types together when that improves comprehension.

Use file-scoped namespaces in new code when that matches the repository. Avoid `Utils`, `Helpers`, and catch-all `Common` files. Name a type after the domain role it performs.

C# 14 extension blocks are optional. Use extensions for pure conversions, formatting, parsing adapters, and operations on types you do not own. Put aggregate creation, mutation, state transitions, and invariant enforcement on the owning type.

A public extension member must use only publicly accessible types in its signature. Keep an extension container `internal` when its target or supporting types are internal.

Use other C# 14 features—including null-conditional assignment and broader span conversions—only where they make one operation clearer or solve a measured hot-path problem.

## 14. Generation procedure

For each task:

1. Inspect the available code, contracts, tests, provider configuration, and conventions that the task touches.
2. Identify the business invariant or boundary involved. Do not create an abstraction without a concrete reason.
3. Choose the smallest fitting model: primitive, value object, enum, state object, entity, DTO, or projection.
4. Keep domain behaviour in the aggregate; keep I/O and orchestration in application or infrastructure code.
5. Use the fixed .NET 10 / C# 14 / EF Core 10 / SQL Server 2022 compatibility-level-160 baseline, with current stable EF Core 10 package references where a package change is required.
6. Write explicit mappings at HTTP, JSON, EF Core, and external-service boundaries; identify the composition root and test target for any persistent-model change.
7. Pass cancellation through asynchronous I/O.
8. Implement the relevant application command unless contracts only were explicitly requested; add or update focused tests that prove the important rule, mapping, or provider behaviour.
9. Re-read public signatures for accessibility, nullability, cancellation, accidental domain leakage, incomplete public command contracts, and any preview feature; require explicit operational approval before adding one.

## 15. Final checklist

Before returning code, verify only the items relevant to the change:

- [ ] The code targets .NET 10, C# 14, EF Core 10, and self-hosted SQL Server 2022 at compatibility level 160.
- [ ] Package changes use the current stable EF Core 10 servicing release; prerelease packages have explicit operational approval, direct EF Core package versions are identical, and extension packages are EF Core 10-compatible.
- [ ] SQL Server options configure `UseCompatibilityLevel(160)` in the composition root with a real connection configuration before database use.
- [ ] Every public signature uses only accessible public types.
- [ ] DTOs, entities, and value objects are not conflated.
- [ ] The model is proportionate to the feature; CRUD-only and read-side work has not acquired domain ceremony without a concrete invariant or policy.
- [ ] Domain invariants are enforced at construction and every state change.
- [ ] A constrained record has get-only properties; `with` cannot bypass a cross-field invariant.
- [ ] No `Empty` identifier sentinel, `FromStorage` factory, or post-construction validity ritual has been introduced.
- [ ] A scalar value object uses an inline converter unless shared reuse, mapping hints, or conversion complexity justifies a converter class.
- [ ] Entity mappings use `IEntityTypeConfiguration<T>` classes unless the model is genuinely trivial; owned and complex members remain configured under their owner.
- [ ] Relationship navigations are chosen deliberately: foreign-key-only mappings use generic `HasOne<T>()` or `HasMany<T>()` and parameterless `WithMany()` or `WithOne()`.
- [ ] Child entities remain inside their aggregate's command boundary; intentional shadow foreign keys are explicitly typed, converted, named, and covered by an integration test.
- [ ] Delete behaviour is explicit, and historical, financial, or audit dependents cannot be cascaded away without an established retention rule.
- [ ] A complex value nested beneath an owned entity uses an intentional owned-entity, converted-column, JSON, or normal-entity design; it does not call unavailable `ComplexProperty` APIs on `OwnedNavigationBuilder`.
- [ ] A persisted calculated complex value is established by constructors and state transitions; no fabricated default merely suppresses nullable warnings.
- [ ] A domain `Id` is the primary key unless distinct domain and storage identities are required.
- [ ] A unique index is not used where a relationship requires an alternate key.
- [ ] Entity inheritance, where genuinely needed, uses an explicit shadow discriminator with stable values; `UseTphMappingStrategy()` is not added merely to restate the default.
- [ ] Economically meaningful decimals have explicit, appropriate precision and scale.
- [ ] `DbContext` is short-lived and never shared concurrently.
- [ ] Expected business rejection, unique-constraint conflict, optimistic-concurrency conflict, and unexpected technical failure remain distinct.
- [ ] A public command has an implementation and focused tests unless contracts-only output was expressly requested.
- [ ] A client-facing update has an explicit concurrency policy (such as opaque revision/ETag or stated server-side handling) where stale overwrites matter.
- [ ] Bulk updates, query-filter bypasses, and raw SQL have explicit security, audit, transaction, and concurrency implications.
- [ ] JSON and validation concerns have not leaked into the domain merely for transport convenience.
- [ ] JSON mappings use `nvarchar(max)` storage; any JSON-path indexing uses an evidenced computed-column design, and later-engine JSON facilities are not generated.
- [ ] Tests cover the domain rule, mapping, or provider behaviour that matters.
- [ ] The response does not claim unperformed inspection, execution, or testing.

## 16. Official reference points

Use the current applicable Microsoft documentation when a framework or provider detail matters:

- C# 14 and `field`: https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-14
- C# records: https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/record
- C# record types and EF Core entity guidance: https://learn.microsoft.com/dotnet/csharp/fundamentals/types/records
- .NET 10 strict JSON: https://learn.microsoft.com/dotnet/core/whats-new/dotnet-10/libraries
- `JsonSerializerOptions.Strict`: https://learn.microsoft.com/dotnet/api/system.text.json.jsonserializeroptions.strict
- Minimal API validation: https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis
- ASP.NET Core OpenAPI: https://learn.microsoft.com/aspnet/core/fundamentals/openapi/aspnetcore-openapi
- EF Core 10 features and breaking changes: https://learn.microsoft.com/ef/core/what-is-new/ef-core-10.0/whatsnew
- EF Core constructors: https://learn.microsoft.com/ef/core/modeling/constructors
- EF Core value conversions: https://learn.microsoft.com/ef/core/modeling/value-conversions
- EF Core complex types: https://learn.microsoft.com/ef/core/modeling/complex-types
- EF Core owned entity types: https://learn.microsoft.com/ef/core/modeling/owned-entities
- EF Core relationship navigations: https://learn.microsoft.com/ef/core/modeling/relationships/navigations
- EF Core inheritance and discriminators: https://learn.microsoft.com/ef/core/modeling/inheritance
- EF Core keys and relationships: https://learn.microsoft.com/ef/core/modeling/relationships/foreign-and-principal-keys
- EF Core `DbContext` lifetime: https://learn.microsoft.com/ef/core/dbcontext-configuration
- EF Core SQL Server provider: https://learn.microsoft.com/ef/core/providers/sql-server/
- EF Core concurrency: https://learn.microsoft.com/ef/core/saving/concurrency
- EF Core transactions: https://learn.microsoft.com/ef/core/saving/transactions
- EF Core bulk updates: https://learn.microsoft.com/ef/core/saving/execute-insert-update-delete
- EF Core global query filters: https://learn.microsoft.com/ef/core/querying/filters
- EF Core testing: https://learn.microsoft.com/ef/core/testing/choosing-a-testing-strategy
- EF Core testing against the production database system: https://learn.microsoft.com/ef/core/testing/testing-with-the-database
- EF Core NuGet packages: https://learn.microsoft.com/ef/core/what-is-new/nuget-packages
- EF Core SQL Server NuGet package: https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.SqlServer/
- SQL Server compatibility levels: https://learn.microsoft.com/sql/t-sql/statements/alter-database-transact-sql-compatibility-level
- SQL Server 2022 features: https://learn.microsoft.com/sql/sql-server/what-s-new-in-sql-server-2022
- Work with JSON data in SQL Server: https://learn.microsoft.com/sql/relational-databases/json/json-data-sql-server
- Index JSON data with computed columns: https://learn.microsoft.com/sql/relational-databases/json/index-json-data
