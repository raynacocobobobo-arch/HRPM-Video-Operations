# Video Operations Scheduling Service Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Produce explainable crew-assignment proposals with Timefold and let the OpenProject plugin store, display, reject, or atomically apply those proposals without giving the solver database write access.

**Architecture:** A stateless Java 21/Quarkus service receives an immutable, versioned scheduling snapshot over HTTP. Timefold Solver assigns eligible crew to production time blocks and post-production capacity demands using explicit hard and soft constraints. The OpenProject plugin records the response as a proposal, compares its input version to current data, and applies the whole proposal in one database transaction only after human confirmation.

**Tech Stack:** Timefold Solver 2.6.0, Quarkus 3.38.3, Java 21, Maven Wrapper, JUnit 5, Timefold ConstraintVerifier, RestAssured, Jackson, Rails HTTP client, RSpec/WebMock.

**Spec:** `../specs/2026-09-01-video-operations-openproject-timefold-design.md`

## Package Constraints

- Packages 1 and 2 must pass first.
- Pin every Timefold artifact to `2.6.0`, Quarkus to `3.38.3`, and Java release to `21`.
- Preserve Apache-2.0 notices for Timefold quickstart-derived structure.
- The solver container receives no PostgreSQL URL, credentials, bind mount, or database network permission.
- Solve requests are snapshots. The service must not retain request data after returning the response.
- A solver result is a proposal, never an instruction to write assignments.
- Hard constraints must be unbreakable. A response with a negative hard score is `INFEASIBLE` and cannot be applied.
- A proposal may be applied only when its `data_version` equals `ProjectState#scheduling_data_version` for that project.
- Application is whole-plan only: no partial row selection, no silent merge, no best-effort write.

---

## Planned File Map

```text
video-operations/
├── scheduler/
│   ├── .mvn/wrapper/
│   ├── mvnw
│   ├── mvnw.cmd
│   ├── pom.xml
│   ├── Dockerfile
│   ├── NOTICE
│   └── src/
│       ├── main/java/com/videoops/scheduler/
│       │   ├── api/ScheduleResource.java
│       │   ├── api/dto/ScheduleRequest.java
│       │   ├── api/dto/ScheduleResponse.java
│       │   ├── api/dto/ScheduleMapper.java
│       │   ├── domain/Schedule.java
│       │   ├── domain/PlannedAssignment.java
│       │   ├── domain/Worker.java
│       │   ├── domain/Demand.java
│       │   ├── domain/LockedAssignment.java
│       │   ├── domain/AvailabilityWindow.java
│       │   ├── domain/LocationTravelTime.java
│       │   ├── solver/ScheduleConstraintProvider.java
│       │   └── service/ScheduleSolverService.java
│       └── test/java/com/videoops/scheduler/
│           ├── api/ScheduleResourceTest.java
│           ├── api/ScheduleContractTest.java
│           ├── solver/HardConstraintsTest.java
│           ├── solver/SoftConstraintsTest.java
│           └── service/ScheduleSolverServiceTest.java
├── compose/docker-compose.override.yml
└── openproject-video-operations/
    ├── db/migrate/20260902090400_create_video_operations_schedule_proposals.rb
    ├── app/models/video_operations/schedule_proposal.rb
    ├── app/services/video_operations/scheduling/
    │   ├── snapshot_builder.rb
    │   ├── proposal_client.rb
    │   ├── proposal_create.rb
    │   └── proposal_apply.rb
    ├── app/controllers/video_operations/api/v1/schedule_proposals_controller.rb
    └── spec/
        ├── services/video_operations/scheduling/
        └── requests/video_operations/api/v1/schedule_proposals_spec.rb
```

### Task 1: Scaffold the pinned Timefold/Quarkus service

**Files:**
- Create: `scheduler/pom.xml`
- Create: `scheduler/mvnw`
- Create: `scheduler/mvnw.cmd`
- Create: `scheduler/.mvn/wrapper/maven-wrapper.properties`
- Create: `scheduler/NOTICE`
- Create: `scheduler/src/main/resources/application.properties`
- Create: `scheduler/src/test/java/com/videoops/scheduler/api/ScheduleResourceTest.java`
- Create: `scheduler/src/main/java/com/videoops/scheduler/api/ScheduleResource.java`
- Modify: `UPSTREAMS.md`

