## 1. Cheetah

### What is the main problem with stateful layer 4 load balancers that the stateless version of cheetah solved?

Stateful load balancers require a lot of memory and complex data structures that can only be deployed on software switches, while stateless cheetah can be also deployed on hardware switches

### What is Per-Connection-Consistency (PCC) and is it difficult for traditional stateless load balancers to guarantee PCC?

Per connection consistency is the ability to send all packets of a given connection to the same server, even when servers or load balancers are added or removed.

It is difficult for traditional stateless load balancers because they rely on hash based distribution without per-connection state, any change in the server set reshuffles some of the existing connections to the new server, violating PCC to some extent.

### How are flow entries inserted, looked up, and removed from the Cheetah stateful LB? What is the advantage compared to existing stateful LBs?

The stateful Cheetah LB uses a connection table and a stack of free indices.

On a new connection, it pops an index from the stack, stores the flow state at that table entry, and writes the index as a cookie in the packets.

Subsequent packets carry this index so the LB can directly access the correct entry, and when the flow ends the index is pushed back onto the stack.

This design makes insertion, lookup, and deletion O(1) in the data plane, unlike traditional cuckoo-hash-based stateful LBs whose insertions can trigger cascading relocations and become slow.

### Consider a network using Stateful Cheetah, where the cookie information added into a packet only contains the index of the flow entry in the connection table. Explain a scenario in which this stateful Cheetah mechanism cannot guarantee PCC. How would you fix this problem?

If the cookie only stores the index into the stateful connection table, PCC breaks when a stateful load balancer fails (or traffic is rerouted) and a different LB receives packets for an existing flow but has no valid state at that index (or has a different flow stored there), possibly also after servers have changed. This can be fixed either by synchronizing the stateful tables across LBs or by also including the stateless Cheetah cookie on all packets so any LB can reconstruct the correct server mapping from the cookie.

### What is an attack that could be launched against a Stateful Cheetah load balancer? Explain one possible countermeasure

An attacker can flood the stateful Cheetah LB with many TCP SYNs using different 5‑tuples, exhausting the connection table and blocking legitimate flows, and a countermeasure is to use TCP SYN cookies so the LB allocates a connection-table entry only after a full, valid 3‑way handshake completes

### What is the key idea of Cheetah and what problem does it solve?

The key idea of Cheetah is to encode the connection‑to‑server mapping into a cookie carried in every packet of a connection, so the mapping is stored in the packet header rather than only in the LB’s state. This solves the problem of guaranteeing Per‑Connection‑Consistency while still allowing arbitrary load-balancing mechanisms and dynamic changes in the number of servers and load balancers.

## 2. APLOMB

### What are the main implications of the survey circulated by the APLOMB authors on the deployment of network functions in a network?

Enterprise networks have a surprisingly high amount of middle boxes and that the operational costs of owning and maintaining the hardware is quite high

### What are the three redirection techniques proposed in APLOMB? Why the authors select one of them instead of the other two techniques?

APLOMB proposes three redirection techniques: bounce redirection, IP redirection, and DNS redirection.

In bounce redirection, the enterprise gateway tunnels all incoming and outgoing traffic to a cloud PoP, where virtual middle-boxes process it, and the cloud then tunnels it back to the enterprise.
This adds an extra “bounce” via the cloud for every packet and causes excessive latency.

In IP redirection, the cloud advertises the enterprise’s IP prefix via BGP from multiple PoPs so client traffic is routed directly to some PoP, processed there, and then tunneled to the enterprise gateway, but BGP chooses which PoP is used and the forward and reverse paths may traverse different PoPs, breaking stateful middle-boxes and giving the enterprise no control over PoP choice.

In DNS redirection, the cloud runs DNS on behalf of the enterprise and answers client lookups with the IP of a specific cloud PoP, then uses the APLOMB gateway to ensure both directions of each flow go through that same PoP; this avoids the extra latency of bounce redirection and the path asymmetry and lack of control of IP redirection, so the authors select DNS redirection.

### What are some of the remaining concerns in the APLOMB paper 'Making middle-boxes someone elses problem'

The main remaining concerns are the increased security risk of sending all enterprise traffic to a third‑party cloud and the extra bandwidth required to detour all traffic through the cloud, which raises cost and performance concerns.

### How is APLOMB improving latency for some src/dst pairs, although it goes through an extra hop

APLOMB sends traffic via a cloud PoP (the extra hop). Because cloud providers sit on very fast, well‑peered backbones, the path:

    Enterprise -> Cloud -> Client
    
