<!-- Header -->
<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=0:0d1117,55:0b3d2c,100:00e59b&height=190&section=header&text=Sachin%20Pandhare&fontSize=50&fontColor=e6edf3&fontAlignY=36&desc=low-latency%20%C2%B7%20high-throughput%20trading%20systems&descSize=17&descAlignY=58&descColor=9be9c8" alt="Sachin Pandhare — low-latency, high-throughput trading systems" width="100%">
</p>

<p align="center">
  <a href="https://github.com/breakbadsp">
    <img src="https://readme-typing-svg.demolab.com?font=JetBrains+Mono&weight=500&size=17&duration=2600&pause=900&color=00E59B&center=true&vCenter=true&width=720&lines=matching+engines+%C2%B7+order+books+%C2%B7+FIFO+price-time;SPSC+rings%2C+pinned+cores%2C+zero+hot-path+allocation;Viewstamped+Replication+for+crash-safe+exchanges;C%2B%2B26+by+day%2C+Rust+by+choice" alt="matching engines · order books · SPSC rings · Viewstamped Replication · C++ and Rust">
  </a>
</p>

<p align="center">
  <a href="https://x.com/breakbadsp"><img src="https://img.shields.io/badge/@breakbadsp-000000?style=for-the-badge&logo=x&logoColor=white" alt="X @breakbadsp"></a>
  <a href="https://github.com/breakbadsp?tab=repositories"><img src="https://img.shields.io/badge/repositories-161b22?style=for-the-badge&logo=github&logoColor=white" alt="Repositories"></a>
</p>

### `$ whoami`

I build trading infrastructure where the budget is counted in **nanoseconds**: order books, matching engines, gateways, and the queues and allocators underneath them. Professionally a C++ engineer, increasingly writing **Rust**. I like designs you can measure, so most of my repos ship with a benchmark, a profiler run, or a fault simulator next to the code.

<p align="center">
  <img src="assets/numbers.svg" alt="2.39M ops/s over loopback TCP (p99 47 µs); ~73 ns per command; 3.75M ops/s with 8 clients; triangulator collector at 4.08% of a core for 1,000 threads" width="100%">
</p>

<p align="center">
  <img src="assets/pipeline.svg" alt="Order path: clients → gateway (epoll/io_uring) → SPSC rings → pinned matching engines → libvsr replication" width="100%">
</p>

---

### ⚡ Exchanges & trading infrastructure

<table>
<tr>
<td width="50%" valign="top">

<h3><a href="https://github.com/breakbadsp/modern_exchng_cpp">modern_exchng_cpp</a></h3>
`C++26` `epoll` `io_uring` `SPSC` `contracts`

A complete exchange: limit order book with price-time (FIFO) matching, a busy-polling gateway with **epoll and io_uring** backends, one **pinned engine thread per core** fed by SPSC rings, and a load generator. Fixed-capacity pools, no heap allocation on the hot path. Replicated across a cluster with libvsr, plus a browser **fault-tolerance game** to crash replicas and watch it survive.

