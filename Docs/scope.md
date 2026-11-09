# Scope: Distributed Confidential Caching

**Author:** Korumilli Sai Vamsi · **Supervisor:** Dr. Valerio Schiavoni (Complex Systems Group, University of Neuchâtel)
**Programme:** Swiss Joint Master in Computer Science · **Submission:** on or before 9 March 2027
**Status:** draft for supervisor review, 9 November 2026

## Research question

> What does it cost, in latency and throughput, to keep a horizontally scaled in-memory cache confidential inside confidential VMs (AMD SEV-SNP and Intel TDX), and how does that cost change as the number of nodes grows?

Sub-questions:

1. **Per-node cost:** how much slower is one Valkey node in a confidential VM than in a normal VM on the same hardware?
2. **Scaling:** does that overhead stay constant, shrink or grow at 1, 2, 4 and 8 nodes?
3. **Where the cost comes from:** how much comes from running inside a confidential VM at all (memory encryption plus the confidential I/O path: extra VM exits and bounce buffers), how much from encrypted network traffic (TLS), and how much from attestation when a node joins?
4. **Vendor difference:** is the *relative* overhead of SEV-SNP (vs. a normal VM on AMD) different from that of TDX (vs. a normal VM on Intel)?

## Motivation

Brescia, Colonnelli, Schiavoni, Felber and Aldinucci (SysTEX 2025, "End-to-End Confidentiality with SEV-SNP Leveraging In-Memory Storage") show that keeping intermediate data in memory inside an SEV-SNP confidential VM gives end-to-end confidentiality without full-disk encryption, on a single machine. A distributed cache is the natural next step: data lives only in memory, but across many nodes and the network between them. This thesis measures what that costs.

## Work so far (9 Sept – 6 Nov 2026)

Background reading on TEEs, remote attestation, caching strategies and TEE benchmarking; evaluation of implementation options; design decisions: Valkey as the cache, Rust for the distributed layer, SEV-SNP and TDX as the two backends.

## Threat model

- **Trusted:** the CPU vendor's hardware and firmware root of trust; the guest OS and Valkey inside each attested confidential VM; the proxy (whether the proxy must itself run in a confidential VM is open question 2).
- **Untrusted:** the host OS, the hypervisor, the cluster or cloud operator, and the network between all components.
- **Not defended against:** denial of service by the host (it can always stop a VM), microarchitectural side channels, and bugs inside Valkey itself.

## What I build

Valkey runs **unmodified** inside each confidential VM. My contribution, written in Rust, is the distributed layer around it:

