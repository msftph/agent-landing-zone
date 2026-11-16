# ADR-0042: Expose AG-UI over the existing durable runtime

**Status:** Accepted for fork implementation; Azure validation pending<br>
**Date:** 2026-11-16<br>
**Owners:** Agent Landing Zone maintainers

## Context

The maintainer requested an AG-UI endpoint in the msftph forks while preserving
Durable Task Scheduler (DTS) execution. The approved scope includes conversation
and research/review workflow starts, progress, and approval/resume. The maintainer
selected no new message broker: output is the committed final response, not
token-by-token streaming. Existing UI, Responses, and DTS interfaces remain.

Affected repositories are agent-app-orchestrator, agent-landing-zone, and the
central AI-Landing-Zones documentation source. Runtime implementation belongs
in the orchestrator, not the platform repository.

Priorities are durable execution across HTTP disconnects, authenticated
continuation, unchanged defaults, bounded operations, and reproducible pins.
The pinned DurableAIAgent proxy explicitly rejects stream=True. Official AG-UI
hosting around an in-process agent would bypass the durable worker.

## Alternatives considered

### Direct Agent Framework AG-UI hosting

Uses the official agent-framework-ag-ui adapter and supports incremental
agent output, but moves execution into the HTTP process. It does not preserve
DTS recovery or share existing durable checkpoints. This does not meet the
selected execution requirement.

### Durable gateway with Redis token delivery

Preserves DTS and adds worker streaming callbacks plus Redis Streams and a
gateway event bridge. It adds infrastructure, retention, correlation,
reconnection, and provisional-output semantics. The maintainer explicitly
excluded this additional broker from this change.

### Do not change

Preserves existing operation but provides no AG-UI endpoint.

## Decision

Add an opt-in AG-UI gateway to the existing Container Apps worker process.
AG_UI_ENABLED defaults to false and is invalid for the Foundry-hosted mode.
When enabled, the worker also serves internal HTTP ingress on port 8080.
The same deployment owns both components; no extra compute or storage is added.

Use the official ag-ui-protocol event models/encoder and the pinned Durable
Task clients. Do not wrap the local agent with an in-process AG-UI host.
Conversation turns use a small native DTS orchestration that calls the existing
durable agent with a stable session. Its durable instance records each turn's
input/output so HTTP reconnects do not depend on process-local tasks or receipt
delivery windows. This is native orchestration, not a second workflow engine.
The research/review path uses the existing DurableWorkflowClient and graph.

The gateway accepts text deltas, thread/run identifiers, and typed human
decisions. It rejects injected system/tool history, client tools, shared state,
and forwarded properties. It emits AG-UI status, final message, interrupt, and
error events over POST/SSE. RUN_FINISHED success is emitted only after DTS reports
committed completion. Timeouts and disconnected streams do not terminate DTS
execution; clients reconnect with the same identifiers.

Ingress is internal to the Container Apps environment. Authenticate a trusted
frontend/BFF using Entra JWT validation, an explicit audience, tenant, and
allowed caller object IDs. Derive durable identifiers from the authenticated
caller and supplied thread/run, never accept native instance/entity IDs.
Publish admission as AG_UI_AUTH_AUDIENCE and AG_UI_ALLOWED_CALLER_IDS rather
than reviving retired ORCHESTRATOR_* authentication keys. Declare HTTPS egress
to login.microsoftonline.com for tenant signing keys in the application
definition; the shared network lifecycle handles both providers.
The BFF remains responsible for its end-user-to-thread ownership checks;
workload identity is not end-user identity. No credentials enter DTS payloads,
logs, or checkpoints. Existing retrieval/OBO enforcement is unchanged; this
gateway does not invent a delegated identity or make unsupported tools usable.

Use non-replacing orchestration starts and verify input fingerprints on retries.
Do not silently reexecute a completed run or accept a reused run identifier
with changed input. Check current native pending requests and typed decisions
before forwarding human input. Concurrent external writes still require their
own idempotency; no exactly-once business-effect guarantee is introduced.

The platform publishes the flag and endpoint under label agent-lz and supplies
the gateway's nonsecret admission settings. The existing shared composition
controls ingress for both Bicep and Terraform; no provider-specific execution
implementation, new foundation resource, or release publication is needed.

