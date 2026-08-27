# Shimmer-net

Shimmer is an ultra-low latency, memory-optimized security wrapper built on Linux eBPF/AF\_XDP kernel bypass. Designed for high-throughput environments, Shimmer operationalizes Automated Moving Target Defense (AMTD) by dynamically morphing network layer signatures, mutating transport parameters, and executing per-packet polymorphic encryption—delivering hyper-secure communication without kernel overhead or latency spikes.

---

## Key Features

### Kernel Bypass Data Plane
Built on AF\_XDP zero-copy memory buffers (UMEM) for single-digit microsecond packet processing. Shimmer bypasses the Linux network stack entirely, eliminating copy overhead and context switches that traditional socket-based solutions incur. Packets are processed directly in userspace via eBPF programs pinned to XDP hooks, achieving wire-speed throughput on commodity hardware.

### Polymorphic Crypto Ratchet
Advances per-packet AEAD encryption keys inline using lightweight symmetric ratchets without session handshakes. Each transmitted packet derives a fresh encryption key from the previous packet's ratchet state, ensuring forward secrecy at the per-packet granularity. The ratchet runs entirely in the data path with no round-trip key exchange, keeping cryptographic overhead sub-microsecond.

### Moving Target Defense (AMTD)
Dynamically shuffles transport parameters, header structures, and IP paths to block network mapping. Shimmer continuously rotates source/destination port assignments, IP TTL values, TCP options ordering, and DSCP markings on a per-flow or per-packet schedule driven by a cryptographically seeded PRNG. Adversarial scanners and traffic classifiers are unable to build stable fingerprints of protected endpoints.

### Side-Channel Mitigation
Disrupts packet size distributions and timing signatures to defeat machine learning traffic analysis. Shimmer applies inter-packet delay jitter and length normalization (padding + fragmentation) to obscure application-layer traffic patterns. Statistical fingerprinting attacks—including deep packet inspection and flow-level ML classifiers—are thwarted by continuously randomizing observable metadata.

### Low Memory Footprint
Cache-optimized, lock-free ring buffers pre-allocated via HugePages with zero runtime allocations. All packet descriptors and cryptographic state are allocated at startup into UMEM regions backed by 2 MB HugePages. The hot path contains no heap allocations, no locks, and no system calls beyond the initial AF\_XDP socket setup, keeping L1/L2 cache pressure minimal even at multi-Mpps rates.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│  NIC Driver                                                 │
│    │  XDP hook (eBPF)  ─────────────────────────────────┐  │
│    │                                                     │  │
│    ▼                                                     ▼  │
│  AF_XDP Socket                                  XDP TX Queue│
│    │  UMEM (HugePages)                                   │  │
│    │  ┌──────────────────────────────────────────────┐   │  │
│    │  │ RX Ring  →  Shimmer Data Plane               │   │  │
│    │  │              ├─ AMTD Header Mutation          │   │  │
│    │  │              ├─ Polymorphic Crypto Ratchet    │   │  │
│    │  │              ├─ Side-Channel Padding/Jitter   │   │  │
│    │  │              └─ TX Ring  ──────────────────── ┘   │  │
│    │  └──────────────────────────────────────────────┘   │  │
└─────────────────────────────────────────────────────────────┘
```

---

## Requirements

- Linux kernel ≥ 5.10 (AF\_XDP with zero-copy support)
- A NIC with XDP zero-copy driver support (e.g., Intel i40e, mlx5)
- `libxdp` / `libbpf` ≥ 1.0
- HugePages configured on the host (`/proc/sys/vm/nr_hugepages`)
- Root or `CAP_NET_ADMIN` + `CAP_BPF` capabilities

---

## Getting Started

### 1. Configure HugePages

```bash
echo 512 | sudo tee /proc/sys/vm/nr_hugepages
```

### 2. Build

```bash
make
```

### 3. Run

```bash
sudo ./shimmer --iface eth0 --queue 0
```

### 4. Verify

```bash
sudo ./shimmer --iface eth0 --stats
```

---

## Configuration

| Option | Default | Description |
|---|---|---|
| `--iface` | *(required)* | Network interface to attach to |
| `--queue` | `0` | NIC hardware queue index |
| `--umem-size` | `4096` frames | Number of UMEM frames (must be power of 2) |
| `--frame-size` | `4096` bytes | UMEM frame size |
| `--amtd-interval` | `100` packets | Packets between AMTD mutation events |
| `--ratchet-algo` | `chacha20poly1305` | AEAD algorithm for the crypto ratchet |
| `--jitter-max-us` | `50` µs | Maximum inter-packet delay jitter |
| `--pad-to` | `1400` bytes | Normalize packet length to this value |
| `--hugepages` | `true` | Use HugePages-backed UMEM |
| `--zero-copy` | `true` | Enable AF\_XDP zero-copy mode |

---

## Security Model

Shimmer treats the network as an untrusted, adversarially observed channel. The combined defense-in-depth stack addresses:

1. **Passive eavesdropping** — per-packet AEAD with forward-secret ratchet keys
2. **Traffic analysis** — length normalization and timing jitter break statistical classifiers
3. **Network fingerprinting** — AMTD mutations prevent stable endpoint identification
4. **Replay attacks** — monotonic packet counters embedded in each ratchet step reject replayed frames
5. **Side-channel timing** — constant-time cryptographic primitives throughout the hot path

---

## License

See [LICENSE](LICENSE).
