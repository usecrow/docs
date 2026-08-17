# Nest user guide

This repository publishes the public Nest guide at
[`docs.usecrow.ai`](https://docs.usecrow.ai). Mintlify deploys `main`
automatically and exposes the same content to Nest Agents through the site's
documentation MCP server at `https://docs.usecrow.ai/mcp`. The desktop enables
only its read-only search and page-retrieval tools and blocks the generated
feedback tool.

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

After deployment, verify that `https://docs.usecrow.ai/.well-known/mcp` and
`https://docs.usecrow.ai/mcp` respond successfully.

The reused Mintlify deployment currently generates search tool IDs with its
legacy `crow_documentation` namespace even though the site and indexed content
are Nest. Keep the desktop's measured IDs and future `nest` aliases in sync with
the production `tools/list` response until the deployment is renamed in the
Mintlify dashboard.
