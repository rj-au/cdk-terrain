# Research: Testing strategy, Sentry SDK validation, and migration feasibility

**Date**: 2026-03-20
**Context**: Before committing to Option B (migrate analytics to Sentry), we need to confirm the SDK supports metrics, understand the test infrastructure, and design a fast validation loop.
**Time-box**: 25 minutes

## Question

Can we validate Sentry metrics in isolation to get fast feedback on the migration, and what does the testing strategy look like?

## Findings

### Finding 1: The "big if" is resolved — @sentry/node@7.120.4 fully supports custom metrics

The installed SDK (`@sentry/node@7.120.3` resolved from `7.120.4` range) has a complete metrics API:

```typescript
import * as Sentry from "@sentry/node";

// Counter — perfect for command invocation tracking
Sentry.metrics.increment("command.invoked", 1, {
  tags: { command: "synth", language: "typescript" },
});

// Distribution — perfect for synth timing
Sentry.metrics.distribution("synth.duration", totalTimeMs, {
  unit: "millisecond",
  tags: { language: "typescript", ci: "github-actions" },
});

// Set — track unique users/projects
Sentry.metrics.set("unique.projects", projectId, {
  tags: { language: "python" },
});

// Gauge — track current values
Sentry.metrics.gauge("providers.count", 3, {
  tags: { command: "get" },
});
```

**One setup requirement**: Add `metricsAggregatorIntegration()` to `Sentry.init()`:

```typescript
Sentry.init({
  dsn: process.env.SENTRY_DSN,
  integrations: [
    Sentry.metricsAggregatorIntegration(), // enables metrics
  ],
});
```

**API status**: Marked `@experimental` but fully functional. Metrics are bucketed into 10-second intervals and flushed automatically. Max 10,000 metrics in memory.

**Confidence**: High — verified directly in `node_modules/@sentry/core/types/metrics/exports.d.ts`.

### Finding 2: No SDK upgrade needed

The current `@sentry/node@7.120.4` across all 3 packages (`commons`, `cli-core`, `cdktn-cli`) already has everything needed. No version bump, no breaking changes, no risk.

This eliminates the biggest risk identified in the effort estimate. The migration is purely about:
1. Adding `metricsAggregatorIntegration()` to `Sentry.init()` (1 line)
2. Converting 7 `sendTelemetry()` call sites to `Sentry.metrics.*` calls
3. Removing HashiCorp-specific code

### Finding 3: Current test infrastructure and gaps

**What exists:**
| Test | File | What it covers |
|------|------|---------------|
| Checkpoint unit tests | `cli-core/src/test/checkpoint.test.ts` | `ReportRequest` HTTP behavior, `CHECKPOINT_DISABLE` env var |

**What doesn't exist (and doesn't need to):**
- No Sentry mock tests — Sentry is never initialized in tests (no `SENTRY_DSN`)
- No `sendTelemetry()` unit tests — only `ReportRequest` is tested
- No error-reporting.ts tests
- No logger breadcrumb tests

**How telemetry is disabled in tests:**
- Unit tests: `SENTRY_DSN` not set → Sentry init skips. `CHECKPOINT_DISABLE=1` set in CI workflows → checkpoint skips.
- Integration tests: `--enable-crash-reporting=false` flag in `TestDriver.init()`

**Test runner**: Jest 29.7 + ts-jest. No global Sentry mocks. Fast per-package feedback:
```bash
cd packages/@cdktn/commons && yarn jest        # seconds
cd packages/@cdktn/cli-core && yarn jest       # seconds
```

### Finding 4: Isolated validation is straightforward

We can validate Sentry metrics **without running the full CLI** by writing a focused unit test that mocks Sentry:

```typescript
// packages/@cdktn/commons/src/test/telemetry.test.ts
import * as Sentry from "@sentry/node";

jest.mock("@sentry/node", () => ({
  metrics: {
    increment: jest.fn(),
    distribution: jest.fn(),
    set: jest.fn(),
    gauge: jest.fn(),
  },
}));

describe("telemetry", () => {
  it("sends command metric via Sentry", () => {
    // Call the new telemetry function
    sendTelemetry("synth", { language: "typescript" });

    expect(Sentry.metrics.increment).toHaveBeenCalledWith(
      "command.invoked", 1,
      expect.objectContaining({
        tags: expect.objectContaining({ command: "synth", language: "typescript" }),
      })
    );
  });

  it("does not send when telemetry is disabled", () => {
    // Verify consent gating works
    // ...
  });
});
```

**Fast feedback loop**: This test runs in isolation in seconds, doesn't need a real Sentry DSN, and validates the exact API contract.

### Finding 5: The 7 call sites map cleanly to Sentry metrics

Each existing `sendTelemetry` call maps 1:1 to a Sentry metric:

