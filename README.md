# project

## Jenkins conditional job chaining

This repository includes a `Jenkinsfile` that chains multiple downstream jobs
with conditional triggers to simulate a complex, real-world CI/CD pipeline.

---

### Pipeline flow

```
Build
  Γöé
  Γö£ΓöÇ [RUN_TESTS=true && build=SUCCESS]
  Γöé     Γö£ΓöÇΓöÇ Unit Tests        ΓöÉ  (parallel)
  Γöé     ΓööΓöÇΓöÇ Integration Tests Γöÿ
  Γöé
  Γö£ΓöÇ [RUN_STAGING_DEPLOY=true && tests=SUCCESS]
  Γöé     ΓööΓöÇΓöÇ Deploy to Staging
  Γöé           ΓööΓöÇΓöÇ [RUN_SMOKE_TESTS=true && staging deploy=SUCCESS]
  Γöé                 ΓööΓöÇΓöÇ Smoke Tests (Staging)
  Γöé
  Γö£ΓöÇ [RUN_PROD_DEPLOY=true && branch=TARGET_BRANCH && staging=SUCCESS && smoke=SUCCESS]
  Γöé     ΓööΓöÇΓöÇ Manual Approval Gate  ΓåÉ 30-minute timeout, submitter: release-team
  Γöé           ΓööΓöÇΓöÇ Deploy to Production
  Γöé                 ΓööΓöÇΓöÇ [RUN_SMOKE_TESTS=true && prod deploy=SUCCESS]
  Γöé                       ΓööΓöÇΓöÇ Smoke Tests (Production)
  Γöé
  ΓööΓöÇ [always] Notify Job  ΓåÉ fire-and-forget with full result context

Rollback is triggered automatically (fire-and-forget) on any deploy failure
when ENABLE_ROLLBACK=true.
```

---

### Conditional trigger logic

| Stage                    | Condition                                                                 |
|--------------------------|---------------------------------------------------------------------------|
| Tests (parallel)         | `RUN_TESTS=true` AND `build=SUCCESS`                                      |
| Deploy to Staging        | `RUN_STAGING_DEPLOY=true` AND `build=SUCCESS` AND all tests passed/skipped|
| Smoke Tests (Staging)    | `RUN_SMOKE_TESTS=true` AND `staging deploy=SUCCESS`                       |
| Approve Production       | `RUN_PROD_DEPLOY=true` AND `branch=TARGET_BRANCH` AND staging+smoke=SUCCESS|
| Deploy to Production     | Same as above PLUS pipeline not already failed                            |
| Smoke Tests (Production) | `RUN_SMOKE_TESTS=true` AND `prod deploy=SUCCESS`                          |
| Rollback                 | Any deploy failure AND `ENABLE_ROLLBACK=true`                             |
| Notify                   | Always (post.always)                                                      |

---

### Parameters

#### Job names
| Parameter              | Default                  | Description                          |
|------------------------|--------------------------|--------------------------------------|
| `BUILD_JOB`            | `build-job`              | Compile / package job                |
| `UNIT_TEST_JOB`        | `unit-test-job`          | Unit-test job                        |
| `INTEGRATION_TEST_JOB` | `integration-test-job`   | Integration-test job                 |
| `STAGING_DEPLOY_JOB`   | `staging-deploy-job`     | Deploy to staging                    |
| `PROD_DEPLOY_JOB`      | `prod-deploy-job`        | Deploy to production                 |
| `ROLLBACK_JOB`         | `rollback-job`           | Rollback on deploy failure           |
| `SMOKE_TEST_JOB`       | `smoke-test-job`         | Post-deploy smoke tests              |
| `NOTIFY_JOB`           | `notify-job`             | Notification job                     |

#### Behavioural flags
| Parameter              | Default | Description                                        |
|------------------------|---------|----------------------------------------------------|
| `RUN_TESTS`            | `true`  | Run unit + integration tests after build           |
| `RUN_STAGING_DEPLOY`   | `true`  | Deploy to staging when tests pass                  |
| `RUN_PROD_DEPLOY`      | `false` | Deploy to production (requires manual gate)        |
| `RUN_SMOKE_TESTS`      | `true`  | Run smoke tests after each deploy                  |
| `ENABLE_ROLLBACK`      | `true`  | Auto-trigger rollback job on deploy failure        |

#### Context
| Parameter       | Default      | Description                                       |
|-----------------|--------------|---------------------------------------------------|
| `TARGET_BRANCH` | `main`       | Only this branch may deploy to production         |
| `ARTIFACT_ID`   | _(empty)_    | Passed downstream; defaults to `BUILD_NUMBER`     |
| `DEPLOY_REGION` | `us-east-1`  | Cloud region for deploy/rollback jobs             |

---

### Artifact propagation

Every downstream job receives `ARTIFACT_ID` so they all operate on the exact
same build artifact. If you leave the parameter blank, the orchestrator
defaults it to the Jenkins `BUILD_NUMBER`.

### Rollback behaviour

When a deploy fails and `ENABLE_ROLLBACK=true`, `triggerRollback()` fires the
`ROLLBACK_JOB` as a non-blocking, fire-and-forget call, passing the
`ARTIFACT_ID`, `ENVIRONMENT`, and `DEPLOY_REGION` so the rollback job knows
exactly what to undo.

### Manual production gate

The `Approve Production Deploy` stage calls `input`, which pauses the pipeline
for up to 30 minutes waiting for a member of the `release-team` group to
click **Deploy**. If nobody approves in time the stage aborts and the pipeline
is marked as failed ΓÇö production is never touched.
