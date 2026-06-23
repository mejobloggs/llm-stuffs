---
name: zoran
description: Generate C# code in Zoran Horvat's style
---


You are generating C# code in the style of Zoran Horvat — a C# architect known for functional domain modeling, discriminated unions, and extreme discipline around value objects, immutability, and explicit domain boundaries.

---

## Meta-Principles

These five principles encode the philosophy underlying every specific rule. When a situation arises that no rule explicitly covers, apply these principles.

### Principle 1: "No primitive passes a domain boundary untyped."

Every `string`, `Guid`, `decimal`, and `DateTime` crossing between layers must be wrapped in a named type. `CompanyHandle` is never a `string`. `AccountId` is never a `Guid`. `Timestamp` is never a `DateTime`. A method signature of `(Guid, Guid, decimal)` tells you nothing. A signature of `(AccountId, EmployeeId, Money)` tells a story.

### Principle 2: "Invalid objects must be unrepresentable."

Validation happens in construction — not in a separate `Validate()` method called later. The type system itself guarantees correctness: a `Money` instance can never hold a negative amount; an `AccountId` can never be `Guid.Empty`; a `Timestamp` can never be non-UTC. If you can hold an invalid object in memory, you will eventually process it. Make it impossible to create.

### Principle 3: "Public constructors are a last resort — use factories."

Domain types have `internal` constructors. Construction happens through `static class XxxConstruction { extension(Xxx) { ... } }` factory methods returning `T?` (null means validation failure) or throwing on truly invalid input. Factories can fail gracefully, accumulate errors, or perform cross-field validation that single-property `init` accessors cannot.

### Principle 4: "Tell the compiler what you know — not what the database knows."

Entities declare their intent via types (discriminated unions, state machines with abstract records), not flags (`bool IsApproved`, `enum Status`). Use `abstract record` + sealed variants for finite states. Use property patterns (`switch` with `when` guards) for value-based dispatch. Use `IEntityTypeConfiguration` to bridge the gap between domain types and database columns. The domain model drives the design. The database conforms to the domain — never the reverse.

### Principle 5: "One concept, one file, one public type."

No `Utils.cs`, no `Helpers.cs`, no `Constants.cs`, no `Enums.cs`. Every concept gets its own file. Even small types like `Temperature`, `Humidity`, `AccountId` earn dedicated files. A developer looking for `AccountId` finds it immediately — not buried inside a `Common.cs` catch-all.

---

> **Companion types**: Many code examples reference types (`Currency`, `InvoiceStatus`, `InvoiceNumber`, `Money`, `EmployeeId`, `TransferTimestamp`, `AirExchange`, `Temperature`, `Sensors`, `AppDbContext`, `TransferResult`, `TransferRequestDto`, `TransferSummaryDto`, `AccountIdDto`, `Passed<T,TProblem>`, `Failed<T,TProblem>`, `IncompleteApproval`, `ApprovedTransfer`, `ICompletedApproval`, `IIncompleteApproval`) that are not defined in this file. These are companion types assumed to exist per the patterns demonstrated in their respective decision points. When generating code, create these types following the same conventions shown here.

## Decision Point 1: Creating a Value Object

Value objects wrap primitive types in domain-meaningful types that enforce invariants at construction, making invalid states unrepresentable. Use C# records for structural equality semantics; use `{ get; init; }` with ternary + throw in the property initializer so validation fires before the object materializes. Static factory methods provide well-known instances (`Empty`, `UtcNow`, `NewId()`) while keeping the primary constructor internal for deserialization. Never add implicit conversion operators — they hide allocation and bypass validation.

### MUST
- MUST declare value objects as `internal record` with a primary constructor wrapping the underlying primitive. **Rationale**: Records deliver structural equality and `{ get; init; }` value semantics out of the box. Value objects are internal to their feature library (see Decision Point 6); value objects shared across multiple features belong in a shared kernel assembly where they may be `public record`.
- MUST validate in the `{ get; init; }` property initializer using ternary + throw, never in a separate `Validate()` method or `IsValid` property on domain objects. **Rationale**: The object must never exist in an invalid state; construction-time failure is fail-fast. (UI-layer types such as view models may legitimately expose `IsValid` for displaying validation errors before submission — this rule applies to the domain model.)
- MUST include `nameof()` in every `ArgumentException` call that refers to a parameter name. **Rationale**: Eliminates magic strings that silently rot under rename refactoring.

### SHOULD
- SHOULD provide a static `NewId()` factory method for ID value objects that wraps `Guid.NewGuid()`. **Rationale**: Centralizes GUID generation; makes intent explicit at the call site.
- SHOULD provide a well-known `Empty` sentinel property for ID types that wraps `Guid.Empty`. **Rationale**: Eliminates null checks and provides an explicit, searchable "no value" representation.
- SHOULD include more than a single field in value objects when additional data naturally clusters with the core value (e.g., `Iso4217Currency` in `Money`). **Rationale**: Co-locating related data prevents primitive obsession and enables domain arithmetic without back-referencing external data.
- SHOULD bundle domain rules into the value object itself — not in a separate service. Example: `Currency` carries `DecimalPlaces` (called `MinorUnit` in ISO 4217) so `Money` can auto-round without consulting an external configuration. **Rationale**: The value object becomes both identity and policy carrier; callers never need to know which currency has which precision.

### MAY
- MAY override `ToString()` for human-readable display of truncated identifiers. **Rationale**: Improves log and debug readability without exposing internal structure.
- MAY define operator overloads for comparable types like timestamps and monetary amounts. **Rationale**: Enables natural comparison syntax (`ts1 > ts2`, `m1 + m2`) without forcing callers to unwrap primitive values. Note: operator overloads survive EF Core → SQL translation, enabling `db.Transfers.Where(t => t.ExecutedAt >= from)`.

<!-- File: code-ef-core-complex-properties/.../TransferId.cs -->

```csharp
internal record TransferId(Guid Id)
{
    public Guid Id { get; init; } =
        Id != Guid.Empty ? Id
        : throw new ArgumentException("TransferId cannot be empty");

    public static TransferId NewId() => new TransferId(Guid.NewGuid());

    public override string ToString() =>
        Id.ToString()[..3] + "…" + Id.ToString()[^3..];
}
```

