---
name: zoran
description: Generate C# code in Zoran Horvat's style
---


You are generating C# code in the style of Zoran Horvat — functional domain modeling, discriminated unions, value objects, immutability, explicit domain boundaries.

His guidelines: making illegal states unrepresentable (rich domain types over
  primitives/nulls), pushing nulls to the boundary (Option-like patterns, no
  null returns mid-domain), exhaustive pattern matching / switch expressions
  over if-chains, immutability by default, polymorphism over conditionals, and
  functional pipelines (LINQ-style transformation chains)

---

**Target**: Domain-heavy line-of-business applications where business rule correctness matters more than development velocity.

**Prerequisites** — this skill is **strictly scoped** to:
- **.NET 10** (LTS, released November 2025, supported until November 2028) — [official docs](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-10/overview)
- **C# 14** — [official docs](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-14)
- **EF Core 10** — [official docs](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/whatsnew)

This skill uses C# 14 `extension` blocks, the `field` keyword, null-conditional assignment, implicit span conversions, and EF Core `ComplexProperty` throughout. C# 12 primary constructors are also used where they reduce boilerplate.

> ⚠️ **C# 14 `extension` blocks — READ BEFORE DP2**: Syntax `extension(TargetType)` defines static extension members (called as `Type.Method()`). Add a receiver name, e.g. `extension(TargetType receiver)`, for instance-like members (called as `receiver.Method()`). Both forms live inside a `public static class XxxRole { ... }`. The examples in DPs 2–5 use this syntax extensively; see Decision Point 12 for the full specification. **If a call like `Plan.TryCreateNew(...)` or `estimate.Add(other)` looks unfamiliar, flip to DP12 now.**

## Meta-Principles

These five principles encode the philosophy underlying every specific rule. When a situation arises that no rule explicitly covers, apply these principles.

### Principle 1: "No primitive passes a domain boundary untyped."

Every `string`, `Guid`, `decimal`, and `DateTime` crossing between layers must be wrapped in a named type. `CompanyHandle` is never a `string`. `AccountId` is never a `Guid`. `Timestamp` is never a `DateTime`. A method signature of `(Guid, Guid, decimal)` tells you nothing. A signature of `(AccountId, EmployeeId, Money)` tells a story.

### Principle 2: "Invalid objects must be unrepresentable."

Validation happens in construction — not in a separate `Validate()` method called later. The type system itself guarantees correctness: a `Money` instance can never hold a negative amount; an `AccountId` can never be `Guid.Empty`; a `Timestamp` can never be non-UTC. If you can hold an invalid object in memory, you will eventually process it.

### Principle 3: "Public constructors are a last resort — use factories."

Domain types have `internal` constructors. Construction happens through `static class XxxConstruction { extension(Xxx) { ... } }` factory methods returning `T?` (null = validation failure) or throwing on truly invalid input. Factories enable cross-field validation that single-property `init` accessors cannot.

### Principle 4: "Tell the compiler what you know — not what the database knows."

Entities declare their intent via types (discriminated unions, state machines with abstract records), not flags (`bool IsApproved`, `enum Status`). Use `abstract record` + sealed variants for finite states. Use property patterns (`switch` with `when` guards) for value-based dispatch. Use `IEntityTypeConfiguration` to bridge the gap between domain types and database columns. The domain model drives the design. The database conforms to the domain — never the reverse.

### Principle 5: "One concept, one file, one type."

No `Utils.cs`, no `Helpers.cs`, no `Constants.cs`, no `Enums.cs`. Every concept gets its own file. Even small types like `Temperature`, `Humidity`, `AccountId` earn dedicated files. A developer looking for `AccountId` finds it immediately — not buried inside a `Common.cs` catch-all.

---

## Anti-Patterns — Common Mistakes That Break the Style

These patterns explicitly violate Zoran's conventions. If you generate any of these, stop and re-evaluate.

| Anti-Pattern | Fix |
|---|---|
| `{ get; set; }` on domain properties | `{ get; init; }` for immutable, `{ get; private set; }` for controlled mutation |
| `(Guid accountId, Guid userId)` as a method signature | `(AccountId accountId, EmployeeId userId)` — wrap every primitive |
| `bool IsApproved` or `enum Status` for state | Discriminated union: `abstract record Approval; sealed record Pending : Approval;` |
| `[Required] public string Name` on a domain model | `{ get; init; }` with ternary + throw validation |
| `[Key]` or `[Column]` attributes on domain classes | Fluent API `IEntityTypeConfiguration<T>` in `Data/EntityConfiguration/` |
| `DbSet<T>.Find(id)` | `FirstOrDefaultAsync(e => e.PublicId == domainKey)` |
| Wrapping `DbContext` in `IRepository<TAggregate, TKey>` | Inject `IDbContextFactory<T>` and create short-lived contexts with `await using` |
| `OwnsOne` / `OwnsMany` for value objects | `ComplexProperty` (EF Core 8+) — flattens into owning table, no JOINs |
| Forgetting parameterless constructor on value objects used in `ComplexProperty` | Add `private Money() : this(default, default!) { }` — required for EF Core materialization |
| `ControllerBase` subclass | `public static class Xxx { ... }` with static Minimal API methods |
| `// Arrange` / `// Act` / `// Assert` comments in tests | Whitespace grouping; self-documenting method names |
| `IServiceProvider.GetService<T>()` inside a class | Constructor or parameter injection |
| `AutoMapper` or `.Map<T>()` for layer conversion | Hand-written extension method: `extension(Source) { public Target ToResponseDto() => ... }` |
| `Utils.cs`, `Helpers.cs`, `Constants.cs`, `Enums.cs` | One concept, one file. If an enum is referenced from multiple files, it gets its own file. An enum used exclusively within a single class may remain colocated. |
| `public` constructor on a domain entity | `internal` constructor + `static class XxxConstruction` with `extension(Xxx)` factories |
| `bool IsValid { get; }` or `void Validate()` | Validation in `{ get; init; }` at construction time — invalid objects never exist |
| FluentAssertions / Shouldly / Moq / NSubstitute | Raw MSTest `Assert.*` only. Zero mocking. |
| `Newtonsoft.Json` / `JsonConvert` | `System.Text.Json` — built into .NET, zero dependencies. |
| `string` for identifiers (`string handle`, `string currencyCode`) | Dedicated type: `CompanyHandle`, `Currency`, `Iso4217Currency` |
| Shadow PK named `"Id"` when domain model already has an `Id` property | Rename shadow PK to `"Key"` with `HasColumnName("Id")` |
| `DateTimeKind` stripped silently by SQL Server | Reapply `DateTime.SpecifyKind(dt, DateTimeKind.Utc)` in value converter |
| Assuming `ComplexProperty` sub-objects populate in order | Null-conditional guard: `Currency?.DecimalPlaces ?? 0` |
| `record with { Prop = badValue }` silently bypassing property initializer validation | Move validation to `init => field = value ?? throw` (single-property) or remove `init` entirely for cross-property checks (see DP1) |

---

> **Companion types**: Code examples reference types not defined in this file (`Currency`, `InvoiceStatus`, `AppDbContext`, `IncompleteApproval`, `ApprovedTransfer`, `ICompletedApproval`, `IIncompleteApproval`, etc.). These follow the same conventions as the types shown here; generate them accordingly.

## Decision Point 1: Creating a Value Object

Value objects wrap primitive types in domain-meaningful types that enforce invariants at construction. Validate in the `init` accessor body using the C# 14 `field` keyword for single-property validation, or use bare `{ get; }` to disable `with` for cross-property validation (see `with`-bypass warning below). Static factory methods provide well-known instances (`UtcNow`, `NewId()`) while keeping the primary constructor internal for deserialization. Never add implicit conversion operators — they hide allocation and bypass validation.

> ⚠️ **C# 14 positional records with explicit property declarations**: The compiler emits CS8907 ("Parameter is unread") when a positional record parameter shares a name with an explicitly declared property that doesn't consume it. Fix by adding `= ParameterName` property initializer — e.g., `public string Code { get; init => ... } = Code;`. Do NOT suppress CS8907 globally; it catches real name-mismatch bugs.
>
> ⚠️ **C# 14 primary constructor parameter scoping with `field` keyword**: When a primary constructor parameter shares a name with a property, **scoping rules differ by context**:
>
> - **Property initializers** (`= paramName`): The parameter **shadows** the member. `= id` correctly references the constructor parameter.
> - **Accessor bodies** (`init =>`, `get =>`, method bodies): The member **shadows** the parameter. `Id` in an accessor body refers to the **property**, not the constructor parameter.
>
> **Correct pattern**: Use **lowercase parameter names** in primary constructors and **`value`** in init accessors:
> ```csharp
> internal record TransferId(Guid id)  // lowercase parameter
> {
>     public Guid Id
>     {
>         get;
>         init => field = value != Guid.Empty ? value  // `value` = implicit init parameter
>             : throw new ArgumentException("TransferId cannot be empty", nameof(Id));
>     } = id;  // `id` = constructor parameter (initializer scope)
> }
> ```
>
> **Incorrect pattern** (works through fragile initialization order, but semantically wrong):
> ```csharp
> internal record TransferId(Guid Id)  // uppercase parameter
> {
>     public Guid Id
>     {
>         get;
>         init => field = Id != Guid.Empty ? Id  // ❌ `Id` = property, not parameter!
>             : throw new ArgumentException("TransferId cannot be empty", nameof(Id));
>     } = Id;
> }
> ```
> In the incorrect pattern, `Id` in the init body calls the property's getter (which reads the backing field), not the constructor parameter. This works only because the property initializer `= Id` already set the field from the parameter. Use `value` explicitly to avoid this ambiguity. See [C# 12 primary constructors spec](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-12.0/primary-constructors#lookup) and [Roslyn issue #80771](https://github.com/dotnet/roslyn/issues/80771).
>
> ⚠️ **CS9264 and `[field: MaybeNull, AllowNull]`**: When the compiler cannot infer null-resilience for a `field`-backed non-nullable property (common with `private` parameterless constructors for EF Core materialization), add both `[field: MaybeNull, AllowNull]` to the property declaration — e.g., `[field: MaybeNull, AllowNull] public string Code { get; init => ... }`. `MaybeNull` tells the compiler the field may be null when read (suppressing CS9264); `AllowNull` permits null writes. Per the official docs, both are needed — `MaybeNull` alone is incomplete. Do NOT suppress CS9264 globally.

### MUST
- MUST declare value objects as `internal record` with a primary constructor wrapping the underlying primitive. **Rationale**: Records deliver structural equality and `{ get; init; }` value semantics out of the box. Value objects are internal to their feature library (see Decision Point 6); value objects shared across multiple features belong in a shared kernel assembly where they may be `public record`.
- MUST validate in the `init` accessor body using ternary + throw, never in a separate `Validate()` method or `IsValid` property on domain objects. **Rationale**: The object must never exist in an invalid state; construction-time failure is fail-fast. Prefer the C# 14 `field` keyword (`init => field = value ?? throw ...`) — this fires for both `new` and `with`. The `{ get; init; } = value ?? throw` property initializer pattern is bypassed by `with` expressions (see warning below). (UI view models may expose `IsValid`; this rule is for domain objects.)
- MUST include `nameof()` in every `ArgumentException` call that refers to a parameter name. **Rationale**: Eliminates magic strings that silently rot under rename refactoring.

### SHOULD
- SHOULD use `DateTimeOffset` (or a `Timestamp` value object wrapping it) for all domain properties that represent moments in time — transaction timestamps, audit fields, scheduling dates, billing periods, expiry dates. `DateTimeOffset` carries its UTC offset through SQL Server's `datetimeoffset` column, making the value unambiguous regardless of time zone. Reserve `DateTime` only for pure dates (`DateOnly`) or abstract times (`TimeOnly`). When wrapping in a `Timestamp` value object, validate that the offset is UTC in the `init` accessor. **Rationale**: Microsoft's official guidance: "Consider DateTimeOffset as the default date and time type for application development." `DateTime.Kind` is silently stripped by SQL Server, requiring a converter to reapply — the converter's existence is evidence that `DateTime` is the wrong type. Billing systems in particular are vulnerable to DST-related double-charging when timestamps are ambiguous. **Exception**: Use `DateTime` when integrating with legacy systems that require it, when the database schema cannot be migrated, or when performance testing shows `DateTimeOffset` overhead is unacceptable.
- SHOULD provide a static `NewId()` factory method for ID value objects that wraps `Guid.NewGuid()`. **Rationale**: Centralizes GUID generation; makes intent explicit at the call site.
- SHOULD prefer `Guid`-wrapped IDs for public-facing identifiers (exposed in DTOs, URLs, or external systems) for their non-guessability and global uniqueness. **Rationale**: `Guid` IDs prevent enumeration attacks and eliminate coordination overhead in distributed systems. MAY use `int` or `long`-wrapped IDs for internal identifiers where sequential ordering or compact storage matters.
- SHOULD provide well-known static instances for reference-data value objects (`Currency.USD`, `Timestamp.UtcNow`). **Rationale**: Static instances serve as a canonical registry discoverable at the call site, eliminating magic strings. ⚠️ Do NOT provide `Empty` sentinels for ID types — Microsoft explicitly advises against sentinel values, and `field`-based validation (`init => field = value != Guid.Empty ? value : throw`) blocks `Guid.Empty` anyway. Use `TryCreate` returning `T?` instead; `null` already means "no ID."
- SHOULD include more than a single field in value objects when additional data naturally clusters with the core value (e.g., `Iso4217Currency` in `Money`). **Rationale**: Co-locating related data prevents primitive obsession and enables domain arithmetic without back-referencing external data.
- SHOULD bundle domain rules into the value object itself — not in a separate service. Example: `Currency` carries `DecimalPlaces` so `Money` can auto-round without consulting an external configuration. **Rationale**: The value object becomes both identity and policy carrier; callers never need to know which currency has which precision. When reference data comes from an external source or needs dependency injection, MAY use a provider/registry pattern (`IIso4217CurrencyProvider`) instead of embedding all metadata in the value object.

