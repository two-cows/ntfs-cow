# References

Sources for understanding NTFS storage structures, Windows filesystem I/O, and copy-on-write implementation choices.

## Suggested reading order

Start with how NTFS represents files on a volume, then how Windows presents those files to applications.

1. **[Linux-NTFS documentation](https://flatcap.github.io/linux-ntfs/ntfs/).** A free community reference for the on-disk format. Start with [volume layout](https://flatcap.github.io/linux-ntfs/ntfs/help/layout.html), master file table (MFT) records, resident and nonresident attributes, and [data runs](https://flatcap.github.io/linux-ntfs/ntfs/concepts/data_runs.html). The data-run examples decode mappings from file clusters to volume clusters. This is older community documentation, not a complete specification of current Windows behavior.

2. **[File System Forensic Analysis, Brian Carrier](https://www.informit.com/store/file-system-forensic-analysis-9780321268174).** Chapters 11–13 cover NTFS concepts, analysis, and data structures, with disk-image analysis and worked examples. Published in 2005; useful for a structured explanation of the underlying format.

3. **[Windows Internals, Part 2, seventh edition](https://www.microsoftpressstore.com/store/windows-internals-part-2-9780135462409).** Chapter 11, “Caching and file systems,” connects filesystem drivers, NTFS, and the Cache Manager. For the I/O foundations, consult chapter 6, “I/O System,” in [Part 1](https://learn.microsoft.com/en-us/sysinternals/resources/windows-internals).

4. **[OSR: Introduction to Standard and Isolation Minifilters](https://www.osr.com/nt-insider/2017-issue2/introduction-standard-isolation-minifilters/).** Explains the distinction between ordinary filtering and presenting isolated file views, including separate cache sections. Useful before choosing where to implement a virtualization layer. This is an architecture article, not a complete cloning implementation.

5. **[WinFsp passthrough filesystem tutorial](https://winfsp.dev/doc/WinFsp-Tutorial/).** Builds a user-mode filesystem that forwards operations to a backing directory. Shows filesystem callbacks, logging, and testing. The passthrough implementation is an educational sample with documented limitations; [WinFsp](https://github.com/winfsp/winfsp) supplies the framework.

## Format and inspection references

- **[libfsntfs NTFS format specification](https://github.com/libyal/libfsntfs/blob/main/documentation/New%20Technologies%20File%20System%20%28NTFS%29.asciidoc).** A detailed working reference based on public information and analysis of test data. It records the project's findings rather than a Microsoft specification.
- **Microsoft: [Master File Table](https://learn.microsoft.com/en-us/windows/win32/fileio/master-file-table) and [Clusters and Extents](https://learn.microsoft.com/en-us/windows/win32/fileio/clusters-and-extents).** Official descriptions of file records and cluster addressing.
- **[fsutil file](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/fsutil-file).** Documents `queryfileid` and `queryextents`, useful for inspecting small and large files and comparing hard links on a disposable NTFS volume.
- **[Microsoft filesystem and filter driver samples](https://learn.microsoft.com/en-us/windows-hardware/drivers/samples/file-system-driver-samples).** MiniSpy logs I/O, SimRep redirects file opens, and NameChanger demonstrates namespace manipulation. These are starting points for driver development. Kernel-driver exercises belong in a test VM.

## Implementations and frameworks

### JuiceFS

[JuiceFS](https://github.com/juicedata/juicefs) is an implemented filesystem whose [clone operation](https://juicefs.com/docs/community/guide/clone/) copies metadata while sharing existing data blocks. Subsequent writes update the writer's mappings independently. Its community source uses the [Apache-2.0 license](https://github.com/juicedata/juicefs/blob/main/LICENSE).

- [Windows support](https://juicefs.com/docs/community/tutorials/windows/) uses WinFsp to mount a JuiceFS filesystem.
- [Local-disk backing storage](https://juicefs.com/docs/community/reference/how_to_set_up_object_storage/#local-disk) allows data blocks to reside in local files, with a separate metadata store.
- [Cloning semantics](https://juicefs.com/docs/community/guide/clone/) distinguish an atomic file clone from a directory clone that traverses entries. Concurrent source changes can produce an inconsistent directory clone.
- The [clone command](https://juicefs.com/docs/community/command_reference/#juicefs-clone) operates within one JuiceFS mount. Data must first be imported into that filesystem; this does not instantly clone an arbitrary existing NTFS directory.

### OSR Isolation Minifilter Solution Framework

[OSR IMSF](https://www.osr.com/imsf/) provides the isolation machinery and integration with the Windows Cache Manager and Memory Manager. OSR supplies C source under a paid commercial license. It is a framework for implementing file virtualization, versioning, and related behavior; clone mappings, shared-storage lifetime, and write redirection remain application design work.

## Comparisons at other storage boundaries

| Reference | Mechanism and boundary |
| --- | --- |
| [ReFS block cloning](https://learn.microsoft.com/en-us/windows-server/storage/refs/block-cloning) and [Dev Drive](https://learn.microsoft.com/en-us/windows/dev-drive/) | Native sharing of file ranges with copy-on-write. Files must reside on ReFS; a Dev Drive can be hosted in a VHDX file. Block cloning alone does not provide an atomic directory snapshot. |
| [VHDX differencing disks](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-vhdx/83f6b700-6216-40f0-aa99-9fcb421206e2) | Children retain changed virtual-disk blocks and consult their parent for other data. Operates below the filesystem, so the volume inside can use NTFS. The shared parent must remain unchanged; see [parent linkage](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-vhdx/b6332a98-624d-46b8-bd0e-b77b573662f9). |
| [Sandboxie-Plus file migration](https://sandboxie-plus.com/sandboxie/filemigrationsettings/) | A process sandbox with private file copies. Copying can occur on an open requesting write access, before any write, as described in its [FAQ](https://sandboxie-plus.com/sandboxie/frequentlyaskedquestions/). The isolation boundary is sandboxed processes. |
| [WinBtrfs](https://github.com/maharmstone/btrfs) | A Windows Btrfs driver with reflinks and subvolume snapshots. Useful implementation source for native copy-on-write; requires Btrfs storage. |
