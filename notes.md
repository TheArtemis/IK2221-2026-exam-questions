# Notes

## 1 CHEETAH

To guarantee PCC inside load balancer have the state information inside the cookie instead of the server.

Pros are that it's easy to scale, as multiple lb instances do not need to have a shared state. It's fault tolerant, as if the lb goes down the state is not lost as all the required information for routing is inside the cookie. It's also easier to handle in the stateless way as the system avoids updating, expiring and syncing the state information. And it also has a faster lookup time as the lb can directly read the cookie information.

In the stateful version it's better than cuckoo hashing as it avoids cascades replacements.
In the stateful version we simply store the cookie as the key and the server as the assigned index.

### Cuckoo Hashing

You have two hash functions. If you put a key in position X, and there is already a key there, that key is evicted to a position Y (Computed using another hash function). So every key has two possible positions. The first one and the one it gets when being evicted.

## 2 APLOMB

They interview 57 companies on how they use middle boxes in their infrastructure and they found out that it's almost as much as the number of routers. This creates a lot of complexity in managing them, so they propose a solution to put everything in cloud, with a trade off of increasing latency by 3% on average they reduce the amount of physical middle boxes by 80%.

For the routing algorithm they ad 3 options, the first one is to make a request to the cloud, have a lot of back and forth from the cloud middle-boxes from the gateway and then send an answer to the client. The second option was to have multiple-clouds and only talk through them. And the third and last one (and the one they adopted) was to have a DNS style approach, where the cloud dns is resolved and then the traffic goes through that. So this is the most flexible solution without having the bouncing from the first solution.

## 3 CLICK

Click is a a good replacement for physical routers as it offers a way more modular configuration experience. while physical routers only have static functionalities that can only be turned on and off, click offers the possibility to combine multiple elements in a graph configuration flow, that offers push and pull streams combined by queues. it also offers comparable performances to physical routers while having the extra flexibility.

The push pull stream works like this, there is a upstream FromDevice that pushes the packets into the NIC, they are processed and then added to a Queue, where the downstream ToDevice pulls them into the NIC output. This is necessary as without a queue we would either have a situation where packets are pulled into the nic's output without being ready or for the other way around, with a over-saturated input stream we would have to drop the incoming packets without being able to put them somewhere to wait to be processed.

it's important to place the queue next to the nic's output as doing otherwise might add delays. As it's best to enqueue for it to be scheduled after it is processed.

The context of line rate is try to process packets as fast as the network serves them. For 100GBps it means we have a packet every 6 nano seconds. Accessing DRAM takes 60 nano seconds and accessing the last level of cache takes 20. So this is why we must reduce the amount of virtual function calls to the minimum. Click reduces that by having one vf call per graph's edge.

If we use interrupts to check for packets arrival we incur in the fact that context switches take time and that it might disrupt the cache and the amount of cache hits we get. Hence at very high throughput rates like today it's better to delegate one core to polling the NIC even under low demanding situations. We do not want to cause cache disruptions!

## 4 METRON

Three positives about metron. Metron uses programmable network switches and NICs to classify packets that will have the same NF applied (hence processed on the same core) to a specific class.

This packets are then tagged only once so that they do not need to be classified by each network function. So this classification tries to happen at the first OpenFLow switch.

Then, since the NIC holds a map of which packets are assigned to which core, on the hardware level we can assign a certain class of packet to the same core. This will avoid the context switch between different cores, that loses time to access the DRAM (60 nano seconds) and the access to the last cache level (20 nano seconds) that in contexts where we are trying to process packets at serving time in a 100 GBps network, loses a lot of CPU cycles.

Metron benefits decreases as the amount of stateful processing increases.
There are two experiments that underline this. The first one was a 40GBPs network with a firewall and DPI setup. Since the the firewall is a big stateless map most of the classification can be delegated to hardware tagging while for the 100GBPs router + NAPT + LB this is not possible as the NAPT and the LB are heavy stateful processes that must be done via software (for example if the LB has to maintain PCC) and with this last setup there was not a big performance increase

But what happens when a single core is overloaded? In some cases Metron is able to split the classes of packets into subclasses that get directed onto another core to mitigate the core overloading. When the traffic spike goes down then Metro merges back the two tables and the tagging points back to a single core.

However this cannot always be done as for example HTTP or other categories of packets that require stronger consistencies cannot be split onto two cores like this.

## 5 SPLIT RPC

There is too much of a overhead with current RPC structure, so we split the RPC in data and control plane.

pNIC (Peer NIC) vs sNIC (Smart NIC). Both the NICs are in charge of moving packets from input to control / data plane, but while the pNIC does the computations on the host's CPU, the sNIC can do them on a dedicated ARM core.

With smaller models, since the inference on the GPU is not super heavy the the sNIC ARM core that processes the split can become a bottleneck. When on the other hand we use heavier models the overhead from the ARM core becomes less noticeable as the GPU inference becomes the real bottleneck.

