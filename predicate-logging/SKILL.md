---
name: predicate-logging
description: Generate logging as a formal logic system where each log is a typed predicate claim about runtime state. Use when writing any code with external dependencies (APIs, databases, services) or multi-step workflows. Solves the "catastrophic 200" problem where technical success doesn't guarantee semantic success. Creates hierarchical proof trees where child logs prove their parent predicates held. Enables modular development where teams build against stable contracts and snap together valid world subtrees.
---

# Predicate Logging System

## The Core Insight

**Logs are not debug strings. They are typed predicate claims that form a runtime proof tree.**

Traditional logging asks: "Did the operation complete?"  
Predicate logging asks: "What truth claims can we make about this run's state?"

This transforms logging from noise into a **self-validating logic system**.

## The Problem This Solves

### The Catastrophic 200

Modern systems fail silently with technical success:
- ✓ 200 OK but response body is malformed JSON
- ✓ 200 OK but data is stale by hours
- ✓ 200 OK but it's an error wrapped in success envelope
- ✓ 200 OK but only 3/5 microservices succeeded
- ✓ 200 OK but the async job it triggered will fail
- ✓ 200 OK but rate limiting just activated

Traditional logs say "success" while business logic dies. Metrics show 99.9% uptime while users scream.

**Root cause:** Logs track execution, not state validation.

## The Nine Core Principles

### 1. Claim Rule — A Log Is a Single Predicate

Every log corresponds to **one explicit predicate your code just evaluated**.

```typescript
// ✅ Good: one explicit predicate
if (!player) {
  logSemantic({
    event: "video_player_ready",
    predicate: "player_exists",
    passed: false,
    scope: "domain.video",
    correlationId,
  });
  return;
}

logSemantic({
  event: "video_player_ready",
  predicate: "player_exists", 
  passed: true,
  scope: "domain.video",
  correlationId,
});
```

**Key:** The log name and fields describe only that predicate. No "vibes logs" - these are falsifiable claims.

### 2. Honesty Rule — Only Log What You Actually Checked

Logs must only assert things the code **concretely tested**.

```typescript
// ❌ Bad: claims auth before checking
logSemantic({ event: "user_authenticated", passed: true });
verifyToken(token);
checkRevocation(token);

// ✅ Good: log only after full predicate evaluated
const tokenValid = verifyToken(token);
const notRevoked = !isRevoked(token);
const isAuth = tokenValid && notRevoked;

logSemantic({
  event: "user_authenticated",
  predicate: "token_valid_and_not_revoked",
  passed: isAuth,
  scope: "domain.auth",
  correlationId,
});
```

If the log says "X passed," X must really have been evaluated.

### 3. Atomic First — Composite Truths Are Inferred

Prefer atomic predicates over composite logs. Combinations are inferred from multiple atomic logs with the same `correlationId`.

```typescript
// Atomic predicates (preferred)
logSemantic({ 
  event: "db_connection_established",
  predicate: "db_connected",
  passed: true, 
  correlationId 
});

logSemantic({ 
  event: "cache_seeded",
  predicate: "cache_ready",
  passed: true, 
  correlationId 
});

// Analysis infers: db_connected ∧ cache_ready ⇒ storage_ready
```

**Rule of thumb:** Log atomic facts; infer combinations. Composite logs are for convenience, not truth.

### 4. Hierarchy Rule — Children Only Exist in Valid Worlds

Logs form a **hierarchy of worlds**. Child logs only exist when parent predicates passed in this run.

```typescript
// Parent world
if (!playerReady) {
  logSemantic({
    event: "video_player_ready",
    predicate: "player_ready",
    passed: false,
    correlationId,
  });
  return; // children forbidden
}

logSemantic({
  event: "video_player_ready", 
  predicate: "player_ready",
  passed: true,
  correlationId,
});

// Child only exists in valid parent world
logSemantic({
  event: "video_playback_started",
  predicate: "playback_started", 
  passed: true,
  correlationId,
});
```

**Inference guarantee:** Observing `video_playback_started` proves `video_player_ready` held in this run.

Presence of a child is **proof** its parents were satisfied.

### 5. Scope Rule — Be Explicit About What World You're In

Every log has a scope indicating its proximity to domain intent vs plumbing.

