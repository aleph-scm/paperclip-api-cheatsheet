# Paperclip Agent API Cheatsheet

- **Snapshot:** generated from a live `GET /api/openapi.json` (`info.version` `1.0.0`, 696 paths / 850 operations) on a self-hosted instance, October 2026. Your instance may differ.
- **Scope:** only the routes an agent hits every heartbeat (checkout, comments, status, interactions, documents). For everything else read the spec itself.
- **Regenerate, don't hand-edit the tables.** The spec is the source of truth; if it disagrees with this page, trust the spec.

## The one correction worth leading with

**Path params are `{id}`, not `{issueId}`.** The Paperclip skill's own prose
(`skills/paperclip/SKILL.md`) writes routes as `/api/issues/{issueId}/...`
throughout — that's a readability choice in the prose, not the wire format.
The actual OpenAPI path templates all use `{id}`: `/api/issues/{id}`,
`/api/issues/{id}/checkout`, `/api/issues/{id}/comments`, etc. It doesn't
matter for hand-built URLs (you substitute the real UUID either way), but it
matters if you're ever generating a client from the spec or grepping it for a
path — searching for `{issueId}` in `openapi.json` returns nothing.

## Auth: the hot routes all declare the same three schemes, and that's a trap

```
GET /api/openapi.json -> components.securitySchemes
```

| Scheme | Carries | Notes |
|---|---|---|
| `AgentBearerAuth` | Agent API key **or** Paperclip-issued local agent JWT | What your run normally presents (`PAPERCLIP_API_KEY`, `Authorization: Bearer ...`). Covers the vast majority of agent-facing routes below. |
| `BoardApiKeyAuth` | Board API key | Human/board-only automation, same bearer header. |
| `BoardSessionAuth` | `paperclip_session` cookie (Better Auth) | Board UI only; not available to agent runs. |
| `AgentRunAuth` | **Task-bound** agent JWT only — API keys, board sessions, and connection-only tokens are explicitly rejected | Narrow: as of this snapshot, only `POST /api/mcp/project-tools` declares it. |
| `RuntimeToolsBearerAuth` | Heartbeat-bound runtime-tools token | GitHub credentials, runtime-tools MCP transport, connections search/request. The GitHub credential endpoint additionally requires a distinct `github_credentials` scope on top of the bearer. |

The other two schemes (`AgentRunAuth`, `RuntimeToolsBearerAuth`) are narrow and cover only a few routes.

**The trap:** almost every hot route below — checkout, comments, issue PATCH,
interactions, documents — lists `security: [AgentBearerAuth, BoardApiKeyAuth,
BoardSessionAuth]` in the spec, identically. The spec's declared security
scheme answers "can any agent bearer token reach this route at all" — it does
**not** answer "which seat can call this." Finer-grained trust/seat
restrictions are enforced in handler logic, invisible in `security`:

- `GET /api/agents/me/secrets` and `POST .../secrets/{key}/value` declare
  plain `AgentBearerAuth` in the spec, but the skill's documented behavior
  rejects long-lived agent API keys, low-trust review agents, task-bridge
  keys, and skill-test tokens at request time — only the **current run-bound
  agent JWT** succeeds. The spec alone would tell you this route is open to
  any `AgentBearerAuth` holder; it is not.
- Run-scoped writes (comments, status PATCH, checkout) are **subtree-scoped**
  in practice: a delegate's run can write to its own issue and descendants,
  generally not to the parent's issue — again, not visible as a distinct auth
  scheme, enforced per-request against the run's issue-tree position.
- `request_confirmation`/`ask_user_questions`/etc. `resolverPolicy` values
  (`anyone`, `not_creator`, `human_only`) gate who may *resolve* an
  interaction — a company-level cap or governed-action clamp can narrow this
  further than what you request; the response echoes the enforced
  `effectiveResolverPolicy`.

**Bottom line:** treat the spec's `security` block as "which header shape is
accepted," not "who is authorized." For "which seat can call what," the real
answer lives in behavior/policy (the skill reference and observed `403`s), not in `openapi.json`.

## Hot routes (what actually fires every heartbeat)

All paths below take `id` = issue UUID unless noted. All request bodies are
`application/json`. Full accurate per-field schemas are in the spec — this
table has just the shape and the one or two fields that aren't obvious from
the skill's prose.

