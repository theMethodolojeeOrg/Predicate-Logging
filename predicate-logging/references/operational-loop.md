# Operational Loop: Test, Fail, Fix, Repeat

The predicate logging system isn't just for observability - it's a **development methodology** where you deliberately test against your predicates to discover what you don't know yet.

## The Core Loop

```
1. Define predicates (what should be true)
2. Implement code
3. Test against predicates
4. Read the proof tree
5. Fix what failed
6. Repeat until all predicates pass
```

This is **fail-upward development** - each failure teaches you something specific about your system's reality vs your assumptions.

## Why This Works

Traditional development:
```
Write code → Hope it works → Debug when it breaks in prod → ???
```

Predicate development:
```
Declare predicates → Write code → Test → Read which predicate failed → Fix that specific thing
```

**You can't fix what you can't measure. Predicates make failures measurable.**

## Phase 1: Define Your Predicates First

Before writing implementation, declare what SHOULD be true.

### Example: Building a Video Player

```typescript
// Define predicates BEFORE implementing
const videoPlayerPredicates = {
  // Infrastructure
  "player_library_loaded": {
    parents: [],
    test: () => typeof VideoPlayer !== 'undefined'
  },
  
  // Configuration
  "player_container_exists": {
    parents: ["player_library_loaded"],
    test: () => document.getElementById('video-container') !== null
  },
  
  // Initialization
  "player_instance_created": {
    parents: ["player_container_exists"],
    test: (player) => player !== null && player !== undefined
  },
  
  // Video loading
  "video_url_valid": {
    parents: ["player_instance_created"],
    test: (url) => url && url.startsWith('http')
  },
  
  "video_metadata_loaded": {
    parents: ["video_url_valid"],
    test: (metadata) => metadata && metadata.duration > 0
  },
  
  // Playback
  "video_ready_to_play": {
    parents: ["video_metadata_loaded"],
    test: (player) => player.readyState >= 3
  },
  
  "playback_started": {
    parents: ["video_ready_to_play"],
    test: (player) => !player.paused && player.currentTime > 0
  }
};
```

**Key insight:** You're declaring your ASSUMPTIONS about what must be true. Many will be wrong. That's good.

## Phase 2: Implement with Predicate Logging

```typescript
async function initializeVideoPlayer(
  videoUrl: string,
  correlationId: string
) {
  // Predicate 1: Library loaded
  const libraryLoaded = typeof VideoPlayer !== 'undefined';
  
  logSemantic({
    event: "video_player_library_check",
    predicate: "player_library_loaded",
    passed: libraryLoaded,
    correlationId,
    scope: "infra.video"
  });
  
  if (!libraryLoaded) {
    return { success: false, reason: "LIBRARY_NOT_LOADED" };
  }
  
  // Predicate 2: Container exists
  const container = document.getElementById('video-container');
  
  logSemantic({
    event: "video_container_check",
    predicate: "player_container_exists",
    passed: container !== null,
    correlationId,
    scope: "infra.video"
  });
  
  if (!container) {
    return { success: false, reason: "CONTAINER_NOT_FOUND" };
  }
  
  // Predicate 3: Instance created
  let player;
  try {
    player = new VideoPlayer(container);
    
    logSemantic({
      event: "player_instance_created",
      predicate: "player_instance_created",
      passed: true,
      correlationId,
      scope: "domain.video"
    });
  } catch (e) {
    logSemantic({
      event: "player_creation_failed",
      predicate: "player_instance_created",
      passed: false,
      error: e.message,
      correlationId,
      scope: "domain.video"
    });
    return { success: false, reason: "PLAYER_CREATION_FAILED" };
  }
  
  // Continue with other predicates...
  
  return { success: true, player };
}
```

## Phase 3: Test Deliberately Against Each Predicate

**Don't just test the happy path. Test each predicate's failure mode.**

### Test Suite Structure

