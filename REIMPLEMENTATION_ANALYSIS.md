# Re-implementing Overpass API in Java 27, Go, or Rust

Status: estimate only, written 2026-11-01. It is based on the structure of this codebase (release 0.7.62) and the Go 1.26 and 1.27 release notes. No prototype was built and no benchmarks were run.

Constraints for a port: keep the same on-disk file format, the same ingestion path, and the same HTTP API.

## Recommendation

**Rust.** It is the only one of the three likely to match or slightly beat the current C++ on query speed and memory use. Go is the pragmatic second choice: the least porting effort, at moderate runtime cost. Java fits worst, because of how much memory small records take and because it doesn't work with the one-process-per-query model.

Ranking for query performance and low server resource use: **Rust > Go > Java**.

## What a port has to reproduce

The codebase is about 106k lines of C++:

| Module | Lines |
|---|---|
| `src/overpass_api/statements` | 35.3k |
| `src/overpass_api/osm-backend` (ingestion) | 20.2k |
| `src/template_db` (storage engine) | 12.5k |
| `src/overpass_api/data` | 10.3k |
| `src/overpass_api/core` | 9.7k |
| `src/overpass_api/frontend` | 4.3k |
| `src/overpass_api/dispatch` | 3.6k |
| `src/overpass_api/output_formats` | 3.2k |

What determines language fit:

- **Storage engine.** `template_db` is heavily templated (`Block_Backend<Index, Object>`, about 130 template declarations). Records are fixed-layout binary structs read with raw pointer casts (about 200 sites). Blocks are compressed with zlib or LZ4 (`zlib_wrapper`, `lz4_wrapper`). Index and data files are read with `mmap` (`file_blocks.h:794`, `file_blocks_index.h:441`).
- **Dispatcher.** It coordinates readers and writers over POSIX shared memory (`shm_open` in `dispatcher.cc` and `dispatcher_client.cc`) plus a unix socket, including the swap to a new index version. A port must reproduce this protocol bit-exactly to share a database with the existing tools.
- **HTTP API.** It runs as CGI, one process per query, with rlimit memory caps and timeouts. Killing the process is how runaway queries are stopped.
- **Query hot path.** Decompress blocks, decode skeletons into `std::map<Index, std::vector<Skeleton>>`, then do set operations, geometry, and XML/JSON/CSV output. Large queries can use gigabytes of heap made of many small 12–40 byte records.

## Comparison

| | Rust | Go | Java 27 |
|---|---|---|---|
| Same file format | Straightforward: `#[repr(C)]` structs via `zerocopy`/`bytemuck`, `memmap2` | OK: `unsafe` plus `encoding/binary`, with manual layout | OK: FFM `MemorySegment` / `ValueLayout`, but verbose |
| Replacing the C++ templates | Monomorphized generics, a 1:1 replacement | Generics compiled per GC shape, so generic hot loops run slower | Generics are erased. Small records become objects unless Valhalla value classes are usable, and they may still be preview-only |
| Small-record memory | Same as C++ | Same layout, but the GC needs heap headroom (about 1.5–2× peak) | Object headers plus pointers cost about 2–3×, unless you hand-write column-style `long[]`/`int[]` storage |
| Process per query (current model) | Native, about 1 ms startup | Fine, a few ms startup | Poor: startup and JIT warm-up are lost on every query, even with CDS/Leyden AOT |
| Long-running server instead | Possible: thread per query, per-query arena, counting allocator for hard memory caps | Goroutines can't be killed, and `GOMEMLIMIT` applies to the whole process, so process isolation is still needed | Required, but there are no per-query heap limits, so one runaway query can cause GC thrash or OOM for everyone |
| Ingestion (XML/OSC parsing, bulk sort/merge) | `quick-xml` is very fast | Standard `encoding/xml` is slow, so a third-party parser is needed | StAX/Aalto is good after warm-up |
| Compression | `lz4_flex`, `flate2` (zlib-ng) | Pure-Go lz4/zlib, somewhat slower, or C libraries via cgo | `java.util.zip`, lz4-java |
| Query speed vs. current C++ (estimate) | 0.9–1.2× | 0.6–0.85× (with Go 1.26/1.27, see below) | 0.6–1.0× once warm. Cold, per process, much worse |
| Server memory (RSS) vs. C++ | about 1× | about 1.3–1.8× (with Go 1.26/1.27) | about 2–4×, plus a 200–500 MB baseline |
| Port effort | Highest (borrow checker vs. the shared-mutable statement/set graph, `unsafe` in storage) | Lowest | Medium |

