---
layout: post
title: Windows Memory Mapped File IO
date: 2024-11-03 00:00
categories:
  - Winapi
  - IO
---

Memory-mapping is my preferred way to do file I/O, on pretty much every platform
I write code for (desktop and console). In this post, we'll start by discussing
some of the key advantages of this approach, as well as some disadvantages.
Then, we'll look at some Windows-specific undocumented user-mode functions which
help mitigate some of those disadvantages. In particular, we'll address an
important perceived limitation of memory-mapped I/O, namely, file resizing.

Note that this post is largely an amalgamation of various notes I've collected over the
years, so some parts of this post aren't well-structured into a linear narrative.
Also, I am in no way suggesting that memory-mapping is a panacea for all file I/O
needs. Memory-mapped I/O has, however, served me well for my use-cases, so consider
the pros and cons for your particular needs and users.

## Recap

Let's quickly recap how Winapi memory-mapped files work. All error-checking is
omitted for brevity.

First, we create a file handle:

```cpp
HANDLE file_handle = CreateFile(
    file_name,
    GENERIC_READ,
    FILE_SHARE_READ,
    nullptr,
    OPEN_EXISTING,
    0,
    nullptr);
```

In this example, we're sticking to a read-only file, but a mutable memory mapped
file is easy to achieve with different protection and access bits.

Next, we need to create a file mapping object:

```cpp
HANDLE file_mapping_handle = CreateFileMapping(
    file_handle,
    nullptr,
    PAGE_READONLY,
    // Passing zeroes for the high and low max-size params here will allow the
    // entire file to be mappable.
    0,
    0,
    nullptr);

// We can close this now because the file mapping retains an open handle to
// the underlying file.
CloseHandle(file_handle);
```

Finally, we map the file mapping to an address range which we can use to read
data.

```cpp
char const* data = (char const*)MapViewOfFile(
    file_mapping_handle,
    FILE_MAP_READ,
    0, // Offset high
    0, // Offset low
    // A zero here indicates we want to map the entire range.
    0);

// When we are done, closing the file mapping handle releases the file for use
// by other applications.
CloseHandle(file_mapping_handle);
```

The code above would need to be adopted to add write or execute access, and would
need special handling for specific cases like file creation (creating a zero-size
file mapping object is disallowed).

## Why would we prefer memory-mapped I/O?

The alternative (in a Windows world) is to use the following ensemble of syscalls:

- `CreateFile`
- `ReadFile`
- `WriteFile`

The read and write syscalls contain additional functionality to support "overlapped"
(i.e. asynchronous) I/O, operating in conjunction with I/O completion ports. We
could, if we wanted, map this set of abstractions to similar syscalls in other
operating systems and platforms.

I consider this somewhat clunky for a few reasons. First, this approach pushes
the onus of memory allocation to the user. Things are easy if we can just allocate
buffers to accommodate the entire file, and read contents wholesale, but at some
point, you need some notion of partially resident files, at which point, juggling
file partitions becomes a real chore.

Another issue with this approach is the complexity involved with supporting a
virtual file system. In a game engine context (where I operate), it's very typical
for large collections of loose assets to be aggregated into "archive" or "pack"
files. Reading contents of these files using some sort of abstraction over
`ReadFile` and `WriteFile` is very clunky compared to the alternative of mapping
memory regions.

Yet another problem with the `ReadFile` and `WriteFile` approach is the pain that
is asynchronous I/O. IO completion ports on Windows aren't so dissimilar from
`epoll` and other analogous abstractions on other platforms that you couldn't
come up with a suitable cross-platform API covering it, but this is complexity
I'd rather avoid at all costs if possible. The semantics are different enough
that even if you do port the functionality successfully, your first few attempts
will have different performance characteristics on different platforms (with
similar hardware).

Finally, I dislike the `ReadFile` and `WriteFile` approach because it ultimately
precludes a number of neat optimizations and functionality provided directly by
the operating system. While we get some of that (buffering mainly), memory-mapping
gets us there much more quickly. Without memory-mapping, we also incur at least
one extra copy to move data into and out of the resident memory managed by our
program.