<!-- File: code-immutable-vs-mutable-persistence/.../AccountId.cs -->

```csharp
internal record AccountId(Guid Id)
{
    public Guid Id { get; init; } =
        Id != Guid.Empty ? Id
        : throw new ArgumentException("Account ID must be a non-empty GUID.", nameof(Id));

    public static AccountId? TryCreate(Guid? id) =>
        id is Guid g && g != Guid.Empty ? new AccountId(g) : null;
}
```

<!-- File: code-immutable-vs-mutable-persistence/.../Iso4217Currency.cs -->

```csharp
internal record Iso4217Currency(string AlphabeticCode, int NumericCode, int MinorUnit)
{
    public string AlphabeticCode { get; init; } =
        !string.IsNullOrWhiteSpace(AlphabeticCode) && AlphabeticCode.Length == 3 && AlphabeticCode.All(char.IsUpper) ? AlphabeticCode
        : throw new ArgumentException("Alphabetic code must be a three-letter uppercase string.", nameof(AlphabeticCode));

    public int NumericCode { get; init; } =
        NumericCode > 0 && NumericCode <= 999 ? NumericCode
        : throw new ArgumentException("Numeric code must be a three-digit number between 001 and 999.", nameof(NumericCode));

    public int MinorUnit { get; init; } =
        MinorUnit >= 0 ? MinorUnit
        : throw new ArgumentException("Minor unit must be a non-negative integer.", nameof(MinorUnit));
}
```

---

## Decision Point 2: Creating a Domain Aggregate/Entity

Aggregates enforce consistency boundaries through controlled mutation paths. Constructors are `internal` to force construction through static factory methods; this guarantees invariants are checked and normalization is applied (e.g., string trimming) before an instance exists. Mutable state uses `{ get; private set; }` while immutable one-time-set properties use `{ get; init; }`. Private collection backing fields paired with a public `IEnumerable<>` accessor prevent external mutation of child entity collections.

### MUST
- MUST declare entity constructors as `internal`, never `public`. **Rationale**: Forces all construction through controlled factory methods that can normalize input and reject invalid combinations.
- MUST use `{ get; private set; }` for mutable state and `{ get; init; }` for immutable one-time-set properties. **Rationale**: Makes mutation intentional and scoped; `private set` exposes mutation only to the aggregate itself.
- MUST back collection properties with a private `List<T>` field and expose a public `IEnumerable<T>` or `IReadOnlyCollection<T>` accessor. **Rationale**: Prevents external code from adding, removing, or replacing child entities without going through aggregate methods.

### SHOULD
- SHOULD provide a static `XxxConstruction` class containing `extension(Xxx)` factory methods for construction and persistence reconstitution. **Rationale**: Separates construction logic from the aggregate itself, keeping the type focused on behavior rather than factory boilerplate.
- SHOULD name factory methods `TryCreateNew()` (returning `T?` for user-input validation) and `TryRestore()` (returning `T?` for persistence reconstitution). **Rationale**: The naming convention distinguishes fresh creation from database reconstitution; nullable return signals soft failure to the caller.
- SHOULD place nested ID record structs inside the aggregate class. **Rationale**: Scopes the ID type to its owning aggregate and avoids namespace pollution.
- SHOULD gate mutable setters behind a status check (`InEditable<TValue>()`) when some states forbid mutation (e.g., an issued invoice cannot be modified). **Rationale**: Encodes the state machine's transition rules at the property level, making it impossible to mutate a property in an invalid state rather than checking elsewhere.

### MAY
- MAY use `required` init-only properties instead of constructor parameters for optional aggregate fields. **Rationale**: Allows callers to set only the properties they care about while keeping the constructor manageable.

<!-- File: code-record-types-validation/.../Person.cs -->

```csharp
internal record class Person
{
    internal Person(Guid publicId, string firstName, string lastName) =>
        (PublicId, FirstName, LastName) = (publicId, firstName, lastName);

    public Guid PublicId { get; }
    public string FirstName { get; }
    public string LastName { get; }
}

public static class PersonConstruction
{
    extension(Person)
    {
        public static Person? CreateNew(string firstName, string lastName) =>
            Person.CreateValid(Guid.NewGuid(), firstName, lastName);

        public static Person? Restore(Guid publicId, string firstName, string lastName) =>
            Person.CreateValid(publicId, firstName, lastName);

        private static Person? CreateValid(Guid publicId, string firstName, string lastName) =>
            string.IsNullOrWhiteSpace(firstName) ? null
            : new(publicId, firstName.Trim(), lastName?.Trim() ?? string.Empty);
    }
}
```

<!-- File: code-ef-core-natural-keys/.../Invoice.cs — InEditable guard pattern -->

```csharp
internal class Invoice(
    InvoiceNumber number, string customerName, DateOnly invoicedOn,
    InvoiceStatus status, Currency currency)
{
    public string CustomerName
    {
        get;
        set => InEditable(AsValidCustomerName(value));      // Gate: rejected if not Editing
    } = AsValidCustomerName(customerName);

    public Currency Currency { get; set => field = InEditable(value); } = currency;

    private TValue InEditable<TValue>(TValue value) =>
        Status == InvoiceStatus.Editing ? value
        : throw new InvalidOperationException("Cannot modify issued invoice.");
}
```

<!-- File: code-ef-core-ddd-patterns/.../Invoice.cs -->

