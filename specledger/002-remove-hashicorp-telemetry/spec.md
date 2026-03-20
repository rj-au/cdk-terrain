# Feature Specification: Replace HashiCorp Telemetry with Sentry Analytics

**Feature Branch**: `002-remove-hashicorp-telemetry`
**Created**: 2026-03-20
**Status**: Draft
**Input**: GitHub Issue #48 - cdktn-cli telemetry uses HashiCorp's endpoint
**Issue**: https://github.com/open-constructs/cdk-terrain/issues/48

## User Scenarios & Testing *(mandatory)*

### User Story 1 - CLI stops sending data to HashiCorp (Priority: P1)

As a cdktn user, I want the CLI to not send any data to HashiCorp's checkpoint API, so that my usage data is not shared with an unrelated third party and the CLI does not depend on HashiCorp infrastructure.

**Why this priority**: This is the core ask of the issue. The cdktn project has forked from cdktf, and the checkpoint module still sends telemetry and error reports to `checkpoint-api.hashicorp.com`. This is a privacy concern and an operational dependency on infrastructure the project does not control.

**Independent Test**: Can be fully tested by running any CLI command (e.g., `cdktn synth`) and verifying no outbound HTTP requests are made to `checkpoint-api.hashicorp.com`. Delivers immediate value by removing the external dependency.

**Acceptance Scenarios**:

1. **Given** a user runs any cdktn CLI command, **When** the command executes, **Then** no HTTP requests are made to `checkpoint-api.hashicorp.com` or any other HashiCorp-owned endpoint.
2. **Given** a user has not set `CHECKPOINT_DISABLE`, **When** they run a cdktn CLI command, **Then** no data is sent to HashiCorp.
3. **Given** a user upgrades from a previous cdktn version, **When** they run CLI commands, **Then** behavior is identical except no HashiCorp calls are made.

---

### User Story 2 - Usage analytics are preserved via Sentry (Priority: P1)

As a project maintainer, I want usage telemetry (which commands are run, which languages are used, synth timing) routed through the project's own Sentry account instead of HashiCorp, so that the project retains data-driven insights for feature prioritization and support without depending on Hashicorp infrastructure.

**Why this priority**: The project has a full Sentry business plan with OSS support that includes metrics features. Deleting analytics entirely would leave the project blind to adoption patterns, language usage, and performance trends. Migrating to Sentry preserves these insights at no additional cost, using infrastructure the project already controls. The existing `@sentry/node@7.120.4` SDK already supports custom metrics (`Sentry.metrics.increment()`, `.distribution()`, etc.) — no SDK upgrade needed.

**Independent Test**: Can be validated in isolation by writing a unit test that mocks `@sentry/node` and verifies `Sentry.metrics.increment()` is called with correct metric names and tags when a CLI command executes. Runs in seconds without a real Sentry DSN.

**Acceptance Scenarios**:

1. **Given** a user has usage telemetry enabled (`sendUsageTelemetry: true` in `cdktf.json`) and `SENTRY_DSN` is set and `CHECKPOINT_DISABLE` is not set, **When** they run `cdktn synth`, **Then** a command invocation metric is emitted to Sentry with tags for command name, language, and environment.
2. **Given** a user has usage telemetry disabled (`sendUsageTelemetry: false` in `cdktf.json`), **When** they run any CLI command, **Then** no usage metrics are sent to Sentry.
3. **Given** a user has `CHECKPOINT_DISABLE` set (regardless of `sendUsageTelemetry` value), **When** they run any CLI command, **Then** no usage metrics are sent to Sentry.
4. **Given** the CLI is running in CI, **When** a command executes with telemetry enabled, **Then** the CI environment is captured as a metric tag.
5. **Given** the `sendTelemetry` function is called, **When** Sentry is not initialized (no DSN), **Then** the metric calls are silent no-ops with no errors.

---

### User Story 3 - Sentry error reporting continues working (Priority: P1)

As a project maintainer, I want the existing Sentry-based error/crash reporting to continue functioning after the HashiCorp endpoint is removed, so that the team retains visibility into production errors.

**Why this priority**: Sentry is the project's own error reporting system and must not be disrupted. The Sentry integration depends on shared utilities (`getUserId`, `getProjectId`) that currently live in the checkpoint module, and the error-handling module currently sends error reports to both Sentry and HashiCorp's checkpoint API. Both coupling points must be handled carefully.

**Independent Test**: Can be tested by triggering a crash-reportable error with `SENTRY_DSN` set and `sendCrashReports: true`, then verifying the error appears in Sentry. Also verify that `getUserId` and `getProjectId` continue to function for Sentry scope tagging.

**Acceptance Scenarios**:

1. **Given** a user has `sendCrashReports: true` in `cdktf.json` and `SENTRY_DSN` is set, **When** an internal error occurs, **Then** the error is reported to Sentry.
2. **Given** the checkpoint module's HashiCorp-specific code is removed, **When** Sentry initializes, **Then** it can still obtain `userId` and `projectId` for scope tagging.
3. **Given** the error-handling module previously sent reports to both Sentry and HashiCorp, **When** an error is created via `Errors.Internal/External/Usage`, **Then** only the Sentry path remains active.

---

### User Story 4 - HashiCorp checkpoint code is cleaned up (Priority: P2)

As a project maintainer, I want all HashiCorp checkpoint-specific code removed from the codebase, so that there is no dead code, no confusion about what telemetry is active, and reduced maintenance burden.

**Why this priority**: Once the HashiCorp endpoint is replaced by Sentry (P1), cleaning up the remaining dead code is a follow-on concern. This includes removing the `ReportRequest` function, the `post()` helper, the `BASE_URL` constant, the `report()` function in errors.ts, and the checkpoint-specific tests.

**Independent Test**: Can be tested by searching the codebase for references to `checkpoint-api.hashicorp.com`, `ReportRequest`, and the `post()` function. Delivers value by reducing code complexity.

**Acceptance Scenarios**:

1. **Given** the cleanup is complete, **When** a maintainer searches the codebase for HashiCorp checkpoint references, **Then** no active code references remain (copyright headers are acceptable).
2. **Given** the checkpoint code is removed, **When** the project is built, **Then** all builds succeed without errors.
3. **Given** the `report()` function in errors.ts is removed, **When** `Errors.Internal/External/Usage` is called, **Then** no errors occur and Sentry error tracking still functions.

---

### Edge Cases

- What happens when existing users have `CHECKPOINT_DISABLE` set in their environment? It continues to disable usage telemetry (now via Sentry instead of HashiCorp). No behavior change from the user's perspective.
- What happens when `CHECKPOINT_DISABLE` is set but `sendCrashReports: true`? Crash reporting still works — `CHECKPOINT_DISABLE` only controls usage telemetry, not crash reports. These are independent concerns.
- What happens to the `~/.cdktf/config.json` file that stores `userId`? The file and `getUserId()` function must be preserved since Sentry uses `userId` for scope tagging.
- What happens to `projectId` in `cdktf.json`? The field and `getProjectId()` function must be preserved since Sentry uses `projectId` for scope tagging.
- What happens to the `report()` function in errors.ts that sends error telemetry to HashiCorp via `ReportRequest`? It must be removed. Sentry already captures these errors with richer context (stack traces, breadcrumbs, environment info) — the HashiCorp path is redundant.
- What happens if Sentry is not initialized (no DSN, user opted out of both flags)? The `Sentry.metrics.*` calls are silent no-ops — no errors, no data sent.
- What happens to `CHECKPOINT_DISABLE` in CI workflows (set in 14 locations)? These continue to work as before — they disable usage telemetry. No CI workflow changes needed.
- What happens when neither `sendUsageTelemetry` nor `sendCrashReports` is set in `cdktf.json`? The user is prompted on first CLI use (non-CI only), consistent with the existing consent flow. In CI, both default to disabled (no prompt).

## Requirements *(mandatory)*

### Functional Requirements

**Telemetry migration:**

- **FR-001**: The CLI MUST NOT make any outbound HTTP requests to `checkpoint-api.hashicorp.com` or any other HashiCorp-owned endpoint.
- **FR-002**: Usage telemetry (command invocations, language, timing, CI environment) MUST be emitted as Sentry custom metrics instead of being sent to HashiCorp.
- **FR-003**: The Sentry metrics aggregator integration MUST be enabled in `Sentry.init()` to support custom metrics.
- **FR-004**: The existing `sendTelemetry(command, payload)` function signature SHOULD be preserved where practical, with the implementation changed from HTTP POST to HashiCorp to Sentry metric emission.

**Consent model:**

- **FR-005**: A new `sendUsageTelemetry` field MUST be added to `cdktf.json` to independently control usage telemetry, separate from `sendCrashReports` which controls crash reporting.
- **FR-006**: The `CHECKPOINT_DISABLE` environment variable MUST continue to disable usage telemetry when set, overriding `sendUsageTelemetry: true`. It MUST NOT affect crash reporting (`sendCrashReports`).
- **FR-007**: Sentry MUST be initialized if either `sendCrashReports` or `sendUsageTelemetry` is true (and `SENTRY_DSN` is set).
- **FR-008**: When neither `sendUsageTelemetry` nor `sendCrashReports` is set, the user MUST be prompted on first CLI use (non-CI only), consistent with the existing consent flow.