In contrast, memory mapping the file gives us:

- The ability to map a file as many times as we want, persistently.
- The ability to let the page cache manage partial residency for us.
- A smaller surface area of cross-platform functionality we need to maintain.
  - The main caveat here being that Windows doesn't support overcommit and has a
    separate notion of "reserved" vs "committed" virtual address space.
- The ability to use `memcpy` or `memset` and any other functions at our disposal
  transparently with the underlying file kernel stack.
- Fewer syscalls and copies.

## What about drawbacks?

There are a few things to keep in mind before going "all in" on memory mapped
IO.

First, there is no dedicated asynchronous APIs when memory mapping files. Page
faults will occur on whatever thread accessed the memory, which could result in
unexpected latency. I actually consider the lack of async APIs here a feature,
and would prefer to fetch pages on my own terms, within the context of my own
async IO threads which function identically on multiple platforms. The exception
is the case where we actually want to incur more concurrent page faults than we
have active threads in the system. Non-desktop platforms have specific APIs that
help manage this, and on Windows, `DirectStorage` and `PrefetchVirtualMemory` are
options to consider.

Another issue is that, compared to `WriteFile`, there is no easy way to track
precisely when mutations to a file are made. Aside from not having a place in
a debugger to break on modification (write watches on pages aside), writes to
a file are typically cached and flushed at some indeterminant point in the
future unless a manual `FlushViewOfFile` is issued. Because order isn't
guaranteed, some care is needed if you need the data on disk to be internally
consistent.

Another disadvantage is the handling of file resizing. With documented APIs,
the user must:

- Create the file handle ([`CreateFile`](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilea))
- Create the file mapping ([`CreateFileMapping`](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createfilemappinga))
- Map a view of the mapping ([`MapViewOfFile`](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-mapviewoffile))

At the point of the file mapping creation, the size of the map is fixed, and
there isn't a way to "grow" the mapping. As another quirk, if you set the mapping
size to be larger than the file itself, the file immediately grows to accommodate
the mapping size. In either case, the capability to append to write-mapped files
appears to be missing. At least, that is the case with Winapi calls! An important
trick here is to rely not on the Winapi functions, but the less-documented Ntdll
functions instead. This will be the subject of the next section.

## YOLO NTDLL file sections

First, I should quickly recap the idea behind virtual memory in Windows. As you
likely know, nobody in user-space works with physical memory addresses. All memory
access is virtualized. When you call `operator new` or `malloc`, you are invoking
a CRT function which, in turn, suballocates from a runtime heap which maps to some
virtual address space assigned to your process.

Windows gives us a few capabilities, with the idea of address reservation being
a mostly Windows concept:

- We can reserve virtual address ranges. This prevents other allocations from
  overlapping with any range we reserve (unless such aliasing is explicitly requested).
- We can commit virtual address ranges. This then assigns page table entries to
  our address range, so that they could be backed by physical memory when we
  read and write to them later.
- We can assign memory protections to virtual address ranges. The OS uses this
  to map pages that are executable and read-only when you launch programs. This
  is also used to create write-combined pages (such as what you'd use to upload
  data to the GPU).
- We can map multiple virtual address ranges to the same physical data, possibly
  with different protection bits.
- We can request that the operating system notify us if a page is made dirty.
- We can trap exceptions of different sorts using structured exception handlers
  (SEH), and use this to set up mechanisms like guard pages and such.

There are _tons_ of things you can do with just the capabilities listed, and
this is really scratching the surface. For our purposes though, what we'd _like_
to do is:

- Create a file handle as before (`CreateFile`).
- Create a file mapping _virtually_, and simply reserve an address range that
  _can be mapped_ in the future.
- Map views over the file mapping as before.
- Have the option to _extend_ the file mapping, which should have the effect of
  both committing the memory range covered by the extension and increasing the
  file size if necessary.

