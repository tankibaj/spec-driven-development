# Frontend Implementation Guide (TypeScript / React)

Reference for the implement skill when executing frontend Work Packages. This guide fills gaps when the WP doesn't specify how to approach a checkpoint. **The WP always takes precedence over this guide.**

Load this file when the target workspace has `type: frontend` in `routes.yaml`.

---

## Project Structure

Follow the structure from `docs/architecture/workspace-bootstrap.md`. Key directories:

```
src/
  main.tsx                    # Entry point
  App.tsx                     # Root component + router
  api/
    client.ts                 # openapi-fetch instance + auth headers
    types.ts                  # Generated from OpenAPI spec (do not hand-edit)
  components/                 # Shared UI components
  features/                   # Feature-scoped modules
    {feature}/
      index.ts
      components/
      hooks/
      store.ts                # Zustand store for feature state
  mocks/
    browser.ts                # MSW browser setup (dev mode)
    server.ts                 # MSW Node setup (test mode)
    handlers/                 # One file per API route group
  types/                      # Shared TypeScript types
tests/
  setup.ts                    # Vitest + Testing Library global setup
  unit/                       # Pure logic and hook tests
  integration/                # Component tests with MSW
```

---

## Per-Checkpoint Procedures

These are fallback procedures. If the WP's Implementation Order specifies steps for a checkpoint, follow the WP. Use these only when the WP is silent on how to approach a checkpoint.

### scaffold

**Goal:** Working project skeleton that starts and renders a shell.

1. Initialize project with `npm create vite@latest . -- --template react-ts`
2. Install dependencies per `workspace-bootstrap.md` toolchain:
   - Core: `react`, `react-dom`, `react-router`, `zustand`, `openapi-fetch`
   - Dev: `vitest`, `@testing-library/react`, `@testing-library/jest-dom`, `@testing-library/user-event`, `msw`, `openapi-typescript`, `typescript`, `@biomejs/biome`
3. Create `biome.json` with project lint/format rules
4. Create `tsconfig.json` with strict mode
5. Create `vite.config.ts` with test setup
6. Create `Dockerfile` (multi-stage: build → nginx) and `nginx.conf`
7. Create `.env.example` with API base URLs
8. Generate OpenAPI types: `npx openapi-typescript docs/api/openapi.json -o src/api/types.ts`
9. Create `src/main.tsx` and `src/App.tsx` with router shell
10. Create `.github/workflows/ci.yml` per `workspace-bootstrap.md`
11. **Verify:** `npm run dev` starts, page renders in browser

```bash
# Verification commands
npm run dev
# Open http://localhost:5173 — should see the shell
npm run typecheck
```

### state

**Goal:** Data layer is ready — stores, API client, routing, hooks.

1. Create API client in `src/api/client.ts` using `openapi-fetch`:
   - Configure base URL from env vars
   - Add auth header injection (if WP specifies authentication)
   - Add `X-Request-ID` header propagation
2. Create Zustand stores in `src/features/{feature}/store.ts`:
   - One store per feature area (e.g., cart, auth, orders)
   - Use `persist` middleware for state that survives page refresh (e.g., cart)
   - Keep stores thin: data + actions, no UI logic
3. Create React Router route definitions in `src/App.tsx`:
   - Map routes to page components (page components can be stubs at this point)
   - Add route guards if the WP specifies auth-protected pages
4. Create custom hooks in `src/features/{feature}/hooks/`:
   - Data fetching hooks that call the API client
   - Store access hooks that read from Zustand
5. **Verify:** stores initialize without errors, routes resolve, API client TypeScript types are correct

```bash
# Verification commands
npm run typecheck
```

**Sub-ordering within state:** API client → stores → routing → hooks. Each layer depends on the previous.

### components

**Goal:** All UI components render correctly with mock data.

1. Create page components in `src/features/{feature}/components/`:
   - One file per page or major UI section
   - Each component uses the hooks from the state checkpoint
