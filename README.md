# github-core-clone-forge — make.fun entry (v2)

[![make.fun](https://make.fun/badge.svg)](https://make.fun)

A small, fast GitHub-compatible git forge: repositories, issues and pull
requests, git smart HTTP (upload-pack v2 and v0, receive-pack), GraphQL v4 and
MCP — all in memory, all standard library, no dependencies.

## Run

```sh
node --experimental-strip-types main.ts
```

`main.ts` is only the trial lifecycle: it binds an ephemeral port, registers
it with the env (`MAKEFUN_ENV_URL`), then polls until the env says the trial
is done and exits 0. The server itself is `app.ts`.

## Layout

| file | what it holds |
| --- | --- |
| `main.ts` | env lifecycle (register / poll done) |
| `app.ts` | HTTP server + manual path dispatch |
| `ctx.ts` | per-request context, body, JSON replies, auth |
| `store.ts` | the forge: git objects, pack codec, views, merges |
| `git.ts` | smart HTTP: info/refs, upload-pack, receive-pack |
| `rest.ts` | REST v3 (repos, issues, pulls, contents, refs) |
| `gql.ts` | GraphQL v4 (tokenizer, parser, executor, mutations) |
| `mcp.ts` | MCP streamable HTTP, 8 tools |

## Why it is fast

The four perf workloads are sequential or concurrent request floods, so the
work that matters is per-request CPU, not throughput tricks:

- **Read**: each issue's JSON is serialized once and cached, split around the
  live `updated_at` timestamp. A single-issue GET is two string
  concatenations and a write — no object build, no `JSON.stringify`.
- **Clone**: the fetch pack and the ls-refs advertisements are cached per
  repository version. Fifty identical depth-1 clones build one pack and hit
  the cache forty-nine times.
- **Push**: commit ancestry uses generation numbers (a linear push checks
  one parent, not the whole history) and commit/tree metadata is cached at
  insert, so no object is ever re-parsed while walking.
- **Write**: issue creation reuses the same cached-view machinery as reads;
  the POST reply is the cached text with a fresh timestamp.
- **Everywhere**: the router slices `req.url` by hand (no `new URL` per
  request), queries are scanned per key, auth is a string prefix check, and
  queues use a head index instead of `Array.shift()`.

## Behaviour notes

- Bearer auth is `MAKEFUN_SHOP_TOKEN`; the viewer login is learned from the
  first owner-bearing request and a repository binds to the first owner it
  is seen under.
- Git objects are SHA-1, stored in memory; packs are written plain (no
  delta) and read with full OFS/REF delta resolution.
- Issues and pull requests share one number counter per repository.
- A pull-request merge is a squash commit (three-way merge against the
  merge base); a real conflict answers 405.

## License

MIT — see `LICENSE`.