```csharp
internal class Invoice
{
    public readonly record struct InvoiceId(Guid Value)
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
- SHOULD provide static factory methods in an `XxxConstruction` class that validate inputs before constructing variants. **Rationale**: Separates validation from the record declarations, keeping variant types clean while preventing invalid states.
- SHOULD implement state transitions on the variants themselves as methods returning the abstract root type. **Rationale**: Each variant knows its own valid transitions; returning the root type ensures callers always pattern-match the result.

### MAY
- MAY colocate all variants in a single file — suitable when variants are simple data carriers with no behavioral logic. When variants carry substantial transition logic (multiple methods, guard clauses, helper logic), split each into its own file following Principle 5 (one concept, one file). **Rationale**: Simple unions benefit from at-a-glance visibility; behavioral variants deserve dedicated files for testability and focus.
- MAY use abstract classes instead of abstract records when mutable state is required in certain variants. **Rationale**: Records are naturally immutable; abstract classes support `{ get; private set; }` when a variant legitimately requires mutation.
- MAY use partial class wrappers when a variant needs to carry substantial behavioral logic. **Rationale**: Keeps the union declaration file focused on type declarations alone.

<!-- File: code-cs14-discriminated-unions/.../Estimate.cs -->

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

<!-- File: code-cs14-discriminated-unions/.../EstimateComposition.cs — exhaustive tuple-pattern switch -->

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

<!-- File: code-immutable-vs-mutable-persistence/.../FourEyesApproval.cs -->

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

<!-- File: code-refactoring-anemic-model/.../Transfer.cs — abstract class state machine with marker interfaces -->

```csharp
internal abstract class Transfer(
    Money amount, Guid id, AccountId from, AccountId to,
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

Property patterns handle value-based dispatch — the counterpart to discriminated unions' type-based dispatch. Use nested `switch` expressions with property patterns (`{ State: State.Heating }`), tuple patterns (`(device, sensors) switch`), `when` guards for threshold tests, and the `not` pattern for negation. These patterns composes naturally into state-machine decision chains that a reader can verify exhaustively.

### MUST
- MUST use `switch` expressions (not statements) for value-based pattern matching. **Rationale**: Expressions return a value and guarantee exhaustiveness; statements can silently fall through.
- MUST use property patterns to destructure records in switch arms: `{ State: State.Off }`, `{ Temperature: var temp }`. **Rationale**: Property patterns extract exactly the fields needed for the decision without verbose constructor destructuring.

### SHOULD
- SHOULD use tuple patterns `(a, b) switch { ... }` when dispatching on two or more values simultaneously. **Rationale**: Tuple patterns let the compiler check exhaustiveness of the cross-product, catching combinations an if/else chain would miss.
- SHOULD use `when` guards for threshold and range checks: `when temp < idealTemperature.low`. **Rationale**: Guards keep the pattern shape clean while pushing numeric comparison logic into the guard clause where it belongs.
- SHOULD use `not` patterns for negation: `{ State: not State.Heating }`. **Rationale**: `not` is more readable than a catch-all `_` arm and documents the specific state being excluded.

### MAY
- MAY use nested records inside the same file as the pattern-matching logic when those records exist only to feed the state machine. **Rationale**: Keeps the domain simulation self-contained without polluting the project namespace.

<!-- File: code-property-patterns/.../ComplexDomainDemo.cs -->

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

Three distinct failure-handling tiers exist. Hard failures — where input is provably corrupt (e.g., empty GUID, negative amount) — use ternary + throw in property initializers; there is no recovery path. Soft failures — where user input may be invalid — use `TryCreateNew()` returning `T?` (null signals invalid). For multi-step validation chains where all errors must accumulate rather than short-circuit, the `Checked<T, TProblem>` monad (see MAY below) is available.

### MUST
- MUST use ternary + throw in property initializers for invariants that are never expected to fail under correct system operation (e.g., non-empty GUID, non-negative amount). **Rationale**: These are programming errors, not recoverable input errors; the stack trace pinpoints the bug immediately.
- MUST NOT define `bool IsValid { get; }` properties or `void Validate()` methods called after construction on domain objects. **Rationale**: Post-construction validation creates a window where an invalid object exists; construction-time validation closes that window entirely. UI-layer types such as view models may legitimately expose `IsValid` for displaying validation errors before submission — this rule applies to the domain model.

### SHOULD
- SHOULD use `TryCreateNew()` returning `T?` for user-input scenarios where failure is expected and recoverable. **Rationale**: Nullable return signals "try again" to the caller without throwing exceptions for normal control flow.

### MAY
- MAY use the `Checked<T, TProblem>` monad when multiple validation steps must compose without short-circuiting. **Rationale**: Monadic `Bind()` chains accumulate all problems before returning, unlike exception-throwing which aborts at the first failure. Teams unfamiliar with monadic composition may prefer simpler alternatives such as `Result<T>` or accumulated error lists.
- MAY provide a `Match()` method on the checked monad for explicit success/failure branching at the consumption site. **Rationale**: Forces callers to handle both cases explicitly, preventing null-reference errors.
- MAY accept a generic `TProblem` type (enum, string, custom record) in the checked monad. **Rationale**: Different domains may need different failure representations (error codes vs. messages vs. structured error objects).
- MAY define monadic `extension(T?)` blocks with `Bind`, `Map`, `MapValue`, and `Match` for nullable types, both class and struct. See `code-csharp-monadic-types/final/Demo/NullableMonad.cs`. **Rationale**: Treating `T?` as a monad enables fluent pipelines on nullable values without repeated null checks.

<!-- File: code-separation-of-concerns/.../Checked.cs -->

```csharp
abstract record Checked<T, TProblem>(T Value)
{
    public static Checked<T, TProblem> Unit(T value) => new Passed<T, TProblem>(value);
    public static Checked<T, TProblem> Failed(T value, TProblem problem) => new Failed<T, TProblem>(value, problem);

    public abstract Checked<T, TProblem> Bind(Func<T, Checked<T, TProblem>> func);
    public abstract Checked<U, TProblem> Bind<U>(Func<T, Checked<U, TProblem>> func, U orElse);
    public abstract Checked<U, TProblem> Map<U>(Func<T, U> func);
    public abstract U Match<U>(Func<T, U> onPassed, Func<T, TProblem, U> onFailed);
}
```

<!-- File: code-separation-of-concerns/.../FileSystemRunner.cs — monadic Bind chain -->

```csharp
string report = Locate(new FileInfo(filePath))              // Checked<FileInfo, string>
    .Bind(Read, [])                                          // Checked<string[], string>
    .Bind(CheckLength)                                       // Checked<string[], string>
    .Bind(CheckSorted)                                       // Checked<string[], string>
    .Bind(CheckGrowingLines)                                 // Checked<string[], string>
    .Match(
        onPassed: lines => $"File is valid with {lines.Length} lines.",
        onFailed: (lines, problem) => $"File check failed: {problem}"
    );
```

<!-- File: code-csharp-monadic-types/.../NullableMonad.cs -->

```csharp
static class NullableMonad
{
    extension<T>(T? value) where T : class
    {
        public U? Bind<U>(Func<T, U?> func) where U : class =>
            value is null ? null : func(value);

        public U Match<U>(Func<T, U> onValue, Func<U> onNull) =>
            value is null ? onNull() : onValue(value);
    }

    extension<T>(Nullable<T> value) where T : struct
    {
        public Nullable<U> Bind<U>(Func<T, Nullable<U>> func) where U : struct =>
            value.HasValue ? func(value.Value) : new Nullable<U>();

        public U Match<U>(Func<T, U> onValue, Func<U> onNull) =>
            value.HasValue ? onValue(value.Value) : onNull();
    }
}
```

---

## Decision Point 6: Designing a Feature Library's Public API

A feature ships as a self-contained class library whose "API" is the set of public interfaces and extension methods a consumer sees — not HTTP routes. Every domain model, entity, value object, EF Core configuration class, and internal service is `internal`. The consumer adds the feature with a single `services.AddFeatureXxx()` call and interacts through one or two public interfaces.

### MUST
- MUST expose exactly one or two `public interface` types as the consumer's sole entry point to the feature. Example: `IMoneyTransferService`, `IInvoiceGenerator`. **Rationale**: One interface tells the consumer everything the feature can do. Two when read and write paths have meaningfully different contracts.
- MUST keep domain models, entities, value objects, EF Core configuration, and internal services as `internal`. **Rationale**: The domain is the feature's private implementation. Exposing domain types to consumers couples them to internal representations; changes to domain types should never break consumers. If a value object (e.g., `Money`, `Timestamp`) is shared across multiple feature libraries, extract it into a shared kernel assembly — but within a single feature, domain types remain internal.
- MUST provide an `IServiceCollection` extension method that registers the entire feature in one call: `services.AddTransfers(configuration)`. **Rationale**: A single registration call replaces the consumer's need to understand the feature's internal dependency graph — one `AddXxx()` and the feature is wired.
- MUST return DTOs and result records from public interface methods — never domain entities or value objects. **Rationale**: DTOs form an explicit, versionable contract between the feature and its consumers. Returning a domain object leaks the internal model and breaks consumers when the model evolves.
- MUST validate inputs at the public interface boundary before they reach domain logic. **Rationale**: The public API is the trust boundary between the outside world and the domain's make-invalid-unrepresentable guarantees; `.TryCreate()` calls and null checks belong here, not buried in internal services.

### SHOULD
- SHOULD use the Options pattern (`IOptions<FeatureOptions>`) for feature configuration, with static defaults so `AddTransfers()` works with zero additional setup. **Rationale**: Options classes self-document every configurable knob; sensible defaults mean the consumer overrides only what they need.
- SHOULD design public interface methods as stateless pure functions of their parameters — pass `DbContext` or other dependencies at the implementation level, not on the interface itself. **Rationale**: Stateless signatures are trivially testable; stateful interface methods hide side effects and make callers guess which method must be called first.
- SHOULD colocate the DI registration extension method inside the feature library itself, in the `Microsoft.Extensions.DependencyInjection` namespace. **Rationale**: The library owns its registration contract; the consumer only needs a `using` directive and a package reference.
- SHOULD return `Task<T?>` or a result union from methods where failure is expected and recoverable — never throw across the public API boundary for business-rule rejections. **Rationale**: Exceptions are for programmer errors; nullable returns and result types are for "this input wasn't valid, try again" — they force the caller to confront the failure path.

### MAY
- MAY use `InternalsVisibleTo` for the feature's test project only — never for sibling feature libraries. **Rationale**: Tests need internal access to arrange state and verify invariants; sibling features communicate exclusively through the public API to maintain decoupling.
- MAY provide a separate `.Abstractions` project containing only interfaces and DTOs when the feature is consumed across process boundaries or by multiple alternative implementations. **Rationale**: An abstractions package lets consumers reference contracts without pulling in the implementation's full transitive dependency graph.
- MAY define a `FeatureXxxOptions` record with a static `Default` property and reasonable initial values. **Rationale**: A default instance enables zero-configuration `AddTransfers()` calls while the Options pattern still allows override from `appsettings.json` or environment variables.

<!-- File: FeatureLibraries/Transfers/IMoneyTransferService.cs — the single public entry point -->

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

<!-- File: FeatureLibraries/Transfers/TransferService.cs — internal implementation -->

```csharp
internal sealed class TransferService(
    AppDbContext db,
    IOptions<TransferOptions> options) : IMoneyTransferService
{
    public async Task<TransferResult?> RequestTransferAsync(
        TransferRequestDto request, CancellationToken ct)
    {
        // Validate at the public boundary — convert DTOs to domain types
        var from = AccountId.TryCreate(request.FromAccountId);
        var to = AccountId.TryCreate(request.ToAccountId);
        var amount = Money.TryCreate(request.Amount, request.CurrencyCode);

        if (from is null || to is null || amount is null)
            return null;   // Soft failure — caller decides how to present the error

        var transfer = Transfer.TryCreateNew(from, to, amount, options.Value.MaxTransferAmount);
        if (transfer is null) return null;

        db.Transfers.Add(transfer);
        await db.SaveChangesAsync(ct);
        return TransferResult.FromTransfer(transfer);
    }
    // ...
}
```

<!-- File: FeatureLibraries/Transfers/ServiceCollectionExtensions.cs — one-call registration -->

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
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("TransfersDb")));
        return services;
    }
}
```

<!-- File: FeatureLibraries/Transfers/TransferOptions.cs — self-documenting configuration -->

```csharp
public record TransferOptions
{
    public static TransferOptions Default => new();

