---
name: Nix Packaging
description: Packages softwares using Nix
---

# Overview

Create nix packaging of target softwares with examples for software of different programming languages.

1. Figure out whether the target software is inside this repo internally or from external (e.g. a github repo).
2. Figure out the programming language(s) the target software is built on.
3. Figure out the structure of the target software
4. Actually package the software, following language-specific guidance depending on the programming language and structure of the software
5. Optionally, add the package to the current project in the proper location (e.g. an exposed overlay, and then the exposed `packages` in the flake)
6. A lot of times, need to run `nix build .#<package-name>` to test the build, read the error message and fix, and test the build again and again.

# Language specific packaging skills

- See [python](./python/python.md) for packaging python modules.
- See [rust](./rust/rust.md) for packaging rust applications.
