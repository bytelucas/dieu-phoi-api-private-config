# Private config bootstrap

Project-wide agent guidance is intentionally local-only. The canonical private copy is stored in the
`dieu-phoi-api-private-config` private repository and is installed into this checkout with:

```bash
git clone git@github.com:bytelucas/dieu-phoi-api-private-config.git ../dieu-phoi-api-private-config
PRIVATE_CONFIG_SOURCE=../dieu-phoi-api-private-config pnpm config:bootstrap
```

The bootstrap installs `AGENTS.md`, context-specific `AGENTS.md` files, `CLAUDE.local.md`, and
`docs/agents/`. These paths are ignored by this repository by design. Keep shared guidance in the
private config repository; put Claude-only instructions in `CLAUDE.local.md`.