The idea here is that we could reserve some large address range (128 GiB or
something) to encompass the theoretical maximum size we'd want the file to
accommodate upfront. This wouldn't change the actual file size on disk. Later,
we could extend the committed range to whatever file was needed, potentially
multiple times. When doing this, the file size _does change_, and we _are_ able
to append new contents to the file, but the address range over which we operate
does _not_ change.

Neat, that sounds like a plan, but how do we accomplish it? As mentioned before,
`CreateFileMapping` takes a mapping size which cannot be modified after initial
creation. Our only choice would be to close that mapping, reopen the handle with
a new range, remap the view, and invalidate all pointers in the process. This
adds several additional syscalls of overhead, and also detracts quite a bit from
the convenience of memory mapping in general.

As the section title indicates, the answer is to rely on a few undocumented
functions that are available to us in `NTDLL.dll`. As with other NT kernel
functions, these won't be available to you in a header you could include from a
typical application. We'll need to map them in ourselves:

```cpp
enum SECTION_INHERIT
{
    // Created section view will be mapped by child processes.
    ViewShare = 1,
    // Created section view will not be mapped by child processes.
    ViewUnmap = 2,
};

struct NTSTATUS
{
    uint32_t code     : 16;
    uint32_t facility : 12;
    uint32_t reserved : 1;
    uint32_t customer : 1;

    // 0x0: Success
    // 0x1: Informational
    // 0x2: Warning
    // 0x3: Error
    uint32_t severity : 2;
};

static_assert(sizeof(NTSTATUS) == 4);

using NtCloseFn = NTSTATUS(NTAPI*)(HANDLE);
static NtCloseFn NtClose;

using NtCreateSectionFn =
    NTSTATUS(NTAPI*)(OUT PHANDLE SectionHandle,
                     IN ACCESS_MASK DesiredAccess,
                     IN void* ObjectAttributes OPTIONAL,
                     IN PLARGE_INTEGER MaximumSize OPTIONAL,
                     IN ULONG PageAttributes,
                     IN ULONG SectionAttributes,
                     IN HANDLE FileHandle OPTIONAL);
static NtCreateSectionFn NtCreateSection;

using NtMapViewOfSectionFn = NTSTATUS(NTAPI*)(
    // This is the section handle emitted by NtCreateSection.
    IN HANDLE SectionHandle,
    // You'll typically use GetCurrentProcess here, but sections are shareable
    // across processes provided handles are inherited.
    IN HANDLE ProcessHandle,
    // If not nullptr, system tries to allocate memory from the specified value.
    IN OUT PVOID* BaseAddress OPTIONAL,
    // Indicates how many high bits must be unset in BaseAddress.
    IN ULONG_PTR ZeroBits OPTIONAL,
    // Size of initially committed memory. Will resize the file if this value
    // exceeds the file size on disk.
    IN SIZE_T CommitSize,
    // Pointer to the beginning of the mapped region in the section.
    IN OUT PLARGE_INTEGER SectionOffset OPTIONAL,
    // Pointer to the size of the mapped region in bytes. Rounded up to the page
    // size.
    IN OUT PSIZE_T ViewSize,
    // Indicates what should happen to handles when child processes are spawned.
    IN SECTION_INHERIT InheritDisposition,
    // Can be one of MEM_COMMIT or MEM_RESERVE
    IN ULONG AllocationType OPTIONAL,
    // Can be one of:
    // - PAGE_NOACCESS
    // - PAGE_READONLY
    // - PAGE_READWRITE
    // - PAGE_WRITECOPY
    // - PAGE_EXECUTE
    // - PAGE_EXECUTE_READ
    // - PAGE_EXECUTE_READWRITE
    // - PAGE_EXECUTE_WRITECOPY
    // - PAGE_GUARD
    // - PAGE_NOCACHE
    // - PAGE_WRITECOMBINE
    IN ULONG Protect);
static NtMapViewOfSectionFn NtMapViewOfSection;

using NtUnmapViewOfSectionFn = NTSTATUS(NTAPI*)(IN HANDLE ProcessHandle,
                                                IN PVOID BaseAddress);
static NtUnmapViewOfSectionFn NtUnmapViewOfSection;

using NtExtendSectionFn = NTSTATUS(NTAPI*)(IN HANDLE SectionHandle,
                                           IN PLARGE_INTEGER NewSectionSize);
static NtExtendSectionFn NtExtendSection;
```