### MAY
- MAY override `ToString()` for human-readable display of truncated identifiers. **Rationale**: Improves log and debug readability without exposing internal structure.
- MAY define operator overloads for comparable types like timestamps and monetary amounts. **Rationale**: Enables natural comparison syntax (`ts1 > ts2`, `m1 + m2`) without forcing callers to unwrap primitive values. Note: operator overloads survive EF Core → SQL translation, enabling `db.Transfers.Where(t => t.ExecutedAt >= from)`.
- MAY use `record struct` instead of `record` for small, frequently-allocated value types (currency codes, monetary amounts, IDs). **Rationale**: Value types avoid heap allocation and reduce GC pressure. When used, validation typically moves to the entity aggregate level since `record struct` init accessors differ from reference-type records.

> ⚠️ **EF Core `ComplexProperty` requires a private parameterless constructor on value objects.** If your value object will be flattened into an entity table via `ComplexProperty` (see Decision Point 8), you MUST add `private TypeName() : this(default, default!) { }`. Without it, EF Core cannot materialize the object and will throw at runtime. Add it when you create the value object — don't backfill later.
>
> ⚠️ **`with` expressions bypass property initializer validation on records.** The compiler-generated copy constructor used by `with` does a field-by-field copy and does NOT run property initializers (the `= value ?? throw` part). This means `original with { Id = Guid.Empty }` silently skips the validation guard. Two fixes, depending on the validation type:
> 
> **Single-property validation** (non-empty GUID, non-null string, non-negative number): move validation into the `init` accessor body using the C# 14 `field` keyword — `init => field = value ?? throw`. This fires for both `new` and `with`.
> 
> **Cross-property validation** (Min ≤ Max, date range ordering): remove `init` from the cross-validated properties entirely, using bare `{ get; }`. `with` cannot touch a property without an `init` setter — the user is forced back through the constructor where all parameters are in scope for simultaneous validation. The record still provides value equality, `ToString()`, and `Deconstruct()`.

```csharp
internal record TransferId(Guid id)
{
    // Single-property validation in init body — fires for both `new` and `with`
    // Use `value` (implicit init parameter), not the property name
    public Guid Id
    {
        get;
        init => field = value != Guid.Empty ? value
            : throw new ArgumentException("TransferId cannot be empty", nameof(Id));
    } = id;

    public static TransferId NewId() => new TransferId(Guid.NewGuid());

    public override string ToString() =>
        Id.ToString()[..3] + "…" + Id.ToString()[^3..];
}
```

```csharp
internal record AccountId(Guid id)
{
    public Guid Id
    {
        get;
        init => field = value != Guid.Empty ? value
            : throw new ArgumentException("Account ID must be a non-empty GUID.", nameof(Id));
    } = id;

    public static AccountId? TryCreate(Guid? id) =>
        id is Guid g && g != Guid.Empty ? new AccountId(g) : null;
}
```

```csharp
internal record Iso4217Currency(string alphabeticCode, int numericCode, int decimalPlaces)
{
    [field: MaybeNull, AllowNull]
    public string AlphabeticCode
    {
        get;
        init => field = !string.IsNullOrWhiteSpace(value) && value.Length == 3 && value.All(char.IsUpper) ? value
            : throw new ArgumentException("Alphabetic code must be a three-letter uppercase string.", nameof(AlphabeticCode));
    } = alphabeticCode;

    public int NumericCode
    {
        get;
        init => field = value > 0 && value <= 999 ? value
            : throw new ArgumentException("Numeric code must be a three-digit number between 001 and 999.", nameof(NumericCode));
    } = numericCode;

    public int DecimalPlaces
    {
        get;
        init => field = value >= 0 ? value
            : throw new ArgumentException("Minor unit must be a non-negative integer.", nameof(DecimalPlaces));
    } = decimalPlaces;

    // Required by EF Core for ComplexProperty materialization
    private Iso4217Currency() : this(default!, default, default) { }
}
```

---

## Decision Point 2: Creating a Domain Aggregate/Entity

Entity constructors are `internal`, mutable state uses `{ get; private set; }`, immutable state uses `{ get; init; }`. Private `List<T>` backing fields with public `IEnumerable<T>` accessors prevent external mutation of child collections.

### MUST
- MUST declare entity constructors as `internal`, never `public`. **Rationale**: Forces all construction through controlled factory methods that can normalize input and reject invalid combinations.
- MUST use `{ get; private set; }` for mutable state and `{ get; init; }` for immutable one-time-set properties. **Rationale**: `private set` scopes mutation to the aggregate.
- MUST back collection properties with a private `List<T>` field and expose a public `IEnumerable<T>` or `IReadOnlyCollection<T>` accessor. **Rationale**: Prevents external mutation of child collections.
- MUST ensure a `TryCreateNew()` method has at least one code path that returns `null`. A factory that always succeeds must use `Create()` or `New()` instead — the `Try` prefix signals to callers that failure is possible and they must handle it. A `TryCreateNew()` that never returns null is a violation of the established .NET Try-pattern convention (every built-in `TryXxx` method — `TryParse`, `TryGetValue`, `TryAdd`, `Uri.TryCreate` — has a documented failure path). **Rationale**: The Try-pattern is defined by Microsoft's Framework Design Guidelines for members that "might throw/return false in common scenarios." A `Try`-prefixed method that always succeeds misleads callers into writing unnecessary null checks and erodes trust in the naming convention.

### SHOULD
- SHOULD provide a static `XxxConstruction` class containing `extension(Xxx)` factory methods for construction and persistence reconstitution. **Rationale**: Separates construction logic from the aggregate itself, keeping the type focused on behavior rather than factory boilerplate.
- SHOULD name persistence reconstitution factories `TryRestore()` (returning `T?`). **Rationale**: Distinguishes database hydration from fresh creation; nullable return signals soft failure if stored data is corrupt.
  > ⚠️ **`TryRestore` and property accessibility**: Since `XxxConstruction` is a separate static class, it cannot set `private set` properties via object initializers. Two approaches:
  > 
  > 1. **Constructor approach** — Pass ALL properties (including mutable ones like `IsActive`, `CanceledAt`) through the entity's primary constructor. The factory calls `new Entity(...)` with all values — never `new Entity(...) { PrivateSetProp = x }`. This keeps properties as `private set` but grows the constructor signature with every mutable field.
  > 2. **`internal set` approach** — Use `{ get; internal set; }` for mutable properties when the construction factory lives in the same assembly and constructor bloat is a concern. `internal set` is accessible to `XxxConstruction` (same assembly) but still blocks external mutation. This is not a workaround; it is a deliberate scoping decision for assemblies that co-locate domain and construction code.
- SHOULD place nested ID record structs inside the aggregate class. **Rationale**: Scopes the ID type to its owning aggregate and avoids namespace pollution.
- SHOULD gate mutable setters behind a status check (`InEditable<TValue>()`) when some states forbid mutation (e.g., an issued invoice cannot be modified). **Rationale**: Encodes state-machine rules at the property level.
- SHOULD model entity state as a discriminated union property (see Decision Point 3) rather than an enum or flag, and guard mutable operations with pattern-match checks against the current state. Example: `_state is Editing edit ? PerformEdit(edit) : throw...`.
- SHOULD use the `field` keyword (C# 14) in `set` and `init` accessors when validation logic or mutation gating needs access to the backing value. **Rationale**: Eliminates the boilerplate of declaring a separate private backing field when all the accessor does is validate-then-assign. Example: `set => field = InEditable(value);` or `init => field = value.All(...) ? value : throw ...`.
- SHOULD use `private init` for identity properties that must never change after construction (e.g., `Ordinal`, `Id` on child entities). **Rationale**: `private init` allows setting during construction or via a `With*` factory within the same type, but blocks all external mutation — the compiler enforces that the property is set exactly once.
- SHOULD snapshot mutable reference data as an immutable value object at the point of association. When an aggregate references another entity whose properties must remain constant for the aggregate's lifetime (e.g., a Subscription needs the Plan's price and billing cycle frozen at creation time), embed those properties as a value object within the aggregate rather than relying solely on a live FK lookup. This is the **Snapshot pattern** (Fowler) — standard in billing systems, where Stripe makes `Price` objects immutable by design. Changes to the referenced entity do not retroactively affect snapshots.

### MAY
- MAY use `required` init-only properties instead of constructor parameters for optional aggregate fields. **Rationale**: Allows callers to set only the properties they care about while keeping the constructor manageable.
- MAY define computed properties that aggregate over child collections (e.g., `Invoice.Total`, `Order.Subtotal`). **Rationale**: Derived values that are a pure function of the aggregate's state should live on the aggregate, not in a separate service or query layer.

```csharp
internal record PersonId(Guid id)
{
    public Guid Id
    {
        get;
        init => field = value != Guid.Empty ? value
            : throw new ArgumentException("PersonId cannot be empty", nameof(Id));
    } = id;

    public static PersonId NewId() => new(Guid.NewGuid());
}

internal record class Person
{
    internal Person(PersonId publicId, string firstName, string lastName) =>
        (PublicId, FirstName, LastName) = (publicId, firstName, lastName);

    public PersonId PublicId { get; init; }
    public string FirstName { get; init; }
    public string LastName { get; init; }
}

internal static class PersonConstruction
{
    extension(Person)
    {
        public static Person? TryCreateNew(string firstName, string lastName) =>
            string.IsNullOrWhiteSpace(firstName) ? null
            : new(PersonId.NewId(), firstName.Trim(), lastName?.Trim() ?? string.Empty);

        public static Person? TryRestore(PersonId publicId, string firstName, string lastName) =>
            string.IsNullOrWhiteSpace(firstName) ? null
            : new(publicId, firstName.Trim(), lastName?.Trim() ?? string.Empty);
    }
}
```

```csharp
// Inline discriminated union for invoice state — see Decision Point 3
internal abstract record InvoiceState;
internal sealed record Editing : InvoiceState;
internal sealed record Issued : InvoiceState;

internal class Invoice(
    InvoiceNumber number, string customerName, DateOnly invoicedOn,
    InvoiceState status, Currency currency)
{
    public string CustomerName
    {
        get;
        set => field = InEditable(AsValidCustomerName(value));      // Gate: rejected if not Editing
    } = AsValidCustomerName(customerName);

    public Currency Currency { get; set => field = InEditable(value); } = currency;

    private TValue InEditable<TValue>(TValue value) =>
        Status is Editing ? value
        : throw new InvalidOperationException("Cannot modify issued invoice.");

    private static string AsValidCustomerName(string name) =>
        !string.IsNullOrWhiteSpace(name) ? name.Trim()
        : throw new ArgumentException("Customer name cannot be empty.", nameof(name));
}
```

```csharp
internal class Invoice
{
    internal readonly record struct InvoiceId(Guid Value)
    {
        public static InvoiceId Empty { get; } = new InvoiceId(Guid.Empty);
        public static InvoiceId NewId() => new InvoiceId(Guid.NewGuid());
    }

    public InvoiceId PublicId { get; private set; } = InvoiceId.NewId();
    public string CustomerName { get; private set; } = string.Empty;
    public Money Amount { get; private set; }   // Money (defined elsewhere); Principle 1: no primitive crosses a domain boundary

    private List<InvoiceLine> LinesCollection { get; set; } = new List<InvoiceLine>();
    public IEnumerable<InvoiceLine> Lines => LinesCollection.AsReadOnly();

    public void AddLine(Product.ProductId productId, decimal amount)
    {
        if (LinesCollection.FirstOrDefault(l => l.ProductId == productId) is InvoiceLine existing)
            existing.AddAmount(amount);
        else
            LinesCollection.Add(new InvoiceLine(productId, amount));
    }
}
```

---

## Decision Point 3: Creating a Discriminated Union (State Machine)

Discriminated unions model mutually exclusive states with exhaustive compile-time checking. Root the hierarchy in `internal abstract record BaseType;` and declare each variant as `sealed record Variant(...) : BaseType;`. Interface markers group variants by shared capability (`IApprovable`, `IRejectable`), enabling methods that accept only a subset of states. Use exhaustive switch pattern matching, never if/else chains, so the compiler enforces that every variant is handled.

### MUST
- MUST root the union in `internal abstract record BaseType;`. **Rationale**: `abstract` prevents direct instantiation of the root, forcing consumers to work with concrete variants.
- MUST handle variant dispatch via exhaustive switch pattern matching, never if/else chains. **Rationale**: Pattern-matching switches produce compiler warnings when a variant is not handled; if/else chains silently miss cases.

### SHOULD
- SHOULD seal union variants to prevent further subclassing that would break exhaustiveness checks. **Rationale**: An open variant hierarchy invites extensions that silently escape pattern-match coverage; `sealed` makes the closed set of states a compiler-enforced guarantee.
- SHOULD define interface markers (`IApprovable`, `IRejectable`) for capability grouping across variants. **Rationale**: Interfaces allow method signatures to accept only the subset of variants that support a given operation, narrowing the surface at the call site.
- SHOULD provide static factory methods in an `XxxConstruction` class that validate inputs before constructing variants. **Rationale**: Separates validation from the record declarations, keeping variant types clean while preventing invalid states. When a variant's construction depends on data from a referenced aggregate (e.g., billing period from a Plan), receive that data as an explicit parameter — **see DP6 for the service-layer orchestration pattern**.
- SHOULD implement state transitions on the variants themselves as methods returning the abstract root type. **Rationale**: Each variant knows its own valid transitions; returning the root type ensures callers always pattern-match the result.

