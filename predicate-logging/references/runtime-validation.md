# Runtime Validation Library

This reference describes a runtime validation library that enforces predicate logging rules automatically.

## Core Validation Rules

The library enforces:
1. **Hierarchy enforcement** - Child logs only fire when parents exist
2. **Correlation tracking** - All predicates scoped to runs
3. **Schema validation** - Required fields present
4. **Orphan detection** - Catch children without parents
5. **Tree visualization** - Display proof tree per correlationId

## TypeScript Implementation

```typescript
interface SemanticLog {
  event: string;
  predicate: string;
  passed: boolean;
  correlationId: string;
  scope: string;
  channel?: string;
  timestamp?: number;
  [key: string]: any;
}

interface PredicateHierarchy {
  predicate: string;
  parents: string[];
  children: string[];
}

class PredicateLogger {
  private logs: Map<string, SemanticLog[]> = new Map();
  private hierarchy: Map<string, PredicateHierarchy> = new Map();
  
  /**
   * Register predicate hierarchy relationships
   */
  registerHierarchy(
    predicate: string,
    parents: string[] = []
  ): void {
    this.hierarchy.set(predicate, {
      predicate,
      parents,
      children: []
    });
    
    // Register this as child of parents
    parents.forEach(parent => {
      const parentNode = this.hierarchy.get(parent);
      if (parentNode) {
        parentNode.children.push(predicate);
      }
    });
  }
  
  /**
   * Log a predicate claim with validation
   */
  log(logEntry: SemanticLog): void {
    // Validate required fields
    this.validateSchema(logEntry);
    
    // Check hierarchy constraints
    if (logEntry.passed) {
      this.validateHierarchy(logEntry);
    }
    
    // Store log
    const correlationId = logEntry.correlationId;
    if (!this.logs.has(correlationId)) {
      this.logs.set(correlationId, []);
    }
    
    this.logs.get(correlationId)!.push({
      ...logEntry,
      timestamp: Date.now()
    });
    
    // Emit to traditional logger
    console.log(JSON.stringify(logEntry));
  }
  
  /**
   * Validate required schema fields
   */
  private validateSchema(log: SemanticLog): void {
    const required = ['event', 'predicate', 'passed', 'correlationId', 'scope'];
    
    for (const field of required) {
      if (!(field in log)) {
        throw new Error(
          `SemanticLog missing required field: ${field}`
        );
      }
    }
    
    if (typeof log.passed !== 'boolean') {
      throw new Error(
        `predicate "passed" must be boolean, got ${typeof log.passed}`
      );
    }
  }
  
  /**
   * Validate hierarchy constraints
   */
  private validateHierarchy(log: SemanticLog): void {
    if (!log.passed) return; // Only check passed predicates
    
    const node = this.hierarchy.get(log.predicate);
    if (!node || node.parents.length === 0) return;
    
    const runLogs = this.logs.get(log.correlationId) || [];
    
    // Check all parents passed in this run
    for (const parentPredicate of node.parents) {
      const parentLog = runLogs.find(
        l => l.predicate === parentPredicate && l.passed === true
      );
      
      if (!parentLog) {
        throw new Error(
          `Hierarchy violation: predicate "${log.predicate}" requires parent "${parentPredicate}" to have passed, but no such log found for correlationId ${log.correlationId}`
        );
      }
    }
  }
  
  /**
   * Get all logs for a correlation ID
   */
  getLogsForRun(correlationId: string): SemanticLog[] {
    return this.logs.get(correlationId) || [];
  }
  
  /**
   * Analyze predicate tree for a run
   */
  analyzeRun(correlationId: string): RunAnalysis {
    const runLogs = this.getLogsForRun(correlationId);
    
    const passedPredicates = runLogs
      .filter(l => l.passed)
      .map(l => l.predicate);
      
    const failedPredicates = runLogs
      .filter(l => !l.passed)
      .map(l => l.predicate);
    
    // Detect orphans (children without parents)
    const orphans: string[] = [];
    
    for (const log of runLogs) {
      if (!log.passed) continue;
      
      const node = this.hierarchy.get(log.predicate);
      if (!node || node.parents.length === 0) continue;
      
      const missingParents = node.parents.filter(
        p => !passedPredicates.includes(p)
      );
      
      if (missingParents.length > 0) {
        orphans.push(
          `${log.predicate} (missing: ${missingParents.join(', ')})`
        );
      }
    }
    
    return {
      correlationId,
      totalLogs: runLogs.length,
      passedPredicates,
      failedPredicates,
      orphans,
      healthy: orphans.length === 0
    };
  }
  
  /**
   * Build proof tree visualization
   */
  buildProofTree(correlationId: string): string {
    const runLogs = this.getLogsForRun(correlationId);
    const passedPredicates = new Set(
      runLogs.filter(l => l.passed).map(l => l.predicate)
    );
    
    const lines: string[] = [];
    const visited = new Set<string>();
    
    const buildNode = (predicate: string, depth: number = 0): void => {
      if (visited.has(predicate)) return;
      visited.add(predicate);
      
      const indent = "  ".repeat(depth);
      const passed = passedPredicates.has(predicate);
      const symbol = passed ? "✓" : "✗";
      
      lines.push(`${indent}${symbol} ${predicate}`);
      
      if (passed) {
        const node = this.hierarchy.get(predicate);
        if (node) {
          node.children.forEach(child => {
            if (passedPredicates.has(child)) {
              buildNode(child, depth + 1);
            }
          });
        }
      }
    };
    
    // Find root predicates (no parents)
    Array.from(this.hierarchy.values())
      .filter(n => n.parents.length === 0)
      .forEach(n => buildNode(n.predicate));
    
    return lines.join("\n");
  }
}

interface RunAnalysis {
  correlationId: string;
  totalLogs: number;
  passedPredicates: string[];
  failedPredicates: string[];
  orphans: string[];
  healthy: boolean;
}

// Global instance
export const predicateLogger = new PredicateLogger();

// Convenience function
export function logSemantic(log: SemanticLog): void {
  predicateLogger.log(log);
}

// Register hierarchies at startup
export function registerPredicateHierarchy(
  predicate: string,
  parents: string[] = []
): void {
  predicateLogger.registerHierarchy(predicate, parents);
}
```

