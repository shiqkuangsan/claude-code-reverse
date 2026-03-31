# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a reverse-engineering study of `@anthropic-ai/claude-code` v2.1.88. The goal is to understand Claude Code's internal architecture by restoring readable source from the published npm package.

## Repository Structure

- `sources/` — Original npm package files extracted from `@anthropic-ai/claude-code@2.1.88`
  - `cli.js` — Bundled entry point (~13MB)
  - `cli.js.map` — Source map (~60MB)
  - `sdk-tools.d.ts` — TypeScript type definitions for SDK tools
  - `package.json`, `LICENSE.md`, `README.md` — Package metadata
- `restored/` — Source code restored from the source map
  - `src/` — Main application source (TypeScript/TSX modules)
  - `vendor/` — Native addon source (audio-capture, image-processor, etc.)
  - `node_modules/` — Bundled third-party dependencies (lodash-es, etc.)

## Key Conventions

- Files under `restored/` are generated from source maps — do not manually reformat or restructure them
- The restored source uses ESM (`"type": "module"` in the original package)
- When analyzing code, cross-reference `restored/src/` with `sources/cli.js.map` for accuracy
- Large files like `sources/cli.js` and `sources/cli.js.map` should be searched with grep, not read in full