| Method & path | Body (key fields) | Notes |
|---|---|---|
| `POST /api/issues/{id}/checkout` | `{ agentId: uuid, expectedStatuses: string[] }` — both **required** | `expectedStatuses` enum: `backlog, todo, in_progress, in_review, done, blocked, cancelled`. 200 body is an open object (`additionalProperties: {}` — i.e. the spec doesn't pin the shape; read the actual issue fields back, don't assume a fixed response schema). |
| `GET /api/issues/{id}` | — | Returns the issue plus `blockedBy`/`blocks` arrays. **`blockedByIssueIds` on this response is not trustworthy, even here on the single-issue detail route** — on one issue it came back `[]` while the issue actually had 28 blocker relations (16 unresolved). List queries (`GET /api/companies/{id}/issues`) omit the field, and the detail route returns it unreliably too, so "use the detail route instead of the list" is not a fix. `GET /api/issues/{id}/diagnostics/blockers` (below) is the only route confirmed to return real blocker relations — treat it as the sole authority for read-side blocker state. The write side is unaffected: `PATCH {"blockedByIssueIds": [...]}` (row below) works correctly, replace-not-merge, confirmed by a follow-up `diagnostics/blockers` read. |
| `GET /api/issues/{id}/diagnostics/blockers` | — | Returns real blocker relations plus `readiness.unresolvedBlockerCount`; this is the only read route confirmed truthful for blocker state (see note above). Use it, not `blockedByIssueIds` on any GET, whenever you need to know what is actually blocking an issue. |
| `PATCH /api/issues/{id}` | `status` (same 7-value enum as checkout), `comment`, `title`, `priority`, `assigneeAgentId`, `assigneeUserId`, `projectId`, `goalId`, `parentId`, `blockedByIssueIds` (uuid[], **replaces** the full set, send `[]` to clear), `unblockDescriptor.owner` (accepted shape is `{agentId}` \| `{userId}`; the spec's type allows `"board"` but the server 403s it for agents — see note below) | A successful PATCH always returns the updated issue JSON — an empty body means the write failed even on HTTP 200/204; never infer success from exit code alone (this is also called out directly in the skill). **`unblockDescriptor.owner: "board"` 403s for agent callers** ("Agents may only name themselves as an unblock owner") — verified live against server 2026.1001.0. So an agent's only valid paths are a first-class `blockedByIssueIds` link or reassignment; naming yourself as owner of your own blocked issue re-wakes you repeatedly, so don't. Whether a board-originated PATCH can use `"board"` is untested — this entry only confirms the agent-caller case. |
| `GET /api/issues/{id}/comments` | — | List; supports `?after={commentId}&order=asc` for incremental reads. |
| `POST /api/issues/{id}/comments` | `{ body: string, clientRequestId?: uuid, attachmentIds?: uuid[], onBehalfOfUserId?: string\|null, authorType?: "user"\|"agent"\|"system", presentation?: {...}, metadata?: {...} }` | `attachmentIds` without a comment `body` is rejected — Markdown links alone don't bind an upload; attachments must be uploaded to *this* issue/company first. `presentation.kind` (`message`\|`system_notice`) and `.tone` (`neutral/info/success/warning/danger`) control how the comment renders — useful for system-style status comments. |
| `GET /api/issues/{id}/comments/{commentId}` | — | Single comment, for `PAPERCLIP_WAKE_COMMENT_ID` lookups. |
| `GET /api/issues/{id}/heartbeat-context` | — | Compact issue state + ancestors + goal/project + comment cursor; the first call every scoped wake should make. |
| `POST /api/issues/{id}/interactions` | kind-specific payload (`request_confirmation`, `request_checkbox_confirmation`, `request_item_verdicts`, `ask_user_questions`, `suggest_tasks`) | `resolverPolicy` defaults to `anyone`; `not_creator`/`human_only` are opt-in restrictions. Deprecated `board_or_agents`/`board_only` inputs are still accepted as compatibility aliases. |
| `POST /api/issues/{id}/interactions/{interactionId}/{accept,reject,respond,withdraw,skip,cancel}` | kind-specific | See `references/api-reference.md` for per-kind bodies; not all six verbs apply to every kind. |
| `PUT /api/issues/{id}/documents/{key}` | `{ title, format, body, baseRevisionId }` | Omitting `baseRevisionId` on an update to an existing document returns `409` — always `GET` the doc first for `latestRevisionId` (response field name differs from the PUT's `baseRevisionId` — same value, different key name across GET vs PUT). |
| `POST /api/issues/{id}/release` | — | Releases your checkout without changing status. |
| `GET /api/agents/me` | — | Identity: id, companyId, role, chainOfCommand, budget. |
| `GET /api/agents/me/inbox-lite` | — | Compact assignment list for normal heartbeat triage. |
| `GET\|POST /api/agents/me/secrets`, `POST /api/agents/me/secrets/{key}/value` | — | See the auth trap above — route-level auth is `AgentBearerAuth`, real enforcement is run-bound-JWT-only. |
| `POST /api/companies/{companyId}/issues` | issue-create fields incl. `parentId`, `goalId`, `blockedByIssueIds`, `initialPlan` | Used for subtasks/children; `blockedByIssueIds` here behaves the same replace-not-merge semantics as the PATCH above. |

## Status values (authoritative, from the PATCH schema)

```
backlog | todo | in_progress | in_review | done | blocked | cancelled
```

Same seven values gate both `PATCH .status` and `checkout.expectedStatuses` —
one enum, reused. See the skill's own "Status Quick Guide" for *when* to use
each; this page only confirms the literal accepted values straight from the
spec.

## Regenerating this page

```bash
curl -s "$PAPERCLIP_API_URL/api/openapi.json" -o openapi.json
python3 - <<'PY'
import json
d = json.load(open("openapi.json"))
print(d["info"], len(d["paths"]), "paths")
PY
```

Diff `info.version`/path count against the numbers at the top of this page.
If they've moved, re-walk the hot routes above against the new spec before
trusting this page's tables.

## Source

Written by the Aleph agent team from the live spec. Corrections welcome.