### MAY
- MAY colocate all variants in a single file — suitable when variants are simple data carriers with no behavioral logic. When variants carry substantial transition logic (multiple methods, guard clauses, helper logic), split each into its own file following Principle 5 (one concept, one file). **Rationale**: Simple unions benefit from at-a-glance visibility; behavioral variants deserve dedicated files for testability and focus.
- MAY use abstract classes instead of abstract records when mutable state is required in certain variants. **Rationale**: Records are naturally immutable; abstract classes support `{ get; private set; }` when a variant legitimately requires mutation.
- MAY use partial class wrappers when a variant needs to carry substantial behavioral logic. **Rationale**: Keeps the union declaration file focused on type declarations alone.

```csharp
internal abstract record Estimate;
internal sealed record Duration(TimeSpan Value) : Estimate;
internal sealed record Interval(TimeSpan Start, TimeSpan Span) : Estimate;
internal sealed record Unknown : Estimate;

public static class EstimateConstruction
{
    extension(Estimate)
    {
        public static Estimate CreateDuration(TimeSpan value) =>
            value < TimeSpan.Zero ? throw new ArgumentOutOfRangeException(nameof(value), "Duration must be non-negative")
            : new Duration(value);

        public static Estimate CreateInterval(TimeSpan start, TimeSpan span) =>
            start < TimeSpan.Zero ? throw new ArgumentOutOfRangeException(nameof(start), "Start must be non-negative")
            : span < TimeSpan.Zero ? throw new ArgumentOutOfRangeException(nameof(span), "Span must be non-negative")
            : new Interval(start, span);

        public static Estimate Unknown =>
            new Unknown();
    }
}
```

```csharp
public static class EstimateComposition
{
    extension(Estimate estimate)
    {
        public Estimate Add(Estimate other) => (estimate, other) switch
        {
            (Duration d1, Duration d2) => Estimate.CreateDuration(d1.Value + d2.Value),
            (Duration d, Interval i) => Estimate.CreateInterval(i.Start + d.Value, i.Span),
            (Interval i, Duration d) => Estimate.CreateInterval(i.Start + d.Value, i.Span),
            (Interval i1, Interval i2) => Estimate.CreateInterval(i1.Start + i2.Start, i1.Span + i2.Span),
            _ => Estimate.Unknown
        };
    }
}
```

```csharp
internal abstract record FourEyesApproval;

internal interface IApprovable { FourEyesApproval Approve(EmployeeId approver); }
internal interface IRejectable { FourEyesApproval Reject(EmployeeId rejector); }

internal record NotRequired : FourEyesApproval;

internal record PendingApproval : FourEyesApproval, IApprovable, IRejectable
{
    public FourEyesApproval Approve(EmployeeId approver) => new PartlyApproved(approver);
    public FourEyesApproval Reject(EmployeeId rejector) => new Rejected(rejector);
}

internal record PartlyApproved(EmployeeId Approver) : FourEyesApproval, IApprovable, IRejectable
{
    public FourEyesApproval Approve(EmployeeId approver) =>
        approver == Approver ? this : new FullyApproved(Approver, approver);
    public FourEyesApproval Reject(EmployeeId rejector) => new Rejected(rejector);
}

internal record FullyApproved(EmployeeId Approver1, EmployeeId Approver2) : FourEyesApproval, IRejectable
{
    public FourEyesApproval Reject(EmployeeId rejector) => new Rejected(rejector);
}

internal record Rejected(EmployeeId Rejector) : FourEyesApproval;
```

```csharp
internal abstract class Transfer(
    Money amount, TransferId id, AccountId from, AccountId to,
    TransferTimestamp expiresAt, FourEyesApproval approval) { /* ... */ }

internal class PendingTransfer : Transfer
{
    public Transfer AddApproval(EmployeeId approver) =>
        IncompleteApproval.Approve(approver) switch
        {
            ICompletedApproval completed =>
                new ApprovedTransfer(Amount, Id, FromAccountId, ToAccountId, ExpiresAt, completed),
            IIncompleteApproval incomplete =>
                new PendingTransfer(Amount, Id, FromAccountId, ToAccountId, ExpiresAt, incomplete),
            _ => throw new InvalidOperationException("Impossible state")
        };
}
```

---

## Decision Point 4: Using Property Patterns & Pattern Matching

Property patterns handle value-based dispatch. Use `switch` expressions with property patterns, tuple patterns, `when` guards, and `not` patterns. These compose naturally into state-machine decision chains that the compiler can verify exhaustively.

### MUST
- MUST use `switch` expressions (not statements) for value-based pattern matching. **Rationale**: Expressions guarantee exhaustiveness; statements silently fall through.
- MUST use property patterns to destructure records in switch arms: `{ State: State.Off }`, `{ Temperature: var temp }`. **Rationale**: Property patterns extract only the needed fields.

### SHOULD
- SHOULD use tuple patterns `(a, b) switch { ... }` when dispatching on two or more values simultaneously. **Rationale**: Tuple patterns let the compiler check exhaustiveness of the cross-product, catching combinations an if/else chain would miss.
- SHOULD use `when` guards for threshold and range checks: `when temp < idealTemperature.low`. **Rationale**: Guards keep the pattern shape clean while pushing numeric comparison logic into the guard clause where it belongs.
- SHOULD use `not` patterns for negation: `{ State: not State.Heating }`. **Rationale**: `not` is more readable than a catch-all `_` arm and documents the specific state being excluded.

### MAY
- MAY use nested records inside the same file as the pattern-matching logic when those records exist only to feed the state machine. **Rationale**: Keeps the domain simulation self-contained without polluting the project namespace.
- MAY use list patterns (`[]`, `[var single]`, `[var first, ..]`) for dispatch on collection shape. **Rationale**: List patterns express intent more directly than indexing or `Count` checks, and produce exhaustiveness warnings when shapes are missed.

```csharp
static class ComplexDomainDemo
{
    enum State { Heating, Cooling, Ventilation, Off, StandBy }
    enum OperatingMode { Off, Heating, Cooling, Ventilation }
    enum Season { Heating, Cooling, Transitional }

    record Hvac(OperatingMode Mode, AirExchange AirExchange, Season Season, State State);

    // Top-level dispatch: nested property patterns on a single record
    private static Hvac Update(Hvac device, Sensors sensors) => device switch
    {
        { State: State.Off } => device,
        { Mode: OperatingMode.Off } => device with { State = State.StandBy },
        { Season: Season.Heating } => RegulateHeating(device, sensors, GetIdealTemperatureRange()),
        { Season: Season.Cooling } => RegulateCooling(device, sensors, GetIdealTemperatureRange()),
        _ => device
    };

    // Composed dispatch: tuple patterns + when guards + not patterns
    private static Hvac RegulateHeating(Hvac device, Sensors sensors,
        (Temperature low, Temperature high) idealTemperature) => (device, sensors) switch
    {
        (_, { Temperature: var temp }) when temp < idealTemperature.low
            => device with { State = State.Heating },
        ({ State: State.Heating, AirExchange: AirExchange.Fresh }, { Temperature: var temp })
            when temp >= idealTemperature.high
            => device with { State = State.Ventilation },
        ({ State: State.Heating }, { Temperature: var temp }) when temp >= idealTemperature.high
            => device with { State = State.StandBy },
        _ => device
    };

    // not patterns for describing state transitions
    private static string DescribeChange(Hvac previous, Hvac current) => (previous, current) switch
    {
        ({ State: State.Heating }, { State: not State.Heating }) => "Turn off heating",
        ({ State: not State.Heating }, { State: State.Heating }) => "Turn on heating",
        ({ State: State.Cooling }, { State: not State.Cooling }) => "Turn off cooling",
        ({ State: not State.Cooling }, { State: State.Cooling }) => "Turn on cooling",
        _ => "No change"
    };
}
```

---

## Decision Point 5: Handling Validation / Construction Failure

Three failure-handling tiers. Hard failures (corrupt input): ternary + throw in `init` accessor bodies — no recovery. Soft failures (invalid user input): `TryCreateNew()` returning `T?`. Multi-step validation pipelines: `.Bind()` on `T?` to chain `TryCreate` calls without nested null-checks.

### MUST

- MUST use ternary + throw in `init` accessor bodies for invariants that are never expected to fail under correct system operation (e.g., non-empty GUID, non-negative amount). Prefer the C# 14 `field` keyword (`init => field = value ?? throw`) as shown in Decision Point 1 — it fires for both `new` and `with`. **Rationale**: Programming errors; the stack trace pinpoints the bug.
- MUST NOT define `bool IsValid { get; }` properties or `void Validate()` methods called after construction on domain objects. **Rationale**: Post-construction validation creates a window where an invalid object exists. (UI view models may expose `IsValid`; this rule is for domain objects.)

### SHOULD

- SHOULD use `TryCreateNew()` returning `T?` for user-input scenarios where failure is expected and recoverable. **Rationale**: Nullable return signals "try again" to the caller without throwing exceptions for normal control flow.
- SHOULD use `.Bind()` on `T?` to compose 3+ sequential `TryCreate` validations into a single expression that short-circuits on first null. **Rationale**: A `.Bind()` pipeline replaces a staircase of `if (x is null) return null` guards with a left-to-right chain. The extension lives in a small `NullableExtensions` static class and works on class nullables. See Decision Point 6's `TransferService.RequestTransferAsync` for the integrated usage pattern.

```csharp
// NullableExtensions — enable Bind on T? for pipeline composition
internal static class NullableExtensions
{
    extension<T>(T? value) where T : class
    {
        internal U? Bind<U>(Func<T, U?> func) where U : class =>
            value is null ? null : func(value);
    }
}
```

With this extension, a 4-step validation staircase:

```csharp
// Without Bind — staircase of null guards
var from = AccountId.TryCreate(request.FromAccountId);
if (from is null) return null;
var to = AccountId.TryCreate(request.ToAccountId);
if (to is null) return null;
var amount = Money.TryCreate(request.Amount, request.CurrencyCode);
if (amount is null) return null;
var transfer = Transfer.TryCreateNew(from, to, amount, maxAmount);
if (transfer is null) return null;
```

becomes:

```csharp
// With Bind — left-to-right, short-circuits on first null
var transfer = AccountId.TryCreate(request.FromAccountId)
    .Bind(from => AccountId.TryCreate(request.ToAccountId)
    .Bind(to => Money.TryCreate(request.Amount, request.CurrencyCode)
    .Bind(amount => Transfer.TryCreateNew(from, to, amount, maxAmount))));

if (transfer is null) return null;
```

Both produce the same result. The pipeline is not more functional — it is the same imperative logic read left-to-right instead of top-to-bottom.

### MAY

- MAY use `Result<T>` instead of `T?` when the caller needs typed error information — error codes for API responses, field-level validation messages, or structured failure objects. **Rationale**: `T?` can only say "something failed"; `Result<T>` can say *what* failed and let the caller `Match` on success vs failure at the boundary. Only adopt `Result<T>` when the `T?` + `.Bind()` pipeline is insufficient — for most features, `T?` is enough. A minimal `Result<T>` is a sealed class with `IsSuccess`, `Errors` (`IReadOnlyList<string>`), `Bind`, `Map`, and `Match`. See [CSharpFunctionalExtensions](https://github.com/vkhorikov/CSharpFunctionalExtensions) or Zoran Horvat's [Result type video](https://www.youtube.com/watch?v=LXF-rRWaIxc) for mature implementations; the pattern is the same — define a small `Result<T>` inline or pull in a library.

> ⚠️ **Do NOT put I/O inside `.Bind()` or async `.Match()` branches.** Domain validation via `.Bind()` is synchronous — it decides what the domain accepts. Persistence, database calls, and other I/O live *after* the guard clause as imperative `await` statements. See Decision Point 6's default pattern.
>
> **Decision guide**: For 1–2 null-checks, use imperative guards. For 3+ sequential `TryCreate` validations where only pass/fail matters, use `.Bind()` on `T?`. When the caller needs error details (HTTP 400 with field names, structured error objects), use `Result<T>`.
---

## Decision Point 6: Designing a Feature Library's Public API

A feature ships as a self-contained class library with one or two public interfaces. Everything else — domain models, entities, value objects, EF Core configuration — is `internal`. One `services.AddFeatureXxx()` call registers it.

### MUST
- MUST expose exactly one or two `public interface` types as the consumer's sole entry point to the feature. Example: `IMoneyTransferService`, `IInvoiceGenerator`. **Rationale**: One interface for the feature; two if read and write paths differ.
- MUST keep domain models, entities, value objects, EF Core configuration, and internal services as `internal`. **Rationale**: The domain is private implementation; exposing it couples consumers to internals. Extract shared types to a shared kernel (see SHOULD below).
- MUST provide an `IServiceCollection` extension method that registers the entire feature in one call: `services.AddTransfers(configuration)`. **Rationale**: A single registration call replaces the consumer's need to understand the feature's internal dependency graph — one `AddXxx()` and the feature is wired.
- MUST return DTOs and result records from public interface methods — never domain entities or value objects. **Rationale**: DTOs form a versionable contract; returning domain objects breaks consumers on model changes.
- MUST validate inputs at the public interface boundary before they reach domain logic. **Rationale**: The public API is the trust boundary; validate here, not in internal services.

### SHOULD
- SHOULD use the Options pattern (`IOptions<FeatureOptions>`) for feature configuration, with static defaults so `AddTransfers()` works with zero additional setup. **Rationale**: Options classes self-document every configurable knob; sensible defaults mean the consumer overrides only what they need.
- SHOULD design public interface methods as stateless pure functions of their parameters — pass `DbContext` or other dependencies at the implementation level, not on the interface itself. **Rationale**: Stateless signatures are trivially testable.
- SHOULD colocate the DI registration extension method inside the feature library itself, in the `Microsoft.Extensions.DependencyInjection` namespace. **Rationale**: The library owns its registration; consumer only needs a `using` directive.
- SHOULD return `Task<T?>` or a result union from methods where failure is expected and recoverable — never throw across the public API boundary for business-rule rejections. **Rationale**: Exceptions for programmer errors; nullable/result types for recoverable failures.
- SHOULD extract shared value objects (`Money`, `Timestamp`, `Currency`) into a shared kernel assembly when they are used by multiple feature libraries. In the shared kernel, these types may be `public record`. **Rationale**: Prevents duplicate implementations of the same concept across features. Within a single feature, domain types remain `internal`.
- SHOULD inject `ILogger<T>` into internal service implementations and log validation failures, business rule rejections, and exceptions. **Rationale**: The public API boundary and validation points are natural logging locations; structured logging enables production diagnostics without coupling to a specific sink.
- SHOULD load referenced aggregates before mutation when their state is needed for domain rules — not merely for FK validation. If the command depends on the referenced aggregate's state (e.g., a Subscription needs the Plan's `BillingPeriod` or must verify the Plan is active), load it in the service layer and return a typed failure (`null`) if absent. A bare existence query whose sole purpose is avoiding a `DbUpdateException` is optional; prefer designs where the referenced aggregate owns the creation flow (e.g., `Plan.StartSubscription()`). **Always retain the database FK constraint** as the final integrity guarantee — the referenced aggregate could be deleted between the read and `SaveChanges`.

  Example `SharedKernel` project:
  ```
  SharedKernel/
  ├── Money.cs          → public record Money(decimal Amount, Iso4217Currency Currency)
  ├── Timestamp.cs      → public record Timestamp(DateTime Value)
  └── Currency.cs       → public record Currency(string AlphabeticCode, int NumericCode, int DecimalPlaces)
  ```

