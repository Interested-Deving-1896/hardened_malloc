[update-readmes]   Mode: rewrite — migrating to template structure...
# hardened_malloc

[![Built with Ona](https://ona.com/build-with-ona.svg)](https://app.ona.com/#https://github.com/Interested-Deving-1896/hardened_malloc)

<!-- AI:start:what-it-does -->
_Description pending._
<!-- AI:end:what-it-does -->

## Architecture

<!-- AI:start:architecture -->
_Architecture documentation pending._
<!-- AI:end:architecture -->

## Install

<!-- Add installation instructions here. This section is yours — the AI will not modify it. -->

```bash
git clone https://github.com/Interested-Deving-1896/hardened_malloc.git
cd hardened_malloc
```

## Usage

<!-- Add usage examples here. This section is yours — the AI will not modify it. -->

## Configuration


You can set some configuration options at compile-time via arguments to the
make command as follows:

    make CONFIG_EXAMPLE=false

Configuration options are provided when there are significant compromises
between portability, performance, memory usage or security. The core design
choices are not configurable and the allocator remains very security-focused
even with all the optional features disabled.

The configuration system supports a configuration template system with two
standard presets: the default configuration (`config/default.mk`) and a light
configuration (`config/light.mk`). Packagers are strongly encouraged to ship
both the standard `default` and `light` configuration. You can choose the
configuration to build using `make VARIANT=light` where `make VARIANT=default`
is the same as `make`. Non-default configuration templates will build a library
with the suffix `-variant` such as `libhardened_malloc-light.so` and will use
an `out-variant` directory instead of `out` for the build.

The `default` configuration template has all normal optional security features
enabled (just not the niche `CONFIG_SEAL_METADATA`) and is quite aggressive in
terms of sacrificing performance and memory usage for security. The `light`
configuration template disables the slab quarantines, write after free check,
slot randomization and raises the guard slab interval from 1 to 8 but leaves
zero-on-free and slab canaries enabled. The `light` configuration has solid
performance and memory usage while still being far more secure than mainstream
allocators with much better security properties. Disabling zero-on-free would
gain more performance but doesn't make much difference for small allocations
without also disabling slab canaries. Slab canaries slightly raise memory use
and slightly slow down performance but are quite important to mitigate small
overflows and C string overflows. Disabling slab canaries is not recommended
in most cases since it would no longer be a strict upgrade over traditional
allocators with headers on allocations and basic consistency checks for them.

For reduced memory usage at the expense of performance (this will also reduce
the size of the empty slab caches and quarantines, saving a lot of memory,
since those are currently based on the size of the largest size class):

    make \
    N_ARENA=1 \
    CONFIG_EXTENDED_SIZE_CLASSES=false

The following boolean configuration options are available:

* `CONFIG_WERROR`: `true` (default) or `false` to control whether compiler
  warnings are treated as errors. This is highly recommended, but it can be
  disabled to avoid patching the Makefile if a compiler version not tested by
  the project is being used and has warnings. Investigating these warnings is
  still recommended and the intention is to always be free of any warnings.
* `CONFIG_NATIVE`: `true` (default) or `false` to control whether the code is
  optimized for the detected CPU on the host. If this is disabled, setting up a
  custom `-march` higher than the baseline architecture is highly recommended
  due to substantial performance benefits for this code.
* `CONFIG_CXX_ALLOCATOR`: `true` (default) or `false` to control whether the
  C++ allocator is replaced for slightly improved performance and detection of
  mismatched sizes for sized deallocation (often type confusion bugs). This
  will result in linking against the C++ standard library.
* `CONFIG_ZERO_ON_FREE`: `true` (default) or `false` to control whether small
  allocations are zeroed on free, to mitigate use-after-free and uninitialized
  use vulnerabilities along with purging lots of potentially sensitive data
  from the process as soon as possible. This has a performance cost scaling to
  the size of the allocation, which is usually acceptable. This is not relevant
  to large allocations because the pages are given back to the kernel.
* `CONFIG_WRITE_AFTER_FREE_CHECK`: `true` (default) or `false` to control
  sanity checking that new small allocations contain zeroed memory. This can
  detect writes caused by a write-after-free vulnerability and mixes well with
  the features for making memory reuse randomized/delayed. This has a
  performance cost scaling to the size of the allocation, which is usually
  acceptable. This is not relevant to large allocations because they're always
  a fresh memory mapping from the kernel.
* `CONFIG_SLOT_RANDOMIZE`: `true` (default) or `false` to randomize selection
  of free slots within slabs. This has a measurable performance cost and isn't
  one of the important security features, but the cost has been deemed more
  than acceptable to be enabled by default.
* `CONFIG_SLAB_CANARY`: `true` (default) or `false` to enable support for
  adding 8 byte canaries to the end of memory allocations. The primary purpose
  of the canaries is to render small fixed size buffer overflows harmless by
  absorbing them. The first byte of the canary is always zero, containing
  overflows caused by a missing C string NUL terminator. The other 7 bytes are
  a per-slab random value. On free, integrity of the canary is checked to
  detect attacks like linear overflows or other forms of heap corruption caused
  by imprecise exploit primitives. However, checking on free will often be too
  late to prevent exploitation so it's not the main purpose of the canaries.
* `CONFIG_SEAL_METADATA`: `true` or `false` (default) to control whether Memory
  Protection Keys are used to disable access to all writable allocator state
  outside of the memory allocator code. It's currently disabled by default due
  to a significant performance cost for this use case on current generation
  hardware, which may become drastically lower in the future. Whether or not
  this feature is enabled, the metadata is all contained within an isolated
  memory region with high entropy random guard regions around it.

The following integer configuration options are available:

* `CONFIG_SLAB_QUARANTINE_RANDOM_LENGTH`: `1` (default) to control the number
  of slots in the random array used to randomize reuse for small memory
  allocations. This sets the length for the largest size class (either 16kiB
  or 128kiB based on `CONFIG_EXTENDED_SIZE_CLASSES`) and the quarantine length
  for smaller size classes is scaled to match the total memory of the
  quarantined allocations (1 becomes 1024 for 16 byte allocations with 16kiB
  as the largest size class, or 8192 with 128kiB as the largest).
* `CONFIG_SLAB_QUARANTINE_QUEUE_LENGTH`: `1` (default) to control the number of
  slots in the queue used to delay reuse for small memory allocations. This
  sets the length for the largest size class (either 16kiB or 128kiB based on
  `CONFIG_EXTENDED_SIZE_CLASSES`) and the quarantine length for smaller size
  classes is scaled to match the total memory of the quarantined allocations (1
  becomes 1024 for 16 byte allocations with 16kiB as the largest size class, or
  8192 with 128kiB as the largest).
* `CONFIG_GUARD_SLABS_INTERVAL`: `1` (default) to control the number of slabs
  before a slab is skipped and left as an unused memory protected guard slab.
  The default of `1` leaves a guard slab between every slab. This feature does
  not have a *direct* performance cost, but it makes the address space usage
  sparser which can indirectly hurt performance. The kernel also needs to track
  a lot more memory mappings, which uses a bit of extra memory and slows down
  memory mapping and memory protection changes in the process. The kernel uses
  O(log n) algorithms for this and system calls are already fairly slow anyway,
  so having many extra mappings doesn't usually add up to a significant cost.
* `CONFIG_GUARD_SIZE_DIVISOR`: `2` (default) to control the maximum size of the
  guard regions placed on both sides of large memory allocations, relative to
  the usable size of the memory allocation.
* `CONFIG_REGION_QUARANTINE_RANDOM_LENGTH`: `256` (default) to control the
  number of slots in the random array used to randomize region reuse for large
  memory allocations.
* `CONFIG_REGION_QUARANTINE_QUEUE_LENGTH`: `1024` (default) to control the
  number of slots in the queue used to delay region reuse for large memory
  allocations.
* `CONFIG_REGION_QUARANTINE_SKIP_THRESHOLD`: `33554432` (default) to control
  the size threshold where large allocations will not be quarantined.
* `CONFIG_FREE_SLABS_QUARANTINE_RANDOM_LENGTH`: `32` (default) to control the
  number of slots in the random array used to randomize free slab reuse.
* `CONFIG_CLASS_REGION_SIZE`: `34359738368` (default) to control the size of
  the size class regions.
* `CONFIG_N_ARENA`: `4` (default) to control the number of arenas
* `CONFIG_STATS`: `false` (default) to control whether stats on allocation /
  deallocation count and active allocations are tracked. See the [section on
  stats](#stats) for more details.
* `CONFIG_EXTENDED_SIZE_CLASSES`: `true` (default) to control whether small
  size class go up to 128kiB instead of the minimum requirement for avoiding
  memory waste of 16kiB. The option to extend it even further will be offered
  in the future when better support for larger slab allocations is added. See
  the [section on size classes](#size-classes) below for details.
* `CONFIG_LARGE_SIZE_CLASSES`: `true` (default) to control whether large
  allocations use the slab allocation size class scheme instead of page size
  granularity. See the [section on size classes](#size-classes) below for
  details.

There will be more control over enabled features in the future along with
control over fairly arbitrarily chosen values like the size of empty slab
caches (making them smaller improves security and reduces memory usage while
larger caches can substantially improves performance).

## CI

<!-- AI:start:ci -->
_CI documentation pending._
<!-- AI:end:ci -->

## Mirror chain

<!-- AI:start:mirror-chain -->
This repo is maintained in [`Interested-Deving-1896/hardened_malloc`](https://github.com/Interested-Deving-1896/hardened_malloc) and mirrored through:

```
Interested-Deving-1896/hardened_malloc  ──►  OpenOS-Project-OSP/hardened_malloc  ──►  OpenOS-Project-Ecosystem-OOC/hardened_malloc
```

Changes flow downstream automatically via the hourly mirror chain in
[`fork-sync-all`](https://github.com/Interested-Deving-1896/fork-sync-all).
Direct commits to OSP or OOC are detected and opened as PRs back to `Interested-Deving-1896`.
<!-- AI:end:mirror-chain -->

## Contributors

<!-- AI:start:contributors -->
_Contributors pending._
<!-- AI:end:contributors -->

## Origins

<!-- AI:start:origins -->
_Original project — no upstream fork._
<!-- AI:end:origins -->

## Resources

<!-- AI:start:resources -->
_No additional resource files found._
<!-- AI:end:resources -->

## License

<!-- AI:start:license -->
[MIT](https://github.com/Interested-Deving-1896/hardened_malloc/blob/main/LICENSE) © 2026 [Interested-Deving-1896](https://github.com/Interested-Deving-1896)
<!-- AI:end:license -->