| Call site | Current code | Sentry replacement |
|-----------|-------------|-------------------|
| `watch.ts:184` | `sendTelemetry("watch", { event: "start" })` | `metrics.increment("command.invoked", 1, { tags: { command: "watch" } })` |
| `synth-stack.ts:279` | `sendTelemetry("synth", { totalTime, language, ... })` | `metrics.increment("command.invoked", ...)` + `metrics.distribution("synth.duration", totalTime, ...)` |
| `synth-stack.ts:294` | `sendTelemetry("synth", { error: true })` | `metrics.increment("command.error", 1, { tags: { command: "synth" } })` |
| `cdktf-project.ts:648` | `sendTelemetry(command, { language, ... })` | `metrics.increment("command.invoked", 1, { tags: { command, language } })` |
| `handlers.ts:169` | `sendTelemetry("convert", { ...stats })` | `metrics.increment("command.invoked", 1, { tags: { command: "convert" } })` |
| `init.ts:285` | `sendTelemetry("init", { language, ... })` | `metrics.increment("command.invoked", 1, { tags: { command: "init", language } })` |
| `get.tsx:58` | `sendTelemetry("get", { language, ... })` | `metrics.increment("command.invoked", 1, { tags: { command: "get", language } })` |

The conversion is mechanical. Each call site changes from a function that POSTs JSON to HashiCorp to a function that emits a Sentry metric. The data is the same, only the transport changes.

### Finding 6: Consent flow already covers this

The existing consent gate in `error-reporting.ts` checks:
1. `cdktf.json` → `sendCrashReports` (persisted user choice)
2. `SENTRY_DSN` env var (build-time bake-in)
3. CI detection (skip consent prompt in CI)

Since Sentry metrics go through the same `Sentry.init()` pipeline, they automatically respect the same opt-out. If a user sets `sendCrashReports: false`, Sentry never initializes, and `Sentry.metrics.*` calls become no-ops.

No new consent mechanism needed.

### Finding 7: Recommended test strategy

**Layer 1: Unit tests (fast, isolated — seconds)**
- Mock `@sentry/node` with `jest.mock()`
- Test that the new telemetry function calls `Sentry.metrics.increment()` with correct tags
- Test consent gating (telemetry disabled when Sentry not initialized)
- Test that removed code paths don't exist (no HTTP calls to HashiCorp)
- Location: `packages/@cdktn/commons/src/test/telemetry.test.ts`

**Layer 2: Integration smoke test (medium — minutes)**
- Run `cdktn synth` on a simple project
- Verify no errors, no outbound calls to `checkpoint-api.hashicorp.com`
- Can use `nock.disableNetConnect()` to catch any unexpected outbound calls
- Location: existing integration test suite with `CHECKPOINT_DISABLE` removed

**Layer 3: Manual Sentry dashboard validation (one-time)**
- Build CLI with real `SENTRY_DSN`
- Run a few commands
- Check Sentry dashboard for metrics appearing
- Not automated — done once during development to confirm end-to-end

**What to delete:**
- `packages/@cdktn/cli-core/src/test/checkpoint.test.ts` — tests HashiCorp endpoint behavior that no longer exists

## Decisions

- **Decision 1: Option B is feasible with low risk.** The SDK supports metrics, no upgrade needed, call sites map cleanly, consent flow is reused. The "big if" is resolved.

- **Decision 2: Write a standalone validation test first.** Before touching any call sites, write a unit test that mocks Sentry and validates `Sentry.metrics.increment()` is called correctly. This gives fast feedback in seconds and de-risks the entire migration.

- **Decision 3: The migration is mechanical, not creative.** Each of the 7 call sites is a straightforward substitution. No architectural decisions needed per-site.

- **Decision 4: One PR is viable.** Given the low risk (no SDK upgrade, same consent flow, mechanical substitution), a single PR covering both removal and migration is clean and reviewable. The PR would be small — mostly deletions with a few line changes at call sites.

## Recommendations

1. **Start with the validation test**: Write `telemetry.test.ts` with Sentry mocked. Get green. This proves the API contract in seconds.

2. **Then do the migration in order**:
   a. Add `metricsAggregatorIntegration()` to `Sentry.init()` in `error-reporting.ts`
   b. Create new `sendTelemetry()` wrapper that calls `Sentry.metrics.*`
   c. Update 7 call sites (or keep the same function signature with new internals)
   d. Remove HashiCorp-specific code (`ReportRequest`, `post()`, `BASE_URL`, `report()`)
   e. Relocate `getUserId`/`getProjectId` out of `checkpoint.ts`
   f. Delete `checkpoint.test.ts`

3. **Keep the same `sendTelemetry(command, payload)` function signature** if possible — this minimizes changes at the 7 call sites. Just change the implementation from "POST to HashiCorp" to "emit Sentry metric".

4. **Run existing tests to catch regressions**: `yarn test` across all packages should pass with the only changes being checkpoint.test.ts deletion.

## References

- Sentry metrics API types: `node_modules/@sentry/core/types/metrics/exports.d.ts`
- Sentry metrics integration: `node_modules/@sentry/core/types/metrics/integration.d.ts`
- Checkpoint test: `packages/@cdktn/cli-core/src/test/checkpoint.test.ts`
- Error reporting: `packages/@cdktn/cli-core/src/lib/error-reporting.ts`
- Jest configs: `packages/@cdktn/{commons,cli-core}/jest.config.js`
- Prior research: `specledger/002-remove-hashicorp-telemetry/research/2026-03-20-checkpoint-usage-analysis.md`
- Prior research: `specledger/002-remove-hashicorp-telemetry/research/2026-03-20-sentry-usage-and-checkpoint-migration.md`