### MAY
- MAY use `InternalsVisibleTo` for the feature's test project only — never for sibling feature libraries. **Rationale**: Tests need internal access to arrange state and verify invariants; sibling features communicate exclusively through the public API to maintain decoupling.
- MAY provide a separate `.Abstractions` project containing only interfaces and DTOs when the feature is consumed across process boundaries or by multiple alternative implementations. **Rationale**: An abstractions package lets consumers reference contracts without pulling in the implementation's full transitive dependency graph.
- MAY define a `FeatureXxxOptions` record with a static `Default` property and reasonable initial values. **Rationale**: A default instance enables zero-configuration `AddTransfers()` calls while the Options pattern still allows override from `appsettings.json` or environment variables.

**File: `FeatureLibraries/Transfers/IMoneyTransferService.cs`** — the single public entry point

```csharp
// The ONLY public types a consumer of this feature ever sees:
//   - IMoneyTransferService (this interface)
//   - TransferResult / TransferSummaryDto (return DTOs)
//   - TransferRequestDto (input DTO)
// Everything else — Transfer, Money, AccountId, TransferOptions, DbContext config — is internal.
public interface IMoneyTransferService
{
    Task<TransferResult?> RequestTransferAsync(
        TransferRequestDto request, CancellationToken ct = default);
    Task<IReadOnlyList<TransferSummaryDto>> GetPendingAsync(
        AccountIdDto account, CancellationToken ct = default);
}
```

**File: `FeatureLibraries/Transfers/TransferService.cs`** — internal implementation

```csharp
internal sealed class TransferService(
    IDbContextFactory<AppDbContext> contextFactory,
    IOptions<TransferOptions> options) : IMoneyTransferService
{
    public async Task<TransferResult?> RequestTransferAsync(
        TransferRequestDto request, CancellationToken ct)
    {
        // Compose synchronous domain validation with Bind() — short-circuits on first null.
        var transfer = AccountId.TryCreate(request.FromAccountId)
            .Bind(from => AccountId.TryCreate(request.ToAccountId)
            .Bind(to => Money.TryCreate(request.Amount, request.CurrencyCode)
            .Bind(amount => Transfer.TryCreateNew(from, to, amount, options.Value.MaxTransferAmount))));

        if (transfer is null) return null;   // Expected domain rejection — caller handles

        // Persistence and I/O are imperative. They are infrastructure, not domain.
        await using var db = await contextFactory.CreateDbContextAsync(ct);
        db.Transfers.Add(transfer);
        await db.SaveChangesAsync(ct);
        return TransferResult.FromTransfer(transfer);
    }
    // ...
}
```

> **Default pattern: compose synchronous domain validation with `.Bind()`, guard-clause the result, then imperative `await` for persistence.**
>
> - `TryCreate()` + `.Bind()` chains are for **synchronous, expected domain validation**. `null` means "this input was rejected by a business rule" — the caller decides how to present it.
> - Persistence, I/O, and other infrastructure live after the guard clause. They are **imperative** and **explicit** — database failures are infrastructure exceptions, not domain rejections, and should remain uncaught so they surface through normal error handling.
> - Do NOT put `SaveChangesAsync` or other I/O inside `.Bind()` or async `.Match()` branches. That mixes domain and infrastructure concerns, requires nested async closures, and obscures the natural boundary where authorisation, transactions, idempotency, auditing, and error handling belong.
>
> This hybrid pattern is clearer, easier for LLMs to generate correctly, and creates an obvious seam between "what the domain decided" and "what the system did with that decision."
>
> For the default case, `T?` + guard clause is sufficient and idiomatic.
> For simpler cases with 1–2 null-checks, imperative style is acceptable. For 3+ sequential validations, prefer the pipeline.

> **SHOULD pass domain data from referenced aggregates into factory methods, not hardcode defaults.** When state creation depends on data from a related aggregate (e.g., a Subscription's active period must match the Plan's `BillingPeriod`), load the referenced aggregate in the service layer, extract the needed value, and pass it as a parameter. **Correct**: `Subscription.ActiveState(plan.BillingPeriod)`. **Incorrect**: `Subscription.ActiveState()` with a hardcoded `AddMonths(1)`. Hardcoding a default that duplicates knowledge already in another aggregate creates silent drift when the source data changes. See DP3 for state machine factory guidance.

**File: `FeatureLibraries/Transfers/ServiceCollectionExtensions.cs`** — one-call registration

```csharp
namespace Microsoft.Extensions.DependencyInjection;

public static class TransferServiceCollectionExtensions
{
    public static IServiceCollection AddTransfers(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.Configure<TransferOptions>(
            configuration.GetSection("Transfers"));
        services.AddScoped<IMoneyTransferService, TransferService>();
        services.AddDbContextFactory<AppDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("TransfersDb")));
        return services;
    }
}
```

> `services.Configure<TOptions>(IConfiguration)` requires the `Microsoft.Extensions.Options.ConfigurationExtensions` package. For a dependency-free alternative, use the `Action<TOptions>` overload: `services.Configure<TransferOptions>(opts => configuration.GetSection("Transfers").Bind(opts))`.
>
> `UseSqlServer()` requires the `Microsoft.EntityFrameworkCore.SqlServer` NuGet package. For provider-agnostic libraries, prefer the `Action<DbContextOptionsBuilder>` overload of `AddTransfers()` so consumers supply their own provider.

> ⚠️ **MUST NOT use `opts => { }` as a default for `Action<DbContextOptionsBuilder>`.** Use `null` if configuration is optional (allows `OnConfiguring` fallback, matching Microsoft's `AddDbContext`). Make the parameter required (non-nullable) if configuration is mandatory (matching Microsoft's `AddDbContextPool`). An empty action default silently produces `InvalidOperationException: No database provider has been configured` at runtime — no compile-time or startup-time detection. EF Core's internal `IsConfigured` check detects provider extensions; an empty action adds none.

**File: `FeatureLibraries/Transfers/TransferOptions.cs`** — self-documenting configuration

```csharp
public record TransferOptions
{
    public static TransferOptions Default => new();

    public decimal MaxTransferAmount { get; set; } = 1_000_000m;
    public int TransferExpiryHours { get; set; } = 24;
}
```

> ⚠️ Options records use `{ get; set; }` (not `{ get; init; }`) because `IOptions<T>` configuration binding writes via property setters. `init`-only properties silently retain defaults and ignore `appsettings.json` overrides. This is the deliberate exception to the immutability rule in Principle 2 — options are configuration containers, not domain objects.

**File: `Consumer/Program.cs`** — integrating the feature, with optional HTTP exposure

```csharp
// Consumer adds the entire feature with one line
builder.Services.AddTransfers(builder.Configuration);

// The feature's public API is IMoneyTransferService — usable from anywhere:
// a background job, a console command, a gRPC handler, or an HTTP endpoint.
// Exposing it via HTTP is an orthogonal decision, not the feature's concern.
app.MapPost("/transfers", async (
    TransferRequestDto request, IMoneyTransferService transfers) =>
{
    var result = await transfers.RequestTransferAsync(request);
    return result is null
        ? Results.BadRequest("Invalid transfer.")
        : Results.Ok(result);
});
```

---

## Decision Point 7: Organizing Files and Namespaces

One-to-one filesystem-to-namespace mapping. File-scoped namespaces, one type per file. `<RootNamespace>` in `.csproj` decouples the project file name from the namespace.

### MUST
- Use file-scoped namespaces exclusively: `namespace X.Y.Z;` with no braces. **Rationale**: Eliminates one level of indentation.
- Place exactly one type per file. **Rationale**: The filename is the type.
- Set `<RootNamespace>` in every `.csproj` so the project file name does not leak into C# namespaces. **Rationale**: Decouples project file name from namespace.

### SHOULD
- Colocate discriminated union variants in a single file when they are simple data carriers (see Decision Point 3 for guidance on splitting behavioral variants). **Rationale**: The variants define one closed hierarchy; splitting scatters exhaustiveness.
- Place standalone `ValueConverter` classes in `Data/Conversions/` (not nested inside the configuration) when the converter contains non-trivial logic (see Decision Point 8). **Rationale**: Converters with domain knowledge deserve their own file.

### MAY
- Keep a console demo project flat when it has fewer than five source files. **Rationale**: Adding `Models/` and `UseCases/` folders for three files creates directory noise with no navigational benefit.

<!-- WebApiModern.csproj and WebApiModern.Models.csproj -->

```xml
<!-- WebApiModern.csproj — entry project: file named WebApiModern, namespace is TbdModern.WebApi -->
<RootNamespace>TbdModern.WebApi</RootNamespace>

<!-- WebApiModern.Models.csproj — models project maps folder to namespace suffix -->
<RootNamespace>TbdModern.WebApi.Models</RootNamespace>
```

---

## Decision Point 8: Configuring EF Core (DbContext + Entity Config)

Strict separation: the domain knows nothing about persistence. Configuration bears sole responsibility for mapping. Shadow keys, alternate keys, `ComplexProperty`, value converters — keep the domain pure, give EF Core what it needs.

### MUST

- **Use a surrogate shadow primary key on every entity**, never exposing a database-generated integer in the domain model. **Rationale**: Shadow keys let EF Core manage identity internally without contaminating domain classes with persistence concerns.

  In the standard case (domain identity field is named `PublicId`):
  ```csharp
  builder.Property<int>("Id").ValueGeneratedOnAdd();
  builder.HasKey("Id");
  ```

  > 🚨 **CRITICAL: Column name collision when domain has `Id` property**
  >
  > If your entity has a domain property named `Id` (typed as a value object like `TransferId`), you **MUST** rename the shadow PK to avoid a column name collision. EF Core 7+ throws `InvalidOperationException: All properties on an entity type must be mapped to unique different columns` when two properties map to the same column.
  >
  > **Incorrect** (causes collision):
  > ```csharp
  > builder.Property<int>("Id").ValueGeneratedOnAdd();   // ← shadow PK maps to column "Id"
  > builder.HasKey("Id");
  > builder.Property(p => p.Id).HasConversion(...);       // ← domain property ALSO maps to column "Id"
  > // ❌ ERROR: Two properties cannot map to the same column "Id"
  > ```
  >
  > **Correct** (rename shadow PK):
  > ```csharp
  > builder.Property<int>("Key")           // shadow property named "Key" — avoids collision
  >     .HasColumnName("Id")               // database column still named "Id"
  >     .ValueGeneratedOnAdd();
  > builder.HasKey("Key");
  > builder.Property(p => p.Id)            // domain property maps to its own column
  >     .HasConversion(new PlanIdConverter())
  >     .HasColumnName("PublicId");        // explicit column name for domain ID
  > ```
  >
  > **Rule**: If your domain model has a property named `Id`, the shadow PK **MUST** use a different name (e.g., `"Key"`). Use `HasColumnName()` to control the actual database column names. This error occurs at model validation time — both during migration generation and at runtime when the `DbContext` is first used.

  **Rationale**: EF Core cannot have a shadow property and a CLR property with the same name mapping to the same column. Rename the shadow property and use `HasColumnName` to preserve the database convention.

  > ⚠️ **Shadow PKs apply to entities only — NOT to value objects mapped via `ComplexProperty`.** Complex types (e.g., `Money`, `Timestamp`, `BillingPeriod`) have no identity — no key, not even a shadow one. They are structurally part of the owning entity's row. Do NOT add `builder.Property<int>("Id")` or `builder.HasKey("Id")` inside an `IEntityTypeConfiguration<SomeValueObject>` — value objects are not configured with `IEntityTypeConfiguration<T>`, only `ComplexProperty()` lambdas. See the SHOULD section below for ComplexProperty configuration.