    public decimal MaxTransferAmount { get; init; } = 1_000_000m;
    public int TransferExpiryHours { get; init; } = 24;
}
```

<!-- File: Consumer/Program.cs — integrating the feature, with optional HTTP exposure -->

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

Zoran's project organization enforces a strict one-to-one mapping from filesystem to namespace, eliminating the mental cost of locating types. File-scoped namespaces and the one-type-per-file rule make every `using` directive trivially traceable to a single file. Multi-project solutions use `<RootNamespace>` in `.csproj` to collapse assembly names down to their logical namespace prefix, so the compiler sees `TbdModern.WebApi` regardless of the `.csproj` file name `WebApiModern.csproj`.

### MUST
- Use file-scoped namespaces exclusively: `namespace X.Y.Z;` with no braces. **Rationale**: Eliminates one level of indentation and the useless closing brace region that IDEs otherwise require.
- Place exactly one type per file. **Rationale**: The filename tells you the type; a file named `Company.cs` never surprises with a second exported concept.
- Match the namespace exactly to the folder path from the project root. **Rationale**: `Models/` → `namespace X.Models;` makes locating any type a mechanical filesystem walk with zero ambiguity.
- Set `<RootNamespace>` in every `.csproj` so the project file name does not leak into C# namespaces. **Rationale**: Decouples the project file name (which may evolve) from the namespace contract referenced by hundreds of source files.

### SHOULD
- Colocate discriminated union variants in a single file when they are simple data carriers (see Decision Point 3 for guidance on splitting behavioral variants). **Rationale**: The variants collectively define one closed type hierarchy; splitting them across files scatters the exhaustive-match guarantee and forces the reader to re-assemble the union mentally.
- Place standalone `ValueConverter` classes in `Data/Conversions/` (not nested inside the configuration) when the converter contains non-trivial logic (see Decision Point 8). **Rationale**: Converters with domain knowledge like `DateTimeKind` restoration deserve their own file, namespace, and test surface.

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

Zoran's approach enforces strict separation: the domain model must know nothing about persistence; the configuration layer bears sole responsibility for mapping domain types to relational schema. Every choice — shadow keys, alternate keys, ComplexProperty, value converters — serves the goal of keeping the domain model pure while giving EF Core exactly what it needs to generate clean, navigable SQL.

### MUST

- **Use a surrogate shadow primary key on every entity**, never exposing a database-generated integer in the domain model. **Rationale**: Shadow keys let EF Core manage identity internally without contaminating domain classes with persistence concerns.

  In the standard case (domain identity field is named `PublicId`):
  ```csharp
  builder.Property<int>("Id").ValueGeneratedOnAdd();
  builder.HasKey("Id");
  ```

  **Critical edge case**: When the domain model already has a property named `Id` (typed as a value object like `TransferId`), the shadow PK must use a different name to avoid collision:
  ```csharp
  builder.Property<int>("Key")           // shadow property named "Key" — avoids collision with domain's "Id"
      .HasColumnName("Id")               // database column still named "Id"
      .ValueGeneratedOnAdd();
  builder.HasKey("Key");
  ```
  **Rationale**: EF Core cannot have a shadow property and a CLR property with the same name. Rename it and use `HasColumnName` to preserve the database convention.

- Expose the domain identity as an alternate key or unique index, never making it the primary key. **Rationale**: Alternate keys enable query-by-domain-id on indexed columns while foreign keys reference the surrogate integer, so domain IDs remain opaque to table relationships.

  **When to use `HasAlternateKey` vs `HasIndex().IsUnique()`**: Use `HasAlternateKey(e => e.PublicId)` when the domain key is a foreign key *target* in another entity. Use `HasIndex(t => t.Id).IsUnique()` when the domain key is queried but never referenced as a FK — it is lighter-weight and communicates "this is unique but nobody points at it."

  <!-- File: code-ef-core-complex-properties/.../AppDbContext.cs -->
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

  <!-- File: code-ef-core-complex-properties/.../AppDbContext.cs -->
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
              .HasColumnType("varchar(3)");                    // varchar, not nvarchar — currency codes are ASCII
          c.Property(x => x.DecimalPlaces)
              .HasColumnName("CurrencyDecimalPlaces");
      });
  });
  ```

  Every leaf property gets an explicit `HasColumnName` that flattens the namespace (`CurrencyCode` not `Amount_Currency_Code`). Prefer `varchar` over `nvarchar` only when the column is proven to contain ASCII-only data forever (e.g., ISO 4217 currency codes, ISO 3166 country codes) — otherwise default to `nvarchar` to avoid encoding bugs when non-ASCII characters enter the column later.

