---
name: sync-docs
description: Use when the user asks to sync, audit, update, refresh, or verify the project's documentation against the source code of the Vela repos on GitHub (vela, vela-nova, vela-common-go, vela-common-ts). Triggers on phrases like "update docs", "sync documentation", "check docs against repos", "refresh docs", "audit the docs", "docs out of date", "verify README against code". Asks which release tag to audit against, clones the repos at that tag, performs a full audit of every .md file in the repo, reports findings first, and only edits after the user approves.
---

# Documentation Sync Audit

This project (`vela-starterkit`) documents the Vela CCE platform but contains no application source code. The canonical source lives in four GitHub repositories:

| Repo | URL | Language | Role |
|------|-----|----------|------|
| `vela` | `https://github.com/HorizenOfficial/vela.git` | Go | Core platform: Manager, Executor, smart contracts |
| `vela-nova` | `https://github.com/HorizenOfficial/vela-nova.git` | TinyGo | Reference WASM app (private transfer) |
| `vela-common-go` | `https://github.com/HorizenOfficial/vela-common-go.git` | Go | Shared WASM types library |
| `vela-common-ts` | `https://github.com/HorizenOfficial/vela-common-ts.git` | TypeScript | Browser client library |

Your job when this skill is invoked: pick a release tag with the user, clone the four repos at that tag, audit every markdown file in this repo against them, report discrepancies, then (on approval) fix them.

Always use fresh clones from GitHub, never local checkouts of these repos that may exist on the machine (they can hold uncommitted or unreleased changes).

In the rest of this file, `$SRC` is the directory the repos are cloned into, so `$SRC/vela` is the `vela` repo at the chosen tag.

## Workflow

### 1. Choose the version

1. Read the version the docs declare: `**Current version: vX.Y.Z**` in `CLAUDE.md`.
2. For each of the four repos, find the latest release tag (only `vX.Y.Z`, skipping pre-release tags such as `-snapshot1` or `-rc1`):
   ```bash
   git ls-remote --tags --refs --sort=-v:refname https://github.com/HorizenOfficial/<repo>.git \
     | grep -E 'refs/tags/v[0-9]+\.[0-9]+\.[0-9]+$' | head -1
   ```
   If `ls-remote` fails (network or auth), stop and tell the user which repo could not be reached.
3. Ask the user which tag to audit against with `AskUserQuestion`, offering:
   - the latest release tag (recommended),
   - the version declared in `CLAUDE.md`, if different,
   - and "Other" for any tag or branch they type.

   The four repos are released together under the same tag. If their latest tags differ, say so in the question and let the user pick a tag for each repo.
4. Check the chosen tag exists in every repo (`git ls-remote --tags <url> <tag>`). If one is missing, stop and ask.

### 2. Clone the repos

Clone into a new, empty directory in the session scratchpad (if there is no scratchpad, use `mktemp -d`):

```bash
SRC=<scratchpad>/vela-src-<tag>
mkdir -p "$SRC"
for r in vela vela-nova vela-common-go vela-common-ts; do
  git clone --quiet --depth 1 --branch <tag> https://github.com/HorizenOfficial/$r.git "$SRC/$r"
done
```

Then record each repo's ref with `git -C "$SRC/<repo>" describe --tags --always` and `git -C "$SRC/<repo>" rev-parse --short HEAD`. You will quote them in the findings report.

The clones are read-only sources: read them, but don't run their builds, scripts, or tests.

### 3. Flag version mismatches

Extract version strings from the docs:
- `CLAUDE.md` declares `**Current version: vX.Y.Z**` and states all Docker images are tagged with that version.
- `dockerfiles/docker-compose.yml` pins the default image tag (`${VELA_IMAGE_TAG:-vX.Y.Z}`).
- `README.md`, `dockerfiles/README.md`, `dockerfiles/.env.dev`, and the docs pages (titles, `go.mod` snippets, release links) may also pin a version.

Compare those declarations to the tag chosen in step 1. List every mismatch as a finding. Do not silently "fix" the version string — the mismatch itself is information the user needs.

