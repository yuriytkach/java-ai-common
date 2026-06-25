# Logging and Observability Rules

Mandatory reading before adding, changing, or removing log statements, MDC keys, or any code
that hands work to another thread that needs to keep the caller's logging context.

For the **policy** of when to log and at which level, see the `## Logging` section in
`CODE-STYLE.md` — that lives in the always-loaded file so an agent writing brand-new code knows
to add logs in the first place. This file covers the **mechanics**: exact format, MDC conventions,
async propagation.

## Message structure: static text first, parameters labelled at the end

Write the human-readable part of the message as a complete, uninterrupted phrase, then append the
parameters as labelled `key={}` tokens. Do NOT splice a `{}` placeholder into the middle of the
sentence, and do NOT emit a bare `{}` with no key.

```java
// WRONG — values spliced mid-sentence, so the static text is fragmented and not searchable as a
// whole; bare {} gives the reader no clue what the value is
log.info("No active subscription for customer {}; skipping renewal for {} orders",
  customerId, orderCount);

// CORRECT — full sentence first, then labelled key={} tokens
log.info("No active subscription for customer; skipping renewal. CustomerId={}, OrderCount={}",
  customerId, orderCount);
```

Why:
- **Full-text search.** A reader who sees one occurrence can copy the static sentence verbatim and
  grep every other occurrence. A `{}` in the middle splits the static text into fragments, so no
  single contiguous substring matches across all log lines.
- **Self-describing values.** `customerId=12345` tells the reader what the value is; a bare `12345`
  floating in prose does not. Labelled `key={}` tokens also match the structured / logfmt style log
  search operators expect (`orderCount=` as a field filter).
- **Consistency.** This is the same shape the exception format below already uses (`orderId={}`
  before the trailing `: {}`).

Keep the leading phrase short — it is the searchable identity of the log line, not a paragraph. A
single trailing parameter that already reads as `… orderId={}` is fine; the rule only forbids
placeholders *before* the end of the human-readable text.

## Exception logging format

Always include `ex.getMessage()` as an explicit placeholder in the message string, followed by the
exception itself as the final argument so SLF4J attaches the full stack trace:

```java
// CORRECT
log.error("Failed to process order orderId={}: {}", orderId, ex.getMessage(), ex);

// WRONG - message is lost in alerts; stack trace is missing
log.error("Failed to process order", ex);
log.error("Failed to process order orderId={}", orderId, ex);
```

The extra `{}` placeholder for `ex.getMessage()` ensures the human-readable failure reason appears
in log-based alerts even when stack traces are suppressed or truncated.

## MDC usage

- Use MDC to attach context that should appear on every log line within a processing scope.
- All MDC keys MUST be prefixed with `x-`, e.g. `x-order-id`, `x-batch-id`.
- If `x-trace-id` is already populated by the request logging infrastructure, do NOT set it manually.
- Set additional MDC keys whenever a unit of work has a stable identifier that aids debugging,
  for example when entering a loop that processes individual items, or inside an async task.
- Always clear MDC keys when leaving the scope. Use `MDC.putCloseable()` with try-with-resources
  so the key is removed automatically - do NOT use `MDC.put()` with a manual `finally` block.

```java
try (var ignored = MDC.putCloseable("x-order-id", orderId)) {
  log.info("Processing order");
  // ... work ...
}
```

## MDC across async boundaries

MDC is a `ThreadLocal`, so it does NOT propagate when work is dispatched to another thread -
`CompletableFuture.runAsync`/`supplyAsync`/`*Async` with an executor, `ExecutorService.submit`,
`@Async`, `@Scheduled`, and reactive `subscribeOn`/`publishOn` all start the inner flow with an
empty MDC and silently lose `x-trace-id` and other caller context.

Snapshot the caller's MDC and install it on the worker. No need to capture/restore the worker's
prior map - a freshly dispatched task should not see anything else, and `MDC.clear()` in
`finally` is enough to keep pooled threads clean.

```java
// WRONG - inner flow runs with an empty MDC
CompletableFuture.runAsync(() -> processBatch(batchId), executor);

// CORRECT
final var callerMdc = MDC.getCopyOfContextMap();
CompletableFuture.runAsync(() -> {
  if (callerMdc != null) {
    MDC.setContextMap(callerMdc);
  }
  try {
    processBatch(batchId);
  } finally {
    MDC.clear();
  }
}, executor);
```

If the project already has an MDC-aware executor wrapper or `Runnable`/`Callable` decorator,
use it instead of inlining this. Never dispatch raw work that drops the context.
