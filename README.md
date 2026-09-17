# ntfs-cow

Exploring copy-on-write (CoW) workspace clones on Windows through a virtualization layer over NTFS.

The goal is cheap, independent, disposable copies of complete directory trees, including `.git`, untracked files, and build artifacts. Applications should access them through normal filesystem paths, without eagerly duplicating every file's contents. Eventually, both the source workspace and its clones should be independently writable.

Current stage: educational. The repository has development tooling and documentation; filesystem implementation has not started.

## Roadmap

### 1. Educational

**First milestone: create a CoW clone of a single file, using whole-file copy-on-first-write.**

- Initially, logical copies share the same backing file contents.
- On the first write to a copy, duplicate the entire file into private storage and apply the write there. Subsequent reads and writes use that private file.
- Verify that modifying one copy leaves the base and other copies unchanged, and that discarding a copy reclaims only its private storage.
- Use whole-file mappings, with no block-level accounting.

This milestone may require an immutable base. Before sharing it, choose, document, and enforce how the base becomes immutable. A live source directory must not be treated as a snapshot.

The interface and implementation approach remain open. Keep the first implementation small enough to trace its behavior and test its isolation guarantees.

### 2. Experimental

Develop toward an experimental workspace implementation. Scope and milestones remain open.

### 3. Maintained / production

Develop toward a maintained project suitable for production use. Scope and readiness criteria remain open.

## Decisions

Significant decisions are recorded in the [ADR index](docs/README.md).
