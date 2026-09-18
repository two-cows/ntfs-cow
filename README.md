# ntfs-cow

Exploring copy-on-write (CoW) workspace clones on Windows through a virtualization layer over NTFS.

The goal is cheap, independent, disposable copies of complete directory trees, including `.git`, untracked files, and build artifacts. Applications should access them through normal filesystem paths, without eagerly duplicating every file's contents. Eventually, both the source workspace and its clones should be independently writable.

Current stage: educational. The repository has development tooling and documentation; filesystem implementation has not started.

## Roadmap

### 1. Educational

**First milestone: clone a single file so that its contents are readable through both the original and clone paths.**

### 2. Experimental

Develop toward an experimental workspace implementation. Scope and milestones remain open.

### 3. Maintained / production

Develop toward a maintained project suitable for production use. Scope and readiness criteria remain open.

## Decisions

Significant decisions are recorded in the [ADR index](docs/README.md).
