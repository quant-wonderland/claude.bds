---
name: Nix Packaging
description: Package software using Nix
---

# Overview

Package target software using Nix, with language-specific examples and guidance.

## Workflow

1. Identify the software source: internal to this repo or external (e.g., a GitHub repository).
2. Determine the programming language(s) used.
3. Analyze the software structure (build system, dependencies, configuration files).
4. Create the package following language-specific guidance below.
5. Optionally, integrate the package into the current project (e.g., add to overlay and expose via `packages` in `flake.nix`).
6. Test iteratively: run `nix build .#<package-name>`, read errors, fix issues, and rebuild until successful.

# Language specific packaging skills

- See [python](./python/python.md) for packaging python modules.
- See [rust](./rust/rust.md) for packaging rust applications.