## Usage Example

```typescript
// 1. Register hierarchy at application startup
registerPredicateHierarchy("http_request_successful", []);
registerPredicateHierarchy("response_parseable", ["http_request_successful"]);
registerPredicateHierarchy("user_data_present", ["response_parseable"]);
registerPredicateHierarchy("data_is_fresh", ["user_data_present"]);

// 2. Use in code - validation happens automatically
async function fetchUser(userId: string, correlationId: string) {
  const response = await fetch(`/api/users/${userId}`);
  
  logSemantic({
    event: "user_fetch_http_complete",
    predicate: "http_request_successful",
    passed: response.ok,
    correlationId,
    scope: "service.api"
  });
  
  if (!response.ok) return null;
  
  const data = await response.json();
  
  logSemantic({
    event: "user_fetch_parsed",
    predicate: "response_parseable",
    passed: true,
    correlationId,
    scope: "service.api"
  });
  
  logSemantic({
    event: "user_data_checked",
    predicate: "user_data_present",
    passed: !!data.user,
    correlationId,
    scope: "domain.users"
  });
  
  // This would throw if parent predicates hadn't passed:
  // logSemantic({
  //   predicate: "data_is_fresh",
  //   passed: true,
  //   correlationId
  // }); // Error: http_request_successful not found!
  
  return data;
}

// 3. Analyze run
const analysis = predicateLogger.analyzeRun("correlation-123");
console.log(analysis);
// {
//   correlationId: "correlation-123",
//   passedPredicates: ["http_request_successful", "response_parseable", "user_data_present"],
//   failedPredicates: [],
//   orphans: [],
//   healthy: true
// }

// 4. Visualize proof tree
console.log(predicateLogger.buildProofTree("correlation-123"));
// ✓ http_request_successful
//   ✓ response_parseable
//     ✓ user_data_present
//       ✓ data_is_fresh
```

## Python Implementation