- Expose the domain identity as an alternate key or unique index, never making it the primary key. **Rationale**: Alternate keys enable query-by-domain-id on indexed columns while foreign keys reference the surrogate integer, so domain IDs remain opaque to table relationships.

  **When to use `HasAlternateKey` vs `HasIndex().IsUnique()`**: Use `HasAlternateKey(e => e.PublicId)` when the domain key is a foreign key *target* in another entity. Use `HasIndex(t => t.Id).IsUnique()` when the domain key is queried but never referenced as a FK — it is lighter-weight and communicates "this is unique but nobody points at it."

```csharp
  b.HasIndex(t => t.Id).IsUnique();   // Unique index: queried but no FK references it
  ```

  vs:

  ```csharp
  builder.HasAlternateKey(i => i.PublicId);  // Alternate key: FK targets exist (InvoiceLine → Invoice)
  ```

- Apply every type conversion through `HasConversion()` — either with a dedicated `ValueConverter<TModel, TProvider>` class or an inline lambda for trivial conversions.

- Use Fluent API exclusively in `OnModelCreating`; never annotate domain classes with `[Key]`, `[Required]`, `[Column]`, or `[Table]`. **Rationale**: Fluent API must be the single source of truth for all ORM mappings so the domain layer can be referenced without an EF Core dependency.

### SHOULD

- **Use `ComplexProperty` (not `OwnsOne`) for nested value objects.** `ComplexProperty` flattens value-object columns into the owning table — no JOINs, no separate tables. Nest `ComplexProperty` within `ComplexProperty` for multi-level value objects (e.g., `Transfer` → `Money` → `Currency`):

  ```csharp
  b.ComplexProperty(t => t.Amount, m =>                   // Level 1: Transfer → Money
  {
      m.Property(x => x.Amount)
          .HasColumnName("Amount")
          .HasColumnType("decimal(18,3)");

      m.ComplexProperty(x => x.Currency, c =>             // Level 2: Money → Currency
      {
c.Property(x => x.Code)
    .HasColumnName("CurrencyCode")
    .HasColumnType("varchar(3)");                    // Provably ASCII-only — ISO 4217 currency codes are always 3 uppercase letters
c.Property(x => x.DecimalPlaces)
    .HasColumnName("CurrencyDecimalPlaces");
      });
  });
  ```

  Every leaf property gets an explicit `HasColumnName` that flattens the namespace (`CurrencyCode` not `Amount_Currency_Code`). Prefer `varchar` over `nvarchar` only when the column is proven to contain ASCII-only data forever (e.g., ISO 4217 currency codes, ISO 3166 country codes) — otherwise default to `nvarchar` to avoid encoding bugs when non-ASCII characters enter the column later.

- **Add a private parameterless constructor on any value object used as a `ComplexProperty`.** EF Core materializes complex types by calling the parameterless constructor first, then backfilling properties:

  ```csharp
  internal record Money(decimal Amount, Currency Currency)
  {
      private Money() : this(default, default!) { }   // Required by EF Core for ComplexProperty materialization
      // ...
  }
  ```

  Similarly, entities using primary constructors need a parameterless constructor:
  ```csharp
  internal class Transfer(TransferId id, Money amount, Timestamp executedAt)
  {
      private Transfer() : this(default!, default!, default!) { }  // Required by EF Core
      // ...
  }
  ```
  The `default!` pattern chains to the primary constructor with null-suppressed defaults. EF Core calls this skeleton constructor, then populates properties separately.

- **Use null-conditional guards in value-object init accessors** when the object is used as a `ComplexProperty`. EF Core materializes sub-properties in an undefined order — `Currency` may still be null when `Money.Amount`'s init runs:

  ```csharp
  public decimal Amount { get; init; } = Math.Round(Amount, Currency?.DecimalPlaces ?? 0);
  //                                    ^^^^^^^^^^^^^^ null-conditional: Currency might not be populated yet
  ```

  **Caution — NRT warnings from `default!` on reference-type constructor parameters.** The `private Money() : this(default, default!) { }` pattern suppresses null warnings at the *call site*, but the property initializer `= Currency` (where `Currency` is a non-nullable reference type parameter) may still produce `CS8601`. This is NOT a false positive — the parameterless constructor really does pass null. Resolve with one of: (a) null-coalescing fallback: `= Currency ?? WellKnown.Default`; (b) suppress with `#pragma warning disable CS8601` on the specific init line; or (c) change the parameter to nullable and validate in the init accessor.

- **Place converters in `Data/Conversions/` as standalone files when they contain non-trivial logic.** Trivial converters (simple wrap/unwrap) may be nested inside the entity configuration class or written as inline lambdas. Converters with domain knowledge deserve their own file:

  ```csharp
  public class TimestampConverter : ValueConverter<Timestamp, DateTime>
  {
      public TimestampConverter() : base(
          ts => ts.Value,
          dt => new Timestamp(DateTime.SpecifyKind(dt, DateTimeKind.Utc)))  // ← reapply UTC — SQL Server strips DateTimeKind
      {
      }
  }
  ```
  **Rationale**: SQL Server does not preserve `DateTimeKind`. The converter must explicitly re-specify `DateTimeKind.Utc` when reading back, otherwise the domain invariant (`Timestamp` must be UTC) silently breaks. **Note**: This converter is only needed when `DateTime` is unavoidable (legacy schemas). Prefer `DateTimeOffset` for all new domain properties — see Decision Point 1.

  For trivial GUID wrappers, inline lambdas or nested converters suffice:
  ```csharp
  // Inline lambda — trivial
  entity.Property(e => e.Number).HasConversion(
      v => $"{v.Year:D4}/{v.Sequence:D3}",
      v => new InvoiceNumber(int.Parse(v[..4]), int.Parse(v[5..])));

  // Nested in configuration — trivial
  public class InvoiceIdConverter : ValueConverter<Invoice.InvoiceId, Guid>
  {
      public InvoiceIdConverter() : base(id => id.Value, value => new Invoice.InvoiceId(value)) { }
  }
  ```

- Place each `IEntityTypeConfiguration<T>` in a dedicated file under `Data/EntityConfiguration/`, one class per entity. **Rationale**: Configuration isolation mirrors the domain model's modularity and keeps `OnModelCreating` a short registry of `ApplyConfiguration` calls.
- Nest child-entity configuration classes inside the parent aggregate configuration (e.g., `InvoiceConfiguration.InvoiceLineConfiguration`). **Rationale**: Nesting communicates ownership and discoverability — readers find all configuration for an aggregate in one file.
- Use `builder.HasDiscriminator<string>().HasValue<TSubtype>()` with TPH inheritance for polymorphic aggregates like `Product → Material | Service`. **Rationale**: TPH avoids complex joins; EF Core materializes the correct subtype from the discriminator column in a single table scan.
- SHOULD decompose composite value objects into private shadow stand-in properties on the entity when the value object serves as a natural key referenced by child entities. **Rationale**: Shadow properties give full column-level control in the Fluent API while keeping the decomposed value object private to the entity. Example: `InvoiceNumber` decomposed into `private int InvoiceYear` and `private int InvoiceSequence`, with `entity.Ignore(e => e.Number)`, shadow properties mapped via `entity.Property<int>("InvoiceYear")`, and a composite alternate key `entity.HasAlternateKey("InvoiceYear", "InvoiceSequence")`. Child entities reference this via `HasForeignKey("InvoiceNumberYear", "InvoiceNumberSequence").HasPrincipalKey("InvoiceYear", "InvoiceSequence")`.
- SHOULD persist discriminated union state machines (from Decision Point 3) using entity-level TPH inheritance via `HasDiscriminator`, not as a JSON column or custom `ValueConverter`. **Rationale**: TPH maps each variant to a row in one table with a discriminator column and separate nullable columns for variant-specific data. This enables `db.Subscriptions.OfType<ActiveSubscription>()` to translate directly to `WHERE Discriminator = 'Active'` — no client-side filtering, no serialization ceremony. See Decision Point 2 for the `Transfer → PendingTransfer | ApprovedTransfer` pattern. Example:
  ```csharp
  builder.HasDiscriminator<string>("State")
      .HasValue<TrialingSubscription>("Trialing")
      .HasValue<ActiveSubscription>("Active")
      .HasValue<PastDueSubscription>("PastDue")
      .HasValue<CanceledSubscription>("Canceled")
      .HasValue<ExpiredSubscription>("Expired");
  ```
  Variant-specific columns (e.g., `TrialEndDate`, `CanceledOn`) are nullable — only populated for the corresponding variant's rows. Each variant needs a private parameterless constructor chaining to its primary constructor via `this(default!, ...)` for EF Core materialization:
  ```csharp
  internal sealed class ActiveSubscription(/* params */) : Subscription(/* base params */)
  {
      private ActiveSubscription() : this(default!, default!, default!, default!, default) { }
  }
  ```
  > ⚠️ **State transitions with TPH are delete-and-insert, not property assignment.** Since the CLR type changes (e.g., `ActiveSubscription` → `CanceledSubscription`), you cannot mutate in place. Load the entity, call the variant's transition method to create a new instance of the target type, then `db.Remove(old)` + `db.Add(new)` + `SaveChangesAsync()`. See Decision Point 2's `PendingTransfer.AddApproval()` for the return-new-instance pattern.
  >
  > ⚠️ **EF Core cannot bind `ComplexProperty` parameters in primary constructors.** When a subtype's primary constructor includes a `Money` or other `ComplexProperty`-mapped value object, EF Core's `ConstructorBindingConvention` throws "No suitable constructor was found ... Cannot bind 'planPriceSnapshot'." Every subtype needs a private parameterless constructor with a `this(default!, ...)` initializer so EF Core can materialize via the parameterless ctor and set properties afterward.
- SHOULD use `[JsonDerivedType]` + `[JsonPolymorphic]` attributes when discriminated union data crosses an external boundary (HTTP API, message bus, external system) and must be serialized to/from JSON. **Rationale**: System.Text.Json does not serialize `abstract record` hierarchies by default — without attributes, `JsonSerializer.Serialize(state)` emits `{}`, losing the type discriminator and all variant data. Annotate the union root and each variant:
  ```csharp
  using System.Text.Json.Serialization;

  [JsonPolymorphic(TypeDiscriminatorPropertyName = "type")]
  [JsonDerivedType(typeof(Trialing), "Trialing")]
  [JsonDerivedType(typeof(Active), "Active")]
  [JsonDerivedType(typeof(PastDue), "PastDue")]
  [JsonDerivedType(typeof(Canceled), "Canceled")]
  [JsonDerivedType(typeof(Expired), "Expired")]
  internal abstract record SubscriptionState;
  ```
  With these attributes, `JsonSerializer.Serialize(state, options)` and `JsonSerializer.Deserialize<SubscriptionState>(json, options)` round-trip the full hierarchy automatically — no custom `JsonConverter` needed. See [polymorphic serialization docs](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/polymorphism).
  > ⚠️ If a timestamp is embedded in the JSON, reapply `DateTimeKind.Utc` after deserialization — SQL Server and JSON both strip `DateTimeKind` on read.

### MAY
- Consolidate all configuration inline in `OnModelCreating` (skipping `IEntityTypeConfiguration<T>`) for simple schemas with few entities. **Rationale**: For small projects, the ceremony of separate configuration files outweighs the organizational benefit.
- Hide EF Core navigation properties from the domain model with `builder.Ignore()` and map relationships through string-based overloads like `builder.HasMany<InvoiceLine>("LinesCollection")`. **Rationale**: Hiding navigations keeps domain classes free of EF concerns while `builder.Navigation("LinesCollection")` still enables eager loading in queries.
- Omit explicit `IsRequired()` calls when using C# nullable reference types — the NRT annotations (`string` vs `string?`) already signal required vs optional to EF Core conventions. **Rationale**: NRT-driven convention is the modern, less verbose approach.
- Use EF Core 10 **named query filters** when multiple global filters must be independently toggled (e.g., soft-delete + multi-tenancy): `builder.HasQueryFilter("SoftDelete", b => !b.IsDeleted).HasQueryFilter("Tenant", b => b.TenantId == _tenantId)`. Disable selectively: `db.Blogs.IgnoreQueryFilters(["SoftDelete"])`. **Rationale**: Named filters replace the single-filter-per-entity limitation of EF Core 9 and earlier, enabling selective filter bypass without losing other filters.

```csharp
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.Property<int>("Id").ValueGeneratedOnAdd();
        builder.HasKey("Id");

        builder.Property(p => p.PublicId)
            .HasConversion(new ProductIdConverter());
        builder.HasAlternateKey(p => p.PublicId);

        builder
            .HasDiscriminator<string>("Type")
            .HasValue<Material>("Material")
            .HasValue<Service>("Service");
    }

    // Nested child-type configuration — discoverable with the parent aggregate
    public class MaterialConfiguration : IEntityTypeConfiguration<Material>
    {
        public void Configure(EntityTypeBuilder<Material> builder)
        {
            builder.Property(m => m.PricePerUnit).HasPrecision(18, 2);
        }
    }

    public class ServiceConfiguration : IEntityTypeConfiguration<Service>
    {
        public void Configure(EntityTypeBuilder<Service> builder)
        {
            builder.Property(s => s.HourlyRate).HasPrecision(18, 2);
        }
    }

    public class ProductIdConverter : ValueConverter<Product.ProductId, Guid>
    {
        public ProductIdConverter()
            : base(id => id.Value, value => new Product.ProductId(value)) { }
    }
}
```

