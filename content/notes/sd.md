+++
title = "System Design Notes"
description = "Benjamin Schwartz's System Design Notes"
date = "2024-09-14"
aliases = ["sd"]
author = "Benjamin Schwartz"
weight = 3
+++

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
- **Stateful:** Remebers client data from one request to the next. Each HTTP request must be routed to the **same server as last time**

![webserver](/images/webserver.png)

## Data Centers

- Separate **web servers, databases, and caches** into their own datacentres. However, they refer to the same **statless DB for user sessions**
- **GeoDNS:** Directs traffic to nearest data center depending on user location
- **Data Synchronization:** Netflix uses asynchronous multi-data center replication

## Message Queue

- Supports **asynchronous communication**
- Producers == > Message Queue < == > Consumer
- **Buffer** for **asynchronous requests**. Multiple consumers can connect to queue

