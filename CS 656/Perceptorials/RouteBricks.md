# RouteBricks: Exploiting Parallelism To Scale Software Routers

## First things first

#### What is the paper about?
- **Scaling software routers**
- The primary objective of this paper is to explore the problem of scalability in software routers in the given context of high-speed parallel processing available in servers.
- The paper takes the approach of parallelizing workloads across servers and across cores within a server
- Router workloads are suited for this approach
###### What are software routers?
Simply put, routing functionalities implemented using software on general-purpose hardware instead of using specialized hardware.
For a more in-depth discussion on software routers see [Implementation and Performance Comparison of High-Capacity Software Routers](https://www.sciencedirect.com/science/article/pii/S1389128620312238)

#### What is the output?
- A router architecture that supports 35Gbps throughput
- This architecture can easily be scaled by adding servers
- Fully programmable - built entirely from off-the-shelf, general-purpose hardware

## Introduction
- The focus of the industry has been to develop hardware with very specific functionalities and with the primary attention to the performance
- Meanwhile, networks have taken up much more sophisticated roles (e.g., IDS, firewalls, data-loss protection)
- But in the absence of software extensibility, the approach taken has been that of middleboxes - high-performance specialized hardware, but the cost of implementation, maintenance and management is high
- Interest has been growing for extensible packet-processing router.
- Problem:
	- The implementation of desired functionality necessarily means modification to the router's high-speed data plane per-packet processing
	- A special purpose hardware is designed to be "closed" to extension by nature (Why is that? Because functionality is usually built into the h/w, and software extensibility in such cases)
	- It is pretty much an "either-or" relation between high performance and extensibility
- Two approaches to solve this problem:
	- Use specialized devices and make them more programmable - only possible to a small extent and requires specialized, low-level knowledge of the hardware
	- Implement software routers and optimize for high performance
		- Easier to innovate - larger community of programmers
		- Possible to extend the data plane and the control plane functionality
		- Functionality can be modified through software-only upgrades
		- Router developers will be relieved of hardware design burden
- Using commodity hardware has other advantages too
	- Economic benefits of large-volume manufacturing, supply and support
	- Benefits from advancements in semi-conductor tech, advanced power management features
- Challenge: Scaling this approach to high-speed networks

This paper explores the second approach - scaling software router performance to match specialized devices

> Performance comparison as of the time of this study -
> **Carrier grade routers** - 10Gbps to 92Tbps
> **Software routers** - 1-5Gbps

Solution devised in this paper -
> **RouteBricks**
> A router architecture that uses both of the following strategies:
> 1 - Parallelize router functionality across servers
> 2 - Parallelize router functionality across cores within one server

Three packet-processing applications are studied -
1. Packet forwarding
2. Traditional IP routing
3. IPsec encryption

## Design Principles
> **Background**
> A router has two functionalities -
> 1. Packet processing - done by linecards connected to input ports
> 2. Packet forwarding - done by the switching fabric
> Each linecard processes packets at line rate R and the switching fabric should do it at the rate NR

Existing software routers follow "single server as router" approach.
NR can be fairly high for even mid-range routers which is impossible to match up to for a single server
#### Design Principle 1
_The router functionality should be parallelized across multiple servers_
- This way the workload can be distributed and workload requirements per server are met.
- Each server acts as a router linecard
- This architecture is incrementally extensible in terms of the number of ports

#### Design Principle 2
_Router functionality should be parallelized across multiple processing paths within each server_
- This is because, to take on the responsibility of its individual workload of a baseline rate of 10Gbps, a server would have to scale to at least 20Gbps ([shown later](RouteBricks#^f56685))

## The Design

**Traditional router architecture**

![Traditional Router Architecture](Traditional-router-architecture.png)

**Cluster Router Architecture**

![Cluster Router Architecture](Cluster-router-architecture.png)

- A router with $N$ ports
- Each server acts as a port - full-duplex line rate $R$ bps
- Packet processing rate per server proportional to $R$
- Functionality is parallelized across multiple routers
- Decentralized solution - load-balanced interconnects so that each node can independently make forwarding decisions
- No server needs to operate at a rate higher than $cR$ where $c$ is independent of $N$ and ranges from 2 to 3
So using the above rules, we design a cluster with $N$ ports and line rate $R$ bps, using servers whose performance need only scale with $cR$, independent of $N$. ^f56685

**Design Tradeoff** - RouteBricks cannot offer strict performance guarantees like traditional routers

## Parallelizing Across Servers
Focus - Designing a distributed switching solution that is compatible with the use of commodity servers

Two problems required to be solved here:
1. Providing a physical path connecting input and output ports
2. Determining which input gets to relay a packet to which output port

Three guarantees:
1. 100% throughput (output ports can operate at $R$ bps if required)
2. Fairness (each input port gets a fair share of the capacity of any output port)
3. Avoid reordering packets (which is an undesirable scenario under TCP)

The choice of using commodity hardware presents the following three constraints:
1. _Limited internal link rates_: Internal links cannot run higher than external line rate $R$
	- This constraint is more of a design constraint to limit the cost. E.g., 100Gbps internal links could be used for 10Gbps external links, but the cost might not be feasible
2. _Limited per-node processing rate_: A single server can run at a rate no higher than $cR$
	- This is due to the performance limitations of servers at the time
3. _Limited per-node fanout_: Each server can only have a physical connection to a few other servers
	- This limitation exists because the servers can have only a small number of NICs with each NIC limited to 2-8 ports

>The topology design has to solve the two problems while providing the three guarantees and keeping the three constraints in mind.
>The attention would be on minimizing the cost, which, in this case is the interconnect capacity (number of servers $×$ link rate).
>The number of servers will have to be minimized as that is the dominant cost.

#### Routing Algorithms (Forwarding)
###### Topology Design options
1. _Single-path_: The traffic from input to output port follows a single path
	1. _Static single-path_: The path remains constant
	2. _Adaptive single-path_: Centralized scheduler recomputes path based on the current demand
2. _Load-balanced_: Traffic is spread between multiple paths

Static single-path cannot be used because the internal link will handle aggregated traffic, which means that it will need to operate at a rate higher than the external links (violation of constraint 1).
Adaptive single-path cannot be used because it requires a centralized component to analyze the router state and should operate at $NR$ (violation of constraint 2).

We use a load-balancing algorithm - Valiant Load Balancing (VLB) and adapt it to our needs

###### Valiant Load Balancing
$N$ nodes are arranged in full mesh as shown below:

![VLB mesh](VLB-mesh.png)

Forwarding from source node $S$ to destination node $D$ is performed in two phases:
- _Phase 1_: Node $S$ chooses at random an intermediate node to which to forward this packet
- _Phase 2_: The intermediate node forwards this packet to output node $D$

As a result of phase 1:
- The traffic received at each node is uniform. This enables a node to autonomously make a forwarding decision (without having a concern about the congestion at the output link). This enables VLB to achieve **100% throughput and fairness** (first and second guarantees).
- Since the traffic is uniform **a speedup is not required** (our first constraint is met). Each internal link must have the capacity $\frac{2R}{N}$ where "2" accounts for the double traversal, and dividing by $N$ reflects the distribution of traffic across all nodes
Since each node is full-duplex, it has to handle $2R$ traffic rate + $R$ because of additional traversal. This means instead of $2R$, each server has to process traffic at rate $3R$, which is a 50% processing overhead.
###### Direct VLB
- This is an optimization over VLB to reduce the overhead caused by the additional traversal.
- It is often the case that the router traffic is already uniform. Redirection of packets to intermediate nodes in this case doesn't add any benefits but does add to the overhead.
- Direct VLB approach says that each source node $S$ forwards up to $\frac{R}{N}$ of the incoming traffic directly to $D$. Rest of the traffic is redirected over intermediate nodes.
- This way, when the incoming traffic is close to uniform, each server processes a maximum rate of $2R$ (no VLB overhead)

> Our second constraint was that each server should process traffic at rate $cR$. For Direct VLB, c can range from 2 (incoming traffic is uniform) to 3 (non-uniform incoming traffic).

###### Issues that still remain
1. Reordering is possible
2. Server fanout  constraint cannot be met for $N$ > server fanout

###### Solution
1. Assign to each server as many ports as it can handle. If the fanout allows, then connect them directly into a mesh topology.
2. If the number of servers cannot be accommodated by the fanout, then use a k-ary n-fly topology, where k is the per-server fanout and $n = log_k N$.

Example of a 2-ary 3-fly network topology:

![Generalized butterfly](generalized-butterfly.png)

#### Cost Analysis
3 Configurations:
1. One server $\Rightarrow$ One router port and 5 NICs (available)
2. One server $\Rightarrow$ One router port and 20 NICs (available as custom motherboard configurations)
3. One server $\Rightarrow$ Two router port and 20 NICs (upcoming servers)

![Cost Analysis](cost-analysis.png)

- For config 1, 32 servers can be connected in full-mesh
- For config 2, it is 128 servers
- For config 3, 2048 servers
- Conclusion: this cluster can scale to hundreds or thousands of ports

## Parallelizing Within Servers

#### Server Architecture

![Server architecture](server-architecture.png)

- Intel Nehalem server
- 2 sockets, four 2.8GHz cores per socket
- Shared 8MB L3 cache per socket
- One integrated memory controller per socket $\Rightarrow$ NUMA architecture
- Sockets are connected to I/O hub and I/O hub to PCIe buses
- Two NICs each holding two 10Gbps ports
#### Software
- Linux 2.6.19 with Click in polling mode - i.e., the CPUs poll for incoming packets rather than being interrupted
- 10G Ethernet driver
- Proprietary performance tool similar to Intel VTune
#### Traffic generation
- 2 NICs $\times$ 2 ports $\times$ 10Gbps = 40Gbps theoretical rate
- But use of x8 PCIe 1.1 slot limits the maximum payload data rate to 12.8Gbps per NIC = 24.6Gbps
- Target hardware expected to offer 4-8 NIC slots
#### Exploiting Parallelism

> **Multi-core is not enough**
> - The same tests were tried on Intel Xeon servers
> 	- Same two socket and total 8-core CPU
> 	- Shared front-side bus instead of a dedicated bus per socket
> - It's performance fell short of Nehalem processors for small and medium sized packets
> - The bottleneck was found to be the shared bus connecting the CPUs to the memory subsystem
> - Packet processing workloads place a heavy load on memory and I/O, and the shared bus didn't provide enough bandwidth
> - Nehalem's architecture provides 2-3x performance improvement over the tested Xeon server

2 rules to be followed to achieve desired performance -
_Rule 1_: Each network queue can only be accessed by a single core
- A core needs to perform a lock on the input queue before reading data from the queue
- This is forbiddingly expensive

_Rule 2_: Each packet will be handled by a single core
- The other approach would be pipelining, which requires synchronization between cores and slows down the performance by as much as 29%, with cache misses 64%

Two common cases for rule violation -
1. Number of cores > number of NICs _and_ a single core can't cope with the processing demands $\Rightarrow$ In this case, one core will poll in a packet, split it between multiple cores
2. Overlapping $\Rightarrow$ Two packets that have different input ports but the same output port

![Multi queue NIC](multi-queue-nic.png)

The problem is solved using Multi-queue NICs, NICs with multiple 1Gbps interfaces.

###### Poll-driven batching + NIC-driven batching
To reduce per-packet book-keeping overhead, the operation can be batched to have fewer number of larger transactions, complemented by poll-driven batching that receives packets in batches.
$k_p$ = number of packets per poll operation
$k_n$ = number of packets batched by NIC (modified NIC driver)

$k_p$ = 32 and $k_n$ = 16 yields the optimal results (shown below).

![Batching Performance](batching-performance.png)

###### NUMA-aware data placement
For router specific workloads, placement of data was not found to make any difference in packet processing. So, no special treatment is being done to keep packets processed by a core closer to that core's socket.

#### Summary
- Optimal results when CPU parallelism is accompanied by memory access parallelism and multi-queue NICs
- Test scenario -
	- 64 KB packets
	- Pre-determined input and output ports $\Rightarrow$ no header processing

![Test scenario results](intra-parallelism-results.png)

**Result**: 670% improvement from Nehalem single-queue, no-batching method, and 1100% increase from Xeon architecture

## Evaluation
1. Describe workloads
2. Black-box testing
3. Observation analysis

#### Workloads
We consider three applications:
1. _Minimal forwarding_: fixed input and output ports
	- Stresses memory and I/O
	- Minimal subset of any packet-processing action
	- Offers upper bound on achievable performance
2. _IP routing_: Full IP routing checksum calculation, header updates, longest-prefix-match lookup
	- References large data structures
3. _IPsec packet encryption_: Every packet is encrypted using AES-128 encryption
	- CPU-intensive

**Metrics** - maximum attainable loss-free forwarding rate in terms of bits per second (bps) or packets per second (pps)

#### Black-box System Performance
###### Minimal Forwarding
Lowest rate for 64B packets $\Rightarrow$ 9.7Gbps
Highest rate for Abilene trace packets $\Rightarrow$ 24.6Gbps
###### IP Routing
64B packets $\Rightarrow$ 6.35Gbps
Abilene packets $\Rightarrow$ 24.6Gbps
###### IPsec Encryption
64B packets $\Rightarrow$ 1.4Gbps
Abilene packets $\Rightarrow$ 4.45Gbps

![Black box performance](black-box-perf.png)

#### System Performance Analysis

The approach would be to understand the bottlenecks

Considering the following components:
1. CPUs
2. memory buses
3. Socket-I/O links
4. Inter-socket link
5. PCIe buses connecting the NICs to the I/O hub

For each component:
1. Estimate an upper bound on the per-packet load that each component can accommodate as a function of the input packet rate.
2. Measure the per-packet load on the component under increasing input packet rates for different workloads.
3. Compare actual loads to upper bounds to find the bottlenecks.

Upper bounds:

![Upper bounds](sytem-components-upper-bounds.png)

Observations:

![CPU load](cpu-load-analysis.png)

![Other components - load](other-components-load.png)

**Bottleneck** -
- For all applications CPU cycles/packet approaches the upper bound
- This indicates that CPU is the bottleneck in all three applications

Why?
- Cycles/packet remains constant under increasing load
- Increasing cycles/packet would indicate that the memory subsystem is struggling (higher access times)
At the time of the study, it was predicted that the number of cores will scale according to Moore's law, so the situation isn't undesirable.

**Small v/s large packets** -
- Load per byte was observed to be higher for smaller packets.
- The reason mentioned for this was smaller packets will have a higher book-keeping overhead

**Non-bottlenecks** -
- Per-packet load on all other components is lower than the upper bound
- Indicates that they are not performance limiters

**Expected Scaling** -
- For all three applications, per-packet load on system remains constant
- Extrapolating from observations, performance rates of 38.8, 19.9, and 5.8Gbps for minimal forwarding, IP routing, and IPsec, respectively, given 64B packets
- CPU will remain the bottleneck
- For Abilene trace packets, if the two NIC limitation is removed, it is possible to reach 80% of upper bound (70Gbps)

#### Summary
- Current servers achieve good performance given the realistic Abilene workloads
- Fare worse given the worst-case 64B-packet workloads
- CPU is the bottleneck, but next-gen expected to provide 4x improvements

## The RB4 Parallel Router

- 4 Nehalem servers
- Interconnected through full-mesh topology
- Direct-VLB routing
- Each server is assigned a 10Gbps external link

#### Implementation

Using the two design principles.
Focus on two details:
1. Reducing CPU load (minimizing packet processing)
	- Avoid having each CPU in the interconnect path processing the packet header
	- Find the destination port in the source port server and put its MAC address as the destination address.
	- NICs have a feature to take an incoming packet and put it into the output queue based on its MAC address
2. Avoiding packet reordering
	- Use the TCP or UDP flow concept
	- Same-flow packets are assigned to the same server
	- Same-flow packets during a 100 msec burst is assigned to one set of intermediate nodes

#### Performance

**Forwarding performance**
- RB4's routing performance is 12Gbps, i.e., each server supports 3Gbps external line rate.
- Expected performance - between 12.7 to 19.4 Gbps
- Slowdown reason - reordering avoidance overhead

**Reordering**
- 0.15% reordering when using reordering-avoidance
- 5.5% reordering without reordering-avoidance

**Latency**
- For 2-3 hops in RB4 - 47.6 - 66.4 $\micro S$
- Cisco 6500 Series router 26.3 $\micro S$

## Conclusion
- Analysis of scaling software routers to replace special-purpose hardware
- Proposed a parallelized router architecture
- Decent baseline performance to build on using future improvements to commodity hardware
- Close to achieving line rate of 10Gbps
- Emerging servers promise to close the gap to 10Gbps


## Questions:
1. With the innovations and programmability of SDNs, what is the relevance of RouteBricks?
2. If they take the same approach as NFVs, what is the difference?
3. What are the applications of programmability in the data plane? (again in the context that SDNs are available as an alternative)
4. Why is the need for scaling the servers with cR instead of just R?
5. Have there been studies on if we took the alternative approach of going with a specialized or programmable computer instead and then build on the programmability?
6. An approach taken by this architecture is to exploit multi-core parallelism in hardware that provides this service (which is almost all processors these days). Have there been studies done on leveraging GPUs for this, given the much higher core count?
7. What would be the implications of using ARM-based SoCs for this architecture, given there speed, power efficiency and cost-effectiveness?