```typescript
interface SemanticLog {
  event: string;
  predicate: string;
  passed: boolean;
  scope: "infra" | "service" | "domain" | "user-facing";
  channel?: string; // e.g., "debug:video", "debug:ai-router"
  correlationId: string;
}

logSemantic({
  event: "video_player_ready",
  predicate: "player_ready",
  passed: true,
  scope: "domain.video",
  channel: "debug:video",
  correlationId,
});
```

Channels control verbosity at runtime without changing semantics.

### 6. Correlation Rule — Claims Belong to a Specific Run

All parent/child and conjunction reasoning is **per correlation ID**.

```typescript
type CorrelationId = string;

interface SemanticLogBase {
  event: string;
  predicate: string;
  passed: boolean;
  correlationId: CorrelationId;
  scope: string;
  channel?: string;
}
```

A "run" = one request, job, or workflow instance. No cross-run inference.

### 7. Modular Worlds — Valid Subsets Are Real Logic

You don't need to build the whole universe at once. Any internally consistent subset that:
1. Respects Claim Rule (logs assert what they test)
2. Respects Hierarchy Rule (children in valid parent worlds)
3. Connects to at least one valid parent

…is a valid logic module.

**This means:** Teams can build independent subtrees that declare parent requirements and snap into the larger system once those parents are wired up.

```typescript
// Checkout module assumes parent world: user_authenticated ∧ catalog_available
logSemantic({ 
  event: "cart_validated",
  predicate: "cart_has_valid_items",
  passed: true,
  scope: "domain.checkout",
  correlationId 
});

logSemantic({
  event: "shipping_option_selected", 
  predicate: "valid_shipping_chosen",
  passed: true,
  scope: "domain.checkout",
  correlationId
});
```

As long as each module states entry predicates honestly and emits scoped logs, integration is just connecting valid worlds.

### 8. Required Shape First — Stable Schemas as Contracts

Define minimal return schemas as **required, stable contract keys** that won't change without versioning.

```typescript
// Required fields (parent world guarantees)
type VideoMetadataRequired = {
  id: string;
  title: string;
  durationMs: number;
  playable: boolean; // derived from internal checks
};

// Optional fields (child predicate worlds)
type VideoMetadataOptional = {
  tags?: string[];
  thumbnailUrl?: string; // when thumbnail_ready = true
  cdnRegion?: string;
};

type VideoMetadataResponse = VideoMetadataRequired & VideoMetadataOptional;
```

**Workflow:**
1. Define required shape
2. Teams mock dummy responses satisfying the schema
3. Build UI/logic against stable keys
4. Real implementation proves predicates in logs
5. Their code "just works" against real thing

Optional fields are feature upgrades, not breaking changes.

### 9. Shakespeare Monkeys — The Intuition

**Classic joke:** Infinite monkeys, infinite typewriters, infinite time → eventually type Shakespeare.

**Our version:** First try, because:
- Monkey 1's typewriter only accepts correct first character
- Monkey 2 only starts after Monkey 1 succeeds, only accepts correct second character
- Monkey 3 only starts after Monkey 2...

Each monkey is dumb. The environment is smart and scoped.

**That's this system:**
- Each log predicate has a tiny scoped job
- Parent logs must succeed before children exist
- Full behavior is locally verified steps stitched together

Instead of one giant fragile brain, you get a chain of tiny honest checkable brains that compose into reliability.

## Implementation Patterns

### Basic Predicate Structure

```typescript
function logSemantic<T extends Record<string, any>>(
  log: SemanticLogBase & T
): void {
  // Runtime validation
  if (!log.correlationId) {
    throw new Error("correlationId required for semantic logs");
  }
  
  // Store for analysis
  semanticLogs.push({
    ...log,
    timestamp: Date.now(),
  });
  
  // Traditional logging
  console.log(JSON.stringify(log));
}
```

### API Call Pattern