```csharp
modelBuilder.Entity<Invoice>(entity =>
{
    entity.Property<int>("Id").ValueGeneratedOnAdd();
    entity.HasKey("Id");

    entity.HasAlternateKey("InvoiceYear", "InvoiceSequence");     // composite natural key — shadow properties mapping InvoiceNumber columns; unreachable from the domain model

    entity.Ignore(e => e.Number);
    entity.Property<int>("InvoiceYear").HasColumnName("Number_Year");
    entity.Property<int>("InvoiceSequence").HasColumnName("Number_Sequence");

    entity.Property(e => e.Currency).HasConversion(new CurrencyConverter());
    entity.Property(e => e.Status).HasConversion(new EnumConverter<InvoiceStatus>());

    entity.Ignore(e => e.Lines);
    entity.HasMany("LinesRepresentation")
        .WithOne()
        .HasForeignKey("InvoiceNumberYear", "InvoiceNumberSequence")
        .HasPrincipalKey("InvoiceYear", "InvoiceSequence");
});
```

---

## Decision Point 9: Persisting Domain Objects (Simplified)

Handler methods receive `IDbContextFactory<T>` via DI, then create a short-lived `DbContext` per unit of work with `await using`. No repository abstractions, no unit-of-work interfaces. Queries always use the domain key (the alternate key or unique index), never the shadow primary key. Reads use `AsNoTracking()` to avoid change-tracker overhead; writes use the default tracking behavior and call `SaveChangesAsync()` to persist.

### MUST
- Query exclusively by the domain key via LINQ — `db.Companies.FirstOrDefaultAsync(c => c.PublicId == id)`. Never call `DbSet<T>.Find(key)`. **Rationale**: `Find()` operates on the shadow primary key known only to EF Core, bypassing the domain's identity boundary entirely.
- Inject `IDbContextFactory<T>` into consuming classes, then create and dispose a `DbContext` per unit of work: `await using var db = await _contextFactory.CreateDbContextAsync();`. **Rationale**: Short-lived contexts prevent stale tracked entities; dispose promptly to free database connections.
- Query domain-typed properties through their `ComplexProperty`-mapped navigation paths — `db.Transfers.Where(t => t.Amount.Currency.Code == "USD")`. **Rationale**: `ComplexProperty` translates deeply nested LINQ expressions to flat SQL columns — use it.

### SHOULD
- Use `AsNoTracking()` on all read-only queries — `db.Companies.AsNoTracking().FirstOrDefaultAsync(...)`. **Rationale**: Read queries must never trigger unintended database writes through the change tracker.
- Use `Include("LinesCollection")` with string-based navigation names when eager-loading child collections that are hidden from the domain model. **Rationale**: The string-based overload maps to the shadow navigation configured in `IEntityTypeConfiguration` or inline `OnModelCreating`.
- Place entity configuration in `Data/EntityConfiguration/` — one `IEntityTypeConfiguration<T>` file per entity, applied via `modelBuilder.ApplyConfiguration()` in `OnModelCreating()`. **Rationale**: Keeps mapping logic separate from the `DbContext` and the domain model, so neither knows about the other.
- For multi-entity write operations, wrap in a transaction: `await using var tx = await db.Database.BeginTransactionAsync(ct);` and commit/rollback explicitly. **Rationale**: Multiple `SaveChangesAsync` calls without a transaction boundary leave the database in an inconsistent state on partial failure.
- Handle `DbUpdateConcurrencyException` at the service layer when optimistic concurrency conflicts occur. Retry or surface the conflict to the caller as a recoverable failure. **Rationale**: Line-of-business applications with concurrent editors need explicit conflict resolution; silent overwrites are unacceptable.
- ⚠️ **EF Core does not fix up navigation properties when you call `db.Add()` or `db.Remove()` directly.** The parent entity's navigation property remains stale in memory until `SaveChanges()` completes. This is by design — the EF team explicitly closed requests for pre-SaveChanges fixup (issues #23940, #10008). **Prefer manipulating the navigation** (fixup triggers automatically): `parent.States.Add(stateEntity);` → `parent.State` is immediately updated. **Or explicitly set the navigation after the operation**: `subscription.State = state;` after `db.Add(state)`. Without this, DTO conversions reading the stale navigation will return incorrect data (e.g., `subscription.ToResultDto()` with `State = "Unknown"` after a successful `CreateSubscriptionAsync`).
- ⚠️ **`is` pattern matching and `OfType<T>()` do not translate to SQL when the property is mapped via `ValueConverter`.** EF Core stores a primitive (e.g., `string`) but materializes a complex CLR type. `db.Invoices.Where(i => i.Status is Draft)` fails at runtime with "could not be translated." Use one of:
  1. Query by the stored primitive value directly (add a shadow discriminator property and filter on it).
  2. `ToListAsync()` then filter client-side with `.Where(i => i.Status is Draft)`.
  3. Restructure as entity-level TPH (see Decision Point 8) — `db.Subscriptions.OfType<ActiveSubscription>()` on the `DbSet` translates to `WHERE Discriminator = 'Active'` because EF Core knows the discriminator column.

### MAY
- For large projects with many query paths, extract complex read queries into dedicated query classes in `Data/Queries/` that take `DbContext` as a dependency. **Rationale**: Prevents the `DbContext` class itself from accumulating query logic.
- Use `.NET 10` `LeftJoin` and `RightJoin` LINQ operators for outer-join queries — `db.Students.LeftJoin(db.Departments, s => s.DepartmentId, d => d.Id, (s, d) => new { s.Name, Dept = d.Name ?? "[NONE]" })`. **Rationale**: .NET 10 first-class `LeftJoin`/`RightJoin` replaces the old `GroupJoin` + `SelectMany` + `DefaultIfEmpty` ceremony; EF Core 10 translates these directly to SQL `LEFT JOIN` / `RIGHT JOIN`.
- Use `ExecuteUpdateAsync` with a regular (non-expression) lambda when building dynamic update setters — `await db.Blogs.ExecuteUpdateAsync(s => { s.SetProperty(b => b.Views, 8); if (nameChanged) s.SetProperty(b => b.Name, "foo"); })`. **Rationale**: EF Core 10 accepts a statement lambda, eliminating the expression-tree construction ceremony required in EF Core 9 and earlier.
- Use EF Core 10 `ExecuteUpdateAsync` to bulk-update properties inside JSON columns mapped via `ComplexProperty` + `ToJson()` — `await db.Blogs.ExecuteUpdateAsync(s => s.SetProperty(b => b.Details.Views, b => b.Details.Views + 1))`. **Rationale**: EF Core 10 translates JSON property updates to efficient SQL (e.g., `JSON_MODIFY` or the new `modify()` method on SQL Server 2025 `json` type). This applies to value-object JSON columns (e.g., `BlogDetails`); for state machine persistence, use entity-level TPH with `HasDiscriminator` — see Decision Point 8.

---

## Decision Point 10: Writing a Test

Tests exercise real infrastructure — for feature libraries, the public interface (`IMoneyTransferService`) resolved from the DI container; for web API projects, the full HTTP stack through `WebApplicationFactory<Program>`. This approach trades fast unit-test feedback for high confidence that the system works end-to-end in its actual runtime environment.

### MUST
- Name test methods as `{Scenario}_{Condition}_{ExpectedResult}` — e.g., `CreateCompany_WithValidRequest_ReturnsCreatedCompany`. **Rationale**: The method signature is the test specification; reading the name tells you what is proven and under what conditions.
- Use ONLY MSTest assertion primitives. **Rationale**: Every assertion library beyond MSTest's built-in primitives adds a dependency that obscures failures with non-standard output.

  **Allowed primitives**: `Assert.AreEqual`, `Assert.IsNotNull`, `Assert.AreNotEqual`, `CollectionAssert.Contains`, `Assert.AreEqual(1, collection.Count())`, `Assert.IsTrue(collection.All(...))`, `Assert.IsTrue(collection.Any())`, `CollectionAssert.DoesNotContain`. For expected exceptions use `Assert.Throws<T>` or `Assert.ThrowsExactly<T>` — never `[ExpectedException]` (removed in MSTest v4), never the deprecated `Assert.ThrowsException`.
- Reference the `MSTest.Sdk` (4.2.x) in the test project — use `<Project Sdk="MSTest.Sdk/4.2.3">` in the `.csproj` to pull in `MSTest.TestFramework`, `MSTest.TestAdapter`, and `MSTest.Analyzers` automatically. For VSTest compatibility, also add `<PackageReference Include="Microsoft.NET.Test.Sdk" />`. **Rationale**: MSTest.Sdk eliminates the ceremony of individual package references; one SDK reference covers the entire test infrastructure.
- Mark each test class with `[TestClass]` and either `[DoNotParallelize]` on the class or `[assembly: DoNotParallelize]` to serialize database access. **Rationale**: A single database instance shared across parallel test runs produces nondeterministic failures.

### SHOULD
- Prefer zero mocks or stubs — exercise the real database and real feature code. For feature libraries, resolve the public interface from the DI container; for web APIs, exercise through `WebApplicationFactory`. **Rationale**: Integration tests validate real component interactions and catch misconfigurations, serialization mismatches, and database dialect issues that mocked tests cannot surface.
- Use `[ClassInitialize]` (or `[AssemblyInitialize]`) to drop and recreate the database once per test run. Use `[TestInitialize]` for per-test setup like creating a fresh `HttpClient`. In MSTest v4, `[ClassCleanup]` always runs at end-of-class (the `ClassCleanupBehavior` enum was removed). **Rationale**: A fresh database per test run guarantees no test order dependency and no leftover state.
- Subclass `WebApplicationFactory<Program>` and override `ConfigureWebHost` to call `builder.UseEnvironment("E2ETests")`. **Rationale**: Environment-specific configuration prevents tests from touching production resources.
- Place `[TestMethod]` and `[DataTestMethod]` methods in an abstract test class receiving `HttpClient` via primary constructor, then create a one-line concrete class that inherits from it and wires the fixture's `HttpClient`. **Rationale**: Separating test logic from infrastructure binding keeps tests portable across fixture implementations.
- Use `[DataTestMethod]` with `[DataRow]` for boundary testing — one `[DataRow]` row per invalid input variant. **Rationale**: Each row is a self-contained proof that a specific invalid value is rejected.
- Define private `record` types at the bottom of the test file for request and response DTOs. **Rationale**: File-local records keep test data contracts visible without polluting the project namespace.
- Use primary constructor injection for fixtures: `public class MyTests(WebApiTestsFixture fixture) : MyAbstractTests(fixture.Client)`. **Rationale**: Primary constructors eliminate boilerplate field declarations and redundant assignment.

### MAY
- Call `response.EnsureSuccessStatusCode()` on happy-path responses before deserializing. **Rationale**: A failed status code throws immediately with the actual status, avoiding confusing null-reference errors downstream.
- Separate the HTTP call from assertions with a blank line to visually delimit the request phase from the verification phase. **Rationale**: Whitespace grouping replaces `// Act` / `// Assert` comments without adding noise.
- Extract a static helper method like `CreateCompanyAsync(HttpClient client, NewCompanyRequest request)` when the same setup appears in three or more tests. **Rationale**: A shared helper eliminates copy-paste while remaining explicit about dependencies through its parameter list.
- MAY add fast unit tests at interface boundaries for rapid feedback during active development. These test domain logic against mocked dependencies and provide sub-second feedback. **Rationale**: Integration tests against real infrastructure remain the merge gate, but unit tests speed up the inner development loop.

```csharp
using System.Net.Http.Json;
using Microsoft.VisualStudio.TestTools.UnitTesting;

namespace EndToEndTests.Companies;

// This example shows web API testing via WebApplicationFactory.
// For feature library tests, resolve the public interface directly from the DI container (see SHOULD above).
// Requires: <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" />
[TestClass]
[DoNotParallelize]
public abstract class CreateCompanyTests
{
    private static WebApplicationFactory<Program> _factory = null!;
    private HttpClient? _client;

    [ClassInitialize(InheritanceBehavior.BeforeEachDerivedClass)]
    public static async Task ClassInitialize(TestContext context)
    {
        _factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(b => b.UseEnvironment("E2ETests"));
        await DatabaseFixture.ResetAsync();
    }

    [TestInitialize]
    public void TestInitialize()
    {
        _client = _factory.CreateClient();
    }

    protected HttpClient Client => _client!;

    [TestMethod]
    public async Task CreateCompany_WithValidRequest_ReturnsCreatedCompany()
    {
        var request = new NewCompanyRequest("Test Company", "USA");
        var createdCompany = await CreateCompanyAsync(Client, request);

        Assert.IsFalse(string.IsNullOrEmpty(createdCompany.Handle));
        Assert.AreEqual(request.Name, createdCompany.Name);
        Assert.AreEqual(request.IncorporatedInCode, createdCompany.IncorporatedInCode);
    }

    [DataTestMethod]
    [DataRow("", "USA")]
    [DataRow("Valid Name", "")]
    [DataRow("   ", "USA")]
    [DataRow("Valid Name", "   ")]
    [DataRow("Valid Name", "SOMETHING")]
    public async Task CreateCompany_WithInvalidCompanyName_ReturnsBadRequest(
        string name, string countryCode)
    {
        var request = new NewCompanyRequest(name, countryCode);
        var response = await Client.PostAsJsonAsync("/api/companies", request);

        Assert.AreEqual(System.Net.HttpStatusCode.BadRequest, response.StatusCode);
    }

    public static async Task<CompanyResponse> CreateCompanyAsync(
        HttpClient client, NewCompanyRequest request)
    {
        var response = await client.PostAsJsonAsync("/api/companies", request);
        response.EnsureSuccessStatusCode();
        var dto = await response.Content.ReadFromJsonAsync<CompanyResponse>();
        return dto ?? throw new InvalidOperationException("Response is null or deserialization failed");
    }

    public record NewCompanyRequest(
        string Name, string IncorporatedInCode, string Handle = null!)
    {
        public string Handle { get; init; } = Handle ?? Guid.NewGuid().ToString();
    }

    public record CompanyResponse(
        string Handle, string Name,
        string IncorporatedInCode, string IncorporatedInName);
}
```

