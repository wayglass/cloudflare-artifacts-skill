# REST control plane

Source: [REST API](https://developers.cloudflare.com/artifacts/api/rest-api/),
[Authentication](https://developers.cloudflare.com/artifacts/guides/authentication/),
[Data localization](https://developers.cloudflare.com/artifacts/guides/data-localization/),
[Wrangler commands](https://developers.cloudflare.com/artifacts/api/wrangler/).

## Setup

```sh
export ACCOUNT_ID=...                 # Cloudflare account ID
export CLOUDFLARE_API_TOKEN=...       # API token with Artifacts Read and/or Artifacts Edit
export NS=default
export ACCT_URL="https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/artifacts"
export NS_URL="$ACCT_URL/namespaces/$NS"
auth=(--header "Authorization: Bearer $CLOUDFLARE_API_TOKEN")
json=(--header "Content-Type: application/json")
```

- **Artifacts > Read** covers read routes; **Artifacts > Edit** covers write
  routes.
- The API token authenticates REST only. Git needs a repo token. A repo token
  does not authenticate REST.
- Keep the API token in the one service that brokers access (usually a Worker,
  where the binding needs no token at all). Never ship it in a client binary.
- Responses use the Cloudflare v4 envelope
  `{ "result", "success", "errors": [{ "code", "message" }], "messages", "result_info"? }`.
  `blob`, `file` and `raw` return bytes on success and the envelope on error.

## Routes

All paths are relative to `/client/v4/accounts/$ACCOUNT_ID`.

| Method | Path | Purpose |
|---|---|---|
| POST | `/artifacts/namespaces` | Create namespace. Body `{ "namespace", "jurisdiction"?: "eu" \| "us" }` |
| GET | `/artifacts/namespaces?limit=&cursor=` | List namespaces |
| GET | `/artifacts/namespaces/:ns` | Get namespace |
| POST | `/artifacts/namespaces/:ns/repos` | Create repo. Body `{ "name", "description"?, "default_branch"?, "read_only"? }` |
| GET | `/artifacts/namespaces/:ns/repos?limit=&cursor=&search=&sort=&direction=` | List repos. `limit` default 50, max 200. `sort`: `created_at` (default), `updated_at`, `last_push_at`, `name`. `direction`: `desc` (default) or `asc` |
| GET | `/artifacts/namespaces/:ns/repos/:name` | Get repo, including `remote` |
| DELETE | `/artifacts/namespaces/:ns/repos/:name` | Delete repo. Returns `202 Accepted` with `{ "id" }` |
| POST | `/artifacts/namespaces/:ns/repos/:name/fork` | Fork `:name`. Body `{ "name", "description"?, "read_only"?, "default_branch_only"? }`. Result adds `objects` |
| POST | `/artifacts/namespaces/:ns/repos/:name/import` | Import into the **new** repo `:name`. Body `{ "url", "branch"?, "depth"?, "read_only"? }` |
| GET | `/artifacts/namespaces/:ns/repos/:name/log?ref=&limit=&offset=` | Commit history |
| GET | `/artifacts/namespaces/:ns/repos/:name/commit/:hash` | One commit |
| GET | `/artifacts/namespaces/:ns/repos/:name/tree/:hash` | One tree |
| GET | `/artifacts/namespaces/:ns/repos/:name/blob/:hash` | Raw blob bytes |
| GET | `/artifacts/namespaces/:ns/repos/:name/file?ref=&path=` | File bytes, `application/octet-stream` |
| GET | `/artifacts/namespaces/:ns/repos/:name/raw/:ref/:path` | File bytes with a sniffed `Content-Type` |
| GET | `/artifacts/namespaces/:ns/repos/:name/tokens?state=&per_page=&page=` | List tokens. `state`: `active` (default), `expired`, `revoked`, `all`. `per_page` default 30, max 100 |
| POST | `/artifacts/namespaces/:ns/tokens` | Mint token. Body `{ "repo", "scope"?, "ttl"? }` |
| DELETE | `/artifacts/namespaces/:ns/tokens/:id` | Revoke token by id |

Note the asymmetry: tokens are **listed** under the repo path but **created
and revoked** under the namespace path.

## Examples

Create a jurisdiction-bound namespace. This is permanent: the jurisdiction
cannot be changed later. Omitting `jurisdiction` makes an unrestricted
namespace.

```sh
curl -sS -X POST "$ACCT_URL/namespaces" "${auth[@]}" "${json[@]}" \
  --data '{"namespace":"agents-eu","jurisdiction":"eu"}'
```

Create a repo and capture the remote. Don't print the whole response; it
contains a token.

```sh
resp=$(curl -sS -X POST "$NS_URL/repos" "${auth[@]}" "${json[@]}" \
  --data '{"name":"trunk","default_branch":"main"}')
printf '%s' "$resp" | jq -e '.success' >/dev/null || { printf '%s' "$resp" | jq '.errors'; exit 1; }
REMOTE=$(printf '%s' "$resp" | jq -r '.result.remote')
```

Fork for one unit of work:

```sh
curl -sS -X POST "$NS_URL/repos/trunk/fork" "${auth[@]}" "${json[@]}" \
  --data '{"name":"trunk-agent7-s42","default_branch_only":true}'
```

Import a public repo as a baseline (private remotes fail with
`REMOTE_AUTH_REQUIRED`):

```sh
curl -sS -X POST "$NS_URL/repos/baseline/import" "${auth[@]}" "${json[@]}" \
  --data '{"url":"https://github.com/example/project","branch":"main","depth":100}'
```

The import can still be running when the call returns. Follow-up calls may
return `409 Conflict`; retry after a short delay.

Mint a 15-minute read token, then revoke it:

```sh
tok=$(curl -sS -X POST "$NS_URL/tokens" "${auth[@]}" "${json[@]}" \
  --data '{"repo":"trunk","scope":"read","ttl":900}')
ARTIFACTS_TOKEN=$(printf '%s' "$tok" | jq -r '.result.plaintext')
TOKEN_ID=$(printf '%s' "$tok" | jq -r '.result.id')
# ...use it...
curl -sS -X DELETE "$NS_URL/tokens/$TOKEN_ID" "${auth[@]}"
```

`scope` defaults to `write` and `ttl` to 86,400 seconds (min 60, max
31,536,000). Always pass both.

Read without cloning:

```sh
curl -sS "$NS_URL/repos/trunk/log?ref=main&limit=10" "${auth[@]}" | jq '.result'
curl -sS "$NS_URL/repos/trunk/raw/main/README.md" "${auth[@]}"
```

## Response types (abridged)

```ts
type Jurisdiction = "eu" | "us";
interface RepoInfo {
  id: string; name: string; description: string | null; default_branch: string;
  created_at: string; updated_at: string; last_push_at: string | null;
  source: string | null; read_only: boolean;
}
interface RepoWithRemote extends RepoInfo { remote: string }          // get, list
interface CreateRepoResult {                                          // create, import
  id: string; name: string; description: string | null;
  default_branch: string; remote: string; token: string;
}
interface ForkRepoResult extends CreateRepoResult { objects: number } // fork
interface CreateTokenResult { id: string; plaintext: string; scope: "read" | "write"; expires_at: string }
interface TokenInfo { id: string; scope: "read" | "write"; state: "active" | "expired" | "revoked"; created_at: string; expires_at: string }
```

REST uses snake_case (`default_branch`, `read_only`, `expires_at`); the
binding uses camelCase (`defaultBranch`, `readOnly`, `expiresAt`). Map them
explicitly in shared code.

The initial `token` from create, fork and import is a bare string with expiry
only in its `?expires=` suffix and no id. To revoke it over REST, find its id
with the list-tokens route. The binding's `revokeToken` takes the plaintext
directly.

## Wrangler commands

```sh
npx wrangler artifacts namespaces list [--json]
npx wrangler artifacts namespaces get <NAME> [--json]
npx wrangler artifacts repos create <NAME> --namespace <NS> [--description ..] [--default-branch ..] [--read-only] [--json]
npx wrangler artifacts repos list --namespace <NS> [--json]
npx wrangler artifacts repos get <NAME> --namespace <NS> [--json]
npx wrangler artifacts repos delete <NAME> --namespace <NS> [--force]
npx wrangler artifacts repos issue-token <REPO> --namespace <NS> [--scope read|write] [--ttl <seconds>] [--json]
```

Wrangler has no namespace-create, fork, import or revoke command. Use REST or
the binding for those.
