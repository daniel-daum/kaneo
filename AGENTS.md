# Kaneo — Project Context for Oz

## What This Project Is

Kaneo is a self-hosted Kanban/project management app (pnpm monorepo). I've forked it from the original
author and am hosting my fork on **tangled.org** (a decentralized Git platform built on AT Protocol / atproto).

The app already has a **two-way GitHub Issues sync**: create a task in Kaneo → issue appears on GitHub,
comment on GitHub → syncs back to Kaneo. The existing GitHub integration lives in:
- Backend plugin: `apps/api/src/plugins/github/`
- Backend routes/controllers: `apps/api/src/github-integration/`
- Frontend fetchers: `apps/web/src/fetchers/github-integration/`
- Frontend hooks: `apps/web/src/hooks/mutations/github-integration/` and `hooks/queries/github-integration/`
- Frontend component: `apps/web/src/components/project/github-integration-settings.tsx`

## My Goal

Build a **Tangled/AT Protocol two-way sync integration** for Kaneo — mirroring the GitHub integration
but for tangled.org. If you create a Kaneo task, it creates a `sh.tangled.repo.issue` record on the
user's PDS (and vice versa). Comments, state changes, labels all sync both ways.

My fork lives at tangled.org. This is a learning project — I am new to TypeScript in a complex
monorepo but have experience in Python, Go, React (light), and SQL.

**Fork URLs:**
- Tangled (primary): `https://tangled.org/danieldaum.net/kaneo`
- GitHub fork (for upstream PRs): `https://github.com/daniel-daum/kaneo`
- Upstream: `https://github.com/usekaneo/kaneo`

## Mentorship Mode — IMPORTANT

**Do not generate large blocks of code for me unless I explicitly ask.**

My primary goal is to *learn* while building this. The preferred interaction style is:
- Explain *why* before *how*
- Show me the relevant existing code pattern, then ask me to try writing it
- Point me to the file/function to look at, let me read it first
- Review and correct what I write rather than writing it for me
- When I'm stuck, give targeted hints rather than full solutions
- If I ask "how do I do X", explain the concept and show a small example — don't implement X in the codebase for me

That said: I want to move at a reasonable pace. If something is purely boilerplate/mechanical (e.g.
adding an env var to a config file, or creating an empty directory structure), go ahead and do it.

## Tangled / AT Protocol Architecture

Full reference docs are in `~/starforge/documentation/tangled/`:
- `001-architecture-and-api.md` — API surface, XRPC endpoints, lexicon catalog, webhooks
- `002-atproto-vs-tangled-boundary.md` — exactly which operations go to atproto vs knot vs appview

### Key facts to keep in mind:

**How Tangled works (two tiers):**
1. **User's PDS** (AT Protocol) — stores records like issues, PRs, labels. Accessed via standard
   `com.atproto.repo.*` methods. This is where ~80% of the work happens.
2. **Knot servers** — host actual git repos. Expose XRPC endpoints for branches, diffs, trees, etc.
   Require a service auth token obtained from the user's PDS.
3. **AppView (tangled.org)** — server-rendered HTML only. NOT an API. Only used for display URLs
   and as the `aud` value when getting service auth tokens (`did:web:tangled.org`).

**Creating an issue on Tangled** = writing a `sh.tangled.repo.issue` record to the user's PDS via
`com.atproto.repo.createRecord`. No Tangled HTTP API involved.

**Knot discovery** — to call knot XRPC, read the `sh.tangled.repo` record from the owner's PDS
and extract the `knot` field. Never hardcode knot hostnames.

**Service auth flow for knot calls:**
1. Auth to user's PDS with handle + app password
2. `com.atproto.server.getServiceAuth` with `aud: did:web:tangled.org`
3. Get a short-lived bearer token (60s expiry)
4. Use it in `Authorization: Bearer {token}` for knot XRPC calls

**Receiving events from Tangled** → use webhooks (documented at docs.tangled.org/webhooks).
Webhooks are configured per-repo in the repo settings UI. Signed payloads, delivery retries.

**Relevant atproto collections:**
- `sh.tangled.repo` — repo metadata (name, knot, description)
- `sh.tangled.repo.issue` — issues (fields: `repo` as at-uri, `title`, `body`, `createdAt`)
- `sh.tangled.repo.issue.comment` — issue comments
- `sh.tangled.repo.issue.state` — state change record
- `sh.tangled.label.op` — label apply/remove
- `sh.tangled.label.definition` — label definitions

**Best third-party reference:** `tangled.org/zzstoatzz/tangled-mcp` — a Python MCP server that
does atproto issue CRUD and knot XRPC calls. Study this when figuring out auth and record writes.

## Tech Stack (Kaneo)

See `CLAUDE.md` for full details. Summary:
- **Backend**: Hono (Node.js), PostgreSQL + Drizzle ORM, Better Auth, Valibot for validation
- **Frontend**: React 19, TanStack Router + Query, Vite, Tailwind CSS v4, Zustand
- **Monorepo**: pnpm + TurboRepo
- **Package manager**: pnpm (not npm/yarn)
- Dev ports: API on 1337, web on 5173

