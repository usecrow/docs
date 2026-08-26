# Turnstone user guide

This repository publishes the public Turnstone guide. The canonical Mintlify
domain is [`docs.myturnstone.ai`](https://docs.myturnstone.ai).
`docs.usecrow.ai` was retired on 2026-08-26. Desktop builds using that host do
not receive remote guide tools until they update. Mintlify deploys `main`
automatically and exposes the same content to Turnstone Agents through the
site's documentation MCP server. The desktop enables only its read-only search
and page-retrieval tools and blocks the generated feedback tool.

## Update the guide

Edit the MDX pages in this repository through a GitHub pull request or the
Mintlify web editor. Keep control names aligned with the shipped desktop app,
and call out platform availability when behavior differs between macOS and
Windows.

Before merging, run:

```bash
npx mint validate
npx mint broken-links
```

After deployment, verify both endpoints:

- `https://docs.myturnstone.ai/.well-known/mcp`
- `https://docs.myturnstone.ai/mcp`

The reused Mintlify deployment may emit legacy `crow_documentation` or `nest`
tool IDs while the Turnstone dashboard rename and search reindex propagate.
Keep the desktop's compatibility aliases aligned with the production
`tools/list` response.
