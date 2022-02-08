---
layout: default
title:  "Debugging ECONNREFUSED/ECONNRESET on a microservices architecture"
---

# Understanding ECONNREFUSED/ECONNRESET on a Microservices architecture

## Every problem starts with a symptom
A few days ago I noticed an increased number of events on our error reporting platform. The events indicated some errors with one of our Node.js microservices.

The error messages were the following:
- `Error: read ECONNRESET`. 
- `Error: connect ECONNREFUSED 172.x.x.x:xxxx[REDACTED]`

I took a closer look and noticed that the errors came from the same line of code in which an HTTP request has been made against other Node.js microservice.

Filtering the errors by server instance revealed that it always started with a few `ECONNRESET` errors, then, a lot of `ECONNREFUSED` errors for a couple of seconds, and then the errors ceased for a couple of minutes until it started again `ECONNRESET->ECONNREFUSED`.

If it was only an `ECONNREFUSED` error, we could assume that the problem was that the API server was down, but there are two problems with this assumption, the errors were not consistent, meaning that sometimes the API server returned a successfully 20x response, second, we can't ignore the second error - `ECONNRESET`.

In contrast to `ECONNREFUSED` which his name is self-explanatory, `ECONNRESET` deserves an explanation:

`ECONNRESET` occurs when the other side of the connection (in this case `172.x.x.x:xxxx`) closed the connection in a way that was probably not normal.

The question is why? why would the API server drop connections and for a couple of seconds later, even refuse them?

# Finding the cause of the symptoms
The first guess we took is that the API server (deployed on Kubernetes) was OOMKilled by Kubernetes.

> The term `OOMKilled` means that the application tried to use more memory than its limit, so Kubernetes "killed" it.

This could explain the errors. When the API server leaked memory, it was forced to kill itself, hence the `ECONNRESET`, and `ECONNREFUSED` could be caused by the time it took the API server to restart.

But wait, it can't be `OOMKilled`, because:
- The API server had multiple replicas (pods), so even if one replica was `OOMKilled` it can explain the `ECONNRESET` but not the `ECONNREFUSED` because Kubernetes will not keep forwarding traffic to an instance that has been `OOMKilled`. 
- There were no restarts on the API server replicas since the last deployment, no sign to `OOMKill` or other kinds of restarts.

The next thing we looked at is the monitoring graphs (using Grafana). In the memory usage graphs of the API server we could clearly see the "half pyramid" shapes that indicated that there's a memory leak in the server. The reason it didn't trigger any alert was that the memory never reached the Kubernetes limits but V8's limits.

At this moment we understood that the error flow was as follows:
- The API server was leaking memory.
- It wasn't stopped by Kubernetes because of Node.js Garbage Collector that handled the leakage before it reached Kubernetes's limit.
- That's why we got the `ECONNRESET` errors when Node.js was trying to unsuccessfully reclaim memory.
- The reason there were `ECONNREFUSED` is because Kubernetes wasn't aware of the memory leak and kept routing traffic even when the API server was too busy reclaiming memory and restarting the application.

# Solving the problem

Since two microservices were involved in the issue, two different teams will have to address it, the `client application team` is the team that maintains the client application that made a request against the API server, maintained by the `server application team`.

## Client application team
> Fault tolerance, fault tolerance, fault tolerance.
- The client application uses [axios](https://axios-http.com/) for making the HTTP requests, so we added a plugin named [`axios-retry`](https://github.com/softonic/axios-retry) that intercepts failed requests and retries them whenever possible.
- We also adjusted the `timeout` option used in `axios` for failing faster in similar situations.

Another good option can be following the [`Circuit Breaker`](https://martinfowler.com/bliki/CircuitBreaker.html) pattern, it can help handle the error from the API server if it's likely to fail.


## Server application team
- Adjusted V8's memory limit to the actual usage. Using the `--max-old-space-size` flag.
- Increase the replication factor to distribute the load and mitigate the pressure on the current instances.
- Find and fix the memory leak (Taking V8 snapshots, etc...)


Until the next bug...