can sometimes have lower latency than:

    Enterprise -> Client

directly via the default Internet path.

### How much state must be stored at the Aplomb gateway in order to route packets to the correct cloud?

In order to route packets to the correct cloud the Aplomb gateway needs to know for each IP source prefix to which cloud it should be sent to.

## 3. CLick

### What is the main motivation for deploying a software switch like Click?

The main reason to deploy software switches like Click is programmability. Traditional routers only offer functions that can be turned on and off, whereas Click is a modular software that allows you to program complex routing and compose packet processing functions.

### What is the role of the queue elements in the system? What happens if one doesn’t add any queue element?

Queue elements are explicit buffers that sit between push and pull elements in Click, storing packets after processing so that schedulers or devices can pull them when ready.

Without any queues, packets cannot be buffered or scheduled, so the router cannot absorb bursts and will drop packets as soon as a downstream element (like the NIC) is not ready to accept them.

### Explain the impact of the placement of the Queue element in the data-flow packet processing graph on the performance of Click

In Click, once a packet starts being processed, the CPU must run it through all elements until it reaches a Queue, so if the queue is placed late, the CPU is forced to do a lot of work per packet before it can switch to another one. Placing queues earlier in the graph reduces this per-packet commitment, letting the CPU interleave work on different packets more flexibly and improving responsiveness for latency-sensitive traffic under load.

### Why has Click switched to using polling instead of interrupts to receive packets?

Click switched to polling because, at very high line rates, interrupts cause many context switches and cache misses, which add too much overhead. By dedicating a core to polling the NIC, Click avoids these context switches and achieves better throughput under heavy load, at the cost of using a core almost fully.

## 4. Metron

### Explain the three key Metron points. In other words, why is it so fast?

1. Offload stateless work to hardware
    Metron pushes the expensive stateless classification and header processing of NF chains into programmable switches and NICs, so CPU cores only handle the stateful parts.

2. Tag packets by traffic class early
    It identifies traffic classes (equivalent packets for the chain), and the first programmable switch both applies stateless rules and tags packets according to their class.

3. NIC dispatches directly to the right core
    The NIC uses these tags to send packets straight to the correct core, avoiding software dispatchers and inter‑core transfers, so processing happens at cache speed.

### Explain two issues with Metron

When a core is overloaded, Metron tries to split the traffic class group assigned to that core and move a subgroup to another, lightly loaded core using a new tag.

However, some traffic classes are unsplittable (e.g., a single heavy HTTP flow), so Metron cannot divide their load across cores, and in those cases it cannot reduce the load on the overloaded core.

Metron’s big speedup comes from offloading stateless work (e.g., large firewall ACLs) and using hardware tagging/dispatch to avoid inter‑core transfers.

When most of the service chain is stateful and cannot be offloaded, more work must run on server cores over slow memory, so Metron’s performance advantage over OpenBox/E2 becomes much smaller.

### Does the Metron‑provided performance benefit increase or decrease as you increase the amount of stateful computation on the servers? Why? (Hint: recall the DPI 40‑Gbps experiment vs the original 100‑Gbps service chain

The Metron benefit decreases as stateful computation on servers increases.

Metron gains most when it can offload expensive stateless classification to hardware and avoid inter‑core transfer.

When more of the chain is stateful and must run on server cores, like in the 100‑Gbps Router+NAPT+LB chain, less can be offloaded so the speedup over OpenBox/E2 becomes much smaller compared to the 40‑Gbps Firewall+DPI case.

## 5. Split RPC

### What does SplitRPC promote? i.e., what was the paper about?

SplitRPC promotes a split {control + data} path design, where a small fixed-size control header is sent to the CPU while the large data payload is DMA’ed directly from the NIC to GPU memory. This minimizes data movement through host memory and the kernel stack while preserving CPU-based orchestration of GPU kernels, which improves end-to-end latency and throughput for ML inference.

### Why is SplitRPC-sNIC slower than SplitRPC-pNIC on ResNet18?

SplitRPC‑sNIC is slower than SplitRPC‑pNIC on ResNet18 because the SmartNIC version runs the control and network-stack logic on a relatively slow ARM NIC core, which becomes the bottleneck when the model is small and GPU compute is fast. In contrast, SplitRPC‑pNIC offloads that control work to the much more powerful host CPU, so for ResNet18 the pNIC can drive more requests per second before saturating.

