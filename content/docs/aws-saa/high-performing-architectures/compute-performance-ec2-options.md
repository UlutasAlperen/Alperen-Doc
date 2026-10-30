---
title: "compute-performance-ec2-options"
weight: 50
---

# Compute Performance: EC2 (Elastic Compute Cloud) Options

"Which instance type / placement / purchase option" questions test whether you know the **instance family alphabet** and the three placement group shapes. (Pricing is covered in [pricing-models](../../cost-optimized-architectures/pricing-models-ri-savings-plans-spot/).)

## Instance families

| Family | Optimized for | Mnemonic |
|---|---|---|
| **T** (t3, t4g) | Burstable general purpose (baseline CPU + credits) | "t" = tiny/test |
| **M** (m5, m6i) | Balanced general purpose | "m" = middle |
| **C** (c5, c7g) | Compute intensive (app servers, game servers, batch) | "c" = compute |
| **R** (r5, x2idn) / **X** / **Z** | Memory intensive (in-memory DB, caches, big data) | "r" = RAM |
| **P / G / VT / Inf** | GPU/ML training and inference | "p/g" = graphics |
| **Hpc / D** | High performance compute, high disk throughput | |
| **Mac** | macOS build farms | |

**Burstable (T family)** detail: instances earn **CPU credits** when idle and spend them under load; "unlimited" mode can pay for extra credits. "Variable low CPU baseline" → T family.

## Placement groups - the three shapes

| Type | What it gives | Limit / use |
|---|---|---|
| **Cluster** | Instances in one AZ (Availability Zone), same rack → **lowest latency, highest throughput** | HPC, tightly-coupled, low-latency networking |
| **Spread** | Instances on **distinct hardware** (max 7 per group per AZ) | Critical instances that must not fail together |
| **Partition** | Hardware partitions (up to 100s of instances) | Big data: HDFS, HBase, Cassandra, Kafka |

Exam keywords: "lowest possible network latency between nodes" → **cluster**; "critical instances must not share hardware" → **spread**; "HDFS / Kafka / topology-aware" → **partition**.

```mermaid
flowchart TB
    subgraph cluster[Cluster placement: one AZ, one rack]
        a[EC2] --- b[EC2] --- c[EC2]
    end
    subgraph spread[Spread placement: distinct hardware]
        d[EC2] .. e[EC2] .. f[EC2]
    end
```

## Networking and storage performance

- **ENA (Elastic Network Adapter)** (enhanced networking): 25 Gbps+ , required for modern types; **EFA (Elastic Fabric Adapter)** = OS-bypass for HPC/ML
- **Instance store** = raw NVMe on the host → highest IOPS (input/output operations per second), but **ephemeral** (data dies with the instance)
- **EBS (Elastic Block Store) gp3/io2** = network block storage; io2 for provisioned IOPS (databases); st/sc1 for cheap cold
- AMI (Amazon Machine Image) choice: pick the right region/architecture (see [creating-and-using-your-own-ami](../../../aws/ec2/creating-and-using-your-own-ami/))

## Exam check

**Q1: A tightly-coupled HPC simulation needs the lowest possible latency between 20 nodes. Which placement group?**

- **A)** Spread
- **B)** Cluster
- **C)** Partition
- **D)** Multi-AZ group

{{% details "Cevabı göster" %}}
**Answer: B.** Cluster placement = single rack, single AZ, minimum latency.
{{% /details %}}

**Q2: Three critical instances must run on separate physical hardware so one host failure can't take them all down. Which placement?**

- **A)** Cluster
- **B)** Partition
- **C)** Spread
- **D)** Auto Scaling

{{% details "Cevabı göster" %}}
**Answer: C.** Spread = distinct hardware (7 per AZ limit).
{{% /details %}}

## Ozet

- T = burstable (CPU kredisi), M = dengeli, C = hesaplama, R/X = bellek, P/G = GPU
- Cluster placement = tek AZ, en dusuk gecikme (HPC)
- Spread = ayri fiziksel donanim, maksimum 7/AZ (kritik instance)
- Partition = donanim bolmeleri (HDFS, Kafka, Cassandra)
- Instance store = en hizli disk ama gecici; EBS = kalici ag diski
- ENA = hizli ag, EFA = HPC/ML icin bypass