- **Add a private parameterless constructor on any value object used as a `ComplexProperty`.** EF Core materializes complex types by calling the parameterless constructor first, then backfilling properties:

  <!-- File: code-ef-core-complex-properties/.../Money.cs -->
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
  public decimal Amount { get; init; }
      = Math.Round(Amount, Currency?.DecimalPlaces ?? 0);
  //                       ^^^^^^^^^ null-conditional: Currency might not be populated yet
  ```

- **Place converters in `Data/Conversions/` as standalone files when they contain non-trivial logic.** Trivial converters (simple wrap/unwrap) may be nested inside the entity configuration class or written as inline lambdas. Converters with domain knowledge deserve their own file:

  <!-- File: code-ef-core-complex-properties/.../Conversions/TimestampConverter.cs -->
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
  **Rationale**: SQL Server does not preserve `DateTimeKind`. The converter must explicitly re-specify `DateTimeKind.Utc` when reading back, otherwise the domain invariant (`Timestamp` must be UTC) silently breaks.

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

- Place each `IEntityTypeConfiguration<T>` in a dedicated file under `Data/Aggregates/`, one class per aggregate root. **Rationale**: Configuration isolation by aggregate mirrors the domain model's modularity and keeps `OnModelCreating` a short registry of `ApplyConfiguration` calls.
- Nest child-entity configuration classes inside the parent aggregate configuration (e.g., `InvoiceConfiguration.InvoiceLineConfiguration`). **Rationale**: Nesting communicates ownership and discoverability — readers find all configuration for an aggregate in one file.
- Use `builder.HasDiscriminator<string>().HasValue<TSubtype>()` with TPH inheritance for polymorphic aggregates like `Product → Material | Service`. **Rationale**: TPH avoids complex joins; EF Core materializes the correct subtype from the discriminator column in a single table scan.

### MAY
- Consolidate all configuration inline in `OnModelCreating` (skipping `IEntityTypeConfiguration<T>`) for simple schemas with few entities. See `code-ef-core-complex-properties/final/Demo/Data/AppDbContext.cs` — one entity, everything inline. **Rationale**: For small projects, the ceremony of separate configuration files outweighs the organizational benefit.
- Hide EF Core navigation properties from the domain model with `builder.Ignore()` and map relationships through string-based overloads like `builder.HasMany<InvoiceLine>("LinesCollection")`. **Rationale**: Hiding navigations keeps domain classes free of EF concerns while `builder.Navigation("LinesCollection")` still enables eager loading in queries.
- Omit explicit `IsRequired()` calls when using C# nullable reference types — the NRT annotations (`string` vs `string?`) already signal required vs optional to EF Core conventions. **Rationale**: NRT-driven convention is the modern, less verbose approach.

<!-- File: code-ef-core-ddd-patterns/.../ProductConfiguration.cs — IEntityTypeConfiguration with TPH -->

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

    public class ProductIdConverter : ValueConverter<Product.ProductId, Guid>
    {
        public ProductIdConverter()
            : base(id => id.Value, value => new Product.ProductId(value)) { }
    }
}
```