## Implementation Roadmap

The full step-by-step plan is in `~/starforge/documentation/tangled/`. High-level phases:

1. **Setup & credentials** — env vars for Tangled handle/app-password, AT Protocol client library
2. **Outbound sync (Kaneo → Tangled)** — when a task is created/updated/commented in Kaneo, write
   the corresponding atproto record to the user's PDS
3. **Inbound sync (Tangled → Kaneo)** — configure a Tangled webhook, handle incoming payloads,
   update Kaneo tasks accordingly
4. **UI** — settings page for connecting a Tangled repo to a Kaneo project (mirrors GitHub settings)
5. **State reconciliation** — handle edge cases, prevent sync loops, label mapping

## Development Setup — Active Plan (updated 2026-04-18)
### Current state snapshot
- `origin` = `git@knot.danieldaum.net:danieldaum.net/kaneo` (tangled). No `upstream` or `github` remote configured yet.
- HEAD detached at `383e545`; `updates.md` has been merged into this file (see appendix).
- Local `main` has three personal commits on top of an old `upstream/main` (~408 commits behind `usekaneo/kaneo`):
  - `27766df ci: add ci mirror to github` — personal; must NOT leak into an upstream PR
  - `6ae6bf5 feat: add atproto/api sdk` — upstream-worthy; belongs on the feature branch
  - `383e545 feat: add stubbed tangled plugin type` — upstream-worthy; belongs on the feature branch
- `.tangled/workflows/mirror-to-github.yaml` force-pushes `main` from tangled → github on every push.
- Upstream branch-naming convention (from `CONTRIBUTING.md` and recent merge commits): `feat/<name>` or `fix/<name>`. Chosen feature branch: **`feat/tangled-integration`**.
### Git model
Three remotes on the local repo (in the orbstack VM at `~/starforge/kaneo`):
- `origin` → tangled knot (`git@knot.danieldaum.net:danieldaum.net/kaneo`) — primary forge, source of truth
- `upstream` → `https://github.com/usekaneo/kaneo.git` — pristine upstream; rebase target and PR target
- `github` → `git@github.com:daniel-daum/kaneo.git` — fork used as PR source (fed by the spindle mirror)
Two long-lived branches on my fork:
- `main` = `upstream/main` + a thin layer of **personal-only** commits (mirror workflow, flake, AGENTS.md). This is the branch that mirrors to GitHub and is never the PR source.
- `feat/tangled-integration` = branched off `upstream/main`; contains the atproto SDK + plugin stub + all future integration work. This becomes the upstream PR.
Flow when contributing:
1. Work on `feat/tangled-integration` locally.
2. Push to `origin` (tangled) → spindle mirrors to `github` → open/update a PR on the github fork targeting `usekaneo/kaneo:main`.
3. When upstream advances, `git fetch upstream && git rebase upstream/main` on the feature branch.
4. Keep `main` in sync by resetting to `upstream/main` and re-applying the personal commits on top.
### Personal-file policy
These files live **only** on `main` (or a local branch derived from main), never on the feature branch:
- `AGENTS.md`
- `flake.nix`, `flake.lock`, `.envrc`
- `.tangled/workflows/**`
- Anything we later add under a `.local/` directory
Guardrails:
- Always `git switch main` (or `jj new main@origin`) before editing those files. Never edit them while HEAD is on `feat/tangled-integration`.
- Before pushing a feature branch: `git diff --name-only upstream/main..HEAD` — none of the paths above should appear.
- Optional: a pre-push hook that fails if those paths appear on any `feat/*` branch.
### Chronological checklist — do ONE step at a time
Stop after each step, verify, then re-read this file before starting the next. Commands are illustrative — pick `git` or `jj` per your preference; `.jj/` is colocated so both work.
1. **Commit the consolidated `AGENTS.md` on `main`.**
   - `jj edit main` (or `git switch main` for plain git).
   - `jj describe -m "docs: consolidate integration context into AGENTS.md"` then `jj new` to open a fresh empty `@`.
   - Verify `jj status` is clean; `jj log -r main..@` shows the new commit.
2. **Add the `upstream` and `github` remotes, fetch them.**
   - `git remote add upstream https://github.com/usekaneo/kaneo.git`
   - `git remote add github  git@github.com:daniel-daum/kaneo.git`
   - `git fetch upstream && git fetch github`
   - Confirm `git log --oneline upstream/main | head` shows ~408 new commits.
3. **Safety net before rewriting history.**
   - `git tag backup/pre-rewrite-main main`
   - `git push origin refs/tags/backup/pre-rewrite-main`
4. **Create `feat/tangled-integration` from `upstream/main` with only the upstream-worthy commits.**
   - `git switch -c feat/tangled-integration upstream/main`
   - `git cherry-pick 6ae6bf5 383e545`   # atproto sdk + stubbed plugin
   - Run `pnpm install` and `pnpm lint` to ensure it still builds against new upstream.
   - `git push -u origin feat/tangled-integration`