**Interfaces:**
- `GET /q/health` is supplied by Quarkus SmallRye Health.
- `GET /api/v1/schedules/health` returns `{"status":"ok","timefoldVersion":"2.6.0"}`.

- [ ] **Step 1: Create the Maven project from the official employee-scheduling quickstart dependency pattern**

The POM must set:

```xml
<properties>
  <maven.compiler.release>21</maven.compiler.release>
  <quarkus.platform.version>3.38.3</quarkus.platform.version>
  <timefold.solver.version>2.6.0</timefold.solver.version>
</properties>
```

Include only these runtime capabilities in this package:

- `ai.timefold.solver:timefold-solver-quarkus:2.6.0`
- `ai.timefold.solver:timefold-solver-quarkus-jackson:2.6.0`
- `io.quarkus:quarkus-rest-jackson`
- `io.quarkus:quarkus-smallrye-health`
- test-scoped `timefold-solver-test`, `quarkus-junit`, `rest-assured`, and AssertJ.

Pin Maven Compiler Plugin `3.15.0` and Surefire `3.5.6`, matching the current official quickstart. Commit the generated Maven Wrapper; do not require a global Maven install.

- [ ] **Step 2: Write the failing service health test**

```java
@QuarkusTest
class ScheduleResourceTest {
    @Test
    void healthReportsPinnedTimefoldVersion() {
        given()
            .when().get("/api/v1/schedules/health")
            .then().statusCode(200)
            .body("status", equalTo("ok"))
            .body("timefoldVersion", equalTo("2.6.0"));
    }
}
```

Run:

```bash
./scheduler/mvnw -f scheduler/pom.xml -Dtest=ScheduleResourceTest test
```

Expected: fail because the resource does not exist.

- [ ] **Step 3: Implement the health resource and verify dependency resolution**

```bash
./scheduler/mvnw -f scheduler/pom.xml -Dtest=ScheduleResourceTest test
./scheduler/mvnw -f scheduler/pom.xml dependency:tree \
  -Dincludes=ai.timefold.solver:timefold-solver-core
```

Expected: test passes and dependency tree contains Timefold `2.6.0` only.

- [ ] **Step 4: Record attribution and commit**

`NOTICE` and `UPSTREAMS.md` must identify the exact official quickstart URL, Apache-2.0 license, inspected release, and the fact that only structure/dependency choices were reused.

```bash
git add scheduler UPSTREAMS.md
git commit -m "build: scaffold pinned Timefold scheduler"
```

### Task 2: Define a stable scheduling snapshot contract

**Files:**
- Create: `scheduler/src/main/java/com/videoops/scheduler/api/dto/ScheduleRequest.java`
- Create: `scheduler/src/main/java/com/videoops/scheduler/api/dto/ScheduleResponse.java`
- Create: `scheduler/src/main/java/com/videoops/scheduler/api/dto/ScheduleMapper.java`
- Create: `scheduler/src/main/java/com/videoops/scheduler/domain/Worker.java`
- Create: `scheduler/src/main/java/com/videoops/scheduler/domain/Demand.java`
- Create: `scheduler/src/main/java/com/videoops/scheduler/domain/LockedAssignment.java`
- Create: `scheduler/src/main/java/com/videoops/scheduler/domain/AvailabilityWindow.java`
- Create: `scheduler/src/main/java/com/videoops/scheduler/domain/LocationTravelTime.java`
- Create: `scheduler/src/main/java/com/videoops/scheduler/domain/PlannedAssignment.java`
- Create: `scheduler/src/main/java/com/videoops/scheduler/domain/Schedule.java`
- Create: `scheduler/src/test/java/com/videoops/scheduler/api/ScheduleContractTest.java`
- Create: `scheduler/src/test/resources/contracts/minimal-request.json`
- Create: `scheduler/src/test/resources/contracts/minimal-response.json`

**Public records:**

```java
public record ScheduleRequest(
    String requestId,
    long dataVersion,
    String timezone,
    List<WorkerDto> workers,
    List<DemandDto> demands,
    List<LockedAssignmentDto> lockedAssignments,
    List<LocationTravelTimeDto> travelTimes,
    SolverOptionsDto options
) {}

public record ScheduleResponse(
    String requestId,
    long dataVersion,
    Status status,
    String score,
    List<ProposedAssignmentDto> assignments,
    List<ViolationDto> violations,
    ImpactDto impact
) {}
```