### 4. Enumerate all markdown files in scope

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
- `dockerfiles/docker-compose.yml` — image tags, and the env vars passed to each service compared with `$SRC/vela/dockerfiles/docker-compose.yml` (new config vars must be passed through).
- `dockerfiles/.env.dev` — compared with `$SRC/vela/dockerfiles/.env.dev` / `.env.template`. If reading it is denied by permissions, say so and list what the user must check manually.

### 5. Audit each doc against the repos

For every file, read it end-to-end and check each factual claim against the relevant repo. Claims worth verifying include (non-exhaustive):

**WASM interface (check against `$SRC/vela-common-go/wasm/types/` and the `$SRC/vela-nova` exports):**
- Exported function names and signatures (`deploy`, `deposit`, `process_request`, `trusted_request`, etc.) — the authoritative list is `$SRC/vela/docs/design/WASM_HOST_ABI.md` ("Exports the guest must provide")
- `requestType` routing values and what each maps to
- Struct field names, types, and ordering for `Address`, `Uint256`, `PlainEvent`, `Withdrawal`, `ProcessResult`, and any other referenced types
- Optional vs required fields (e.g. `ProcessResult.Report`)

**Core platform (check against `$SRC/vela`):**
- Manager / Executor / deployer / authority-service behavior
- Smart contract names, functions, events, and addresses. The dev contract addresses in `dockerfiles/README.md` and `docs/2_private-transfer-app.md` follow the deploy order in `$SRC/vela/contracts/scripts/deploy/all.ts`; confirm them from the deployer log of a fresh run (`docker logs vela-skit-deployer`) — a log saying "already deployed … Skipping" means the volumes are stale and the addresses are not current
- Changelogs: `$SRC/vela/CHANGELOG.md`, `$SRC/vela-nova/CHANGELOG.md`, `$SRC/vela-common-go/CHANGELOG.md` list breaking changes per release — use them as a checklist of what to look for
- Crypto primitives and key-derivation flows
- Environment variables, especially `EXECUTOR_KEYSET_RECOVERY_TYPE` and anything referenced in `dockerfiles/.env.dev`
- Container names, volume prefixes (`vela-skit-*`), network range (`10.10.40.0/24`), exposed ports (`8545`, `8081`, `8000`)
- Image names and tags (`horizen/cce-*`)

**TypeScript client (check against `$SRC/vela-common-ts`):**
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
- Which repo + path + symbol backs up the correction (paths relative to the repo root, e.g. `vela-common-go/wasm/types/result.go`, not the scratchpad path)

### 6. Report findings, wait for approval

Present findings grouped by file, with a clear summary at the top:
- The tag audited against and the four refs recorded in step 2
- Number of files audited
- Number of discrepancies found
- Version-string mismatches (if any) called out first

Do NOT edit anything yet. Ask the user to approve — either wholesale ("apply all"), selectively ("apply #1, #3, #5"), or case-by-case.

If there are zero findings, say so and stop. Don't invent work.

### 7. Apply approved edits

Use `Edit` (not `Write`) for targeted changes. Preserve surrounding formatting, headings, and code-fence languages. After edits:
- Re-read each modified file to confirm the change landed correctly.
- Summarize what changed in one or two sentences.

## Rules

- **Never modify or push to the Vela repos.** The clones are read-only sources of truth.
- **Never guess a fact that isn't in the source.** If a doc makes a claim you cannot verify in the repos, flag it as "unverifiable" in the findings rather than rewriting it.
- **Follow the project's naming convention:** "Vela" in prose, `horizen/cce-*` in Docker image paths. See `CLAUDE.md` "Naming Conventions".
- **Don't reorder or restructure docs** unless the user explicitly asks. Audit is about factual accuracy, not rewriting.
- **Version strings are findings, not auto-fixes.** Report mismatches; let the user decide whether to bump the docs or audit against a different tag.
- **Respect the root `CLAUDE.md`:** it is itself in scope for the audit (its "Key Architecture Facts" section is a prime target for drift).