5. **Rewrite `main` = upstream/main + personal commits only.**
   - `git switch main`
   - `git reset --hard upstream/main`
   - `git cherry-pick 27766df`   # ci: add ci mirror to github (personal)
   - Re-apply personal files if they are no longer present: `AGENTS.md` and (later) `flake.nix`.
   - `git push origin main --force-with-lease`
   - Spindle mirrors the new `main` to github; verify on the github fork.
6. **Extend spindle workflow so feature branches also mirror to github.**
   - Edit `.tangled/workflows/mirror-to-github.yaml` on `main`: expand `when.branch` to `["main", "feat/*", "fix/*"]` and push each ref generically (`git push github HEAD:refs/heads/$BRANCH --force`).
   - Commit on `main`, push. Then push a no-op commit (or `--allow-empty`) to `feat/tangled-integration` and confirm it appears on the github fork.
7. **Add the Nix dev flake (minimal profile).**
   - On `main`: add `flake.nix` with inputs `nixpkgs`, output `devShells.default` containing: `nodejs_22`, `corepack_22` (for `pnpm@10.32.1` pinning), `git`, `jj`.
   - Optional `.envrc` with `use flake` for direnv.
   - Commit: `chore(nix): add dev flake`.
   - Inside the VM: `nix develop` → `corepack enable` → `pnpm install` → `pnpm dev` must work.
8. **Open a draft PR on the github fork.**
   - From `daniel-daum:feat/tangled-integration` → `usekaneo:main`.
   - Mark as draft; fill out `.github/PULL_REQUEST_TEMPLATE.md`.
   - Keep as draft until the integration is end-to-end usable.
9. **Resume integration work** from "Next Session — Start Here" below. All subsequent feature commits land on `feat/tangled-integration`, get mirrored to github, and accumulate on the draft PR.
### Pitfalls & traps
- **Leaking personal files into the PR.** The single biggest risk. Always diff against `upstream/main` before pushing `feat/*`.
- **Force-pushing `main`.** The mirror workflow already uses `--force`, so this is acceptable. The rule is: never open a PR *from* `main` — always from `feat/*`.
- **`pnpm-lock.yaml` churn.** Pulling 408 upstream commits will change the lockfile and may change Node engine constraints. Always re-run `pnpm install` after the rebase/reset.
- **Drizzle migrations drift.** Upstream likely added migrations. Expect to rebuild the local Postgres or run pending migrations after the upstream pull.
- **jj colocation quirks.** After the `reset --hard` and cherry-picks on `main`, run `jj git import` and verify `jj log` matches. Use `jj bookmark set main -r @` if bookmarks fall behind.
- **SSH/PAT paths differ per remote.** `origin` (ssh → knot), `github` (ssh → github.com), `upstream` (https, read-only). Confirm `ssh -T git@github.com` succeeds inside the VM. The spindle's `GITHUB_PAT` secret must still have `repo` scope for the mirror to push.
- **Spindle only knows `main` today.** If step 6 is skipped, feature branches will not appear on the github fork and no PR can be opened.
- **Branch-naming drift.** Upstream uses `feat/…` / `fix/…`. Do not invent other prefixes — their maintainers may close PRs that do not match.
- **@atproto/api session expiry.** Sessions expire; cache and `agent.resumeSession` rather than logging in per request (see appendix below).
- **Sync loop prevention.** When a Tangled webhook arrives, don't re-fire outbound sync. Plan an idempotency key before writing the first outbound handler.
- **App password scopes.** The Tangled app password must permit PDS writes for the target collections (`sh.tangled.repo.issue`, `.comment`, `.state`, `sh.tangled.label.op`). Verify against `tangled/001-architecture-and-api.md` before generating the token.
- **Biome scope.** Biome only lints JS/TS, so `flake.nix`, YAML, and markdown are safe; still run `pnpm lint` after every significant change.

## Session Progress

### Session 1 (2026-03-20)
**Completed:**
- Installed `@atproto/api` as a dependency in `apps/api/package.json`
- Created `apps/api/src/plugins/tangled/` directory
- Wrote `config.ts` — Valibot schema (`tangledConfigSchema`), inferred `TangledConfig` type,
  and `validateTangledConfig` async function
- Wrote `index.ts` — `tangledPlugin: IntegrationPlugin` object with stubbed async handler
  functions for all task events (`onTaskCreated`, `onTaskStatusChanged`, `onTaskTitleChanged`,
  `onTaskDescriptionChanged`, `onTaskPriorityChanged`, `onTaskCommentCreated`)

**Config shape (may expand later):**
- `repoAtUri: string` — the AT-URI of the Tangled repo to sync with
  (e.g. `at://did:plc:abc123.../sh.tangled.repo/reponame`)

**Concepts covered this session:**
- Valibot: runtime validation library, similar to Python's pydantic. Schema = source of truth;
  `v.InferOutput` derives the TS type from it. Used for API input validation and plugin config validation.
