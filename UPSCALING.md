# DPI Engine Upscaling Plan

This document covers how to take the DPI Engine from a learning prototype to a reliable, scalable traffic-classification tool. It starts with problems found in the current code, then lays out the work in priority order.

See also [upgrade.md](upgrade.md) for the original phase-by-phase roadmap.

---

## 1. Current State (Audit)

Findings from building and running the code on macOS (Apple clang, C++17):

| # | Issue | Where | Impact |
|---|-------|-------|--------|
| 1 | The CMake build only produces `packet_analyzer` (a packet viewer). None of the DPI code is built. | [CMakeLists.txt](CMakeLists.txt) | `cmake --build` does not give you a DPI tool. |
| 2 | The modular engine does not compile on macOS: the enum value `DOMAIN` clashes with a `DOMAIN` macro in the system math headers. | [include/rule_manager.h:93](include/rule_manager.h#L93) | `main_dpi.cpp` and the engine sources fail with `error: expected identifier`. |
| 3 | App classification uses substring matching, so `www.netflix.com` (contains `x.com`) and `www.microsoft.com` (contains `t.co`) are labelled **Twitter/X**. | [src/types.cpp:110-111](src/types.cpp#L110-L111) | Wrong reports, and wrong blocks when blocking Twitter/X. |
| 4 | Five separate entry points (`main.cpp`, `main_dpi.cpp`, `main_simple.cpp`, `main_working.cpp`, `dpi_mt.cpp`), with logic duplicated between them. | `src/` | Unclear which one is the product; fixes have to be made in several places. |
| 5 | No automated tests. | — | Changes cannot be checked for regressions. |
| 6 | VLAN-tagged frames and IPv6 are not inspected (IPv6 is only recognised for printing). | [src/packet_parser.cpp](src/packet_parser.cpp) | That traffic goes through unclassified, and nothing reports it. |
| 7 | SNI and HTTP Host extraction only looks at a single packet. | [src/sni_extractor.cpp](src/sni_extractor.cpp) | A TLS Client Hello split across TCP segments is missed. |
| 8 | Build outputs and captures are tracked in git (`dpi_engine` binary, `output.pcap`). | repo root | Repo bloat; a stale binary may not match the source. |

What already works well:

- `dpi_mt.cpp` builds on its own and correctly identifies most SNIs in `test_dpi.pcap`.
- The multi-threaded design hashes each connection to a fixed worker thread, so all packets of a connection are handled by the same worker. This is the right basis for scaling.
- `thread_safe_queue.h` already has a bounded queue with shutdown support.

---

## 2. Quick Fixes (Do First)

1. **Rename the clashing enum value**: `DOMAIN` → `DOMAIN_NAME`, or switch to `enum class`.
2. **Fix domain matching**: match the exact domain or a subdomain (the host equals `x.com` or ends with `.x.com`). Never use a plain substring search. Lowercase the name and strip any trailing dot first.
3. **Add a `.gitignore`** for `build/`, compiled binaries and generated `.pcap` outputs, and remove the tracked ones from git. Keep small test captures in a `tests/fixtures/` folder.
4. **Pick one entry point**, move the rest to `examples/` or delete them, and update the README to match.

---

## 3. Phase 1: One Buildable, End-to-End Tool

- One CMake project with:
  - a `dpi_core` static library (parser, PCAP I/O, connection tracker, SNI extractor, rules)
  - a `dpi_engine` executable
  - a `dpi_tests` executable
- Command line: `dpi_engine -i in.pcap -o out.pcap [--rules rules.yaml] [--threads N] [--json report.json]`.
- Keep packet timestamps and the capture header in the output file.
- Report counts for packets read, parsed, forwarded, dropped and unsupported. Read + parsed must add up, and so must forwarded + dropped.
- Turn on warnings (`-Wall -Wextra -Wpedantic`) and add build presets for debug, release and sanitizers (ASan/UBSan, TSan).

**Done when:** a clean CMake build passes on macOS and Linux, a block rule removes only the matching packets, and the counts agree with what Wireshark shows (`capinfos`).

---

## 4. Phase 2: Tests and CI

- Use GoogleTest or Catch2, fetched with CMake `FetchContent`.
- Unit tests for:
  - PCAP reading: valid, truncated and bad-magic-number files
  - Ethernet, IPv4, TCP and UDP parsing, including short and malformed headers
  - SNI and HTTP Host extraction from captured byte samples
  - Rule matching: IP, port, app and domain, including edge cases for subdomains
- An end-to-end test: run a small fixed capture through the binary and compare the output file and the JSON report with expected copies.
- Fuzzing: libFuzzer targets for the packet parser and the SNI extractor, since both read untrusted bytes.
- GitHub Actions: build and test on `ubuntu-latest` and `macos-latest`, plus a sanitizer job.

---

## 5. Phase 3: Protocol and Correctness

- **VLAN (802.1Q / QinQ)** and **IPv6**, including walking the IPv6 extension headers.
- Check bounds on every field read. Never trust a length field in the packet without checking it against the captured length.
- **Limited per-connection TCP reassembly** (for example, buffer up to 16 KB per connection until the Client Hello or HTTP headers are complete), with a timeout.
- **IP fragments**: reassemble them, or count them and skip them explicitly. Never parse a partial transport header.
- **Connection lifecycle**: track SYN, FIN and RST, expire idle connections, and cap the connection table size so memory stays bounded.
- Write down the limits of classification. SNI can be absent, or hidden by ECH (Encrypted Client Hello). Guessing the app from the port is a hint, not proof.

---

## 6. Phase 4: Features That Add Real Value

| Feature | Why |
|---------|-----|
| **TLS fingerprints (JA3 / JA4)** | Identify client software (browser, malware, bots) even without SNI. |
| **DNS parsing** | Link IP addresses to the domain names they were looked up as, which helps when SNI is missing. |
| **QUIC / HTTP/3 Initial parsing** | A large share of Google, YouTube and Meta traffic uses QUIC over UDP 443, which the engine currently cannot see. |
| **Versioned rules file (YAML/JSON)** | Clear error messages, per-rule counters and comments. Consider a hot-reload option. |
| **JSON / NDJSON output** | Per-connection records and summary stats that other tools can read. |
| **A recorded reason for each block** | Every dropped connection records which rule matched it. |
| **Reading pcapng files** | The default format Wireshark saves in. |
| **Aho-Corasick or a suffix trie for domains** | Fast matching against thousands of domain rules. |

---

## 7. Phase 5: Performance and Scale

Benchmark first, then optimise based on the numbers.

1. **Benchmark setup**: a large synthetic PCAP (1–10 GB) and a replay of real traffic. Measure packets per second, Gbps, and processing time per packet (p50/p99). Use `perf` or Instruments to find hot spots.
2. **Zero-copy parsing**: parse in place over the PCAP buffer (`std::span` or pointer + length), and memory-map (`mmap`) the input file instead of reading packets one at a time.
3. **Multi-threading**: build on the design in `dpi_mt.cpp`:
   - Hash connections to workers symmetrically, so both directions of a connection go to the same worker.
   - Use bounded lock-free single-producer/single-consumer ring buffers instead of mutex-based queues.
   - Shut down cleanly: drain the queues, flush the output, and propagate worker errors.
   - Test that output with N threads matches the single-threaded output, after sorting packets back into order.
4. **Memory**: pre-allocate packet buffers in a pool, use a flat hash map for the connection table (e.g. `absl::flat_hash_map` or `robin_hood`), and avoid creating `std::string` objects on the hot path.
5. **Batching**: process packets in bursts of 32–256 to make better use of the CPU cache.

Rough targets: the single thread at 1–2 Mpps from a memory-mapped PCAP, and close to linear speed-up up to about 8 workers.

---

## 8. Later: Live Capture and Deployment

Live traffic is a separate milestone. It brings platform-specific code, root/admin privileges, the risk of dropping packets, and privacy concerns.

- **Capture back ends**:
  - libpcap: portable, the easy first step
  - AF_PACKET with TPACKET_V3: Linux
  - AF_XDP or DPDK: 10G+ line rate
- **Inline blocking**: Linux NFQUEUE (iptables/nftables) lets the engine accept or drop real packets. A filtered PCAP file does not block anything on a live network.
- **Observability**: Prometheus metrics (pps, drops, top apps, rule hits) and a Grafana dashboard, or a small web UI that reads the JSON stats.
- **Packaging**: a Dockerfile, a systemd service, and a config file.
- **Privacy**: by default, keep metadata only, never stored payloads. Document data handling and retention.

---

## 9. Suggested Order

1. Quick fixes (Section 2)
2. Phase 1: one tool and one CMake build
3. Phase 2: tests and CI
4. Phase 3: VLAN, IPv6, reassembly
5. Phase 4: JA3/JA4, DNS, QUIC, JSON output
6. Phase 5: benchmarks, then performance work
7. Live capture and the dashboard

Each step leaves the project working and testable, and later steps depend on the earlier ones being correct.
