# The Cost of Communication

https://www.youtube.com/watch?v=YOR9hAFoe2o

Your code almost never runs in a vacuum. Your service is part of a giant screaming nervous system of other services.

## The Fallacy of the Local Call

```Python
def get_user_profile(user_id):
  user = db.fetch_user(user_id)
  perferences = cache.get(f"perfs:{user_id}")
```
Seems clean and simple. Make a call get a value back. Feels local.   
But a network call gets physical. If we scale up:  
L1 cache hit -> 1 second  
RAM Access -> 3 minutes  
1ms Network Call -> **11 days**  
> Imagine you had to wait 11 days to check your phone. You would organize your life very differently.  

The local call abstractions hide this truth from us.

## The Tax

This is what the operating system and the protocols demand at every step.  

### IP lookup.  
Your computer doesn't know what `api.payments.internal` is. First it checks the local cache and if not found sends a UDP packet to a DNS server. If it doesn't know it sends to another one. A round trip. 1-10ms.

### Connection handshake.  
You want TCP because you need reliablity so you send SYN, server sends SYN-ACK, you send ACK. Another round trip, still no data.

### Crypto overhead.  
Your data is sensitive so now you need the TLS handshake. Exchange certificates, agree on cipher suite, generate session keys. 2 more round trips (in TLS 1.2). Still no data. Your CPU also had to do elliptic curve math.

### Context switching.  
Once you get the data it is recieved by the network interface card and it resides in kernel space. The bytes need to be copied to your user space app. That is a syscall. The OS stops runnig the code, switches to kernel model, copies the data then needs to switch back.

### Reliability tax. 
The network is a shared resource. If you packet gets dropped, TCP will eventually send it again, but your 1ms request became 200ms.

### Serialization tax.  
The network doesn't understand 'objects' only streams of bytes so whatever the data is, we need to flatten it out into a series of bytes then deserialize it on the other end. JSON is a massive amount of string manipulation, parsing and memory allocation (GC pressure). Binary or Protobuf is direct copy without parsing, order of magnitude faster.

### Physics tax.  
The fastest way for light to get from London to San Francisco is 85ms so the fastest https/tcp request is still 85 + 170 + 85 = 340ms. QUIC / HTTP3 can bring this down to 170ms by merging handshakes. CDNS can also move handshakes closer to the user.

## Notable cases

### Discord  
Uses HTTP2 and gRPC. The app only connects to the Gateway service and calls other services through that, they use the same TCP connection. gRPC allow zero-copy deserialization so the network buffer maps directly to app memory.

### Apache ARROW
The task: send massive datasets through the network. This means serialize to CSV or JSON, send over the wire then deserialize to a NumPy array. Instead they defined a Standard Memory Layout, where data on the disk, on the wire and in the RAM are identical, thus allowing zero-copy deserialization.

## The Architect's Rules

The most efficient way to communicate is to not communicate at all.

### 1. Batching
Gather all the data and requests you can into a single trip.

### 2. Data Locality
Every time you split a call into a service call you turn a local call into a network call. Keep the data close to the compute.

### 3. Coarse-Grained APIs
Design interfaces to be chunky not chatty. Keep both request and response high density.
