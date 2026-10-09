# Storage Correctness: Block Boundaries, ext4, Caching and Durability

**How myOS separates device I/O, partition geometry, filesystem meaning and persistence**

*Source: myOS master storage architecture, generic block boundary, GPT partitioning, ext4 implementation, VFS page cache, file-backed memory and asynchronous I/O design. Reviewed 9 October 2026.*

## 1. Why the layers must remain independent

A filesystem must not depend on how bytes reach a block device. A driver must not interpret filesystem inodes. A partition table should not determine which filesystem code executes, and the Linux or Windows personality should not own the raw storage device by virtue of using it.

The architecture therefore follows a sequence of independent contracts:

**Hardware authority → driver and transport → generic block device → partition view → filesystem implementation → native VFS → environment-facing file APIs.**

Each boundary has a narrower purpose than the one below it.

## 2. Generic block devices and bounded partitions

The native block interface describes addressable blocks, geometry, permitted operations and device lifetime without exposing VirtIO or PCI mechanisms.

GPT metadata is interpreted above that layer. The partition parser validates untrusted geometry and headers, and the partition abstraction presents a **bounded block-device view**. A filesystem mounted on one partition cannot legally address another partition merely by requesting a large logical block number.

That is also a security property. A filesystem provider can receive a capability for the particular source volume it serves rather than broad access to a controller or every storage device.

## 3. ext4 is a real filesystem implementation

The ext4 implementation has more substance than mounting a Linux-formatted filesystem read-only. The architecture records feature negotiation, checksum handling, inode and directory identity, extent mapping, HTree directory lookup, allocation, writes, rename/delete, deferred reclamation and journal/crash-recovery behavior.

Feature negotiation is especially important: unknown incompatible filesystem options must cause a defined refusal rather than silent corruption. Corrupt metadata and malformed extents likewise need bounded validation before they influence physical-block reads and writes.

An open file's lifetime differs from the lifetime of its directory entry. Removing a name does not necessarily permit immediate reclamation of storage still referenced by an open file. This is one reason deferred reclamation is coordinated with the native VFS.

The source documents Linux interoperability work and corruption-handling tests, while also recording unsupported ext4 features and configurations. “ext4 implemented” is a meaningful but qualified claim.

## 4. Journaling and durability

A filesystem mutation often updates several related pieces of metadata. Without atomicity, a crash could leave a directory referring to an unallocated inode or an allocation bitmap disagreeing with the inode data.

The ext4 journal coordinates these updates and replay/recovery. The cache and journaling behavior must also agree with explicit durability operations: **successfully buffered** is different from **durably committed to storage**.

The performance records show why this distinction mattered. Treating every tiny write as a complete synchronous transaction caused major overhead in realistic SQLite/k3s activity. Batched journal commits and cached writes improved behavior by avoiding unnecessary device round trips while preserving the requested synchronization semantics.

## 5. Shared asynchronous block I/O

The device-facing asynchronous queue carries descriptors in shared memory. A driver can accept more than one request and complete them independently. Shared buffers can reduce payload copying when the operation's size and lifetime justify the mapping cost.

The queue does not need to understand ext4, NTFS or a container image. It only knows the generic device-class operation. A ring-3 driver is responsible for translating those generic requests into the physical or paravirtual device's protocol.

Cancellation, timeouts and priority handling require class-specific end-to-end behavior. The reviewed async I/O architecture explicitly distinguishes features defined by a queue protocol from features fully consumed and validated by each device class.

## 6. Filesystem providers in ring 3

The native VFS can delegate filesystem behavior to a provider process through a shared-memory request interface. The provider can use an authorized block-device capability, but does not gain arbitrary DMA or device claims.

The documented original Linux ISO9660 and UDF implementations have read-only end-to-end test coverage through the provider model. This is separate from the kernel's own ext4 implementation and from the Android development branch's EROFS work.

## 7. Cache, file-backed memory and the current limitation

A native page cache can buffer filesystem data, track dirty pages and schedule writeback. File-backed memory objects allow multiple processes to map identical executable or shared-library pages.

However, the reviewed zero-copy audit records a remaining distinction between the writable page cache and the frames used for some file-backed mappings. Consequently **myOS must not yet be described as having one completely unified cache across every file-I/O and mmap path**.

This difference matters for coherence, memory pressure, truncation, writeback and concurrent readers. Those interactions are stronger evidence of filesystem correctness than a simple file-copy benchmark.

## 8. File events and filtering

The native VFS also owns filesystem-change generation and selected pre-operation interception. Linux inotify and Windows directory-change APIs can therefore observe shared mutations through the same source. A Windows minifilter may eventually consume a generic interception interface, but authentic Microsoft Filter Manager execution remains a separate integration requirement.

See [One Native VFS, Two Distinct Event Mechanisms](15-vfs-interception-and-file-notifications.md).

## 9. Summary

The storage achievement is the **separation and demonstrated integration of real layers**, not the assertion that every filesystem format or disk controller is already supported. myOS has native block and partition abstractions, a substantive ext4 implementation, asynchronous driver-facing I/O, ring-3 filesystem providers and a VFS shared by operating-system personalities. The remaining coherence and coverage issues are stated explicitly rather than hidden by the diagram.
