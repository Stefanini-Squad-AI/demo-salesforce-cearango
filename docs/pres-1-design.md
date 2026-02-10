# PRES-1: Next Best Action design

## Background
Commercial and service teams must see a ranked Top 3 next-best-actions directly on the Salesforce record so they can act without hunting through notes, playbooks, or external dashboards. The experience has to be explainable, actionable, and auditable, producing telemetry that measures adoption, conversion, and handling time.

## Acceptance criteria coverage
- **AC1:** `NextBestActionService` exposes a scored Top 3 per context (Account, Opportunity, Case, etc.) with score, priority, and reason metadata that is consumable by a Lightning Web Component (LWC) or Flow (screen, notification, utility).
- **AC2:** Each recommendation includes an `Execute` action that delegates to `NBAActionExecutor`, which handles Tasks, Opportunity updates, Platform Events, or Flow invocations.
- **AC3:** Recommendation lifecycle events (shown → accepted/rejected → executed → outcome) create/update an `NBA_Recommendation__c` audit record with Customer, Context, ActionType, Score, Reason, Status, Outcome, and timestamps.
- **AC4:** Rule metadata (`NBA_Rule__mdt`) is configurable without deployments; the service respects flags such as `Active`, `ActionType`, `BaseScore`, `Priority`, and `Condition` and ranks recommendations in real time.

## Component sketch

### NextBestActionService (Apex / Platform Cache aware)
- Inputs: record context (record ID, type, related case/opportunity), user role, and optional signal overrides (e.g., urgency flag).
- Pipeline:
 1. `NBA_Rule__mdt` records are filtered by context, `Active=true`, sharing rules, and any exclusion criteria.
 2. Each rule calculates a final score: base score + modifiers from record data or telemetry (e.g., time since last activity, opportunity amount).
 3. Rules emit explainable reasons (rule label + trigger detail) for LWC consumption.
 4. Results sorted by score/priority, trimmed to top three, and cached per context when safe (e.g., per page load, invalidated when key fields change).
- Outputs: `List<NBA_Recommendation__c>` payload (serializable for Apex/REST) with Score, Priority, Reason, ActionType, TargetObjectId, SuggestedAction, RuleId.
- Constraints: bulk-safe, respects FLS/CRUD, caches (Platform Cache or custom cache keyed by record + user) to limit SOQL, logs anonymized context for debugging.

### NBAActionExecutor (Apex service and Flow actions)
- Accepts recommendation ID + payload; ensures idempotency (check for existing audit record) before acting.
- Actions supported:
 1. Create Task/Follow-up with templated description.
 2. Update Opportunity or Case (Stage, Status, Priority) via limited field set and sharing.
 3. Publish Platform Event for downstream integrations (Agentforce, analytics).
 4. Invoke Flow (screen or autolaunched) for complex multi-step actions.
- Returns execution outcome (success, error, note) stored on `NBA_Recommendation__c`.

### NBA_Recommendation__c (custom object)
- Fields: `Customer__c` (lookup Account/Contact), `Context__c` (polymorphic reference: Case, Opportunity, etc.), `ActionType__c`, `Rule__c`, `Score__c`, `Priority__c`, `Reason__c`, `Status__c` (Shown, Accepted, Rejected, Executed), `Outcome__c`, `ExecutionDetails__c`, `ExecutionTime__c`, `CreatedBy`, etc.
- Lifecycle triggers (Apex/Flow) update status/outcome when shown, accepted, or executed, supporting metrics such as adoption rate (executed vs shown) and conversion (success per action type).
- Consider global secondary index (record type + status + created date) for reporting.

### NBA_Rule__mdt (metadata type for business rules)
- Fields: `Active__c`, `ActionType__c`, `BaseScore__c`, `Priority__c`, `Condition__c` (formula or Flow reference), `TargetObject__c`, `Description__c`, `ExecutionStrategy__c` (task, update, flow, event).
- Business users toggle `Active` or adjust score/priority via Custom Metadata records; the service queries metadata with `with sharing`, `Active = true`, `Condition` evaluated via Formula or Flow that returns boolean.
- Use `Refresh Metadata` caching pattern (e.g., `@AuraEnabled` helper caches results for minutes but invalidates when metadata changed).

### LWC / Flow surface
- Expose Top 3 recommendations with `score`, `priority`, `reason`, and a CTA (button) that invokes `NBAActionExecutor`.
- Surface includes accept/reject toggles that update `NBA_Recommendation__c` and capture telemetry immediately.
- Optionally embed Flow (via `lightning-flow` or Flow screen) for guided execution if rule requires inputs.

## Telemetry & Operations
- Each recommendation update logs `ContextId`, `RuleId`, `Status`, and `Outcome` to `NBA_Recommendation__c`; consider using Platform Events for asynchronous processing (e.g., to update data cloud or dashboards).
- Audit the execution path: `shown -> accepted -> executed -> outcome`; track timestamps and user IDs for time-to-handle metrics.
- Logging should capture only IDs/metrics; avoid PII. Field-level security enforced when reading/writing.
- Service should use `Limits.getQueries()`/`Limits.getDML()` instrumentation and guard bulk contexts (e.g., page with multiple contexts). Provide `@future` or Queueable fallback for long-running scoring.

## Implementation roadmap
1. Create `NBA_Recommendation__c` and `NBA_Rule__mdt` definitions (fields, layouts) outside of codebase; document required fields for metadata deployment.
2. Build Apex services:
   - `NextBestActionService` to evaluate rules, compute scores, rank, and serialize reason/action metadata.
   - `NBAActionExecutor` for executing tasks/updates/events/Flows with shared utility methods.
   - `NBARecommendationHandler` triggers (before insert/update) for lifecycle tracking.
3. Develop LWC/Flow screens:
   - LWC to display Top 3, show reasons, and call `executeRecommendation`.
   - Flow action (Invocable Apex) that wraps `NBAActionExecutor` for flow builders.
4. Define telemetry views:
   - Reports/dashboards referencing `NBA_Recommendation__c` (status, outcome, conversion).
   - Platform Event for `RecommendationOutcome__e` to stream to Data Cloud.
5. Security/performance:
   - Use `with sharing`, enforce `isAccessible` before DML.
   - Cache rule metadata in Platform Cache with manual invalidation (metadata change handler).
   - Unit tests for scoring logic, execution paths, and metadata toggling.

## Next steps
- Align metadata field names/layouts with the Salesforce org (confirm with admins).
- Prototype LWC/Flow surfaces using mock data to validate UX.
- Deploy metadata and Apex via CI once functionality passes automated tests and code review.