Where Rust gains over C++: better hash maps, sorting, and B-trees than `std::map` (a red-black tree), plus safe in-process limits per query.

## General notes

- **Disk I/O is a wash.** All three read through `mmap`/`pread`, so they share the OS page cache the same way the C++ does. The differences show up in decompression, decoding, set operations, and heap size, which is where Rust's lack of a garbage collector and its value-type layouts pay off.
- **Architecture matters more than language.** Most of the realistic gain over the current setup would come from replacing CGI fork-per-query with a pre-forked or threaded server that keeps decoded indexes warm. Rust can do that safely because it can enforce per-query memory caps in-process. Go and Java can't cleanly, so they effectively push you back to process-per-query.
- **Suggested order for any port.** Port `template_db` and the dispatcher protocol first. Then run a back-to-back test against the C++ binaries on the same database directory, like the existing map-file replication test (commit `bf581419`), before touching `statements/`.

## Go 1.26 and 1.27: what helps

Source: the official release notes, <https://go.dev/doc/go1.26> and <https://go.dev/doc/go1.27>. Go 1.27 was released 2026-08-19.

| Release | Feature | Status | Effect for Overpass |
|---|---|---|---|
| 1.26 | Green Tea GC: "10–40% reduction in garbage collection overhead", about 10% more on Ice Lake / Zen 4 and newer | Default (opt-out `nogreenteagc`) | **Biggest win.** Large queries build heaps of many small skeleton records, which is exactly the workload this GC targets |
| 1.27 | Size-specialized malloc: allocations under 80 bytes up to 30% cheaper | Default | Skeleton and tag records are 12–40 bytes. The notes estimate about 1% in real programs |
| 1.27 | Generic methods (a method may declare its own type parameters) | Default | Lets the Go API mirror `Block_Backend<Index,Object>` more closely. Ergonomics only, not speed |
| 1.26 | Generic types may refer to themselves in their own type-parameter list | Default | Helps express the C++ index-type constraints |
| 1.26 | Slice backing arrays allocated on the stack in more cases | Default | Fewer heap allocations for temporary buffers when decoding blocks |
| 1.27 | Faster `compress/flate`, which also speeds up `compress/zlib` | Default | Speeds up **ingestion**, because writing blocks means compressing them. The notes only mention compression, so queries may not benefit |
| 1.26 | cgo call overhead about 30% lower | Default | Makes calling C liblz4/zlib-ng through cgo for block decompression more viable |
| 1.27 | `simd` (any architecture) and `simd/archsimd` (amd64, arm64 Neon, wasm) | `GOEXPERIMENT=simd`, API not stable | Could speed up bbox/polygon filtering and coordinate decoding. Not ready for production |
| 1.27 | `encoding/json` backed by v2: unmarshal "significantly faster", marshal "at parity" | Default | Little effect. Overpass writes its JSON output with hand-written code |
| 1.27 | Goroutine leak profile, pprof labels in tracebacks | Default | Easier to run a long-lived query server |
| 1.27 | HTTP/2 client priority signals (RFC 9218), `Server.MaxHeaderValueCount` | Default | Minor, if CGI is replaced with a built-in `net/http` server |

Older Go features that are also useful (from memory, not re-checked against the release notes): Swiss-table maps (1.24), `weak` and `runtime.AddCleanup` (1.24), `unique` for interning tag strings (1.23), range-over-func iterators for block scans (1.23), and container-aware `GOMAXPROCS` (1.25).

### Go: still missing

- No arenas and no per-goroutine memory limit. `GOMEMLIMIT` applies to the whole process, so one runaway query can't be capped inside a shared server. You still need a process per query, or at least a pre-forked worker per query.
- Generics are still compiled per GC shape, not per type, so the hot `Block_Backend` loops pay indirection that C++ templates and Rust don't.
- `encoding/xml` hasn't changed. OSM/OSC ingestion still needs a third-party streaming XML parser.

Effect on the estimate: the Go range moves from 0.5–0.8× to **0.6–0.85×** of C++ query speed, and memory from 1.5–2× to **1.3–1.8×**. The ranking is unchanged.

## Open next steps

- Prototype `template_db` read paths (`Block_Backend` range scan plus LZ4/zlib decompression) in Rust and Go against an existing database directory, and benchmark them against C++.
- Decide whether to keep CGI process-per-query or move to a long-running server. This decides more about the outcome than the language does.
- Check the status of Java Valhalla value classes in JDK 27 if Java stays under consideration.
