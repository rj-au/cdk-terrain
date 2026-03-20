# Research: What checkpoint.ts does and where it is used

**Date**: 2026-03-20
**Context**: Need to understand the full blast radius before removing HashiCorp telemetry endpoint (Issue #48)
**Time-box**: 15 minutes

## Question

What does `packages/@cdktn/commons/src/checkpoint.ts` do, what are all its exports, and where is each export used across the codebase?

## Findings

### Finding 1: checkpoint.ts has two distinct responsibilities

The file serves two purposes:

1. **HashiCorp checkpoint telemetry** (TO REMOVE): Sends usage data to `https://checkpoint-api.hashicorp.com/v1/telemetry/cdktn` via HTTP POST. This includes command name, language, OS, arch, CI info, user ID, project ID, and arbitrary payload data.

2. **ID generation utilities** (TO KEEP): `getUserId()` and `getProjectId()` generate/persist UUIDs for identifying users and projects. These are used by Sentry error reporting.

**Exports:**
| Export | Type | Used by HashiCorp telemetry | Used by Sentry | Action |
|--------|------|---------------------------|----------------|--------|
| `sendTelemetry()` | function | Yes - wraps ReportRequest | No | Remove |
| `ReportRequest()` | function | Yes - sends HTTP POST to HashiCorp | No | Remove |
| `ReportParams` | interface | Yes - shapes telemetry payload | No | Remove |
| `getUserId()` | function | Yes - enriches telemetry | Yes - Sentry scope tagging | Keep |
| `getProjectId()` | function | Yes - enriches telemetry | Yes - Sentry scope tagging | Keep |

**Internal (non-exported):**
| Symbol | Purpose | Action |
|--------|---------|--------|
| `BASE_URL` | HashiCorp endpoint URL | Remove |
| `post()` | HTTP POST helper | Remove |
| `getId()` | Shared UUID helper for getUserId/getProjectId | Keep |
| `homeDir()` | Resolves ~/.cdktf path | Keep |

### Finding 2: sendTelemetry has 7 call sites across 5 files

All call sites send usage analytics to HashiCorp. None are related to Sentry.

| File | Line | Command | What it reports |
|------|------|---------|-----------------|
| `cli-core/src/lib/watch.ts` | 184 | `"watch"` | Event: start |
| `cli-core/src/lib/synth-stack.ts` | 279 | `"synth"` | Language, timing, stack metadata, providers |
| `cli-core/src/lib/synth-stack.ts` | 294 | `"synth"` | Error flag |
| `cli-core/src/lib/cdktf-project.ts` | 648 | varies | Language + command-specific payload |
| `cdktn-cli/src/bin/cmds/handlers.ts` | 169 | `"convert"` | Conversion stats |
| `cdktn-cli/src/bin/cmds/helper/init.ts` | 285 | `"init"` | Template, language, providers |
| `cdktn-cli/src/bin/cmds/ui/get.tsx` | 58 | `"get"` | Provider get payload |

### Finding 3: errors.ts has a hidden HashiCorp telemetry path

`packages/@cdktn/commons/src/errors.ts` imports `ReportRequest` and calls it via a private `report()` function. Every time `Errors.Internal()`, `Errors.External()`, or `Errors.Usage()` is called, it sends error details to HashiCorp's endpoint **in addition to** Sentry (which is handled separately via `Sentry.configureScope` in the same file).

This is a **fire-and-forget** call — the `report()` result is not awaited in `reportPrefixedError`, so the error creation doesn't depend on the telemetry succeeding. This means removing `report()` won't change error behavior.

### Finding 4: getUserId and getProjectId are shared with Sentry

`packages/@cdktn/cli-core/src/lib/error-reporting.ts` (lines 120-125):
```typescript
Sentry.configureScope(function (scope) {
    scope.setUser({ id: getUserId() });
    scope.setTag("projectId", getProjectId());
});
```

These functions MUST be preserved. They currently live in `checkpoint.ts` but should be relocated to a more appropriate module (e.g., a new `identity.ts` or kept in a stripped-down `checkpoint.ts`).

### Finding 5: CHECKPOINT_DISABLE is set in 14 CI workflow locations

All GitHub Actions workflows set `CHECKPOINT_DISABLE: "1"` as a global env var. Affected files:
- `build.yml`, `unit.yml`, `examples.yml`, `pr-depcheck.yml`
- `integration.yml` (3 locations)
- `provider-integration.yml` (3 locations)
- `release.yml`, `release_next.yml`
- `registry-docs-pr-based.yml` (2 locations)

These can be cleaned up after removal, but are low priority since setting an unused env var is harmless.

### Finding 6: Dependency analysis

| Dependency | Used in checkpoint.ts | Used elsewhere | Action |
|------------|----------------------|----------------|--------|
| `uuid` | `getId()` (line 104), `ReportRequest()` (line 160) | `init.ts` (line 143 for project ID generation) | Keep — used by `getId()` which is preserved, and by `init.ts` |
| `ci-info` | `ReportRequest()` (line 175) | `error-reporting.ts` (line 13, for Sentry CI detection) | Keep — used by Sentry |
| `https` (node built-in) | `post()` function | N/A | No action needed |

### Finding 7: Test file is entirely HashiCorp-specific

`packages/@cdktn/cli-core/src/test/checkpoint.test.ts` tests only `ReportRequest` against the HashiCorp endpoint using `nock`. The entire file can be deleted.

## Decisions

- **Decision 1**: Split `checkpoint.ts` — extract `getUserId()`, `getProjectId()`, `getId()`, and `homeDir()` into a preserved module. Remove everything else (`sendTelemetry`, `ReportRequest`, `ReportParams`, `post`, `BASE_URL`).
- **Decision 2**: Remove the `report()` function and `ReportRequest`/`ReportParams` imports from `errors.ts`. The `Errors` object and its Sentry integration remain unchanged.
- **Decision 3**: Keep `uuid` and `ci-info` dependencies — both are used outside the checkpoint system.
- **Decision 4**: Delete `checkpoint.test.ts` entirely — it only tests HashiCorp endpoint communication.
- **Decision 5**: `CHECKPOINT_DISABLE` env var in CI workflows can be cleaned up as a low-priority follow-up. Setting an unused env var is harmless.
- **Decision 6**: `CHECKPOINT_DISABLE` in `environment.ts` should be removed since nothing will reference it after checkpoint removal.

## Recommendations

1. **Refactor checkpoint.ts**: Rename to `identity.ts` (or similar) containing only `getUserId`, `getProjectId`, `getId`, `homeDir`. Update `index.ts` export accordingly.
2. **Remove 7 sendTelemetry call sites** across watch.ts, synth-stack.ts, cdktf-project.ts, handlers.ts, init.ts, get.tsx.
3. **Clean up errors.ts**: Remove `report()` function, `ReportRequest`/`ReportParams` imports. Keep `Errors` object and Sentry integration.
4. **Delete checkpoint.test.ts**.
5. **Remove `CHECKPOINT_DISABLE` from environment.ts**.
6. **Low priority**: Clean up `CHECKPOINT_DISABLE: "1"` from 14 CI workflow env vars.

## References

- Issue: https://github.com/open-constructs/cdk-terrain/issues/48
- Checkpoint.ts: `packages/@cdktn/commons/src/checkpoint.ts`
- Error reporting: `packages/@cdktn/cli-core/src/lib/error-reporting.ts`
- Errors module: `packages/@cdktn/commons/src/errors.ts`
- Environment: `packages/@cdktn/commons/src/environment.ts`
- Checkpoint tests: `packages/@cdktn/cli-core/src/test/checkpoint.test.ts`
