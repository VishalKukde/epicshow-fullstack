# EpicShow – Technical Project Breakdown
Link : https://epicshow.vercel.app/

## Section 1: Context (Brief)

EpicShow is a scalable full-stack ticket booking platform designed to provide users with a seamless movie and event booking experience through real-time seat selection, secure authentication, wallet-based transactions, and responsive cross-device accessibility. The platform was built to simulate production-grade booking systems where multiple users may attempt to reserve the same seats simultaneously, requiring low-latency communication, strong concurrency handling, and reliable transaction consistency. The system supports dynamic show scheduling, real-time seat locking, booking management, notifications, refund workflows, and admin-level theater management while maintaining a modern and optimized user experience.

The primary technical constraints during development included handling concurrent seat bookings without double allocation, maintaining low response times during high-traffic booking operations, ensuring consistency between cached and database states, implementing real-time updates across clients, and designing scalable APIs capable of supporting future microservice decomposition. Additional constraints included deployment optimization across multiple cloud platforms, minimizing unnecessary database reads, maintaining secure authentication flows, and ensuring responsive UI rendering for complex seat layouts and booking interactions.

---

# Section 2: Technical Implementation (Detailed)

## System Architecture

```text
 ┌─────────────────────┐
 │     Client UI       │
 │  Next.js + React    │
 └─────────┬───────────┘
           │ REST APIs / WebSocket
           ▼
 ┌─────────────────────┐
 │   Node.js Backend   │
 │ Express.js Server   │
 └─────────┬───────────┘
           │
 ┌─────────┴──────────────┐
 │                        │
 ▼                        ▼
Redis Cache          MongoDB Database
Seat Locks           Persistent Storage
Pub/Sub              Bookings / Users
Caching              Shows / Payments
```

The frontend was developed using Next.js and communicates with the backend through RESTful APIs and WebSocket connections for real-time interactions. The backend, built with Node.js and Express.js, handles authentication, booking logic, payment workflows, and concurrency control while integrating Redis for caching, distributed locking, and pub/sub communication, alongside MongoDB for persistent storage.

---

# Critical Function Walk-through – Seat Locking System

One of the most critical backend functions in EpicShow is the seat-locking mechanism responsible for preventing double bookings when multiple users attempt to reserve the same seats simultaneously.

### Core Objective

Ensure that:

* A seat can only be temporarily reserved by one user at a time
* Locks expire automatically after a timeout
* Race conditions are prevented during concurrent booking attempts
```

### Technical Breakdown

1. A unique Redis key is generated using the show ID and seat ID.
2. Redis checks whether the seat is already locked.
3. The `NX` flag ensures the key is only created if it does not already exist.
4. The `EX` expiration automatically releases the lock after a fixed timeout.
5. WebSocket events notify connected users about seat availability updates in real time.

This approach significantly reduced booking conflicts while keeping the booking experience responsive under concurrent load conditions.

---

# Data Flow – Ticket Booking Operation

## Step-by-Step Booking Workflow

```text
User Selects Seats
        │
        ▼
Frontend Sends Lock Request
        │
        ▼
Backend Checks Redis Lock
        │
 ┌──────┴─────────┐
 │ Seat Available │
 └──────┬─────────┘
        ▼
Seat Temporarily Locked
        │
        ▼
WebSocket Broadcast Updates
        │
        ▼
User Completes Payment
        │
        ▼
Booking Persisted in MongoDB
        │
        ▼
Seat Marked as Booked
        │
        ▼
Redis Lock Removed
```

### Operational Explanation

When a user selects seats, the frontend immediately requests temporary seat locks through the backend API. Redis acts as the first layer of concurrency protection by validating whether the seats are already reserved. Once locked, WebSocket events instantly notify other users about seat status changes to maintain UI synchronization across clients.

After payment confirmation, booking details are persisted in MongoDB and the seat status transitions from “locked” to “booked.” If the payment fails or the lock expires, Redis automatically clears the temporary reservation, allowing other users to reserve those seats.

---

# Section 3: Technical Decisions (Core)

## Technology Choice 1 – Redis for Distributed Seat Locking

### Decision

Redis was selected as the core mechanism for temporary seat locking and real-time synchronization.

### Why It Was Chosen

Traditional database-based locking approaches introduced higher latency and increased the risk of race conditions under concurrent requests. Redis provided atomic operations, expiration-based locks, and significantly lower response times for high-frequency booking operations.

### Trade-offs

| Advantage                             | Trade-off                             |
| ------------------------------------- | ------------------------------------- |
| Extremely fast in-memory operations   | Increased infrastructure complexity   |
| Native expiration support             | Additional operational overhead       |
| Atomic lock handling                  | Requires cache consistency management |
| Pub/Sub support for real-time updates | Data is non-persistent by default     |

### Result

Redis dramatically improved booking responsiveness and reduced seat conflict issues during concurrent booking simulations.

---

## Technology Choice 2 – Next.js for Frontend Architecture

### Decision

The frontend was built using Next.js to support optimized rendering, modular UI architecture, and production-ready deployment workflows.

### Why It Was Chosen

The project required strong SEO support, optimized routing, component scalability, and excellent developer experience. Next.js provided server-side rendering capabilities, efficient page-based routing, and optimized performance handling out of the box.

### Trade-offs

| Advantage                         | Trade-off                                    |
| --------------------------------- | -------------------------------------------- |
| Optimized rendering performance   | Higher learning curve                        |
| Built-in routing system           | More opinionated structure                   |
| Strong deployment integration     | SSR complexity in some components            |
| Excellent React ecosystem support | Increased bundle optimization responsibility |

### Result

Next.js improved frontend maintainability, responsiveness, and deployment simplicity while enabling scalable UI architecture for future feature additions.

---

# Scaling Bottleneck & Mitigation Strategy

## Bottleneck

The largest scalability bottleneck identified was the high frequency of seat availability reads and writes during peak booking operations. Repeated database queries for seat states introduced increased latency and unnecessary database load.

## Mitigation Strategy

To address this:

* Redis caching was introduced for frequently accessed seat data
* Temporary seat states were managed in-memory before persistence
* WebSocket broadcasts minimized repeated polling requests
* Database indexing was optimized for show and booking lookups
* Backend APIs were designed to minimize redundant database operations

### Future Scaling Plan

If traffic significantly increases, the system can evolve toward:

* Microservice-based booking architecture
* Dedicated booking queue processing
* Horizontal scaling with container orchestration
* Event-driven communication using message brokers like Kafka or RabbitMQ

---

# Section 4: Learning & Iteration (Concise)

## Technical Mistake & Learning

One of the major technical mistakes during early development was initially handling seat reservation logic directly through MongoDB updates without implementing distributed locking. Under concurrent booking simulations, this occasionally resulted in race conditions where multiple users could temporarily reserve the same seat before database synchronization completed.

This experience highlighted the importance of concurrency management in real-time systems and taught me how critical distributed locking, atomic operations, and caching layers are when building scalable transactional applications. It also improved my understanding of system consistency, temporary state management, and real-time architecture design.

---

## What I Would Do Differently Today

If rebuilding EpicShow today, I would architect the system using a more service-oriented approach from the beginning instead of keeping all business logic inside a centralized backend application. I would separate authentication, booking, notifications, and payment processing into independently scalable services with asynchronous event-driven communication.

I would also implement:

* Centralized logging and monitoring
* Queue-based booking processing
* Automated load testing pipelines
* Kubernetes-based orchestration
* Better observability using tracing and metrics
* Role-based infrastructure environments for staging and production

This would improve maintainability, scalability, deployment flexibility, and operational reliability for handling production-scale traffic.
