---
name: vanilla-spa-architecture
description: >
  Use this skill when building or maintaining a frontend web application that is
  primarily maintained by an AI agent, especially internal dashboards, admin tools,
  authenticated apps, or tools with a small user base. Applies when the user asks
  to scaffold, refactor, or extend a React app — even if they don't explicitly
  mention "SPA", "Next.js", or "architecture". Also use when deciding between
  Next.js (or any SSR framework) and a plain React + API setup for AI-maintained
  codebases.
---

# Vanilla SPA Architecture for AI-Maintained Codebases

## Why this approach exists

Next.js and similar SSR frameworks introduce layered abstractions (server vs. client
components, file-based routing conventions, framework-coupled auth, build-time DB
dependencies) that AI agents must understand and maintain correctly on every single
edit. For **authenticated internal apps with no SEO requirements**, these abstractions
provide zero user-facing value and significant agent friction.

A **React SPA + standard HTTP API** architecture removes this overhead entirely:
every component is a standard React component, every endpoint is a standard HTTP
handler, and every deployment is a static file upload. The agent writes less
boilerplate and makes fewer framework-specific mistakes.

---

## ✅ DO — Preferred patterns

### Frontend (SPA)

- **Vite + React** for the SPA build. No Next.js, Remix, or similar SSR frameworks.
- **React Router v7** (or similar) for client-side routing — standard `<Route>` config,
  not file-based conventions.
- **Standard React components** — no `"use client"` / `"use server"` directives, no
  server/client distinction. Every component is just a component.
- **`fetch()` from the browser** to call the API. Keep a single `apiFetch(url, opts)`
  wrapper that attaches the auth token and handles errors uniformly.
- **MSAL.js** (or `@auth/core`) for authentication in the SPA — standard browser-side
  auth, no framework-coupled session library.
- **Tailwind + shadcn/ui** for UI — works identically in any React setup.

### API (Backend)

- **Azure Functions** (HTTP triggers) or **Express / Hono** for API endpoints.
  Standard `async function handler(req, res)` — no Next.js route format.
- **Timer triggers** (Azure Functions) or a standalone worker process for background
  jobs (cron, polling). Never inside the web process.
- **JWT middleware** for API auth — validate tokens on every endpoint, stateless,
  no session store needed.
- **One API** shared by all consumers (SPA, Power BI, external services, etc.).

### Folder structure (reference)

```
spa/              ← Vite + React project
  src/
    components/   ← All UI components (plain React)
    pages/        ← Route-level components (imported in router config)
    api/          ← apiFetch wrapper + typed API helpers
    auth/         ← MSAL.js config and hooks
  vite.config.ts
  index.html

api/              ← Azure Functions or Express/Hono
  src/
    functions/    ← One file per HTTP endpoint
    timers/       ← Background jobs (timer triggers)
    services/     ← Business logic (unchanged from any previous setup)
    repositories/ ← Data access (unchanged)
    middleware/   ← JWT auth, error handling
```

---

## ❌ DON'T — Anti-patterns for AI-maintained codebases

| Anti-pattern | Why it hurts AI maintenance |
|---|---|
| `"use client"` / `"use server"` directives | Agent must track server/client boundary on every file; easy to break silently |
| `export const dynamic = "force-dynamic"` | Boilerplate that opts out of the framework's main feature — net-zero value, pure noise |
| File-based routing (`app/`, `page.tsx`, `layout.tsx`) | Agent must know framework routing conventions; a config file is explicit and readable |
| `next/image`, `next/link`, `next/font` | Framework-coupled wrappers around standard HTML — agent must remember which import to use |
| NextAuth / Auth.js in Next.js mode | Beta library with complex JWT→session callback chain; session is framework-coupled |
| Background jobs via `instrumentation.ts` | Jobs are invisible — no independent health check, restart couples to UI deploys |
| `next build` requiring DB at build time | Makes CI fragile; static SPA builds with zero runtime dependencies |
| Server Components fetching data and passing as props | Duplicates API routes; agent must maintain two data paths for the same data |
| Parallel routes, intercepting routes, route groups | Almost never needed; adds nav complexity the agent must learn |
| `generateStaticParams`, `generateMetadata` | Not applicable to authenticated internal apps |

---

## Standard patterns the agent should follow

### Auth token in every API call

```typescript
// api/apiFetch.ts
export async function apiFetch<T>(url: string, opts?: RequestInit): Promise<T> {
  const token = await getAccessToken(); // MSAL silently refreshes
  const res = await fetch(`/api${url}`, {
    ...opts,
    headers: { Authorization: `Bearer ${token}`, "Content-Type": "application/json", ...opts?.headers },
  });
  if (!res.ok) throw new Error(`API error ${res.status}: ${await res.text()}`);
  return res.json();
}
```

### Data fetching in components

```typescript
// Standard pattern — no server components, no getServerSideProps
export function RevenueCard() {
  const { data, isLoading } = useQuery({ queryKey: ["revenue"], queryFn: () => apiFetch("/revenue") });
  if (isLoading) return <Skeleton />;
  return <Card>{data.total}</Card>;
}
```

### API endpoint (Azure Function HTTP trigger)

```typescript
// api/src/functions/revenue.ts
export async function getRevenue(req: HttpRequest, context: InvocationContext): Promise<HttpResponseInit> {
  const user = await verifyJwt(req); // standard JWT middleware
  const data = await revenueService.get(user.company);
  return { jsonBody: data };
}
app.http("revenue", { methods: ["GET"], authLevel: "anonymous", handler: getRevenue });
```

### Background job (Azure Function timer trigger)

```typescript
// api/src/timers/auto-sync.ts — independent of the web process
export async function autoSync(timer: Timer, context: InvocationContext): Promise<void> {
  await syncService.run();
}
app.timer("auto-sync", { schedule: "0 */6 * * *", handler: autoSync });
```

### Client-side routing config

```typescript
// spa/src/main.tsx — explicit, readable, no file conventions to memorize
const router = createBrowserRouter([
  { path: "/", element: <Dashboard /> },
  { path: "/revenue", element: <RevenuePage /> },
  { path: "/admin", element: <AdminPage /> },
]);
```

---

## Deployment targets (recommended free/low-cost)

| Layer | Service | Cost |
|---|---|---|
| SPA | Azure Static Web Apps | Free tier |
| API | Azure Function App (Flex Consumption) | Free tier (1M exec/mo) |
| Background jobs | Same Function App (timer triggers) | Included |
| Database | Azure SQL Serverless | Free tier |
| **Total** | | **~$0/mo** |

---

## Gotchas

- **CORS**: The SPA (CDN domain) and API (Function App domain) are different origins. Configure CORS on the Function App to allow the SPA origin. In local dev, use Vite's `server.proxy` to avoid CORS entirely.
- **Auth redirect URI**: Register the SPA's production *and* localhost URLs in Azure Entra (or your IdP). MSAL will fail silently if the redirect URI doesn't match exactly.
- **Environment variables**: Vite only exposes `VITE_*` prefixed env vars to the browser. API secrets must stay in the Function App's application settings — never in the SPA build.
- **Static Web App routing**: Add a `staticwebapp.config.json` with `"navigationFallback": { "rewrite": "/index.html" }` so React Router handles deep links correctly.
- **Timer trigger cold start**: Azure Function Consumption plan has cold starts. For jobs that must run exactly on schedule, use Premium or Flex Consumption. For best-effort sync jobs, Consumption is fine.
- **`apiFetch` base URL**: In dev, Vite proxies `/api` to the local Function host. In production, set `VITE_API_BASE_URL` to the Function App URL and use it as the prefix in `apiFetch`.