```python
from typing import Dict, List, Set, Optional, Any
from dataclasses import dataclass
import json
from datetime import datetime

@dataclass
class SemanticLog:
    event: str
    predicate: str
    passed: bool
    correlationId: str
    scope: str
    channel: Optional[str] = None
    timestamp: Optional[float] = None

@dataclass 
class PredicateHierarchy:
    predicate: str
    parents: List[str]
    children: List[str]

@dataclass
class RunAnalysis:
    correlationId: str
    total_logs: int
    passed_predicates: List[str]
    failed_predicates: List[str]
    orphans: List[str]
    healthy: bool

class PredicateLogger:
    def __init__(self):
        self.logs: Dict[str, List[Dict[str, Any]]] = {}
        self.hierarchy: Dict[str, PredicateHierarchy] = {}
    
    def register_hierarchy(
        self, 
        predicate: str, 
        parents: List[str] = None
    ) -> None:
        parents = parents or []
        
        self.hierarchy[predicate] = PredicateHierarchy(
            predicate=predicate,
            parents=parents,
            children=[]
        )
        
        for parent in parents:
            if parent in self.hierarchy:
                self.hierarchy[parent].children.append(predicate)
    
    def log(self, **kwargs) -> None:
        # Validate schema
        required = ['event', 'predicate', 'passed', 'correlationId', 'scope']
        for field in required:
            if field not in kwargs:
                raise ValueError(f"Missing required field: {field}")
        
        if not isinstance(kwargs['passed'], bool):
            raise TypeError("'passed' must be boolean")
        
        # Validate hierarchy
        if kwargs['passed']:
            self._validate_hierarchy(kwargs)
        
        # Store log
        correlation_id = kwargs['correlationId']
        if correlation_id not in self.logs:
            self.logs[correlation_id] = []
        
        log_entry = {
            **kwargs,
            'timestamp': datetime.now().timestamp()
        }
        
        self.logs[correlation_id].append(log_entry)
        
        # Emit to traditional logger
        print(json.dumps(log_entry))
    
    def _validate_hierarchy(self, log: Dict[str, Any]) -> None:
        predicate = log['predicate']
        
        if predicate not in self.hierarchy:
            return
        
        node = self.hierarchy[predicate]
        if not node.parents:
            return
        
        run_logs = self.logs.get(log['correlationId'], [])
        
        for parent_predicate in node.parents:
            parent_log = next(
                (l for l in run_logs 
                 if l['predicate'] == parent_predicate and l['passed']),
                None
            )
            
            if not parent_log:
                raise RuntimeError(
                    f"Hierarchy violation: predicate '{predicate}' "
                    f"requires parent '{parent_predicate}' to have passed"
                )
    
    def analyze_run(self, correlation_id: str) -> RunAnalysis:
        run_logs = self.logs.get(correlation_id, [])
        
        passed = [l['predicate'] for l in run_logs if l['passed']]
        failed = [l['predicate'] for l in run_logs if not l['passed']]
        
        orphans = []
        for log in run_logs:
            if not log['passed']:
                continue
            
            predicate = log['predicate']
            if predicate not in self.hierarchy:
                continue
            
            node = self.hierarchy[predicate]
            missing = [p for p in node.parents if p not in passed]
            
            if missing:
                orphans.append(
                    f"{predicate} (missing: {', '.join(missing)})"
                )
        
        return RunAnalysis(
            correlationId=correlation_id,
            total_logs=len(run_logs),
            passed_predicates=passed,
            failed_predicates=failed,
            orphans=orphans,
            healthy=len(orphans) == 0
        )

# Global instance
predicate_logger = PredicateLogger()

def log_semantic(**kwargs):
    predicate_logger.log(**kwargs)

def register_predicate_hierarchy(predicate: str, parents: List[str] = None):
    predicate_logger.register_hierarchy(predicate, parents)
```

## Testing with Validation

```typescript
describe("PredicateLogger", () => {
  let logger: PredicateLogger;
  
  beforeEach(() => {
    logger = new PredicateLogger();
    logger.registerHierarchy("parent", []);
    logger.registerHierarchy("child", ["parent"]);
  });
  
  test("allows child log when parent passed", () => {
    expect(() => {
      logger.log({
        event: "parent_checked",
        predicate: "parent",
        passed: true,
        correlationId: "test",
        scope: "test"
      });
      
      logger.log({
        event: "child_checked",
        predicate: "child",
        passed: true,
        correlationId: "test",
        scope: "test"
      });
    }).not.toThrow();
  });
  
  test("blocks child log when parent missing", () => {
    expect(() => {
      logger.log({
        event: "child_checked",
        predicate: "child",
        passed: true,
        correlationId: "test",
        scope: "test"
      });
    }).toThrow(/Hierarchy violation/);
  });
  
  test("detects orphans in analysis", () => {
    // Manually insert orphaned log (bypassing validation for test)
    logger['logs'].set("test", [{
      event: "orphan",
      predicate: "child",
      passed: true,
      correlationId: "test",
      scope: "test"
    }]);
    
    const analysis = logger.analyzeRun("test");
    expect(analysis.orphans.length).toBeGreaterThan(0);
    expect(analysis.healthy).toBe(false);
  });
});
```

## Integration with Monitoring

```typescript
// Export to monitoring system
function exportToDatadog(correlationId: string) {
  const analysis = predicateLogger.analyzeRun(correlationId);
  
  // Send metrics
  datadogClient.gauge('predicate.logs.total', analysis.totalLogs, {
    correlation_id: correlationId
  });
  
  datadogClient.gauge('predicate.passed.count', 
    analysis.passedPredicates.length
  );
  
  datadogClient.gauge('predicate.failed.count',
    analysis.failedPredicates.length
  );
  
  // Alert on orphans
  if (analysis.orphans.length > 0) {
    datadogClient.event({
      title: 'Predicate Hierarchy Violation',
      text: `Orphaned predicates detected: ${analysis.orphans.join(', ')}`,
      alert_type: 'error',
      tags: [`correlation_id:${correlationId}`]
    });
  }
}
```

This runtime validation library ensures the predicate logging system is self-enforcing and catches violations immediately during development and testing.