## Consequences

The existing UI does not automatically switch to AG-UI. A trusted AG-UI client
must use the new endpoint and retain its own authorized thread identifiers.
Workflow progress is coarse status, not token or executor-level telemetry.
Attachments, arbitrary frontend tools, and client-controlled model options are
not accepted by this initial interface. Native presentation/citation output
must be mapped explicitly rather than exposing internal tool or SDK payloads.

An enabled gateway and worker share a process and identity. HTTP polling has
bounded RPCs and deadlines; stopping the process leaves execution persisted in
DTS. Availability/scaling of the worker and internal ingress remain operational
requirements. Existing native sessions are not migrated into gateway sessions.

## Adoption and migration

1. Implement and test runtime gateway and optional startup on the orchestrator
   feature branch from upstream develop.
2. Coordinate platform settings/ingress/source binding and central documentation
   on feature branches; do not publish a release or deploy Azure resources.
3. Keep the compatible core/durable pins; use the validated AG-UI protocol pin.
4. Validate both providers and the existing shared lifecycle. Bind the fork
   source revision without claiming a released/validated component combination.
5. Enable only in Container Apps after admission settings and trusted BFF
   ownership handling are configured.

This fork binds the orchestrator repository to
https://github.com/msftph/agent-app-orchestrator.git, development ref
feature/ag-ui-durable-gateway and immutable commit
581a5915b47cf27ace6eaabd562704621cfea984 in both manifest and application
definition. The manifest's tag field is the development checkout ref, not a
released component version. This is not an umbrella release or a claim that
the other released components were Azure-validated with the new worker.
Release preparation must replace the development ref with an approved,
validated component release; no tag or release is created by this change.

Rollback disables the flag and restores ingress-disabled worker operation.
Native DTS state is retained. Restore the previous worker image/configuration
if needed; do not delete the task hub or convert sessions between hosts.

## Compliance verification

Tests must prove JWT/caller rejection before DTS access, scoped identifiers,
input retry conflicts, no replaced completed runs, committed completion before
success, continuation after disconnect, explicit timeout/failure outcomes,
and stale/wrong-request approval rejection. Exercise real pinned SDK wire
shapes, registration, and AG-UI encoding with mocked network boundaries.

Check Python typing/lint, targeted/full runtime tests as risk requires, both
provider contracts, and shared PowerShell/shell lifecycle checks. Local
compilation/mocks do not prove deployed private networking, crash recovery,
Entra admission, or service-side duplicate-start behavior. Those remain explicit
Azure acceptance tests before production use.

### Local implementation evidence

The fork's runtime commit is 581a5915b47cf27ace6eaabd562704621cfea984.
The full runtime pytest suite passed 1,278 tests with one existing skip.
Git Bash was selected for the existing cross-platform shell tests.
Ruff and mypy passed for all 15 native/gateway source files; installed runtime
and development dependencies passed uv pip check.

The shared platform selection/publication/preparation/interface tests passed
206 cases. With the contract-pinned Bicep 0.42.1 compiler available, the native
foundation suite passed all five cases, including compilation of the actual
foundation. Bicep compilation retains existing warnings.
Fresh clones of the exact remote fork runtime and manifest-pinned UI passed
the seven real-source/published-SDK integration checks. This includes the
unchanged UI transport and matching native binding/registration contracts.

Terraform initialized with backend disabled and lockfile read-only,
validated successfully, and passed all 31 compute mock plans, including
explicit internal-port-8080 and disabled-ingress worker cases.
The provider lock file and foundation dependency pins are unchanged.
These checks do not authorize deployment or substitute for live Azure tests.

## Documentation impact

Update the central orchestration/deployment/configuration guidance on a feature
branch. Do not publish unshipped fork behavior into the current product site.
Service READMEs remain short links to that site.
The coordinated documentation is committed on feature/ag-ui-durable-gateway
in the local AI-Landing-Zones checkout at 26791a42. It is not published to
the documentation site's main branch.

## Review trigger

Revisit when token streaming, browser-direct access, additional workflows,
different AG-UI features, new SDK pins, or independently scaled gateway compute
is requested, or if service-side continuation/duplicate-start evidence fails.