```typescript
async function fetchUserProfile(userId: string, correlationId: string) {
  const response = await fetch(`/api/users/${userId}`);
  
  // Predicate: HTTP request succeeded
  if (!response.ok) {
    logSemantic({
      event: "user_profile_fetch_failed",
      predicate: "http_request_successful",
      passed: false,
      status: response.status,
      scope: "service.api",
      correlationId,
    });
    return null;
  }
  
  logSemantic({
    event: "user_profile_http_success",
    predicate: "http_request_successful",
    passed: true,
    scope: "service.api",
    correlationId,
  });
  
  // Predicate: Response is parseable JSON
  let data;
  try {
    data = await response.json();
  } catch (e) {
    logSemantic({
      event: "user_profile_parse_failed",
      predicate: "response_parseable",
      passed: false,
      parseError: e.message,
      scope: "service.api",
      correlationId,
    });
    return null;
  }
  
  logSemantic({
    event: "user_profile_parsed",
    predicate: "response_parseable",
    passed: true,
    scope: "service.api",
    correlationId,
  });
  
  // Predicate: User data exists (not null/empty response)
  if (!data.user) {
    const isNewUser = await checkIfNewUser(userId);
    
    logSemantic({
      event: "user_profile_data_missing",
      predicate: "user_data_present",
      passed: false,
      isNewUser,
      scope: "domain.users",
      correlationId,
    });
    
    if (!isNewUser) {
      // This is catastrophic - existing user lost data
      logSemantic({
        event: "data_loss_detected",
        predicate: "existing_user_has_data",
        passed: false,
        userId,
        severity: "critical",
        scope: "domain.users",
        correlationId,
      });
    }
    
    return null;
  }
  
  logSemantic({
    event: "user_profile_data_present",
    predicate: "user_data_present",
    passed: true,
    scope: "domain.users",
    correlationId,
  });
  
  // Predicate: Data is fresh (not stale)
  const dataAge = Date.now() - new Date(data.user.updatedAt).getTime();
  const STALENESS_THRESHOLD = 3600000; // 1 hour
  const isFresh = dataAge < STALENESS_THRESHOLD;
  
  logSemantic({
    event: "user_profile_freshness_checked",
    predicate: "data_is_fresh",
    passed: isFresh,
    dataAgeMs: dataAge,
    thresholdMs: STALENESS_THRESHOLD,
    scope: "domain.users",
    correlationId,
  });
  
  return data;
}
```

### Empty Array Detection

```typescript
async function getActiveUsers(correlationId: string) {
  const results = await db.query("SELECT * FROM users WHERE active = true");
  
  if (results.length === 0) {
    // Predicate: Some users exist in system
    const totalUsers = await db.query("SELECT COUNT(*) FROM users");
    const usersExist = totalUsers > 0;
    
    logSemantic({
      event: "active_users_query_empty",
      predicate: "users_exist_but_none_active",
      passed: usersExist,
      totalUsers,
      scope: "domain.users",
      correlationId,
    });
    
    if (usersExist) {
      // This might indicate: bulk deactivation, migration issue, query bug
      logSemantic({
        event: "all_users_inactive_anomaly",
        predicate: "normal_active_user_ratio",
        passed: false,
        severity: "warn",
        scope: "domain.users",
        correlationId,
      });
    }
  } else {
    logSemantic({
      event: "active_users_found",
      predicate: "active_users_exist",
      passed: true,
      count: results.length,
      scope: "domain.users",
      correlationId,
    });
  }
  
  return results;
}
```

### Multi-Step Workflow

```typescript
async function processOrder(orderId: string, correlationId: string) {
  // Step 1: Fetch order
  const order = await getOrder(orderId);
  
  if (!order) {
    logSemantic({
      event: "order_fetch_complete",
      predicate: "order_exists",
      passed: false,
      orderId,
      scope: "domain.orders",
      correlationId,
    });
    return { success: false, reason: "NOT_FOUND" };
  }
  
  logSemantic({
    event: "order_fetch_complete",
    predicate: "order_exists",
    passed: true,
    orderId,
    scope: "domain.orders",
    correlationId,
  });
  
  // Step 2: Validate inventory (child of order_exists world)
  const available = await checkInventory(order.items);
  
  logSemantic({
    event: "inventory_check_complete",
    predicate: "all_items_available",
    passed: available.allAvailable,
    requestedCount: order.items.length,
    availableCount: available.availableCount,
    scope: "domain.inventory",
    correlationId,
  });
  
  if (!available.allAvailable) {
    return { success: false, reason: "INSUFFICIENT_INVENTORY" };
  }
  
  // Step 3: Process payment (child of inventory_available world)
  const payment = await chargeCard(order.payment);
  
  logSemantic({
    event: "payment_processed",
    predicate: "payment_successful",
    passed: payment.success,
    amount: order.total,
    paymentId: payment.id,
    scope: "domain.payments",
    correlationId,
  });
  
  if (!payment.success) {
    return { success: false, reason: "PAYMENT_FAILED" };
  }
  
  // Step 4: Reserve inventory (child of payment_successful world)
  const updated = await reserveInventory(order.items);
  
  logSemantic({
    event: "inventory_reserved",
    predicate: "inventory_reservation_successful",
    passed: updated.success,
    scope: "domain.inventory",
    correlationId,
  });
  
  if (!updated.success) {
    // CRITICAL: Payment succeeded but inventory failed
    logSemantic({
      event: "payment_inventory_mismatch",
      predicate: "payment_and_inventory_consistent",
      passed: false,
      requiresManualIntervention: true,
      severity: "critical",
      orderId,
      paymentId: payment.id,
      scope: "domain.orders",
      correlationId,
    });
    
    await initiateRefund(payment.id);
    return { success: false, reason: "INVENTORY_RESERVATION_FAILED" };
  }
  
  // All predicates passed - order is valid
  logSemantic({
    event: "order_processing_complete",
    predicate: "order_fully_processed",
    passed: true,
    orderId,
    paymentId: payment.id,
    itemCount: order.items.length,
    scope: "domain.orders",
    correlationId,
  });
  
  return { success: true, orderId };
}
```

