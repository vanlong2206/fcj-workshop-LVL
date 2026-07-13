---
title: "AWS EventBridge"
date: 2026-06-18
weight: 8
chapter: false
pre: " <b> 5.8. </b> "
---

#### AWS EventBridge

#### 5.8.1 Concept

**Amazon EventBridge** is a serverless event bus service provided by AWS that allows connecting applications together using events. EventBridge helps build Event-Driven Architecture easily, flexibly, and with high scalability.

#### 5.8.2 EventBridge System Architecture

![1783852903546](image/_index.vi/1783852903546.png)

<div align="center"><i>Figure 5.8.1: EventBridge architecture diagram.</i></div>

- **EventBridge** triggers Maintenance Lambda on a Cron schedule.
- **Lambda** enables Maintenance Mode — updates API Gateway stage variable (`maintenance=true`) and DB flag (`SystemConfig.maintenance_mode=true`) to reject new requests.
- **Lambda** connects directly to Aurora PostgreSQL via TypeORM (IAM authentication).
- Performs maintenance tasks:
  - `start_maintenance`: Enables maintenance mode (no DB tasks).
  - `stop_maintenance`: Disables maintenance mode.
  - `vacuum_analyze`: VACUUM ANALYZE + REINDEX on 15 tables.
  - `reset_daily`: Reset stamina to 20.0, record `last_daily_reset` timestamp.
- After completion (or if action is not start/stop), Lambda disables maintenance mode.

#### 5.8.3 Creating EventBridge Scheduler

Declared in the events block of the Lambda (automatic) in file services/lambda-maintenance/serverless.yml:

```yaml
functions:
  handler:
    handler: src/lambda.handler
    timeout: 120
    memorySize: 256
    events:
      - eventBridge:
          schedule: cron(0 10 ? * MON *)
          input:
            action: start_maintenance
      - eventBridge:
          schedule: cron(0 12 ? * MON *)
          input:
            action: stop_maintenance
      - eventBridge:
          schedule: cron(0 3 * * ? *)
          input:
            action: vacuum_analyze
      - eventBridge:
          schedule: cron(0 0 * * ? *)
          input:
            action: reset_daily
```

#### 5.8.4 Building Maintenance Lambda

##### Directory structure

```
services/lambda-maintenance/
├── package.json
├── serverless.yml
├── tsconfig.json
└── src/
    ├── lambda.ts                 # Entry point — EventBridge handler
    ├── index.ts                  # Dev script (local run)
    └── handlers/
        ├── startMaintenance.ts   # Enable maintenance mode
        ├── stopMaintenance.ts    # Disable maintenance mode
        ├── vacuumAnalyze.ts      # VACUUM + ANALYZE + REINDEX
        ├── resetDaily.ts         # Reset stamina + timestamp
```

##### Entry point dispatcher

File: `services/lambda-maintenance/src/lambda.ts`

Lambda receives EventBridge event, parses the `action` field, and dispatches to the corresponding handler:

```typescript
type MaintenanceAction = 'start_maintenance' | 'stop_maintenance'
  | 'vacuum_analyze' | 'reset_daily' | 'cleanup_data';

export const handler = async (event: EventBridgeEvent<'Scheduled Event', MaintenanceEvent>): Promise<void> => {
  await initializeApplicationDbContext();

  const { action } = event.detail;

  switch (action) {
    case 'start_maintenance':  await handleStartMaintenance();  break;
    case 'stop_maintenance':   await handleStopMaintenance();   break;
    case 'vacuum_analyze':     await handleVacuumAnalyze();     break;
    case 'reset_daily':        await handleResetDaily();        break;
  }

  await putMetric('MaintenanceAction', 1, 'Count', ...);
  await putMetric('MaintenanceDuration', elapsed, 'Milliseconds', ...);
};
```

##### Lambda execution steps

With action `start_maintenance`:

**Enable Maintenance Mode** — call API Gateway `UpdateStageCommand` set `maintenance=true`

**Set DB flag** — upsert `SystemConfig.maintenance_mode = true`

**Write CloudWatch Logs** — each step has logs

With action `vacuum_analyze`:

**Connect to Aurora PostgreSQL** — via `ApplicationDbContext`

**VACUUM ANALYZE** — per table

**REINDEX** — per table

**Write CloudWatch Logs** + Metrics