2. Create shared components in `src/components/`:
   - Reusable UI elements (buttons, inputs, cards, pagination, etc.)
3. For each data-fetching component, implement all three states:
   - **Loading:** skeleton or spinner while data is being fetched
   - **Error:** error message with retry action (if applicable)
   - **Empty:** informative message when data set is empty (e.g., "No products available")
   - Check the WP's ACs — they typically specify the exact empty/error messages
4. For forms, implement:
   - Field-level validation with error messages per the WP
   - Submit handling that calls the API client
   - Loading state during submission (disable button, show spinner)
5. **Verify:** pages render with hardcoded or mock data, all states (loading, error, empty, populated) are visually correct

```bash
# Verification commands
npm run dev
# Manually check each page renders
npm run typecheck
```

**Sub-ordering within components:** shared components → page layouts → feature pages → forms. Build reusable pieces first.

### integration

**Goal:** Components work end-to-end with MSW intercepting API calls.

1. Create MSW handlers in `src/mocks/handlers/`:
   - One file per API route group (e.g., `products.ts`, `orders.ts`, `auth.ts`)
   - Derive response types from `src/api/types.ts` (generated from OpenAPI spec)
   - Cover both success and error responses per the WP's test scenarios
2. Create MSW server setup:
   - `src/mocks/server.ts` — Node setup for Vitest
   - `src/mocks/browser.ts` — Browser setup for dev mode
3. Wire MSW browser setup into `src/main.tsx` (dev mode only):
   ```typescript
   if (import.meta.env.DEV) {
     const { worker } = await import("./mocks/browser");
     await worker.start({ onUnhandledRequest: "warn" });
   }
   ```
4. Verify full user flows work with MSW:
   - Navigate through pages, interact with forms, verify API calls are intercepted
   - Check that error states render when MSW returns error responses
5. **Verify:** full user flows work in dev mode with MSW, no unhandled request warnings

```bash
# Verification commands
npm run dev
# Walk through user flows — all API calls should be intercepted by MSW
```

### tests

**Goal:** Every TS scenario in the WP has a passing test.

1. Create test setup in `tests/setup.ts`:
   ```typescript
   import { server } from "../src/mocks/server";
   import "@testing-library/jest-dom";

   // When INTEGRATION=true, MSW is disabled — tests hit real backends.
   // Used by `/implement --validate` to verify FE against deployed BE services.
   if (!process.env.INTEGRATION) {
     beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
     afterEach(() => server.resetHandlers());
     afterAll(() => server.close());
   }
   ```

   **Integration mode:** When `INTEGRATION=true` is set (triggered by `/implement --validate`), MSW is not started. Tests run against real backend services via docker compose. The same test suite works in both modes — MSW mocks during development, real services during validation. Tests that use `server.use()` for error simulation will be flagged as mock-only during validation.
2. Read the WP's "Test Scenarios" section — this is your test list
3. Create test files in `tests/integration/` — one file per feature area or AC group
4. Each TS scenario becomes one test function:
   - Name: describe/it blocks referencing the scenario ID:
     `describe("TS-001-001") { it("displays product catalog with name, price, and image") }`
   - Set up MSW handlers for the specific scenario (use `server.use()` for per-test overrides)
   - Render the component (wrap in `MemoryRouter` if it uses routing)
   - Simulate user actions with `userEvent`
   - Assert expected outcomes with `screen` queries and `waitFor`
5. For Zustand store tests, use `renderHook` from Testing Library
6. **Verify:** `npm test` passes with all green

```bash
# Verification commands
npm test -- --reporter=verbose
```

**Counting rule:** If the WP lists N test scenarios, you must have N test functions (or N it-blocks). Count before and after.

### dod

**Goal:** The WP's Definition of Done checklist passes completely.

1. Read the DoD section from the WP — run each check listed there
2. Fix any failures before marking the WP done
3. Common checks (verify only if the WP lists them):
   - `npx biome check .` — zero errors
   - `npx tsc --noEmit` — zero errors
   - `npm test` — all passing
   - MSW handlers cover every API call (`onUnhandledRequest: "error"` catches gaps)
   - `openapi-typescript` types regenerated and committed
   - No secrets in code

