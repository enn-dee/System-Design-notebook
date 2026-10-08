# System Design Notebook

Gaurav Sen's System Design playlist, one page per video, from basics to advanced. Tap "watched" on a page to tick it off.

These notes are written from the topics each video covers, not from its transcript, so numbers and examples can differ from what he says. Use them as a revision layer on top of watching. Video titles marked **?** are slots whose exact title I could not confirm.

[1 Primer](#v1)[2 Scaling](#v2)[3 Load balancing](#v3)[4 Consistent hashing](#v4)[5 Message queue](#v5)[6 Microservices](#v6)[7 Sharding](#v7)[8 Caching](#v8)[? CDN, Pub/Sub, NoSQL](#vx)[Tinder](#v-tinder)[Instagram](#v-insta)[WhatsApp](#v-wa)[Netflix](#v-nf)[Estimation](#v-cap)[TikTok](#v-tt)[Tips](#v-tips)[Cheat sheet](#cheat)

Part A foundations: why and how systems grow

## 1. System Design Primer: how to start with distributed systems

### The story of one server

- Start: one machine runs the app and the database. Works for hundreds of users.
- It breaks in two ways: it is slow when load grows, and when it dies everything is down (single point of failure).
- Fix step by step: split the DB from the app, add more app servers, put a **load balancer** in front, add a **cache**, add a **queue** for slow work, **replicate** then **shard** the DB.

### Words to own

- **Scalability**: handles more load by adding resources.
- **Availability**: fraction of time the system answers (99.9% = about 8.8 hours down per year).
- **Latency**: time for one request. **Throughput**: requests per second.
- **Fault tolerance**: keeps working when parts fail.
- **Consistency**: every reader sees the same data.

Every later video is one tool that fixes one limit of the single server. Ask "which limit does this tool fix, and what does it cost me?"

## 2. System Design Basics: Horizontal vs. Vertical Scaling

|  | Vertical (scale up) | Horizontal (scale out) |
| --- | --- | --- |
| How | Bigger CPU, RAM, disk | More machines |
| Calls between parts | In-process, fast | Network calls (RPC), slower, can fail |
| Failure | One machine, one point of failure | Survives losing a node |
| Limit | Hardware ceiling, cost rises steeply | Almost none, commodity boxes |
| Data | Consistent, one copy | Needs sharding, replication, consistency work |
| Needs | Downtime to upgrade | Load balancer, service discovery |

"I would scale vertically first because it is simple, then go horizontal once I hit the hardware limit or need fault tolerance. Horizontal costs me network latency and consistency work."

## 3. What is Load Balancing?

### Why

- Spread traffic so no server is overloaded, remove the single server as a failure point, and add or remove servers without clients knowing.

### How it picks a server

- **Round robin**: 1, 2, 3, 1, 2, 3. Simple, ignores load.
- **Weighted round robin**: bigger machines get more turns.
- **Least connections** / **least response time**: send to the least busy.
- **IP hash**: same client goes to the same server (cheap stickiness).
- **Consistent hashing**: for caches and stateful servers (video 4).

### Details that get asked

- **Health checks**: LB pings servers and stops sending to dead ones.
- **Layer 4** (TCP/IP, fast, blind to content) vs **Layer 7** (HTTP, can route by URL, header, cookie).
- **Sticky sessions** vs keeping servers stateless and putting session data in Redis. Stateless is better.
- The LB is itself a single point of failure, so run two (active-passive or active-active) and use DNS in front.
- Tools: Nginx, HAProxy, AWS ELB, hardware appliances.

Load balancers work best in front of stateless servers. State lives in a DB or cache.

Part B data: spreading, storing and speeding it up

## 4. What is Consistent Hashing and where is it used?

### The problem

- Spread keys over N servers with `hash(key) % N`.
- Change N (add or lose a server) and almost every key maps somewhere new. A cache loses nearly everything and the DB gets flooded.

### Virtual nodes

- With few servers the ring is lopsided. Give each server many points (A1, A2, A3...) so load evens out and a failed server's keys spread over many survivors.
- A stronger machine gets more virtual nodes.

### Used in

- Distributed caches (Memcached clients), Cassandra, DynamoDB, CDNs, load balancers, sharded databases.

"Plain modulo hashing reshuffles nearly all keys when the cluster size changes. A hash ring with virtual nodes moves only about K/N keys."

## 5. What is a Message Queue and where is it used?

### Why use one

- **Decouple**: producer does not wait for, or even know, the consumer.
- **Absorb spikes**: queue buffers, consumers work at their own pace.
- **Async work**: send the email, resize the image, encode the video later.
- **Retry** failed work. Messages that keep failing go to a dead letter queue.

### Things to know

- Delivery: at-most-once, **at-least-once** (most common, so make consumers idempotent), exactly-once (hard, costly).
- Ordering holds per partition, not across the whole topic.
- Scale consumers by adding more, and by adding partitions.
- Tools: Kafka (log, replayable, very high throughput), RabbitMQ (smart routing), SQS (managed).

If the user does not need the result immediately, put it on a queue and answer "accepted".

## 6. What is a Microservice Architecture and what are its advantages?

- **Monolith**: one codebase, one deploy, one DB. Easy to start, hard to scale one part or let many teams work in parallel.
- **Microservices**: many small services, each owns one business capability and its own data, talking over HTTP/gRPC or events.

### Wins

- Deploy and scale each service on its own (scale only "search").
- A bug or crash stays in one service (fault isolation).
- Teams own services, pick their own tech.

### Costs

- Every call is a network call: latency, timeouts, partial failure.
- No cross-service ACID transaction. Use **saga** (steps plus compensating actions).
- Harder debugging: need tracing, central logs, monitoring.

### Pieces around them

- **API gateway**: single entry, auth, rate limit, routing.
- **Service discovery**: how services find each other (Consul, Eureka, Kubernetes DNS).
- **Circuit breaker**: stop calling a failing service for a while.
- **Database per service**, async events between services (video 5).

Start with a well-structured monolith. Split when team size or scaling needs justify the operational cost.

## 7. What is Database Sharding?

- Sharding = split one big table across machines by a **shard key**, each holds a slice. Fixes both storage size and write load.
- It is **horizontal partitioning**. Vertical partitioning splits by columns or by feature instead.
- Not the same as replication: replication copies the same data (for reads and failover), sharding divides different data.

### Ways to pick a shard

- **Range** (A to H): easy, but hot ranges.
- **Hash** of key: even spread, range queries get hard. Pair with consistent hashing.
- **Directory** (lookup table): flexible, the table is a new point of failure.
- **Geo**: users near their data.

### Pain points

- Hot shards (celebrity user), uneven data.
- Joins and transactions across shards are slow or unsupported.
- Re-sharding when you outgrow it.

"I would shard by user id with hashing, replicate each shard for availability, and avoid cross-shard queries by keeping related rows on the same shard."

## 8. Caching in distributed systems: a friendly introduction

### Where caches sit

- Browser, CDN, app memory, a shared cache (Redis, Memcached), DB buffer. Closer to the user is faster.

### Write strategies

- **Cache-aside**: app reads cache, on miss loads DB and fills cache. Most common.
- **Read-through**: cache loads from DB itself.
- **Write-through**: write cache and DB together. Safe, slower writes.
- **Write-back**: write cache, flush to DB later. Fast, can lose data.
- **Write-around**: write DB only, cache fills on read.

### Eviction and trouble

- Eviction: **LRU** (default), LFU, FIFO, plus **TTL**.
- Stale data: invalidate on write or accept a short TTL. "Cache invalidation" is the hard part.
- **Stampede**: a hot key expires and thousands hit the DB. Use locks, early refresh, jittered TTLs.
- Distributed cache: spread keys with consistent hashing (video 4) and replicate hot keys.

Cache what is read often and changes rarely. Say how stale data is allowed to be.

## ? Slots I could not confirm: CDN, Pub/Sub, NoSQL

The playlist is said to cover these between caching and the case studies. Notes below are generic, check the video titles against your list.

### CDN

- Servers spread worldwide that keep copies of static files (images, video, JS) near users. Lower latency, less load on your origin.
- **Pull**: fetched from origin on first request. **Push**: you upload ahead of time.
- Control freshness with cache headers, TTL, versioned file names.

### Publisher / Subscriber

- Publisher sends to a **topic**, every subscriber gets a copy (queue: one consumer per message).
- Use for events: "order placed" feeds billing, email, analytics at once. Tools: Kafka topics, Google Pub/Sub, SNS.

### NoSQL

- Types: key-value (Redis, DynamoDB), document (MongoDB), wide-column (Cassandra), graph (Neo4j).
- Trade strict schema and joins for easy horizontal scale and flexible data. Often **eventual consistency**.
- Pick SQL for relations and transactions, NoSQL for huge scale, simple access patterns, flexible shape.

**CAP**: during a network partition you choose consistency or availability. Many NoSQL stores choose availability.

Part C case studies: put the tools together

## System Design: Tinder as a microservice architecture

### Requirements

- Create profile with photos, see nearby profiles, swipe left or right, match when both swipe right, chat after a match.

### Services

- **Profile service**: user data in a DB, photos in object storage behind a CDN.
- **Recommendation service**: finds nearby users who fit filters. Needs a geo index (geohash or quadtree, search index) and precomputed candidate stacks so swiping is instant.
- **Swipe / match service**: records swipes; on a right swipe checks if the other person already swiped right, then creates a match. Very write heavy, so shard it by user id.
- **Chat service**: websocket connections, messages stored per match (see WhatsApp).
- **Notification service** fed by a queue: "you have a match".

### Points to raise

- Do not show someone twice: keep a seen set per user (a Bloom filter works).
- Location changes often, so do not shard only by location. Shard users by id, index location separately.

## Designing Instagram: System Design of News Feed

### Core pieces

- Upload photo (blob storage plus CDN), metadata in a sharded DB, follow graph, feed per user.
- Read heavy: far more feed reads than posts. Optimise the read path.

### Feed generation: the key choice

- **Pull (fan-out on read)**: build the feed when the user opens the app, merging recent posts of everyone they follow. Cheap to write, slow to read.
- **Push (fan-out on write)**: when someone posts, push the post id into each follower's feed list in cache. Fast reads, heavy writes.
- **Hybrid**: push for normal users, pull for celebrities with millions of followers, merge at read time.

### Supporting detail

- Fan-out work runs on queues so posting stays quick.
- Rank the feed with a score (recency, closeness, engagement), page it with a cursor.
- Store only post ids in the feed cache, fetch post bodies in a batch.

"Push for most users, pull for celebrities, merge at read time. That keeps both write fan-out and read latency under control."

## WhatsApp System Design: Chat Messaging Systems for Interviews

### Connection

- Clients keep a persistent **websocket** to a chat server, so the server can push messages instead of the client polling.
- A session store (Redis or ZooKeeper-like) maps user id to the chat server they are connected to.

### Sending a 1:1 message

- A sends to its chat server, the server looks up B's server and forwards.
- If B is offline, store the message in a DB and deliver when B reconnects, then delete.
- Acks drive the ticks: one tick = reached server, two = delivered to device, blue = read.
- Order messages with a per-chat sequence number or timestamp.

### Groups and media

- Group has a member list, server fans the message out to each member. Cap group size to limit fan-out.
- Media goes to blob storage, the chat message carries only a link and a key.
- End-to-end encryption: server relays ciphertext it cannot read.
- "Last seen" comes from heartbeats on the connection.

Chat = websocket + session map + durable store for offline users + acks.

## How Netflix onboards new content: video processing at scale

- A studio uploads a huge source file. Players need many **resolutions, bitrates, codecs** for phones, TVs, slow and fast networks.
- Split the file into chunks so chunks encode in parallel on many workers. Work is a graph of tasks (DAG) fed through queues.
- Run quality checks, then store outputs in object storage.
- Push popular titles ahead of time to **CDN** servers near users (Netflix runs its own, Open Connect).
- Playback uses **adaptive bitrate streaming** (HLS / DASH): the player switches quality chunk by chunk as bandwidth changes.

Chunk, parallelise, retry failed chunks, pre-position on the CDN.

Part D interview craft: estimation and communication

## Capacity Planning and Estimation: how much data does YouTube store daily?

### Method

- State assumptions out loud, round hard, use powers of ten, show the arithmetic.
- Convert to per second: 1 day is about 86,400 s, roughly 10^5 s.

### Worked example (assumptions are mine)

- Assume 500 hours of video uploaded per minute.
- 500 x 60 x 24 = 720,000 hours per day.
- Assume about 1 GB per hour for the stored original: roughly 720 TB per day.
- Multiply for several encoded resolutions (say x3) and 3 replicas: about 6 PB per day.
- Do not forget thumbnails and metadata, which are small by comparison.

### Numbers to keep in your head

- Memory read about 100 ns, SSD read about 100 us, disk seek about 10 ms.
- Same-datacenter round trip about 0.5 ms, across continents about 150 ms.
- 1 KB = 10^3, 1 MB = 10^6, 1 GB = 10^9, 1 TB = 10^12 bytes.
- Rule of thumb: 1 million requests per day is about 12 per second.

Interviewers judge your reasoning, not the exact number. Estimate storage, read and write QPS, bandwidth.

## System Design Interview: TikTok architecture (with sudoCODE)

- **Upload path**: client uploads to blob storage, a queue triggers transcoding into several qualities, thumbnails and metadata are written, then the video is pushed to the CDN.
- **Feed ("For You")**: recommendation is the product. A candidate generator picks a few hundred videos from millions, a ranking model scores them, the feed service returns the top ones.
- Signals: watch time, likes, shares, replays, follows. Events stream through Kafka into feature stores and training jobs.
- **Cold start**: show a new video to a small test audience, widen reach if engagement is good.
- **Playback**: prefetch the next few videos while the user watches, serve chunks from the nearest CDN.
- Counters (likes, views) are write-heavy: batch them through a queue and aggregate.

## 5 Tips for System Design Interviews

Below is the usual framework, check it against what he says in the video.

- **Clarify** the problem: functional features, then non-functional needs (scale, latency, availability, consistency).
- **Estimate** load and storage so design choices have numbers behind them.
- **Draw the high level**: clients, load balancer, services, DB, cache, queue. Then deep dive on what the interviewer cares about.
- **Discuss trade-offs** for every choice (SQL vs NoSQL, push vs pull). No answer is free.
- **Think aloud** and invite feedback. It is a conversation, not an exam.

Video 26 (InterviewReady walkthrough) is a walkthrough of his paid course, so there are no notes for it.

## Cheat sheet: problem to tool

### One server is slow

- Scale out + load balancer

### Cache keeps losing keys when nodes change

- Consistent hashing

### Slow work blocks users

- Message queue

### DB too big or too many writes

- Sharding (+ replicas for reads)

### Same reads again and again

- Cache, CDN for static files

### Many teams, parts scale differently

- Microservices

### Real-time messages

- Websockets + session map

### Heavy file processing

- Chunk + parallel workers + queue

### Celebrity or hot key

- Hybrid push/pull, replicate hot keys