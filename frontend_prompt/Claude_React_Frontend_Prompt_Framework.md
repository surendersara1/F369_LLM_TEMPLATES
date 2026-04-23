# Claude React Frontend Generation Framework

**Version:** 2.0
**Author:** SSS_AGENT / NorthBay Solutions / 369Forecast Elite Capital
**Purpose:** A reusable, Claude-native prompt system for generating production-grade React frontends consistently across any business domain.
**Quality target:** Deployable to real users on first pass, with < 15% revision cycles on Claude's initial output.

---

## Changelog

### v2.0 (2026-04 — hardened from the Research America Inc. audio analytics build)
- **§1 Master Prompt** rewritten into 12 numbered subsections including a §1.12 self-audit checklist Claude must run before returning code.
- **§2 Business Context** expanded with persona matrix, tenancy model, feature flags, environments, observability stack, audit/compliance fields.
- **§5 Quality Gates** expanded from 10 to 15, each with how-to-verify instructions.
- **§6 Reusable Patterns** expanded from 7 to 15 (added: Search+Filter+Facets, Inline Edit Table, Activity Feed, Command Palette, Notification Center, Onboarding Tour, Empty-state CTA Cluster, Sortable Data Table).
- **11 brand-new sections** added based on gaps surfaced during real builds:
  §7 State Management Decision Tree · §8 Data-Fetching Contract · §9 Forms Architecture · §10 Auth & Identity Pattern · §11 Layout Primitives · §12 Loading/Empty/Error Primitives · §13 API Client Architecture · §14 Observability & Telemetry · §15 Feature Flags · §16 Testing Strategy · §17 Performance Budget · §18 Dark Mode Audit Procedure · §19 A11y Audit Procedure · §20 Anti-Patterns Catalog · §21 Dangerous Libraries Blacklist · §22 Escape Hatches · §23 Claude Output Format · §24 Version Matrix · §25 CLAUDE.md Template · §26 PR Template · §27 Scaffolding Protocol.

### v1.0 — initial release.

---

## How To Use This Framework

Three artefacts, three roles:

1. **MASTER SYSTEM PROMPT** (§1) — paste verbatim at the start of every new project conversation with Claude. Defines non-negotiable stack, standards, conventions, and self-audit protocol.
2. **BUSINESS CONTEXT BLOCK** (§2) — fill in once per project. Defines the business, users, data, branding, tenancy, compliance.
3. **FEATURE PROMPTS** (§3) — one per feature. Short, specific, references the system + context.

Paste §1 + §2 as the first message of a new Claude conversation. Then send §3s one at a time. Claude will produce consistent, production-grade code.

For larger projects, also commit a `CLAUDE.md` at the repo root using the template in §25, and set up the scaffolding in §27 before the first feature prompt.

---

## 1. MASTER SYSTEM PROMPT

Copy this entire block verbatim as the first turn of every new project conversation.

```
You are a senior React frontend architect generating production code for a live business application. Follow every rule below without deviation. If a requirement is ambiguous, resolve it in favor of the rest of the codebase and note the assumption in a single line at the end of your output.

1.1  TECH STACK (non-negotiable — do not substitute)
- React 18 with functional components and hooks only. No class components.
- TypeScript strict mode. All props typed. No `any`. No `as` casts except at API boundaries.
- Vite 5 as build tool.
- Tailwind CSS v3 for all styling. No inline styles, no CSS modules, no styled-components, no Emotion.
- shadcn/ui as the base component library (primitives copied into /src/components/ui — NOT an npm package). Use its generator.
- Radix UI primitives underneath shadcn (already installed by shadcn).
- lucide-react for icons (current minor: 0.400+; never install anything older).
- @tanstack/react-query v5 for all server state.
- React Hook Form + Zod + @hookform/resolvers for every form and validation flow.
- react-router-dom v6 for routing with lazy-loaded route elements.
- Zustand for client-only global state — only when React Query + local state are insufficient.
- date-fns for all date formatting and arithmetic. Never moment.js.
- sonner for toast notifications (shadcn-recommended).
- next-themes for dark-mode class strategy.
- recharts for charts. No Chart.js, no D3 unless unavoidable.
- pnpm as package manager.

1.2  FOLDER ARCHITECTURE
/src
  /components
    /ui          — shadcn primitives (button.tsx, card.tsx, dialog.tsx, ...)
    /shared      — cross-feature composite components (UploadWizard, KPICard, ...)
  /features
    /{feature}
      /components  — feature-scoped components
      /hooks       — useX hooks
      /api         — endpoints.ts, types.ts, hooks.ts, mutations.ts
      /types       — local types not exported beyond the feature
      index.ts     — barrel export
  /routes
    /{area}/PageName.tsx   — top-level route components, one per route
  /lib
    utils.ts     — cn() helper, formatters
    http.ts      — typed fetch wrapper + HttpError class
    env.ts       — typed env accessor
  /hooks         — cross-feature hooks (useAuth, useDebounce, useMediaQuery, useKeyboard)
  /stores        — Zustand stores (one per domain)
  /routes.ts     — named route constants (no hardcoded paths anywhere else)
  /providers.tsx — QueryClientProvider, ThemeProvider, RouterProvider composition
  /App.tsx
  /main.tsx
  /index.css

1.3  COMPONENT STANDARDS
- Default-export the main component. Named-export subcomponents.
- Props interface named `{ComponentName}Props`, declared directly above the component.
- One component per file. File name matches component name (PascalCase.tsx).
- Composition over prop drilling. If drilling goes 3+ levels, lift to context or Zustand.
- Every data-fetching component ships three states: loading (Skeleton), empty (EmptyState), error (ErrorState).
- All interactive elements have a discernible accessible name (text, aria-label, or aria-labelledby).
- All interactive non-button-or-anchor elements have both keyboard handler and role.
- No business logic inside components — extract to hooks inside /features/{x}/hooks.
- No direct fetch() in components — always through a React Query hook.

1.4  STYLING RULES
- Mobile-first. Use sm: md: lg: xl: breakpoints exactly as Tailwind defaults.
- All colors come from the HSL CSS-variable token system in index.css. Never hardcode hex except inside index.css itself. Reference tokens with `bg-primary`, `text-foreground`, `border-border`, etc.
- Dark mode via `dark:` variants. Every color decision must work in both themes. Test with a theme toggle.
- Spacing: use Tailwind defaults (1,2,3,4,6,8,10,12,16,20,24). No arbitrary `[13px]`.
- Typography: text-xs/sm/base/lg/xl/2xl/3xl. No custom font sizes.
- Shadows: shadow-sm/md/lg/xl. Avoid shadow-2xl unless it's a modal overlay.
- Rounded: rounded (default) / rounded-md / rounded-lg / rounded-xl / rounded-full. Match the project radius token.
- Always use `cn()` helper (clsx + tailwind-merge) when conditionally composing class names.

1.5  ACCESSIBILITY (WCAG 2.1 AA — non-negotiable)
- Every form input has an associated `<Label>` with matching `htmlFor`.
- Every button has discernible text or aria-label.
- Focus-visible ring on every interactive element (the shadcn primitives handle this — do not remove).
- Color contrast ≥ 4.5:1 for text, ≥ 3:1 for large text and UI components.
- Semantic HTML: nav, main, section, article, aside, footer, header, h1-h6 in logical order (exactly one h1 per page).
- Icon-only buttons always include a sr-only or aria-label description.
- Skip-to-content link on page shells.
- Images and media elements include alt text or empty alt for decorative.
- Live regions (aria-live="polite") for toasts and async announcements.

1.6  ERROR HANDLING
- Every async action is wrapped in try/catch OR uses React Query's onError. User-facing toast on failure.
- Every React Query hook returns isLoading, isError, error at minimum. Components render the three states.
- A top-level React Router errorElement renders a graceful fallback. No white screen.
- HttpError from /lib/http.ts classifies errors by status. UI reacts differently to 401 (redirect to /login), 403 (forbidden screen), 404 (not-found), 5xx (retry button).
- Never swallow errors silently. Log to the central logger (assume `logger.error()` exists) and surface to user when actionable.

1.7  PERFORMANCE
- Lazy-load routes with React.lazy + Suspense at the router level.
- Memoize expensive pure computations with useMemo. Memoize callbacks passed to memoized children with useCallback.
- React Query: set appropriate staleTime (default 30_000 for lists, 0 for mutations). Use select to project minimal data.
- Virtualize lists over 100 rows with @tanstack/react-virtual.
- Images: width + height attributes, loading="lazy", decoding="async".
- Code-split heavy dependencies (recharts, wavesurfer, prosemirror) behind a dynamic import in the route file.
- Avoid re-rendering by keeping form state in RHF (uncontrolled) rather than useState per field.

1.8  SECURITY
- Never use dangerouslySetInnerHTML without DOMPurify sanitization; prefer a Markdown renderer.
- Every piece of user input validated through Zod before submit.
- Never log tokens, passwords, or PII. Scrub error payloads.
- Auth tokens in HttpOnly cookies (assume backend handles); if a Bearer is required, store in memory only, never localStorage.
- CSP-compatible: no eval, no inline <script>, no new Function.
- External links use rel="noopener noreferrer" target="_blank".

1.9  CONVENTIONS
- Named route constants in /src/routes.ts. Never hardcode paths in components.
- API endpoints declared in /src/features/{feature}/api/endpoints.ts as `const`. Typed with satisfies.
- Environment variables accessed only through /src/lib/env.ts (typed, throws on missing required vars).
- Dates via date-fns. Display user-friendly (formatDistanceToNow, format). Persist ISO strings.
- Numbers via Intl.NumberFormat. Money via helpers in /lib/utils.ts (formatUSD).
- IDs prefixed by entity: user_xxx, session_xxx, job_xxx (match backend).
- All enums as string literal unions; no TS enums.

1.10  WHAT NOT TO DO
- No Redux, no Redux Toolkit — use Zustand.
- No Axios — use /lib/http.ts (fetch wrapper).
- No CSS-in-JS (Emotion, styled-components) — Tailwind only.
- No material-ui, Ant Design, Chakra, Mantine, Bootstrap — shadcn only.
- No Moment, no Day.js — date-fns only.
- No lodash full import — import specific functions or use native.
- No class components, no HOCs, no render props — hooks only.
- No process.env direct access — use /lib/env.ts.
- No alert() or confirm() — use Dialog primitives.
- No document.querySelector — use refs.
- No untyped props, no `any`, no `@ts-ignore` without a comment explaining why.
- Do not invent placeholder data unless explicitly asked — use realistic domain-specific examples.
- Do not apologize or narrate; output code and brief architectural notes only.

1.11  OUTPUT FORMAT (when producing code)
- Full file contents, not diffs. Always produce complete files.
- File path as the first line of each code block: `// FILE: src/features/foo/Foo.tsx`
- Group multi-file outputs with clear headers. Keep ordering: types → endpoints → hooks → components → route page.
- End with: (a) a one-paragraph architecture note, (b) a DEPENDENCIES block listing new npm packages with exact versions, (c) any ASSUMPTIONS made.
- If producing more than 6 files in one response, split logically and label them (1/3, 2/3, 3/3).