1. **Sharding proxy:** routes each key to a node using consistent hashing, so adding or removing one of N nodes moves only about 1/N of the keys.
2. **Attestation-gated join:** a node receives keys and traffic only after the proxy has verified its attestation evidence. The evidence's 64-byte report data is a hash over a fresh nonce from the proxy and the node's TLS public key. The nonce stops an old report from being replayed; the key binding stops a genuine report from being relayed by a different machine.
3. **Encrypted traffic:** client↔proxy and proxy↔node traffic uses TLS, with credentials released only after successful attestation and only for the public key bound in the evidence. Memory encryption does not protect data on the wire, so this layer is required.
4. **Generic TEE backend:** a `TeeBackend` trait with two implementations, SEV-SNP and TDX. On guests with direct access to the TEE (the group's servers, plain KVM), both obtain evidence through the Linux configfs-tsm interface (`/sys/kernel/config/tsm/report`, kernel 6.7+); verification stays vendor-specific (AMD VCEK certificate chain vs. Intel DCAP quote verification).
5. **Benchmark suite:** scripts that run every configuration, write CSV results and generate every plot in the thesis.

**Why TDX and not SGX as the second backend:** SEV-SNP and TDX both protect a whole VM, so the same unmodified Valkey and the same proxy run on both and the comparison is like for like. SGX protects a single process and would need a library OS (e.g. Gramine) around Valkey, which changes what is being measured.

**Why a proxy instead of Valkey Cluster mode:** Valkey Cluster already shards keys across nodes, and with `tls-cluster` it admits only nodes that hold a certificate from the configured CA. But membership is then decided by certificate possession and spread by gossip: any existing member can introduce a new node, and there is no single point that checks attestation at join time or can cut off one node. A proxy gives one place to enforce "no attestation, no keys, no traffic" without modifying Valkey. Valkey Cluster with TLS on the same confidential VMs is used as a comparison point in the evaluation.

## Evaluation plan

| Axis | Values |
| --- | --- |
| Platform | normal VM and confidential VM on AMD (SEV-SNP); normal VM and confidential VM on Intel (TDX) |
| TLS | off, on (to separate network-encryption cost from confidential-VM cost) |
| Cluster size | 1, 2, 4, 8 nodes |
| Workload | read-heavy (95% GET / 5% SET), write-heavy (50% / 50%); value size, key distribution (uniform vs. Zipfian), client connections and pipelining fixed in Phase 2 |
| Metrics | throughput (ops/s), p50 and p99 latency, node join time including attestation, proxy CPU usage |

This is 4 × 2 × 4 × 2 = 64 configurations, 320 runs at 5 runs each, plus the Valkey Cluster comparison.

- **Change one factor at a time.** Each confidential VM is compared against a normal VM on the same machine type, so the CPU is held constant and only the TEE changes. SEV-SNP and TDX are compared by their relative overhead, never by absolute numbers, because they run on different CPUs.
- **Baseline:** the same deployment (my proxy plus N unmodified Valkey nodes) on normal VMs, so the only difference to the confidential runs is the TEE. **Proxy cost:** one node reached directly vs. the same node reached through the proxy. **Comparison point:** Valkey Cluster with TLS in the same confidential VMs, without my proxy.
- **TLS-off runs are for measurement only.** They violate the threat model (the network is untrusted) and exist to isolate the cost of TLS; they are not a supported deployment.
- **Proxy as bottleneck.** All traffic passes through one proxy, so at 4 and 8 nodes the proxy may saturate before the cache nodes do. I record proxy CPU in every run and report the point where the proxy, not the nodes, limits throughput, rather than attributing that limit to the TEE.
- **Load generator** runs outside the confidential VMs, on a separate machine or pinned to separate cores, and its placement is stated with every result.
- **Tools:** `valkey-benchmark` and `memtier_benchmark` (both support TLS). Each configuration runs at least 5 times; I report the median with min/max.
- **Persistence (RDB/AOF) is turned off** in all runs, so results measure the in-memory path only.
- **Existing tooling:** I will check whether SNPGuard, the open-source framework that the SysTEX 2025 paper extends to boot and attest SEV-SNP VMs, can be reused for VM setup, and whether ConfBench (from the prerequisite reading) can drive part of the benchmarks.

## What I do not build (non-goals)

- A cache written from scratch, or changes to Valkey's internals
- An Intel SGX backend (stretch goal only, if Phase 4 runs ahead of schedule)
- Defences against side-channel attacks
- Confidential persistence to disk
- Replication, automatic failover or strong consistency guarantees beyond rebalancing keys when a node joins or leaves
- Production key management (HSM, key rotation service) or orchestration (Kubernetes)
- The secure-aggregation layer from the original combined proposal

## Phases and gates

Each gate ends with a short phase report to the supervisor.

| Phase | Dates | Gate |
| --- | --- | --- |
| 1 · Scope and first code | 9 – 13 Nov 2026 | Scope agreed; Valkey benchmarked locally; consistent-hashing ring tested |
| 2 · Baseline | 16 Nov – 4 Dec 2026 | Normal-VM vs. confidential-VM baseline numbers (SEV-SNP; TDX if hardware is available); phase report on Fri 4 Dec |
| 3 · Build | 7 Dec 2026 – 15 Jan 2027 | 3-node confidential cluster serves benchmark load; phase report on Fri 15 Jan |
| 4 · Evaluate | 18 Jan – 5 Feb 2027 | Full benchmark matrix done; code freeze on Fri 5 Feb; every plot generated by a script |
| 5 · Write and submit | 8 Feb – 5 Mar 2027 | Full draft to supervisor on Fri 19 Feb; submitted on Fri 5 Mar |

The background, threat-model and design chapters are drafted during Phases 2 and 3 from the phase reports, so Phase 5 covers results, discussion and revision. Phase 3 contains the Christmas and New Year break.

## Dependencies and risks

| Dependency or risk | Plan |
| --- | --- |
| Access to the group's SEV-SNP servers: requested 9 Nov, pending | Develop on Docker and normal VMs until access works; if still no working access by Fri 27 Nov, agree a fallback with the supervisor (e.g. Azure DCasv5 confidential VMs) |
| TDX-capable hardware not confirmed | Ask whether the cluster has a TDX-capable Intel Xeon; otherwise use cloud TDX VMs (e.g. Azure DCesv5) |
| Cloud confidential VMs attest differently | Azure confidential VMs run behind a paravisor: the hardware report is read from a vTPM instead of configfs-tsm, and the guest does not set the report data directly. A cloud fallback therefore needs a third, vTPM-based evidence path in the `TeeBackend`, or a provider that exposes the TEE directly. To be decided with the supervisor before choosing a cloud |
| Cloud baseline CPU may differ | A cloud provider's non-confidential VM family is not guaranteed to run on the same CPU generation as its confidential one. I record the CPU model of every VM and only compare pairs with the same model |
| Enough capacity for 8 confidential nodes plus proxy and load generator | Open question 5; if not, the largest cluster size is reduced and stated as a limitation |
| Cloud VMs add noise from shared hardware | Repeated runs; compare platforms only within the same environment |

## Open questions for the supervisor

1. Is there TDX-capable hardware in the group, or should TDX results come from cloud VMs?
2. Should the proxy itself run in a confidential VM and prove itself to clients, or is a trusted proxy acceptable for the thesis?
3. Is a normal VM the right baseline, with Valkey Cluster + TLS in confidential VMs as the comparison point, or does he expect a different reference?
4. Can SNPGuard be reused for booting and attesting the SEV-SNP VMs on the group's servers, and is ConfBench suitable for part of the measurements?
5. Can the group's servers host 8 confidential VMs plus the proxy and a load generator without them competing for the same cores?
6. Is TDX as the second backend, with SGX only as a stretch goal, acceptable given the earlier suggestion to cover SGX and SEV-SNP?

## Deliverables

- Open-source repository: Rust proxy, both TEE backends, benchmark and plotting scripts, setup documentation
- Master's thesis (submitted by 9 March 2027)
- Target: results strong enough for a workshop or conference submission after the thesis