For performance evaluation is important to keep in mind the open loop model. With a closed loop model, so let's say 32 concurrent clients, since every client awaits for the server's answer to send the next request, we are not truly overloading the server's capabilities as the server slows down also the amount of requests slow down. So it's important in these kind of settings to keep a open loop testing framework. Where every X seconds we send a request to the server independently if we have received an answer or not. So that we can truly measure the tail (that in this case is thicker) of the p99 latency.

To wrap up. SplitRPC enforces on every packet a header that it's directed to the CPU and a large data body that it's directed to the GPU. Using DMA (Direct memory access) it's able to send directly to the CPU and the GPU their respective parts, keeping data and control separated where each one moves best.

The RPC tax consists in the fact that without the split we would need to send into the CPU also the bulky data body.

## 6 RDMA (Remote Direct Memory Access) RedN

RedN it's important as it provides a Turing complete instruction set to bypass the CPU and provide direct memory access to do operations.
The CPU can be overloaded and it can become the bottleneck of operations. For example if in a situation where the server is busy. If the client sends a request to access the memcached, instead of needing the CPU to lookup the table, the NIC can access it with RDMA instruction set.

But this is better only on overloaded servers, when the CPU is busy and cannot perform operations immediately. The bottleneck is the PCIe to memory speed. While to access the DRAM the CPU takes only 60 nanoseconds, for the same operation via NIC it takes 2 microseconds. So if we could only do operation via RDMA we would be bound to 1 / 2 \* 10 ^ -6 operations per second. Hence half a million instead of the order of billion of operations per second.

RDMA PRovides a certain set of operations like CAS, WAIT, ENABLE. That can be used to directly alter the verb execution flow. In this way, also for bidirectional communication between CPU and RDMA, after the initial setup phase the CPU can even be turned off

## 7 Paged Attention and vLLM

In normal settings KV Cache needs large spaces of memory to allocate the K V tensors for many tokens and many layers. So from start to finish of the request we end up with a lot of small empty segments of memory that are not big enough to fit other kv cache tensors. These do add up and hog precious VRAM.

vLLM enables a better management of the kv cache. Instead of storing it into a large contiguous region of memory, it pages it to avoid fragmentation and makes memory allocation more flexible.

It make's trivial the use of Beam Search and situations where the first prompt is reused.

## 8 Sarathi

The main motivation for Sarathi is that Prefill is way more GPU heavy than Decode and most inference serving systems are only optimizing for one of them. The idea is to batch the prefills into equal size chunks and schedule at most of of them with many decoding phases at the same time.

By doing this users do not incur in the annoying situation where a new heavy gpu intensive prefill phase starves the ongoing decoding processes. The end user often sees it as lag or freeze in the token generation that is streamed to the frontend.

Another example is, if we have 2 GPU's, the stronger one should run the prefill, while the shittier one should run the decode.

Types of parallelism:

1. Data Parallelism - when you can fit the whole model in one GPU but you want to increase throughput. Gotta be careful in the batch size tho because it can get very large if we keep it the same while adding GPU's and it can hurt training. To keep the same global batch size we should divide it by the number of GPUs to assign the local batch size.
2. Transformer parallelism - You cannot fit the whole model so you spread it in multiple GPUs
3. Pipeline parallelism - The full model cannot fit in the GPU but the biggest layer can, so you split the X layers between the N gpus
4. Expert parallelism - When you have a mixture of experts.

## 9 JANUS

Mixture of experts parallelism works like this. Different experts are hosted on different GPUs. Every processed token is first dispatched to all the experts (All to All communication) and after that all the GPUs (all the experts) sends back the activations to the home GPU.

This is a expert centric approach that becomes costly for large setups as there is a lot ot tokens and activations that need to be sent around and the network becomes the bottleneck as the expert are fixed on their own GPUs.

What Janus proposes is a data centric approach (that can be switched on and off between the expert centric approach) where instead of sending the tokens, we load the weights of the expert we need in the GPU.

To archive this Janus uses a credit based buffer. Whenever it needs to fetch an expert it spends a credit, while it's downloading the expert it does not remain idle but it computes on a previously fetched expert. Once the expert has been downloaded it gains the credit back.

The key is that we limit concurrency using credits while always keeping the GPU busy with a computation.

One optimization done by Janus is the Inter and Intra node cache. The setup is multiple nodes that host multiple GPUs. When an expert is fetched via RDMA, it is cached inside a shared memory inside the node, so that when another GPU needs it can fetch it directly via NVLink instead of going trough the network again.

### Speculative Decoding

Speculative decoding it's an inference technique that it's used to increase the TPOT. We first run a small model to quickly produce several next tokens and then we use a bigger model to reject them or validate them.

Janus is suitable for training, not for inference. This is because in the beginning of the computation Janus computes a ration R that defines if it's more efficient to transfer the tokens or transfer the weights. The parameters used to compute this are all constants and do not change between iterations.

Since decoding it's an auto regressive loop what happens is that the input grows by one for each iteration, altering the Ratio. Even recomputing the ratio

## 10 Mega Scale Infer