1.12  SELF-AUDIT BEFORE RETURNING CODE — run this checklist mentally on every response
- [ ] Every file has a path comment and imports from the correct paths (@/components/ui/..., @/features/...).
- [ ] No `any`, no implicit any, no untyped props.
- [ ] Every React Query hook declares queryKey, queryFn, and appropriate staleTime.
- [ ] Every form uses React Hook Form + Zod + zodResolver.
- [ ] Every data-fetching component renders Skeleton / EmptyState / ErrorState.
- [ ] Every interactive element has an accessible name and keyboard handler.
- [ ] Dark mode classes (dark:) present on every color utility that needs them.
- [ ] Tokens (bg-primary, text-muted-foreground, border-border) used instead of hex.
- [ ] No banned libraries imported (see §1.10).
- [ ] Route is lazy-loaded.
- [ ] If a new npm dep is added, it is listed in the DEPENDENCIES block with exact version.

If any checklist item fails, fix before sending.
```

---

## 2. BUSINESS CONTEXT BLOCK (fill in per project)

Paste this immediately after the master system prompt, filled out.

```
BUSINESS CONTEXT

2.1  Identity
- Company: [legal name]
- Product name: [customer-facing name of this frontend]
- Repo / codename: [internal name]
- Stakeholders: [exec sponsor, product owner, tech lead]

2.2  Personas (one row per persona)
| Persona       | Role group name     | Primary workflows                                  | Auth scope          |
|---------------|---------------------|---------------------------------------------------|---------------------|
| Admin         | Admin               | Tenants, Users, Cost, Quality, Ops cockpit        | Full                |
| Analyst       | Analyst             | Projects, sessions, transcripts, themes, quotes   | Own tenant          |
| Moderator     | Moderator           | Approve/reject analyst output, edit quotes        | Own tenant          |
| Client viewer | ClientViewer        | Read-only project dashboards                      | Own tenant, read    |

2.3  Core Workflows (numbered; detailed enough that a developer could implement)
1. [Workflow 1 — step-by-step what the user does]
2. [Workflow 2]
3. [Workflow 3]

2.4  Data Domain
- Primary entities: [Poll, Session, Theme, Quote, Project, Tenant, User, Job, AuditEvent, CostLedgerEntry, ...]
- Key relationships: [Project has many Sessions; Session has many Themes and Quotes; Job belongs to Session]
- Sensitive fields: [PII attributes, pre-publication client names, cost numbers for ClientViewer]
- Retention: [e.g. 90-day audio, 7-year audit]

2.5  Branding
- Primary color: [HSL — e.g. 221 83% 53%]
- Secondary color: [HSL]
- Accent color: [HSL]
- Persona ribbon colors: [e.g. Admin=cyan 199 89% 48%, Analyst=amber 27 96% 45%]
- Font: [e.g. Inter UI, JetBrains Mono code, Source Serif 4 for published]
- Logo placement: [top-left of sidebar, 32px height]
- Tone: [e.g. "Authoritative, academic, understated"]
- Radius token: [e.g. 0.5rem]

