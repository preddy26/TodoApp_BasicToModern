# [TICKET] Horizontal Scaling of Quartz Background Jobs - AdoJobStore with MySQL Clustering


**Status:** Ready for Implementation
**Priority:** HIGH
**Epic:** Production Readiness / Scaling Infrastructure
**T-Shirt Size:** 5 days (L)
**Date Created:** May 16, 2026


---


## Summary


Migrate Quartz.NET scheduler from **in-memory store** (`RAMJobStore`) to **distributed database-backed store** (`AdoJobStore` with MySQL) with clustering enabled. This enables horizontal scaling of the API layer while guaranteeing exactly-once job execution across multiple pods/instances.


**Current state:** 3 pods = 3 duplicate emails 
**After implementation:** 3 pods = 1 email ✓


---


## Background & Context

### Current Architecture Problem


The TodoApp backend uses **Quartz.NET 3.15.1** with in-memory job storage configured in:
- **File:** `Todo.API/Extensions/QuartzServiceExtensions.cs` (lines 12-13)


```csharp
q.UseSimpleTypeLoader();
q.UseInMemoryStore();  // ❌ Non-persistent, no clustering
```


**Issues with in-memory store:**
1. **No Distributed Coordination:** Each pod runs independent Quartz scheduler
2. **Duplicate Executions:** When 3 pods scale up, 3x daily reports sent (3 emails instead of 1)
3. **No Persistence:** Job state lost on pod restart; no execution history
4. **No Visibility:** Each pod has independent, incomplete execution metrics
5. **Manual Trigger Race Conditions:** Pause/resume only affects single pod
6. **Unrecoverable Failures:** Failed jobs have no persistent retry state


### Current Jobs Affected


Three email jobs run on hardcoded schedules:


| Job Class | Schedule | Trigger | Impact |
|-----------|----------|---------|--------|
| `DailyTodoItemReportJob` | `0 0 18 * * ?` (6:00 PM daily) | `DailyReportTrigger` | Duplicate daily reports → Email spam |
| `WeeklyTaskSummaryJob` | `0 0 9 ? * MON` (9:00 AM Monday) | `WeeklySummaryTrigger` | Duplicate weekly summaries |
| `TaskReminderJob` | `0 0 8 * * ?` (8:00 AM daily) | `ReminderTrigger` | Duplicate task reminders |


All jobs in namespace: `EmailJobs`


### Deployment Context


- **Current deployment:** 1 pod (docker-compose.yml) ✓ Works
- **Near-future deployment:** 3+ pods (Kubernetes/scaling) ❌ Breaks
- **Database:** MySQL 8.0 (already present, already used)
- **Cache:** Redis 7 (already present, optional)
- **Framework:** .NET 8, ASP.NET Core with dependency injection


---


## Impact Areas


### User-Facing Impact


| Scenario | Current | After Fix |
|----------|---------|-----------|
| **Duplicate Emails** | 3 pods = 3 daily reports to user inbox | 3 pods = 1 daily report ✓ |
| **Email Reliability** | Silent failures, no retries | Automatic retries via Quartz ✓ |
| **Job Visibility** | Per-pod only | Centralized execution history ✓ |


### Operational Impact


| Aspect | Current | After Fix |
|--------|---------|-----------|
| **Manual Job Trigger** | Only affects routed pod | Affects all pods consistently ✓ |
| **Pause/Resume Jobs** | Single pod scope | Cluster-wide scope ✓ |
| **Pod Restart** | Job state lost | State persists in database ✓ |
| **Execution Auditing** | Logs per pod | Centralized query `qrtz_fired_triggers` ✓ |
| **Debugging Failed Jobs** | Ephemeral (lost) | Persistent (queryable) ✓ |


### Code Changes Required


**Scope:**
- Modify 1 core file: `Todo.API/Extensions/QuartzServiceExtensions.cs`
- Update configuration: `appsettings.json`, `appsettings.Production.json`
- Create 1 EF Core migration: Add Quartz schema
- Add 1 NuGet package: `Quartz.Providers.MySql`


**Scope NOT included in this ticket:**
- Email idempotency improvements (separate ticket: TICKET-IDEMPOTENCY)
- SMTP timeout configuration (separate ticket: TICKET-EMAIL-RELIABILITY)
- Job schedule externalization to configuration (separate ticket: TICKET-DYNAMIC-JOBS)


## Code Quality Improvements (In-Scope)


The following issues were identified during review and should be fixed
as part of this implementation:


### Critical: WeeklyTaskSummaryJob Never Sends Email
**File:** `Jobs/WeeklyTaskSummaryJob.cs` lines 187–195 
**Issue:** Email body is built but `SendEmailAsync` is never called — weekly
emails are silently never sent. This is a pre-existing bug independent of
clustering. 
**Fix:** Add `await _emailService.SendEmailAsync(message);` after email
construction.


### Medium: Inconsistent DateTime Usage
**Files:** Multiple job classes 
**Issue:** Mix of `DateTime.Now` and `DateTime.UtcNow` — in a clustered
environment across pods this causes inconsistent timestamps in logs and
qrtz_fired_triggers. 
**Fix:** Standardise on `DateTime.UtcNow` across all job classes.


### Medium: Log Parameter Typo
**Issue:** `{JoKey}` should be `{JobKey}` — causes structured logging to emit
malformed entries, breaking any log-based alerting. 
**Fix:** Correct the parameter name.


---


## Proposed Approach


### Option Selection: AdoJobStore with MySQL Clustering


**Chosen:** ✅ **AdoJobStore with MySQL** (Quartz JDBC equivalent for .NET)


**Why this approach:**


| Criteria | In-Memory | **AdoJobStore (MySQL)** | MongoDB | Redis Custom | Cloud Service |
|----------|-----------|------------------------|---------|--------------|---------------|
| **Tech Stack Fit** | ❌ No | ✅✅ Perfect | ❌ No | ✅ Good | ⚠️ Depends |
| **Effort** | ✅ 0 hrs | ✅ 3-4 days | ⚠️ 3-4 days | ❌ 1-2 weeks | ⚠️ 3-4 days |
| **Maintenance** | ✅ Zero | ✅ Low | ⚠️ Medium | ❌ High | ✅ Zero |
| **Production Ready** | ❌ No | ✅✅ Yes | ✅ Yes | ⚠️ Risk | ✅ Yes |
| **Infrastructure** | N/A | ✅ Existing MySQL | ❌ New DB | ✅ Existing Redis | ⚠️ External |
| **Battle Tested** | N/A | ✅✅ Thousands of users | ✅ Many | ⚠️ Custom code | ✅ Varies |


**Selected because:**
1. **Zero new infrastructure** - Uses existing MySQL database
2. **3-day implementation** - Faster than custom Redis or new MongoDB
3. **Production-grade** - Battle-tested by thousands of Quartz.NET users worldwide
4. **Low maintenance** - Standard Quartz tables, no custom code to maintain
5. **Clear migration path** - Drop-in replacement for in-memory store
6. **Perfect team fit** - Team already knows MySQL, no new tech needed


---


## Proposed Solution Architecture


```
Before (In-Memory):
┌─────────────────────────────────────────────────┐
│          Load Balancer / Kubernetes             │
│                     │                           │
│      ┌──────────────┼──────────────┐            │
│      │              │              │            │
│      ▼              ▼              ▼            │
│    Pod 1         Pod 2          Pod 3          │
│  (Quartz 1)    (Quartz 2)    (Quartz 3)        │
│   in-memory      in-memory      in-memory       │
│      RAM1          RAM2           RAM3          │
│                                                  │
│   Result: 3 independent schedulers              │
│   Problem: 3 jobs execute same trigger          │
└─────────────────────────────────────────────────┘


After (AdoJobStore with Clustering):
┌─────────────────────────────────────────────────┐
│          Load Balancer / Kubernetes             │
│                     │                           │
│      ┌──────────────┼──────────────┐            │
│      │              │              │            │
│      ▼              ▼              ▼            │
│    Pod 1         Pod 2          Pod 3          │
│  (Quartz 1)    (Quartz 2)    (Quartz 3)        │
│   Clustered      Clustered      Clustered       │
│      │              │              │            │
│      └──────────────┼──────────────┘            │
│                     │                           │
│              Distributed Locks                  │
│              (row-level, auto-expire)           │
│                     │                           │
│         ┌───────────▼───────────┐              │
│         │   Shared MySQL DB    │              │
│         ├──────────────────────┤              │
│         │ qrtz_job_details     │              │
│         │ qrtz_triggers        │              │
│         │ qrtz_locks (⭐)      │              │
│         │ qrtz_scheduler_state │              │
│         │ qrtz_fired_triggers  │              │
│         └──────────────────────┘              │
│                                                 │
│   Result: 1 distributed scheduler               │
│   Guarantee: Only 1 job executes per trigger   │
└─────────────────────────────────────────────────┘
```


### How Clustering Prevents Duplicates


