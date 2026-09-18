# ntfs-cow

This project explores copy-on-write (CoW) workspace clones on Windows through a virtualization layer over NTFS.

The goal is cheap, independent, disposable copies of complete directory trees, including `.git`, untracked files, and build artifacts. Applications should access them through ordinary file system paths, and cloning should avoid eagerly duplicating file contents. Both the source workspace and its clones should support independent writes.

Current stage: educational. The repository contains development tools and documentation. File system implementation has not started.

## Roadmap

Development starts with educational work and progresses toward production use.

### Educational

**First milestone: clone a single file so that its contents are readable through both the original and clone paths.**

### Experimental

Develop an experimental workspace implementation. Scope and milestones remain open.

### Maintained for production

Maintain the project for production use. Scope and readiness criteria remain open.

## Decisions

The [architecture decision record (ADR) index](docs/README.md) lists significant project decisions.