2.6  Backend API
- Base URL: [https://api.example.com/v1]
- Auth: [Cognito JWT in Authorization header / HttpOnly cookie]
- Response envelope: [{ data, meta, errors } or raw arrays]
- Pagination: [cursor-based with nextCursor / offset-based / none]
- Rate limit headers: [X-RateLimit-Remaining, Retry-After]
- Error contract: [{ code, message, details? } on non-2xx]
- Key endpoints: [list 10-20]

2.7  Tenancy
- Multi-tenant: [yes/no]
- Isolation model: [subdomain / path prefix / header / Cognito custom:tenant_id]
- Tenant-switching UI: [yes — admin only / no]

2.8  Integrations
- File storage: [e.g. S3 with presigned URLs]
- Long-running jobs: [Step Functions, SQS, custom polling]
- Realtime: [WebSockets / SSE / Query refetchInterval]
- Audio/video: [AWS Transcribe + Comprehend PII + Bedrock]
- Auth provider: [Cognito User Pool with 4 groups]
- Email: [SES]
- Analytics: [PostHog / Datadog RUM / nothing]
- Error tracking: [Sentry / CloudWatch RUM]

2.9  Environments & Feature Flags
- Envs: [dev / staging / prod — with per-env base URLs]
- Flags provider: [LaunchDarkly / GrowthBook / env-var-based]
- Known flags at project start: [LIST THEM so Claude knows to gate features]

2.10  Compliance & Audit
- Standards: [SOC 2 / HIPAA / GDPR / ISO 27001]
- Audit events: [every mutation logs user_id, tenant_id, entity, action, at]
- PII handling: [fields that must be masked by default with reveal control]
- Export restrictions: [can ClientViewer export? Admin only?]

2.11  Performance Budget
- LCP: [< 2.5s]
- TTI: [< 3.5s]
- Bundle: [< 300kb gzip per route]
- JS heap: [< 80MB steady state]
- Keyboard input latency: [< 50ms]

2.12  Target Devices
- Primary: [desktop — 1280w and up]
- Secondary: [tablet 768-1280]
- Mobile: [read-only / full / not supported]

2.13  Anti-patterns specific to this product
- [e.g. "No auto-publish — editor review mandatory"]
- [e.g. "No inline PII — always behind reveal control with audit log"]
- [e.g. "No analyst self-serve tenant creation — admin-only"]
```

---

## 3. FEATURE PROMPT TEMPLATE (one per feature)

Keep it short. Claude already has §1 and §2. Just describe the feature.

```
FEATURE: [Short name, e.g. "Session Detail View"]

USER STORY:
As a [role], I want to [action], so that [outcome].

ACCEPTANCE CRITERIA:
- [ ] [Specific behavior 1]
- [ ] [Specific behavior 2]
- [ ] [Edge case]
- [ ] [Empty / loading / error states]
- [ ] [Keyboard / a11y specific]

DATA:
- Input shape: [{ field: type, ... }]
- Output shape: [what's sent to API]
- API endpoint(s): [from §2.6]
- Query key(s): [e.g. ["session", sessionId]]

UI NOTES:
- Layout: [e.g. "Two-column: transcript 60%, sidebar 40%"]
- Primitives: [e.g. "Tabs for Transcript/Summary/Quotes/Themes"]
- Interactions: [e.g. "Click transcript line → jump audio playhead"]
- Shortcuts: [e.g. "Space/K = play, J/L = ±10s, arrows = ±5s"]

ROLES:
- Who can see: [Admin, Analyst]
- Who can mutate: [Analyst (own tenant), Moderator (approve)]

NON-GOALS:
- [What this feature is NOT responsible for]

Produce all files required: endpoints, types, hooks, components, route page, and nav wiring.
```

---

## 4. ITERATION PROTOCOL

When refining what Claude already built, keep changes surgical.

```
ITERATE ON: [Feature name]

CHANGE:
[One specific change. E.g. "Add a 'Save as Draft' button next to 'Submit' that calls POST /polls/drafts instead of /polls."]

CONSTRAINTS:
- Do not restructure existing files unless necessary
- Preserve all existing prop signatures
- If you add a dependency, justify it in one line
- If you rename an export, list every file that needs updating
```

---

## 5. QUALITY GATES (15 — run these after Claude delivers)

Each gate lists the **how-to-verify** so review is mechanical.

1. **TypeScript compiles strict.** `pnpm tsc --noEmit` — zero errors. No `any`, no implicit any, no unused vars, no `@ts-ignore` without comment.
2. **ESLint clean.** `pnpm lint` — zero warnings with react-hooks + typescript-eslint recommended.
3. **Prettier formatted.** `pnpm format --check` passes.
4. **All components have loading / empty / error states.** Open each data-fetching component; grep for `isLoading`, `isError`, and confirm empty-case branch.
5. **All forms validate with Zod.** Grep for `<form` → confirm `useForm({ resolver: zodResolver(...) })`.
6. **Keyboard navigation works.** Tab through every page; every interactive element receives focus with visible ring; Enter/Space activates; Escape closes dialogs.
7. **Lighthouse ≥ 90** on Performance / Accessibility / Best Practices / SEO for the main authenticated route and login.
8. **No console.log, no commented-out code.** `grep -rn "console\." src/` and `grep -rn "^[[:space:]]*//[[:space:]]*<" src/`.
9. **No hardcoded strings** that belong in config, env, or i18n. Grep for URL patterns, known magic strings.
10. **Tests exist for the critical path** when specified. Playwright / Vitest / RTL as per §16.
11. **Dark mode parity.** Toggle dark mode → every screen passes visual sanity. Contrast still ≥ 4.5:1.
12. **Mobile responsive.** Chrome DevTools at 375w / 768w / 1280w — layout holds, no horizontal scroll, no clipped buttons.
13. **Network failure resilience.** DevTools → Throttling: Offline → every screen shows a useful error with retry, not a white screen.
14. **No banned dependencies.** Grep package.json against §21's blacklist.
15. **Bundle budget holds.** `pnpm build` → `dist/assets/*.js` — initial route chunk under the §17 budget.

---

## 6. REUSABLE PATTERNS LIBRARY (15)

Paste into a feature prompt when you need the pattern.

### 6.1 Data Table with server-side pagination, sorting, filtering
```
Use @tanstack/react-table with server-side pagination, sorting, column filters. Table state serialized to URL search params so refresh and share-link preserve state. Column visibility toggle, row selection with bulk actions bar, CSV export. React Query hook with the query key derived from table state.
```

### 6.2 Multi-step Wizard (3–5 steps)
```
Three to five steps in a Dialog or full-page layout. Pill-bar progress indicator at top. Each step owns its RHF state with a step-local Zod schema; combined schema validates on final submit. Back navigation preserves data. Final submit shows a loading spinner, then a success screen with next-action buttons (View, Start another).
```

### 6.3 File Upload (S3 presigned)
```
react-dropzone drag-drop zone. Flow: POST /uploads/presign → PUT directly to S3 with XMLHttpRequest for progress → POST metadata back. Multiple files, per-file progress bars, cancel, retry. Validate file type/size client-side before presign.
```

### 6.4 Real-time Dashboard
```
KPI row (4 cards, 2x2 on mobile). Main chart below (recharts). Side panel with recent activity. React Query refetchInterval=15000 with refetchIntervalInBackground=false. Last-updated timestamp. Pulsing "live" indicator when focused and fresh.
```

### 6.5 Rich Document Editor
```
Tiptap (ProseMirror) with extensions: headings, lists, code blocks, tables, images, footnotes, citations. Auto-save on 2s debounce. Sidebar outline derived from headings. Contextual toolbar (different controls in table vs paragraph). Export Markdown and PDF.
```

### 6.6 Audio / Video Review
```
Custom player: wavesurfer.js for audio waveform, native <video> for video. Timestamp-anchored comments pinned to timeline. Synced transcript panel; clicking a line seeks playback; active line highlighted. Playback speed (0.5, 1, 1.25, 1.5, 2). Keyboard shortcuts: Space/K=play, J/L=±10s, arrows=±5s, 0-9 seek by decile.
```

### 6.7 Role-based Dashboard Shell
```
Top nav (logo, search, notifications, user menu, theme toggle). Left sidebar (feature nav grouped by persona, collapsible to icon rail). Right rail optional per feature. Nav items filtered by useAuth().roles. Active route highlighted with ribbon stripe colored by persona. Mobile: sidebar collapses to Sheet drawer.
```

### 6.8 Comparison / Cross-reference View
```
Two to four items side-by-side. Synchronized scrolling toggle. Diff highlighting for text fields. Sticky column headers. "Add to comparison" CTA on detail views. Export as PDF or shareable URL with state encoded in search params.
```

### 6.9 Search + Filter + Facets
```
Top search bar debounced 300ms. Left rail faceted filters (checkbox group per facet with counts). URL search params hold query + facet state. Results as infinite scroll or paginated list. Clear-all button. Zero-results state with suggestion chips.
```

### 6.10 Inline Edit Table
```
Each cell is read-mode by default. Click a cell → input/select appears; Enter saves, Escape cancels. Optimistic update via React Query mutation with onMutate rollback on error. Row-level toast "Saved" / "Reverted — reason". Disabled cells show tooltip explaining why.
```

### 6.11 Activity Feed
```
Reverse-chronological list of audit events. Each row: avatar, actor, verb phrase, entity link, relative timestamp. Group by day with sticky day headers. Filter by actor / entity type / date range. Infinite scroll with React Query's useInfiniteQuery. Live updates prepend with a "New events" pill if scroll position ≠ top.
```

### 6.12 Command Palette (⌘K)
```
cmdk library inside a Dialog. Triggered by ⌘K / Ctrl+K globally. Sections: Recent, Pages, Actions, Help. Fuzzy search. Keyboard only (no mouse needed). Shows shortcut hints next to each item. Action handlers are registered by feature modules via a provider.
```

### 6.13 Notification Center
```
Bell icon in top nav with unread count badge. Popover panel with tabs: Unread / All. Each notification: icon, title, body snippet, timestamp, action link. Mark-read on click; mark-all-read button. React Query refetches every 30s; integrates with sonner toasts for realtime arrivals.
```

### 6.14 Onboarding Tour
```
First-login tour using react-joyride. 5-8 steps that highlight key UI. Skippable. Stored completion in user preferences (backend) and localStorage fallback. Each step has a title, body, and optional "Show me" CTA that performs a demo action.
```

### 6.15 Empty-state CTA Cluster
```
When a list has zero items: centered illustration or lucide icon, a one-line headline, a two-line supporting description, and 1-2 CTAs (primary action + secondary learn-more). If the user lacks permission to create, show a different variant explaining how to get access.
```

---

## 7. STATE MANAGEMENT DECISION TREE

Choose the right tool for each kind of state. Claude should follow this tree — do not use Redux, do not over-engineer.

```
Is this state derived from the server (cached remote data)?
  YES → React Query (useQuery / useMutation / useInfiniteQuery)
        Key: [resource, ...filters]; set staleTime, not cacheTime-only.

Is it local UI state only used by a single component or its direct children?
  YES → useState / useReducer

Does more than one unrelated component subtree need to read/write it?
  Is it short-lived (modal open, form draft)?
    YES → lift to the nearest common parent, or React Context
  Is it app-wide (current user, theme, tenant, feature flags)?
    YES → Zustand store in /src/stores/{domain}.ts

Is it a URL-shareable filter / page / tab state?
  YES → URL search params via useSearchParams. Never duplicate into useState.

Is it form field state?
  YES → React Hook Form (do NOT useState per field). Zod schema for validation.
```

Rules of thumb:
- **Never** put server data in Zustand. React Query is the cache.
- **Never** put form state in Zustand or Context. RHF owns it.
- **Never** sync state between two hooks (useEffect loop) — pick one source of truth.

---

## 8. DATA-FETCHING CONTRACT

Every feature's /api folder follows the same four-file shape.

```typescript
// FILE: src/features/sessions/api/endpoints.ts
export const sessionEndpoints = {
  list: (projectId: string) => `/projects/${projectId}/sessions`,
  detail: (id: string) => `/sessions/${id}`,
  create: () => `/sessions`,
  approve: (id: string) => `/sessions/${id}/approve`,
} as const;
```

```typescript
// FILE: src/features/sessions/api/types.ts
import { z } from "zod";

export const SessionStatus = z.enum(["uploaded", "transcribing", "ready", "approved", "archived"]);
export type SessionStatus = z.infer<typeof SessionStatus>;

export const Session = z.object({
  id: z.string(),
  project_id: z.string(),
  title: z.string(),
  status: SessionStatus,
  duration_sec: z.number().int().nonnegative(),
  created_at: z.string().datetime(),
});
export type Session = z.infer<typeof Session>;

export const SessionListResponse = z.object({
  data: z.array(Session),
  meta: z.object({ nextCursor: z.string().nullable() }),
});
export type SessionListResponse = z.infer<typeof SessionListResponse>;
```

```typescript
// FILE: src/features/sessions/api/hooks.ts
import { useQuery } from "@tanstack/react-query";
import { http } from "@/lib/http";
import { sessionEndpoints } from "./endpoints";
import { Session, SessionListResponse } from "./types";

export function useSessions(projectId: string) {
  return useQuery({
    queryKey: ["sessions", projectId],
    queryFn: async () => {
      const raw = await http.get(sessionEndpoints.list(projectId));
      return SessionListResponse.parse(raw);
    },
    staleTime: 30_000,
    enabled: Boolean(projectId),
  });
}

export function useSession(id: string) {
  return useQuery({
    queryKey: ["session", id],
    queryFn: async () => Session.parse(await http.get(sessionEndpoints.detail(id))),
    staleTime: 60_000,
    enabled: Boolean(id),
  });
}
```

```typescript
// FILE: src/features/sessions/api/mutations.ts
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { toast } from "sonner";
import { http } from "@/lib/http";
import { sessionEndpoints } from "./endpoints";
import { Session } from "./types";

export function useApproveSession() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: async (id: string) => Session.parse(await http.post(sessionEndpoints.approve(id))),
    onSuccess: (session) => {
      qc.invalidateQueries({ queryKey: ["sessions", session.project_id] });
      qc.setQueryData(["session", session.id], session);
      toast.success("Session approved");
    },
    onError: (e) => toast.error(e instanceof Error ? e.message : "Approval failed"),
  });
}
```

Claude must follow this contract. Never call fetch() from a component. Never skip the Zod parse on response.

---

## 9. FORMS ARCHITECTURE

Every form is RHF + Zod. No exceptions. No controlled useState-per-field.

```typescript
// FILE: src/features/projects/components/CreateProjectForm.tsx
import { zodResolver } from "@hookform/resolvers/zod";
import { useForm } from "react-hook-form";
import { z } from "zod";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Textarea } from "@/components/ui/textarea";
import { useCreateProject } from "../api/mutations";

const schema = z.object({
  name: z.string().min(3, "Name must be at least 3 characters").max(80),
  description: z.string().max(500).optional(),
  client_code: z.string().regex(/^[A-Z0-9_]{2,16}$/, "Uppercase letters, digits, underscore (2-16)"),
});
type FormValues = z.infer<typeof schema>;

export default function CreateProjectForm({ onSuccess }: { onSuccess: () => void }) {
  const createProject = useCreateProject();
  const form = useForm<FormValues>({
    resolver: zodResolver(schema),
    defaultValues: { name: "", description: "", client_code: "" },
  });

  const onSubmit = form.handleSubmit(async (values) => {
    await createProject.mutateAsync(values);
    onSuccess();
  });

  return (
    <form onSubmit={onSubmit} className="space-y-4">
      <div className="space-y-2">
        <Label htmlFor="name">Project name</Label>
        <Input id="name" {...form.register("name")} aria-invalid={!!form.formState.errors.name} />
        {form.formState.errors.name && (
          <p className="text-sm text-destructive">{form.formState.errors.name.message}</p>
        )}
      </div>

      <div className="space-y-2">
        <Label htmlFor="client_code">Client code</Label>
        <Input id="client_code" {...form.register("client_code")} />
        {form.formState.errors.client_code && (
          <p className="text-sm text-destructive">{form.formState.errors.client_code.message}</p>
        )}
      </div>

      <div className="space-y-2">
        <Label htmlFor="description">Description</Label>
        <Textarea id="description" rows={3} {...form.register("description")} />
      </div>

      <Button type="submit" disabled={form.formState.isSubmitting || createProject.isPending}>
        {createProject.isPending ? "Creating…" : "Create project"}
      </Button>
    </form>
  );
}
```

Rules:
- Schema at the top of the file (or shared from types.ts if reused).
- `defaultValues` always set (never leave undefined).
- Error messages live in the schema, not sprinkled in the JSX.
- `disabled` on the submit combines `isSubmitting` + mutation `isPending`.
- `aria-invalid` on the input.

---

## 10. AUTH & IDENTITY PATTERN

Every app has a single `useAuth()` hook backed by a Zustand store. Roles from the JWT. RequireRole wraps protected routes.

```typescript
// FILE: src/stores/auth.ts
import { create } from "zustand";

type Role = "Admin" | "Analyst" | "Moderator" | "ClientViewer";
type User = { id: string; email: string; roles: Role[]; tenant_id: string };
type AuthState = {
  user: User | null;
  setUser: (u: User | null) => void;
  hasRole: (role: Role) => boolean;
  hasAny: (roles: Role[]) => boolean;
};

export const useAuth = create<AuthState>((set, get) => ({
  user: null,
  setUser: (user) => set({ user }),
  hasRole: (role) => !!get().user?.roles.includes(role),
  hasAny: (roles) => !!get().user?.roles.some((r) => roles.includes(r)),
}));
```

```typescript
// FILE: src/components/shared/RequireRole.tsx
import { Navigate, useLocation } from "react-router-dom";
import { useAuth } from "@/stores/auth";

type Role = "Admin" | "Analyst" | "Moderator" | "ClientViewer";

export default function RequireRole({ roles, children }: { roles: Role[]; children: React.ReactNode }) {
  const { user, hasAny } = useAuth();
  const loc = useLocation();
  if (!user) return <Navigate to="/login" state={{ from: loc }} replace />;
  if (!hasAny(roles)) return <Navigate to="/403" replace />;
  return <>{children}</>;
}
```

Rules:
- `RequireRole` wraps every protected route element, not the entire subtree once.
- Login page redirect logic must VERIFY the new user's roles permit `location.state.from.pathname` before honoring it. If they don't, redirect to the persona's default home.
- Never persist tokens in localStorage. Either HttpOnly cookies (preferred) or in-memory.
- Logout clears Zustand AND invalidates all queries (`queryClient.clear()`).

---

## 11. LAYOUT PRIMITIVES

Seven primitives cover ~95% of layout needs. Compose them; don't reach for flex/grid utility chains inline.

```tsx
// /src/components/shared/layout/Stack.tsx   — vertical rhythm with gap
<Stack gap={4}>...</Stack>

// /src/components/shared/layout/Cluster.tsx  — horizontal wrap with gap
<Cluster gap={2}>...</Cluster>

// /src/components/shared/layout/Grid.tsx     — responsive auto-fit grid
<Grid min={280} gap={4}>...</Grid>

// /src/components/shared/layout/Center.tsx   — centers child, max-width optional
<Center max={720}>...</Center>

// /src/components/shared/layout/Sidebar.tsx  — sidebar + content; sidebar collapses at sm
<Sidebar side="left" width={240}>nav</Sidebar>

// /src/components/shared/layout/TwoColumn.tsx — 60/40 or 70/30 responsive two-col
<TwoColumn left={<Transcript/>} right={<Inspector/>} />

// /src/components/shared/layout/Container.tsx — page container with padding + max-w
<Container>...</Container>
```

These wrap Tailwind utilities so features stay terse and consistent. Build them once per project.

---

## 12. LOADING / EMPTY / ERROR PRIMITIVES

Every data-fetching component renders all three states. Make them first-class components, not ad-hoc JSX.

```tsx
// /src/components/shared/SkeletonList.tsx — N rows of Skeleton
<SkeletonList rows={5} height={48} />

// /src/components/shared/EmptyState.tsx
<EmptyState
  icon={<Inbox className="h-6 w-6" />}
  title="No sessions yet"
  description="Upload your first audio file to start transcription."
  action={<Button onClick={() => setOpen(true)}>New session</Button>}
/>

// /src/components/shared/ErrorState.tsx — classifies by HttpError.status
<ErrorState error={error} onRetry={refetch} />
// renders different copy for 401 / 403 / 404 / 429 / 5xx / offline
```

`ErrorState` inspects `error instanceof HttpError && error.status`:
- 401 → "Your session expired. Please sign in again." + Sign in button
- 403 → "You don't have access to this. Ask an admin to grant it."
- 404 → "We couldn't find that." + Back link
- 429 → "Too many requests. Try again in {Retry-After} seconds."
- 5xx / TypeError (network) → "Something went wrong." + Retry button + copy of request id if available

---

## 13. API CLIENT ARCHITECTURE

One typed fetch wrapper. One HttpError class. Every request goes through it.

```typescript
// FILE: src/lib/http.ts
import { env } from "./env";

export class HttpError extends Error {
  constructor(
    public status: number,
    public code: string,
    message: string,
    public requestId?: string,
    public details?: unknown
  ) {
    super(message);
    this.name = "HttpError";
  }
}

type Options = { body?: unknown; signal?: AbortSignal; headers?: Record<string, string> };

async function request(method: string, path: string, opts: Options = {}) {
  const res = await fetch(`${env.API_BASE_URL}${path}`, {
    method,
    credentials: "include",
    headers: {
      "Content-Type": "application/json",
      ...(opts.headers ?? {}),
    },
    body: opts.body !== undefined ? JSON.stringify(opts.body) : undefined,
    signal: opts.signal,
  });

  const requestId = res.headers.get("x-request-id") ?? undefined;

  if (!res.ok) {
    let payload: any = null;
    try { payload = await res.json(); } catch { /* non-JSON error */ }
    throw new HttpError(
      res.status,
      payload?.code ?? `http_${res.status}`,
      payload?.message ?? res.statusText,
      requestId,
      payload?.details
    );
  }

  if (res.status === 204) return null;
  return res.json();
}

