# Exception Handling in ASP.NET Core

*A deep-dive walkthrough of exception handling in ASP.NET Core — covering the fundamentals of try/catch and custom exception hierarchies, the built-in exception-handling middleware in depth, the modern `IExceptionHandler` interface introduced in .NET 8, `ProblemDetails` (RFC 7807) as the standard error response shape, how middleware-level and filter-level exception handling relate and layer together, proper exception logging, and the judgment calls around when to catch an exception versus letting it propagate.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [try/catch/finally: The Fundamentals, Precisely](#1-trycatchfinally-the-fundamentals-precisely)
3. [Custom Exception Hierarchies](#2-custom-exception-hierarchies)
4. [When to Catch vs. When to Let It Bubble](#3-when-to-catch-vs-when-to-let-it-bubble)
5. [UseExceptionHandler: The Classic Middleware Approach](#4-useexceptionhandler-the-classic-middleware-approach)
6. [IExceptionHandler: The Modern .NET 8+ Approach](#5-iexceptionhandler-the-modern-net-8-approach)
7. [ProblemDetails: The Standard Error Response Shape](#6-problemdetails-the-standard-error-response-shape)
8. [Mapping Specific Exception Types to Specific Responses](#7-mapping-specific-exception-types-to-specific-responses)
9. [Exception Filters vs. Middleware: Choosing the Right Layer](#8-exception-filters-vs-middleware-choosing-the-right-layer)
10. [Logging Exceptions Properly](#9-logging-exceptions-properly)
11. [What NOT to Expose in an Error Response](#10-what-not-to-expose-in-an-error-response)
12. [The Developer Exception Page](#11-the-developer-exception-page)
13. [The Performance Cost of Exceptions](#12-the-performance-cost-of-exceptions)
14. [Common Pitfalls](#13-common-pitfalls)
15. [Quick Reference Table](#quick-reference-table)
16. [Conclusion](#conclusion)

---

## Introduction

Exception handling in a real ASP.NET Core application isn't just "wrap risky code in try/catch" — it's a layered system spanning the language's own exception mechanics, custom exception types that carry meaning specific to your domain, and framework-level infrastructure (middleware, filters, the modern `IExceptionHandler` interface) that catches whatever wasn't handled closer to where it occurred and turns it into a well-formed, safe, standardized HTTP response. This series' Middleware guide's Section 9 and Filters guide's Section 6 both introduce pieces of this system; this guide goes deep on the whole picture — from precisely how `try`/`catch`/`finally` actually behaves, through custom exception hierarchies, to the two generations of global exception-handling infrastructure ASP.NET Core provides, and the `ProblemDetails` standard that gives error responses across the whole .NET ecosystem a consistent, machine-readable shape.

```plaintext
Action throws → (Exception Filter, this series' Filters guide's Section 6,
                   IF it can meaningfully handle THIS specific exception TYPE)
                        ↓ (unhandled)
             → Global Exception-Handling Middleware (Section 4-5) — the
                LAST LINE of defense, catching EVERYTHING nothing else handled
                        ↓
             → A well-formed, standardized ProblemDetails response (Section 6)
```

---

## 1. try/catch/finally: The Fundamentals, Precisely

### `catch` blocks are evaluated top-to-bottom, and only the FIRST matching one runs

```csharp
try
{
    ThrowSomeException();
}
catch (ArgumentNullException ex) { /* runs ONLY if the exception is EXACTLY this type, or a subtype of it */ }
catch (ArgumentException ex)     { /* runs if it's an ArgumentException but NOT an ArgumentNullException */ }
catch (Exception ex)              { /* the CATCH-ALL — runs for anything not matched above */ }
```

C# evaluates `catch` clauses in the order they're written, and stops at the first one whose exception type matches (via `is`-style compatibility, including subtypes) — this is why ordering matters: a more specific exception type must be listed *before* a more general one that would otherwise also match it, or the specific `catch` block becomes unreachable, dead code the compiler doesn't even warn about by default.

### `finally`: guaranteed to run, exception or not — the same guarantee this series' Memory Management guide's `using` relies on

```csharp
try
{
    OpenConnection();
    DoWork(); // might throw
}
finally
{
    CloseConnection(); // ALWAYS runs — whether DoWork() succeeded, threw, or the try block returned early
}
```

This is exactly the same guaranteed-execution mechanism this series' Memory Management guide's Section 7 shows `using` compiling down to, and this series' Threading guide's Section 4 shows `lock` compiling down to — `finally` is the single, foundational language guarantee both of those higher-level constructs are built on top of.

### `throw` vs. `throw ex`: preserving the original stack trace

```csharp
catch (Exception ex)
{
    // ❌ throw ex;   — RESETS the stack trace to THIS line, losing where it ORIGINALLY occurred
    throw;           // ✅ RE-THROWS the SAME exception, preserving its ORIGINAL stack trace
}
```

This is a genuinely common, easy-to-get-wrong detail worth stating precisely: bare `throw` (no expression) re-throws the currently-caught exception with its original stack trace intact, while `throw ex` throws it as if it were a brand-new exception originating from that line, destroying the information about where it actually first occurred — for debugging any exception that's been caught and re-thrown, this distinction is often the difference between a stack trace that's immediately useful and one that's actively misleading.

---

## 2. Custom Exception Hierarchies

### Why the built-in exception types aren't enough for a real domain

```csharp
// ❌ Using a generic exception loses all DOMAIN MEANING — every catch site
//    has to inspect the MESSAGE STRING to figure out what actually went wrong
throw new Exception("Order 42 cannot be cancelled because it has already shipped");
```

A bare `Exception` (or even a somewhat more specific built-in type like `InvalidOperationException`) carries no structured, catchable information about *which specific domain rule* was violated — any code trying to react differently to different failure modes is reduced to string-matching the message, which is exactly the kind of fragile, error-prone code a proper exception hierarchy exists to avoid.

### Defining a base exception type for your domain, with meaningful subtypes

```csharp
public abstract class OrderException : Exception // per this series' Abstract Classes guide's Section 3
{
    protected OrderException(string message) : base(message) { }
}

public class OrderNotFoundException : OrderException
{
    public int OrderId { get; }
    public OrderNotFoundException(int orderId) : base($"Order {orderId} was not found.") => OrderId = orderId;
}

public class OrderAlreadyShippedException : OrderException
{
    public int OrderId { get; }
    public OrderAlreadyShippedException(int orderId) : base($"Order {orderId} cannot be modified — it has already shipped.")
        => OrderId = orderId;
}
```

This directly applies this series' Abstract Classes guide's own reasoning for when inheritance genuinely earns its place — `OrderNotFoundException` and `OrderAlreadyShippedException` genuinely share a real "is-a" relationship (both are, specifically, order-related domain failures) and genuinely share real behavior (the `Message` construction pattern, and critically, the ability for calling code to catch `OrderException` broadly to handle "any order-related problem," or catch the specific subtype for a precise, differentiated response, per Section 7).

### Carrying structured data on the exception, not just a formatted message string

```csharp
catch (OrderNotFoundException ex)
{
    logger.LogWarning("Order lookup failed for {OrderId}", ex.OrderId); // structured logging, per Section 9 —
                                                                            //  using ex.OrderId directly, not
                                                                            //  parsing it back out of ex.Message
}
```

This is the concrete, practical payoff of a well-designed custom exception — `OrderId` as a real, typed property means calling code can use it directly (for logging, for building a specific response, for any conditional logic), rather than needing to parse it back out of a human-readable message string that was never meant to be machine-parsed in the first place.

---

## 3. When to Catch vs. When to Let It Bubble

### The core principle: only catch an exception where you can genuinely, meaningfully DO something about it

```csharp
// ❌ Catching and doing NOTHING useful — this actively HIDES a real problem
try
{
    await _repository.SaveAsync(order);
}
catch (Exception)
{
    // silently swallowed — the caller has NO IDEA the save failed
}
```

This is one of the most consequential exception-handling anti-patterns, worth stating as plainly as possible: catching an exception and doing nothing meaningful with it (not logging it, not handling it, not re-throwing it) doesn't make the problem go away — it hides it, turning a loud, visible failure into a silent, much harder to diagnose one, often discovered only much later when its downstream consequences finally surface somewhere unrelated.

### The legitimate reasons to catch an exception, stated precisely

```plaintext
1. You can genuinely RECOVER — retry the operation, fall back to an
   alternative, or otherwise continue in a way that's actually correct.
2. You need to TRANSLATE it into something more meaningful to the
   caller (a low-level SqlException becomes a domain-specific
   OrderSaveFailedException, per Section 2's hierarchy pattern).
3. You need to ADD CONTEXT before re-throwing (logging, or wrapping it
   with additional information) — but you STILL re-throw or throw a new,
   appropriately-wrapped exception; you don't just swallow it.
4. You're at the GLOBAL, top-level boundary (Sections 4-5) — this is
   the ONE place "catch everything and turn it into a safe response" is
   not just acceptable but the entire point.
```

Every one of these is a genuinely deliberate, purposeful catch — the common thread is that something *meaningful* happens as a result of catching, whether that's recovery, translation, enrichment, or (at the top level specifically) producing a safe, well-formed response instead of letting an unhandled exception crash the request.

### Letting an exception propagate is often the CORRECT choice, not a failure to handle it

```csharp
public async Task<Order> GetOrderAsync(int id)
{
    var order = await _repository.GetByIdAsync(id);
    if (order is null) throw new OrderNotFoundException(id); // deliberately propagates UP —
                                                                //  THIS method has no business deciding
                                                                //  what an HTTP 404 response should look like
    return order;
}
```

This is worth internalizing directly: a repository or service method genuinely shouldn't be catching and converting exceptions into HTTP responses itself — that's a presentation-layer concern, and the correct design is for domain/service-layer code to throw meaningful, well-typed exceptions (Section 2) and let them propagate upward, to be caught and translated into an appropriate response at the boundary that actually knows what "response" means (Sections 4-7).

---

## 4. UseExceptionHandler: The Classic Middleware Approach

### The mechanics, revisited with full depth from this series' Middleware guide's Section 9

```csharp
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exceptionHandlerFeature = context.Features.Get<IExceptionHandlerFeature>();
        var exception = exceptionHandlerFeature?.Error;

        context.Response.StatusCode = exception switch
        {
            OrderNotFoundException => StatusCodes.Status404NotFound,
            OrderAlreadyShippedException => StatusCodes.Status409Conflict,
            _ => StatusCodes.Status500InternalServerError
        };

        await context.Response.WriteAsJsonAsync(new { error = exception?.Message });
    });
});
```

This series' Middleware guide's Section 9 already establishes the core mechanic — `UseExceptionHandler` wraps everything registered after it and re-executes the pipeline against a configured error-handling branch on catching an unhandled exception — this section goes further into the practical pattern of pattern-matching on the caught exception's *type* to determine the appropriate status code, directly applying Section 2's custom exception hierarchy to produce a genuinely differentiated response per failure mode, rather than one generic 500 for everything.

### Why it MUST be registered first, restated with the full mechanical reasoning

```plaintext
Per this series' Middleware guide's Section 2's chain model: UseExceptionHandler
  can only catch exceptions from middleware registered AFTER it — this is
  not a convention, it's a direct, mechanical consequence of how the
  RequestDelegate chain is built (Section 1 of that guide), which is
  exactly why every ASP.NET Core project template places it first.
```

---

## 5. IExceptionHandler: The Modern .NET 8+ Approach

### A dedicated, DI-friendly interface, replacing the inline-lambda pattern with a proper, testable class

```csharp
public class OrderExceptionHandler : IExceptionHandler
{
    private readonly ILogger<OrderExceptionHandler> _logger;
    public OrderExceptionHandler(ILogger<OrderExceptionHandler> logger) => _logger = logger; // GENUINE constructor injection

    public async ValueTask<bool> TryHandleAsync(HttpContext httpContext, Exception exception, CancellationToken cancellationToken)
    {
        if (exception is not OrderException orderException)
            return false; // ❌ this handler doesn't know how to handle THIS exception — let another handler try

        var statusCode = orderException switch
        {
            OrderNotFoundException => StatusCodes.Status404NotFound,
            OrderAlreadyShippedException => StatusCodes.Status409Conflict,
            _ => StatusCodes.Status500InternalServerError
        };

        _logger.LogWarning(exception, "Order exception handled: {Message}", exception.Message);

        httpContext.Response.StatusCode = statusCode;
        await httpContext.Response.WriteAsJsonAsync(new ProblemDetails // Section 6
        {
            Status = statusCode,
            Title = exception.Message
        }, cancellationToken);

        return true; // ✅ successfully handled
    }
}
```

Introduced in .NET 8, `IExceptionHandler` is the modern, recommended replacement for the inline-lambda `UseExceptionHandler` pattern — as an ordinary, DI-registered class (per this series' ASP.NET Core Dependency Injection guide, following whatever lifetime you register it with), it gets genuine constructor injection (a logger, here, but any DI-resolved service works identically), and it's independently unit-testable in a way an inline lambda embedded in `Program.cs` simply isn't.

### Registering one or more handlers, evaluated in registration order

```csharp
builder.Services.AddExceptionHandler<OrderExceptionHandler>(); // more specific handlers registered FIRST
builder.Services.AddExceptionHandler<GlobalExceptionHandler>(); // a catch-all, registered LAST
builder.Services.AddProblemDetails(); // ensures a ProblemDetails response even if NO registered handler claims it

// in the pipeline:
app.UseExceptionHandler(); // no lambda needed — delegates to the REGISTERED IExceptionHandler instances
```

This is the return value's real purpose (`TryHandleAsync` returning `bool`): multiple handlers can be registered, evaluated in the order they were added, and each gets a chance to claim (`return true`) or decline (`return false`) responsibility for a given exception — precisely the same "chain of handlers, first willing one wins" pattern this series' Authorization guide's Section 7 describes for multiple `AuthorizationHandler`s evaluating the same requirement, just applied here to exception handling instead.

### A catch-all handler as the final safety net in the chain

```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext httpContext, Exception exception, CancellationToken cancellationToken)
    {
        httpContext.Response.StatusCode = StatusCodes.Status500InternalServerError;
        await httpContext.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = 500,
            Title = "An unexpected error occurred." // deliberately GENERIC — Section 10 covers why
        }, cancellationToken);
        return true; // ALWAYS claims responsibility — nothing gets past this one unhandled
    }
}
```

Registering an unconditional, always-`return true` handler last in the chain ensures every exception is genuinely handled by *something*, even one no more specific handler recognized — this mirrors the layered-defense philosophy this series' Authentication guide's Section 9 (short-lived tokens) and Rate Limiter guide's Section 9 (fail-open/fail-closed policy) both apply in their own domains: specific handling where possible, with a deliberate, unconditional fallback ensuring nothing slips through entirely unhandled.

---

## 6. ProblemDetails: The Standard Error Response Shape

### RFC 7807's standardized JSON structure for HTTP API error responses

```json
{
  "type": "https://example.com/errors/order-not-found",
  "title": "Order not found",
  "status": 404,
  "detail": "Order 42 was not found.",
  "instance": "/api/orders/42"
}
```

`ProblemDetails` is a formal, RFC-defined structure specifically so that error responses across genuinely different APIs — not just different endpoints in the same application — share a common, predictable, machine-parseable shape, letting generic client tooling (error-handling middleware in a frontend framework, a monitoring dashboard) understand *any* compliant API's errors without needing bespoke, per-API parsing logic.

### The standard fields, and what each is actually for

```plaintext
type: a URI identifying the SPECIFIC problem type (ideally a link to
  documentation about it) — defaults to "about:blank" if not set.
title: a short, human-readable SUMMARY of the problem, generally the
  SAME across every occurrence of this specific problem type.
status: the HTTP status code, duplicated here for convenience
  (per this series' REST guide's Section 9 precise status code usage).
detail: a human-readable explanation SPECIFIC to this occurrence
  (e.g., naming the specific order ID, unlike `title`'s generic wording).
instance: a URI identifying THIS SPECIFIC occurrence of the problem
  (often the request path that triggered it).
```

### ASP.NET Core's built-in `ProblemDetails` support and automatic generation

```csharp
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = context =>
    {
        context.ProblemDetails.Extensions["traceId"] = context.HttpContext.TraceIdentifier; // custom extension field
    };
});
```

`AddProblemDetails()` wires up automatic `ProblemDetails` generation for a range of built-in failure scenarios (unhandled exceptions when paired with Section 5's `IExceptionHandler`, and model-validation failures, among others) — `CustomizeProblemDetails` is the extension point for adding your own fields (a correlation/trace ID, for instance) consistently across every generated `ProblemDetails` response, which is directly useful for tying an error response back to the corresponding log entries (Section 9).

---

## 7. Mapping Specific Exception Types to Specific Responses

### A centralized mapping table, avoiding scattered, duplicated exception-to-status-code logic

```csharp
public static class ExceptionStatusCodeMapper
{
    private static readonly Dictionary<Type, int> _mapping = new()
    {
        [typeof(OrderNotFoundException)] = StatusCodes.Status404NotFound,
        [typeof(OrderAlreadyShippedException)] = StatusCodes.Status409Conflict,
        [typeof(ValidationException)] = StatusCodes.Status400BadRequest,
    };

    public static int GetStatusCode(Exception exception) =>
        _mapping.TryGetValue(exception.GetType(), out var code) ? code : StatusCodes.Status500InternalServerError;
}
```

For an application with more than a handful of custom exception types, centralizing the exception-to-status-code mapping in one place (rather than repeating a `switch` expression in every exception handler) keeps the mapping consistent and gives you exactly one place to update when a new exception type is introduced — directly echoing this series' Authorization guide's Section 4 reasoning for centralizing named policies rather than scattering the same logic across many attributes.

### Why a base exception type's status code shouldn't be assumed from its subtype's

```csharp
public static int GetStatusCode(Exception exception) => exception switch
{
    OrderNotFoundException => 404,
    OrderAlreadyShippedException => 409,
    OrderException => 400, // a fallback for ANY OTHER OrderException subtype not specifically listed
    _ => 500
};
```

Worth being deliberate about this: relying on C#'s pattern matching to fall through to a less-specific base type (`OrderException` here) as a reasonable default for any subtype you haven't explicitly mapped is a genuinely useful technique — but it requires the base type's chosen default (400, here, treating any otherwise-unmapped order problem as a client-correctable bad request) to actually be a sensible, safe default for the whole hierarchy, which is a real design decision worth making consciously rather than by accident.

---

## 8. Exception Filters vs. Middleware: Choosing the Right Layer

### This series' Filters guide's Section 6 covers exception filters directly — worth restating the decision here, now with the full global-handling picture in view

```plaintext
Exception FILTERS (this series' Filters guide's Section 6): scoped to a
  specific controller/action, or globally registered but still only
  covering MVC action execution — genuinely useful for exception
  handling that needs rich MVC context (which action threw, its bound
  arguments) or needs to differ per controller.
Global exception-handling MIDDLEWARE/IExceptionHandler (this guide's
  Sections 4-5): the universal, application-wide safety net, catching
  EVERYTHING — including exceptions from non-MVC middleware, minimal
  API endpoints, and anything an exception filter didn't claim.
```

This is precisely this series' Filters guide's Section 6 complementary-layers framing, restated here with this guide's fuller depth: most real applications benefit from BOTH — a small number of targeted exception filters for genuinely MVC-context-specific handling, and a global `IExceptionHandler` chain (Section 5) as the comprehensive, final safety net that nothing escapes.

### A concrete decision point: does the handling logic genuinely need MVC-specific context?

```plaintext
Needs context.ActionArguments, or is genuinely SPECIFIC to one
  controller's particular failure modes → exception filter.
Needs to apply UNIVERSALLY, including to minimal APIs or non-MVC
  middleware, or is a general-purpose "map exception type to status
  code" concern → global IExceptionHandler.
```

---

## 9. Logging Exceptions Properly

### Always log the exception OBJECT itself, not just its message

```csharp
// ❌ Discards the stack trace, inner exceptions, and structured exception DATA entirely
logger.LogError("An error occurred: " + ex.Message);

// ✅ Passes the EXCEPTION OBJECT as its own parameter — the logging framework captures
//    the full stack trace, exception type, and any structured data properly
logger.LogError(ex, "Failed to process order {OrderId}", orderId);
```

This is a genuinely common, costly logging mistake — string-concatenating an exception's `.Message` into a log line throws away the stack trace, the exception's actual type, any inner exceptions, and any structured properties (like Section 2's `OrderId`) entirely; passing the exception object as its own logging parameter (the first argument, by convention, in most .NET logging frameworks) preserves all of that, letting a log aggregation/analysis tool actually search, filter, and correlate on it later.

### Choosing the right log level: not every exception is an `Error`

```plaintext
LogWarning: an EXPECTED, recoverable, or user-caused condition —
  OrderNotFoundException from a client requesting a nonexistent order
  is arguably a WARNING, not an ERROR — nothing is actually broken.
LogError: a GENUINE, unexpected failure — a database connection
  failure, a null reference that should never have happened.
LogCritical: reserved for failures threatening the APPLICATION'S
  own ability to continue functioning at all.
```

Logging every single caught exception as `Error` — regardless of whether it represents a genuine system failure or an entirely expected, routine condition (a client requesting a resource that doesn't exist) — creates real, practical noise that drowns out the log entries that genuinely warrant urgent attention, degrading the value of error-level alerting for the whole application.

### Correlation IDs: tying a specific error response back to its exact log entry

```csharp
httpContext.Response.Headers.Append("X-Correlation-Id", httpContext.TraceIdentifier);
logger.LogError(ex, "Unhandled exception. TraceId: {TraceId}", httpContext.TraceIdentifier);
```

Including `HttpContext.TraceIdentifier` (a unique ID ASP.NET Core generates per request automatically) in both the logged entry and the error response returned to the client is what makes "the client reports an error, and we need to find exactly what happened" actually tractable — without a shared correlation identifier, matching a user's bug report to the corresponding log entry, among potentially millions of others, is a genuinely difficult, often impossible task.

---

## 10. What NOT to Expose in an Error Response

### Stack traces, internal exception messages, and implementation details are a real, documented security risk

```csharp
// ❌ Leaks internal implementation details — table names, connection strings in
//    exception messages, internal file paths, the FULL .NET stack trace
await context.Response.WriteAsJsonAsync(new { error = exception.ToString() }); // NEVER do this in production
```

This connects directly to this series' Authentication guide's own security-conscious framing — an exception's full details (a raw SQL exception's message, which might reveal table/column names; a file-system exception revealing internal directory structure; a stack trace revealing exact library versions in use) is genuinely useful information for an attacker probing an API's internals, and exposing it by default in production is a real, well-documented vulnerability class, not a hypothetical concern.

### The correct pattern: a generic, safe message to the client; full detail only to logs

```csharp
public async ValueTask<bool> TryHandleAsync(HttpContext httpContext, Exception exception, CancellationToken cancellationToken)
{
    _logger.LogError(exception, "Unhandled exception occurred"); // FULL detail, to LOGS only

    await httpContext.Response.WriteAsJsonAsync(new ProblemDetails
    {
        Status = 500,
        Title = "An unexpected error occurred.", // GENERIC, safe message, to the CLIENT
        Extensions = { ["traceId"] = httpContext.TraceIdentifier } // the CORRELATION ID (Section 9), not the exception itself
    }, cancellationToken);
    return true;
}
```

The client gets a genuinely safe, generic message plus a correlation ID they can reference when reporting the issue; the full, potentially sensitive detail goes exclusively to the application's own logs, accessible only to people with legitimate access to them — this is the correct, standard pattern for balancing "the client needs to know something went wrong" against "the client shouldn't learn anything about your internals from the failure."

### Environment-conditional detail: safe to be MORE verbose in Development specifically

```csharp
builder.Services.AddProblemDetails(); // combined with:
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage(); // Section 11 — full detail, but ONLY when NOT in production
}
else
{
    app.UseExceptionHandler(); // the SAFE, generic handler from this section, for production
}
```

---

## 11. The Developer Exception Page

### A genuinely useful, but strictly Development-only, diagnostic tool

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage(); // detailed, INTERACTIVE HTML page showing the FULL exception, stack trace,
                                        //  request details, query parameters, and more
}
```

The Developer Exception Page is ASP.NET Core's built-in, richly detailed error page specifically meant for local development — it shows the complete exception (including inner exceptions), the full stack trace with the ability to inspect source code inline (if source is available), request headers, query string values, cookies, and more — genuinely valuable during active development, and precisely why Section 10's guidance is so firm about never letting this same level of detail reach a production response.

### Why this MUST be gated behind an environment check, without exception

```plaintext
This is worth stating as close to an absolute rule as this guide makes:
  the Developer Exception Page reveals EXACTLY the kind of internal
  detail Section 10 warns against exposing — accidentally leaving it
  enabled in Production (a genuinely real, documented misconfiguration
  that has happened to real applications) directly hands an attacker
  full stack traces, internal paths, and potentially even snippets of
  source code, for EVERY unhandled exception the application throws.
```

Worth treating `if (app.Environment.IsDevelopment())` gating this specific call as one of the single most consequential lines in a typical `Program.cs` — the cost of getting this one check wrong is genuinely severe, disproportionate to how small and easy-to-overlook the line itself is.

---

## 12. The Performance Cost of Exceptions

### Throwing and catching an exception is genuinely, measurably more expensive than ordinary control flow

```plaintext
Constructing an exception captures a full STACK TRACE at the point it's
  thrown (or, in some cases, at the point it's constructed) — this is a
  real, non-trivial cost, meaningfully more expensive than an ordinary
  method return or an `if` check, precisely BECAUSE exceptions are
  designed to carry rich diagnostic information about where and how they occurred.
```

This is worth knowing precisely, not as a vague "exceptions are slow" folk wisdom, but as a specific, understood cost — the overhead comes largely from stack trace capture and the runtime's exception-handling machinery unwinding the call stack looking for a matching `catch`, both of which are doing genuinely more work than a normal, non-exceptional return path.

### Why this means exceptions should be reserved for genuinely EXCEPTIONAL conditions, not routine control flow

```csharp
// ❌ Using an exception for a routine, EXPECTED outcome (not finding an item) —
//    this is control flow, not an exceptional condition, and paying exception
//    overhead for something that happens on a meaningful fraction of requests is wasteful
public Order GetOrder(int id)
{
    var order = _orders.FirstOrDefault(o => o.Id == id);
    if (order is null) throw new OrderNotFoundException(id); // debatable — see below
    return order;
}

// ✅ For a GENUINELY routine "might not exist" check, a nullable return or a
//    Result/TryGet pattern avoids exception overhead entirely for the common case
public bool TryGetOrder(int id, out Order? order)
{
    order = _orders.FirstOrDefault(o => o.Id == id);
    return order is not null;
}
```

This is worth presenting as a genuine, nuanced judgment call rather than an absolute rule — `OrderNotFoundException` for a genuinely rare, unexpected "this ID should have existed but doesn't" case is a defensible use of Section 2's exception hierarchy; but if "not found" is a routine, expected, frequently-occurring outcome (checking whether an item exists in a cache, say, where misses happen constantly and aren't exceptional at all), a non-exception-based pattern (`TryGetValue`-style, or returning a nullable/`Result` type) avoids paying real, repeated exception overhead for something that isn't actually exceptional.

---

## 13. Common Pitfalls

| Pitfall | Why it hurts | Better approach |
|---|---|---|
| Catching an exception and doing nothing with it | Silently hides a real problem, turning a loud, immediately visible failure into one discovered much later, if ever | Only catch where you can genuinely recover, translate, enrich, or (at the top level) produce a safe response (Section 3) |
| Using `throw ex` instead of bare `throw` when re-throwing | Resets the stack trace, destroying information about where the exception actually first occurred | Use bare `throw` to preserve the original stack trace when re-throwing a caught exception (Section 1) |
| Logging only `ex.Message` as a string, rather than the exception object itself | Discards the stack trace and structured exception data the logging framework would otherwise capture | Always pass the exception object as its own logging parameter, not string-concatenated into the message (Section 9) |
| Returning full exception details (stack traces, raw messages) to clients in production | A real, documented security risk — reveals internal implementation details useful to an attacker | Return a generic, safe message plus a correlation ID to the client; keep full detail exclusively in logs (Section 10) |
| Leaving `UseDeveloperExceptionPage()` enabled outside of Development | Exposes complete stack traces and internal application detail to every caller in production | Gate it strictly behind an environment check; use the safe, generic exception handler for every other environment (Section 11) |
| Logging every caught exception at `Error` level, regardless of whether it's genuinely unexpected | Creates noise that drowns out log entries that actually warrant urgent attention | Choose log level based on whether the condition is genuinely unexpected, not just because an exception was involved (Section 9) |
| Using exceptions for routine, frequently-occurring "not found" or validation outcomes | Exception construction and unwinding carries real, measurable overhead, wasteful when paid on every occurrence of an expected outcome | Reserve exceptions for genuinely exceptional conditions; use `TryGetValue`-style or `Result`-based patterns for routine outcomes (Section 12) |
| Scattering exception-to-status-code mapping logic across many separate handlers or filters | Inconsistent responses for the same exception type depending on which handler happened to catch it first | Centralize the mapping in one place, referenced consistently everywhere it's needed (Section 7) |

---

## Quick Reference Table

| Concept | Syntax/Mechanism | Purpose |
|---|---|---|
| Preserve stack trace on re-throw | `throw;` (bare, not `throw ex;`) | Keeps the original point of failure visible for debugging |
| Custom exception hierarchy | `class OrderException : Exception`, with typed subtypes | Carries structured, domain-specific failure information |
| Classic global handling | `app.UseExceptionHandler(errorApp => ...);` | Middleware-based catch-all, wrapping everything registered after it |
| Modern global handling (.NET 8+) | `IExceptionHandler` + `AddExceptionHandler<T>()` | DI-friendly, testable, chainable exception handler classes |
| Standard error shape | `ProblemDetails` (RFC 7807) | A consistent, machine-parseable error response structure across APIs |
| MVC-context-specific handling | `IExceptionFilter` (this series' Filters guide) | Handles exceptions with access to action-specific context |
| Correlation ID | `HttpContext.TraceIdentifier` | Ties a client-visible error response back to its exact log entry |
| Development-only detail | `app.UseDeveloperExceptionPage();` (Development-gated) | Full diagnostic detail locally; never in production |

---

## Conclusion

Exception handling done well in ASP.NET Core is a genuinely layered system — precise `try`/`catch`/`finally` mechanics at the code level, custom exception hierarchies that carry real, structured domain meaning rather than opaque message strings, and a global, DI-integrated safety net (`IExceptionHandler`, paired with `ProblemDetails`) that ensures nothing an application throws ever reaches a client as a raw, unhandled, potentially sensitive stack trace. The judgment call this guide returns to repeatedly — catch only where you can genuinely do something meaningful, and let everything else propagate to a boundary that actually knows how to handle it — is what separates deliberate, layered exception handling from the anti-pattern of catching everything everywhere "just in case," which in practice just hides real problems behind a false sense of safety.

The security dimension this guide spends real effort on — never exposing internal exception detail to a production client, and treating the Developer Exception Page's environment gating as close to sacred — is worth carrying as seriously as any other security control covered elsewhere in this series, precisely because an unhandled exception is exactly the kind of unplanned, unreviewed code path where a security-relevant mistake (leaking a connection string, an internal file path, a stack trace revealing exact dependency versions) is easiest to introduce by accident and easiest to overlook until it's already been exploited.

---

*Found this useful? Feel free to star the repo, open an issue with corrections, or share the DeveloperExceptionPage-left-on-in-production discovery that made environment gating feel like the single highest-leverage line in the whole Program.cs.*