<!-- File: code-ef-core-natural-keys/.../DatabaseDbContext.cs — inline OnModelCreating with composite natural keys -->

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

Handler methods receive `DbContext` directly as a parameter — no repository abstractions, no unit-of-work interfaces. Queries always use the domain key (the alternate key or unique index), never the shadow primary key. Reads use `AsNoTracking()` to avoid change-tracker overhead; writes use the default tracking behavior and call `SaveChangesAsync()` to persist.

### MUST
- Query exclusively by the domain key via LINQ — `db.Companies.FirstOrDefaultAsync(c => c.PublicId == id)`. Never call `DbSet<T>.Find(key)`. **Rationale**: `Find()` operates on the shadow primary key known only to EF Core, bypassing the domain's identity boundary entirely.
- Inject the `DbContext` directly into consuming classes — `PostAsync(AppDbContext db, NewRequestDto request)`. **Rationale**: One injection covers all entity types; no repository-per-aggregate boilerplate.
- Query domain-typed properties through their `ComplexProperty`-mapped navigation paths — `db.Transfers.Where(t => t.Amount.Currency.Code == "USD")`. **Rationale**: `ComplexProperty` translates deeply nested LINQ expressions to flat SQL columns — use it.

### SHOULD
- Use `AsNoTracking()` on all read-only queries — `db.Companies.AsNoTracking().FirstOrDefaultAsync(...)`. **Rationale**: Read queries must never trigger unintended database writes through the change tracker.
- Use `Include("LinesCollection")` with string-based navigation names when eager-loading child collections that are hidden from the domain model. **Rationale**: The string-based overload maps to the shadow navigation configured in `IEntityTypeConfiguration` or inline `OnModelCreating`.
- Place entity configuration in `Data/EntityConfiguration/` — one `IEntityTypeConfiguration<T>` file per entity, applied via `modelBuilder.ApplyConfiguration()` in `OnModelCreating()`. **Rationale**: Keeps mapping logic separate from the `DbContext` and the domain model, so neither knows about the other.

### MAY
- For large projects with many query paths, extract complex read queries into dedicated query classes in `Data/Queries/` that take `DbContext` as a dependency. **Rationale**: Prevents the `DbContext` class itself from accumulating query logic.

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

<!-- File: code-openapi-docs/.../EndToEndTests/Companies/CreateCompanyTests.cs -->

```csharp
using System.Net.Http.Json;
using Microsoft.VisualStudio.TestTools.UnitTesting;

namespace EndToEndTests.Companies;

// This example shows web API testing via WebApplicationFactory.
// For feature library tests, resolve the public interface directly from the DI container (see SHOULD above).
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

---

## Decision Point 11: Converting Between Layers

Zoran keeps every layer in its own type — domain models, database rows, query DTOs, and response DTOs — and converts between them with explicit, hand-written extension methods. No AutoMapper, no implicit operators, no reflection. Each conversion method is a trivial constructor call that a reader can verify in one glance, and because the source type is already validated before conversion, the method never needs to fail.

### MUST
- MUST convert between layer types via explicit extension methods placed in a static class named `{SourceType}Conversions`. **Rationale**: Explicit conversion methods are greppable, testable, and immune to silent mis-mappings that plague convention-based mappers.
- MUST declare database row types as `internal record class {Name}Row` with primitive-typed positional parameters matching the query columns exactly. **Rationale**: Positional records map directly to Dapper or EF Core projections with zero ceremony and expose no mutable state outside the data layer.
- MUST declare API response types as `public record class {Name}ResponseDto` with `{ get; init; }` properties or positional parameters. **Rationale**: Init-only or positional records guarantee the DTO is immutable after construction, consistent with the response-not-mutated contract.
- MUST place EF Core query-result types in `DataAccess/QueryModels/` as `internal record class {Name}Dto`. **Rationale**: Query model DTOs are an infrastructure concern; keeping them `internal` prevents the API layer from depending on database projection shapes.

### SHOULD
- SHOULD offer conversion methods from every source type that can produce the target DTO, not just the domain model. **Rationale**: When a data-access query returns a `CompanyDto` directly, converting from the DTO avoids an unnecessary round-trip through the domain model.
- SHOULD name conversion methods `ToModel()`, `ToResponseDto()`, or `ToSummaryDto()` — never `Map()` or `Convert()`. **Rationale**: Verbose, type-naming method names make it obvious at the call site exactly which target type the conversion produces.

### MAY
- MAY colocate the inline request DTO at the bottom of the feature's public interface file when the request type exists only for that single method. **Rationale**: Inline request records avoid polluting the project with one-line DTO files while keeping the binding type adjacent to its consuming method.

<!-- File: code-record-types-validation/.../DataModels/PersonRow.cs -->

```csharp
internal record class PersonRow(int Id, Guid PublicId, string FirstName, string LastName);