export const http = {
  get: (p: string, o?: Options) => request("GET", p, o),
  post: (p: string, body?: unknown, o?: Options) => request("POST", p, { ...o, body }),
  put: (p: string, body?: unknown, o?: Options) => request("PUT", p, { ...o, body }),
  patch: (p: string, body?: unknown, o?: Options) => request("PATCH", p, { ...o, body }),
  delete: (p: string, o?: Options) => request("DELETE", p, o),
};
```

Rules:
- No per-feature fetch calls. Always `http.*`.
- 401 handling: either a React Query global onError at the QueryClient level, or an Axios-style interceptor wrapper. Never scatter.
- Retries for GETs (idempotent) — React Query's default retry: 3 with backoff is fine. Turn retries OFF for mutations.

---

## 14. OBSERVABILITY & TELEMETRY

Every non-trivial app needs four signals from the frontend:

1. **Errors** — Sentry (or CloudWatch RUM). Wire `Sentry.init()` in `main.tsx`. Wrap the router with `<Sentry.ErrorBoundary>`.
2. **Web vitals** — `onCLS, onFCP, onLCP, onTTFB, onINP` from `web-vitals` → POST to backend telemetry endpoint.
3. **Product events** — thin `track(event, props)` wrapper. Fire on: page view, key action submits, key feature clicks. No PII.
4. **Console signal** — a `logger` module with levels; in dev it's console, in prod it's Sentry breadcrumb + no-op console.

Claude should NOT import `posthog-js` or `@sentry/react` directly in feature code. It should use `/src/lib/telemetry.ts` which abstracts the provider.

---

## 15. FEATURE FLAGS

Flags gate incomplete or experimental features. Check a flag, don't hardcode a condition to env.

```typescript
// FILE: src/lib/flags.ts
import { useAuth } from "@/stores/auth";