```
Trigger Fire Time Arrives (6:00 PM):


Pod 1 Quartz:                 Pod 2 Quartz:                 Pod 3 Quartz:
├─ Check trigger fired?       ├─ Check trigger fired?       ├─ Check trigger fired?
├─ YES (scheduled)            ├─ YES (scheduled)            ├─ YES (scheduled)
├─ Try to acquire lock        ├─ Try to acquire lock        ├─ Try to acquire lock
│  INSERT qrtz_locks ✓        │  INSERT qrtz_locks ✗        │  INSERT qrtz_locks ✗
│  Lock acquired!             │  Wait... (polling)           │  Wait... (polling)
│                             │                              │
├─ Execute job                │                              │
├─ Send email                 │                              │
├─ Update qrtz_fired_triggers │                              │
├─ Release lock               │                              │
│  DELETE qrtz_locks          │                              │
│                             │                              │
│                             ├─ Lock released, try again    │
│                             ├─ INSERT qrtz_locks ✓         │
│                             ├─ But check FIRED flag        │
│                             │  (already in qrtz_fired)     │
│                             ├─ Skip execution (fired)      │
│                             ├─ Release lock                │
│                             │                              │
│                             │                              │
│                             │                              │
│                             │                              │
│                             │                              │
│                             │  (Pod 3 same as Pod 2)       │
│                             │  Skips execution             │


Result: Exactly 1 email sent, not 3 ✓
```


---


## Implementation Plan


### Phase 1: Dependencies & Schema (1 day)


**Task 1.1: Add NuGet Package**
```bash
cd TodoApp.Server/src/Todo.API
dotnet add package Quartz.Providers.MySql --version 3.15.1
```


**Task 1.2: Create EF Core Migration**


Create migration file: `Todo.Models/Migrations/[TIMESTAMP]_AddQuartzSchema.cs`


```csharp
using Microsoft.EntityFrameworkCore.Migrations;


namespace Todo.Models.Migrations
{
   public partial class AddQuartzSchema : Migration
   {
       protected override void Up(MigrationBuilder migrationBuilder)
       {
           migrationBuilder.Sql(@"
-- Copy entire Quartz schema from QUARTZ_SCHEMA_DDL.sql (see appendix)
-- 11 tables total, with clustering support tables:
-- - qrtz_job_details (job definitions)
-- - qrtz_triggers (cron/simple triggers)
-- - qrtz_locks (distributed locking - CRITICAL for clustering)
-- - qrtz_scheduler_state (node registration - CRITICAL for clustering)
-- - qrtz_fired_triggers (execution history)
-- - [plus 6 others for calendar, blob storage, etc.]
           ");
       }


       protected override void Down(MigrationBuilder migrationBuilder)
       {
           migrationBuilder.Sql(@"
DROP TABLE IF EXISTS qrtz_simprop_triggers;
DROP TABLE IF EXISTS qrtz_simple_triggers;
DROP TABLE IF EXISTS qrtz_cron_triggers;
DROP TABLE IF EXISTS qrtz_blob_triggers;
DROP TABLE IF EXISTS qrtz_triggers;
DROP TABLE IF EXISTS qrtz_job_details;
DROP TABLE IF EXISTS qrtz_calendars;
DROP TABLE IF EXISTS qrtz_paused_trigger_grps;
DROP TABLE IF EXISTS qrtz_locks;
DROP TABLE IF EXISTS qrtz_scheduler_state;
DROP TABLE IF EXISTS qrtz_fired_triggers;
           ");
       }
   }
}
```


**Task 1.3: Apply Migration to Local Database**
```bash
cd Todo.Models
dotnet ef database update --startup-project ../Todo.API/Todo.API.csproj
# Verify qrtz_* tables created:
# mysql> SHOW TABLES LIKE 'qrtz_%';
```


---


### Phase 2: Configuration Updates (0.5 days)


**Task 2.1: Update `appsettings.json`**


**File:** `Todo.API/appsettings.json`


Replace this:
```json
"Quartz": {
 "quartz.scheduler.instanceName": "TodoScheduler",
 "quartz.scheduler.instanceId": "AUTO",
 "quartz.jobStore.type": "Quartz.Simpl.RAMJobStore, Quartz",
 "quartz.threadPool.threadCount": 3,
 "quartz.serializer.type": "json"
}
```