`Status` is exactly `FEASIBLE`, `INFEASIBLE`, or `ERROR`. IDs cross the boundary as opaque strings. Instants use RFC 3339 UTC text; dates use ISO 8601. The only accepted timezone in v1 is `Asia/Shanghai`.

**Demand modes:**

- `FIXED_BLOCK`: production work with exact `startsAt`, `endsAt`, and `location`.
- `DAILY_CAPACITY`: post-production work with `windowStartDate`, `windowEndDate`, `requiredMinutes`, `deadline`, and optional daily cap.

**Impact fields:** moved existing assignments, unassigned demands, overtime minutes, travel minutes, fragmentation count, and estimated labor cost cents.

- [ ] **Step 1: Write failing JSON contract tests**

Deserialize `minimal-request.json`, map it to the planning domain, map a known solution back to `minimal-response.json`, and assert exact field names and enum spellings. Add rejection tests for duplicate IDs, missing request ID, nonpositive versions, malformed intervals, unsupported timezone, and daily demands without a deadline.

- [ ] **Step 2: Implement immutable DTOs and mapper validation**

Keep JSON DTOs separate from Timefold annotations. `ScheduleMapper` returns either a valid `Schedule` or a list of ordered validation violations; it must not drop invalid records silently.

- [ ] **Step 3: Run contract tests**

```bash
./scheduler/mvnw -f scheduler/pom.xml -Dtest=ScheduleContractTest test
```

- [ ] **Step 4: Commit**

```bash
git add scheduler
git commit -m "feat: define scheduling snapshot contract"
```

### Task 3: Implement hard production and capacity constraints

**Files:**
- Create: `scheduler/src/main/java/com/videoops/scheduler/solver/ScheduleConstraintProvider.java`
- Create: `scheduler/src/test/java/com/videoops/scheduler/solver/HardConstraintsTest.java`
- Modify: `scheduler/src/main/java/com/videoops/scheduler/domain/PlannedAssignment.java`
- Modify: `scheduler/src/main/java/com/videoops/scheduler/domain/Schedule.java`

**Planning model:**

- One `PlannedAssignment` exists for each required headcount slot, using a stable ID derived from `demandId + slotIndex`.
- The planning variable is nullable `Worker worker`; null means unassigned and receives a hard penalty.
- For `FIXED_BLOCK`, start/end are problem facts.
- For `DAILY_CAPACITY`, the planning variables are `Worker worker` and a date within the demand window; allocated minutes are a problem fact and daily capacity is evaluated across assignments.
- Confirmed/manual assignments supplied in `lockedAssignments` are immutable problem facts and must be considered by overlap, rest, travel, and daily-capacity constraints.
- `Schedule` declares `@PlanningScore(bendableHardLevelsSize = 1, bendableSoftLevelsSize = 3)` so hard feasibility, published-schedule stability, operating quality, and cost/preferences cannot accidentally trade across levels.

**Hard constraints and stable constraint IDs:**

1. `required-worker-assigned`
2. `required-role-and-skill`
3. `worker-available`
4. `no-overlap-with-proposed`
5. `no-overlap-with-locked`
6. `travel-buffer-between-locations`
7. `minimum-rest-between-production-days`
8. `post-daily-capacity`
9. `within-post-window-and-deadline`
10. `demand-stage-ready`

- [ ] **Step 1: Write one failing ConstraintVerifier example per hard rule**

Use small, named fixtures. Exact edge cases must include:

- skill level `2` rejected for minimum `3`;
- a leave interval overlapping by one minute rejected, while touching endpoints pass;
- two proposed fixed blocks overlap;
- a proposed block overlaps a locked manual booking;
- different locations with a configured 60-minute travel time but only 30 minutes between jobs;
- less rest than the worker's configured `minimumRestMinutes` (fixture value `660`);
- 500 post minutes assigned against a worker's 480-minute daily capacity;
- a post assignment placed after its deadline;
- a demand whose OpenProject stage-ready flag is false.

Run:

```bash
./scheduler/mvnw -f scheduler/pom.xml -Dtest=HardConstraintsTest test
```

Expected: all cases fail before the provider is implemented.

- [ ] **Step 2: Implement constraints one at a time**

After each constraint, rerun only the named test method. Use Timefold's `ConstraintFactory` joins/grouping; do not bury scoring in entity setters. Use `BendableLongScore` with one hard level and three soft levels; hard penalties use hard level `0`, multiplied by minutes or one unit as defined in the test.

- [ ] **Step 3: Run the full hard-constraint class**