**2.39M ops/s** · p99 **47 µs** end-to-end over TCP ([docs](https://github.com/breakbadsp/modern_exchng_cpp/blob/master/docs/performance.md))

</td>
<td width="50%" valign="top">

<h3><a href="https://github.com/breakbadsp/libvsr">libvsr</a></h3>
`C++23` `consensus` `deterministic simulation`

A generic **Viewstamped Replication** library inspired by TigerBeetle. Keeps a deterministic state machine identical across machines and survives crashes and power loss. The app supplies three functions (execute, snapshot, restore); libvsr replicates opaque bytes.

Verified by a **seeded simulator** that runs the production replica code over a fake network, disk and clock, injecting loss, delay, reordering and storage faults.

</td>
</tr>
<tr>
<td width="50%" valign="top">

<h3><a href="https://github.com/breakbadsp/spx">spx</a></h3>
`Rust` `cargo workspace`

A low-latency stock exchange in Rust: separate crates for the **matching engine**, order server, message types and a concurrency lib. Price-time matching with a per-symbol order book.

</td>
<td width="50%" valign="top">

<h3><a href="https://github.com/breakbadsp/MktDataAggregator">MktDataAggregator</a></h3>
`C++` `mmap` `k-way merge`

Merges ~10,000 per-symbol market-data files (100+ GB) into one time-ordered tape for backtesting. **K-way merge** over memory-mapped, chunked readers, built to stay under OS file-handle limits.

</td>
</tr>
</table>

### 🧵 Concurrency & systems primitives

| Project | What it is | Under the hood |
|:--|:--|:--|
| **[sp](https://github.com/breakbadsp/sp)** | The STL, rebuilt from scratch in modern C++ | lock-free MPSC queue, spinlock, latch, object & memory pools, open-addressing hash set, `shared_ptr` |
| **[spmalloc](https://github.com/breakbadsp/spmalloc)** | A `malloc`/`free`/`realloc` of my own | `sbrk` heap, block splitting & coalescing, `LD_PRELOAD`-able `.so` |
| **[1brc](https://github.com/breakbadsp/1brc)** | One Billion Row Challenge in C++ | `mmap` zero-copy input, core-pinned threads, per-thread FNV-1a open-addressing maps |
| **[rust_mpsc_queue](https://github.com/breakbadsp/rust_mpsc_queue)** · **[rust_mpmc_queue](https://github.com/breakbadsp/rust_mpmc_queue)** | Channels in Rust, built to study MPSC vs MPMC contention | `Arc<Mutex>` + `Condvar`, receiver-side batching |
| **[ping_pong_server](https://github.com/breakbadsp/ping_pong_server)** | Minimal Rust TCP round-trip server | `std::net`, buffered I/O |

### 🔭 Observability

<table>
<tr>
<td valign="top">

<h3><a href="https://github.com/breakbadsp/triangulator">triangulator</a></h3>
`C++23` `TypeScript` `/proc` `UDP` `SQLite`

See what **every thread** of a Linux process is doing, live or recorded: CPU demand, scheduler wait channels, memory, I/O and sockets. A C++ sampler reads `/proc` and streams UDP to a collector that stores SQLite day files and serves a dashboard with **time-travel replay**. Rewriting the collector from Python to C++ cut it from **32.2% → 4.08%** of a core at 1,000 threads, and dashboard p95 from **78.8 ms → 0.89 ms**.

</td>
</tr>
</table>

---

### 🛠️ Toolbox

<p align="center">
  <img src="https://skillicons.dev/icons?i=cpp,rust,c,linux,cmake,python,ts,git&theme=dark&perline=9" alt="C++, Rust, C, Linux, CMake, Python, TypeScript, Git">
</p>

<p align="center">
  <code>lock-free queues</code> · <code>cache-line layout</code> · <code>CPU pinning</code> · <code>epoll / io_uring</code> · <code>mmap</code> · <code>custom allocators</code> · <code>perf</code> · <code>ASan</code> · <code>UBSan</code> · <code>deterministic simulation</code>
</p>

### 📈 Activity

<p align="center">
  <img src="https://streak-stats.demolab.com?user=breakbadsp&theme=github-dark-blue&hide_border=true&background=0D1117&stroke=30363D&ring=00E59B&fire=00E59B&currStreakNum=00E59B&currStreakLabel=00E59B&sideNums=E6EDF3&sideLabels=8B949E&dates=6E7681&disable_animations=true" alt="GitHub streak for breakbadsp" height="170">
</p>

<p align="center">
  <img src="https://github-readme-activity-graph.vercel.app/graph?username=breakbadsp&bg_color=0d1117&color=8b949e&line=00e59b&point=e6edf3&area=true&area_color=00e59b&hide_border=true&custom_title=Contributions%20(last%2030%20days)" alt="Contribution graph for breakbadsp" width="100%">
</p>

<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=0:00e59b,45:0b3d2c,100:0d1117&height=110&section=footer" alt="" width="100%">
</p>