For **feature library tests** (class libraries with no HTTP surface), resolve the public interface directly from a DI container built with real database configuration:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.VisualStudio.TestTools.UnitTesting;

namespace FeatureTests.Subscriptions;

[TestClass]
[DoNotParallelize]
public sealed class CreateSubscriptionTests
{
    private static IServiceProvider _services = null!;

    [ClassInitialize(InheritanceBehavior.BeforeEachDerivedClass)]
    public static Task ClassInitialize(TestContext context)
    {
        var services = new ServiceCollection();
        services.AddBillingSubscriptions(
            configure: _ => { },
            dbContextOptionsAction: options =>
                options.UseSqlServer("Server=.;Database=BillingTests;Trusted_Connection=True;"));
        _services = services.BuildServiceProvider();
        return DatabaseFixture.ResetAsync();
    }

    [TestInitialize]
    public void TestInitialize()
    {
        // Each test gets a fresh scope with a clean ISubscriptionService
        _scope = _services.CreateScope();
        _sut = _scope.ServiceProvider.GetRequiredService<ISubscriptionService>();
    }

    [TestCleanup]
    public void TestCleanup()
    {
        _scope?.Dispose();
    }

    private IServiceScope? _scope;
    private ISubscriptionService _sut = null!;

    [TestMethod]
    public async Task CreateSubscription_WithValidRequest_ReturnsSubscriptionInTrialingState()
    {
        var plan = await CreatePlanHelperAsync(_sut, "Pro Plan", "USD", 29.99m);
        var request = new CreateSubscriptionRequest(
            CustomerId: Guid.NewGuid(),
            PlanId: plan.Id,
            TrialDays: 14);

        var result = await _sut.CreateSubscriptionAsync(request);

        Assert.IsNotNull(result);
        Assert.AreEqual("Trialing", result.State);
    }

    [DataTestMethod]
    [DataRow(0)]     // Empty GUID → should fail construction in SubscriptionId init
    [DataRow(-1)]    // Negative days → no trial possible
    public async Task CreateSubscription_WithInvalidInput_ReturnsNull(int trialDays)
    {
        var request = new CreateSubscriptionRequest(Guid.Empty, Guid.NewGuid(), trialDays);
        var result = await _sut.CreateSubscriptionAsync(request);
        Assert.IsNull(result);
    }

    private static async Task<PlanSummaryDto> CreatePlanHelperAsync(
        ISubscriptionService svc, string name, string currency, decimal amount)
    {
        var plan = await svc.CreatePlanAsync(new CreatePlanRequest(
            name, null, amount, currency, 1, "Monthly"));
        return plan ?? throw new InvalidOperationException("Failed to create test plan");
    }
}
```

> **Feature library tests resolve the public interface directly** — no `WebApplicationFactory`, no `HttpClient`. `CreateScope()` provides a fresh `DbContext` per test. This is the pattern for class library APIs; use `WebApplicationFactory` only for projects with an HTTP surface (Minimal API or Controller projects).

---

## Decision Point 11: Converting Between Layers

Zoran keeps every layer in its own type — domain models, database rows, query DTOs, and response DTOs — and converts between them with explicit, hand-written extension methods. No AutoMapper, no implicit operators, no reflection. Each conversion method is a trivial constructor call that a reader can verify in one glance, and because the source type is already validated before conversion, the method never needs to fail.

### MUST
- MUST NOT allow a domain type file (e.g., `Company.cs`, `Plan.cs`) to import or reference a DTO, response type, or presentation concern. **Rationale**: This is the real dependency test — a domain type that knows about `CompanyResponseDto` is coupled to an outer layer. The domain must have zero dependencies on outer layers.
- MUST place domain→DTO conversion code where it can reference both domain types (inward) and DTO types (outward) without creating a backward dependency. In **multi-assembly** projects, conversions live in the Application or Presentation assembly. In **single-assembly** feature libraries, a `Conversions/` subfolder or separate namespace is sufficient — the conversion file references both types as the bridge between layers, while domain type files remain isolated. **Correct** (multi-assembly): `Application/Conversions/CompanyConversions.cs` with `extension(Company model) { public CompanyResponseDto ToResponseDto() => ... }`. **Correct** (single-assembly): `FeatureLibrary/Conversions/CompanyConversions.cs` with `extension(Company model) { public CompanyResponseDto ToResponseDto() => ... }` — `Company.cs` never imports `CompanyResponseDto`. **Incorrect**: `Domain/Entities/Company.cs` with a `ToDto()` method, or a domain type file with a `using` for a DTO namespace.
- MUST convert between layer types via explicit extension methods placed in a static class named `{SourceType}Conversions` or `{TargetType}Mappings`. **Rationale**: Explicit conversion methods are greppable, testable, and immune to silent mis-mappings that plague convention-based mappers.
- MUST declare database row types as `internal record class {Name}Row` with primitive-typed positional parameters matching the query columns exactly. **Rationale**: Positional records map directly to Dapper or EF Core projections with zero ceremony and expose no mutable state outside the data layer.
- MUST declare API response types as `public record class {Name}ResponseDto` with `{ get; init; }` properties or positional parameters. **Rationale**: Init-only or positional records guarantee the DTO is immutable after construction, consistent with the response-not-mutated contract.
- MUST place EF Core query-result types in `DataAccess/QueryModels/` as `internal record class {Name}Dto`. **Rationale**: Query model DTOs are an infrastructure concern; keeping them `internal` prevents the API layer from depending on database projection shapes.

> ⚠️ **Backward dependency anti-pattern**: The test is simple — does a domain type file import a DTO namespace? If `Company.cs` has `using FeatureLibrary.Dtos;`, the domain is coupled outward. In multi-assembly projects, this is a physical build error (circular reference). In single-assembly projects, it's a logical violation that prevents clean extraction later. Conversion files are the bridge — they necessarily reference both domain types (inward) and DTO types (outward). Place them in a `Conversions/` folder or separate namespace, never inside domain type files.

### SHOULD
- SHOULD offer conversion methods from every source type that can produce the target DTO, not just the domain model. **Rationale**: When a data-access query returns a `CompanyDto` directly, converting from the DTO avoids an unnecessary round-trip through the domain model.
- SHOULD name conversion methods `ToModel()`, `ToResponseDto()`, or `ToSummaryDto()` — never `Map()` or `Convert()`. **Rationale**: Verbose, type-naming method names make it obvious at the call site exactly which target type the conversion produces.

### MAY
- MAY colocate the inline request DTO at the bottom of the feature's public interface file when the request type exists only for that single method. **Rationale**: Inline request records avoid polluting the project with one-line DTO files while keeping the binding type adjacent to its consuming method.

```csharp
// File: Infrastructure/DataAccess/Conversions/PersonConversions.cs
// ← Lives in Infrastructure layer, references Domain types (inward dependency)
internal record class PersonRow(int Id, Guid PublicId, string FirstName, string LastName);

internal static class PersonConversions
{
    extension(PersonRow dataModel)
    {
        internal Person ToModel() =>
            new Person(new PersonId(dataModel.PublicId), dataModel.FirstName, dataModel.LastName);
    }
}
```

```csharp
// File: Presentation/Api/Conversions/CompanyResponseMappings.cs
// ← Lives in Presentation layer, references Domain types (inward) and DTOs (outward)
public record class CompanyResponseDto(CompanyHandle Handle, string Name, string IncorporatedInCode, string IncorporatedInName);

public static class CompanyResponseMappings
{
    extension(Company model)  // Domain type — inward reference
    {
        public CompanyResponseDto ToResponseDto() =>
            new(model.Handle, model.Name, model.IncorporatedIn.Code, model.IncorporatedIn.Name);
    }

    extension(CompanyDto queryModel)  // Query DTO — also in Application layer
    {
        public CompanyResponseDto ToResponseDto() =>
            new(queryModel.Handle, queryModel.Name, queryModel.IncorporatedIn.Code, queryModel.IncorporatedIn.Name);
    }
}
```

> **Layer placement summary**:
> - `PersonRow → Person` conversion: Infrastructure layer (data access → domain)
> - `Company → CompanyResponseDto` conversion: Application/Presentation layer, or `Conversions/` folder in a single-assembly library (domain → API response)
> - `CompanyDto → CompanyResponseDto` conversion: Same layer as above (query DTO → API response)
> - **Never** place conversion logic inside a domain type file itself; conversion files bridge layers and reference both sides

---

## Decision Point 12: Extending a Type with Behavior (Extension Members)

C# 14 introduces `extension(TargetType)` blocks, which Zoran uses pervasively instead of traditional `static class` extension methods. An extension block lives inside a static class named `{Entity}{Role}` (e.g., `BookCreation`, `EstimateComposition`) and adds factory methods, transformations, or composition operators to an existing type — all using instance-method syntax. This pattern keeps the core type small and declarative while housing domain logic in role-specific companion classes.

### MUST
- MUST wrap all extension methods in an `extension(TargetType) { }` block inside a `public static class`. **Rationale**: The C# 14 `extension` block is the compiler-recognized syntax; traditional `static class` extension methods are obsolete in Zoran's idiom.
- MUST name the containing static class `{Entity}{Role}` (e.g., `BookCreation`, `BookManagement`, `EstimateComposition`). **Rationale**: The class name signals which entity is being extended and what role the extension fulfills.

### SHOULD
- SHOULD use factory methods named `New`, `CreateNew`, or `Restore` that return the entity type (possibly nullable). **Rationale**: Explicit factory names with nullable return communicate that construction can fail and avoid throwing from constructors.
- SHOULD prefer the instance-method calling convention enabled by `extension(TargetType target)` over traditional `this Type` parameter syntax. **Rationale**: The instance-style syntax `book.WithAuthor(a)` reads more naturally than `BookExtensions.WithAuthor(book, a)`.
- **When to use each form**: Use `extension(TargetType)` (no receiver) for factory methods (`TryCreateNew`, `New`, `Restore`) where no existing instance exists yet — these are called as `Type.Method()`. Use `extension(TargetType receiver)` (with receiver) for transformations on an existing instance — these are called as `receiver.Method()`. The rule: if you need an instance to call it, use a receiver; if it creates a new instance from nothing, omit the receiver.

### MAY
- MAY define private helper methods in a separate `extension(AuxiliaryType)` block within the same static class. **Rationale**: Auxiliary extension blocks keep type-specific helpers scoped to the extension class without polluting the target type's namespace.
- MAY define generic `extension<T>` blocks for utility methods applied to generic types. **Rationale**: Generic extension blocks apply a method to any type parameter while keeping the helper discoverable near the domain it serves.

```csharp
// Requires: using System.Collections.Immutable;
internal record Book(Guid Id, string Title, ImmutableList<Author> Authors);

public static class BookCreation
{
    extension(Book)
    {
        public static Book New(string title) =>
            new Book(Guid.NewGuid(), title.AsValidTitle(), []);

        public static Book New(string title, IEnumerable<Author> authors) =>
            new Book(Guid.NewGuid(), title.AsValidTitle(), authors.ToImmutableList());

        public static Book Restore(Guid id, string title, IEnumerable<Author> authors) =>
            new Book(id, title.AsValidTitle(), authors.ToImmutableList());
    }

    extension(string title)
    {
        private string AsValidTitle() =>
            string.IsNullOrWhiteSpace(title) ? throw new ArgumentException("Title cannot be empty")
            : title;
    }
}
```

```csharp
public static class EstimateComposition
{
    extension(Estimate estimate)
    {
        public Estimate Add(Estimate other) => (estimate, other) switch
        {
            (Duration d1, Duration d2) => Estimate.CreateDuration(d1.Value + d2.Value),
            (Duration d, Interval i) => Estimate.CreateInterval(i.Start + d.Value, i.Span),
            (Interval i, Duration d) => Estimate.CreateInterval(i.Start + d.Value, i.Span),
            (Interval i1, Interval i2) => Estimate.CreateInterval(i1.Start + i2.Start, i1.Span + i2.Span),
            _ => Estimate.Unknown
        };

