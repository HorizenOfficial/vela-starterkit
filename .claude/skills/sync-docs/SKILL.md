---
name: sync-docs
description: Use when the user asks to sync, audit, update, refresh, or verify the project's documentation against the source code in the sibling repos (vela, vela-nova, vela-common-go, vela-common-ts). Triggers on phrases like "update docs", "sync documentation", "check docs against repos", "refresh docs", "audit the docs", "docs out of date", "verify README against code". Performs a full audit of every .md file in the repo, reports findings first, and only edits after the user approves.
---

# Documentation Sync Audit

This project (`vela-starterkit`) documents the Vela CCE platform but contains no application source code. The canonical source lives in four sibling repositories checked out next to this one:

| Repo | Path | Language | Role |
|------|------|----------|------|
| `vela` | `../vela` | Go | Core platform: Manager, Executor, smart contracts |
| `vela-nova` | `../vela-nova` | TinyGo | Reference WASM app (private transfer) |
| `vela-common-go` | `../vela-common-go` | Go | Shared WASM types library |
| `vela-common-ts` | `../vela-common-ts` | TypeScript | Browser client library |

Your job when this skill is invoked: audit every markdown file in this repo against whatever is currently checked out in those four sibling repos, report discrepancies, then (on approval) fix them.

## Workflow

### 1. Verify sibling repos are present

For each of `../vela`, `../vela-nova`, `../vela-common-go`, `../vela-common-ts`:
- Confirm the directory exists.
- Run `git -C <path> describe --tags --always` and `git -C <path> rev-parse --abbrev-ref HEAD` to record the current ref.
- If any repo is missing, stop and tell the user which one.

Record the four refs — you will quote them in the findings report.

### 2. Flag version mismatches

Extract version strings from the docs:
- `CLAUDE.md` declares `**Current version: vX.Y.Z**` and states all Docker images are tagged with that version.
- `dockerfiles/docker-compose.yml` pins the default image tag (`${VELA_IMAGE_TAG:-vX.Y.Z}`).
- `README.md`, `dockerfiles/README.md`, `dockerfiles/.env.dev`, and the docs pages (titles, `go.mod` snippets, release links) may also pin a version.

Compare those declarations to the refs recorded in step 1. If any sibling repo is checked out at a different tag/branch than the docs claim, list it as a finding. Do not silently "fix" the version string — the mismatch itself is information the user needs.

### 3. Enumerate all markdown files in scope

Use Glob for `**/*.md`, excluding `.git`, `node_modules`, and any `.claude/` files. Expected set at time of writing:
- `README.md`
- `CLAUDE.md`
- `docs/1_summary.md`
- `docs/2_private-transfer-app.md`
- `docs/3_typescript-client.md`
- `docs/4_trigger-contract-app.md`
- `dockerfiles/README.md`

If Glob returns files you don't recognize, audit them too.

Also audit the non-markdown local environment files:
- `dockerfiles/docker-compose.yml` — image tags, and the env vars passed to each service compared with `../vela/dockerfiles/docker-compose.yml` at the matching tag (new config vars must be passed through).
- `dockerfiles/.env.dev` — compared with `../vela/dockerfiles/.env.dev` / `.env.template`. If reading it is denied by permissions, say so and list what the user must check manually.

### 4. Audit each doc against the sibling repos

For every file, read it end-to-end and check each factual claim against the relevant sibling repo. Claims worth verifying include (non-exhaustive):

**WASM interface (check against `vela-common-go/wasm/types/` and the `vela-nova` exports):**
- Exported function names and signatures (`deploy`, `deposit`, `process_request`, `trusted_request`, etc.) — the authoritative list is `../vela/docs/design/WASM_HOST_ABI.md` ("Exports the guest must provide")
- `requestType` routing values and what each maps to
- Struct field names, types, and ordering for `Address`, `Uint256`, `PlainEvent`, `Withdrawal`, `ProcessResult`, and any other referenced types
- Optional vs required fields (e.g. `ProcessResult.Report`)

**Core platform (check against `vela/`):**
- Manager / Executor / deployer / authority-service behavior
- Smart contract names, functions, events, and addresses. The dev contract addresses in `dockerfiles/README.md` and `docs/2_private-transfer-app.md` follow the deploy order in `../vela/contracts/scripts/deploy/all.ts`; confirm them from the deployer log of a fresh run (`docker logs vela-skit-deployer`) — a log saying "already deployed … Skipping" means the volumes are stale and the addresses are not current
- Changelogs: `../vela/CHANGELOG.md`, `../vela-nova/CHANGELOG.md`, `../vela-common-go/CHANGELOG.md` list breaking changes per release — use them as a checklist of what to look for
- Crypto primitives and key-derivation flows
- Environment variables, especially `EXECUTOR_KEYSET_RECOVERY_TYPE` and anything referenced in `dockerfiles/.env.dev`
- Container names, volume prefixes (`vela-skit-*`), network range (`10.10.40.0/24`), exposed ports (`8545`, `8081`, `8000`)
- Image names and tags (`horizen/cce-*`)

**TypeScript client (check against `vela-common-ts/`):**
- Class name (`VelaClient` — renamed from `HorizenCCEClient` in v0.0.25)
- Package name (`@horizen/vela-common-ts`, see its `package.json` — renamed from `horizen-cce-common-ts`)
- Public API surface, method names and signatures
- Key-derivation and encryption flows
- Subgraph query shapes

**Naming:**
- "Vela" / "Vela CCE" branding (not "Horizen CCE" / "horizen-pes", which were renamed in v0.0.25)
- Docker image path `horizen/cce-*` is still correct — do NOT rebrand this one.

For every discrepancy, record:
- File and line number (`README.md:42`)
- What the doc says
- What the source code says
- Which sibling repo + path + symbol backs up the correction

### 5. Report findings, wait for approval

Present findings grouped by file, with a clear summary at the top:
- Number of files audited
- Number of discrepancies found
- Version-string mismatches (if any) called out first

Do NOT edit anything yet. Ask the user to approve — either wholesale ("apply all"), selectively ("apply #1, #3, #5"), or case-by-case.

If there are zero findings, say so and stop. Don't invent work.

### 6. Apply approved edits

Use `Edit` (not `Write`) for targeted changes. Preserve surrounding formatting, headings, and code-fence languages. After edits:
- Re-read each modified file to confirm the change landed correctly.
- Summarize what changed in one or two sentences.

## Rules

- **Never modify the sibling repos.** They are read-only sources of truth.
- **Never guess a fact that isn't in the source.** If a doc makes a claim you cannot verify in the sibling repos, flag it as "unverifiable" in the findings rather than rewriting it.
- **Follow the project's naming convention:** "Vela" in prose, `horizen/cce-*` in Docker image paths. See `CLAUDE.md` "Naming Conventions".
- **Don't reorder or restructure docs** unless the user explicitly asks. Audit is about factual accuracy, not rewriting.
- **Version strings are findings, not auto-fixes.** Report mismatches; let the user decide whether to bump the docs or check out a different ref in the sibling repos.
- **Respect the root `CLAUDE.md`:** it is itself in scope for the audit (its "Key Architecture Facts" section is a prime target for drift).
