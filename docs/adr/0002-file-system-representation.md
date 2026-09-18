# ADR 0002: File system representation

Date: 2026-09-18

Status: Proposed

## Context

Workspace copies must share initial file contents, support independent writes, and remain accessible through ordinary Windows file system paths. The first feasibility test clones a single file and verifies that its contents are readable through both paths. The proposed design uses whole-file copy-on-write (CoW), with no block-level accounting, and must allow later redirection of block ranges.

Files and directory entries must remain on NTFS. Cloning must support destinations in ordinary NTFS directories without requiring a separate file system mount. The layer must preserve isolation when applications access the source through its original path. When sharing ends, files must be able to return to native NTFS access at the same paths.

The architecture is provisional. Its feasibility and the choice of driver framework remain open.

## Alternatives considered

- **Eager copies** provide independent NTFS files but immediately duplicate all contents.
- **Hard links and pathname redirection** direct applications to the same underlying stream. They do not provide independent writable views.
- **WinFsp's mounted file system design** can use NTFS backing files, but applications must access clones through its file system namespace. Directory mounts provide flexibility, but they do not let this design manage individual files in arbitrary existing NTFS directories. Direct access to a live source through its native NTFS path bypasses the CoW callbacks. Protecting shared backing requires an additional mechanism. Reject this design because it does not meet the destination and source-isolation requirements.

  The documented [kernel-mode option](https://winfsp.dev/doc/WinFsp-Kernel-Mode-File-Systems/) also creates a WinFsp volume. The reviewed documentation does not describe a mode that acts as an NTFS minifilter. This conclusion does not rule out a custom design that combines WinFsp with additional redirection and source protection.

ReFS and differencing virtual disks remain comparisons outside the project scope. ReFS requires a different file system, and differencing disks share storage at the virtual-disk level.

## Decision

Investigate the following isolation minifilter design. The choice of driver framework remains open.

### Logical files

The isolation minifilter identifies CoW files by a custom reparse tag. For the initial clone, the driver retains the original NTFS file as backing file `S` and creates two placeholder files, `A` and `B`. The original path refers to `A`; the clone path refers to `B`. Both placeholders initially refer to `S` through their reparse metadata.

The driver interprets these backing references and manages their lifetime. It uses supported file system operations to create files, rename files, and set reparse points. NTFS maintains the master file table (MFT) records and directory indexes.

Applications open `A` or `B` as distinct logical files. The driver opens `S` internally to supply their contents. Each logical file must have an independent view for buffered I/O, memory mappings, and paging I/O. For the required cache separation, see [OSR's isolation minifilter architecture](https://www.osr.com/nt-insider/2017-issue2/introduction-standard-isolation-minifilters/).

### Clone creation and writes

Cloning requires exclusive access to the source: no competing handles, mappings, or operations. The driver must prevent new access until clone creation completes.

For the feasibility test, a console application opens the source with sharing disabled and keeps the handle open until the driver completes the clone request. The request transport, how the request identifies the open source file, and how the driver protects the path replacement remain open.

Before sharing `S`, the driver must prevent direct modification through any other path or access method. `S` remains immutable while shared. An internal directory alone does not provide this protection, and a live source is not a snapshot.

On the first write to `A`, the driver:

1. Copies the whole backing file `S` to a private file `T`.
2. Changes `A`'s mapping to `T`.
3. Applies the write to `A`'s private contents.

The driver must synchronize these steps across all handles and processes that open the same logical file. Only one operation at a time may perform the first-write copy for `A`. Concurrent writers must wait for the transition to finish. If the transition succeeds, they must use the same private file `T`. The driver must coordinate reads and writes with the transition to prevent reads of an incomplete copy, writes to shared `S`, and lost writes from competing mapping changes.

The protocol must also cover cached, memory-mapped, and paging I/O, ownership updates, and failed copies. The synchronization mechanism remains open.

`B` continues to read `S`. Writes to `B` follow the same rule. All open instances of a logical file must observe its current mapping and contents.

### Backing-file lifetime (open issue)

The mechanism for retaining backing files while clones need them, recovering them after a crash or reboot, and reclaiming them safely remains open. Keeping a hard link in a visible backing directory is not an accepted solution.

Reparse references and the driver's logical-owner counts do not create NTFS hard links. An open handle alone cannot preserve a backing file across a reboot. The design must account for NTFS's [file deletion semantics](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/ntddk/ns-ntddk-_file_disposition_information_ex).

### Conversion to native files

The driver maintains metadata to count each backing file's logical owners and identify the remaining owner when sharing ends. It tracks active use separately; opening a handle does not create another CoW copy.

When a backing file has one logical owner and the driver can safely remove that owner's placeholder, the driver converts the logical file to a native NTFS file. After conversion, the visible directory entry refers to the backing file itself, and the placeholder no longer exists. This operation requires no additional content copy. The driver uses supported namespace operations and lets NTFS update its records.

Conversion requires exclusive access to the logical file until the path replacement completes. This requirement applies independently to `A` and `B`. An exclusive handle is a candidate mechanism. If the driver cannot obtain exclusive access, it defers conversion and keeps the logical file usable.

The driver checks conversion eligibility when the owner count drops to one and when the last application handle closes. Windows can retain cache and memory-manager references after handles close; mapped views can also outlive their handles. The access protocol must account for all open instances, mappings, dirty data, and pending I/O. A no-sharing file open alone does not establish these conditions. See Microsoft's [file cleanup semantics](https://learn.microsoft.com/en-us/previous-versions/windows/drivers/ifs/irp-mj-cleanup) and [file mapping lifetime](https://learn.microsoft.com/en-us/windows/win32/memory/closing-a-file-mapping-object).

After `A` switches to `T`, both `T` and `S` have one logical owner. Each logical file becomes eligible for conversion independently. Once both meet the access and lifetime requirements, both paths refer to native NTFS files and the driver removes both placeholders.

## Consequences

- Separating logical files from backing mappings allows later redirection by byte range. Block-level sharing requires changes to ownership accounting and conversion eligibility.
- Writes to either logical file must leave the other file's contents unchanged. The driver must retain backing storage while any logical owner or outstanding operation needs it.
- Conversion removes unnecessary placeholders but changes the NTFS file ID at the path. This proposal does not preserve file IDs across conversion.
- Isolation requires integration with the Cache Manager and Memory Manager. Framework selection and source licensing remain open.
- Hard-link aliases remain an open issue: replacing the original path with `A` leaves any other hard links pointing directly to `S`. Writes through those names could modify shared backing and break clone isolation.
- Metadata preservation, existing reparse points, path replacement, and crash recovery remain open. Cloning and conversion have no defined crash-atomicity guarantee. A failed or deferred conversion must preserve a usable logical file and its backing data.
