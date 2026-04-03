# Skill: Add a React Feature Module

Use this skill when implementing a new user-facing feature in a frontend workspace.

---

## Steps

### 1. Regenerate API types (if the OpenAPI spec changed)

```bash
npm run generate:types
# output: src/api/types.ts — do not hand-edit this file
```

Run this before writing any component that calls the API. Stale types cause silent type drift.

### 2. Add the MSW handler (`src/mocks/handlers/{feature}.ts`)

Derive the response shape from the generated types — do not invent it.

```typescript
import { http, HttpResponse } from "msw";
import type { paths } from "../../api/types";

type CreateOrderResponse =
  paths["/v1/orders"]["post"]["responses"]["201"]["content"]["application/json"];

export const orderHandlers = [
  http.post("/v1/orders", () => {
    const body: CreateOrderResponse = {
      id: "mock-uuid",
      reference: "ORD-20260404-TEST",
      status: "confirmed",
      // ... all required fields from the type
    };
    return HttpResponse.json(body, { status: 201 });
  }),
];
```

Register in `src/mocks/browser.ts` and `src/mocks/server.ts`.

### 3. Add the API client call (`src/features/{feature}/hooks/use{Feature}.ts`)

```typescript
import createClient from "openapi-fetch";
import type { paths } from "../../api/types";

const client = createClient<paths>({ baseUrl: import.meta.env.VITE_API_URL });

export function usePlaceOrder() {
  const [status, setStatus] = useState<"idle" | "loading" | "success" | "error">("idle");

  async function placeOrder(request: PlaceOrderRequest) {
    setStatus("loading");
    const { data, error } = await client.POST("/v1/orders", { body: request });
    if (error) { setStatus("error"); return; }
    setStatus("success");
    return data;
  }

  return { placeOrder, status };
}
```

Always handle loading, error, and success states — never assume a happy path only.

### 4. Add Zustand store slice (`src/features/{feature}/store.ts`)

Only if state must persist across components or page navigations.
Prefer local `useState` for ephemeral UI state — do not over-store.

```typescript
import { create } from "zustand";

interface OrderStore {
  confirmedOrder: OrderResponse | null;
  setConfirmedOrder: (order: OrderResponse) => void;
  reset: () => void;
}

export const useOrderStore = create<OrderStore>((set) => ({
  confirmedOrder: null,
  setConfirmedOrder: (order) => set({ confirmedOrder: order }),
  reset: () => set({ confirmedOrder: null }),
}));
```

### 5. Build components (`src/features/{feature}/components/`)

One component per file. Export from `src/features/{feature}/index.ts`.

### 6. Wire into routing (`src/App.tsx`)

Add a `<Route>` if this feature introduces a new page.

### 7. Write integration tests (`tests/integration/{feature}.test.tsx`)

Test from the user's perspective — not the implementation.

```typescript
import { render, screen, fireEvent, waitFor } from "@testing-library/react";
import { server } from "../../src/mocks/server";
import { http, HttpResponse } from "msw";
import { CheckoutPage } from "../../src/features/checkout";

test("guest can place an order and sees confirmation", async () => {
  render(<CheckoutPage />);

  fireEvent.change(screen.getByLabelText("Email"), { target: { value: "test@example.com" } });
  fireEvent.click(screen.getByRole("button", { name: "Place Order" }));

  await waitFor(() => {
    expect(screen.getByText("ORD-20260404-TEST")).toBeInTheDocument();
  });
});

test("shows error message when order placement fails", async () => {
  server.use(
    http.post("/v1/orders", () => HttpResponse.json({ code: "STOCK_CONFLICT" }, { status: 409 }))
  );

  render(<CheckoutPage />);
  fireEvent.click(screen.getByRole("button", { name: "Place Order" }));

  await waitFor(() => {
    expect(screen.getByRole("alert")).toHaveTextContent("item is no longer available");
  });
});
```

---

## Checklist before moving on

- [ ] `generate:types` run after any contract change — `src/api/types.ts` is up to date
- [ ] MSW handler written and registered in both `browser.ts` and `server.ts`
- [ ] Loading, error, and success states all handled visibly in the UI
- [ ] Integration tests written with MSW — no module mocking
- [ ] `onUnhandledRequest: "error"` active in test setup — no silent unmocked calls
- [ ] Production build passes (`npm run build`)
