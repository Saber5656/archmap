# 37 — Viewer UI (bundled, client-rendered)

## Summary

Build the viewer's single-page UI: sidebar navigation from `nav.json`, client-side markdown +
mermaid rendering with sanitization, in-memory search, and a fully bundled asset set (zero
external requests) per DESIGN §12.2.

## Context

The viewer gives the DeepWiki-like reading experience over the Markdown source of truth
(ADR-002: viewer is a thin layer that only reads). CSP from 36 forbids external assets and inline
scripts; everything ships in the npm package.

## Scope

- In: `src/viewer/ui/` source (vanilla TS + small deps), esbuild bundling step into
  `dist/viewer-assets/`, `index.html` shell.
- Out: server routing (36), serve command (38).

## Detailed Requirements

1. Stack: vanilla TypeScript (no framework), bundled with esbuild (build script
   `npm run build:viewer` invoked by main build); deps: `marked`, `dompurify`, `mermaid`,
   `minisearch` — all bundled locally, versions pinned. No fonts/icons fetched: system font
   stack, inline SVG icons.
2. Layout: left sidebar (nav tree from `/wiki/_meta/nav.json` `pages`, collapsible Modules
   section, active-page highlight), main content pane, top bar (repo name from nav
   `meta.repoName` — the contract issue 19 provides, search input, freshness badge from
   `/wiki/_meta/manifest.json` `generatedAt` + toolVersion).
3. Rendering pipeline per page load: fetch `/wiki/<slug>.md` → strip front-matter → `marked`
   (gfm tables on) → `DOMPurify.sanitize` with the locked policy: forbid `iframe`, `form`,
   `link`, `style`, `object`, `embed`; strip `srcset`; allowed URI pattern = relative paths and
   `#` fragments ONLY (no `http:`, `https:`, `//`, `data:`, `javascript:` — remote/absolute
   URLs render as inert text) → inject → then for each `code.language-mermaid` block:
   `mermaid.render` with `{securityLevel: "strict", startOnLoad: false}`; render failure →
   keep the code block visible with an error note (never blank).
4. Internal links (locked discriminator): resolve each relative link against the current wiki
   page path; if the normalized wiki-relative target (minus `.md`) matches a nav slug → rewrite
   to the SPA route `#/page/<slug>`; EVERY other link (citations into source, README targets,
   non-nav `.md` files) stays a citation-style link with a distinct class + copy-path button —
   no protocol handlers in v1 (documented decision).
5. Search: built LAZILY on first search-input focus (locked; supersedes any page-load wording):
   fetch all pages from the nav list once, build minisearch index (fields: title, headings,
   body; boost title); result dropdown (page, matched heading, snippet); Enter navigates.
   Wiki ≤ 200 pages assumption documented; loading indicator while indexing.
6. Routing: hash-based (`#/page/<slug>`), default `index`; unknown slug → friendly 404 view
   listing nav. Browser back/forward work.
7. Accessibility/minimums: keyboard navigable sidebar + search; prefers-color-scheme dark mode;
   content max-width readable; mobile: sidebar collapses (this is a local tool; minimal but not
   broken at 375px).
8. Bundle budget: total assets ≤ 2.5 MB (mermaid dominates; documented in build output check).

## Acceptance Criteria

- [ ] Playwright (or vitest browser-mode) smoke against the generated `fixture-ts` wiki (27's
      fixtures): loads index, sidebar lists pages with `meta.repoName` in the top bar,
      navigates to a module page, mermaid diagram SVG present.
- [ ] Hostile page (generated `fixture-inject` wiki): DOM contains no `script`, no event
      handlers, no `link`/`style`/`iframe`/`object` elements, no `srcset`, and no
      `http(s)/data:/javascript:` URLs in any attribute; remote-image markdown renders inert;
      zero external requests (interception assert).
- [ ] Link discriminator: a nav-slug `.md` link becomes an SPA route; a citation link to
      `src/…` and a non-nav `.md` link stay citation-styled with copy-path (three-case test).
- [ ] Search builds on first focus (no page fetches before focus — request log proof), finds a
      module page by symbol word; keyboard flow works.
- [ ] Mermaid render failure fixture shows code + note; app remains navigable.
- [ ] `npm run build` produces assets under budget; served by 36 with CSP without console
      violations.

## Validation

```bash
npm run build:viewer && npm run test -- viewer/ui
```

## Dependencies

27 (fixture wikis), 36.

## Non-goals

Editing, theming config, protocol-handler editor integration, server-side rendering.

## Design References

- DESIGN §12.2 (pipeline, strict mermaid), §13.4/13.5 (sanitization layers); ADR-002
