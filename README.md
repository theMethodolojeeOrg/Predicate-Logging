# Predicate Logging

**A formal logic system for runtime state validation that transforms logs from debug noise into self-validating proof trees.**

---
## Demo

<iframe src="index.html" width="100%" height="600px" frameborder="0"></iframe>

## The Problem: The Catastrophic 200

Modern distributed systems fail silently with technical success masking semantic failure:

- HTTP 200 OK, but the response body is malformed JSON
- HTTP 200 OK, but the data is stale by hours
- HTTP 200 OK, but it's an error wrapped in a success envelope
- HTTP 200 OK, but only 3 of 5 microservices succeeded
- HTTP 200 OK, but the async job it triggered will fail

Your logs say "success." Your metrics show 99.9% uptime. Your users are screaming.

**Root cause:** Traditional logs track execution flow, not state validity.

---

## The Solution: Logs as Typed Predicates

Every log is a falsifiable claim about runtime state. Instead of asking "did the operation complete?", we ask "what truth claims can we make about this run's state?"

### Traditional Logging
```javascript
log.info("User profile fetched");  // HTTP 200 OK
// (data.user is null, application crashes)
```

**Problem:** Technical success ≠ Semantic success

### Predicate Logging
```javascript
logSemantic({
  predicate: "http_request_successful",
  passed: response.ok,
  correlationId
});

logSemantic({
  predicate: "response_parseable",
  passed: true,
  correlationId
});

logSemantic({
  predicate: "user_data_present",
  passed: !!data.user,
  correlationId
});
```

**Solution:** Each log is a verifiable predicate claim. When something fails, you know exactly which assumption broke.

---

## How It Works: Three Core Concepts

### 1. Hierarchical Proof Trees

Logs form parent-child relationships. Child predicates only exist when their parents passed.

```
✓ http_request_successful
  ✓ response_parseable
    ✗ user_data_present
```

**Reading this tree:** The HTTP request succeeded, the response was valid JSON, but the user data field was missing. The exact failure point is obvious.

### 2. Correlation Across Distributed Calls

Every predicate belongs to a specific execution run via `correlationId`. Track multi-step workflows across services, functions, and async operations.

### 3. Fail-Upward Development

Instead of hoping your code works and debugging blindly when it doesn't, you:

1. **Declare predicates first** - What should be true?
2. **Implement with logging** - Log each predicate as you validate it
3. **Test by deliberately breaking things** - Test each predicate's failure mode
4. **Read the proof tree** - See exactly which assumption failed
5. **Fix the specific predicate** - Not the whole system
6. **Repeat** - Each failure teaches you something measurable

---

## Real Impact

### Before (Traditional)
```
Production metrics: 99.9% success rate
User reports: "Nothing works!"
Your response: "But the metrics say it's working..."

Debug cycle:
- Can't reproduce locally
- Works on my machine
- 3 hours of log spelunking
- Root cause: unclear
```

### After (Predicate)
```
Production dashboard:
  http_request_successful: 99.9% ✅
  response_parseable: 99.8% ✅
  user_data_present: 95.2% ⚠️  ← The problem

Alert triggered: "user_data_present dropped below threshold"
Investigation: Filter logs where passed=false
Finding: { error: "User deleted" }
Root cause: Not handling deleted users
Fix: Add user_not_deleted predicate

Debug cycle: 10 minutes
```

**Precision over hope. Structure over chaos. Learning over guessing.**

---

## The Nine Core Principles

1. **Claim Rule** - A log is a single predicate claim
2. **Honesty Rule** - Only log what you actually checked
3. **Atomic First** - Composite truths are inferred from atomic predicates
4. **Hierarchy Rule** - Children only exist in valid parent worlds
5. **Scope Rule** - Be explicit about proximity to domain vs infrastructure
6. **Correlation Rule** - Claims belong to a specific execution run
7. **Modular Worlds** - Valid subtrees are real logic modules
8. **Required Shape First** - Stable schemas as contracts between teams
9. **Shakespeare Monkeys** - Reliability emerges from tiny, honest, checkable claims

See [`predicate-logging/SKILL.md`](predicate-logging/SKILL.md) for complete specifications and implementation patterns.

---

## Quick Start

### 1. Define Your Predicates

Before implementing, declare what should be true:

```typescript
// What must hold for user authentication?
const authPredicates = {
  "token_present": "Request includes auth token",
  "token_valid": "Token signature is valid",
  "token_not_expired": "Token is within validity period",
  "token_not_revoked": "Token hasn't been revoked",
  "user_authenticated": "All auth checks passed"
};
```

### 2. Implement with Predicate Logging