type Flag = "new_session_detail" | "command_palette" | "realtime_notifications";

const defaults: Record<Flag, boolean> = {
  new_session_detail: false,
  command_palette: true,
  realtime_notifications: false,
};

export function useFlag(name: Flag): boolean {
  const { user } = useAuth();
  const override = user?.flags?.[name];
  return override ?? defaults[name];
}
```

- Branch on `useFlag("x")` in the route / component, not at import time.
- Never ship a flag permanently — remove the old branch within 2 sprints of default-on.
- New features behind a flag by default in prod; default-on in dev.

---

## 16. TESTING STRATEGY (three tiers)

Claude should produce tests when asked. Know which tier.

**Tier 1: Unit — Vitest + React Testing Library**
- Pure functions in /lib (formatters, validators).
- Hooks with QueryClientProvider wrapper.
- One happy-path render + one error render per data-fetching component.

**Tier 2: Component — Vitest + RTL + MSW**
- Full feature component with Mock Service Worker intercepting the HTTP layer.
- Cover: happy path, validation error, network error, empty state.

**Tier 3: E2E — Playwright**
- Critical user journeys only (login → dashboard → complete primary workflow → logout).
- One spec per persona. Do NOT try to test every feature in E2E.
- Run against a dev server with mocked API (MSW) or an ephemeral backend.

Golden rule: **never test implementation**. Test behavior users can observe. Prefer `screen.getByRole("button", { name: /save/i })` over `getByTestId`.

---

## 17. PERFORMANCE BUDGET

Enforce these per route. If Claude's output blows a budget, iterate.

| Metric            | Target     | How to measure                         |
|-------------------|------------|----------------------------------------|
| LCP               | < 2.5s     | Lighthouse / web-vitals in prod        |
| TTI               | < 3.5s     | Lighthouse                              |
| INP               | < 200ms    | web-vitals in prod                      |
| Initial JS (gzip) | < 200kb    | `pnpm build` + inspect dist/            |
| Route chunk       | < 80kb     | Same                                    |
| CSS               | < 30kb     | Same                                    |
| Images above fold | < 200kb    | Network tab                             |
| React re-renders on input | ≤ 3 | React DevTools profiler              |

Common culprits: moment.js (use date-fns), chart.js (use recharts), lodash full (use per-function), ant-design full bundle (don't use), unoptimized SVGs, not lazy-loading routes, not memoizing tables.

---

## 18. DARK MODE AUDIT PROCEDURE

Run this checklist every time a new screen ships.

1. Toggle to dark mode from the UI.
2. For each screen: background, cards, borders, text, icons, hover states, focus rings all visible.
3. Charts: axis labels and grid lines readable; chart colors tested in both themes.
4. Images: if any have white backgrounds, wrap with `dark:invert` or ship dark variants.
5. Screenshots side-by-side (light/dark). No element should be invisible or low-contrast.
6. Contrast checker (axe DevTools) passes in dark mode.

Common failures: plain `border` class without a token, hardcoded hex inside `style={}`, SVGs with fill="#000", chart tooltips with white bg.

---

## 19. A11Y AUDIT PROCEDURE

For each new screen:

1. Run axe DevTools — zero violations.
2. Tab through from the top — every interactive element reached in a logical order; focus ring always visible.
3. Screen reader pass (VoiceOver on Mac or NVDA on Windows): headings announce, labels read with inputs, buttons announce text.
4. Zoom to 200% — layout still usable, no horizontal scroll.
5. Keyboard-only — complete the primary workflow without touching the mouse.
6. Color-blind: use axe or Stark to simulate protan/deutran/tritan — critical info not color-only.

Headings: exactly one h1 per page, logical h2 > h3 > h4 order, no skipped levels.

---

## 20. ANTI-PATTERNS CATALOG

Claude must NEVER produce any of these. Each one should trigger a self-audit failure.

1. **useState for server data.** Use React Query.
2. **useEffect to sync two useStates.** Pick one source of truth.
3. **useState per form field.** Use React Hook Form.
4. **Hardcoded hex colors outside index.css.** Use CSS-var tokens.
5. **Hardcoded route paths.** Use /src/routes.ts.
6. **Direct process.env access.** Use /src/lib/env.ts.
7. **Axios, fetch directly in components.** Use /src/lib/http.ts + React Query.
8. **Unsanitized dangerouslySetInnerHTML.** DOMPurify or Markdown renderer.
9. **Non-semantic divs for interactive elements.** Use button / a / input / select.
10. **Icon-only button without aria-label.** Always label.
11. **Drilling props > 2 levels.** Context or Zustand.
12. **Boolean role checks in JSX (`user.email.includes("admin")`).** Use `hasRole("Admin")`.
13. **useEffect with empty array calling a hook.** That's a side effect in render logic — use an event handler.
14. **localStorage for auth tokens.** HttpOnly cookie or in-memory only.
15. **Class components, HOCs, render props.** Hooks only.
16. **Redux, Redux Toolkit.** Zustand.
17. **CSS-in-JS (Emotion, styled-components).** Tailwind only.
18. **Axios.** fetch wrapper only.
19. **Moment.js.** date-fns only.
20. **`any`, `@ts-ignore`, untyped props.** Fix the types.

---

## 21. DANGEROUS LIBRARIES BLACKLIST

Do not install any of these. Alternatives listed.

| Banned                   | Use instead                         |
|--------------------------|-------------------------------------|
| moment                   | date-fns                            |
| axios                    | fetch wrapper (/lib/http.ts)        |
| redux, @reduxjs/toolkit  | Zustand + React Query               |
| lodash (full)            | per-function imports or native      |
| jquery                   | refs + vanilla DOM                  |
| chart.js                 | recharts                            |
| moment-timezone          | date-fns-tz                         |
| bootstrap                | Tailwind + shadcn                   |
| material-ui / @mui/*     | shadcn                              |
| antd                     | shadcn                              |
| chakra-ui                | shadcn                              |
| styled-components        | Tailwind                            |
| @emotion/*               | Tailwind                            |
| yup                      | Zod                                 |
| formik                   | React Hook Form                     |
| class-validator          | Zod                                 |
| react-hot-toast          | sonner                              |

---

## 22. ESCAPE HATCHES

When the rules don't fit, here's how to escape without breaking the system.

- **Need an external embed (Stripe Checkout, Calendly)?** Allowed. Wrap in a feature-scoped component with an explicit comment `// EXTERNAL EMBED: <vendor>` at the top.
- **Need a non-shadcn primitive (complex date-range picker, rich calendar)?** OK if shadcn has no equivalent. Prefer a headless library (react-day-picker, downshift) over a styled kit.
- **Need CSS that Tailwind can't express?** Add to index.css under `@layer components` as a named class. Never inline `style={}`.
- **Need to bypass TypeScript?** Use `// @ts-expect-error` (not `@ts-ignore`) with a comment explaining why and a tracking issue link.
- **Need to bypass React Query cache?** Use `queryClient.fetchQuery()` for a one-shot; do NOT invent a side cache.