I've taken the liberty here to document the parameters that aren't
self-explanatory. Let's go over each function in turn.

`NtCreateSection` creates a section which is another name for our file mapping
object. This is analogous to the user-space `CreateFileMapping` function and
provides an important `ACCESS_MASK` capability, namely, `SECTION_EXTEND_SIZE`.
Furthermore, we need to use this function to make a valid section handle for
use with the next function...

`NtMapViewOfSection` is analogous to `MapViewOfFile` but gives us the ability to
pass `MEM_RESERVE` as the allocation type. In particular, this means we can use
`NtMapViewOfSection` to map a _reserved_ virtual address range over a file that
exceeds the extent of the _physical_ range of the file.

The last ingredient here is the `NtExtendSection` function. When invoked on a
given section with a new size, this function commits pages and resizes the file.

To recap, we needed these functions to _reserve_ a mapped view of a file, which
can now extend beyond the actual file size on disk. The excess address space
reserved doesn't actually affect the file itself and can't be written to until
we call `NtExtendSection` to perform the resize operation. However, we can now
do this without needing to re-create the file mapping, remapping the view, and
invalidating pointers! Let's see what this looks like in action.

First, we need to actually query for all these function pointers. We aren't
implicitly linking `NTDLL.dll`, so we rely instead on explicit procedure address
fetches. This sort of thing can be done once at application start.

```cpp
HMODULE handle = GetModuleHandleA("NTDLL.DLL");

NtClose = (NtCloseFn)GetProcAddress(handle, "NtClose");

NtCreateSection = (NtCreateSectionFn)GetProcAddress(handle,
                                                    "NtCreateSection");

NtMapViewOfSection = (NtMapViewOfSectionFn)
    GetProcAddress(handle, "NtMapViewOfSection");

NtUnmapViewOfSection = (NtUnmapViewOfSectionFn)
    GetProcAddress(handle, "NtUnmapViewOfSection");

// This function is undocumented.
NtExtendSection = (NtExtendSectionFn)GetProcAddress(handle,
                                                    "NtExtendSection");
```

Next, we create a file handle to an as-yet uncreated file. This file handle is
usable with the other Nt functions we'll use soon.

```cpp
HANDLE file = CreateFileA(
    "test.txt",
    GENERIC_READ | GENERIC_WRITE,
    FILE_SHARE_READ,
    nullptr, // LPSECURITY_ATTRIBUTES
    OPEN_ALWAYS,
    FILE_ATTRIBUTE_NORMAL,
    nullptr)
```

With a file handle, we can now create the memory section.

```cpp
HANDLE section;

LARGE_INTEGER initial_size = {.QuadPart = 1};

NTSTATUS status = NtCreateSection(
    &section,
    SECTION_EXTEND_SIZE | SECTION_MAP_READ | SECTION_MAP_WRITE,
    nullptr, // Optional object attributes
    &initial_size,
    PAGE_READWRITE,
    SEC_RESERVE,
    file)
```

This part is perhaps the weirdest part here. In the second argument, it's
important that we pass `SECTION_EXTEND_SIZE` so we can resize it later. That's
not the weird bit though. The weird bit is our use of the initial size of 1
byte. This size argument normally indicates the maximum size we can map, and
also has the effect of potentially increasing (but not decreasing) the size of
the file. In this case, our `test.txt` file hasn't been created yet, so we actually end
up adding a single null-byte to its contents right off the bat. It turns out that
when using `NtCreateSection` backed by files, it's impossible to request 0
initial committed bytes. We'll discuss ways to workaround this limitation later.

Note also the use of `SEC_RESERVE` in the second to last argument. This will
cause mapped views of this section to _reserve_ address space without
immediately committing it. We could pass `SEC_COMMIT` here if we wanted. This
would have the effect of immediately allocating storage in physical memory
when we map the view in the next step. This would have the disadvantage of
throwing off our memory accounting, and also incurring more commit overhead
than would be strictly needed.