For the decoding phase in the Transformer we have two blocks. One attention layer and a FFN (Feed Forward Network layer). In the attention layer we look at the kv cache and we compute the attention for the token. After that we have a hidden state for this token and we select the top K to do a big ass matrix multiplication.

This happens in batches, we have many requests at the same time and we run the decode on all of them in parallel.

The idea behind Mega Scale Infer is that the attention and the ffn have very different needs in term of requests. So we implement Pipeline Parallelism! On one GPU that is optimized for memory bandwidth (For the kv cache) we put the attention layer (N Cluster). And on the other one (the M Cluster) is used for the MoE FFN, that's optimized for FLOPS and GEMM.

Another key thing is to implement ping pong micro batching, so that when a micro batch is on the N Cluster the other one is on the M Cluster and the GPU is always kept busy!

H20s GPUs are optimized for memory capacity (how much can we fit) and memory bandwidth (how fast can we read) so they are better for the attention layers, while LP20s have a higher TFLOPs per unit so they are better for GEMM (General Matrix Multiplication)

## 11 Spec Infer

Spec Infer builds up on speculative decoding. The classical approach is that there is a SSM (Small speculative model) that generates a sequence of token and then there is the actual model that checks that the predictions are the ones that he would actually produce. The problem here is that if the sequence is of 5 tokens and the LLM evaluates that it is right till token 2, then the rest of the tokens are eliminated and we need to run another iteration.

So what Spec Infer proposes is to have token being speculated in a tree like structure by the SSM and then through a special attention mask, this branched predictions can be evaluated all at the same time against the LLM ones.

Compared to normal approaches the TTFT is increased as we have to evaluate against more candidates but then the token throughput increases as it's more efficient to process branches in parallel.

It's important that both the SSM and the LLM are highly aligned, we cannot use a SSM that's trained on producing novels as the token prediction for a novel is very different from a token prediction for code, hence we would have a very bad rate of matched predictions.

The topology aware mask enables to run the attention over a flattened list of token in one pass but enforcing that each token only attends to it's ancestors from the tree and not the flattened version of the tree, preserving the parent-children relation instead of the simple left to right relation.

## 12 Cacheblend

### RAG

The user inputs a query, then RAG fetches from a vector database the chunks that are most relevant for that user query. The selected chunks are then fed to the LLM with the user query.

Since the same chunks are often reused, cache blend first runs a prefill and computes the kvcache for all the chunks individually (that are now missing the cross attention between them) and then when it does the RAG, for the selected chunks, layer by layer it recomputes the kvcache for a subset of those tokens so that now the chunks have cross attention between them.

Since the same chunks are oftern reused, cache blend first runs prefill on all the chunks singularly and after the RAG it first loads the computed kvcaches and concatenates them (so that now it is an approximation of the kvcache of the chunks + query normal prefill) and then for a subset of the tokens, layer by layer it selects the most important tokens and then it recomputes the Q / K / V and then runs the attention on all the tokens. (In this way the cross attention information is computed).

This solves the problem of just concatenating precomputed kvcaches of the chunks as they do not retain the cross attention information between them.

The selected tokens by cache blend are HKVD tokens, that basically are the tokens that have the biggest deviation from the kv cache obtained from a full prefill. These tokens are estimated with a gradual filtering starting from the first layers where the deviations are the most evident and narrowing it down layer by layer as the high deviation tokens are similar between neighboring layers. (Of course we cannot run the full prefill at runtime!)

The saved kvcache can live under different storages. How do we decide which one? Cache blend is smart, first it does some operation pipelining, when we are computing the HKVD kvcache for the tokens in layer l, we are also loading in VRAM the kvcache for layer l+1. So the latency for the TTFT is not bound to the sum of the computation and the loading but only to the upper one of the two. So what cache blend does is that it selects the storage solution that keeps loading time < computing time. (Usually is the ssd).

## 13 LLumnix

LLumnix is a load balancer for LLM requests that cleverly enables the LIVE migration of a request from a GPU to another. The LB keeps track of the virtual memory usage and it tries to split it evenly across all GPUs. When a new request comes it also carries the request for the virtual memory needed for that + a headroom for priority and queued requests.

When it needs to do a live migration for a request. It starts by copying in parallel the kv cache from one GPU to the other. Since the kvcache is append only the amount of time does not depend on the sequence length of the request. After the big chunks of the kv cache have been copied, the only downtime is to copy the last part of the kvcache (and that takes one decoding cycle) so that the decoding can resume on the other GPU.

To keep the priority simple each request defines a virtual memory usage that can be used by the scheduler to properly spread the workload evenly around the GPUs + an extra headroom that can be used to increase the requested virtual memory usage so that since that request virtually requires a lot of memory, subsequent requests won't end up in that same GPU to not overload it (even tho it would actually be able to handle it but we want to keep it free for faster decoding).

The assumption that is done here tho is that transferring the kvcache is faster than recomputing it. In situations where the network is a bottleneck it might be faster to recompute it directly.