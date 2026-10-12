---
name: 1panel-write
description: Perform 1Panel write operations (create, update, delete, start, stop, restart) on containers, websites, apps, cronjobs and related resources. Use when the user asks to modify or remove something managed by 1Panel and the official 1panel-skills read-only actions cannot do it.
---

# 1Panel write operations

Prerequisite: the official `1Panel-dev/1Panel-skills` skill. Its actions are read-only; each
mutation endpoint exists only as a `reservedMutations` entry (method, path, one-line note). This
skill drives those endpoints through the CLI's generic `request` passthrough.

## Nothing here is mechanically enforced

`request` is a raw signed passthrough: any method, any path, any body. Nothing in either skill
checks it, and no API-key scope is fine-grained enough to cover individual endpoints. The only
real fences are the human confirmation below and the user's own decision. Treat every rule here as
required behaviour with no safety net behind it.

## A write needs a version-matched spec

Bodies are not in the official skill. They come from the panel's own spec, and only for the
version the panel reports — a shape from any other version is a guess.

Before the first write of a session, and again after any panel upgrade:

1. Read the version the panel reports:
   `POST /api/v2/core/settings/search/base` → `dto.SettingBaseInfo.systemVersion`
2. Equal to the `panel version` in [references/mutations.md](references/mutations.md) → that table
   is this panel's shapes; go on.
3. Not equal, or the read fails → fetch that version's spec and regenerate, then compare the
   `shapes digest` the generator prints against the one in the table. Regenerate to a temp path:
   the shipped `mutations.md` is the maintainer's artifact, not something a live session rewrites.

   ```bash
   curl -sSL -o /tmp/sw.json \
     https://raw.githubusercontent.com/1Panel-dev/1Panel/<systemVersion>/core/cmd/server/docs/swagger.json
   node <this-skill>/references/generate.mjs /tmp/sw.json \
     --version=<systemVersion> --out=/tmp/mutations-<systemVersion>.md
   ```

   - **same digest** → the mutation shapes did not change between those versions; the shipped
     table still applies and only its `panel version` line is stale.
   - **different digest** → read the write's fields from `/tmp/mutations-<systemVersion>.md` and
     re-confirm them before running it.

Writing is never allowed on unverified shapes. There is no fallback to memory and no fallback to
another version's table; regenerating is the whole recovery path.

## Outcomes, and what the user hears

| Outcome | What you say |
| --- | --- |
| versions equal | "Panel v2.3.2, table v2.3.2 — shapes verified." One line, no ceremony. |
| differs, same digest | "Panel is v2.4.0; the table is for v2.3.2. Regenerated from the v2.4.0 spec — mutation shapes are identical, so the table still applies." Then continue. |
| differs, different digest | "Panel is v2.4.0, table was v2.3.2. Regenerated: <name what actually changed>." Then show this write's fields as read from the new table and confirm them before running. |
| version unreadable | Stop there. "Could not read `systemVersion` (<the error>). Writes are paused until the panel answers this API key." |
| spec unfetchable | Stop there. "Panel is v2.4.0 but the v2.4.0 spec is not reachable (<the error>). Writes are paused — give me that version's `swagger.json` and I will regenerate from it." |

The last two are the only refusals, and each one names the missing thing and what would unblock it.
Never soften either into a proceed, and never offer to run it "from memory" or from the older
table — that is exactly the failure this gate exists to prevent. Reads (`run`, `actions`, `modules`)
are never gated by any of this.

The live Swagger UI is **not** usable here: `init/router/router.go` wraps `/1panel/swagger/*` in
`SessionAuth()`, which wants a web session cookie — an API key does not satisfy it, and the route
is registered before the global API-key middleware so it never sees one. The tagged spec is the
same content, reachable without logging in. Shape drift between releases is real but small and
additive, which is why a version bump calls for a one-command regeneration rather than a stop.

## Reading

Reads go through declared actions, never through `request`:

```bash
node <1panel-skills>/dist/scripts/cli.js run containers listContainers
node <1panel-skills>/dist/scripts/cli.js run websites searchWebsites --input-json '{"name":"example"}'
node <1panel-skills>/dist/scripts/cli.js run nodes listAllSimpleNodes
```

`node <1panel-skills>/dist/scripts/cli.js modules` lists modules; `... actions <module>` lists what
is currently writable.

## Writing

```bash
node <1panel-skills>/dist/scripts/cli.js request <METHOD> <PATH> --body-json '<JSON>' [--node <nodeId>]
```