        public Estimate InParallelWith(Estimate other) =>
            // NOTE: This example assumes an extended Estimate model with Optimistic/Pessimistic
            // properties and FromNullable/NullableMax helpers. See the full source for the complete model.
            Estimate.FromNullable(
                TimeSpan.NullableMax(estimate.Optimistic, other.Optimistic),
                TimeSpan.NullableMax(estimate.Pessimistic, other.Pessimistic));
    }
}
```

---

## Decision Point 13: Handling Concurrency / Mutable State in an Immutable World

**Advanced pattern for aggregates requiring deep immutability** (event sourcing, strict concurrent editing). For most entities, prefer the `{ get; private set; }` mutation pattern in Decision Point 2. Immutable domain models avoid accidental mutation but clash with Entity Framework Core's change tracking. The bridge is a copy-constructor pattern: each `With*` method copies the object via a private copy-constructor and sets only the changed property via `{ get; init; }`. For EF Core persistence, one approach is an `UpdateImmutable<T>()` extension method that detaches the original tracked entity, copies scalar/complex property values from the modified copy, and recursively aligns collection and reference navigation graphs using key-based matching.

### MUST
- MUST define a private copy-constructor that shallow-copies all properties from another instance of the same type. **Rationale**: Provides a single, auditable copy point that every `With*` method delegates to.
- MUST implement mutation as `With*` methods that return a new instance via the copy-constructor with only the target property changed. **Rationale**: Preserves immutability at the call site — existing references are never mutated.
- MUST use `{ get; init; }` on all properties of immutable aggregates, never `{ get; set; }`. **Rationale**: `init` enables the copy-constructor and `With*` methods to set properties during initialization while blocking post-construction mutation.
- MUST ensure EF Core persistence of immutable records uses an `UpdateImmutable<T>()` extension that detaches the original, copies values from the modified copy, and aligns navigation graphs. **Rationale**: EF Core's change tracker cannot detect property changes on immutable types; manual graph alignment is required.

### SHOULD
- SHOULD validate invariants spanning multiple properties in the `init` accessor body of the collection property itself (e.g., verifying all items share the same currency as the parent). **Rationale**: Co-locates cross-property validation with the data it guards. Unlike property initializers (`= value ?? throw`), `init` accessor bodies (`init { ... }` or `init =>`) do fire during copy-constructor-based initialization (`new(this) { ... }`). For single-property validation, prefer the C# 14 `field` keyword: `init => field = value ?? throw` — this fires for both construction and copy-constructor paths.

### MAY
- MAY use `ImmutableList<T>` for collection properties in deep-immutable aggregates. **Rationale**: Prevents accidental mutation of the backing collection from any reference holder. However, `ImmutableList<T>` has O(log n) indexed access, ~40 bytes per element memory overhead, and is not natively supported as an EF Core navigation type — EF Core cannot initialize or materialize `ImmutableList<T>` automatically (dotnet/efcore issue #21176, open since 2020). The recommended pattern for EF Core aggregates is the `private List<T>` backing field with `public IReadOnlyCollection<T>` accessor shown in Decision Point 2.
- MAY use reflection-based key extraction in `UpdateImmutable<T>()` for generic collection alignment. **Rationale**: Avoids writing per-entity-type sync logic. The name `UpdateImmutable` is misleading — the pattern mutates collections inside EF Core's change tracker, defeating the purpose of immutability. Consider naming it `SyncCollection<T>` or `ApplyCollectionDiff<T>`.

```csharp
internal class Invoice(
    InvoiceNumber number, string customerName, DateOnly invoicedOn,
    InvoiceStatus status, Currency currency)
{
    private Invoice(Invoice other)
        : this(other.Number, other.CustomerName, other.InvoicedOn, other.Status, other.Currency)
    {
        Id = other.Id;
        PublicId = other.PublicId;
        Lines = other.Lines;
    }

    private int Id { get; init; }
    public Guid PublicId { get; private init; } = Guid.NewGuid();
    public InvoiceNumber Number { get; init; } = number;
    public string CustomerName
    {
        get;
        init => field = !string.IsNullOrEmpty(customerName) ? customerName
            : throw new ArgumentException("Customer name cannot be null or empty.", nameof(customerName));
    } = customerName;

    private List<InvoiceLine> _lines = new();
    public IReadOnlyCollection<InvoiceLine> Lines
    {
        get => _lines.AsReadOnly();
        init
        {
            if (value.Any(line => line.UnitPrice.Currency != Currency))
                throw new ArgumentException("Mismatched currencies found in invoice lines.");
            _lines = value.ToList();
        }
    }

    public Invoice WithCustomerName(string customerName) =>
        new(this) { CustomerName = customerName };

    public Invoice WithLines(IReadOnlyCollection<InvoiceLine> lines) =>
        new(this) { Lines = lines };
}
```

---

## Decision Point 14: Setting Up DI / Project Wiring

DI follows a minimal recipe: register an `IDbContextFactory<T>` and nothing else for data access. Feature classes receive the factory, create a short-lived `DbContext` per unit of work via `await using`, and dispose it promptly. No repository interfaces, no service locator, no extra abstraction layers. The container never calls back into itself — there is no `IServiceProvider` argument and no `GetService<T>()` outside the composition root.

### MUST
- Register EF Core `DbContext` with `AddDbContextFactory<T>()` and a connection-string-derived options callback. **Rationale**: The factory pattern gives explicit control over context lifetime; each unit of work creates a fresh, short-lived `DbContext` that is disposed when the `using` block exits, preventing long-lived change tracker accumulation.
- Create a new `DbContext` per unit of work with `await using var db = await _contextFactory.CreateDbContextAsync();`. **Rationale**: Short-lived contexts prevent stale tracked entities and reduce memory pressure from accumulated change tracker entries.
- Enable `Nullable` and `ImplicitUsings` in every `.csproj`. **Rationale**: Nullable guards force explicit null-handling decisions; implicit usings eliminate boilerplate `using` lines for `System`, `System.Linq`, etc.
- Use only parameter injection in consuming classes — never pass `IServiceProvider` into a class. **Rationale**: Parameter injection makes dependencies explicit in the method signature; service location hides them.
- Use `System.Text.Json` exclusively for all JSON serialization — never add a `Newtonsoft.Json` package reference or use `JsonConvert`. **Rationale**: `System.Text.Json` is built into .NET, requires zero external dependencies, and has been the standard since .NET Core 3.0.
- Leverage .NET 10 `JsonSerializerOptions` defaults: `AllowDuplicateKeys` now defaults to `false` (rejecting duplicate JSON keys), and the new `JsonSerializerOptions.Strict` property enables maximum validation for API boundaries. **Rationale**: Strict defaults catch malformed input at deserialization time rather than letting invalid data reach domain logic.

### SHOULD
- Use primary constructors for DI-injected classes whose parameters map directly to stored fields. **Rationale**: Removes the ceremony of private readonly field declarations and assignment-only constructor bodies.
- SHOULD use `TypedResults` with explicit return type unions (`Results<Ok<T>, ProblemHttpResult>`) when OpenAPI documentation is required. **Rationale**: Typed return values enable `[Produces]` attribute metadata generation and compile-time verification that all response types are handled.

### MAY
- MAY use `AddPooledDbContextFactory<T>()` instead of `AddDbContextFactory<T>()` when benchmarks prove pooling reduces allocation pressure. **Rationale**: Pooling recycles context instances to reduce GC overhead; however, Microsoft recommends only switching when performance testing shows a real benefit. The factory pattern and `using` usage pattern remain identical — only the registration method changes.
- MAY centralize API error responses in a static `Problem` class with domain-specific factory methods (`Problem.CompanyNotFound()`, `Problem.InvalidRequestFields(...)`). **Rationale**: Eliminates duplicated `TypedResults.Problem()` calls with magic strings; makes the set of possible errors discoverable by scanning one file.

```csharp
using Microsoft.EntityFrameworkCore;
using Tbd.WebApi.Api;
using Tbd.WebApi.DataAccess;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenApi();
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.Converters.Add(new JsonStringEnumConverter());
    options.SerializerOptions.Converters.Add(new CompanyHandleConverter());
});

builder.Services.AddDbContextFactory<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.UseHttpsRedirection();

app.MapGet("/api/companies", Companies.GetAllAsync);
app.MapPost("/api/companies", Companies.PostAsync);
app.MapGet("/api/companies/{handle}", Companies.GetSingleAsync);

app.Run();

namespace Tbd.WebApi { public partial class Program { } }
```

---

## Performance Considerations

- **`ImmutableList<T>`**: O(log n) indexed access, ~40 bytes per element overhead, ~100x slower `ToImmutableList()` vs `AsReadOnly()`. Prefer `private List<T>` + `public IReadOnlyCollection<T>` for EF Core collection navigations. See Decision Point 2 for the recommended collection pattern.
- **Copy-constructor pattern**: Allocates a new object per mutation. Acceptable for domain operations (infrequent); avoid in hot paths. See Decision Point 13.
- **`UpdateImmutable<T>()`**: Reflection-based collection alignment. Acceptable for small collections; write explicit per-aggregate sync for large graphs. See Decision Point 13.
- **`{ get; init; }` with ternary + throw**: Executes validation once at construction — zero ongoing cost. See Decision Point 1.
- **EF Core 10 split-query ordering**: EF Core 10 fixes split-query ordering inconsistencies that could return incorrect data under `Take()`/`Skip()` — no code changes needed; just upgrade. See Decision Point 9.
- **EF Core 10 parameterized-collection padding**: EF Core 10 defaults to multi-parameter translation for `Contains()` on collections, with list padding to reduce SQL plan cache bloat. See Decision Point 9.

## Extrapolating to New Domains

When you encounter a concept not shown in the examples, derive the correct pattern from the meta-principles:

**To model a new value type** (e.g., `EmailAddress`, `GeoCoordinate`, `PhoneNumber`):
1. Declare an `internal record` with a positional primary constructor (use `public record` only in a shared kernel assembly — see Decision Point 6)
2. Validate in the `init` accessor body using the C# 14 `field` keyword for single-property validation (`init => field = value ?? throw`); for cross-property validation, remove `init` entirely (`{ get; }` only) to force construction through the constructor where all parameters are in scope
3. Use `nameof()` in all exceptions
4. If construction can fail for user input, add a `TryCreate()` factory returning `T?` (use `TryCreateNew()` for aggregate roots — see Decision Point 2)
5. If the type needs arithmetic or comparison, define operator overloads

**To model a new state machine** (e.g., `OrderStatus`, `PaymentState`):
1. Root: `internal abstract record BaseType;`
2. Variants: prefer non-positional record syntax with an explicit parameterless constructor for EF Core materialization and a domain constructor for normal instantiation. Positional records (`sealed record VariantName(DateTime Date) : BaseType`) work for primitive-only parameters in EF Core 8+ but fail when variants carry complex-type parameters (issue #31621, still open), create `with`-expression identity-tracking conflicts, and trigger a compiler-generated copy constructor that adds noise to EF Core binding errors. See Decision Point 8 for the full pattern and TPH configuration.
3. Group capabilities with marker interfaces
4. All variant dispatch via exhaustive switch pattern matching
5. For value-based dispatch within states, use property patterns with `when` guards (Decision Point 4)

**To model a new aggregate** (e.g., `ShoppingCart`, `Reservation`):
1. `internal` constructor + `static class XxxConstruction { extension(Xxx) { ... } }`
2. `{ get; private set; }` for mutable state, `{ get; init; }` for immutable
3. Private `List<T>` backing field + public `IEnumerable<T>` accessor for collections
4. `TryCreateNew()` returning `T?`, `TryRestore()` for persistence reconstitution
5. If some states forbid mutation, add `InEditable<TValue>()` guard on mutable setters

**To map a new value object to EF Core**:
1. Value object stored in single column → `HasConversion()` with `ValueConverter` class or inline lambda
2. Value object with sub-fields stored in parent table → `ComplexProperty()` with nested column config
3. Value object used as `ComplexProperty` → add `private` parameterless constructor with `default!` chaining
4. If the value object wraps `DateTime` → reapply `DateTimeKind` in the converter (SQL Server strips it)
5. If object properties depend on other properties → null-conditional guard for EF Core materialization ordering

**To map a new state machine to EF Core** (persisting a discriminated union like `OrderStatus` or `PaymentState`):
1. Use entity-level TPH inheritance with `HasDiscriminator<string>()` — declare a custom discriminator column name (avoid the default `"Discriminator"` column name on EF Core 10 due to [regression #37571](https://github.com/dotnet/efcore/issues/37571); fixed in 10.0.5). Call `UseTphMappingStrategy()` explicitly on the root entity for clarity.
2. Use non-positional record syntax for variants (see "To model a new state machine" above) — provides a parameterless constructor for EF Core materialization and avoids complex-type parameter binding failures (issue #31621).
3. Register ALL derived variant types in the model — EF Core does not auto-discover subtypes. Each variant needs its own `IEntityTypeConfiguration<T>` (or inline `modelBuilder.Entity<Variant>(...)`).
4. Map same-named properties across sibling variants to shared columns via `HasColumnName("StateDate1")` — without this, EF Core creates separate prefixed columns per variant.
5. Add a concurrency token (`IsRowVersion()`) — state machines with concurrent transitions have race conditions. EF Core throws `DbUpdateConcurrencyException` on version mismatch.
6. Prefer `DateTimeOffset` for `DateTime` properties — the database stores the UTC offset automatically, eliminating the need for a per-property `ValueConverter<DateTime, DateTime>` to reapply `DateTimeKind.Utc`. If `DateTime` is unavoidable, apply the converter globally via `ConfigureConventions` instead of per-property.
7. Consider a separate state table with an FK back to the aggregate, rather than embedding the TPH hierarchy directly in the aggregate table — keeps the aggregate lean and the state table independently testable.
8. Do NOT use `ComplexProperty` + `ToJson()` or `ValueConverter<string>` with `JsonSerializer` for entity state machines — EF Core 10 does not support complex type inheritance (issue #31250), and JSON serialization prevents SQL-level queries on variant-specific properties.

**When in doubt between `record` and `class`**:
- Prefer `record` — for values, DTOs, and discriminated unions
- Use `class` only when internal mutation via `{ get; private set; }` is required, or for inheritance hierarchies with behavior
- Prefer `init` over `set` — even on classes