---

## 23. CLAUDE OUTPUT FORMAT CONVENTIONS

When Claude produces a multi-file feature, output should be structured so a reviewer can scan quickly.

```
ARCHITECTURE (3-5 lines describing how the files fit together)

// FILE: src/features/sessions/api/types.ts
(full file)

// FILE: src/features/sessions/api/endpoints.ts
(full file)

// FILE: src/features/sessions/api/hooks.ts
(full file)

// FILE: src/features/sessions/api/mutations.ts
(full file)

// FILE: src/features/sessions/components/SessionCard.tsx
(full file)

// FILE: src/routes/analyst/SessionList.tsx
(full file)

// DEPENDENCIES (new)
- @tanstack/react-virtual ^3.10.0  — for virtualizing the session list

// ASSUMPTIONS
- session.duration_sec is always integer seconds, not ms (matches backend contract in §2.6).
- ClientViewer role does not see the Approve button (§2.13 anti-pattern).
```

If the response would exceed a reasonable length (~8 files), split into sequential messages labelled `(1/3)`, `(2/3)`, `(3/3)`.

---

## 24. VERSION MATRIX

The stack pinned so Claude never suggests incompatible combos.

| Package                        | Version (min)  | Notes                                         |
|--------------------------------|----------------|-----------------------------------------------|
| react, react-dom               | ^18.3.0        | No React 19 features yet                      |
| typescript                     | ^5.5.0         | strict mode                                    |
| vite                           | ^5.4.0         | with @vitejs/plugin-react                      |
| tailwindcss                    | ^3.4.0         | v4 once stable                                 |
| tailwind-merge                 | ^2.5.0         |                                               |
| class-variance-authority       | ^0.7.0         |                                               |
| clsx                           | ^2.1.0         |                                               |
| @tanstack/react-query          | ^5.50.0        |                                               |
| react-hook-form                | ^7.52.0        |                                               |
| @hookform/resolvers            | ^3.9.0         |                                               |
| zod                            | ^3.23.0        |                                               |
| react-router-dom               | ^6.26.0        | Not v7 until migration planned                 |
| zustand                        | ^4.5.0         |                                               |
| lucide-react                   | ^0.440.0       | NEVER install 1.x — that's an unrelated pkg    |
| sonner                         | ^1.5.0         |                                               |
| next-themes                    | ^0.3.0         |                                               |
| date-fns                       | ^3.6.0         |                                               |
| recharts                       | ^2.12.0        |                                               |
| @radix-ui/react-*              | ^1.1.0         | Pulled in by shadcn init                       |
| tailwindcss-animate            | ^1.0.7         |                                               |

