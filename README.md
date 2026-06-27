# Over-Engineering My Personal Platform (Because Why Not?)

I could have thrown together a basic template for this portfolio and called it a day. But where is the fun in that? 

I wanted a complete personal ecosystem—built with an extensible, bulletproof foundation ready to ingest whatever chaotic backend ideas I dream up in the future. The portfolio you are looking at right now isn't the whole system; it is just Tenant #1. 

This architecture didn't appear overnight; it evolved through constant iteration. I originally started with a monolithic Python backend. I eventually rewrote it into a Go REST API for two reasons: I wanted to learn Go, and I wanted to eliminate the heavy runtime overhead of Python. On the frontend, my administrative dashboards went through two distinct iterations using JavaScript and Bun as I refined the UI patterns. 

While those monolithic REST APIs worked fine for a simple website, they presented a structural roadblock: if I wanted to build a completely separate project (like a financial ledger or a custom analytics engine), I would have to recreate authentication, rate-limiting, and core boilerplate all over again. I wanted to genuinely learn how to build distributed systems, and my personal platform was the perfect playground to do it.

So, I abandoned the monolithic REST approach entirely and built a strictly-typed microservice platform from scratch—extracting authentication and routing so any future service can plug into the grid for free. Here is how I wired together Go, gRPC, React 19, and a slightly paranoid PostgreSQL setup to make it happen.

---

## The Grand Plan

The architecture relies on absolute isolation. The frontends talk exclusively to a central API perimeter, which handles the dirty work before passing clean, binary payloads down to the internal backend services.