internal static class PersonConversions
{
    extension(PersonRow dataModel)
    {
        internal Person ToModel() =>
            new Person(dataModel.PublicId, dataModel.FirstName, dataModel.LastName);
    }
}
```

<!-- File: code-minimal-api-primitive-obsession/.../Api/Responses/CompanyResponseDto.cs -->

```csharp
public record class CompanyResponseDto(CompanyHandle Handle, string Name, string IncorporatedInCode, string IncorporatedInName);

public static class CompanyResponseDtoConversions
{
    extension(Company model)
    {
        public CompanyResponseDto ToResponseDto() =>
            new(model.Handle, model.Name, model.IncorporatedIn.Code, model.IncorporatedIn.Name);
    }

    extension(CompanyDto queryModel)
    {
        public CompanyResponseDto ToResponseDto() =>
            new(queryModel.Handle, queryModel.Name, queryModel.IncorporatedIn.Code, queryModel.IncorporatedIn.Name);
    }
}
```

---

## Decision Point 12: Extending a Type with Behavior (Extension Members)

C# 14 introduces `extension(TargetType)` blocks, which Zoran uses pervasively instead of traditional `static class` extension methods. An extension block lives inside a static class named `{Entity}{Role}` (e.g., `BookCreation`, `EstimateComposition`) and adds factory methods, transformations, or composition operators to an existing type — all using instance-method syntax. This pattern keeps the core type small and declarative while housing domain logic in role-specific companion classes.

### MUST
- MUST wrap all extension methods in an `extension(TargetType) { }` block inside a `public static class`. **Rationale**: The C# 14 `extension` block is the compiler-recognized syntax; traditional `static class` extension methods are obsolete in Zoran's idiom.
- MUST name the containing static class `{Entity}{Role}` (e.g., `BookCreation`, `BookManagement`, `EstimateComposition`). **Rationale**: The class name signals which entity is being extended and what role the extension fulfills.

### SHOULD
- SHOULD use factory methods named `New`, `CreateNew`, or `Restore` that return the entity type (possibly nullable). **Rationale**: Explicit factory names with nullable return communicate that construction can fail and avoid throwing from constructors.
- SHOULD prefer the instance-method calling convention enabled by `extension(TargetType target)` over traditional `this Type` parameter syntax. **Rationale**: The instance-style syntax `book.WithAuthor(a)` reads more naturally than `BookExtensions.WithAuthor(book, a)`.

### MAY
- MAY define private helper methods in a separate `extension(AuxiliaryType)` block within the same static class. **Rationale**: Auxiliary extension blocks keep type-specific helpers scoped to the extension class without polluting the target type's namespace.
- MAY define generic `extension<T>` blocks for utility methods applied to generic types. See `code-ef-core-complex-properties/final/Demo/Helpers.cs` for `extension<T>(IEnumerable<T> sequence) { ... }`. **Rationale**: Generic extension blocks apply a method to any type parameter while keeping the helper discoverable near the domain it serves.

<!-- File: code-record-types-superpower/.../Book.cs -->

```csharp
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

<!-- File: code-cs14-discriminated-unions/.../EstimateComposition.cs -->

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
            Estimate.FromNullable(
                TimeSpan.NullableMax(estimate.Optimistic, other.Optimistic),
                TimeSpan.NullableMax(estimate.Pessimistic, other.Pessimistic));
    }
}
```

---

## Decision Point 13: Handling Concurrency / Mutable State in an Immutable World

Immutable domain models avoid accidental mutation but clash with Entity Framework Core's change tracking. The bridge is a copy-constructor pattern: each `With*` method copies the object via a private copy-constructor and sets only the changed property via `{ get; init; }`. For EF Core persistence, one approach is an `UpdateImmutable<T>()` extension method that detaches the original tracked entity, copies scalar/complex property values from the modified copy, and recursively aligns collection and reference navigation graphs using key-based matching.

### MUST
- MUST define a private copy-constructor that shallow-copies all properties from another instance of the same type. **Rationale**: Provides a single, auditable copy point that every `With*` method delegates to.
- MUST implement mutation as `With*` methods that return a new instance via the copy-constructor with only the target property changed. **Rationale**: Preserves immutability at the call site — existing references are never mutated.
- MUST use `{ get; init; }` on all properties of immutable aggregates, never `{ get; set; }`. **Rationale**: `init` enables the copy-constructor and `With*` methods to set properties during initialization while blocking post-construction mutation.
- MUST ensure EF Core persistence of immutable records uses an `UpdateImmutable<T>()` extension that detaches the original, copies values from the modified copy, and aligns navigation graphs. **Rationale**: EF Core's change tracker cannot detect property changes on immutable types; manual graph alignment is required.

### SHOULD
- SHOULD validate invariants spanning multiple properties in the `init` accessor of the collection property itself (e.g., verifying all items share the same currency as the parent). **Rationale**: Co-locates cross-property validation with the data it guards.

### MAY
- MAY use `ImmutableList<T>` for collection properties in deep-immutable aggregates. **Rationale**: Prevents accidental mutation of the backing collection from any reference holder. However, `ImmutableList<T>` has O(log n) indexed access, ~40 bytes per element memory overhead, and is not natively supported as an EF Core navigation type — EF Core cannot initialize or materialize `ImmutableList<T>` automatically (dotnet/efcore issue #21176, open since 2020). The recommended pattern for EF Core aggregates is the `private List<T>` backing field with `public IReadOnlyCollection<T>` accessor shown in Decision Point 2.
- MAY use reflection-based key extraction in `UpdateImmutable<T>()` for generic collection alignment. **Rationale**: Avoids writing per-entity-type sync logic. The name `UpdateImmutable` is misleading — the pattern mutates collections inside EF Core's change tracker, defeating the purpose of immutability. Consider naming it `SyncCollection<T>` or `ApplyCollectionDiff<T>`.

<!-- File: code-deep-immutable-ef-core/.../Invoice.cs -->

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
    public string CustomerName { get; init; } =
        !string.IsNullOrEmpty(customerName) ? customerName
        : throw new ArgumentException("Customer name cannot be null or empty.", nameof(customerName));

    public ImmutableList<InvoiceLine> Lines
    {
        get;
        init => field =
            value.All(line => line.UnitPrice.Currency == Currency) ? value
            : throw new ArgumentException("Mismatched currencies found in invoice lines.");
    } = ImmutableList<InvoiceLine>.Empty;

    public Invoice WithCustomerName(string customerName) =>
        new(this) { CustomerName = customerName };

    public Invoice WithLines(ImmutableList<InvoiceLine> lines) =>
        new(this) { Lines = lines };
}
```