Rerun `pnpm outdated` quarterly; bump by minor, test, commit.

---

## 25. CLAUDE.md TEMPLATE (drop at repo root)

```markdown
# {Project Name}

## Stack
React 18 · TypeScript strict · Vite · Tailwind · shadcn/ui · React Query · RHF+Zod · react-router v6 · Zustand · date-fns · sonner · next-themes · recharts · lucide-react.

## Non-negotiables
- Follow /E:/F369_LLM_TEMPLATES/frontend_prompt/Claude_React_Frontend_Prompt_Framework.md (v2.0).
- See §20 Anti-Patterns Catalog and §21 Dangerous Libraries Blacklist — never violate.
- Every data-fetching component: Skeleton / EmptyState / ErrorState.
- Every form: React Hook Form + Zod.

## Folder layout
See framework §1.2.

## Personas & routes
- Admin → /admin/* (Ops, Users, Tenants, Cost, Quality)
- Analyst → /analyst/* (Projects, Sessions, Themes, Quotes)

## Running
pnpm install
pnpm dev            # http://localhost:5173
pnpm test
pnpm build

## Env
.env.local required:
VITE_API_BASE_URL=...
VITE_COGNITO_POOL_ID=...
VITE_COGNITO_CLIENT_ID=...

## Conventions
- New feature: scaffold with /src/features/{name}/{api,components,hooks,types}/index.ts.
- New route: register in /src/routes.ts and lazy-load.
- New shadcn primitive: `pnpm dlx shadcn-ui@latest add <name>`.

## Do NOT
- Do not add Redux, Axios, Moment, MUI, Chakra, styled-components.
- Do not hardcode colors or paths outside index.css / routes.ts.
- Do not put server data in Zustand.
- Do not put form state in useState.
```

---

## 26. PR TEMPLATE

Commit to `.github/pull_request_template.md`.

```markdown
## What
One-sentence description.

## Why
Link to ticket / issue / Slack thread.

## Screens
- Light: <image>
- Dark: <image>
- Mobile: <image>

## Quality gates (§5 of framework)
- [ ] tsc clean
- [ ] lint clean
- [ ] loading/empty/error states rendered
- [ ] forms validated with Zod
- [ ] keyboard nav works
- [ ] Lighthouse ≥ 90 on touched routes
- [ ] dark mode parity
- [ ] no banned deps
- [ ] bundle budget holds

## Tests
- [ ] unit (vitest)
- [ ] component (MSW)
- [ ] e2e (playwright) — if critical path changed

## Follow-ups
- [ ] ...
```

---

## 27. SCAFFOLDING PROTOCOL (before the first feature prompt)

Run these 11 steps once at project start. Do NOT let Claude start a feature on an empty skeleton.