```typescript
async function authenticateUser(token: string, correlationId: string) {
  logSemantic({
    predicate: "token_present",
    passed: !!token,
    correlationId,
    scope: "domain.auth"
  });

  if (!token) return { authenticated: false };

  const isValid = verifySignature(token);

  logSemantic({
    predicate: "token_valid",
    passed: isValid,
    correlationId,
    scope: "domain.auth"
  });

  if (!isValid) return { authenticated: false };

  // Continue validating each predicate...
}
```

### 3. Test Each Predicate's Failure Mode

```typescript
describe("Authentication predicates", () => {
  it("token_present fails when token is null", async () => {
    await authenticateUser(null, "test-1");

    const log = logs.find(l => l.predicate === "token_present");
    expect(log.passed).toBe(false);
  });

  it("token_valid fails when signature is invalid", async () => {
    await authenticateUser("invalid-token", "test-2");

    const log = logs.find(l => l.predicate === "token_valid");
    expect(log.passed).toBe(false);
  });

  // Test EVERY predicate's failure mode
});
```

### 4. Monitor in Production

```typescript
// Dashboard shows predicate pass rates
function getPredicateHealth() {
  return {
    token_present: "99.9%",
    token_valid: "98.2%",      // ← Dropping!
    token_not_expired: "99.5%",
    token_not_revoked: "99.8%"
  };
}

// Alert when rates drop
// Investigate specific failures
// Discover new failure modes
// Add predicates to fill gaps
```

---

## When to Use This

Predicate logging is essential for:

- **External dependencies** - APIs, databases, third-party services
- **Multi-step workflows** - Checkout flows, data pipelines, distributed transactions
- **Semantic vs technical success** - When HTTP 200 doesn't mean the operation actually worked
- **Production debugging** - When you need to know exactly what failed, not just "something broke"
- **Team coordination** - When modules need stable contracts to integrate
- **Formal guarantees** - When you need runtime proof that invariants hold

---

## Migration Path

You don't need to rewrite everything. Start small:

**Week 1:** Pick one critical flow, map out predicates
**Week 2:** Implement root predicates, test failure modes
**Week 3:** Complete predicate tree, add validation library
**Week 4:** Build monitoring dashboard
**Week 5-6:** Prove value in production
**Week 7+:** Expand to next flow

**Effort:** 1-2 engineers, 4-6 weeks for first complete flow

See [`predicate-logging/references/migration-guide.md`](predicate-logging/references/migration-guide.md) for the complete 7-phase strategy.

---

## Documentation

### Understanding the Concept
- **[Traditional vs Predicate Comparison](Traditional-vs-Predicate-Comparison.md)** - Side-by-side visual comparison
- **[Complete System Summary](COMPLETE-SYSTEM-SUMMARY.md)** - Overview and philosophy

### Implementation
- **[SKILL.md](predicate-logging/SKILL.md)** - Complete specification with all 9 principles
- **[Operational Loop](predicate-logging/references/operational-loop.md)** - How to actually use this in development
- **[Runtime Validation](predicate-logging/references/runtime-validation.md)** - Enforcement library implementation
- **[Migration Guide](predicate-logging/references/migration-guide.md)** - Retrofit existing codebases

---

## The Core Insight

**Logs aren't debug noise. They're how you build the app.**

When you declare predicates first, you're forced to articulate:
- What should be true
- How to detect when it's not
- What the failure modes are

This is **design by contract** + **test-driven development** + **formal verification** rolled into one.

Each log is a monkey with a constrained typewriter. The environment is smart and scoped. Shakespeare emerges from composition of tiny, honest, checkable claims.

---

## The Operational Loop

**Development Phase:**
```
1. Define predicates (your assumptions about what should be true)
2. Implement with logging (log each predicate as you check it)
3. Test by deliberately breaking things (break each predicate on purpose)
4. Read the proof tree (see exactly which predicate failed)
5. Fix the specific predicate (not the whole system)
6. Repeat until all predicates pass (each failure teaches you something)
```

**Production Phase:**
```
7. Monitor predicate pass rates (which ones are dropping?)
8. Investigate failures (what's the new failure mode?)
9. Add predicates (fill gaps you discovered)
10. Fix and repeat (continuous improvement)
```

**You're not trying to avoid failures. You're trying to make failures informative.**

That's fail-upward development.

---

## Credits

This system synthesizes:
- **Formal logic** - Predicates, hierarchies, modal worlds
- **Practical engineering** - Catastrophic 200 problem, real-world patterns
- **Runtime validation** - Self-enforcing rules and proof trees
- **Development methodology** - Operational loop, deliberate failure testing

It's the bridge between elegant theory and "teams can actually use this."

---

## License

Apache 2.0

---

**"Each monkey is dumb. The environment is smart and scoped. Reliability emerges from tiny, honest, checkable claims."**
