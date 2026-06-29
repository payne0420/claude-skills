---
name: forgejo
description: >-
  Interact with a self-hosted Forgejo instance (Gitea-compatible API) — manage
  repositories, files, branches, issues, pull requests, labels, milestones, and
  releases via the Forgejo REST API, or clone/push with git over HTTPS/SSH. Use
  whenever a task involves a Forgejo server or "my git server / my forge / my
  repos" on a self-hosted forge. Configure the instance URL, account, and a
  Personal Access Token via environment variables (see Setup) — every request,
  even reads, needs authentication.
---

# Forgejo (self-hosted) — agent skill

This skill lets you operate a self-hosted Forgejo forge. Forgejo is a Gitea fork; its
API is Gitea-compatible. Facts below were verified against the Forgejo **v15** Swagger
spec and source (June 2026). Endpoint paths are stable across v1.x of the API.

> **Configure it first.** This is a generic template — nothing is hardcoded. Set the
> environment variables in [Setup](#setup-one-time-do-this-first) to point at your own
> instance. Replace `forgejo.example.com` / `your-username` in any copied snippet, and
> **never commit a real token.**

## Instance constants (set via env — see Setup)

| Thing | Value |
|---|---|
| Web / clone-over-HTTPS base | `$FORGEJO_URL` (e.g. `https://forgejo.example.com`) |
| REST API base | `$FORGEJO_API` = `$FORGEJO_URL/api/v1` |
| Swagger UI / OpenAPI | `$FORGEJO_URL/api/swagger` · `$FORGEJO_URL/swagger.v1.json` |
| Account (owner) | `$FORGEJO_USER` |
| Git over SSH | `ssh://git@<host>:<ssh-port>/<owner>/<repo>.git` (port is per-instance; often `22` or `2222`) |
| Auth | Personal Access Token (PAT), required on **every** request |

**Two preconditions before anything works:**
1. **Network** — if your instance is LAN-only or behind a VPN (e.g. WireGuard with
   split-horizon DNS), it only resolves/answers when you're on that network. If DNS fails
   or you get connection-refused/timeout, you're likely off the network — stop and tell the
   user; don't retry blindly.
2. **TLS** — if TLS is terminated by your own nginx/reverse proxy with a cert from a
   private/internal CA, `curl`/`git` may reject it: point them at the CA bundle
   (`curl --cacert …`, `git config http.sslCAInfo …` or `CURL_CA_BUNDLE`). Only use
   `curl -k` / `GIT_SSL_NO_VERIFY=1` as a temporary diagnostic, never as the default.

## Setup (one-time, do this first)

Configure the instance via environment variables. **Keep the token out of source control** —
read it from your shell profile, a secrets manager, or a `.env` file that is gitignored.
Never paste a real token into a committed file.

```sh
export FORGEJO_URL="https://forgejo.example.com"      # your instance base URL
export FORGEJO_API="$FORGEJO_URL/api/v1"
export FORGEJO_USER="your-username"                    # the account that owns the token
export FORGEJO_TOKEN="${FORGEJO_TOKEN:?set your Forgejo PAT}"   # never hardcode; export it in your shell
```

### Creating a token
Create one — **don't** invent or guess it. Either:

- **Web UI:** `$FORGEJO_URL/user/settings/applications` → *Generate New Token* → name it,
  pick scopes, optionally restrict to specific repos. The token string is shown **once** —
  copy it immediately.
- **API (HTTP Basic auth — the one call that uses the password, not a token):**
  ```sh
  curl -fsSL -u "$FORGEJO_USER:PASSWORD" -H "Content-Type: application/json" \
    -d '{"name":"agent","scopes":["write:repository","write:issue","write:user"]}' \
    "$FORGEJO_API/users/$FORGEJO_USER/tokens"
  # → JSON with "sha1": that value IS the token. If 2FA is on, add: -H "X-Forgejo-OTP: 123456"
  ```
  Body keys: `name` (required), `scopes` (optional array), `repositories` (optional array of
  `{"owner","name"}` to lock the token to specific repos).

**Recommended scopes for a general repo+issue+PR agent: `write:repository`, `write:issue`, `write:user`.**
Forgejo gotcha (verified on v15): the `/user/*` endpoints — `GET /user` (identity),
`GET /user/repos` (list your repos), and `POST /user/repos` (**create a repo**) — are gated by the
**`user`** scope, *not* `repository`. Without a `user` scope you get `403` on those and can't create
repos or confirm identity. `write:user` covers all three (it implies read). PRs need no separate
scope (they ride on the repository/issue API). `write:X` implies read. Avoid `read:admin`/`write:admin`
and the catch-all `all`.

Narrower, lower-privilege alternative: `write:repository` + `write:issue` **only** — works for
operating on **existing** repos addressed as `/repos/{owner}/{repo}/…` (files, branches, issues, PRs,
releases), but **cannot** create repos or call any `/user` endpoint. Pair it with the *specific
repositories* restriction (only `read/write:repository` and `read/write:issue` allowed in that mode)
for the tightest blast radius.

## API conventions (read once)

- **Auth header:** `Authorization: token <TOKEN>` — the literal word **`token`** is required
  for a PAT. (`Authorization: Bearer <TOKEN>` is only for OAuth2-provider tokens.) You *can*
  pass `?token=…`/`?access_token=…` in the URL, but **don't** — it leaks into logs.
- **JSON:** send `Content-Type: application/json`.
- **`{index}`** in `/issues/{index}` and `/pulls/{index}` is the **per-repo sequential number**
  shown in the UI (e.g. `#42`), not a database id. Issues and PRs **share one counter** per repo,
  so a PR is also a valid issue index — you comment on a PR via `/issues/{index}/comments`.
  Listing issues returns **both** issues and PRs unless you filter `type=issues|pulls`.
- **File `content` is base64.** In create/update/delete-file bodies the `content` field **must be
  base64-encoded**; updates/deletes also require the file's current blob `sha`.
- **Pagination:** `?page=` (1-based) and `?limit=`. Responses carry `Link` and `x-total-count`
  headers; instance defaults at `GET /settings/api`.
- **Private instance:** a `404` often means "exists but you're not authorized" — check the token
  and its scopes before assuming the resource is missing.

### Reusable helper
```sh
# Authenticated JSON curl. Usage: fj <method?> <url> [curl-args…]
fj() { curl -fsSL -H "Authorization: token $FORGEJO_TOKEN" -H "Content-Type: application/json" "$@"; }

# Sanity check — who am I? (also confirms reachability + token)
fj "$FORGEJO_API/user" | jq '{login, id, is_admin}'
```

## REST recipes

### Repositories
```sh
# List my repos
fj "$FORGEJO_API/user/repos?page=1&limit=50" | jq -r '.[].full_name'

# Get one
fj "$FORGEJO_API/repos/$FORGEJO_USER/myrepo"

# Search
fj "$FORGEJO_API/repos/search?q=infra&limit=20" | jq -r '.data[].full_name'

# Create (only "name" required; auto_init makes a default branch so you can write files immediately)
# NOTE: repo creation + the /user/* reads below need the **user** scope (write:user), not repository.
fj -X POST "$FORGEJO_API/user/repos" \
  -d '{"name":"myrepo","description":"created by agent","private":true,"auto_init":true,"default_branch":"main"}'

# Branches
fj "$FORGEJO_API/repos/$FORGEJO_USER/myrepo/branches" | jq -r '.[].name'
fj -X POST "$FORGEJO_API/repos/$FORGEJO_USER/myrepo/branches" \
  -d '{"new_branch_name":"feature-x","old_branch_name":"main"}'
```

### Files (Contents API — content is base64; use `base64 -w0`)
```sh
OWNER=$FORGEJO_USER; REPO=myrepo

# Read a file (decode the base64 content)
fj "$FORGEJO_API/repos/$OWNER/$REPO/contents/README.md?ref=main" | jq -r '.content' | base64 -d

# Create a file
fj -X POST "$FORGEJO_API/repos/$OWNER/$REPO/contents/docs/new.md" \
  -d "$(jq -n --arg c "$(printf 'hello\n' | base64 -w0)" \
        '{content:$c, message:"add docs/new.md", branch:"main"}')"

# Update a file — needs the CURRENT blob sha from a GET first
SHA=$(fj "$FORGEJO_API/repos/$OWNER/$REPO/contents/README.md?ref=main" | jq -r '.sha')
fj -X PUT "$FORGEJO_API/repos/$OWNER/$REPO/contents/README.md" \
  -d "$(jq -n --arg c "$(printf 'updated\n' | base64 -w0)" --arg s "$SHA" \
        '{content:$c, sha:$s, message:"update README", branch:"main"}')"

# Delete a file (also needs sha)
fj -X DELETE "$FORGEJO_API/repos/$OWNER/$REPO/contents/old.md" \
  -d "$(jq -n --arg s "$SHA" '{sha:$s, message:"remove old.md", branch:"main"}')"
```
For an atomic multi-file commit, `POST …/contents` (no filepath) with a `ChangeFilesOptions`
`{ "files": [ {operation, path, content…}, … ], "message", "branch" }`. For many/large files
it's usually simpler to use git directly (below).

### Issues, comments, labels, milestones (scope: `write:issue`)
```sh
# List (filter to real issues, not PRs)
fj "$FORGEJO_API/repos/$OWNER/$REPO/issues?state=open&type=issues&labels=bug&limit=30"

# Create  (labels are integer label IDs, not names)
fj -X POST "$FORGEJO_API/repos/$OWNER/$REPO/issues" \
  -d '{"title":"Bug report","body":"Steps…","labels":[1],"assignees":["your-username"]}'

# Comment on issue #42 (same path works for a PR #42)
fj -X POST "$FORGEJO_API/repos/$OWNER/$REPO/issues/42/comments" -d '{"body":"On it."}'

# Labels / milestones
fj -X POST "$FORGEJO_API/repos/$OWNER/$REPO/labels"     -d '{"name":"bug","color":"#ee0701"}'
fj -X POST "$FORGEJO_API/repos/$OWNER/$REPO/milestones" -d '{"title":"v1.0","due_on":"2026-09-01T00:00:00Z"}'
```
Label `color` must be a valid hex like `#ee0701` (no spaces) — an invalid color returns `422`.

### Pull requests (scope: `write:repository`)
```sh
# Create  (head=source branch, base=target; head as "owner:branch" for cross-repo)
fj -X POST "$FORGEJO_API/repos/$OWNER/$REPO/pulls" \
  -d '{"head":"feature-x","base":"main","title":"Add feature","body":"Why/what"}'

# List / get
fj "$FORGEJO_API/repos/$OWNER/$REPO/pulls?state=open&limit=20" | jq -r '.[] | "\(.number) \(.title)"'
fj "$FORGEJO_API/repos/$OWNER/$REPO/pulls/7"

# Is it merged? (204 = yes, 404 = not yet)   /   Merge it
fj -o /dev/null -w '%{http_code}\n' "$FORGEJO_API/repos/$OWNER/$REPO/pulls/7/merge"
fj -X POST "$FORGEJO_API/repos/$OWNER/$REPO/pulls/7/merge" \
  -d '{"Do":"squash","delete_branch_after_merge":true}'
```
Merge body: required key is **`Do`** (capital D) ∈ `merge|rebase|rebase-merge|squash|fast-forward-only|manually-merged`.

### Releases (scope: `write:repository`)
```sh
fj -X POST "$FORGEJO_API/repos/$OWNER/$REPO/releases" \
  -d '{"tag_name":"v1.2.0","name":"v1.2.0","body":"Notes","target_commitish":"main"}'
# only tag_name is required; target_commitish creates the tag if it doesn't exist yet
```

## Git over HTTPS (token = password)

Use the PAT as the **password**. The token is what authenticates; the username can be the
account name. Both of these forms work:
```sh
git clone "https://$FORGEJO_USER:$FORGEJO_TOKEN@forgejo.example.com/$FORGEJO_USER/myrepo.git"
git clone "https://$FORGEJO_TOKEN:x-oauth-basic@forgejo.example.com/$FORGEJO_USER/myrepo.git"
```
- **Do NOT use the `oauth2:<token>@…` form** — that's a *GitLab* mirror convention, not Forgejo's;
  it does not authenticate Forgejo HTTPS clones.
- Avoid baking the token into a long-lived remote URL (it ends up in `.git/config`). Prefer a
  credential helper or `~/.netrc`:
  ```sh
  printf 'machine forgejo.example.com login %s password %s\n' "$FORGEJO_USER" "$FORGEJO_TOKEN" >> ~/.netrc
  chmod 600 ~/.netrc
  ```
- Plain *account password* over HTTPS is rejected when 2FA is on — a PAT-as-password is the fix.
  (Basic-auth git relies on the instance's `ENABLE_BASIC_AUTHENTICATION`, on by default.)

## Git over SSH (non-default port)

```sh
# 1) Add your public key once (token-authenticated)
fj -X POST "$FORGEJO_API/user/keys" \
  -d "$(jq -n --arg k "$(cat ~/.ssh/id_ed25519.pub)" '{title:"agent-host", key:$k, read_only:false}')"

# 2) Clone — a non-default SSH port MUST use the ssh:// URL form
#    (scp-style git@host:path can't carry a port). Replace 2222 with your instance's port.
git clone "ssh://git@forgejo.example.com:2222/$FORGEJO_USER/myrepo.git"
```
Or pin the port in `~/.ssh/config` so plain remotes work:
```
Host forgejo.example.com
    Port 2222
    User git
    IdentityFile ~/.ssh/id_ed25519
```

## `tea` CLI (optional convenience)

```sh
tea login add --name homelab --url "$FORGEJO_URL" --token "$FORGEJO_TOKEN"
tea repos list --login homelab
tea issues  --login homelab --repo "$FORGEJO_USER/myrepo"
tea pulls   --login homelab --repo "$FORGEJO_USER/myrepo"
tea clone   "$FORGEJO_USER/myrepo" --login homelab
```
Note: `tea login add` has **no `--ssh-port`/`--ssh-host` flag** — for SSH git ops set the port in
`~/.ssh/config` (above). Add `--output json` to most commands for machine-readable output.

## Working etiquette for agents

- **Never hardcode or commit the token.** Keep it in an env var / secrets manager; treat any file
  that holds a real token as a secret and keep it out of source control. Scope new tokens minimally;
  prefer the specific-repos restriction.
- Don't force-push or rewrite history on `main`/protected branches. Make a feature branch and open
  a PR; let the user merge unless they've said otherwise.
- Use clear commit/PR messages; reference issue numbers (`#42`).
- On `401`/`403`: token missing/expired or lacks the needed scope. On `404` against a known repo:
  almost always an auth/scope problem (private instance), not a missing resource. On
  connection/TLS errors: you're likely off the network or hitting an untrusted internal CA — surface
  it to the user rather than retrying.
- Treat a personal/private forge as such: destructive actions (delete repo/branch, close issues,
  merge) require explicit intent.

## Primary references
- API usage & auth: `https://forgejo.org/docs/latest/user/api-usage/`
- Token scopes: `https://forgejo.org/docs/latest/user/token-scope/`
- Live API schema for your instance: `$FORGEJO_URL/api/swagger`