#### 5.8.5 Maintenance Mode Handling

##### Maintenance Mode uses 2 parallel mechanisms:

**API Gateway V2 Stage Variable**: `maintenance=true/false` — set via `UpdateStageCommand`

**Database Flag**: `SystemConfig` key=`maintenance_mode`, value=`true/false`

##### Middleware check

File: `shared/src/middlewares/maintenance.middleware.ts`

```typescript
export const maintenanceMiddleware = async (req, res, next) => {
  const repo = ApplicationDbContext.getRepository(SystemConfig);
  const config = await repo.findOne({ where: { key: "maintenance_mode" } });

  if (config?.value === "true") {
    res.status(503).json({
      error: "Service Unavailable",
      message: "System is under maintenance. Please try again later.",
    });
    return;
  }
  next();
};
```

Middleware is injected into all 6 Lambda domain services (auth, economy, inventory, transaction, progression-world, loot-reward), running right after DB initialization and before routes.

##### When isMaintenance = true

```
HTTP 503 Service Unavailable
{
  "error": "Service Unavailable",
  "message": "System is under maintenance. Please try again later."
}
```

##### When isMaintenance = false

System operates normally, middleware bypass.

#### 5.8.6 Testing

##### * Manual Trigger

```json
{
  "version": "0",
  "id": "test-event-001",
  "detail-type": "Scheduled Event",
  "source": "aws.events",
  "time": "2026-07-10T10:00:00Z",
  "detail": {
    "action": "start_maintenance"
  }
}
```

![1783870580554](image/_index.vi/1783870580554.png)

<div align="center"><i>Figure 5.8.2: Start maintenance in Test tab.</i></div>

Click **Test** — Lambda is triggered, expected to return successfully (200 OK).

##### * Check Lambda triggered

![1783870803214](image/_index.vi/1783870803214.png)

<div align="center"><i>Figure 5.8.3: Observe charts in Monitor tab.</i></div>

- **Invocations** — chart shows increased call count after testing
- **Duration** — processing time (expected a few seconds to tens of seconds)
- **Error count & success rate (%)** — expected 0% errors

##### * Check API during maintenance

![1783871048041](image/_index.vi/1783871048041.png)

<div align="center"><i>Figure 5.8.4: Returns HTTP 503 during maintenance.</i></div>

##### * Check CloudWatch Logs

![1783873833229](image/_index.vi/1783873833229.png)

<div align="center"><i>Figure 5.8.5: Check logs in CloudWatch.</i></div>

##### * Check CloudWatch Metrics & Alarm

![1783871319914](image/_index.vi/1783871319914.png)

<div align="center"><i>Figure 5.8.6: Check metrics charts.</i></div>

![1783871376765](image/_index.vi/1783871376765.png)

<div align="center"><i>Figure 5.8.7: Check Alarms.</i></div>

Alarms OK — No errors, maintenance running well.

##### * Stop maintenance

```json
{
  "version": "0",
  "id": "test-event-002",
  "detail-type": "Scheduled Event",
  "source": "aws.events",
  "time": "2026-07-10T12:00:00Z",
  "detail": {
    "action": "stop_maintenance"
  }
}
```

![1783873987556](image/_index.vi/1783873987556.png)

<div align="center"><i>Figure 5.8.8: Stop maintenance in Test tab.</i></div>

#### 5.8.7 Results

The maintenance system is fully automated via **4 EventBridge Rules**, each rule triggers `gameapi-maintenance-dev-handler` Lambda on its own Cron schedule. Maintenance Mode uses both API Gateway stage variable and DB flag (`SystemConfig.maintenance_mode`), combined with middleware returning HTTP 503 to block requests during maintenance.

For database optimization, Lambda runs **VACUUM ANALYZE + REINDEX** on 15 tables at 03:00 UTC daily. Periodic data reset occurs at 00:00 UTC daily, including cleaning up old data (deactivating expired gift codes, deleting expired shop logs and save data), resetting player stamina to 20.0, and recording `last_daily_reset` timestamp.

The entire process is recorded in detail through **CloudWatch Logs** and 3 metrics (`MaintenanceAction`, `MaintenanceDuration`, `MaintenanceActionFailed`). CloudWatch Alarm `gameapi-maintenance-action-failed` will alert if any errors occur during maintenance, helping the operations team detect and handle issues promptly.