### Partial Success States

```typescript
async function processBatch(items: Item[], correlationId: string) {
  const results = await Promise.allSettled(
    items.map(item => processItem(item, correlationId))
  );
  
  const successCount = results.filter(r => r.status === 'fulfilled').length;
  const allSucceeded = successCount === items.length;
  
  logSemantic({
    event: "batch_processing_complete",
    predicate: "all_items_processed_successfully",
    passed: allSucceeded,
    successCount,
    totalCount: items.length,
    successRate: successCount / items.length,
    scope: "domain.batch",
    correlationId,
  });
  
  if (!allSucceeded) {
    // Log which specific predicates failed
    results.forEach((result, idx) => {
      if (result.status === 'rejected') {
        logSemantic({
          event: "batch_item_failed",
          predicate: "item_processed_successfully",
          passed: false,
          itemId: items[idx].id,
          error: result.reason,
          scope: "domain.batch",
          correlationId,
        });
      }
    });
  }
  
  return { successCount, totalCount: items.length, results };
}
```

### Performance Predicates

```typescript
async function fetchWithLatencyCheck(
  endpoint: string,
  correlationId: string
) {
  const startTime = Date.now();
  const data = await fetch(endpoint);
  const latency = Date.now() - startTime;
  
  const LATENCY_THRESHOLD = 200; // ms
  
  logSemantic({
    event: "fetch_complete",
    predicate: "fetch_latency_acceptable",
    passed: latency < LATENCY_THRESHOLD,
    actualLatency: latency,
    threshold: LATENCY_THRESHOLD,
    endpoint,
    scope: "service.cache",
    correlationId,
  });
  
  return data;
}
```

## Language-Specific Implementations

### Python

```python
from typing import TypedDict, Optional, Literal
from dataclasses import dataclass
import time

class SemanticLog(TypedDict):
    event: str
    predicate: str
    passed: bool
    correlationId: str
    scope: Literal["infra", "service", "domain", "user-facing"]
    channel: Optional[str]

def log_semantic(**kwargs: any) -> None:
    """Log a semantic predicate claim."""
    if "correlationId" not in kwargs:
        raise ValueError("correlationId required")
    
    log_entry = {
        **kwargs,
        "timestamp": time.time()
    }
    
    # Store for analysis
    semantic_logs.append(log_entry)
    
    # Traditional logging
    print(json.dumps(log_entry))

def fetch_user_profile(user_id: str, correlation_id: str) -> Optional[dict]:
    response = requests.get(f"/api/users/{user_id}")
    
    # Predicate: HTTP success
    if not response.ok:
        log_semantic(
            event="user_profile_fetch_failed",
            predicate="http_request_successful",
            passed=False,
            status=response.status_code,
            scope="service.api",
            correlationId=correlation_id
        )
        return None
    
    log_semantic(
        event="user_profile_http_success",
        predicate="http_request_successful",
        passed=True,
        scope="service.api",
        correlationId=correlation_id
    )
    
    # Predicate: Response parseable
    try:
        data = response.json()
    except json.JSONDecodeError as e:
        log_semantic(
            event="user_profile_parse_failed",
            predicate="response_parseable",
            passed=False,
            parseError=str(e),
            scope="service.api",
            correlationId=correlation_id
        )
        return None
    
    log_semantic(
        event="user_profile_parsed",
        predicate="response_parseable",
        passed=True,
        scope="service.api",
        correlationId=correlation_id
    )
    
    # Predicate: User data present
    if not data.get("user"):
        log_semantic(
            event="user_profile_data_missing",
            predicate="user_data_present",
            passed=False,
            scope="domain.users",
            correlationId=correlation_id
        )
        return None
    
    log_semantic(
        event="user_profile_data_present",
        predicate="user_data_present",
        passed=True,
        scope="domain.users",
        correlationId=correlation_id
    )
    
    return data
```