```typescript
describe("Video Player Initialization", () => {
  describe("Predicate: player_library_loaded", () => {
    it("passes when VideoPlayer is defined", async () => {
      // Setup: ensure library exists
      global.VideoPlayer = MockVideoPlayer;
      
      const result = await initializeVideoPlayer(
        "http://example.com/video.mp4",
        "test-correlation-1"
      );
      
      const logs = getLogsByCorrelation("test-correlation-1");
      const libraryLog = logs.find(
        l => l.predicate === "player_library_loaded"
      );
      
      expect(libraryLog.passed).toBe(true);
    });
    
    it("fails when VideoPlayer is not loaded", async () => {
      // DELIBERATELY BREAK IT
      delete global.VideoPlayer;
      
      const result = await initializeVideoPlayer(
        "http://example.com/video.mp4",
        "test-correlation-2"
      );
      
      const logs = getLogsByCorrelation("test-correlation-2");
      const libraryLog = logs.find(
        l => l.predicate === "player_library_loaded"
      );
      
      // Verify the predicate correctly detected failure
      expect(libraryLog.passed).toBe(false);
      expect(result.success).toBe(false);
      expect(result.reason).toBe("LIBRARY_NOT_LOADED");
    });
  });
  
  describe("Predicate: player_container_exists", () => {
    it("passes when container exists in DOM", async () => {
      global.VideoPlayer = MockVideoPlayer;
      document.body.innerHTML = '<div id="video-container"></div>';
      
      const result = await initializeVideoPlayer(
        "http://example.com/video.mp4",
        "test-correlation-3"
      );
      
      const logs = getLogsByCorrelation("test-correlation-3");
      const containerLog = logs.find(
        l => l.predicate === "player_container_exists"
      );
      
      expect(containerLog.passed).toBe(true);
    });
    
    it("fails when container missing from DOM", async () => {
      global.VideoPlayer = MockVideoPlayer;
      // DELIBERATELY BREAK IT
      document.body.innerHTML = ''; 
      
      const result = await initializeVideoPlayer(
        "http://example.com/video.mp4",
        "test-correlation-4"
      );
      
      const logs = getLogsByCorrelation("test-correlation-4");
      const containerLog = logs.find(
        l => l.predicate === "player_container_exists"
      );
      
      expect(containerLog.passed).toBe(false);
      expect(result.reason).toBe("CONTAINER_NOT_FOUND");
    });
  });
  
  // Test EVERY predicate's failure mode
  // This is where you discover your wrong assumptions
});
```

## Phase 4: Read the Proof Tree

When tests fail, the proof tree tells you EXACTLY where reality diverged from assumptions.

```typescript
test("full initialization with all predicates", async () => {
  const result = await initializeVideoPlayer(
    "http://example.com/video.mp4",
    "test-full-init"
  );
  
  // Visualize what happened
  console.log(predicateLogger.buildProofTree("test-full-init"));
});
```

Output:
```
✓ player_library_loaded
  ✓ player_container_exists
    ✓ player_instance_created
      ✗ video_url_valid
```

**Reading:** We got as far as creating the player instance, but the URL validation failed. The video URL predicate is the problem.

## Phase 5: Fix What Failed

Look at the specific predicate that failed:

```typescript
const logs = getLogsByCorrelation("test-full-init");
const urlLog = logs.find(l => l.predicate === "video_url_valid");

console.log(urlLog);
// {
//   predicate: "video_url_valid",
//   passed: false,
//   url: undefined,  // AHA! URL is undefined
//   correlationId: "test-full-init"
// }
```

**Discovery:** Our test didn't pass the URL correctly. Fix:

```typescript
// Before (wrong assumption)
const player = new VideoPlayer(container);
player.loadVideo(videoUrl);  // We assumed this would work

// After (reality)
const player = new VideoPlayer(container);

// Predicate: URL is valid BEFORE trying to load
logSemantic({
  event: "video_url_validated",
  predicate: "video_url_valid",
  passed: videoUrl && videoUrl.startsWith('http'),
  url: videoUrl,
  correlationId,
  scope: "domain.video"
});

if (!videoUrl || !videoUrl.startsWith('http')) {
  return { success: false, reason: "INVALID_URL" };
}

await player.loadVideo(videoUrl);
```

## Phase 6: Repeat Until All Predicates Pass

Keep testing, finding failures, fixing them, until:

```
✓ player_library_loaded
  ✓ player_container_exists
    ✓ player_instance_created
      ✓ video_url_valid
        ✓ video_metadata_loaded
          ✓ video_ready_to_play
            ✓ playback_started
```

**Now you know:** All your assumptions held for this run.

## Real Example: Discovering Unknown Failure Modes

### Iteration 1: Initial Implementation

```typescript
async function fetchUserData(userId: string, correlationId: string) {
  const response = await fetch(`/api/users/${userId}`);
  
  logSemantic({
    predicate: "http_request_successful",
    passed: response.ok,
    correlationId,
    scope: "service.api"
  });
  
  if (!response.ok) return null;
  
  const data = await response.json();
  
  logSemantic({
    predicate: "user_data_present",
    passed: !!data.user,
    correlationId,
    scope: "domain.users"
  });
  
  return data;
}
```

### Test 1: Happy Path

```typescript
test("fetches user successfully", async () => {
  mockFetch.mockResolvedValue({
    ok: true,
    json: async () => ({ user: { id: 1, name: "Alice" } })
  });
  
  const result = await fetchUserData("1", "test-1");
  
  const tree = predicateLogger.buildProofTree("test-1");
  console.log(tree);
  // ✓ http_request_successful
  //   ✓ user_data_present
});
```

**Status:** Happy path works!

### Test 2: Deliberate Failure - What if JSON parsing fails?

```typescript
test("handles malformed JSON", async () => {
  mockFetch.mockResolvedValue({
    ok: true,
    json: async () => { throw new SyntaxError("Unexpected token") }
  });
  
  await fetchUserData("1", "test-2");
  
  const tree = predicateLogger.buildProofTree("test-2");
  console.log(tree);
  // ✓ http_request_successful
  //   (no user_data_present log - code crashed!)
});
```

**Discovery:** We never logged whether JSON parsing succeeded! Our predicate tree has a GAP.

### Iteration 2: Fix the Gap

```typescript
async function fetchUserData(userId: string, correlationId: string) {
  const response = await fetch(`/api/users/${userId}`);
  
  logSemantic({
    predicate: "http_request_successful",
    passed: response.ok,
    correlationId,
    scope: "service.api"
  });
  
  if (!response.ok) return null;
  
  // NEW: Add predicate for JSON parsing
  let data;
  try {
    data = await response.json();
    
    logSemantic({
      predicate: "response_parseable",
      passed: true,
      correlationId,
      scope: "service.api"
    });
  } catch (e) {
    logSemantic({
      predicate: "response_parseable",
      passed: false,
      error: e.message,
      correlationId,
      scope: "service.api"
    });
    return null;
  }
  
  logSemantic({
    predicate: "user_data_present",
    passed: !!data.user,
    correlationId,
    scope: "domain.users"
  });
  
  return data;
}
```

**Now test again:**

```
✓ http_request_successful
  ✗ response_parseable (error: "Unexpected token")
```

Perfect! The gap is filled.

### Test 3: What if data is stale?

```typescript
test("detects stale data", async () => {
  const oneHourAgo = Date.now() - 3600000;
  
  mockFetch.mockResolvedValue({
    ok: true,
    json: async () => ({ 
      user: { 
        id: 1, 
        name: "Alice",
        updatedAt: oneHourAgo
      } 
    })
  });
  
  await fetchUserData("1", "test-3");
  
  const tree = predicateLogger.buildProofTree("test-3");
  console.log(tree);
  // ✓ http_request_successful
  //   ✓ response_parseable
  //     ✓ user_data_present
  //       (no staleness check - another gap!)
});
```

**Discovery:** We're not checking data freshness!

### Iteration 3: Add Staleness Predicate

```typescript
// After user_data_present check...

const dataAge = Date.now() - new Date(data.user.updatedAt).getTime();
const STALENESS_THRESHOLD = 3600000; // 1 hour

logSemantic({
  predicate: "data_is_fresh",
  passed: dataAge < STALENESS_THRESHOLD,
  dataAgeMs: dataAge,
  thresholdMs: STALENESS_THRESHOLD,
  correlationId,
  scope: "domain.users"
});

if (dataAge >= STALENESS_THRESHOLD) {
  // Trigger background refresh but still return stale data
  triggerBackgroundRefresh(userId);
}
```