With the memory section created backed by the file handle, we can now close the
file handle and map the section.

```cpp
// Reserve a good amount of address space. This won't affect the actual file size.
size_t view_size = 32ull << 30;

// The file size at this point will be 1-byte, but in general, it could be larger.
BY_HANDLE_FILE_INFORMATION info;
GetFileInformationByHandle(file, &info);
size_t size = (size_t)info.nFileSizeHigh << 32 | info.nFileSizeLow;

// You can keep this file handle if you want (I do, mainly to reuse the handle
// for other functions later), but this is here to demonstrate that the file
// handle here doesn't need to persist any longer if not necessary.
CloseHandle(file);

char* data = nullptr;

status = NtMapViewOfSection(
    section,
    GetCurrentProcess(),
    (void**)&data,
    0,          // Mapped address zero bits
    size,       // Initial commit size
    0,          // Section offset
    &view_size, // Mapped size
    ViewUnmap,
    MEM_RESERVE,
    PAGE_READWRITE);
```

At this point, `data` should be mapped to actual memory backed by the file itself.
Of course, given that this was a new file, its size is exactly 1-byte. In the
initial mapping, we opt to initially commit memory based on the file size.
Importantly, we pass `MEM_RESERVE` as the allocation type. This way, even though
we opted to map 32 GiB of data, we aren't also committing it. This mirrors the
use of the `SEC_RESERVE` flag we passed to `NtCreateSection`.

Let's suppose we now want to write data to the file. First, we'd have to resize
it.

```cpp
char const* new_data = "hello";
size_t bytes = strlen(new_data);

status = NtExtendSection(section, (PLARGE_INTEGER)&bytes);

memcpy(data, new_data, bytes);
```

After the call to `NtExtendSection`, you should see the amount of committed
memory increase by a page size (4 KiB). However, the file itself will only
grow to be four bytes in size.

Let's quickly recap what we're now capable of:

- We can create file handles as before with [`CreateFile`](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilea).
- Instead of creating file mapping objects, we create an NT section backed by
  the file using [`NtCreateSection`](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/ntifs/nf-ntifs-ntcreatesection). We indicate that the section can be extended
  and should reserve memory instead of committing it.
- Instead of [`MapViewOfFile`](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-mapviewoffile),
  we use [`NtMapViewOfSection`](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nf-wdm-zwmapviewofsection) to associate a reserved
  address range that is much larger than however we anticipate the file to be.
- When needed, we simply call `NtExtendSection` to simultaneously grow the file
  size and commit the memory from the mapped base address.

## Supporting file truncation

One quirk here is that we can increase the size of the file with `NtExtendSection`,
but we can't decrease it. Truncation tends to be rarer, but in my experience,
there are a few viable approaches.

First, you can simply decide not to support truncation at all. This isn't as
crazy as it seems. Often, when mutating a file, the best approach is to simply
write data to a new one, and then _rename_ that new file to replace the old one
atomically. This approach ensures old data is recoverable, and also that the
data is never left in an inconsistent state. Consider, for example the
consequences of a power outage after _some_ dirty pages are flushed but not all.

Another approach is to simply destroy the section with `CloseHandle`, resize the
file using [`SetFilePointerEx`](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-setfilepointerex)
and [`SetEndOfFile`](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-setendoffile),
or a single call to [`SetFileInformationByHandle`](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-setfileinformationbyhandle),
followed by section recreation via `NtCreateSection` as before.
This truncation could be done eager or lazily when the file itself is destroyed, since
we need to track the size of the committed region anyways (if only to support
file size queries cheaply). Eager truncation has a significant drawback of invalidating
pointers, but you could _try_ to reserve the same region of memory you had access
to before. This is, of course, impossible to guarantee since some other part of
your program may reserve an overlapping range in the time it takes to remap the
section.