### Look at the figure from the paper, what was the likely batch size for resnet18 with SplitRPC-pNIC?

For ResNet18, the figure shows a GPU time of about 588 µs, which I approximate as 0.5 ms, so a batch‑1 GPU can serve roughly 2000 requests/s. Under SplitRPC‑pNIC, the throughput with dynamic batching is about 7000 RPS, so the likely batch size is 7000 / 2000 ≈ 3.5, i.e., around 3–4 requests per batch.

### Why is open-loop necessary to find SplitRPC’s saturation point

With a closed-loop generator, each client waits for a response before sending the next request, so when SplitRPC (or any server) slows down the clients automatically send fewer requests and the system never gets pushed far past saturation. With an open-loop generator, the arrival rate is fixed and independent of completions, so when the offered load exceeds the server’s capacity you see the true saturation point and a sharp spike in p95/p99 latency that a closed-loop setup would hide.

## 6. RedN / RDMA

### Why is RedN Important?

RedN is important because it shows that commodity RDMA NICs are Turing complete, so they can run arbitrary computations using only RDMA verbs. This lets us offload complex work from expensive, power-hungry CPUs onto cheap RDMA NICs, and these offloaded RDMA verb chains can keep serving requests even if the kernel on the CPU crashes.

### How many if statements can RedN execute per second and why

RedN can execute only about 0.5 million if statements per second, because each if is implemented as a CAS that must cross PCIe to host DRAM and back, which takes on the order of a few microseconds per operation. With roughly 2 microseconds per CAS, the PCIe round-trip latency directly bounds the rate at about 1 / (2 × 10^-6) ≈ 5×10^5 operations per second (≈ 0.5 million ops/s).

### Compare RedN and SmartNIC offloads

SmartNICs include extra compute (CPU cores or FPGAs) on the NIC, so they can offload very complex tasks, but they are several times more expensive and harder to manage than ordinary RDMA NICs. RedN instead shows that commodity RDMA NICs can be made effectively Turing complete by running self‑modifying RDMA verb chains, so we can offload operations like hash‑table lookups onto cheap RDMA NICs instead of buying and managing costly SmartNICs

### Why does using CAS and WAIT/ENABLE let RDMA implement conditionals and loops?

CAS is used like an if: it compares a value in memory with an expected value, and if they match the NIC can change a later work request (for example, turning a NOOP into a WRITE), so that instructions run conditionally.

WAIT tells the NIC not to execute some work requests until previous ones have completed, so it can safely use the results of earlier operations.

ENABLE lets the NIC mark a sequence of work requests as ready to execute, and by re‑enabling earlier work requests it can reuse a previous block, effectively creating loops.

### What are practical limitations of using RDMA for complex offloads?

Practical limitations include that complex RDMA offloads are bound by PCIe latency (microseconds per host DRAM access), so fine-grained CAS-based logic runs at only about 0.5 M ops/s, and that the programming model is fragile and vendor-specific, relying on low-level verb chains and fields with limited tooling and no compiler support.

## 7. vLLM / Paged Attention

### what key problem does vLLM solve and how?

vLLM solves KV-cache fragmentation by splitting the KV cache into fixed-size blocks and mapping those logical pages to non-contiguous physical memory, which makes memory usage more flexible and efficient

### what does vLLM enable almost trivially?

vLLM enables efficient KV-cache sharing by mapping logical KV blocks to non-contiguous physical memory, so different sequences can reuse shared prompt blocks and only copy data when a shared block needs to be modified. This makes beam search, shared-prefix caching, and similar decoding patterns much easier

### Is Attention in need of high memory bandwidth?

Yes. During auto-regressive decoding, attention is very demanding on memory bandwidth because each new token needs access to the KV cache of all previous tokens, so performance is limited by how fast that cache can be read from GPU memory

### Why is the KV cache critical for batching?

The KV cache is critical for batching because it is the main memory cost per request. If multiple requests share a common prefix, the system can reuse that prefix’s KV cache across requests, which saves memory and lets more requests fit in the batch.

### What is Paged Attention?

PagedAttention is an attention algorithm that splits each sequence’s KV cache into fixed-size KV blocks and lets these blocks be stored in non-contiguous GPU memory. For each request, vLLM keeps a block table that maps logical KV blocks (the sequence order) to physical KV blocks in DRAM, so the attention kernel can follow this mapping to fetch the right blocks while avoiding fragmentation and enabling sharing.

### Describe block-size trade-offs (performance vs fragmentation) and typical default choices