With this:
```json
"Quartz": {
 "quartz.scheduler.instanceName": "TodoAppScheduler",
 "quartz.scheduler.instanceId": "AUTO",
  "quartz.jobStore.type": "Quartz.Impl.AdoJobStore.JobStoreTX, Quartz.Providers.MySql",
 "quartz.jobStore.driverDelegateType": "Quartz.Impl.AdoJobStore.MySql.MySqlConnectorDelegate, Quartz.Providers.MySql",
  "quartz.jobStore.dataSource": "default",
 "quartz.jobStore.tablePrefix": "qrtz_",
 "quartz.jobStore.useProperties": "true",
 "quartz.jobStore.misfireThreshold": "60000",
  "quartz.jobStore.clustered": "true",
 "quartz.jobStore.clusterCheckinInterval": "15000",
 "quartz.jobStore.clusterCheckinThrottleWindowMs": "40000",
 "quartz.dataSource.default.connectionString": "#{QUARTZ_DB_CONNECTION_STRING}#",
 "quartz.dataSource.default.provider": "MySqlConnector",
  "quartz.threadPool.type": "Quartz.Simpl.DefaultThreadPool, Quartz",
 "quartz.threadPool.threadCount": "3",
 "quartz.serializer.type": "json"
}
```


**Task 2.2: Update `appsettings.Production.json` (if exists)**

Add same Quartz configuration section (or ensure environment-specific values are set via deployment variables)


---


### Phase 3: Core Code Changes (1 day)


**Task 3.1: Instance ID Generation Strategy** 🔴 CRITICAL


Before implementing clustering, establish pod identity strategy:


**Problem:** Default "AUTO" instance ID generation in containers uses hostname, which could collide across environments.


**Solution:** Use explicit environment variable (pod name) for instance ID


**Implementation:**
```csharp
// In Program.cs, retrieve pod identity BEFORE Quartz configuration
var quartzInstanceId = builder.Configuration["Quartz:InstanceId"]
    ?? Environment.GetEnvironmentVariable("POD_NAME")
    ?? Environment.GetEnvironmentVariable("HOSTNAME")
    ?? throw new InvalidOperationException(
        "Quartz instance ID not configured. Set 'Quartz:InstanceId' in appsettings or POD_NAME environment variable");


builder.Configuration["Quartz:InstanceId"] = quartzInstanceId;


if (string.IsNullOrWhiteSpace(quartzInstanceId))
    throw new InvalidOperationException("Instance ID cannot be empty or whitespace");


logger.LogInformation("✓ Quartz instance ID configured: {InstanceId}", quartzInstanceId);
```


**Environment Variable Examples:**
```bash
# Kubernetes deployment (via downward API)
POD_NAME=todo-api-prod-pod1


# Docker Compose
POD_NAME=todo-api-service-1


# Local development
POD_NAME=localhost
```


**Verification Query** (after deployment):
```sql
SELECT instance_name, last_checkin_time FROM qrtz_scheduler_state;
-- Expected output (3 pods):
-- +---------------------+----------------------+
-- | instance_name       | last_checkin_time    |
-- +---------------------+----------------------+
-- | todo-api-prod-pod1  | 2026-05-16 10:30:45  |
-- | todo-api-prod-pod2  | 2026-05-16 10:30:43  |
-- | todo-api-prod-pod3  | 2026-05-16 10:30:44  |
-- +---------------------+----------------------+
-- Each row should have unique instance_name (no duplicates)
-- Each row should have recent last_checkin_time (within 30 seconds)
```


**Troubleshooting Duplicate Instance IDs:**
If two pods show the same instance_name in qrtz_scheduler_state:
```sql
-- Check for duplicates
SELECT instance_name, COUNT(*) FROM qrtz_scheduler_state
GROUP BY instance_name HAVING COUNT(*) > 1;


-- If duplicates found, environment variable not properly set
-- Action: Verify POD_NAME environment variable is unique per pod
-- Action: Restart affected pods after fixing environment variable
```


---


**Task 3.2: Update QuartzServiceExtensions.cs**


**File:** `Todo.API/Extensions/QuartzServiceExtensions.cs`


Replace entire `AddQuartzConfiguration()` method:


```csharp
public static IServiceCollection AddQuartzConfiguration(this IServiceCollection services, IConfiguration configuration)
{
    services.AddQuartz(q =>
    {
        // Type loader configuration
        q.UseSimpleTypeLoader();

        // ⭐ CHANGE: Switch to AdoJobStore with MySQL
        q.UseAdoJobStore(options =>
        {
            // Database connection
            // Added the injection via IConfiguration to bound to environment variable
            options.ConnectionString = configuration["Quartz:DataSource:ConnectionString"]
                ?? throw new InvalidOperationException("Quartz connection string not configured");
            options.Provider = StdAdoStoreProvider.MySqlConnector;
            options.TablePrefix = "qrtz_";

            // ⭐ CLUSTERING CONFIGURATION (Critical for horizontal scaling)
            options.ClusteredNode = true; // Enable distributed coordination
            options.InstanceId = configuration["Quartz:InstanceId"]
                ?? throw new InvalidOperationException(
                    "Quartz instance ID not configured. Set POD_NAME or Quartz:InstanceId."); // From environment (see Task 3.1)
            options.CheckInInterval = 15000; // Heartbeat every 15 seconds (tuned for small clusters)
            options.ClusterCheckinThrottleWindowMs = 40000; // Throttle check-in window (prevents DB thrashing)

            // Job data serialization
            options.UseProperties = true;
            options.MisfireThreshold = 60000; // 60 second misfire threshold
        });

        // Thread pool configuration (unchanged)
        q.UseDefaultThreadPool(tp =>
        {
            tp.MaxConcurrency = 3;
        });


        // ============================================
        // Job & Trigger Definitions (UNCHANGED)
        // ============================================

        // Daily Report Job
        var dailyReportJobKey = new JobKey("DailyTaskReportJob", "EmailJobs");
        var dailyReportTrigger = new TriggerKey("DailyReportTrigger", "EmailJobs");
        q.AddJob<DailyTodoItemReportJob>(opts => opts
            .WithIdentity(dailyReportJobKey)
            .WithDescription("Send daily task report email at 6:00 PM")
            .DisallowConcurrentExecution());


        q.AddTrigger(opts => opts
            .ForJob(dailyReportJobKey)
            .WithIdentity(dailyReportTrigger)
            .WithCronSchedule("0 0 18 * * ?")
            .WithDescription("Trigger for daily report at 6:00 PM"));


        // Weekly Summary Job
        var weeklySummaryJobKey = new JobKey("WeeklyTaskSummaryJob", "EmailJobs");
        var weeklySummaryReportTrigger = new TriggerKey("WeeklySummaryTrigger", "EmailJobs");
        q.AddJob<WeeklyTaskSummaryJob>(opts => opts
            .WithIdentity(weeklySummaryJobKey)
            .WithDescription("Send weekly task summary every Monday at 9:00 AM")
            .DisallowConcurrentExecution());


        q.AddTrigger(opts => opts
            .ForJob(weeklySummaryJobKey)
            .WithIdentity(weeklySummaryReportTrigger)
            .WithCronSchedule("0 0 9 ? * MON")
            .WithDescription("Trigger for weekly summary every Monday"));


        // Task Reminder Job
        var reminderJobKey = new JobKey("TaskReminderJob", "EmailJobs");
        var reminderReportTrigger = new TriggerKey("ReminderTrigger", "EmailJobs");
        q.AddJob<TaskReminderJob>(opts => opts
            .WithIdentity(reminderJobKey)
            .WithDescription("Send task reminder every morning at 8:00 AM")
            .DisallowConcurrentExecution());


        q.AddTrigger(opts => opts
            .ForJob(reminderJobKey)
            .WithIdentity(reminderReportTrigger)
            .WithCronSchedule("0 0 8 * * ?")
            .WithDescription("Trigger for morning task reminder"));
    });

    // Hosted service configuration (unchanged)
    services.AddQuartzHostedService(options =>
    {
        options.WaitForJobsToComplete = true;
        options.AwaitApplicationStarted = true;
    });

    return services;
}
```


    "Quartz": {
        "quartz.scheduler.instanceName": "TodoAppScheduler",
        "quartz.scheduler.instanceId": "AUTO",
        "quartz.jobStore.type": "Quartz.Impl.AdoJobStore.JobStoreTX, Quartz.Providers.MySql",
        "quartz.jobStore.driverDelegateType": "Quartz.Impl.AdoJobStore.MySql.MySqlConnectorDelegate, Quartz.Providers.MySql",
        "quartz.jobStore.dataSource": "default",
        "quartz.jobStore.tablePrefix": "qrtz_",
        "quartz.jobStore.useProperties": "true",
        "quartz.jobStore.misfireThreshold": "60000",
        "quartz.jobStore.clustered": "true",
        "quartz.jobStore.clusterCheckinInterval": "15000",
        "quartz.jobStore.clusterCheckinThrottleWindowMs": "40000",
        "quartz.dataSource.default.connectionString": "#{QUARTZ_DB_CONNECTION_STRING}#",
        "quartz.dataSource.default.provider": "MySqlConnector",
        "quartz.threadPool.type": "Quartz.Simpl.DefaultThreadPool, Quartz",
        "quartz.threadPool.threadCount": "3",
        "quartz.serializer.type": "json"
# Verify: Job executes
# Verify: qrtz_fired_triggers table populated
# Verify: qrtz_scheduler_state shows 1 active node
```


**Task 4.3: Multi-Instance Integration Test** ⭐ CRITICAL
```bash
# Start database with migration applied
# Start Docker Compose with 3 backend services
# At scheduled job time (adjust clock for testing), verify:
# - Only 1 email sent (not 3)
# - All 3 pods show in qrtz_scheduler_state
# - Exactly 1 row in qrtz_fired_triggers per job execution
# - Lock table (qrtz_locks) shows transient entries (appear/disappear quickly)
```


**Task 4.4: Pod Restart Resilience Test**
```bash
# Start 3 pods
# Wait for scheduled job (e.g., 6:00 PM)
# Kill Pod 1 mid-execution (if possible)
# Verify: Pod 2 or 3 takes over, job completes
# Restart Pod 1
# Verify: Pod 1 rejoins cluster (appears in qrtz_scheduler_state)
# Wait for next scheduled job
# Verify: Only 1 email sent (not duplicated)
```


**Task 4.5: Query Execution History**
```sql
-- Verify operations team can query job history
SELECT trigger_name, fired_time, instance_name, state
FROM qrtz_fired_triggers
WHERE sched_name = 'TodoAppScheduler'
ORDER BY fired_time DESC
LIMIT 10;


-- Expected: Shows execution history from all pods
-- Example:
-- DailyReportTrigger | 1715872800000 | pod1-abc123 | EXECUTED
-- DailyReportTrigger | 1715789400000 | pod2-def456 | EXECUTED
-- etc.
```


---


### Phase 5: Deployment & Rollout (0.5 days)


**Task 5.1: Database Migration Deployment**
- Deploy EF Core migration to production database
- Verify all qrtz_* tables created
- Monitor: No locking, reasonable table sizes


**Task 5.2: Code Deployment (Rolling Update)**
```bash
# Update deployment YAML to use new image with AdoJobStore code
# Rolling update (1 pod at a time):
# - Pod 1 updated with new code
# - Verify: Pod 1 registers in qrtz_scheduler_state
# - Pod 1 executes scheduled jobs (no duplicates with existing pods)
# - Pod 2 updated with new code
# - Verify: Pod 2 registers, both pods coordinate correctly
# - Pod 3 updated with new code
# - Verify: All 3 pods coordinating, no duplicate executions
```


**Task 5.3: Post-Deployment Validation**
```bash
# Monitoring checklist:
# ✓ Monitor qrtz_locks table: Locks acquired/released without deadlocks
# ✓ Monitor qrtz_scheduler_state: All pods registered, regular heartbeats
# ✓ Monitor qrtz_fired_triggers: Job executions recorded correctly
# ✓ Monitor email logs: Exactly 1 email per scheduled time (not N)
# ✓ Monitor application logs: No clustering-related errors
# ✓ Monitor job execution metrics: All 3 jobs executing as expected
```


---


## Risks & Mitigations


| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|-----------|
| **Database Lock Contention** | Medium | Medium | Set appropriate check-in intervals; monitor qrtz_locks table; scale database if needed |
| **Migration Failure** | Low | High | Test migration locally first; backup production database before; rollback plan ready |
| **Job Schedule Mismatch During Rollout** | Low | Medium | Use rolling deployment (1 pod at a time); verify schedules match during transition |
| **Increased Database Load** | Medium | Low | Quartz overhead minimal; monitor DB CPU/memory; scale database if needed |
| **Network Partition (Pod Isolation)** | Low | High | Implement health checks; auto-remove dead node entries after TTL (auto-handled by Quartz) |
| **Deadlock in qrtz_locks** | Low | High | Quartz handles lock expiry; monitor logs for deadlock messages; escalation plan: manual lock cleanup |
| **Configuration Misalignment** | Medium | Medium | Validate config at startup (see code below); pre-prod testing; automated deployment tests |


### Mitigation Details


**Database Lock Contention:**
```sql
-- Monitor lock wait times
SELECT * FROM qrtz_locks WHERE lock_time > DATE_SUB(NOW(), INTERVAL 1 MINUTE);


-- Alert if locks held > 5 minutes (indicates deadlock)
-- Action: Check job logs; if safe, manually DELETE FROM qrtz_locks
```


**Configuration Startup Validation:**
```csharp
// Add to Program.cs after AddQuartzConfiguration()
var quartzSection = builder.Configuration.GetSection("Quartz");
if (!quartzSection.Exists())
   throw new InvalidOperationException("Quartz configuration section not found");


var clustered = quartzSection["quartz.jobStore.clustered"];
if (clustered != "true")
   throw new InvalidOperationException("Quartz clustering not enabled (required for multi-pod deployment)");


_logger.LogInformation("✓ Quartz clustering enabled and validated");
```


**Rollback Plan:**
```bash
# If issues detected post-deployment:
# 1. Revert to previous image version
# 2. Re-apply old code with UseInMemoryStore()
# 3. Quartz gracefully falls back to in-memory (no data loss risk, tables still exist)
# 4. Investigate root cause
# 5. Retest before re-deploying
```


---


## Assumptions


1. **MySQL 8.0 is stable and will remain in use** - The application already uses MySQL; no plans to change databases
2. **Current Quartz job definitions don't change during implementation** - 3 jobs remain (Daily, Weekly, Reminder)
3. **Production environment supports MySQL connection pooling** - Standard in cloud environments
4. **Deployment supports rolling updates** - Required to avoid temporary duplicate executions during rollout
5. **Operational team can query databases for troubleshooting** - Querying qrtz_* tables for debugging
6. **Job execution is idempotent at email level** - If somehow email is sent twice, business logic handles it (assumed for now; separate ticket for full idempotency)
7. **Network is reasonably stable** - Distributed locking requires connectivity; temporary partitions auto-recover


---


## Definition of Done


### Code Changes
- [ ] NuGet package `Quartz.Providers.MySql` added to `Todo.API.csproj`
- [ ] EF Core migration created and tested locally
- [ ] `QuartzServiceExtensions.cs` updated with AdoJobStore configuration
- [ ] `appsettings.json` updated with Quartz JDBC settings
- [ ] `appsettings.Production.json` reviewed and updated (if exists)
- [ ] All code changes reviewed and merged to main branch
- [ ] No compilation errors or warnings


### Testing
- [ ] Unit tests pass (existing job tests)
- [ ] Single-instance integration test passes
 - [ ] Job executes at scheduled time
 - [ ] qrtz_fired_triggers populated with execution record
 - [ ] qrtz_scheduler_state shows 1 active node
- [ ] Multi-instance integration test passes (Docker Compose 3 backends)
 - [ ] Only 1 email sent (not 3) at job execution time
 - [ ] All 3 pods appear in qrtz_scheduler_state
 - [ ] Distributed locking visible in qrtz_locks (transient entries)
 - [ ] qrtz_fired_triggers shows 1 execution per trigger per time
- [ ] Pod restart resilience test passes
 - [ ] Pod crash during execution handled gracefully
 - [ ] Job completes via remaining pods
 - [ ] Restarted pod rejoins cluster
 - [ ] No duplicate executions on restart
- [ ] Execution history queryable
 - [ ] SQL query on qrtz_fired_triggers returns job history
 - [ ] Timestamps accurate and consistent (UTC)


### Deployment
- [ ] Database migration applied to production (pre-deployment)
- [ ] All qrtz_* tables verified to exist in production DB
- [ ] Code deployed via rolling update (1 pod at a time)
- [ ] All 3 pods successfully registered in qrtz_scheduler_state
- [ ] All 3 pods coordinating (no duplicate executions observed)
- [ ] Monitoring dashboards show expected metrics
 - [ ] qrtz_locks: No deadlocks, locks acquired/released cleanly
 - [ ] qrtz_scheduler_state: 3 active nodes with regular heartbeats
 - [ ] qrtz_fired_triggers: Correct execution counts (1 per trigger per time)
 - [ ] Email delivery: 1 email per job per scheduled time (not 3)


### Documentation
- [ ] Runbook created: "Troubleshooting Quartz Job Execution Issues"
 - [ ] How to query job history
 - [ ] How to detect duplicate executions
 - [ ] How to manually recover from deadlocks
 - [ ] How to pause/resume jobs
- [ ] Architecture diagram updated showing distributed scheduler
- [ ] Code comments added explaining clustering configuration
- [ ] Team notified of new qrtz_* tables in database


### Monitoring & Alerts
- [ ] Alert configured: "Quartz lock contention (locks held >5 min)"
- [ ] Alert configured: "Quartz dead node detected (missing heartbeat)"
- [ ] Alert configured: "Duplicate job execution (>1 execution per trigger per time)"
- [ ] Dashboard created showing Quartz cluster health
 - [ ] Active nodes
 - [ ] Job execution counts
 - [ ] Lock contention metrics
 - [ ] Failed job count


---


## Acceptance Criteria


### Functional
1. **Exactly-Once Execution:** With N pods (N ≥ 2), scheduled jobs execute exactly once per scheduled time
2. **Distributed Locking:** Quartz automatically prevents concurrent execution across pods via database locks
3. **Execution History:** Job execution history queryable from `qrtz_fired_triggers` table
4. **Pod Resilience:** If one pod crashes, remaining pods continue executing jobs without interruption
5. **Cluster Coordination:** All pods register themselves and maintain heartbeat in `qrtz_scheduler_state`


### Non-Functional
1. **Performance:** Job execution latency unchanged (< 30 seconds for email delivery)
2. **Database Load:** Quartz overhead minimal (< 5% additional CPU on MySQL)
3. **Scalability:** Solution supports 10+ pods without lock contention issues
4. **Backwards Compatibility:** Existing job definitions work unchanged; no API changes


### Operational
1. **Configuration:** All environment-specific settings in configuration files (not hardcoded)
2. **Monitoring:** Operators can diagnose job issues using qrtz_* tables
3. **Debugging:** Failed jobs logged with clear error messages (not silent failures)
4. **Troubleshooting:** Runbook available for common issues (deadlocks, duplicate executions, missed jobs)


---


## Related Issues & Dependencies


**Depends On:**
- ✓ Database migrations must be applied before deployment


**Enables:**
- [ ] TICKET-IDEMPOTENCY: Add email idempotency tokens (downstream, not blocking)
- [ ] TICKET-EMAIL-RELIABILITY: SMTP timeout configuration (upstream, not blocking)
- [ ] TICKET-DYNAMIC-JOBS: Externalize job schedules to configuration (future enhancement)
- [ ] Kubernetes horizontal scaling: Can now safely scale to 3+ pods


**Related (No direct dependency):**
- TICKET-CACHING: Redis cache layer for performance
- TICKET-MONITORING: Prometheus metrics for job execution


---


## Questions & Notes


**Q: What about data migration from in-memory to database?** 
A: Not needed. In-memory store doesn't persist data. Starting fresh with database-backed store.


**Q: Will this impact existing users while we're rolling out?** 
A: Minimal impact. Rolling deployment ensures at least one pod is always available. Short window during pod transition where old and new code run together (both safe).


**Q: How do we know if clustering is working?** 
A: Query `SELECT * FROM qrtz_scheduler_state;` - should see all pods listed with recent `LAST_CHECKIN_TIME` values.


**Q: Can we disable clustering if something goes wrong?** 
A: Yes. Revert to `UseInMemoryStore()` in code. Database tables remain but are ignored. Rollback plan documented above.


**Q: What happens during pod restart?** 
A: Old pod's entry in `qrtz_scheduler_state` eventually times out (auto-removed by Quartz). New pod registers immediately. Other pods continue executing jobs without interruption.


---


## References & Appendices


### Appendix A: Quartz MySQL Schema DDL
[See separate QUARTZ_SCHEMA_DDL.sql file - 11 tables with indexes]


### Appendix B: Configuration Properties Reference
| Property | Value | Purpose |
|----------|-------|---------|
| `jobStore.type` | `Quartz.Impl.AdoJobStore.JobStoreTX` | Job storage implementation |
| `jobStore.driverDelegateType` | `Quartz.Impl.AdoJobStore.MySql.MySqlConnectorDelegate` | MySQL dialect |
| `jobStore.clustered` | `true` | Enable clustering mode |
| `jobStore.clusterCheckinInterval` | `15000` | Heartbeat every 15 seconds |
| `jobStore.clusterCheckinThrottleWindowMs` | `40000` | Throttle window |
| `jobStore.tablePrefix` | `qrtz_` | Table name prefix |


### Appendix C: Monitoring Queries
```sql
-- Active scheduler nodes
SELECT scheduler_name, instance_name, last_checkin_time FROM qrtz_scheduler_state;


-- Recent job executions
SELECT trigger_name, fired_time, instance_name, state FROM qrtz_fired_triggers
ORDER BY fired_time DESC LIMIT 20;


-- Detect duplicate executions
SELECT trigger_name, DATE(FROM_UNIXTIME(fired_time/1000)) as execution_date, COUNT(*)
FROM qrtz_fired_triggers
GROUP BY trigger_name, execution_date
HAVING COUNT(*) > 1;


-- Lock contention check
SELECT * FROM qrtz_locks WHERE lock_time < DATE_SUB(NOW(), INTERVAL 5 MINUTE);
```


---


**Created:** May 16, 2026 
**Last Updated:** May 16, 2026 
**Version:** 1.0 - Ready for Implementation