# Migration Guide: Adopting Predicate Logging

This guide helps teams retrofit predicate logging onto existing codebases without requiring a full rewrite.

## Migration Philosophy

**Start small. Prove value. Expand.**

You don't need to convert your entire system at once. Begin with one critical flow, demonstrate the debugging power, then expand organically.

## Phase 1: Identify Your Critical Path

Pick **one** flow that:
- Is business-critical (auth, checkout, data pipeline)
- Causes frequent production issues
- Has unclear failure modes
- Would benefit most from better observability

**Example critical paths:**
- User authentication flow
- Payment processing
- Data ingestion pipeline
- Video playback initialization
- Search query processing

## Phase 2: Map the Predicate Tree

For your chosen flow, identify:

### 2.1 Required Predicates (Must Be True)
What conditions **must** hold for success?

**Auth example:**
```
1. token_present - Request includes token
2. token_parseable - Token is valid JWT
3. token_not_expired - Token timestamp is current
4. token_not_revoked - Token not in revocation list
5. user_exists - User ID from token exists in DB
6. user_active - User account is not suspended
```

### 2.2 Parent/Child Relationships
Which predicates depend on others?

```
token_present (root)
  └─ token_parseable
      └─ token_not_expired
          └─ token_not_revoked
              └─ user_exists
                  └─ user_active
```

### 2.3 Failure Modes
What can go wrong at each step?

- `token_present = false` → 401, redirect to login
- `token_parseable = false` → 401, token malformed
- `token_expired = true` → 401, refresh flow
- `token_revoked = true` → 401, force re-auth
- `user_not_exists = true` → 404, data loss
- `user_suspended = true` → 403, show suspension message

## Phase 3: Implement Incrementally

### 3.1 Start with Root Predicates

Begin at the top of your tree. Don't worry about children yet.

**Before:**
```typescript
async function authenticate(token: string) {
  if (!token) {
    return null;
  }
  
  const decoded = jwt.verify(token, SECRET);
  return decoded;
}
```

**After (Phase 3.1):**
```typescript
async function authenticate(
  token: string,
  correlationId: string
) {
  // Root predicate: token present
  logSemantic({
    event: "auth_token_checked",
    predicate: "token_present",
    passed: !!token,
    correlationId,
    scope: "domain.auth"
  });
  
  if (!token) {
    return null;
  }
  
  const decoded = jwt.verify(token, SECRET);
  return decoded;
}
```

**Value at this stage:**
- Know immediately when auth fails due to missing token vs other reasons
- Can measure: "What % of auth failures are missing tokens?"

### 3.2 Add Next Level Predicates

Once root predicates are stable, add their immediate children.

**After (Phase 3.2):**
```typescript
async function authenticate(
  token: string,
  correlationId: string
) {
  logSemantic({
    event: "auth_token_checked",
    predicate: "token_present",
    passed: !!token,
    correlationId,
    scope: "domain.auth"
  });
  
  if (!token) return null;
  
  // Child predicate: token parseable
  let decoded;
  try {
    decoded = jwt.verify(token, SECRET);
    
    logSemantic({
      event: "auth_token_parsed",
      predicate: "token_parseable",
      passed: true,
      correlationId,
      scope: "domain.auth"
    });
  } catch (e) {
    logSemantic({
      event: "auth_token_parse_failed",
      predicate: "token_parseable",
      passed: false,
      error: e.message,
      correlationId,
      scope: "domain.auth"
    });
    return null;
  }
  
  return decoded;
}
```

**Value at this stage:**
- Distinguish between "no token" vs "malformed token"
- Catch production tokens that somehow got corrupted

### 3.3 Complete the Tree

Continue down the hierarchy until all predicates are logged.