**Now test:**

```
✓ http_request_successful
  ✓ response_parseable
    ✓ user_data_present
      ✗ data_is_fresh (dataAge: 3600000, threshold: 3600000)
```

We discovered a previously invisible failure mode!

## The Pattern: Fail Upward

Each iteration:
1. **Test deliberately** - Break something on purpose
2. **Read the proof tree** - See exactly where it failed
3. **Discover gaps** - Find predicates you didn't know you needed
4. **Add predicates** - Fill the gaps
5. **Test again** - Verify the gap is filled

**This is how you discover what you don't know.**

## Production: The Loop Never Stops

In production, the loop continues:

### Week 1
```
✓ http_request_successful (99.9%)
  ✓ response_parseable (99.8%)
    ✓ user_data_present (98.5%)
```

Everything looks good!

### Week 2
```
✓ http_request_successful (99.9%)
  ✓ response_parseable (99.8%)
    ✓ user_data_present (95.2%)  ← Dropped!
```

**Alert triggered:** `user_data_present` pass rate dropped below threshold.

**Investigation:** Check logs with `passed: false` for this predicate:

```typescript
const failures = logs.filter(
  l => l.predicate === "user_data_present" && !l.passed
);

console.log(failures[0]);
// {
//   predicate: "user_data_present",
//   passed: false,
//   userId: "12345",
//   responseStructure: { error: "User deleted" }
// }
```

**Discovery:** Users are being deleted but we're not handling the "deleted user" case!

**Fix:** Add new predicate:

```typescript
logSemantic({
  predicate: "user_exists_not_deleted",
  passed: data.user && !data.user.deleted,
  isDeleted: data.user?.deleted || false,
  correlationId,
  scope: "domain.users"
});
```

## Fail-Upward Mindset

Traditional development:
```
"I hope this works"
"It works on my machine"
"Can't reproduce the bug"
```

Predicate development:
```
"What predicates should hold?"
"Let me test each one's failure mode"
"This predicate failed - now I know what to fix"
```

**You're not trying to avoid failures. You're trying to make failures informative.**

## Testing Checklist

For each predicate:

- [ ] Test when it passes (happy path)
- [ ] Test when it fails (unhappy path)
- [ ] Test edge cases (empty strings, null, undefined, 0, etc.)
- [ ] Test race conditions (if async)
- [ ] Test when parent predicate failed (should never fire)
- [ ] Test when parent predicate passed (should fire)

**Example:**

```typescript
describe("Predicate: user_data_present", () => {
  it("passes when user object exists");
  it("fails when user is null");
  it("fails when user is undefined");
  it("fails when response is empty object");
  it("fails when response is malformed");
  it("does not fire when http_request_successful failed");
  it("does not fire when response_parseable failed");
});
```

## The Ultimate Loop

```
1. Define predicates (assumptions)
2. Implement with logging
3. Test happy path
   ✓ Works? Good.
4. Test each predicate's failure mode
   ✗ Found a gap? Add predicate.
5. Test again
6. Deploy
7. Monitor predicate pass rates
8. Alert on anomalies
9. Investigate failures
   → Discover new failure modes
   → Add predicates
   → Fix
10. Repeat forever
```

**This is continuous improvement through structured failure.**

## Summary

Predicate logging isn't just observability - it's a **development loop**:

1. **Declare what should be true** (predicates)
2. **Implement** (with logging)
3. **Test by deliberately breaking things** (fail on purpose)
4. **Read the proof tree** (see exactly what failed)
5. **Fix the specific predicate** (not the whole system)
6. **Repeat** (until all predicates pass)

**In production:**
7. **Monitor predicate pass rates** (which are dropping?)
8. **Investigate failures** (what's the new failure mode?)
9. **Add predicates** (fill the gaps you discovered)
10. **Fix** (now you know what to fix)

You're not trying to write perfect code on the first try. You're building a system that **tells you exactly what's wrong** when your assumptions are incorrect.

**That's fail-upward development:** Each failure makes the system more robust because you learned something specific and measurable.