The approach I would have preferred (if it were possible) is one where the
committed memory can be transitioned back to the reserved state using `NtFreeVirtualMemory`,
and then recommitted after the `SetEndOfFile` truncation. Unfortunately,
attempting to decommit memory committed by `NtExtendSection` results in an
`UNABLE_TO_DELETE_SECTION` status code (`0xc000001b`), so this option isn't
available to us. This limitation also exists with non-`Nt` prefixed APIs as
well. If we create a `SEC_RESERVE` page-backed file mapping and commit pages
with `VirtualAlloc`, we can't later free those pages with `VirtualFree`.

My recommendation is to truncate immediately and document that existing pointers
are invalidated. While this is inconvenient and a potential footgun, it seems
better than the alternative of having files that are never successfully truncated
in the event of a program crash.

## Mapped read-only memory

With virtual memory, one thing worth considering is having a secondary region
of mapped memory that overlaps the first region set with read-only page
protections (`PAGE_READONLY`). This can be done with regular Winapi functions.

```cpp
HANDLE mapping = CreateFileMappingA(
    INVALID_HANDLE_FILE,
    nullptr,
    PAGE_READONLY | SEC_RESERVE,
    (DWORD)(size >> 32),
    (DWORD)(size & 0xffffffffu),
    nullptr);
    
void* readonly_data = nullptr;

MapViewOfFileEx(
    mapping,
    FILE_MAP_READ,
    (void**)&readonly_data,
    0u, // Offset high
    0u, // Offset low
    max_size,
    (void*)data);
```

This way, you can hand out a pointer to read-only memory in cases where users
should _not_ have write access. Attempting to modify pages mapped in this way
will cause an exception.

The idea here is to use `SEC_RESERVE` to ensure future mapped views aren't
committed, and then always keep `readonly_data` persistently mapped to
encompass the same range of data we associated with the file section. As the
file is resized, we would use `VirtualAlloc` to commit regions as needed.

If you wanted, you could support this type of thing in a `Debug` build
configuration, as an additional form of validation, but I've found that the extra
mapping doesn't introduce much overhead and is generally worth it. Remember that
aliasing views in this way are all coherent, so writes to memory in one section
are immediately visible in the aliased section (modulo cross-core cache consistency).

## Optimizing the working set

Memory that's committed isn't necessarily "active." Windows has the ability to
page the data in and out to the paging file. When the data is used by the
program, this data must become part of the working set (i.e. resident in
physical memory).

This promotion of committed but paged-out memory to the working set happens
transparently. In some cases, you may want to to manually indicate that parts
of a memory mapped file aren't needed. To do this, simply invoke `VirtualUnlock`
for as many ranges as needed. This will leave the affected pages committed and
remove them from the working set. Just be sure you aren't doing this on pages
that will be used soon, as this will result in unnecessary page faults. Note that
despite the name, you do _not_ need to call `VirtualLock` on memory you wish to
releases in this way first. As mentioned before, you cannot transition memory
committed in this approach back to the "reserved" state.

I use this in very narrow circumstances, typically when processing large files
sequentially, and consider it "advanced" API usage I'd only reach for with a
profiler handy. It's nice to have the option available though, particularly in
memory-constrained scenarios.

## Dealing with new files

As we've seen, when creating a new section, we were not able to initialize the
file with 0 initially committed bytes. In the demo code above, for a new file, we just
created the file and extended it to 1-byte immediately. My recommendation here
is to actually defer the section creation in this scenario until the initial
resize operation is called. This way, we don't add an empty byte to a newly
created file unnecessarily. Similarly, we would do this if the user opens a file
that has 0-size on disk.

## NtExtendSection vs NtAllocateVirtualMemory

After creating the section with the `SECTION_EXTEND_SIZE`, one other detail
worth knowing about is that `NtAllocateVirtualMemory` is now _also_ capable of
simultaneously committing memory and extending the file size. There is, however,
a crucial distinction between `NtAllocateVirtualMemory` and `NtExtendSection`.
Namely, while both routines commit memory at page granularities, the former will
also _change the file size_ at page granularity also.