**After (Phase 3.3):**
```typescript
async function authenticate(
  token: string,
  correlationId: string
) {
  // Predicate 1: token present
  logSemantic({
    event: "auth_token_checked",
    predicate: "token_present",
    passed: !!token,
    correlationId,
    scope: "domain.auth"
  });
  
  if (!token) return null;
  
  // Predicate 2: token parseable
  let decoded;
  try {
    decoded = jwt.verify(token, SECRET);
    logSemantic({
      event: "auth_token_parsed",
      predicate: "token_parseable",
      passed: true,
      correlationId,
      scope: "domain.auth"
    });
  } catch (e) {
    logSemantic({
      event: "auth_token_parse_failed",
      predicate: "token_parseable",
      passed: false,
      error: e.message,
      correlationId,
      scope: "domain.auth"
    });
    return null;
  }
  
  // Predicate 3: token not expired
  const now = Date.now() / 1000;
  const notExpired = decoded.exp > now;
  
  logSemantic({
    event: "auth_token_expiry_checked",
    predicate: "token_not_expired",
    passed: notExpired,
    expiresAt: decoded.exp,
    currentTime: now,
    correlationId,
    scope: "domain.auth"
  });
  
  if (!notExpired) return null;
  
  // Predicate 4: token not revoked
  const revoked = await checkRevocationList(decoded.jti);
  
  logSemantic({
    event: "auth_token_revocation_checked",
    predicate: "token_not_revoked",
    passed: !revoked,
    tokenId: decoded.jti,
    correlationId,
    scope: "domain.auth"
  });
  
  if (revoked) return null;
  
  // Predicate 5: user exists
  const user = await db.users.findById(decoded.userId);
  
  logSemantic({
    event: "auth_user_lookup_complete",
    predicate: "user_exists",
    passed: !!user,
    userId: decoded.userId,
    correlationId,
    scope: "domain.auth"
  });
  
  if (!user) return null;
  
  // Predicate 6: user active
  const isActive = user.status === 'active';
  
  logSemantic({
    event: "auth_user_status_checked",
    predicate: "user_active",
    passed: isActive,
    userId: user.id,
    status: user.status,
    correlationId,
    scope: "domain.auth"
  });
  
  if (!isActive) return null;
  
  // All predicates passed
  logSemantic({
    event: "auth_complete",
    predicate: "authentication_successful",
    passed: true,
    userId: user.id,
    correlationId,
    scope: "domain.auth"
  });
  
  return { user, claims: decoded };
}
```

**Value at this stage:**
- Complete visibility into auth failures
- Know exactly which predicate failed for each request
- Can build monitoring: "Alert when user_exists failures spike"

## Phase 4: Add Correlation IDs

### 4.1 Generate Correlation IDs

Create unique IDs at request entry points:

```typescript
// Express middleware
app.use((req, res, next) => {
  req.correlationId = req.headers['x-correlation-id'] || 
    `req-${Date.now()}-${Math.random().toString(36)}`;
  next();
});

// AWS Lambda
export const handler = async (event) => {
  const correlationId = event.requestContext?.requestId || 
    `lambda-${Date.now()}-${Math.random().toString(36)}`;
  
  return processRequest(event, correlationId);
};
```

### 4.2 Thread Through Call Stack

Pass correlationId to all functions in the flow:

```typescript
app.post('/api/checkout', async (req, res) => {
  const { correlationId } = req;
  
  const user = await authenticate(req.token, correlationId);
  const cart = await getCart(user.id, correlationId);
  const order = await processOrder(cart, correlationId);
  
  res.json(order);
});
```

### 4.3 Cross-Service Propagation

Include correlation IDs in outgoing requests:

```typescript
async function callPaymentService(
  paymentData: PaymentData,
  correlationId: string
) {
  const response = await fetch('https://payment-service/charge', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-Correlation-ID': correlationId  // Propagate!
    },
    body: JSON.stringify(paymentData)
  });
  
  return response.json();
}
```

## Phase 5: Add Runtime Validation

### 5.1 Register Hierarchy

At application startup:

```typescript
// config/predicate-hierarchy.ts
import { registerPredicateHierarchy } from './predicate-logger';

export function initializePredicateHierarchy() {
  // Auth tree
  registerPredicateHierarchy("token_present", []);
  registerPredicateHierarchy("token_parseable", ["token_present"]);
  registerPredicateHierarchy("token_not_expired", ["token_parseable"]);
  registerPredicateHierarchy("token_not_revoked", ["token_not_expired"]);
  registerPredicateHierarchy("user_exists", ["token_not_revoked"]);
  registerPredicateHierarchy("user_active", ["user_exists"]);
  
  // Add other trees...
}
```

### 5.2 Enforce in Development

Use the validation library in dev/staging:

```typescript
if (process.env.NODE_ENV !== 'production') {
  // Strict validation in dev
  predicateLogger.setStrictMode(true);
}
```

