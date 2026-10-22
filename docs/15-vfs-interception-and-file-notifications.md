# One Native VFS, Two Distinct Event Mechanisms

**Filesystem interception and filesystem change notification across Linux and Windows**

*Architecture basis: myOS implementation, VFS operation interception decision dated 9 October 2026 and the native VFS file-change notification decision. This is an evidence-limited description of the current mechanism, not a statement that every filesystem filter or watcher scenario works.*

## 1. Why this is a major architectural distinction

A shared VFS must give Linux and Windows access to filesystem operations without building an independent set of storage events for each environment.

Two different questions arise:

- **Before an operation:** may another authorized service inspect or refuse it?
- **After an operation:** which interested processes should be told that something changed?

These are not the same mechanism. A filter that makes a permission decision belongs in the synchronous operation path. A watcher that reports an already completed write or rename cannot retroactively refuse it.

myOS therefore distinguishes **operation interception** from **change notification** while locating both mechanisms at the common VFS boundary.

## 2. A neutral source of filesystem mutations

Applications reach the native VFS through their environment-specific file APIs. Linux uses Linux pathname and file-descriptor conventions, Windows uses NT paths and HANDLE conventions. Their filesystem operations ultimately pass through common VFS mutation entry points.

This ensures that a change created by a Linux process can be observed by a Windows directory watcher, and vice versa, without teaching a filesystem provider to emit Linux or Windows protocol records.

Events describe filesystem objects, mutations and names. They are not generated separately by ext4, a memory filesystem and every hosted provider. That would create duplicate events and inconsistent semantics across filesystem implementations.

### What an event identifies

The neutral event model can identify a filesystem object, an entry name, the kind of change, whether the entry is a directory and additional relevant metadata. A rename has correlated old/new halves so each personality can translate it to its own expected format.

A subscription can observe one object or a subtree. Notifications are tied to object identity and subscription lifetime, not merely to a pathname string that may become stale after a rename.

## 3. Linux inotify is an adapter

Linux applications use inotify watch descriptors, event masks, event record layouts and file-descriptor readiness. myOS maps those Linux-facing conventions to neutral VFS change subscriptions.

The adapter translates create, delete, rename, modify, attribute and use events. Its queue is finite and reports overflow under defined circumstances. A reader waits through the normal Linux fd/readiness contract and the shared native wait infrastructure.

The VFS itself does not need to understand inotify record layouts or Linux watch-descriptor allocation.

## 4. Windows directory change is another adapter

Windows applications use the NT directory-change interface and, commonly, asynchronous completion. A .NET application using **FileSystemWatcher** relies on underlying Windows directory-change behavior and completion-port integration; it does not need a new myOS-specific filesystem watching API.

The Windows adapter converts the same native mutations into the expected Windows name/action records, including old and new names for renames. It coordinates synchronous and asynchronous requests, caller buffer size, I/O status, event notification, cancellation and completion-port delivery.

If a requested sequence of changes no longer fits the buffered contract, the result must indicate an overflow that requires the caller to enumerate the directory again. Silently discarding changes and returning ordinary success would violate the Windows API contract.

## 5. Why the change is delivered after success

A write that fails must not look like a successful content change. The VFS emits notifications after the underlying operation reports the relevant successful state, outside locks that would prevent subscribers from taking further actions.

This supports mixed Linux/Windows filesystem activity and prevents a filesystem provider from having to know which personality initiated an operation.

The exact Windows and Linux records can differ while remaining projections of one native event.

## 6. Interception: before/after control, not event watching

The separate native interception mechanism lets an authorized service register on a filesystem and take part in selected file operations.

For an intercepted operation, the VFS can call the service **before** performing it. The service may allow or refuse the operation. Afterward, earlier participants may be told the result. Interceptors have an ordering convention so several filters can be invoked deterministically.

This is the generic facility that makes authentic Windows file-filter infrastructure possible without placing a Windows-specific filter manager inside the VFS. It can also support other permission-oriented consumers, provided they have been explicitly authorized.

### Authority and re-entrancy

An interceptor needs rights over the filesystem and a valid service endpoint. It cannot take over arbitrary filesystem operations just by supplying an object name.

The call is performed without retaining broad VFS locks, because the interceptor may itself need filesystem operations. An interceptor is not invoked for its own process's operations, avoiding trivial recursion. If an interceptor disappears during a pending request, the documented policy provides a defined way for the operation to continue or terminate without leaving a stranded registration.

### Current coverage limitation

The source's current interception scope includes selected operations such as open, create, remove, rename, read, write, truncate and attribute updates. It does **not** cover every internal writeback, mapped-image access, lookup, enumeration, symlink or special-file operation. The initial interception evidence is a dedicated kernel test on a memory filesystem. This is a **generic interception facility**, not proof that Microsoft's original Filter Manager is already functioning.

## 7. Lifetime and backpressure

Both mechanisms must handle closure and object destruction:

| Question | Native behavior |
|---|---|
| A watched object is removed | Subscribers receive an ending condition and subscriptions are retired according to object lifetime |
| A filesystem is unmounted | Its subscriptions are ended |
| A watching process exits | Its subscription and pending work are cleaned up |
| A watch buffer fills | Environment-specific overflow behavior is returned |
| A filter host exits | Its interception registration ceases to be active |
| A pending file operation is canceled | The relevant NT or Linux-facing operation resolves through its documented cancellation contract |

A queue cannot simply grow forever. Backpressure and finite buffering are ordinary correctness requirements, not reasons to duplicate the native source of truth.

## 8. Architectural consequence

The same VFS can provide **Linux inotify**, **Windows directory-change notifications** and **permission-oriented interception** without introducing three unrelated mechanisms for observing storage.

This also clarifies what remains unfinished in Windows-driver compatibility: authentic Windows minifilter behavior depends on both this common VFS foundation and a sufficiently complete original Microsoft Filter Manager ABI. One does not imply the other.

The native VFS has become more than a path-resolution layer. It is a coherent shared authority for file identity, operations, interception and post-operation notification.