```bash
./scheduler/mvnw -f scheduler/pom.xml -Dtest=HardConstraintsTest test
```

Expected: all hard examples pass with exact scores.

- [ ] **Step 4: Commit**

```bash
git add scheduler
git commit -m "feat: enforce hard video scheduling constraints"
```

### Task 4: Implement soft quality, stability, and cost constraints

**Files:**
- Create: `scheduler/src/test/java/com/videoops/scheduler/solver/SoftConstraintsTest.java`
- Modify: `scheduler/src/main/java/com/videoops/scheduler/solver/ScheduleConstraintProvider.java`

**Soft constraints, in priority order:**

1. `preserve-existing-assignment` — strongly prefer the worker/date from the current published schedule.
2. `protect-critical-path` — prefer earlier completion for critical demands.
3. `project-continuity` — prefer the same worker across related work packages.
4. `avoid-overtime` — penalize minutes beyond preferred daily capacity before the hard maximum.
5. `minimize-travel` — penalize configured travel minutes.
6. `minimize-fragmentation` — penalize avoidable gaps and split post days.
7. `balance-workload` — penalize deviation from team-average allocated minutes.
8. `lower-labor-cost` — use effective rate cents as a final tie-breaker.
9. `honor-worker-preference` — use declared preferred/avoid project tags as the lowest operational preference.

- [ ] **Step 1: Write paired failing tests for every priority**

Each test presents two hard-feasible alternatives and proves the intended one scores better. Include one combined test proving stability outranks a small cost saving and critical-path completion outranks workload balance.

- [ ] **Step 2: Implement explicit soft weights**

Use the three soft levels lexicographically: soft level `0` for preserving published assignments, soft level `1` for critical path/continuity/overtime/travel/fragmentation/workload, and soft level `2` for labor cost/preferences. Define within-level weights as named constants, avoid runtime-configurable weights in v1, use long scores, and cap cost conversion so cent values cannot overflow.

- [ ] **Step 3: Verify exact soft scores**

```bash
./scheduler/mvnw -f scheduler/pom.xml -Dtest=SoftConstraintsTest test
```

- [ ] **Step 4: Commit**

```bash
git add scheduler
git commit -m "feat: rank stable low-friction schedule proposals"
```

### Task 5: Expose a deterministic, bounded solve endpoint

**Files:**
- Create: `scheduler/src/main/java/com/videoops/scheduler/service/ScheduleSolverService.java`
- Modify: `scheduler/src/main/java/com/videoops/scheduler/api/ScheduleResource.java`
- Create: `scheduler/src/test/java/com/videoops/scheduler/service/ScheduleSolverServiceTest.java`
- Modify: `scheduler/src/test/java/com/videoops/scheduler/api/ScheduleResourceTest.java`
- Modify: `scheduler/src/main/resources/application.properties`
- Create: `scheduler/Dockerfile`
- Modify: `compose/docker-compose.override.yml`

**Interfaces:**
- `POST /api/v1/schedules/solve` accepts `ScheduleRequest` and returns `ScheduleResponse`.
- Default `spentLimit` is five seconds; v1 accepts only `1–15` seconds.
- Same request payload, fixed seed, and pinned versions produce the same assignment ordering.
- Validation errors return `400` with ordered violations, affected opaque object IDs, a recovery-action code, and echoed `requestId`/`dataVersion`.
- Solver exceptions return `500` with an error/recovery-action code but no worker PII in logs or response.

- [ ] **Step 1: Write failing service and RestAssured tests**

Prove:

- a minimal feasible request returns `200`, `FEASIBLE`, and the same request/version;
- an impossible request returns `200`, `INFEASIBLE`, named hard violations, and no assignments eligible for apply;
- malformed input returns `400` before the solver runs;
- solve duration respects the configured upper bound plus a one-second test allowance;
- repeated fixture requests return assignments sorted by demand ID/slot and identical worker selections.

- [ ] **Step 2: Implement bounded solving and explanation mapping**

Use one `SolverManager<Schedule, String>` created by Quarkus injection. Await the final best solution within the request boundary; do not expose asynchronous job IDs in v1. Map `ScoreExplanation` constraint matches into Chinese-capable violation codes and involved opaque IDs.

- [ ] **Step 3: Containerize without database access**

The scheduler service must:

- listen only on the internal Compose backend network and publish no host port;
- receive no OpenProject/PostgreSQL secrets;
- mount no OpenProject volumes;
- run as a non-root user;
- include a health check against `/q/health`.

Add `VIDEO_OPERATIONS_SCHEDULER_URL: "http://scheduler:8080"` to the existing `x-video-operations-environment` map in the override, then add the service below. Because the pinned upstream Compose file is first, relative build paths resolve from `vendor/openproject-docker-compose`:

```yaml
services:
  scheduler:
    build:
      context: ../../scheduler
      dockerfile: Dockerfile
    image: video-operations/scheduler:2.6.0
    expose:
      - "8080"
    networks:
      - backend
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/q/health"]
      interval: 10s
      timeout: 3s
      retries: 3
```

Do not add `depends_on: db`, a database environment variable, or any volume mount.

- [ ] **Step 4: Verify endpoint and container isolation**

```bash
./scheduler/mvnw -f scheduler/pom.xml verify
./ops/compose.sh build scheduler
./ops/compose.sh up -d scheduler
./ops/compose.sh exec -T scheduler curl -fsS http://localhost:8080/q/health
scheduler_container_id="$(./ops/compose.sh ps -q scheduler)"
docker inspect "$scheduler_container_id" --format '{{json .Config.Env}}'
docker inspect "$scheduler_container_id" --format '{{json .Mounts}}'
```

Expected: tests pass; inspection contains no database URL/password and no database/assets mount.

- [ ] **Step 5: Commit**

```bash
git add scheduler compose/docker-compose.override.yml
git commit -m "feat: expose bounded stateless schedule solver"
```

### Task 6: Build and store versioned proposals in the plugin

**Files:**
- Create: `openproject-video-operations/db/migrate/20260902090400_create_video_operations_schedule_proposals.rb`
- Create: `openproject-video-operations/app/models/video_operations/schedule_proposal.rb`
- Create: `openproject-video-operations/app/services/video_operations/scheduling/snapshot_builder.rb`
- Create: `openproject-video-operations/app/services/video_operations/scheduling/proposal_client.rb`
- Create: `openproject-video-operations/app/services/video_operations/scheduling/proposal_create.rb`
- Create: `openproject-video-operations/app/controllers/video_operations/api/v1/schedule_proposals_controller.rb`
- Create: `openproject-video-operations/spec/services/video_operations/scheduling/snapshot_builder_spec.rb`
- Create: `openproject-video-operations/spec/services/video_operations/scheduling/proposal_client_spec.rb`
- Create: `openproject-video-operations/spec/services/video_operations/scheduling/proposal_create_spec.rb`
- Create: `openproject-video-operations/spec/requests/video_operations/api/v1/schedule_proposals_spec.rb`

**Schema:**

- `schedule_proposals`: `project_id`, unique `request_id`, `data_version`, `status` (`feasible`, `infeasible`, `applied`, `rejected`, `stale`), `request_payload` JSONB, `response_payload` JSONB, `score`, `created_by_id`, nullable `applied_by_id`, nullable `applied_at`, timestamps.
- Payload rows are immutable except for the explicit lifecycle fields.

**Interfaces:**
- `SnapshotBuilder.call(project:) -> Hash` locks no rows and returns a consistent read with the current `ProjectState#scheduling_data_version` as `dataVersion`.
- `ProposalClient.solve(snapshot:) -> Hash` uses a two-second connect timeout and a twenty-second total timeout.
- `ProposalCreate.call(project:, actor:, options:) -> ScheduleProposal` records both request and response for audit.
- `POST /video_operations/api/v1/projects/:project_id/schedule_proposals` creates a proposal.
- `GET /video_operations/api/v1/projects/:project_id/schedule_proposals?latest=1` returns the newest stored request status/result, allowing a closed or reloaded browser page to recover the last completed solve.

- [ ] **Step 1: Write failing snapshot contract tests**

Prove the builder includes only active crew, effective rates, availability, skills, open demands, locked/current assignments, travel facts, stage readiness, current alternatives, and the exact project scheduling version. Assert phone numbers and free-text private notes are excluded.

- [ ] **Step 2: Write failing HTTP client tests with WebMock**

Cover success, 400 validation, timeout, connection refusal, malformed response, request/version mismatch, and solver `INFEASIBLE`. The client may retry one connection failure before any response; it must not retry timeouts or application errors. The request spec must also prove latest-result retrieval is project-scoped, authorized, and returns the stored terminal response after a simulated browser reload.