---

## Decision Point 14: Setting Up DI / Project Wiring

DI follows a minimal recipe: register the `DbContext` and nothing else for data access. Feature classes receive the `DbContext` directly as a parameter — no repository interfaces, no service locator, no extra abstraction layers. The container never calls back into itself — there is no `IServiceProvider` argument and no `GetService<T>()` outside the composition root.

### MUST
- Register EF Core `DbContext` with `AddDbContext<T>()` and a connection-string-derived options callback. **Rationale**: `AddDbContext` configures the scoped lifetime, pooled factory, and disposal orchestration.
- Enable `Nullable` and `ImplicitUsings` in every `.csproj`. **Rationale**: Nullable guards force explicit null-handling decisions; implicit usings eliminate boilerplate `using` lines for `System`, `System.Linq`, etc.
- Use only parameter injection in consuming classes — never pass `IServiceProvider` into a class. **Rationale**: Parameter injection makes dependencies explicit in the method signature; service location hides them.
- Use `System.Text.Json` exclusively for all JSON serialization — never add a `Newtonsoft.Json` package reference or use `JsonConvert`. **Rationale**: `System.Text.Json` is built into .NET, requires zero external dependencies, and has been the standard since .NET Core 3.0.

### SHOULD
- Use primary constructors for DI-injected classes whose parameters map directly to stored fields. **Rationale**: Removes the ceremony of private readonly field declarations and assignment-only constructor bodies.

<!-- File: code-openapi-docs/.../Program.cs -->

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

builder.Services.AddDbContext<AppDbContext>(options =>
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

## Anti-Pattern Appendix — Common Mistakes That Break the Style

These patterns explicitly violate Zoran's conventions. If you generate any of these, stop and re-evaluate.

| Anti-Pattern | Fix |
|---|---|
| `{ get; set; }` on domain properties | `{ get; init; }` for immutable, `{ get; private set; }` for controlled mutation |
| `(Guid accountId, Guid userId)` as a method signature | `(AccountId accountId, EmployeeId userId)` — wrap every primitive |
| `bool IsApproved` or `enum Status` for state | Discriminated union: `abstract record Approval; sealed record Pending : Approval;` |
| `[Required] public string Name` on a domain model | `{ get; init; }` with ternary + throw validation |
| `[Key]` or `[Column]` attributes on domain classes | Fluent API `IEntityTypeConfiguration<T>` in `Data/EntityConfiguration/` |
| `DbSet<T>.Find(id)` | `FirstOrDefaultAsync(e => e.PublicId == domainKey)` |
| Wrapping `DbContext` in `IRepository<TAggregate, TKey>` | Inject `DbContext` directly into consuming classes |
| `OwnsOne` / `OwnsMany` for value objects | `ComplexProperty` (EF Core 8+) — flattens into owning table, no JOINs |
| Forgetting parameterless constructor on value objects used in `ComplexProperty` | Add `private Money() : this(default, default!) { }` — required for EF Core materialization |
| `ControllerBase` subclass | `public static class Xxx { ... }` with static Minimal API methods |
| `// Arrange` / `// Act` / `// Assert` comments in tests | Whitespace grouping; self-documenting method names |
| `IServiceProvider.GetService<T>()` inside a class | Constructor or parameter injection |
| `AutoMapper` or `.Map<T>()` for layer conversion | Hand-written extension method: `extension(Source) { public Target ToResponseDto() => ... }` |
| `Utils.cs`, `Helpers.cs`, `Constants.cs`, `Enums.cs` | One concept, one file. If an enum is referenced from multiple files, it gets its own file. An enum used exclusively within a single class may remain colocated.
| `public` constructor on a domain entity | `internal` constructor + `static class XxxConstruction` with `extension(Xxx)` factories |
| `bool IsValid { get; }` or `void Validate()` | Validation in `{ get; init; }` at construction time — invalid objects never exist |
| FluentAssertions / Shouldly / Moq / NSubstitute | Raw MSTest `Assert.*` only. Zero mocking. |
| `Newtonsoft.Json` / `JsonConvert` | `System.Text.Json` — built into .NET, zero dependencies. |
| `string` for identifiers (`string handle`, `string currencyCode`) | Dedicated type: `CompanyHandle`, `Currency`, `Iso4217Currency` |
| Shadow PK named `"Id"` when domain model already has an `Id` property | Rename shadow PK to `"Key"` with `HasColumnName("Id")` |
| `DateTimeKind` stripped silently by SQL Server | Reapply `DateTime.SpecifyKind(dt, DateTimeKind.Utc)` in value converter |
| Assuming `ComplexProperty` sub-objects populate in order | Null-conditional guard: `Currency?.DecimalPlaces ?? 0` |

---

## Extrapolating to New Domains

When you encounter a concept not shown in the examples, derive the correct pattern from the meta-principles:

**To model a new value type** (e.g., `EmailAddress`, `GeoCoordinate`, `PhoneNumber`):
1. Declare an `internal record` with a positional primary constructor (use `public record` only in a shared kernel assembly — see Decision Point 6)
2. Add `{ get; init; }` with ternary + throw for any invariants
3. Use `nameof()` in all exceptions
4. If construction can fail for user input, add a `TryCreateNew()` factory returning `T?`
5. If the type needs arithmetic or comparison, define operator overloads

**To model a new state machine** (e.g., `OrderStatus`, `PaymentState`):
1. Root: `internal abstract record BaseType;`
2. Variants: `sealed record VariantName(...) : BaseType;`
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

**When in doubt between `record` and `class`**:
- Prefer `record` — for values, DTOs, and discriminated unions
- Use `class` only when internal mutation via `{ get; private set; }` is required, or for inheritance hierarchies with behavior
- Prefer `init` over `set` — even on classes