- `try/catch` with `v.parse()`: parse throws on failure, catch returns structured errors.
- `IntegrationPlugin` interface in `plugins/types.ts` is the contract all plugins must satisfy.
- Handler function naming: no collision risk since each plugin lives in its own module scope.
- Prefer named function declarations over arrow functions — valid style, scales well when
  handlers move to their own files under `events/`.

---

### Next Session — Start Here

**Step 1: Register the plugin**
Find where `githubPlugin` is passed to the plugin registry and where `initializeGitHubPlugin()`
is called in the main API app. Add `tangledPlugin` alongside it.
Likely in `apps/api/src/index.ts` or a `plugins/index.ts` — grep for `githubPlugin` to find it.

**Step 2: Add env vars**
Add `TANGLED_HANDLE` and `TANGLED_APP_PASSWORD` to:
- The root `.env` file
- Wherever env vars are declared/typed in the API (look for `env.ts` or `config.ts` near
  `apps/api/src/` — grep for `GITHUB_` to find the pattern)

These are the credentials used to authenticate to the user's PDS via `@atproto/api`.

**Step 3: Write an atproto client helper**
Once env vars are in place, create a small helper (e.g. `apps/api/src/plugins/tangled/client.ts`)
that authenticates with `@atproto/api` using those env vars and exports a ready-to-use client.
This will be used by all the event handlers.

---

## Working Notes

- The GitHub plugin uses an **event system** (`publishEvent`) to react to Kaneo-side changes.
  The Tangled plugin will follow the same pattern.
