# 36 — Viewer HTTP server (loopback, confined)

## Summary

Implement the viewer's HTTP layer per DESIGN §12.2: loopback-only static server for wiki files,
nav, and bundled UI assets, with path confinement, security headers, and GET/HEAD-only semantics.

## Context

Trust boundary B4's second surface (after MCP). The server must be boring: no dynamic execution,
no writes, no network egress, no non-loopback exposure. UI assets come from issue 37; this issue
serves them.

## Scope

- In: `src/viewer/server.ts` (node:http, no framework), route table, mime map.
- Out: UI implementation (37), serve command UX (38).

## Detailed Requirements

1. API (locked):
   ```ts
   createViewerServer({wikiRoot, uiRoot, port}): ViewerServer
   interface ViewerServer {
     listen(): Promise<{address: "127.0.0.1", port: number, url: string}>
     close(): Promise<void>          // stop accepting, drain ≤ 2 s, resolve
     urlFor(path?: string): string
   }
   ```
   Binds `127.0.0.1` (hard-coded literal; no host option exists anywhere in the signature).
   `listen()` rejects with `ArchmapError(E_USAGE)` preserving the `EADDRINUSE` cause + hint
   (`--port`) — issue 38 consumes this.
2. Routes:
   - `/` → `index.html`; `/ui/*` → bundled UI assets from `uiRoot` (fs-confined via
     `joinConfined` + `realpath` per 05's contract). UI MIME allowlist (locked; anything else
     under uiRoot → 404): `.html` `text/html; charset=utf-8`, `.js`
     `text/javascript; charset=utf-8`, `.css` `text/css; charset=utf-8`, `.svg`
     `image/svg+xml`, `.json` `application/json`, `.map` `application/json`, `.woff2`
     `font/woff2` (in case a font is ever bundled).
   - `/wiki/*` → exact allowlist under `wikiRoot`: any `*.md` page
     (`text/markdown; charset=utf-8`) plus EXACTLY `_meta/nav.json` and `_meta/manifest.json`
     (`application/json`). Any other file under wikiRoot — including other `.json` such as
     `_meta/module-graph.json` — → 404.
   - `/api/health` → `{ok: true, toolVersion, pageCount}` where `toolVersion` =
     `TOOL_VERSION` (issue 02 build constant) and `pageCount` = count of slug-bearing entries
     in `_meta/nav.json` (read under confinement; missing/malformed nav → `pageCount: 0`).
   - Unknown → 404 JSON body; non-GET/HEAD → 405 with `Allow: GET, HEAD`, body NOT consumed.
3. Confinement: resolve request path → decode percent-encoding ONCE → reject paths containing
   `..` segments, null bytes, backslashes → `joinConfined` → `realpath` must stay under root
   (symlink escape blocked). Failure → 404 (never 500, never echo the path).
4. Headers on every response: `Content-Security-Policy: default-src 'self'; img-src 'self'
   data:; style-src 'self' 'unsafe-inline'` (mermaid injects inline styles — documented
   exception; NO script-src relaxation), `X-Content-Type-Options: nosniff`,
   `Referrer-Policy: no-referrer`, `Cache-Control: no-cache` (local dev semantics).
5. Directory requests → 404 (no listings, no index fallback except `/` → UI shell).
6. Request bodies (locked): non-GET/HEAD → 405 without reading the body; GET/HEAD carrying a
   body indicator (`Content-Length > 0` or `Transfer-Encoding`) → 413, body never read
   (stream not consumed — test proves it). HEAD returns the same status/headers as GET with an
   empty body. Response streaming via `fs.createReadStream`; stream error → 404 handler.
7. Port busy surfaces through `listen()` per requirement 1.

## Acceptance Criteria

- [ ] Traversal corpus (from DESIGN §13.7 set: `..%2f..`, `%2e%2e/`, double-encoding, symlink
      inside wikiRoot pointing to `/etc`) → all 404, fs access spy confirms no read outside
      roots.
- [ ] Binds only loopback: `server.address().address === "127.0.0.1"`; `listen()` resolves the
      locked shape.
- [ ] Header set present on 200/404/405 responses alike; POST → 405 with unread body; GET with
      `Content-Length: 5` → 413 with unread body; HEAD mirrors GET headers with empty body.
- [ ] MIME matrix: html/js/css/svg served with the locked types under `nosniff`; unknown UI
      extension → 404; `.md` + the two `_meta` JSONs served; any other wikiRoot file (incl.
      `_meta/module-graph.json` and planted `x.json`) → 404 (allowlist proof).
- [ ] `/api/health` correct on normal nav and `pageCount: 0` on malformed nav.
- [ ] Port-busy: `listen()` rejects `E_USAGE` with the EADDRINUSE hint.

## Validation

```bash
npm run test -- viewer/server
```

## Dependencies

19.

## Non-goals

UI content (37), TLS (loopback only), auth (none by design), HTTP/2.

## Design References

- DESIGN §12.2, §13.5 (B4), §5 (confinement primitives)