For example, requesting 5 bytes of committed memory with `NtAllocateVirtualMemory`
will result in 4096 bytes of committed memory and a 4096 byte file (if the file
was smaller than 4096 bytes to begin with). Doing the same operation with
`NtExtendSection` will _also_ commit 4096 bytes, but the file on disk won't grow
beyond 5 bytes (unless it was larger than 5 bytes to begin with).

Note that if your application _only_ needs to operate on page-sized units of
persisted memory, you may not need `NtExtendSection` at all. This might be the
case for certain database implementations, for example.

## Gotchas to be aware of

If using memory-mapped I/O, it's important to never intermix `ReadFile` and
`WriteFile` invocations with mapped views of a file active. The data in the
mapped view will not be coherent with data read or written to from those functions,
which interact with other internal OS buffers.

Another gotcha to be aware of is that writes to memory-mapped regions aren't
immediately flushed to disk. This type of buffering is similar to buffering you'd
get with `WriteFile`, but when memory-mapping files, `FILE_FLAG_NO_BUFFERING` is
ignored. I generally advocate for staging writes to temporary files and renaming,
as mentioned earlier. Excessive flushing of file views and calls to
`FlushFileBuffers` dramatically impact performance in the typical case, and
regardless, it's extremely impractical to guarantee that writes survive some
catastrophe that interrupts the data pipeline flowing from RAM to hard storage.
When using the temp-file-and-rename strategy, use `SetFileInformationByHandle`
with the `FILE_RENAME_INFO` structure.

Additionally, it's important to remember that when incurring a page fault, the OS
suspends the thread that issued the faulting instruction until the page is ready.
In the memory mapped case, it's easy to inadvertently trigger page faults that
impact performance on the wrong thread at a bad time if you aren't careful. On
the flip side, it's also possible that a file manually committed into memory with
`ReadFile` is also paged out, resulting in the same problem. If random page faults
impact performance in a way that's unacceptable for your code, you're better off
reasoning about how to manage your memory access patterns accordingly, irrespective
of the I/O approach taken.

## Concurrent page faults

A legitimate problem here is the case where you need to read from or write to many
locations at once. With the memory-mapped scheme as described, you can only issue
as many concurrent page faults as you have threads in the program.

Unfortunately, I don't have great answers here, and view this as a legitimate use case
that warrants a dedicated code path using other mechanisms at your disposal. On
Windows, I have been exploring [DirectStorage](https://github.com/microsoft/DirectStorage)
for this use case, and may write more about it in the future as I gain confidence
in my own understanding. What would have been handy is an API that allows us to
make pages resident _without_ doing so synchronously.

The most similar option to this ideal worth considering is the [`PrefetchVirtualMemory`](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-prefetchvirtualmemory) call,
which acts as a strong hint that the supplied memory range should be made resident if
possible. This can be a useful accelerant, but there is no event emitted when the
memory is in fact resident (if it ever becomes resident). In a perfect world, we
would be able to asynchronously map memory ranges and latch on completion such
that subsequent access to those ranges will not fault. We are not in said perfect
world.

All that said, I actually consider this use case to be more niche, and anything
but the common path. For the most part, sticking to sequential processing and
relying on built-in caching behaviors gets you very far. Exotic use cases requiring
tons of random access across a very large dataset are a difficult beast to wrangle,
memory-mapping or otherwise.

## Conclusion

I think the main takeaways are:

- `ReadFile` and `WriteFile` are clunky to use, require an extra in-memory buffer,
  and incur more overhead in the typical case.
- Aside from specific async workloads, memory-mapping is far more convenient to
  work with, although undocumented APIs are needed to expose file-resizing
  functionality.
- For async work, faulting on background/worker threads is often "good enough",
  especially paired with sequential access and manual prefetching.
- If you somehow need even more bleeding-edge performance than that, you probably
  live in a world where you'd want to reach for dedicated APIs that interface
  with the storage driver and kernel IO routines in other ways.

Hopefully, the notes in this post are helpful in constructing your perfect `File`
abstraction, which accommodates the majority of use cases (at least on Windows).
On other platforms, these concepts also map reasonably well, and the main thing
to consider is whether your platform supports memory overcommit.