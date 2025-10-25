# Scaling Elastic SIEM: Rebuilding Our SOC with LXCs and Revamped Storage
2025-07-16

12 min read

by Charlotte Croce

When I took on the responsibility of migrating the [Leahy Center](https://www.champlain.edu/academics/career-focused-academics/leahy-center/) centralized SIEM from VMware to ProxmoxVE ([thanks, Broadcom](https://community.broadcom.com/blogs/julia-klaus/2024/08/15/vmware-it-academy-and-of-life)), I initially thought it would be a straightforward process. After all, there were already established scripts available for hypervisor conversion. However, what seemed like a simple migration quickly revealed itself to be far more complex, uncovering several roadblocks and architectural inefficiencies that made this an ideal opportunity for a complete infrastructure redesign.

We began planning in March 2025. Working with Samual Richardson, our former SOC Lead and Champlain College Class of 2025 graduate, we found the most glaring problems in our existing setup and brainstormed solutions. After Sam's graduation in May, I took on the technical execution independently throughout the next two months.

The first challenge we encountered was the inefficiency of a simple lift-and-shift, as our Elastic Stack processes billions of logs from hundreds of endpoints. A direct migration would have required converting multi-terabyte datasets. This would have created an unacceptable amount of downtime and an extreme bandwidth utilization spike. Our SIEM provides security monitoring for local nonprofits who depend on our services, meaning any extended downtime would violate our [SLA](https://en.wikipedia.org/wiki/Service-level_agreement) and impact real organizations.

The second issue was in our existing infrastructure. Built in 2018 (when I was in 7th grade), the system remained functionally stable but was built with suboptimal design choices that became more apparent as time passed.

This migration presented an opportunity to address these issues with a complete rebuild. We could implement a storage architecture specifically designed for our particular I/O patterns. Additionally, we could greatly reduce hypervisor overhead through LXC containerization.

## Migration Strategy

To ensure continuous security monitoring during the transition, we executed a two-phase migration:

**Phase 1: VMware to VMware** - We first migrated the existing cluster to an unused vSphere server. Since both environments used the same hypervisor, we could leverage [vMotion](https://www.vmware.com/products/cloud-infrastructure/vsphere/vmotion) and shared storage, making this relatively straightforward with minimal risk. Phase 1 took less than 6 hours in total.

**Phase 2: Build from scratch on ProxmoxVE** - The cross-platform migration proved much more complex due to the aforementioned roadblocks. Although we rebuilt from zero, we did use prior configuration files and our internal documentation to build with similar design patterns.

## Storage Overhaul: Eliminating Redundancy Layers

Here is the most pressing issue we found: our legacy architecture utilized a single [RAID6](https://en.wikipedia.org/wiki/Standard_RAID_levels#RAID_6) array across all nodes. While RAID6 provides fault tolerance through dual parity blocks, this created an inefficient redundancy stack when combined with Elasticsearch's native replication mechanisms.

### Redundancy Analysis

Elasticsearch implements distributed redundancy through its [primary/replica shard architecture](https://www.elastic.co/docs/deploy-manage/distributed-architecture/shard-allocation-relocation-recovery). In our case, each primary shard maintained one replica distributed across cluster nodes, providing both fault tolerance and query load distribution. Combined with RAID6's dual parity protection, we effectively maintained the following storage replication structure:

| Layer   | Description  |
|---------|--------------|
| **Data**    | Original shard data                        |
| **Application** | Elasticsearch replica shards (1:1 ratio) |
| **Storage** | RAID6 parity blocks (2 per stripe)         |

Our 6-drive RAID6 array combined with Elasticsearch's 1:1 replication consumed 3x raw storage capacity (6 drives / 4 usable drives × 2 replicas), with functional efficiency dropping even further due to filesystem overhead, fragmentation, and operational space requirements.

| RAID6 Array Size | RAID6 Overhead | + ES Replication | Total Overhead | Efficiency |
|------------------|----------------|------------------|----------------|------------|
| 4 drives         | 2x             | 2x               | 4x             | 25%        |
| 5 drives         | 1.67x          | 2x               | 3.33x          | 30%        |
| **6 drives (ours)** | **1.5x**   | **2x**           | **3x**         | **33%**    |
| 8 drives (proposed) | 1.33x          | 2x               | 2.67x          | 37%        |
| 10 drives        | 1.25x          | 2x               | 2.5x           | 40%        |

This [write amplification](https://en.wikipedia.org/wiki/Write_amplification) became even more problematic during Elasticsearch's routine write-heavy maintenance operations. When the cluster rebalanced shards, performed index rollovers, or migrated data between hot and cold tiers, each data movement triggered additional parity recalculations. Moving a shard from one node to another required reading the original data, writing it to the new location, and recalculating parity bits for both the source and destination stripes. This made routine cluster operations much more I/O expensive.

We initially considered simply expanding our RAID6 array by adding two more drives, which would have improved efficiency from 33% to 37% and given us more storage. However, this small improvement still fell short of the 50% efficiency achievable with direct-attached storage, and would have perpetuated the write amplification issues inherent to RAID6. The marginal storage and efficiency gain didn't justify maintaining the I/O penalties.

## Solution: A Direct-Attached Storage Architecture

We redesigned the storage topology around dedicated direct-attached SSDs per node, eliminating RAID redundancy in favor of Elasticsearch's native replication. For the new architecture, we expanded to 10 total drives with dedicated allocation for Elasticsearch nodes:

| Service | Storage | Prior usage |
|---------|---------|-------------|
|ProxmoxVE Host|2x NVMe drives (RAID1)|vCenter OS|
|Elasticsearch Hot Tier|4x dedicated SSDs (no RAID)|RAID6 array|
|Elasticsearch Cold Tier|2x dedicated SSDs (no RAID)|RAID6 array|
|Auxiliary Services|2x SSDs (RAID1)|NEW|

We configured Elasticsearch nodes with [XFS](https://en.wikipedia.org/wiki/XFS) filesystems for optimal performance, while shared drives and the hypervisor OS utilize [LVM](https://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux)) block storage for flexibility in volume management.

This design reduces storage overhead to approximately 2x raw capacity, while improving I/O performance through eliminated parity calculations.

| Storage Configuration | Replication | Total Overhead | Efficiency | Use Case |
|----------------------|----------------|----------------|------------|----------|
| Direct-attached, 0 replicas | None | 1x | 100% | Dev/testing only |
| **Direct-attached, 1 replica (ours)** | **1:1** | **2x** | **50%** | **Production standard** |
| Direct-attached, 2 replicas | 1:2 | 3x | 33% | High availability |

In our legacy system, storing 1TB of log data required 3TB of raw disk space (1TB × 1.5 RAID6 overhead × 2 ES replicas), compared to 2TB in the new direct-attached approach (1TB × 2 ES replicas).

If a non-RAID SSD fails, the node and its configuration will be lost. However no log data will be lost. Due to the ease of reinstating an Elasticsearch node via CT Template (more on LXCs below), we concluded that simply rebuilding was a viable solution in the event of a drive failure. We will lose a machine, but not any of the logs.

### Hot/Cold Tier Strategy

We retained our existing tiered data strategy: 4 nodes for active log ingestion (0–14 days), and 2 nodes for historical data (15–90 days). This matches our usage patterns: real-time alerts and analyst queries focus on the most recent logs, while long-term threat hunting tolerates higher latency. The 4:2 hot/cold layout reflects this query distribution and maximizes I/O performance on the hot tier.

## Linux Containers: Eliminating Hypervisor Overhead

Virtual Machines are the tried and tested way for service isolation, but for resource intensive applications, they have unnecessary overhead, including:

| Overhead Type | Description |
|---------------|-------------|
| Memory    | Guest OS memory allocation, hypervisor memory management |
| CPU       | Hardware emulation, context switching between guest/host |
| Storage   | Virtual disk layers, snapshot management |
| Network   | Virtual network stack, bridge processing |

For our Elasticsearch cluster, these overheads were problematic given the application's intensive memory and I/O requirements. We were spending a lot of system resources just to run the environment, not the workload itself.

### LXC Implementation Benefits

[LXCs (Linux Containers)](https://pve.proxmox.com/wiki/Linux_Container) share the host kernel while providing process-level isolation through cgroups and namespaces, eliminating the guest OS layer entirely while maintaining security boundaries. This lightweight approach offered an alternative to traditional virtualization for our Elasticsearch deployment.

LXC containers typically consume 2–5% overhead compared to 15–20% for full VMs. For memory-intensive applications like Elasticsearch, this difference is very noticeable given the application's requirement for large heap allocations and buffer caches. Additionally, containers achieve near-native I/O performance since they access storage directly through the host kernel, eliminating the virtual disk layers that introduce latency in traditional VM deployments.

Without the overhead of individual guest operating systems, we gained the freedom to segment our cluster as granularly as needed. Our cluster size effectively doubled from the previous VM-based deployment, and when combined with auxiliary services such as Kibana, Fleet Server, and various proxies, our total container count reached 30 instances. Despite this dramatic increase in service segmentation, we actually consume less CPU than our previous infrastructure of approximately 12 VMs.

## Client Migration

We use [Elastic Agents](https://www.elastic.co/docs/reference/fleet) for endpoint communication, which has a rather simple installation and enrollment process. I wrote two scripts (one in Bash, one in PowerShell [well... AI wrote my PowerShell]) to uninstall the legacy agents, trust our CA, and install a new agent enrolled into the new Elastic Stack.

I gave the scripts to our IT Team, which manages client infrastructure, and together we updated client firewalls and remotely migrated endpoints over.

## Results

These combined architectural changes gave us significant performance improvements:

| Component | Legacy Architecture | New Architecture | Benefit |
|-----------|-------------------|------------------|---------|
| **Storage** | 3x overhead, 30% efficiency | 2x overhead, 50% efficiency | 67% more usable space |
| **Compute** | VMware VMs (15–20% overhead) | ProxmoxVE LXC (2–5% overhead) | 75–83% less virtualization overhead |
| | 15% ES CPU usage | 5% ES CPU usage | 67% CPU reduction |
| **Architecture** | ~12 VMs | ~30 Containers | ~150% more service segmentation |

## Conclusion

By aligning storage architecture with application-level redundancy mechanisms and eliminating hypervisor overhead through containerization, we achieved a more resilient and performant SIEM infrastructure.

With the Elastic Stack in a stable state, our next focus shifts towards an internal project repurposing the legacy server. But first, I'm taking a 5-month break to [study in Dublin](https://international.champlain.edu/section/dublin/). Let's hope it holds!

---

*The Leahy Center is a laboratory dedicated to development, both professional and technological. This insight into our operations has been approved for publication by Ryan Gillen, Leahy Center Manager.*