- Tangled webhooks will play the role that GitHub webhooks play in the existing integration.
- There is no official AT Protocol TypeScript SDK maintained by Tangled; use the `@atproto/api`
  package (Bluesky's SDK) — it works for any PDS including Tangled's.
- Watch out for sync loops: when Kaneo handles a Tangled webhook, it must not re-fire outbound sync.

## Appendix — Full Integration Context (merged from updates.md on 2026-04-18)
Sections above are the current canonical plan. This appendix preserves the deeper reference material originally drafted in `updates.md`.

# Kaneo × Tangled Integration — Context Document

> Generated 2026-04-16. Drop this file into any Claude/Warp session for full context.

---

## 1. What Is Kaneo

Self-hosted TypeScript kanban/project-management app. pnpm monorepo (Turbo). MIT license.

- **Backend**: Hono (Node.js), PostgreSQL + Drizzle ORM, Better Auth, Valibot validation
- **Frontend**: React 19, TanStack Router + Query, Vite, Tailwind v4, Zustand
- **Upstream GitHub**: https://github.com/usekaneo/kaneo
- **Your Tangled fork**: https://tangled.org/danieldaum.net/kaneo
- **Dev ports**: API `1337`, web `5173`
- **Package manager**: pnpm (pinned 10.28.0), Node ≥ 18

Kaneo already has a fully working **two-way GitHub Issues sync** and a recently added **Gitea integration**. These are the direct models for the Tangled integration.

---

## 2. Existing Integration Architecture (your reference model)

### File layout — GitHub integration

```
apps/api/src/plugins/github/          ← event handler plugin (outbound sync)
apps/api/src/github-integration/      ← Hono routes + controllers (inbound webhook + CRUD)
apps/web/src/fetchers/github-integration/
apps/web/src/hooks/mutations/github-integration/
apps/web/src/hooks/queries/github-integration/
apps/web/src/components/project/github-integration-settings.tsx
```

### Plugin system (`apps/api/src/plugins/types.ts`)

All integrations implement `IntegrationPlugin`:

```ts
interface IntegrationPlugin {
    type: string;
    name: string;
    validateConfig(
        config: unknown,
    ): Promise<{ valid: boolean; errors?: string[] }>;
    onTaskCreated(event: TaskCreatedEvent, ctx: PluginContext): Promise<void>;
    onTaskStatusChanged(
        event: TaskStatusChangedEvent,
        ctx: PluginContext,
    ): Promise<void>;
    onTaskTitleChanged(
        event: TaskTitleChangedEvent,
        ctx: PluginContext,
    ): Promise<void>;
    onTaskDescriptionChanged(
        event: TaskDescriptionChangedEvent,
        ctx: PluginContext,
    ): Promise<void>;
    onTaskPriorityChanged(
        event: TaskPriorityChangedEvent,
        ctx: PluginContext,
    ): Promise<void>;
    onTaskCommentCreated(
        event: TaskCommentCreatedEvent,
        ctx: PluginContext,
    ): Promise<void>;
}
```

### Database tables (already exist, reuse for Tangled)

| Table           | Purpose                                                                                    |
| --------------- | ------------------------------------------------------------------------------------------ |
| `integration`   | Generic row per project, `type` field = `"github"` / `"gitea"` / `"tangled"`               |
| `external_link` | Maps `task_id → external_id (rkey)` + `integration_id`, `resource_type`, `url`, `metadata` |
| `workflow_rule` | Optional: auto-move task column on external event                                          |
| `activity`      | Task timeline; has `external_source`, `external_user_name`, `external_url` fields          |

A dedicated `github_integration` table (project_id unique FK, repo owner/name, installation_id) exists as a model — you'll want an analogous `tangled_integration` table.

---

## 3. Tangled / AT Protocol Architecture

### Two-tier model

1. **User's PDS** (AT Protocol server) — stores all forge records (issues, PRs, labels). Accessed via standard `com.atproto.repo.*` XRPC methods. This is ~80% of the work for issue sync.
2. **Knot server** — hosts actual git repos. Exposes XRPC endpoints for git operations. Requires a service auth token. **Not needed for issue sync.**
3. **AppView (tangled.org)** — server-rendered HTML only. Not an API. Used only for display URLs.

### Key principle

**Creating/updating a Tangled issue = writing an atproto record to the user's PDS.**
There is no Tangled REST API for this. No Tangled app registration needed.

### Auth flow (for PDS writes)

```
1. AtpAgent.login({ identifier: TANGLED_HANDLE, password: TANGLED_APP_PASSWORD })
2. All com.atproto.repo.* calls are now authenticated
3. No OAuth, no app installation — just handle + app password
```

Service auth tokens (`com.atproto.server.getServiceAuth`) are only needed for knot XRPC (git ops). Skip for issue sync.

### Relevant atproto collections (lexicons)

| Collection                      | Purpose            | Key fields                                    |
| ------------------------------- | ------------------ | --------------------------------------------- |
| `sh.tangled.repo`               | Repo metadata      | `name`, `knot`, `description`                 |
| `sh.tangled.repo.issue`         | Issue record       | `repo` (at-uri), `title`, `body`, `createdAt` |
| `sh.tangled.repo.issue.comment` | Comment on issue   | `issue` (at-uri), `body`, `createdAt`         |
| `sh.tangled.repo.issue.state`   | State change       | `issue` (at-uri), `open` (bool)               |
| `sh.tangled.label.op`           | Apply/remove label | `issue`, `label`, `op`                        |
| `sh.tangled.label.definition`   | Label definition   | `name`, `color`                               |

### Creating an issue (pseudocode)

```ts
await agent.api.com.atproto.repo.createRecord({
    repo: agent.session.did, // your DID
    collection: "sh.tangled.repo.issue",
    record: {
        $type: "sh.tangled.repo.issue",
        repo: config.repoAtUri, // "at://did:plc:xxx/sh.tangled.repo/reponame"
        title: task.title,
        body: task.description ?? "",
        createdAt: new Date().toISOString(),
    },
});
// Response: { uri, cid } — store uri/rkey in external_link.external_id
```

### Knot discovery (not needed for issues, FYI)

Read the `sh.tangled.repo` record from owner's PDS, extract `knot` field. Never hardcode knot hostnames.

### Receiving events (inbound sync)

Tangled sends **webhooks** — signed HTTP POST payloads to a URL you configure in repo settings UI.

- Docs: https://docs.tangled.org/webhooks
- Signature: HMAC-SHA256, header `X-Tangled-Signature` (verify before processing)
- Events include: `issue.created`, `issue.state`, `issue.comment.created`, etc.
- Delivery retries on failure

### Best reference implementation

`tangled.org/zzstoatzz/tangled-mcp` — Python MCP server doing atproto issue CRUD + knot XRPC. Study for auth and record write patterns.

---

## 4. What You've Already Built (Session 1, 2026-03-20)

### Commits on your fork

- `6ae6bf57` — `feat: add atproto/api sdk` — `@atproto/api` added to `apps/api/package.json`
- `383e5457` — `feat: add stubbed tangled plugin type` — scaffolded plugin, AGENTS.md added

### Files created

```
apps/api/src/plugins/tangled/config.ts   ← Valibot schema, TangledConfig type, validateTangledConfig()
apps/api/src/plugins/tangled/index.ts    ← tangledPlugin: IntegrationPlugin (all handlers stubbed)
AGENTS.md                                ← agent context doc (Oz/mentorship mode)
```

### Config shape (current)

```ts
// config.ts
export const tangledConfigSchema = v.object({
    repoAtUri: v.string(), // e.g. "at://did:plc:abc123/sh.tangled.repo/reponame"
});
export type TangledConfig = v.InferOutput<typeof tangledConfigSchema>;
```

### Plugin shape (current — all handlers are empty `{}`)

```ts
export const tangledPlugin: IntegrationPlugin = {
    type: "tangled",
    name: "Tangled",
    onTaskCreated: handleTaskCreated,
    onTaskStatusChanged: handleTaskStatusChanged,
    onTaskPriorityChanged: handleTaskPriorityChanged,
    onTaskTitleChanged: handleTaskTitleChanged,
    onTaskDescriptionChanged: handleTaskDescriptionChanged,
    onTaskCommentCreated: handleTaskCommentCreated,
    validateConfig: validateTangledConfig,
};
```

---

## 5. Remaining Work — Ordered Roadmap

### Phase 1 — Wire up the plugin (no auth yet)

- [ ] Find where `githubPlugin` is registered — grep `githubPlugin` in `apps/api/src/index.ts` or nearby `plugins/index.ts`
- [ ] Register `tangledPlugin` alongside it in the same place
- [ ] Add `TANGLED_HANDLE` and `TANGLED_APP_PASSWORD` to `.env.sample` and the API's env config (grep `GITHUB_APP_ID` to find the pattern file)

### Phase 2 — atproto client helper

- [ ] Create `apps/api/src/plugins/tangled/client.ts`
- [ ] Use `AtpAgent` from `@atproto/api`, login with env vars, export the authenticated agent
- [ ] Handle re-auth (sessions expire; catch auth errors and re-login)

### Phase 3 — Outbound sync (Kaneo → Tangled)

- [ ] `handleTaskCreated`: `createRecord` → `sh.tangled.repo.issue` → store returned `uri` rkey in `external_link`
- [ ] `handleTaskStatusChanged`: `createRecord` → `sh.tangled.repo.issue.state` (open/closed)
- [ ] `handleTaskCommentCreated`: `createRecord` → `sh.tangled.repo.issue.comment`
- [ ] `handleTaskTitleChanged` / `handleTaskDescriptionChanged`: `updateRecord` on the issue record
- [ ] Status mapping: `to-do`/`in-progress` → `open: true`; `done`/`archived` → `open: false`

### Phase 4 — Database schema

- [ ] Add `tangled_integration` table to `apps/api/src/database/schema.ts`:
    - `id` (CUID2), `project_id` (unique FK → project), `repo_at_uri`, `is_active`, `created_at`, `updated_at`
- [ ] Reuse existing `integration` table (type `"tangled"`) and `external_link` table as-is
- [ ] Run: `pnpm --filter @kaneo/api db:generate`

### Phase 5 — Inbound sync (Tangled → Kaneo)

- [ ] Add Hono route: `POST /tangled-integration/webhook` in new `apps/api/src/tangled-integration/`
- [ ] Verify `X-Tangled-Signature` HMAC before processing
- [ ] Handle `issue.created` → create Kaneo task + `external_link`
- [ ] Handle `issue.state` → update task status column
- [ ] Handle `issue.comment.created` → add activity record with `external_source: "tangled"`
- [ ] **Loop prevention**: check `external_source` flag so inbound webhook handler does not re-fire outbound plugin events

### Phase 6 — API routes (CRUD for integration config)

- [ ] Mirror `apps/api/src/github-integration/` controllers:
    - `POST /tangled-integration` — save config (repo_at_uri), create `tangled_integration` row
    - `GET /tangled-integration/:projectId` — fetch config
    - `DELETE /tangled-integration/:projectId` — remove (cascade deletes external_links)
    - `POST /tangled-integration/:projectId/import` — bulk import existing Tangled issues as tasks

### Phase 7 — Frontend

- [ ] `apps/web/src/fetchers/tangled-integration/` — fetch/mutate integration config
- [ ] `apps/web/src/hooks/queries/tangled-integration/` + `hooks/mutations/tangled-integration/`
- [ ] `apps/web/src/components/project/tangled-integration-settings.tsx` — mirrors GitHub settings component
    - Input: paste the repo's AT-URI (simpler than GitHub — no OAuth app install)
    - Toggle active/inactive
    - Import issues button

### Phase 8 — Edge cases

- [ ] Idempotency: check `external_link` before creating duplicate records
- [ ] Handle Tangled record deletions (webhook `issue.deleted` if it exists)
- [ ] Label sync (optional, `sh.tangled.label.op` + `sh.tangled.label.definition`)

---

## 6. Git Workflow & Remote Setup

### Remote topology

```
usekaneo/kaneo (upstream — source of truth)
        ↓  manual pull via "upstream" remote
local machine
    ├── main             → push → tangled:danieldaum.net/kaneo main
    │                          → spindle CI → github:yourusername/kaneo main
    └── feat/tangled-integration
                         → push → tangled:danieldaum.net/kaneo feat/tangled-integration
                                → spindle CI → github:yourusername/kaneo feat/tangled-integration
```

### One-time local remote setup

```bash
# Rename existing origin if needed, then add all three
git remote add upstream https://github.com/usekaneo/kaneo.git
git remote add tangled  git@knot.danieldaum.net:danieldaum.net/kaneo
git remote add github   https://github.com/yourusername/kaneo.git
# verify
git remote -v
```

### Pulling upstream updates (run periodically)

```bash
git fetch upstream
git checkout main
git rebase upstream/main        # keeps history clean
git push tangled main           # spindle CI then mirrors to github main automatically
```

### Daily dev workflow

```bash
# Keep base current before starting work
git fetch upstream && git rebase upstream/main

# Work on your feature branch
git checkout feat/tangled-integration   # create with -b if first time
# ... make commits ...
git push tangled feat/tangled-integration
# spindle CI mirrors it to github feat/tangled-integration automatically

# When ready to PR upstream:
# Open PR on GitHub: yourusername/kaneo feat/tangled-integration → usekaneo/kaneo main
```

### Spindle CI workflow (current → updated)

**Current** `.tangled/workflows/mirror-to-github.yaml` only triggers on `main` and hardcodes the branch ref. This means feature branch commits are never mirrored.

**Updated workflow:**

```yaml
# .tangled/workflows/mirror-to-github.yaml

when:
    - event: ["push", "manual"]
      branch: ["main", "feat/*"]

engine: "nixery"

clone:
    depth: 0

dependencies:
    nixpkgs:
        - git

steps:
    - name: "Mirror to GitHub"
      command: |
          git fetch --unshallow origin || true
          git fetch origin refs/heads/${SPINDLE_BRANCH}:refs/heads/${SPINDLE_BRANCH}
          git remote add github https://x-access-token:${GITHUB_PAT}@github.com/${GITHUB_USERNAME}/${GITHUB_REPO_NAME}.git
          git push github refs/heads/${SPINDLE_BRANCH}:refs/heads/${SPINDLE_BRANCH} --force-with-lease
      environment:
          GITHUB_USERNAME: "YOUR-GITHUB-USERNAME"
          GITHUB_REPO_NAME: "YOUR-GITHUB-REPO-NAME"
          # GITHUB_PAT must be set as a spindle secret, not here
```

**Changes from original:**

- `branch: ["main", "feat/*"]` — now triggers on feature branches too
- `${SPINDLE_BRANCH}` — dynamic branch name instead of hardcoded `main`
- `--force-with-lease` instead of `--force` — won't overwrite GitHub commits you haven't seen
- `GITHUB_PAT` stays in secrets

⚠️ **Verify the branch variable name.** `SPINDLE_BRANCH` is the expected name based on spindle's architecture but confirm it by adding `echo "Branch: ${SPINDLE_BRANCH}"` as a first step on your next run. Check `tangled.org/tangled.org/core/tree/master/spindle` if unsure.

---

## 7. Contributing Back to Upstream Kaneo

### The standard OSS flow (what CONTRIBUTING.md says)

Kaneo uses the standard GitHub fork → feature branch → PR model. You do **not** need prior write access.

**Step-by-step:**

```bash
# 1. Fork usekaneo/kaneo on GitHub (click Fork in the UI) → creates github.com/yourusername/kaneo
# 2. Clone your GitHub fork locally
git clone https://github.com/yourusername/kaneo.git
cd kaneo

# 3. Add upstream as a remote so you can stay in sync
git remote add upstream https://github.com/usekaneo/kaneo.git

# 4. Create a feature branch (their convention: feat/ prefix)
git checkout -b feat/tangled-integration

# 5. Do your work, commit using conventional commits
git commit -m "feat: add Tangled/AT Protocol two-way issue sync integration"

# 6. Keep your branch up to date with upstream main
git fetch upstream
git rebase upstream/main

# 7. Push to YOUR GitHub fork
git push origin feat/tangled-integration

# 8. Open a PR on GitHub from your fork's branch → usekaneo/kaneo main
```

### Your Tangled fork vs. your GitHub fork — two separate things

- `tangled.org/danieldaum.net/kaneo` — your experimental/learning fork, hosted on Tangled. This is where you develop.
- `github.com/yourusername/kaneo` — the GitHub fork you create specifically to open a PR upstream.

You can keep both in sync by pushing to both remotes, or just cherry-pick your Tangled commits over to the GitHub fork when ready.

### Before opening the PR

- Run `pnpm lint` (Biome) — the pre-commit hook will block if this fails
- Run `pnpm build` — full monorepo build check (also runs in pre-commit)
- Discuss with the maintainer (Discord or a GitHub Discussion) before building the full feature — CONTRIBUTING.md says "Have an idea? Let's discuss it first" for new features
- Open a GitHub Discussion or Issue titled something like "feat: Tangled/AT Protocol integration (mirrors Gitea integration)" to get a thumbs-up before submitting a large PR

### Permissions

You have no write access to `usekaneo/kaneo` — that's correct and expected. The fork + PR model means you never need it. The maintainer merges your PR.

### PR description checklist (what makes a clean PR)

- What: short summary of what the integration does
- Why: link to Tangled, note it mirrors the existing Gitea integration pattern
- How: list the files added/changed, note the new env vars required (`TANGLED_HANDLE`, `TANGLED_APP_PASSWORD`)
- Testing: describe how a reviewer can test it locally (what to configure, what to observe)
- i18n: add any new UI strings to `i18n/en-US.json` (their CONTRIBUTING.md requires this)

---

## 7. Key File Paths Quick Reference

```
apps/api/src/plugins/tangled/config.ts        ← YOUR FILE (exists)
apps/api/src/plugins/tangled/index.ts         ← YOUR FILE (exists, stubbed)
apps/api/src/plugins/tangled/client.ts        ← TODO: atproto agent helper
apps/api/src/plugins/types.ts                 ← IntegrationPlugin interface
apps/api/src/plugins/github/                  ← REFERENCE: GitHub plugin
apps/api/src/github-integration/              ← REFERENCE: GitHub routes/controllers
apps/api/src/database/schema.ts               ← Add tangled_integration table here
apps/api/src/database/relations.ts            ← Add relations here
apps/web/src/components/project/github-integration-settings.tsx  ← REFERENCE: UI component
i18n/en-US.json                               ← Add any new UI strings here
```

---

## 8. Useful Commands

```bash
# Start dev environment
pnpm dev

# After schema changes
pnpm --filter @kaneo/api db:generate
# (migrations auto-run on API startup)

# Lint/format (required before commit)
pnpm lint

# Full build (also runs in pre-commit hook)
pnpm build

# Filter to one package
pnpm --filter @kaneo/api dev
pnpm --filter @kaneo/web dev
```

---

## 9. Known Pitfalls & Hard Problems

### 1. Sync loop prevention ⚠️ (highest risk)

When Tangled fires a webhook → Kaneo updates a task → the plugin event system fires → writes back to Tangled → another webhook fires → infinite loop.

The GitHub integration has a guard for this somewhere. **Before writing any inbound webhook handler, find it first.** Grep for `external_source` or look for a flag/context field passed through `publishEvent`. The pattern is likely: if the task update was triggered by an inbound webhook, skip outbound plugin dispatch (or check `external_link` to detect it's already synced).

Mitigation strategies:

- Pass a `syncSource: "tangled"` flag through the event context; have the plugin no-op if it sees its own source
- Use a short-lived in-memory set of "currently syncing" task IDs and skip outbound if the task is in it
- Check the `external_source` field on the activity record before dispatching

**Do not ship inbound sync without this solved.**

---

### 2. atproto session expiry (silent failure risk)

`AtpAgent` sessions expire. In a long-running Hono process, the agent will eventually return auth errors on every call and the integration will silently stop working — no crash, no obvious log.

Plan:

- Wrap every `createRecord`/`updateRecord` call in a helper that catches `401`/`AuthenticationRequired` errors, calls `agent.login()` again, and retries once
- The `@atproto/api` agent has a `session` property and a `resumeSession()` method — read the SDK source before rolling your own
- Consider storing the session in a module-level singleton so re-auth doesn't require a full cold login on every request

---

### 3. `repoAtUri` is hostile UX

Asking a user to paste `at://did:plc:abc123.../sh.tangled.repo/reponame` directly is a bad experience and error-prone. The GitHub integration avoids this with an OAuth install flow.

Options (pick one for the UI):

- Accept a friendly `owner.handle/repo-name` format and resolve it to a DID + AT-URI server-side on save
- Make the settings page do a PDS lookup: given a handle, resolve the DID via `com.atproto.identity.resolveHandle`, then list their `sh.tangled.repo` records, and let the user pick from a dropdown
- Minimum viable: accept the raw AT-URI but show a helper link to tangled.org where they can copy it

The second option is the best UX but requires two extra API calls. Implement the raw paste version first, improve later.

---

### 4. Issue state is a separate record (not a field)

Tangled issue state (`open`/`closed`) is stored as a distinct `sh.tangled.repo.issue.state` record — it is **not** a field on the issue itself. This has two consequences:

- **On import**: to know if an existing Tangled issue is open or closed, you must query `sh.tangled.repo.issue.state` records separately and join them by issue AT-URI. You can't just read the issue record.
- **On status sync**: `handleTaskStatusChanged` must write a new `sh.tangled.repo.issue.state` record, not update the issue record. Multiple state records can exist; the most recent one wins.

The `external_link` table's `metadata` JSON field is a good place to cache the last-known state to avoid extra PDS round-trips on every read.

---

### 5. Local webhook development requires a public URL

Tangled webhooks need an HTTPS URL it can POST to. `localhost:1337` will not work.

You'll need one of:

- `ngrok http 1337` — creates a temporary public tunnel, free tier sufficient for dev
- `cloudflared tunnel` — Cloudflare's equivalent, also free
- Deploy to a staging environment for inbound sync testing

Set this up before starting Phase 5 or you won't be able to test inbound sync at all. Add a note in your `.env.sample`: `WEBHOOK_BASE_URL=https://your-ngrok-url.ngrok.io`.

---

## 10. External References

| Resource                     | URL                                                        |
| ---------------------------- | ---------------------------------------------------------- |
| Kaneo upstream               | https://github.com/usekaneo/kaneo                          |
| Your Tangled fork            | https://tangled.org/danieldaum.net/kaneo                   |
| Tangled docs                 | https://docs.tangled.org                                   |
| Tangled webhooks             | https://docs.tangled.org/webhooks                          |
| Tangled core monorepo        | https://tangled.org/tangled.org/core                       |
| DeepWiki: GitHub integration | https://deepwiki.com/usekaneo/kaneo/5.4-github-integration |
| Reference MCP impl (Python)  | https://tangled.org/zzstoatzz/tangled-mcp                  |
| @atproto/api npm             | https://www.npmjs.com/package/@atproto/api                 |
| Kaneo Discord                | https://discord.gg/rU4tSyhXXU                              |