**Sentry preservation:**

- **FR-009**: The Sentry error reporting system MUST continue to function unchanged, including `userId` and `projectId` scope tagging, breadcrumbs, and crash reporting.
- **FR-010**: The `getUserId()` and `getProjectId()` utility functions MUST be preserved and relocated from `checkpoint.ts` to an appropriate module.

**Code cleanup:**

- **FR-011**: The `ReportRequest` function, `post()` helper, `BASE_URL` constant, and `ReportParams` interface MUST be removed.
- **FR-012**: The `report()` function in errors.ts that sends error data to HashiCorp via `ReportRequest` MUST be removed. The `Errors` object and its Sentry integration MUST be preserved.
- **FR-013**: All existing tests MUST continue to pass, with checkpoint-specific tests replaced by Sentry metrics tests.

**Out of scope:**

- **OS-001**: Removing `CHECKPOINT_DISABLE` from CI workflows (14 locations) is explicitly out of scope. The env var continues to function as a usage telemetry override.
- **OS-002**: Renaming `sendCrashReports` to a broader name is out of scope. The two flags remain independent.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Zero outbound HTTP requests to HashiCorp endpoints when running any CLI command.
- **SC-002**: Usage metrics (command name, language, OS, CI environment) appear in Sentry when `sendUsageTelemetry: true` and `CHECKPOINT_DISABLE` is not set.
- **SC-003**: Usage metrics are not sent when `CHECKPOINT_DISABLE` is set, regardless of `sendUsageTelemetry` value.
- **SC-004**: Sentry error reporting functions correctly (errors appear in Sentry when `sendCrashReports: true`), independent of usage telemetry settings.
- **SC-005**: All unit tests pass, including new telemetry tests validating Sentry metric emission and consent gating.
- **SC-006**: All integration tests pass after the migration.
- **SC-007**: No dead code remains related to HashiCorp's checkpoint API.

### Previous work

- **SL-6b54af - Update Sentry telemetry release tag** (001-cdktn-package-rename): Related task that updates Sentry telemetry release tags as part of the rename. Sentry error reporting is preserved and unaffected by this feature.
- **SL-3d9d60 - Add migration telemetry events** (001-cdktn-package-rename): Related task for adding migration telemetry events. This feature replaces the HashiCorp transport with Sentry, so migration telemetry should use the new Sentry-based `sendTelemetry` function.
- **GitHub Issue #48**: The originating issue reporting that cdktn-cli telemetry uses HashiCorp's endpoint.

## Dependencies & Assumptions

### Assumptions

- **Sentry business plan includes metrics**: The project has a full Sentry business plan with OSS support. Custom metrics features are available at no additional cost.
- **`@sentry/node@7.120.4` supports custom metrics**: Verified — the installed SDK has `Sentry.metrics.increment()`, `.distribution()`, `.set()`, `.gauge()` and `metricsAggregatorIntegration()`. No SDK upgrade required.
- **Two independent consent flags**: `sendCrashReports` controls crash/error reporting. `sendUsageTelemetry` controls usage analytics. Both are in `cdktf.json`. Sentry is initialized if either is true.
- **`CHECKPOINT_DISABLE` remains a usage telemetry override**: The env var continues to disable usage telemetry when set, providing backward compatibility for existing CI workflows (14 locations) and users. It does not affect crash reporting.
- **`getUserId` and `getProjectId` are shared utilities**: Used by both the checkpoint system (being removed) and Sentry error reporting (being preserved). Must be retained and relocated from `checkpoint.ts`.
- **`ci-info` is a shared dependency**: Used by both the checkpoint system and Sentry error reporting, so it must not be removed.
- **`uuid` is still needed**: `getUserId()` uses `uuidv4()` and is preserved for Sentry. Also used by `init.ts` for project ID generation.
- **Metrics are silent no-ops when Sentry is not initialized**: If `SENTRY_DSN` is not set or user has opted out of both flags, `Sentry.metrics.*` calls do nothing — no errors, no data sent.

### Research

The following research spikes informed this specification:

- [Checkpoint usage analysis](research/2026-03-20-checkpoint-usage-analysis.md): Complete map of checkpoint.ts exports, 7 sendTelemetry call sites, and dependency analysis.
- [Sentry usage and migration feasibility](research/2026-03-20-sentry-usage-and-checkpoint-migration.md): Sentry metrics capabilities, migration mapping, and decision to route analytics through Sentry.
- [Testing strategy and SDK validation](research/2026-03-20-testing-strategy-and-sentry-sdk-validation.md): Confirmed SDK supports metrics without upgrade, designed isolated validation test approach, and mapped call site conversions.