Block size trades off fragmentation against overhead and swap efficiency.

Large blocks reduce the number of blocks per sequence and thus reduce indirections, but they increase internal fragmentation because each sequence can waste almost one full block in its partially filled tail.

Small blocks greatly reduce fragmentation but require many more block-table lookups and, if swapping is used, create many tiny PCIe transfers that cannot fully utilize bandwidth, making swapping inefficient.

In practice, vLLM uses a medium block size to balance these effects, and for small blocks it often prefers recomputation over swapping because recomputation cost is nearly independent of block size while swap overhead grows sharply with smaller blocks.

### Describe vLLM’s copy-on-write mechanism for parallel sampling and how reference-counted physical blocks are used

In parallel sampling, the prompt KV blocks are shared across all samples. When a sample eventually needs to write into a shared KV block (where multiple samples still point), vLLM allocates a new block, copies the old data into it, and then that sample continues writing into its private block, leaving the other samples’ shared block unchanged.

## 8. Sarathi Serve

### What is the key idea in Sarathi Serve?

The key innovation in Sarathi-Serve is to recognize that prefill and decode have very different hardware behavior: prefill is compute‑intensive, while decode is more memory‑bound and benefits a lot from batching.

Sarathi-Serve introduces chunked‑prefills, splitting each prefill into fixed‑size chunks and scheduling at most one prefill chunk together with many decodes in each iteration, which avoids long stalls and visible lags in token streaming.

### Suppose your system had a few cheap gaming GPUs with lots of computational power but relatively slow memory, and server‑grade GPUs with lots of fast and expensive memory. Both are connected with NVLink (not possible today, but imagine). How would you structure prefill and decode operations on the system? What should run where and why?

With cheap gaming GPUs that have strong compute but slower memory, and server‑grade GPUs with lots of fast memory, you would run prefills on the gaming GPUs (batch size ≈ 1, using their compute) and then ship the activations / KV cache over NVLink to the server‑grade GPUs, where you run the decode phase in large batches using their big, fast memory.

### Why doesn't prefill benefits from batching?

Prefill does not benefit much from batching because it is very compute‑intensive and already processes all tokens of a prompt in parallel, so even with batch size 1 it essentially saturates the GPU; increasing the batch size adds little extra throughput.

### Why does decode benefits from batching?

Decode generates only one token per request per iteration, so a single request under‑utilizes the GPU; by batching many decode requests together, we can keep the GPU busy and greatly improve throughput.

### how does having a fixed token budget per iteration help Sarathi-Serve control time‑between‑tokens (TBT)?

Sarathi-Serve sets a fixed token budget per iteration and always fills it first with all ongoing decodes, then with at most one prefill chunk, so that the work per iteration stays roughly constant and the time‑between‑tokens (TBT) stays within the latency SLO.

### Why are previous approaches inefficient?

Previous approaches are inefficient because request‑level batching wastes compute on padding shorter sequences up to the longest one and then keeps running with a shrinking effective batch size as some requests finish. In addition, iteration‑level schedulers that admit unconstrained full prefills into decode iterations create large, variable iteration times that cause generation stalls and unreliable, high time‑between‑tokens (TBT).

## 9. Janus

### What is Janus’s data-centric paradigm, and why does it reduce inter-node traffic compared with the usual expert-centric approach?

In the usual expert-centric MoE setup, experts are fixed on GPUs and, after routing, each GPU sends its token activations across machines to the GPUs hosting the selected experts, then receives the expert outputs back; this incurs two large All-to-All communications per MoE layer and can make the inter-node network the bottleneck. In Janus’s data-centric paradigm, tokens stay on their home GPUs and the system instead pulls the required expert weights to those GPUs; because each expert module is much smaller than the batch’s token activations, each machine transfers each expert only once per iteration and then reuses it from a local cache, which eliminates the All-to-All token shuffles and can reduce inter-node traffic by up to 16×

### Janus treats every expert pull as an independent task. Explain how (a) the credit-based buffer in each GPU and (b) the intra-/inter-node cache hierarchy together overlap computation with communication and relieve RDMA and NVLink congestion

Each GPU maintains a credit-based buffer for expert pulls: when it needs to fetch an expert, it consumes a credit and issues the pull; while that expert is being copied, the GPU computes on a previously fetched expert, and only when that computation finishes is the credit released. This limits the number of concurrent pulls so RDMA/NVLink are not overloaded, while guaranteeing that communication is overlapped with useful computation.

