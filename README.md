# Lexari Docs

Standalone documentation site for [Lexari](https://lexari.ai), served at docs.lexari.ai.

Next.js static export. Pages are Markdown files in `content/` with front matter (`title`, `slug`, `group`, `summary`). Sidebar groups are in `lib/docs.ts`.

```bash
npm ci
npm run dev      # http://localhost:3330
npm run build    # static site in out/
DOCS_BASE=/docs npm run build   # same site under a /docs path
```

The app lives at [app.lexari.ai](https://app.lexari.ai). Source: [github.com/xElvolution/lexari](https://github.com/xElvolution/lexari).
