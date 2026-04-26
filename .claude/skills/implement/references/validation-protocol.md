# Validation Protocol — Integration Test Verification

## Identity

You are validating a frontend WP's test suite against real backend services. You verify that existing tests pass without MSW mocks. You do NOT implement new code.

---

## Interaction Model

This protocol is **low-interaction**. Execute the validation procedure and report results.

**Do NOT ask the human for:**
- Permission to start backends
- Guidance on port configuration (read from docker-compose.yml)
- Whether to continue after individual test failures

**DO stop for the human ONLY when:**
- A backend service fails to start after 2 attempts
- The test setup file cannot be modified (permissions or unexpected structure)
- All tests fail (suggests a systemic issue, not individual contract mismatches)

---

## Procedure

### 1. Pre-Flight

1. Read the WP to identify which backend endpoints are consumed
2. Read the FE workspace's test setup file (typically `tests/setup.ts`)
3. Check if the setup file already has the `INTEGRATION` environment guard:
   ```typescript
   if (!process.env.INTEGRATION) {
     beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
     afterEach(() => server.resetHandlers());
     afterAll(() => server.close());
   }
   ```
4. If the guard is missing: add it. This is the **only** code change allowed.
   - Replace the unconditional MSW setup with the conditional version
   - Commit the change: `feat({feature-ID}): add INTEGRATION env guard to test setup`

### 2. Start Backend Services

For each mock dependency workspace:

1. Navigate to the backend workspace directory
2. Read `docker-compose.yml` to identify:
   - Service names and ports
   - Database requirements (postgres, redis)
   - Environment variable requirements
3. Start services: `docker compose up -d --wait`
4. Run database migrations if applicable: `docker compose run --rm app alembic upgrade head`
5. Seed test data if the WP's test preconditions require it (check the WP's "Test Scenarios" section for preconditions)
6. Verify health:
   ```bash
   curl http://localhost:{port}/health   # expect 200
   curl http://localhost:{port}/ready    # expect 200
   ```
7. If a service fails to start: retry once. If it fails again, report the error and skip validation for WPs depending on that service.

### 3. Configure Environment

Set environment variables for the FE test run:

1. Read backend `docker-compose.yml` files to determine host ports
2. Set `INTEGRATION=true`
3. Set API base URLs to point to local docker services:
   ```bash
   export INTEGRATION=true
   export VITE_API_URL=http://localhost:{order-service-port}
   # Add additional URLs as needed per the WP's API client configuration
   ```
4. If the FE workspace has a `.env.test` or similar test env file, verify the URLs match the running services

### 4. Run Tests

```bash
cd workspaces/{fe-workspace-id}
INTEGRATION=true npm test -- --reporter=verbose 2>&1
```

Capture the full output for analysis.

### 5. Analyze Results

Parse test output:

1. Count total tests, passed, failed
2. For each failing test:
   - Extract the TS scenario ID from the test name (e.g., `TS-001-001` from `describe("TS-001-001")`)
   - Identify the specific assertion that failed
   - Determine which API endpoint returned an unexpected response
   - Cross-reference with the WP's "Relevant Contracts" section to identify the contract mismatch
3. Categorize failures:
   - **Contract mismatch**: BE returns a different shape/status than expected
   - **Data mismatch**: test preconditions assume specific seed data that doesn't exist
   - **Connection failure**: FE can't reach the BE service
   - **Timeout**: request takes too long (may indicate missing endpoint)

### 6. Report

Return results in this format:

```
VALIDATION_RESULT_START
wp: {WP-ID}
status: {PASS|FAIL}
tests_total: {N}
tests_passed: {N}
tests_failed: {N}
failures:
  - test: "TS-XXX-YYY: {test description}"
    category: {contract_mismatch|data_mismatch|connection_failure|timeout}
    assertion: "{expected vs actual}"
    endpoint: "{METHOD /path}"
    contract_section: "{relevant contract reference from WP}"
VALIDATION_RESULT_END
```

### 7. Update status.yaml

```
IF all tests PASS:
  → Set phase_4.{WP-ID}.status to "done"
  → Remove mock_dependencies
  → Remove validation_failures if present

IF any tests FAIL:
  → Keep phase_4.{WP-ID}.status as "done_with_mocks"
  → Set validation_failures with the failure details
```

### 8. Teardown

After reporting results:

```bash
cd workspaces/{be-workspace-id}
docker compose down
```

Repeat for each backend workspace that was started.

---

## Guardrails

- Do NOT modify source code (except the INTEGRATION guard in test setup — Step 1.4)
- Do NOT modify test files
- Do NOT modify MSW handlers
- Do NOT modify backend code
- Do NOT modify the WP or any spec files
- If a backend service won't start, report the error — do not attempt to fix it
- Write ONLY your own `phase_4.{WP-ID}` entry in `status.yaml`

---

## Edge Cases

### Multiple FE WPs sharing the same backend dependency

If multiple FE WPs need the same backend service, the orchestrator may launch their validations sequentially or in parallel:
- **Sequential**: each validation starts/stops backends independently (simpler, slower)
- **Parallel**: validation agents share already-running backends (faster, but requires port awareness)

The orchestrator decides. As a validation agent, check if backend services are already running before starting them. If they are, skip startup and go directly to running tests.

### Backend service has no seed data mechanism

If the WP's test preconditions require specific data but the backend has no seed script:
1. Check if the backend has a POST endpoint to create the required data
2. Use `curl` to create test data via the API before running FE tests
3. If neither option works, note it in the report as a `data_mismatch` failure

### Tests that use `server.use()` for per-test overrides

When `INTEGRATION=true`, MSW is completely disabled. Tests that use `server.use()` for error scenarios (e.g., "API returns 500") will skip the MSW override and hit the real backend — which will likely return 200, not 500.

These tests are expected to fail in integration mode. Categorize them as **mock-only tests** in the report — they are valid for unit/integration testing but not for end-to-end validation. The orchestrator should not treat these as contract mismatches.

To identify mock-only tests: they call `server.use()` to override handlers with error responses. In integration mode, these tests verify behavior that can only be triggered by backend failures, not normal operation. Flag them separately:

```
mock_only_tests:
  - test: "TS-001-005: shows error when API fails"
    reason: "Uses server.use() to simulate 500 — cannot trigger against real backend"
```