### Why can't Janus be used for inference?

Janus is a training framework built for fixed-length, teacher-forced MoE training where All-to-All communication dominates and the R formula can choose between moving tokens or experts once per job; in inference and speculative decoding, sequence lengths and batch shapes change per step, the main bottlenecks become FFN and KV-cache memory bandwidth rather than MoE All-to-All, and there is no backward pass to hide expert movement, so Janus’s mechanisms do not apply.

## 10. Mega Scale Infer

### Why does MegaScale-Infer need a custom M2N communication library instead of NCCL for token dispatch and aggregation?

MegaScale-Infer uses a custom M2N communication library instead of NCCL because NCCL is optimized for collective patterns like AllReduce/All-to-All, while disaggregated attention–expert layers generate a dynamic M-senders to N-receivers token dispatch/aggregation pattern between the two clusters. In this pattern, NCCL’s GPU→CPU→NIC copies, group-initialization overhead, and implicit GPU synchronizations cause high and unstable latency, so MegaScale-Infer’s library instead uses RDMA WRITE-with-immediate and GPUDirect to send tensors directly GPU→NIC→network→NIC→GPU, removing CPU bounces and collective setup and adding ACK-prioritization and tuned congestion control for much better median and tail latency.

### What are the key ideas of Mega Scale Infer?

The key idea of MegaScale-Infer is to exploit the asymmetry in MoE decoding by dis-aggregating attention and expert FFNs onto separate GPU clusters so each can use hardware and parallelism tuned to its bottleneck.

Attention is dominated by KV-cache reads and is memory-bandwidth/capacity-bound, while the MoE FFN side is dominated by expert GEMMs and wants compute-dense GPUs and large effective batch sizes, so putting both on the same GPUs under utilizes one side.

MegaScale-Infer puts attention on a memory-rich “M-cluster” (e.g., H20) and experts on a compute-dense “N-cluster” (e.g., L40S), then uses ping-pong pipeline parallelism to micro-batch requests and keep both clusters busy while overlapping M2N/N2M communication with computation

### Is attention in need of high memory bandwidth? Why?

Yes. Attention during decoding needs high memory bandwidth because, for each token in the batch, it has to read that request’s entire KV cache (K and V for all previous tokens across all heads), which is large and grows with sequence length and batch size. This makes the attention module spend most of its time moving KV data between GPU memory and compute units, so its performance is limited by how fast memory can be read and written, not by raw FLOPs

### What is ping-pong pipeline parallelism in MegaScale-Infer, and why is it needed?

Ping-pong pipeline parallelism in MegaScale-Infer means splitting the global decode batch into several micro-batches and alternating them between the attention (M) and expert (N) clusters so that different micro-batches are in different stages at the same time.

While micro-batch i is in attention on the M-cluster, an earlier micro-batch i−1 is in the FFN experts on the N-cluster and the M2N/N2M transfers for another micro-batch are happening, which hides most of the cross-cluster communication and prevents either cluster from sitting idle.

## 11. Spec Infer

### What is the key idea of Spec Infer?

SpecInfer uses small speculative models to generate a tree of candidate token sequences instead of a single sequence, and then runs the large LLM once as a token‑tree verifier, using tree‑based parallel attention to check all branches in parallel while preserving the same output distribution as standard incremental decoding.

### How would using SpecInfer affect time-to-first-token (TTFT)? Would TTFT go up or down compared to plain decoding, and why?

SpecInfer increases TTFT, because after the usual prefill the system must first let the SSM speculate and then run an extra verification pass on the large LLM before emitting the first token. However, it reduces the per-token latency after TTFT, since each verification step can accept multiple speculated tokens in parallel while preserving the original model’s output distribution

### Can SpecInfer be applied to any arbitrary model that you might be using for inference (e.g., a random closed-source LLM API)? Explain why or why not

SpecInfer cannot be applied to arbitrary black‑box models, because it requires small speculative models that are trained or distilled to closely match the target LLM’s output distribution. Without such aligned SSMs, most speculated tokens would be rejected, so speculative decoding would add overhead instead of speeding things up

### You want to speed up an agentic workflow or complex multi-step tool-using assistant on your own PC. Which paper would you choose to rely on more heavily, CacheBlend or SpecInfer, and why?

For an agentic workflow or complex multi-step tool-using assistant on a local machine, CacheBlend is the more appropriate paper. It directly optimizes RAG-style long-context inputs by reusing and selectively recomputing KV caches across chunks, reducing TTFT and increasing throughput without hurting quality.

