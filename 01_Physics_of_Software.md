## Physics of Software

It's all about **latency, bandwidth and cost**. Without understanding the hardware, abstractions will eventually leak and the system will fail.

### Latency Hierarchy

Most important timers:
Assuming a 3Ghz core, scaled up as 1 ns -> 1 sec
| Action | Time |
| ------ | ---- |
| CPU register | 0.3 sec |
| L1 cache | 1 sec |
| Mutex lock | 17 sec |
| RAM lookup | 1 minute 40 sec |
| **1 real microsecond** | 16 minutes 40 sec |
| Read 1 MB from RAM sequentially | 2 hours 46 minutes |
| Kernel launch overhead | 8 hours - 1 day |
| CUDA API call overhead | 1 day |
| Small array multiply (CPU, 101 elements) | 1 day 12 hours |
| PCIe random access (8-byte read, GPU) | 2 - 3 days |
| RTT data center | 5 days 19 hours |
| **1 real millisecond** | 11 days 14 hours |
| Read 1 MB from SSD sequentially | 12 days |
| Local database insert operation (e.g., PostgreSQL commit) | 1 month 6 days |
| Small vector square (CPU, 5,000 elements) | 1 month 18 days |
| Read 1 MB over 1 Gbps network sequentially | 3 months 26 days |
| Seek and read 1 MB from HDD sequentially | 6 months |
| **60 fps thershold** | 6 months 6 days |
| Large vector square (GPU, 1M elements) | 1 year |
| RTT to other city on the same continent | 1 year 17 days |
| Human perception threshold of time passing | 1 year 4 months - 2 years 8 months |
| DNS query (typical) | 1 year 7 months |
| Wifi latency to internet | 1-6 years |
| **100 real milliseconds** | 3 years 2 months |
| 5G mid-band latency to internet | 4-7 years |
| TCP round trip between continents | 4 years 9 months |
| Large vector square (CPU, 1M elements) | 28 years |
| **1 real second** | 31.7 years |

Data has distance. **Abstractions hide distance, they do not remove it.**

### Mechanical Sympathy

Understanding the physical constraints of the machine and designing software that cooperates with them.
To be a great driver you don't need to be an engineer but you must have a sympathy how the machine works.

Hardware reality:
* Data is stored in blocks
* Access has setup cost
* Sequential access is predictable
* Random access is expensive

Databases are shaped by disks, i.e., LSM trees optimize for writes to append sequentially to disk and trade off read complexity. **The machine rewards sequential work**.

### The Pipe Problem

Need to know if we're latency bound or bandwidth bound?

| Latency                 | Bandwidth           |
| ----------------------- | ------------------- |
| Time for ONE request    | Requests per second |
| Speed-of-light limited  | Pipe width          |
| Distance dominated      | Parallelism         |
| Hard to improve         | Easy to buy more    |

Latency = how long to move a unit from point A to B  
Bandwidth = how many units you can move from point A to B

**Cant solve latency problems with more bandwidth.**

### Little's Law

```L = λ * W```  
Where:
* L = in-flight requests. Long-term average number of items in a *stable* system.
* λ = arrival rate. Long-term average *effective* arrival rate.
* W = latency. Average time an item spends in the system.

Interpretation: If latency increases then in-flight requests must increase.
> If the service takes 1 sec to process a request and you get 100 requests per second you need to be able to accomodate 100 in-flight requests on average. If latency increases to 2 seconds we need to store 200 in-flight requests on average.  

Let:
* S = average service time
* N = number of workers
* `μ = N / S` = service capacity

A system is stable if `λ < μ`.  
**Unstable systems don't fail gradually. They hit a wall.**

Utilization:  
`ρ = λ / μ`  
> If λ = 10 requests/sec, N = 5 workers, and S = 0.4 sec, then μ = 5/0.4 = 12.5 req/sec, and ρ = 10/12.5 = 0.8 (80% utilized).  
> The system remains stable as long as λ < μ, i.e., ρ < 1.  

This can also be interpreted as the probability that the server is busy. With this we can also get the expected latency:  
`W = S / (1 - ρ)`  
which says:
* At low utilization (ρ → 0): W ≈ S, latency is just the service time itself, no waiting.
* At high utilization (ρ → 1): W → ∞, latency blows up as the queue backs up.  
> This is why systems that run at 90%+ utilization tend to feel so slow, at ρ = 0.9, expected latency is already 10× the service time.

### Economics of the Machine

In the cloud you are not buying compute, you are renting time and physics. Every architectural choice is a physical trade-off expressed as a bill.  
Relative storage costs:
* RAM = 100 x SSD
* SSD = 10 x cold storage (S3)

Latency decreases as cost rises.

| Hot data | Cold data |
| -------- | --------- |
| Frequently accessed | Rarely accessed |
| Latency sensitive | Latency tolerant |
| Worth expensive physics | Should live cheaply |

**The goal is to align the value of the data with the cost of the medium.**

### Questions to ask

1. How far the data has to travel? (latency)
2. How wide is the pipe? (bandwidth)
3. Is the work sequential or random?
4. What does Little's Law imply about my capacity?
5. Is the cost justified by the business value?
