---
name: cloudflare-artifacts
description: Build on Cloudflare Artifacts (beta), Git-compatible repositories for agents, through the Workers binding, the REST API and plain git over HTTPS. Use when creating, forking or importing Artifacts repos, minting or revoking repo tokens, cloning or pushing to *.artifacts.cloudflare.net remotes, choosing namespaces and jurisdictions, consuming Artifacts events from Queues or Workflows, or planning around Artifacts limits and pricing. Not for R2 object storage or for Workers Builds connected to an external Git host.
license: "MIT"
---

# Cloudflare Artifacts

Artifacts is Cloudflare's Git-compatible, versioned storage. Each repo is an
isolated Git smart-HTTP server with its own remote URL, refs, tokens and
lifecycle. It is in open beta; re-check the
[changelog](https://developers.cloudflare.com/artifacts/platform/changelog/)
before relying on anything below.

Verified against the live docs, `wrangler` 4.149.0 and
`@cloudflare/workers-types` 5.20261012.1 on October 11, 2026; token format
and handle disposal re-checked against a live namespace on October 12. Treat the
generated `worker-configuration.d.ts` in the project as the source of truth for
binding types.

## When to use it

- Many isolated work units (one per agent, session, user or task) that each
  need real Git history. Cloudflare's guidance: if you have 10,000 agents,
  create 10,000 repos.
- Agents or sandboxes that already speak `git` and need a remote you can create,
  fork and scope from code.
- Handing someone a snapshot of in-progress work: fork the repo and give them
  the new remote and a token.

Do not use it as a shared branch that many autonomous agents push to. Concurrent
pushes to one ref are rejected as non-fast-forward and agents fall into retry
loops. Give each agent its own repo (usually a fork) and merge deliberately.

## Three interfaces, one repo

| Interface | Auth | Use for |
|---|---|---|
| Workers binding `env.ARTIFACTS` | The binding itself, no token | Create, import, fork, delete, read objects and files, mint and revoke tokens |
| REST API under `/accounts/$ACCOUNT_ID/artifacts` | Cloudflare API token, **Artifacts Read** or **Artifacts Edit** | Same control plane from outside Workers |
| Git over HTTPS | Repo-scoped token, `read` or `write` | Clone, fetch, pull, push: the only way to write commits |

The binding and REST API are control plane only. **There is no diff, merge or
commit-write API, and Workers cannot run the `git` binary.** Writes happen in a
Git client: a CLI or server with `git`, a Sandbox or Container, or
`isomorphic-git` with an in-memory filesystem inside a Worker. See
[references/patterns.md](references/patterns.md).

## Rules that prevent most mistakes

1. **Namespaces and jurisdiction.** A namespace holds repos. Its jurisdiction
   (`eu`, `us`, or unrestricted when omitted) is set once at creation through
   `POST /accounts/$ACCOUNT_ID/artifacts/namespaces` and can never change.
   Creating a repo under a namespace name that does not exist yet creates the
   namespace implicitly, and the docs give no way to set a jurisdiction on that
   path, so create jurisdiction-bound namespaces explicitly first. Get approval
   before creating one; a mistake cannot be undone.
2. **Use the returned `remote`.** The format is
   `https://<ACCOUNT_ID>.artifacts.cloudflare.net/git/<namespace>/<repo>.git`,
   but read it from the create, fork, import, get or `info()` response rather
   than building it.
3. **Keep tokens out of URLs and off disk.** Pass them per command:
   `git -c http.extraHeader="Authorization: Bearer $ARTIFACTS_TOKEN" clone "$REMOTE"`.
   Never `git remote add` a URL that contains a token; it lands in
   `.git/config`. The docs also allow `https://x:<secret>@host/...` for
   one-off commands. Prefer the header.
4. **Mint short-lived, narrow tokens per session.** Format is
   `art_v2_<letter>_<40 hex>?expires=<unix seconds>` (the docs still show
   `art_v1_<40 hex>`; match both). `read` allows clone, fetch and
   pull. `write` adds push. Scope defaults to `write` and TTL to 86,400 s
   (min 60, max 31,536,000), so always pass both. One token covers one repo.
   Never hand every agent one long-lived write token.
5. **Create, fork and import return a token.** Treat it as a write-capable
   secret. Don't log it or return it to an untrusted caller. Revoke it with
   `repo.revokeToken(plaintext)` if you will mint scoped tokens instead.
6. **Push is protocol v1 `receive-pack` only.** Clone and fetch work over v1
   and v2, and shallow and deepen fetches work. Partial clone (`--filter`) and
   `include-tag` are not supported. Stock `git push` works as is.
7. **Dispose repo handles.** `env.ARTIFACTS.get(name)` returns an RPC stub,
   not metadata. Declare it with `using`, and call `await repo.info()` for
   `remote`, `defaultBranch` and the rest.
8. **New repos may not be ready.** `get()` throws `CREATE_IN_PROGRESS`,
   `IMPORT_IN_PROGRESS` or `FORK_IN_PROGRESS`, and REST returns `409` while an
   import or fork runs. Retry with backoff on those codes only.

## Minimal flow

```ts
// wrangler.jsonc: "artifacts": [{ "binding": "ARTIFACTS", "namespace": "agents" }]
// Authorize the caller before this runs; it hands out a write token.
async function startSession(env: Env, agentId: string, sessionId: string) {
  using base = await env.ARTIFACTS.get("baseline"); // a reviewed, trusted repo
  // Names: letters, digits, ".", "_", "-"; unique within the namespace.
  const fork = await base.fork(`${agentId}-${sessionId}`, { defaultBranchOnly: true });
  using repo = await env.ARTIFACTS.get(fork.name); // retry on FORK_IN_PROGRESS
  await repo.revokeToken(fork.token);               // drop the initial token
  const token = await repo.createToken("write", 3600);
  return { remote: fork.remote, token: token.plaintext, expiresAt: token.expiresAt };
}
```

```sh
git -c http.extraHeader="Authorization: Bearer $ARTIFACTS_TOKEN" clone "$REMOTE" work
git -C work -c http.extraHeader="Authorization: Bearer $ARTIFACTS_TOKEN" push origin HEAD:main
```

## References

Read only the file the task needs.

- [references/workers-binding.md](references/workers-binding.md): wrangler
  config, every binding method with TypeScript, result types, local dev.
- [references/rest-api.md](references/rest-api.md): every REST route with
  curl, bodies, responses, pagination, and the Wrangler commands.
- [references/git-and-tokens.md](references/git-and-tokens.md): remotes,
  passing tokens safely (including from Go), protocol support, notes, token
  lifecycle.
- [references/events.md](references/events.md): Queues event subscriptions,
  payload shapes, a consumer Worker, and Workflow push triggers.
- [references/patterns.md](references/patterns.md): diffing, merging and
  committing without those APIs; repo-per-agent layouts; changed-file
  detection.
- [references/limits-pricing-errors.md](references/limits-pricing-errors.md):
  limits, billing, error codes, and gotchas, including places where the docs
  contradict themselves.

## Authoritative sources

- [Artifacts docs index for LLMs](https://developers.cloudflare.com/artifacts/llms.txt).
  Any page is available as Markdown by appending `index.md`.
- [Workers binding](https://developers.cloudflare.com/artifacts/api/workers-binding/),
  [REST API](https://developers.cloudflare.com/artifacts/api/rest-api/),
  [Git protocol](https://developers.cloudflare.com/artifacts/api/git-protocol/),
  [Best practices](https://developers.cloudflare.com/artifacts/concepts/best-practices/)
- [Launch post](https://blog.cloudflare.com/artifacts-git-for-agents-beta/)
  (April 2026; some details predate the open beta)