```bash
# Common verification commands
npx biome check .
npx tsc --noEmit
npm test
```

---

## Testing Patterns

### Component test with MSW

```typescript
import { render, screen, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { MemoryRouter } from "react-router";
import { http, HttpResponse } from "msw";
import { server } from "../../src/mocks/server";
import { ProductCatalog } from "../../src/features/catalog/components/ProductCatalog";

test("TS-001-001: displays product catalog with name, price, and image", async () => {
  server.use(
    http.get("*/products", () => {
      return HttpResponse.json({
        data: [
          { id: "1", name: "Widget", skus: [{ price_minor: 1999 }], image_url: "/img.jpg" },
        ],
        meta: { total: 1, page: 1, per_page: 20 },
      });
    })
  );

  render(
    <MemoryRouter>
      <ProductCatalog />
    </MemoryRouter>
  );

  await waitFor(() => {
    expect(screen.getByText("Widget")).toBeInTheDocument();
    expect(screen.getByText("$19.99")).toBeInTheDocument();
  });
});
```

### Testing error states

```typescript
test("TS-001-005: shows error when API fails", async () => {
  server.use(
    http.get("*/products", () => {
      return HttpResponse.json({ error: "Internal Server Error" }, { status: 500 });
    })
  );

  render(
    <MemoryRouter>
      <ProductCatalog />
    </MemoryRouter>
  );

  await waitFor(() => {
    expect(screen.getByText(/something went wrong/i)).toBeInTheDocument();
  });
});
```

### Testing Zustand stores

```typescript
import { act, renderHook } from "@testing-library/react";
import { useCartStore } from "../../src/features/cart/store";

test("TS-001-010: adding item to cart increments count", () => {
  const { result } = renderHook(() => useCartStore());

  act(() => {
    result.current.addItem({ skuId: "sku-1", name: "Widget", price: 1999, quantity: 1 });
  });

  expect(result.current.items).toHaveLength(1);
  expect(result.current.totalItems).toBe(1);
});
```

### Testing forms and validation

```typescript
test("TS-001-020: checkout form shows field-level validation errors", async () => {
  const user = userEvent.setup();

  render(
    <MemoryRouter initialEntries={["/checkout"]}>
      <CheckoutForm />
    </MemoryRouter>
  );

  // Submit without filling required fields
  await user.click(screen.getByRole("button", { name: /place order/i }));

  await waitFor(() => {
    expect(screen.getByText(/address line 1 is required/i)).toBeInTheDocument();
    expect(screen.getByText(/postal code is required/i)).toBeInTheDocument();
  });
});
```

---

## Common Pitfalls

| Pitfall | Fix |
|---|---|
| Hand-editing `src/api/types.ts` | Always regenerate from OpenAPI spec: `npm run generate:types`. Commit the result. |
| MSW handlers with hardcoded response shapes | Derive response types from `src/api/types.ts`. TypeScript catches drift. |
| Components that call `fetch` directly | Use the `openapi-fetch` client in `src/api/client.ts`. Centralizes auth headers and base URL. |
| Missing loading/error/empty states | Every data-fetching component needs all three states. The WP's ACs usually specify them. |
| State that doesn't survive page refresh | Use Zustand `persist` middleware for cart and other client-side state the WP requires to persist. |
| Tests that render without a router context | Wrap components in `MemoryRouter` when the component uses `useNavigate`, `useParams`, or `<Link>`. |
| Unhandled MSW requests in tests | Set `onUnhandledRequest: "error"`. Fix every unmocked call — it means a component is hitting an API you didn't account for. |
| Tests that pass individually but fail together | Each test must reset MSW handlers (`afterEach(() => server.resetHandlers())`). Use per-test overrides via `server.use()`. |
| Committing to the spec-hub repo | All implementation code goes in the workspace submodule. The only spec-hub commit is the submodule reference. |
