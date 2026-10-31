# Famfs - CXL

Shared memory between two QEMU VMs, exposed as a devdax device, with [famfs](https://famfs.org) mounted on top. Lab project for the Operating Systems course at NTUA (ECE).

The idea is to emulate a CXL shared-memory setup without CXL hardware. Both VMs are backed by the same file on the host, which QEMU exposes to each guest as an NVDIMM. Inside the guests it shows up as `/dev/dax0.0`. One VM acts as the famfs master and the other as a client.

**Note on performance numbers:** this is an emulation. The "shared memory" is a file on the host, so anything we measure reflects host DRAM/page cache and virtualization overhead, not real CXL latency or bandwidth. For reference, the Cxlalloc paper (ASPLOS '26) reports ~357 ns read latency and ~19.9 GB/s on an actual CXL device, compared to 112 ns and 114 GB/s for local DRAM. Our results are mostly about the software stack (famfs + mmap + QEMU).

## Setup

```
host (gold3)
  cxl-device/cxl-pmem.raw   <- shared backing file
      |                 |
   VM1 (:2224)       VM2 (:2225)
   /dev/dax0.0       /dev/dax0.0
   famfs master      famfs client
   /mnt/famfs        /mnt/famfs
```

- Host: `gold3`, VM storage on `/mnt/sdd2`
- Guests: Ubuntu with a custom kernel, `6.14.11-famfs_standalone`. The stock `6.8.0-111-generic` kernel has no famfs support.
- Kernel source: [`cxl-micron-reskit/famfs-linux`](https://github.com/cxl-micron-reskit/famfs-linux), branch `famfs_standalonev3`
- famfs user space: [`cxl-micron-reskit/famfs`](https://github.com/cxl-micron-reskit/famfs)

See [DEPLOY.md](DEPLOY.md) for how to bring everything up.

## How famfs works (short version)

famfs is a filesystem for memory that several hosts can map at the same time. Files are pre-allocated chunks of the shared device, and `mmap` on a famfs file gives direct access to that memory with no page cache in between.

Only one node, the master, writes metadata. It creates the filesystem and the files, and appends every change to a log stored in the shared memory itself. Clients replay that log to get the same view of the files. There is no delete or truncate, which is what keeps a client with stale metadata from ever pointing at the wrong data. famfs keeps its own metadata consistent but does not handle coherence for file contents, so that's up to the application. In practice this means one writer at a time, or write once and read from everywhere.

## Repo layout

```
src/        dax_test.c (raw devdax mmap test), famfs_stream.c (not written yet)
scripts/    QEMU launch scripts and helpers
kernel/     kernel .config and the commit we built from
docs/       literature notes and background
results/    raw numbers (csv) and plots
```

The backing file, VM images, kernel sources and build outputs are not in the repo (see `.gitignore`). The kernel source is public, so we only keep the branch, commit hash and config.

## Tools

**`src/dax_test.c`**: maps `/dev/dax0.0` directly, without famfs. This is what we used to first confirm that the two VMs actually see the same memory: write from one, read from the other.

```
gcc -O2 -Wall -o dax_test src/dax_test.c
sudo ./dax_test   # TODO: document arguments
```

Size and offset of the mapping must be multiples of 2 MB. With 4 KB it fails.

**`src/famfs_stream.c`**: planned STREAM-style benchmark (copy/scale/add/triad) over mmap'ed famfs files. It will also run on plain `malloc` memory as a DRAM baseline. Not written yet.

## Experiments

1. **Bandwidth over famfs.** STREAM-style kernels on mmap'ed famfs files.
2. **DRAM baseline.** Same benchmark on `malloc`, to put the famfs numbers in context.
3. **Raw devdax vs famfs mmap.** Tries to measure how much the filesystem layer costs on the data path. We expect it to be small since famfs only resolves page faults and stays out of the way afterwards, so this is more of a sanity check.
4. **Single-master model.** Look at what happens, and what's supported, when more than one node wants to write. famfs doesn't support concurrent multi-host writes by design.

Results go in `results/`. Since both VMs run on the same physical host, they share CPU and caches, so runs on both VMs at once will interfere with each other.

## Things we ran into

- devdax mmap needs 2 MB alignment. Anything assuming 4 KB pages just fails. famfs itself allocates in 2 MB units.
- The kernel build filled up `/mnt/sdd2` at one point and both VMs went into `paused (io-error)`. Check free space before building.
- After installing the new kernel, VM2 had no network until the initramfs was regenerated for that kernel version.
- After a power outage the device has to be brought back up by the lab admins. Numbers taken before that are not trustworthy, which is why some earlier results looked off.
- `famfs_standalonev3` is the in-kernel version of famfs. Upstream has since moved to a FUSE-based implementation, so this setup follows the earlier design.

## Status

- [x] Two VMs sharing one NVDIMM backing file
- [x] Cross-VM read/write through `/dev/dax0.0` with `dax_test`
- [x] famfs kernel built and booting on both VMs, networking ok
- [x] Cross-VM sharing through famfs
- [ ] Re-check the device after the power outage and redo measurements
- [ ] `famfs_stream.c`
- [ ] Experiments 1-4
- [ ] Report

## References

- M. K. Aguilera et al. *Remote regions: a simple abstraction for remote memory.* USENIX ATC 2018.
- J. Groves. *Introduce the famfs shared-memory file system* (RFC). https://lwn.net/Articles/963406/
- J. Groves. *Famfs and CXL Shared Memory: Progress, Challenges and Usability.* LPC 2024.
- J. Groves. *famfs: open source scale-out shared memory file system.* MSST 2025.
- J. Edge. *Famfs: a filesystem interface to shareable memory.* LWN, 2024. https://lwn.net/Articles/983105/
- N. Ni, Y. Sun, Z. Zhu, E. Witchel. *Cxlalloc: Safe and Efficient Memory Allocation for a CXL Pod.* ASPLOS 2026.

## License

TBD. famfs and the Linux kernel are GPL-2.0 and are not included here.