SpecInfer accelerates token-by-token decoding via speculative trees, but does not directly address the cache reuse patterns that dominate multi-step, retrieval-heavy workflows

### SpecInfer suffers from tree-attention blocking the GPU while the big model is verifying. Why didn’t the authors simply use Sarathi-Serve’s chunked prefill style to reduce this blocking, and what kind of challenges or assumptions might prevent directly combining the two?

Because it’s fundamentally a topology problem: Sarathi‑Serve’s chunked prefill assumes a linear sequence split into contiguous time‑chunks, while SpecInfer’s LLM verification runs tree attention over a token tree with shared prefixes and divergent branches. The bottleneck is the attention topology: to combine them you’d need a notion of ‘tree chunks’ that preserves the tree’s parent–child relationships and KV reuse, which is nontrivial and not addressed in the paper


### Why didn't the authors of the paper use sarathi serve to speed up the problem of the tree verification holding up the GPU?

Because it’s fundamentally a topology problem: Sarathi‑Serve’s chunked prefill assumes a linear sequence split into contiguous time‑chunks, while SpecInfer’s LLM verification runs tree attention over a token tree with shared prefixes and divergent branches.

The bottleneck is the attention topology: to combine them you’d need a notion of ‘tree chunks’ that preserves the tree’s parent–child relationships and KV reuse, which is nontrivial and not addressed in the paper.

## 12. Cache Blend

### What is the key idea of cache blend?

The key idea of cache blend is to reuse the kv cache from document chunks to speed up RAG without sacrificing accuracy.

At initialization time prefill is run on all the chunks so that we can store the kv cache.

When a question is asked to the LLM the chunks related to that question are retrieved and their kv caches are concatenated. The problem now is that the single chunks have self attention but it's missing the cross attention between different chunks.

Cache blend, layer by layer re-computes the attention of the HKVD tokens (thos tokens that have a high error) comparing it with all the tokens from all chunks (This is why it scales quadratically, not n^2 but still k^2 with k << n)


### Explain how TTFT vs chunk size behaves in CacheBlend. What is the shape of this relationship, and why is it still quadratic?

In CacheBlend, each layer loads concatenated cached KV for all n tokens, but only recomputes attention for a small subset k of high-error tokens, which still attend over all n tokens. This makes the per-layer cost scale like a constant times n^2, so TTFT vs. chunk size is still quadratic, but with a much smaller constant than full prefill because k << n, giving a 2-3x TTFT reduction.

### How does Cache blend pipelines operations?

CacheBlend pipelines operations by recomputing HKVD tokens' KV on layer l while simultaneously loading the cached KV for layer l+1, so the latency per step is bounded by the maximum of compute time and load time, not their sum. The controller chooses a recompute fraction and storage tier such that the per-layer recompute time is no longer than the KV load time, allowing KV caches to sit on cheaper, slower devices without increasing TTFT.

## 13. LLumnix

### What is virtual usage in Llumnix and how does it let one simple load-balancing rule handle several different scheduling goals?

n Llumnix, virtual usage is an accounting value for a request’s expected memory demand, not just its current physical memory use.

By inflating that value for queued or high-priority requests, Llumnix can use one simple rule — send work to the freest instance — and still get load balancing, priority isolation, and de-fragmentation.

### Llumnixs live migration keeps downtime almost constant 20–30 ms even for 8k-token requests. Why is the downtime independent of sequence length?

Llumnix’s downtime is independent of sequence length because the KV cache is append‑only, so almost all previously computed KV blocks can be copied to the destination GPU in parallel with ongoing decoding.

It only needs to pause briefly to transfer the last small set of blocks generated during the copy window, so the pause is about one decode iteration (20–30 ms) regardless of how long the sequence is.

## 14. Pie

### What is the key idea of Pie?

The key idea of Pie is to replace the monolithic, opaque prefill–decode loop in today’s LLM serving systems with a programmable model where applications run inferlets that control inference through a fine‑grained API.

These inferlets get explicit control over KV cache management, the generation loop (prefill and decode), and integration of computation and IO, so each application can implement its own caching strategies, decoding logic, and agentic workflows instead of relying on fixed system‑wide policies.

### What are the three requirements that current inference services are missing?

R1: Application‑specific KV cache control.
R2: Customizable generation processes (per‑request control over the decoding loop).
R3: Integrated computation and IO inside the generation workflow.
