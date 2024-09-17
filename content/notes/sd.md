+++
title = "System Design Notes"
description = "Benjamin Schwartz's System Design Notes"
date = "2024-09-14"
aliases = ["sd"]
author = "Benjamin Schwartz"
weight = 3
+++

## Table of Contents

- [Chapter 1](#chapter-1)
  - [Web Tier](#web-tier)
  - [Data Tier](#data-tier)
  - [Cache Tier](#cache-tier)
    - [Content delivery network (CDN)](#content-delivery-network-(cdn))
  - [Statefless vs Stateful Architecture](#statefless-vs-stateful-architecture)
  - [Data Centers](#data-centers)
  - [Message Queue](#message-queue)
  - [Logging, Metrics, Automation](#logging,-metrics,-automation)
  - [Vertical vs. Horizontal Scaling](#vertical-vs.-horizontal-scaling)
    - [Sharding for scaling Databases](#sharding-for-scaling-databases)
  - [Summary of Scaling Up](#summary-of-scaling-up)
- [Chapter 2](#chapter-2)
  - [Useful Latency Numbers](#useful-latency-numbers)
  - [Back-of-the-envelope Estimations](#back-of-the-envelope-estimations)
    - [Twitter Example](#twitter-example)
  - [Framework](#framework)
    - [1. Understand problem and establish design scope (3-10mins)](#1.-understand-problem-and-establish-design-scope-(3-10mins))
    - [2. Propose high-level design and get buy-in (10-15mins)](#2.-propose-high-level-design-and-get-buy-in-(10-15mins))
    - [3. Design Deep Dive (10-25mins)](#3.-design-deep-dive-(10-25mins))
    - [4. Wrap up (3-5mins)](#4.-wrap-up-(3-5mins))
- [Rate Limiter](#rate-limiter)
    - [High-level Architecture](#high-level-architecture)
- [Design a News Feed System](#design-a-news-feed-system)
  - [Design Proposal](#design-proposal)
    - [Feed Publishing API](#feed-publishing-api)
    - [Newsfeed Retrieval API](#newsfeed-retrieval-api)
    - [Feed Publishing Features](#feed-publishing-features)
    - [Feed Publishing Features](#feed-publishing-features)
  - [Deep Dive](#deep-dive)

# System Design

[Alex Xu - System Design Interview](https://www.amazon.com.au/System-Design-Interview-insiders-Second/dp/B08CMF2CQF)

# Chapter 1

## Web Tier

- **DNS:** Domain Name system ⇒ paid service provided by 3rd parties.
    - Request: domain name, response: IP address
- **HTTP:** Format by which requests are sent between client and server
    - Web server might respond with HTML pages or JSON response
- **Mobile Application vs. Web Application**
    - Makes up the traffic to our web server
    - **Web app**: uses **server-side** languages for **business logic** (Java, Python, etc.) as well as **client-side** languages for **presentation**
    - **Mobile app:** uses HTTP communication, JSON is commonly used response format

## Data Tier

- **Databases**
    - Separate web/mobile traffic and database servers ⇒ scale independently
    - Web server communicates with Database server
    - **RELATIONAL (RDBMS) / SQL**
        - E.g. MySQL, PostgreSQL, Oracle DB
        - Can perform join operations
    - **NON-RELATIONAL / NoSQL**
        - E.g. CouchDB, Neo4j, MongoDB
        - Used for: super-low latency, unstructured (not relational), need to serialize/deserialize data (JSON, XML, etc.), massive amount of data
- **Vertical vs Horizontal scaling**
    - **Vertical**: just adding more compute power. Simpler solution, good for low-traffic
    - **Horizontal**: adding more servers, better for large scale, if one server fails can still continue working
- **Load balancer**
    - Solution for adding more servers
    - Works by converting a **public IP** into a series of **private IPs** (reachable only between servers on the same network) to reach the servers
- **Database Replication**
    - Follows **master/slave** model. Master is **write-only** and slaves are **read-only**. Slaves receive copy of data from master. Many more slaves than masters.
    - Slaves are essentially copies of the master. Read operations are distributed across slaves (so can be done in parallel)
    - Preserves data, more performant, highly available (even if a database is offline)
    - If **master goes offline** a slave is **promoted**

## Cache Tier

- **Cache:** temporary storage area between web server and DB. Is much faster and reduces DB workload
    - Typically: **read-through** cache (if cache has data, send back to client. Else, query database and store response in cache)
    - Good for **reading frequency, modifying infrequently**.
- **Considerations:** Decide when to use, expiration policy (remove expired from cache), consistency (DB and Cache in-sync), Mitigating failures (avoid SPOF by having multiple cache servers), Eviction policy (LRU, LFU, FIFO)

### Content delivery network (CDN)

- Basically just a **read-through** cache for static content (images, videos, CSS, JavaScript files, etc.)
- **Geographically dispersed servers** ⇒ CDN closest will try to deliver content
- URL’s domain is provided by the CDN provider (e.g. Amazon, Akamai)
    - If not found in CDN server, requests from source (either web server or online storage)
    - TTL (Time-to-Live) describes how long image is cached for

## Statefless vs Stateful Architecture

Want to move **session data** from **web tier** to **data store**

- **Stateless:** Want to store state (e.g. session data) in a DB (each webserver can access this). Effectively a **shared data source**
    - Simpler, more robust, scalable
    - NB: use a **NoSQL** data store as its easy to scale.
- **Stateful:** Remebers client data from one request to the next. Each HTTP request must be routed to the **same server as last time**

<aside>
💡

***KEY: 
Stateless server:*** Keeps **no state information**. All of this is stored in a **shared data source**

***Stateful server:*** Remebers client data/state (e.g. user session data, user profile image, etc.) from one request to the next

**Stateless** moves the session data **out of the web tier**

</aside>

![webserver](/images/webserver.png)

## Data Centers

- Separate **web servers, databases, and caches** into their own datacentres. However, they refer to the same **statless DB for user sessions**
- **GeoDNS:** Directs traffic to nearest data center depending on user location
- **Data Synchronization:** Netflix uses asynchronous multi-data center replication

## Message Queue

- **Buffer** which supports **asynchronous communication**
- Producers == > Message Queue < == > Consumer
- Multiple consumers can connect to queue

## Logging, Metrics, Automation

- **Logging** monitors errors in system
- **Metrics** could mean system health (CPU, Memory, disk I/O etc.), aggregated level (e.g. entire DB tier), or business metrics
- **Automation** could be CI/CD tools to help productivity

## Vertical vs. Horizontal Scaling

- **Upper limit** on improving hardware, also very expensive. Increases single point of failure risk

### Sharding for scaling Databases

- Method of separating databases into smaller pieces
    - can use a **hash function** on a **sharding** **key** (like a `user_id` ) to determine which DB to use ⇒ choose a **sharding key** that evenly distributes data
    - Means each **shard** contains **unique data**
- **Problems:**
    - **resharding data** (uneven data distribution)
    - **celebrity** problem (certain shard swamped with accesses ⇒ social applications)
    - **join and de-normalization** ⇒ hard to perform joins across shards
        
![webserver2](/images/webserver2.png)

## Summary of Scaling Up

- Stateless web tier (move state to shared DB)
- Redundancy at each level (DB replication)
- Cache data wherever possible
- Support multiple data centers (Load balancer distributes requests)
- Host static assets on CDN
- Scale DB’s by sharding
- Split tiers into individual services
- Monitor system and automation tools

# Chapter 2

## Useful Latency Numbers

| Operation | Time |
| --- | --- |
| L1 cache reference | 0.5ns |
| Branch mispredict | 5ns |
| L2 cache reference | 7ns |
| Mutex lock/unloc | 100ns |
| Main memory reference | 100ns |
| Read 1MB sequentially from memory | 250μs |
| Round trip within same datacenter | 500μs |
| Disk seek | 10ms |
| 1MB sequentially from network | 10ms |
| 1MB sequentially from disk | 30ms |
| Pack from CA → Netherlands → CA | 150ms |
- Disk seeks very slow, simple compression is fast, compress data before sending

## Back-of-the-envelope Estimations

- **Common estimations:** QPS (Queries per second, peak QPS, storage, cache, number of servers, etc.)

### Twitter Example

**Estimate QPS and Storage requirements**

- Assumptions:
    - 300 millions monthly active users
    - 50% use daily
    - Users post 2 tweets/day average
    - 10% data contains media
    - Data stored for 5 years
    - Average tweet size: 64B tweet_id, 140B content, 1MB media
- **QPS**: 150M * 2 / 24 / 3600 = 3500 QPS
- Peak QPS = 2 * QPS = 7000
- **Media storage:**
    - 150M * 2 * 10% * 1MB = 30TB /day
    - 30TB * 365 * 5 = 55PB / year

## Framework

### 1. Understand problem and establish design scope (3-10mins)

- slow down and think deeply
- ask questions to clarify requirements and assumptions
    - what specific features are we going to build?
    - how many users does the product have?
    - how fast is the anticipated scale up?
    - what is the tech stack? what existing services to simplify design?

### 2. Propose high-level design and get buy-in (10-15mins)

- create initial blueprint. Ask for feedback
- use box diagrams with key components
- back-of-the-envelope calculations, think out loud
- think through some concrete cases. Get areas to focus on in deep dive

### 3. Design Deep Dive (10-25mins)

- Interviewer may identify component to prioritize (e.g. how to reduce latency)

### 4. Wrap up (3-5mins)

- Discuss potential improvements
- Give a recap of the design
- Error cases are interesting
- Operation issues → monitor metrics/error logs
- Scale up, what changes needed?

# Rate Limiter

- Limits HTTP requests from a given client to a certain threshold
    - Prevents DoS attacks, reduces cost (fewer servers needed), prevent overworking
- **middleware:** “software glue”. Could place the rate limiter between client and API servers
- Can be handled by a component called **API Gateway** (supports rate limiting, SSL termination, authentication, IP whitelisting, servicing static conent) ⇒ **microservice architecture**
    - Implementing on server-side gives full control
- **Rate limiting algorithms:**
    - **token bucket:** Tokens represent **amount of requests we can service.**
        - Bucket is refilled at a preset rate. Bucket has fixed capacity. Once full, no more tokens are added. Every request consumes a token, and once empty request is dropped.
        - Parameters: **bucket size, refill rate**
        - Might have one bucket per IP address, or a global bucket
    - **leaking bucket:** implement with FIFO queue, requests dropped if queue full
    - **fixed window counter:** counter for each time window, drops requests once exceeds
        - spike in traffic at edge of a window could cause more requests than allowed to go through
    - **sliding window log:** fixes previous issue.
        - Each request has a **timestamp.** When a new request comes in, remove old requests (time outside of window). If log size exceeds limit, drop request, otherwise process it.
    - **sliding window counter:** complicated hybrid of fixd window and sliding window

### High-level Architecture

- For counter, **in-memory** **cache** will be much faster than a database (disk is slow)
    - **Redis** for rate limiting: INCR/EXPIRE

# Design a News Feed System

- Constantly updating list of stories
- **Facebook**: Includes status updates, photos, videos, links, app activity, and likes from people, pages and groups you follow

## Design Proposal

- Two flows: **feed publishing** (user publishes post, write data to cache and DB, post populated to friends’ news feeds) **** and **feed building** (aggregate friends’ posts in reverse chronological)

### Feed Publishing API

- HTTP **POST** request: POST/v1/me/feed. **params:** content, auth_token (authenticate API requests)

### Newsfeed Retrieval API

- HTTP **GET** request: GET/v1/me/feed. **params:** auth_token

### Feed Publishing Features

- Load balancer
- Web servers
- Post service (persist post in DB and **post cache**)
- Fanout service (push content to friends’ news feed, this data stored in **news feed cache** for fast retrieval)
- Notification service (inform friends that new content available, send push notification)
    
![feedpublisher](/images/feedpublisher.png)
    

### Newsfeed Building Features

- Load balancer
- Web servers
- Newsfeed service (fetches from **news feed cache**)
- **Newsfeed cache** (stores news feed IDs needed to render news feed)
    
![feedbuilder](/images/feedbuilder.png)

## Deep Dive