1. `pnpm create vite@latest {name} --template react-ts`
2. `cd {name} && pnpm install`
3. `pnpm add -D tailwindcss@latest postcss autoprefixer tailwindcss-animate`
4. `pnpm dlx tailwindcss init -p` → replace with the project's `tailwind.config.ts` (see §4 of this framework's companion config, or the Research America repo for a known-good example).
5. Replace `src/index.css` with the HSL-token block (see §4 in the framework or Research America repo).
6. `pnpm dlx shadcn@latest init` → New York style, neutral base, CSS variables yes, RSC no.
7. Add primitives: `pnpm dlx shadcn@latest add button card dialog input label select sheet skeleton table tabs textarea toast tooltip avatar badge dropdown-menu progress` (15 primitives covers 90% of needs).
8. Install runtime deps: `pnpm add @tanstack/react-query react-router-dom react-hook-form @hookform/resolvers zod zustand date-fns sonner next-themes recharts lucide-react`
9. Create directory skeleton:
   ```
   src/
     features/.gitkeep
     hooks/.gitkeep
     stores/.gitkeep
     lib/
       utils.ts          # cn() + formatters
       http.ts           # §13
       env.ts            # typed env
       telemetry.ts      # §14
       flags.ts          # §15
     components/shared/.gitkeep
     routes.ts           # empty named exports
     providers.tsx       # QueryClient + Theme + Router composition
   ```
10. Commit `CLAUDE.md` (§25) and `.github/pull_request_template.md` (§26).
11. `pnpm dev` — verify the shell loads with dark-mode toggle before writing any feature.

One-liner bash:
```bash
pnpm create vite@latest app --template react-ts && \
cd app && \
pnpm add @tanstack/react-query react-router-dom react-hook-form @hookform/resolvers zod zustand date-fns sonner next-themes recharts lucide-react && \
pnpm add -D tailwindcss@latest postcss autoprefixer tailwindcss-animate && \
pnpm dlx tailwindcss init -p && \
pnpm dlx shadcn@latest init -y && \
for p in button card dialog input label select sheet skeleton table tabs textarea toast tooltip avatar badge dropdown-menu progress; do pnpm dlx shadcn@latest add $p -y; done && \
mkdir -p src/features src/hooks src/stores src/lib src/components/shared && \
touch src/features/.gitkeep src/hooks/.gitkeep src/stores/.gitkeep src/components/shared/.gitkeep src/routes.ts src/providers.tsx
```

Now — and only now — send your first FEATURE prompt.

---

## 28. EXAMPLE: FULL PROMPT SEQUENCE (Research America audio analytics)

**Turn 1 — System prompt (§1) + Business context (§2):**
```
[Paste §1 verbatim]

BUSINESS CONTEXT

2.1 Identity
- Company: Research America Inc.
- Product name: Qualitative Audio Analytics Portal
- Stakeholders: VP Research (owner), CTO (tech)

2.2 Personas
| Admin    | Admin   | Tenants, Users, Cost, Quality, Ops cockpit | full          |
| Analyst  | Analyst | Projects, sessions, transcripts, quotes     | own tenant    |

2.3 Core Workflows
1. Analyst uploads interview audio → transcribed + PII-redacted → themes extracted → quotes tagged
2. Analyst reviews session, approves, publishes summary to client
3. Admin monitors cost, quality, retention, tenant isolation

2.4 Data Domain
- Session, Project, Theme, Quote, Tenant, User, Job, CostLedgerEntry, AuditEvent
- Session → many Themes, many Quotes; Job belongs to Session; Session belongs to Project belongs to Tenant

2.5 Branding
- Primary: 221 83% 53% (HSL) — deep blue
- Admin ribbon: 199 89% 48% (cyan)
- Analyst ribbon: 27 96% 45% (amber)
- Font: Inter / JetBrains Mono / Source Serif 4

2.6 Backend API
- Base URL: https://api.nbs-research-america.com/v1
- Auth: Cognito JWT (HttpOnly cookie)
- Envelope: { data, meta, errors }
- Key endpoints: /projects, /sessions, /sessions/{id}/approve, /admin/ops, /admin/cost, /admin/quality, /admin/users, /admin/tenants

2.7 Tenancy: multi-tenant via Cognito custom:tenant_id, admin can switch.

2.8 Integrations: S3 presigned, Step Functions for transcription, AWS Transcribe + Comprehend PII + Bedrock Claude.

2.10 Compliance: SOC 2, HIPAA toggle per tenant; audit every mutation.

2.11 Performance: LCP < 2.5s; bundle < 250kb gzip initial.

2.13 Anti-patterns:
- No analyst tenant-switch
- No cost view for ClientViewer
- No inline PII (always redacted in UI)
```

**Turn 2 — First feature:**
```
FEATURE: Analyst Session Detail View

USER STORY:
As an analyst, I want to listen to an interview with synced transcript, themes, and quotes so I can review and approve a session.

ACCEPTANCE CRITERIA:
- [ ] Route /analyst/sessions/:id (analyst + admin only)
- [ ] Left: audio player (wavesurfer) with keyboard shortcuts
- [ ] Right: Tabs — Transcript / Summary / Quotes / Themes
- [ ] Transcript lines highlighted on current playback time
- [ ] Click transcript line → seek audio
- [ ] Approve button (Moderator role only) → POST /sessions/:id/approve
- [ ] Export dropdown (CSV, PDF) — analyst+
- [ ] Skeleton / empty / error states

DATA:
- Query: GET /sessions/:id → Session + transcript[] + themes[] + quotes[]
- Query key: ["session", id]
- Mutation: POST /sessions/:id/approve

UI NOTES:
- Two-column layout (60/40)
- Shortcuts: Space/K=play, J/L=±10s, arrows=±5s
- Highlight active transcript line, auto-scroll into view

ROLES:
- View: Admin, Analyst, Moderator
- Approve: Moderator (+ Admin)

NON-GOALS:
- Editing quotes inline (separate feature)

Produce all files: api types/hooks/mutations, components, route.
```

**Turn 3 — iterate:**
```
ITERATE ON: Analyst Session Detail View

CHANGE:
Add a "Compare to previous" button next to Approve. When clicked, opens a side-by-side Dialog with the previous approved session's summary and quotes.

CONSTRAINTS:
- Do not restructure existing tab layout
- Reuse existing Quote/Theme components
- New dep only if unavoidable
```

---

## 29. CUSTOMIZING THE FRAMEWORK

Keep §1 stable across projects. Customize by:
- Swapping the UI library (replace shadcn references globally if another kit is mandated).
- Swapping state mgmt (Zustand → Jotai / Redux Toolkit only if team standard demands).
- Adding/removing integrations in §2.8.
- Tightening/loosening §17 budgets per product lifecycle stage.

Version the framework in git. Bump the version at the top when a rule changes.

---

## 30. INTEGRATION WITH EXISTING NORTHBAY / 369FORECAST WORK

- **PAF Retail / AFIE / Sales AI Platform** — swap the business context.
- **369Forecast Elite Capital dashboards** — MacroPulse, options simulators, gamma scalping visualizations. Stack already matches.
- **Tamimi Flyer Platform** — adapt §2.5 for Arabic/English; add RTL handling to §1.4 styling rules.
- **F369_LLM_TEMPLATES kit library** — use this framework's §1+§2 as every kit's frontend system prompt.
- **Research America Inc.** — reference implementation at `E:\NBS_Research_America_regen\frontend\`.

---

## 31. QUICK START CHECKLIST

For the next project:
- [ ] Run §27's one-liner to scaffold.
- [ ] Commit §25 CLAUDE.md + §26 PR template.
- [ ] Open a new Claude conversation.
- [ ] Paste §1 verbatim.
- [ ] Fill in §2 (business context) and paste.
- [ ] Send your first §3 feature prompt.
- [ ] Review against §5 quality gates (all 15).
- [ ] Run §18 dark-mode audit + §19 a11y audit.
- [ ] Commit, review, iterate.

Same framework, any business domain, consistent production-grade React frontends.