## Testing Implications

Since logs are predicates, testing becomes verification of predicate truth:

### Unit Tests - Verify Predicates Fire Correctly

```typescript
test("logs correct predicate when user data missing", async () => {
  const logs: SemanticLog[] = [];
  
  mockFetch.mockResolvedValue({
    ok: true,
    json: async () => ({ user: null })
  });
  
  await fetchUserProfile("user123", "test-correlation-id");
  
  const dataPresentLog = logs.find(
    l => l.predicate === "user_data_present"
  );
  
  expect(dataPresentLog).toBeDefined();
  expect(dataPresentLog.passed).toBe(false);
});
```

### Integration Tests - Verify Parent/Child Relationships

```typescript
test("child logs only exist when parent succeeds", async () => {
  const logs: SemanticLog[] = [];
  
  await processOrder("order123", "test-correlation-id");
  
  const paymentLog = logs.find(l => l.predicate === "payment_successful");
  const inventoryLog = logs.find(l => l.predicate === "inventory_reservation_successful");
  
  if (inventoryLog) {
    // If child exists, parent must have passed
    expect(paymentLog).toBeDefined();
    expect(paymentLog.passed).toBe(true);
  }
});
```

### Production Monitoring - Expected Predicate Trees

```typescript
function analyzePredicateTree(correlationId: string): HealthCheck {
  const logs = getLogsByCorrelation(correlationId);
  
  const requiredPredicates = [
    "http_request_successful",
    "response_parseable",
    "user_data_present"
  ];
  
  const missingPredicates = requiredPredicates.filter(
    p => !logs.some(l => l.predicate === p && l.passed)
  );
  
  return {
    healthy: missingPredicates.length === 0,
    missingPredicates
  };
}
```

## Migration Path

Retrofitting existing codebases:

1. **Identify critical paths** - Checkout, auth, data load
2. **Define required predicates** - What must be true for each path?
3. **Add semantic logs incrementally** - Start with top-level predicates
4. **Expand hierarchy** - Add child predicates as you go deeper
5. **Validate with tests** - Verify predicate trees form correctly

See `references/migration-guide.md` for complete 7-phase migration strategy.

Start small. One critical flow. Prove the value. Expand.

## The Operational Loop: Fail-Upward Development

Predicate logging isn't just for observability - it's a **development methodology** where you deliberately test against predicates to discover what you don't know.

See `references/operational-loop.md` for the complete process, but the core loop is:

**Development:**
1. **Define predicates first** - What should be true? (Your assumptions)
2. **Implement with logging** - Log each predicate as you check it
3. **Test by deliberately breaking things** - Test each predicate's failure mode
4. **Read the proof tree** - See exactly which predicate failed
5. **Fix the specific predicate** - Not the whole system
6. **Repeat until all predicates pass** - Each failure teaches you something

**Production:**
7. **Monitor predicate pass rates** - Which are dropping?
8. **Investigate failures** - What's the new failure mode we didn't know about?
9. **Add predicates** - Fill the gaps you discovered
10. **Fix and repeat** - Continuous improvement through structured failure

**Key insight:** You're not trying to write perfect code on the first try. You're building a system that tells you **exactly what's wrong** when your assumptions are incorrect.

This is **fail-upward development** - each failure makes the system more robust because you learned something specific and measurable. The predicate tree shows you precisely what failed, not just "something broke somewhere."

## Summary

**Traditional logging = execution trace**  
**Predicate logging = runtime proof tree**

When you:
- Log only evaluated predicates (Honesty)
- Structure parent/child relationships (Hierarchy)
- Scope by correlation ID (Correlation)
- Build modular subtrees (Modularity)

You get:
- Self-documenting architecture
- Formal verification of runtime behavior
- Team coordination via contracts
- Debugging that's actually possible

Each log is a monkey with a constrained typewriter. The environment is smart and scoped. Shakespeare emerges from composition of tiny, honest, checkable claims.