![Architecture Diagram](https://raw.githubusercontent.com/Aditya-0011/Architecture/refs/heads/main/diagram.svg)

---

## 1. The Traffic Cop: Fiber API Gateway

I built the API gateway using Fiber, a ridiculously fast web framework for Go, to act as the primary entry point for all external traffic. 

The gateway executes exactly **zero** business logic. Its entire job is to keep the noise out. It intercepts incoming traffic, handles rate-limiting via Redis to stop script kiddies from spamming my endpoints, and manages two distinct authentication lanes:
* **The Admin Dashboards:** Authenticate via secure, Redis-backed session cookies.
* **The Public Portfolio:** Communicates via long-lived API keys for secure, server-to-server communication.

Once a request passes the gauntlet, the gateway handles the JSON-to-Protobuf translation and fires a lightweight binary payload into the internal gRPC cluster.

---

## 2. Internal Contracts via gRPC & Buf

When you split a monolith into microservices, keeping internal communication in sync across different codebases can quickly turn into a nightmare of broken endpoints and mismatched JSON. To kill this friction forever, the API gateway and internal backend services communicate strictly over gRPC using Protocol Buffers (`.proto`).

I manage all schemas centrally using **Buf**. It acts as the ultimate gatekeeper, handling linting and compiling the exact Go structs my services expect. To catch bad data before it even hints at touching my application logic, I use `protovalidate` to bake validation constraints directly into the raw `.proto` definitions. 

Right now, the cluster runs two isolated microservices, but the grid is designed to ingest more:
* **Auth Service:** The central identity provider. It handles SSO (Single Sign-On) for the entire ecosystem and issues/rotates API keys.
* **Manager Service (Tenant #1):** The headless CMS. It drives the CRUD operations for my work experiences, portfolio projects, and incoming contact messages.
* **Future Services:** Because the Gateway and Auth layers are completely decoupled, I can drop a new service—like a Wallet ledger or an analytics engine—directly into the grid without touching the existing architecture.

---

## 3. Escaping the Denormalization Trap (Moving to PostgreSQL)

As the first real-world stress test of this platform, I migrated my public-facing website from a legacy MongoDB cluster to this new architecture. 

MongoDB was great for fast document reads, but it introduced a structural flaw: the denormalization trap. To avoid slow joins, MongoDB encourages embedding documents inside other documents. But when a piece of data changes, that update has to be manually propagated across multiple documents. 

Yes, modern MongoDB supports multi-document transactions to prevent data fracturing during network hiccups. But any experienced engineer knows that heavily relying on Mongo transactions is an architectural anti-pattern. They introduce massive lock contention and performance degradation because you are forcing a NoSQL database to behave like a relational one. It is a hellhole I had no interest in managing.

Moving to PostgreSQL solved this immediately. Relational normalization means updating a single row propagates the change everywhere, instantly and safely, with native ACID compliance that doesn't tank system performance. 

It also allowed me to push the data processing closest to the data itself. Instead of making multiple database calls, dragging raw data across the network, and processing it in the application layer (or writing massive, unmaintainable MongoDB aggregation pipelines), I can execute a single SQL function directly on the server to get exactly what I need.

### Paranoid Database Security
To keep the microservices genuinely decoupled, the Auth and Manager services run on entirely separate PostgreSQL databases. Then, I took the security model to a slightly paranoid extreme. 

The Go services connect to PostgreSQL using heavily restricted application roles. I explicitly **revoked `INSERT`, `UPDATE`, and `DELETE` permissions** from the tables. 

> **The Security Philosophy:** If the Go service is compromised, a malicious actor still cannot run arbitrary write queries or drop tables. 

Instead, all mutations are forced to route through tightly controlled, pre-compiled **PL/pgSQL stored procedures**. The Go application can only call specific execution paths. If it tries to write outside the lines, the database itself violently rejects it.

### The Single-Tenant Constraint
You might look at the Auth service schema and think you found a bug: the user ID is hardcoded to `1`. 

That is by design. This is a personal platform. Allowing open registration introduces a massive attack surface and demands a mountain of verification overhead. To maintain a zero-waste resource footprint and physically guarantee that no rogue accounts can ever be created, the schema rejects anyone else at the database level. It is a strictly single-tenant system.

---

## 4. The Presentation Layer

The client infrastructure consists of three distinct React applications. To mirror the backend's separation of concerns, the administrative tools are cleanly split by domain:

* **Console UI:** The identity and operations dashboard. It acts as the SSO hub, routing requests through the gateway to the Auth service to manage active login sessions and handle API key rotation. 
* **Manager UI:** The dedicated CMS dashboard. It handles the CRUD operations for the portfolio data once the gateway verifies the SSO session.

Both admin interfaces are built for absolute speed: **React 19, Vite, Bun, and React Router v8**, with `@tanstack/react-query` handling asynchronous caching flawlessly.

---

## What Comes Next?

Building a multi-language gRPC microservice cluster just to serve a portfolio website is the literal definition of over-engineering. 

But this portfolio was never the end goal; it was just the proof of concept. By drawing hard boundaries between the REST perimeter, the gRPC transit layers, and the locked-down PostgreSQL storage, I now have an enterprise-grade backend ecosystem running on micro-resources. 

The chassis is built. Now it's time to start dropping new services onto the grid.

---

## The Source Code

The architecture is entirely open-source. You can explore the exact implementations, gRPC contracts, and deployment setups across the cluster repositories:

**Infrastructure & Contracts**
* [`common`](https://github.com/Aditya-0011/common) - The centralized Protocol Buffer contracts, generated Go/C# structs, and shared middleware.
* [`gateway`](https://github.com/Aditya-0011/gateway) - The Fiber-based API perimeter, rate limiter, and gRPC translator.

**Backend Microservices**
* [`auth-service`](https://github.com/Aditya-0011/auth-service) - The identity provider, SSO hub, and API key manager.
* [`manager-service`](https://github.com/Aditya-0011/manager-service) - The headless CMS backend powering the portfolio data.

**Client Applications**
* [`portfolio-website`](https://github.com/Aditya-0011/portfolio-website) - The source code for this exact Next.js portfolio website, acting as Tenant #1.
* [`console-ui`](https://github.com/Aditya-0011/console-ui) - The identity and systems operations dashboard.
* [`manager-ui`](https://github.com/Aditya-0011/manager-ui) - The dedicated CMS content dashboard.