- [ ] **Step 3: Implement builder, client, persistence, and API**

Store the exact serialized snapshot before the request is sent, then the exact validated response. A failed request creates no proposal record. Enforce request ID uniqueness to prevent duplicate submissions.

- [ ] **Step 4: Run focused specs**

```bash
./ops/compose.sh run --rm web bundle exec rails db:migrate
./ops/compose.sh run --rm web bundle exec rspec \
  openproject-video-operations/spec/services/video_operations/scheduling \
  openproject-video-operations/spec/requests/video_operations/api/v1/schedule_proposals_spec.rb
```

- [ ] **Step 5: Commit**

```bash
git add openproject-video-operations
git commit -m "feat: create auditable schedule proposals"
```

### Task 7: Apply or reject the whole proposal transactionally

**Files:**
- Create: `openproject-video-operations/app/services/video_operations/scheduling/proposal_apply.rb`
- Create: `openproject-video-operations/spec/services/video_operations/scheduling/proposal_apply_spec.rb`
- Modify: `openproject-video-operations/app/controllers/video_operations/api/v1/schedule_proposals_controller.rb`
- Modify: `openproject-video-operations/spec/requests/video_operations/api/v1/schedule_proposals_spec.rb`

**Interfaces:**
- `ProposalApply.call(proposal:, actor:, expected_data_version:) -> Result`.
- `POST /video_operations/api/v1/schedule_proposals/:id/apply` applies every returned assignment.
- `POST /video_operations/api/v1/schedule_proposals/:id/reject` records rejection without changing assignments.
- Apply returns `409` if proposal, request, response, expected, or current project data versions differ.
- Apply returns `422` for `INFEASIBLE`, already rejected, missing effective rate, or any domain validation failure.
- Repeated apply of an already applied proposal by the same endpoint is idempotent and returns the original result.

- [ ] **Step 1: Write failing transaction tests**

Cover exact behaviors:

- a matching feasible proposal replaces only solver-owned future assignments in its project;
- manual and locked assignments remain untouched;
- each confirmed new assignment snapshots its effective rate;
- data changed after solve marks proposal `stale` and writes nothing;
- a missing rate on the third row rolls back the first two rows;
- an exception while incrementing the version rolls back assignments and proposal state;
- a proposal cannot apply to a different project;
- a second apply does not duplicate assignments;
- reject changes only proposal status.

- [ ] **Step 2: Implement one locked database transaction**

Lock the project workflow/scheduling state and proposal row, re-evaluate proposal eligibility, validate every proposed row before destructive replacement, then perform the complete replacement, snapshot rates, update proposal state, and increment scheduling version. Use `delete_all` only on the explicit relation of future `source: solver` assignments for that one project; never delete manual, import, completed, or unrelated rows.

- [ ] **Step 3: Run service and request specs**

```bash
./ops/compose.sh run --rm web bundle exec rspec \
  openproject-video-operations/spec/services/video_operations/scheduling/proposal_apply_spec.rb \
  openproject-video-operations/spec/requests/video_operations/api/v1/schedule_proposals_spec.rb
```

- [ ] **Step 4: Run both language suites and end-to-end API smoke**

```bash
./scheduler/mvnw -f scheduler/pom.xml verify
./ops/compose.sh run --rm web bundle exec rspec openproject-video-operations/spec
./ops/compose.sh up -d scheduler web worker
./ops/compose.sh exec -T scheduler curl -fsS http://localhost:8080/q/health
curl -fsS http://localhost:8080/video_operations/api/v1/health
```

- [ ] **Step 5: Commit**

```bash
git add openproject-video-operations README.md
git commit -m "feat: apply schedule proposals atomically"
git status --short
```

Expected: clean working tree.

## Package Completion Criteria

- Scheduler dependency tree resolves only Timefold `2.6.0`; Java and Quarkus pins match the roadmap.
- JSON request/response fixtures lock the versioned contract.
- Every named hard and soft constraint has an isolated score test.
- Solve requests terminate within the configured bound and yield deterministic, ordered proposals.
- The scheduler has no database credentials, database mount, or write path.
- OpenProject stores the exact request/response and exposes human-controlled proposal lifecycle endpoints.
- Stale or infeasible proposals cannot apply.
- A valid proposal applies all rows or none, preserves manual/locked work, snapshots rates, and is idempotent.
- Java, Ruby, image, health, and isolation checks all pass.