This catches hierarchy violations before they reach production.

## Phase 6: Build Monitoring

### 6.1 Predicate Success Rates

```typescript
function calculatePredicateMetrics(correlationId: string) {
  const analysis = predicateLogger.analyzeRun(correlationId);
  
  metrics.gauge('auth.predicates.passed', 
    analysis.passedPredicates.length,
    { correlation_id: correlationId }
  );
  
  metrics.gauge('auth.predicates.failed',
    analysis.failedPredicates.length,
    { correlation_id: correlationId }
  );
  
  // Per-predicate metrics
  analysis.passedPredicates.forEach(p => {
    metrics.increment(`auth.predicate.${p}.success`);
  });
  
  analysis.failedPredicates.forEach(p => {
    metrics.increment(`auth.predicate.${p}.failure`);
  });
}
```

### 6.2 Proof Tree Dashboard

Build a dashboard showing:
- Most common failure predicates
- Predicate pass rates over time
- Proof trees for failed requests
- Orphan detection alerts

### 6.3 Alerting

Alert on anomalies:

```typescript
// Alert when a normally reliable predicate starts failing
if (getPredicateFailureRate('user_exists') > 0.01) {
  alert({
    title: 'User lookup failures spiking',
    severity: 'critical',
    description: 'user_exists predicate failing above threshold'
  });
}

// Alert on hierarchy violations
if (analysis.orphans.length > 0) {
  alert({
    title: 'Predicate hierarchy violation',
    severity: 'error',
    orphans: analysis.orphans
  });
}
```

## Phase 7: Expand to Other Flows

Once your first flow is stable:

1. **Choose next critical path** (e.g., checkout, data pipeline)
2. **Map predicates** for that flow
3. **Implement incrementally** (root → leaves)
4. **Add to monitoring** dashboard
5. **Repeat**

## Common Pitfalls

### Pitfall 1: Logging Before Testing

```typescript
// ❌ Bad: log claims success before actually testing
logSemantic({ predicate: "user_authenticated", passed: true });
const valid = await validateUser();
```

**Solution:** Only log after predicate evaluated.

### Pitfall 2: Composite Predicates Too Early

```typescript
// ❌ Bad: one log for multiple checks
const allGood = tokenValid && userExists && cartValid;
logSemantic({ predicate: "everything_ok", passed: allGood });
```

**Solution:** Log atomic predicates separately.

### Pitfall 3: Missing Correlation IDs

```typescript
// ❌ Bad: no way to track run
logSemantic({ predicate: "user_active", passed: true });
```

**Solution:** Always include correlationId.

### Pitfall 4: Ignoring Hierarchy

```typescript
// ❌ Bad: child log without parent validation
logSemantic({ 
  predicate: "payment_processed",  // child
  passed: true 
});
// But did cart_validated (parent) pass? Unknown!
```

**Solution:** Use runtime validation library to enforce.

## Success Metrics

You'll know the migration is successful when:

1. **Mean Time to Debug (MTTD) decreases** - Finding root causes is faster
2. **False positive alerts decrease** - Monitoring is more precise
3. **Silent failures become visible** - Catastrophic 200s are caught
4. **Teams request predicate logging** - Developers see the value
5. **Proof trees replace theories** - Debugging uses logs, not guesswork

## Timeline Example

**Week 1:** Pick critical path, map predicates  
**Week 2:** Implement root predicates, add correlation IDs  
**Week 3:** Complete predicate tree, add validation  
**Week 4:** Build monitoring dashboard  
**Week 5-6:** Prove value, document learnings  
**Week 7+:** Expand to next critical path

**Expected effort:** 1-2 engineers, 4-6 weeks for first complete flow.

## Getting Help

When stuck:

1. **Map the tree visually** - Draw parent/child relationships
2. **Start smaller** - Pick a sub-flow if full flow is too complex
3. **Use validation library** - Let it catch violations
4. **Review predicates** - Are they atomic? Honest? Scoped?
5. **Check examples** - Refer to patterns in main SKILL.md

## Conclusion

Predicate logging is not an all-or-nothing migration. Each flow you convert provides immediate value. The system is designed for **incremental adoption** - valid subtrees are real logic even before the full universe is built.

Start small. One flow. Prove the debugging power. Then expand organically.
