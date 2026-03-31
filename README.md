# claude-code-reverse

Reverse-engineered source of [`@anthropic-ai/claude-code`](https://www.npmjs.com/package/@anthropic-ai/claude-code) v2.1.88, restored from the published source map for educational study.

## Structure

```
sources/          # Original npm package files
  cli.js          # Bundled entry point
  cli.js.map      # Source map
  sdk-tools.d.ts  # SDK type definitions
  package.json    # Package metadata

restored/         # Source restored from source map
  src/            # Application source (TypeScript/TSX)
  vendor/         # Native addon source
  node_modules/   # Bundled dependencies
```

## Disclaimer

This repository is for **educational and research purposes only**. All rights to Claude Code belong to [Anthropic](https://www.anthropic.com/). The restored source is derived from publicly available npm package artifacts. Do not use this code for commercial purposes or to create competing products.