Take `<METHOD>` and `<PATH>` verbatim from a `reservedMutations` entry. If an operation is not
declared there it is out of scope — `prune`, `flush` and similar are absent by design, not "not yet
found". Do not hunt for a working path and do not reach for HTTP directly.

Before running a write:

1. **Read the target.** One `run` call gives the exact names/ids, the current state, and the node.
   It catches pointless mutations (restarting a stopped container, deleting a name that does not
   exist) and it is the only way to name the target accurately in step 3.
2. **Get the body from the version-matched spec, not from memory.** Then show the user the concrete
   body, with every field's provenance, and let them accept or correct it:

   ```
   POST /api/v2/containers/operate   (shape from references/mutations.md @ v2.3.2, version matched)
     names      = ["web"]       verified (containers/search returned this name)
     operation  = "restart"     verified (enum: up/start/stop/restart/kill/pause/unpause/remove)
     taskID     =               dropped  (optional, not needed)
   Unverified fields: none.
   Confirm: 1) run as drafted  2) change a field  3) abort
   ```

   Label each field `verified (source)`, `unverified`, or `dropped`. If a field is required and you
   cannot confirm it, say which one and stop. Watch for paths whose URL says one thing and whose
   body says another — the table calls these out per endpoint.
3. **Decide the destructive flags out loud.** See below.
4. **For irreversible writes, state exactly what will be destroyed** — the resource list and the
   cascade — and wait for an explicit go-ahead.
5. **Report** endpoint, status code, and the key result. A `200` with a `"code"` field in the body
   is still a failure; read the body.

## Destructive flags must be chosen, not defaulted

Some endpoints carry booleans that destroy something else as a side effect — deleting a website
can take its app, database and backups with it. `references/mutations.md` marks every such field
**cascade** on the endpoint it belongs to; that generated table is the current list, not a list
written down here.

Never set one of them yourself and never omit one silently. For each, say what it destroys and let
the user decide. Omitting a flag is a decision too, and it is theirs. This is why a vague "this may
also remove related resources" is not acceptable: the API names the consequences, so the request
can name them too.

## Irreversible operations

Reversible (still confirm with the user; a mistake is recoverable): creation, updates, HTTPS config,
monitor settings, cronjob enable/disable, and the start / stop / restart / upgrade / kill / pause /
unpause operations.

Irreversible — deleting data, or running something whose side effects you have not reviewed:

| Endpoint | Why |
| --- | --- |
| `POST /api/v2/websites/del` | deletes the website, plus whatever its flags say |
| `POST /api/v2/websites/ssl/update` | SSL material is replaced, not versioned |
| `POST /api/v2/apps/installed/op` with `operate: "delete"` | app removal, plus its flags |
| `POST /api/v2/apps/installed/params/update` | rewrites an installed app's configuration |
| `POST /api/v2/cronjobs/handle` | runs the job's own logic once; whatever it does, is done |
| `POST /api/v2/core/logs/clean` | deletes logs (`logType`: `login` or `operation`) |
| `POST /api/v2/hosts/monitor/clean` | deletes historical monitor data; takes no body |
| any `remove` / `delete` operation | container removal, etc. |

`POST /api/v2/core/licenses/bind/free` (the `nodes` module's `changeBind`) is declared by
`reservedMutations` but does **not** exist in the open-source build's spec — no license paths do.
Treat it as unavailable: do not attempt it and do not guess a body.

## Node targeting

`--node <id>` is appended as the `operateNode` query parameter. Resolve the id to a name with
`run nodes listAllSimpleNodes` first and put that name in the confirmation: a mistyped id is
otherwise invisible, and on a multi-node panel it means writing to the wrong machine. On Git Bash
use `--node=<id>`, or MSYS will rewrite the value.

## Diagnostics

- `{"code":401,"message":"API 接口密钥错误"}` — the signature is `md5("1panel" + key + timestamp)`,
  so the likely causes are a wrong key, an IP whitelist, or clock skew.
- Upstream may wire these mutations into real actions with typed inputs eventually. When
  `actions <module>` stops returning them under `reservedMutations`, use the real action.

## Files

- `references/mutations.md` — request shapes for the declared endpoints: required fields, enums,
  destructive flags. Generated from the spec of one panel version; valid only for that version.
- `references/generate.mjs` — regenerates the above from any version's `swagger.json`:
  `node generate.mjs <swagger.json> --version=<systemVersion> [--out=<path>]`. No dependencies.
  Maintaining it is the whole maintenance story: when a panel version ships, take that tag's spec,
  regenerate `mutations.md`, commit it. `generate.mjs` itself rarely changes. A live session
  regenerates to `--out=/tmp/...` instead, so it never rewrites the committed artifact.
