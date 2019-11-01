1. Usecase of FB: a lot a read, less write
2. Memcached - distributed sysmtem of in-memory cache(sharding memory)
3. Read: go the memcache if not go to database
   Write: go to database and delete items in memcache(choose delete because delete is idempotent and cache is not persisten store) 
4. Items are distributed across memcached servers through consistent hashing
5. how to reduce latency and load:
- latency reducing
  - use memcache client
  - memcache client runs on same server of webserver, functionalitity including serialization, compression, request routing, error handling, and request batching(can think of preprocessing and fail-fast). Clients maintains a map of availble servers, which is updated through auxiliary configuration system
  - UDP for read, TCP for write(set and delete）
  - parallel requests and batching: webserver knows the shortest way to get data(DAG of dependency of data), webserver batch average 24 keys per request
  - how to address incast congestion: flow-control
  - a client use a sliding window mechanism to control number of outstanding request, when the clients receive a response, the next request can be sent, size of window grows slowly upon a successful request and shrinks when a request goes unanswered like a TCP congestion control. Window applies to all memcache request independently of destination.
  - lower window size: the application will have to dispatch more groups pf memcache requests serially, increasing the duration of the web request
  - larger window size: number of simultaneous memcache request causes incast congestion
  - tuning paramemter of size of sliding window
- load reducing
  - stale sets: A stale set occurs when a web server sets a value in memcache that does not reflect the latest value that should be cached. This can occur when concurrent updates to memcache get reordered
  - thundering herd - happens when a specific key undergoes heavy read and write activity
  - As the write activity repeatedly invalidates the recently set values, many reads default to the more costly
path. lease mechanism solves both problems
  - memcache grant a lease to application when client experiencing a cache miss. The lease is a 64-bit token bound to the specific key the client originally requested. The client provides the lease token when setting the value int the cache, memcached can verfiy and determine whether the data should be stored and thus arbitrate concurrent writes. Verification can fail if memcached has invalidated the lease token due to receiving a delete request. This mitigates stale sets
  - Each memcached server regulates the rate at which it returns tokens. By default, we configure these servers to return a token only once every 10 seconds per key. Requests for a key’s value within 10 seconds of a token being issued results in a special notification telling the client to wait a short amount of time. Typically, the client with the lease will have successfully set the data within a few milliseconds. Thus, when waiting clients retry the request, the data is often present in cache.
  - memcache pools
    - problem to address: memcache as a general-purpose cacheing layer requires workloads to share infra despite of access patterns, memory footprints, and quaility-of-service requirements. Different applications' workloads can produce nagative interference resulting in decreated hits rates
     - partition cluster memcached servers into separate pools, for example:
       - a small pool for keys that are accessed frequently but for which a cache miss is inexpensive
       - a large pool for infrequently accessed keys which cache misses are prohibitively expensive


 
