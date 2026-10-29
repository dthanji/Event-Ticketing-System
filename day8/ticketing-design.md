TicketHub System Design
1. Requirements
Functional Requirements
Users can register and log in.
Users can browse concerts and events.
Users can view available seats.
Users can temporarily hold seats before purchasing.
Users can purchase tickets.
Users can view purchased tickets.
Event organizers can create and manage events.
The system must prevent double-booking of seats.
Non-Functional Requirements
Performance
Event pages should load within 2 seconds.
Seat availability should be updated in near real-time.
The system should handle sudden spikes in traffic.
Correctness
A seat can only be sold once.
Payment and ticket issuance must be consistent.
All transactions should be atomic.
Fairness
Requests should be processed in the order they are received.
During major ticket releases, a virtual waiting room should ensure fair access.
Availability
The service should aim for 99.9% uptime.
Scalability
The platform must support both normal traffic and large traffic surges.
2. Traffic Estimates
Normal Traffic
Registered users: 2,000,000
Daily visitors: 50,000
Pages viewed per visitor: 10
Total daily page views:

50,000 × 10 = 500,000 page views/day

Average requests per second:

500,000 ÷ 86,400 ≈ 5.8 requests/sec

Approximately 6 requests per second.

Daily ticket sales:

5,000 tickets/day

Popular Concert Sale
200,000 users attempt to purchase tickets
20,000 seats available
Sale occurs within first 10 minutes
10 minutes = 600 seconds

Purchase attempts:

200,000 ÷ 600 ≈ 333 requests/sec

Assuming each user refreshes the seat map several times, seat-related traffic could exceed:

333 × 5 ≈ 1,665 requests/sec

Comparison
Metric	Normal Day	Popular Sale
Visitors	50,000/day	200,000 in 10 minutes
Requests/sec	~6	~333
Seat Requests/sec	Low	~1,665
Tickets Sold	5,000/day	20,000 in minutes
3. API Design
Browse Events
GET /api/events

Event Details
GET /api/events/{eventId}

View Seats
GET /api/events/{eventId}/seats

Hold Seats
POST /api/seats/hold

Example request:

{
  "eventId": 1001,
  "seatIds": [101,102]
}
Purchase Tickets
POST /api/orders

View Purchased Tickets
GET /api/users/{userId}/tickets

4. Data Model
Users
Column	Type
user_id	PK
name	VARCHAR
email	VARCHAR
created_at	DATETIME
Events
Column	Type
event_id	PK
title	VARCHAR
venue	VARCHAR
event_date	DATETIME
Seats
Column	Type
seat_id	PK
event_id	FK
section	VARCHAR
row_number	VARCHAR
seat_number	VARCHAR
status	AVAILABLE, HELD, SOLD
Orders
Column	Type
order_id	PK
user_id	FK
created_at	DATETIME
payment_status	VARCHAR
Order_Seats
Column	Type
order_id	FK
seat_id	FK
Relationships
Users 1 ---- * Orders

Events 1 ---- * Seats

Orders * ---- * Seats (through Order_Seats)

5. Preventing Double-Booking
Preventing double-booking is the most important requirement.

When a user selects a seat:

Start a database transaction.
Lock the seat row.
Check whether the seat is AVAILABLE.
If available, change its status to HELD.
Commit the transaction.
Example:

SELECT *
FROM Seats
WHERE seat_id = 101
FOR UPDATE;
The row-level lock prevents another transaction from updating the same seat simultaneously.

When payment is completed:

Begin transaction.
Verify the hold has not expired.
Change seat status from HELD to SOLD.
Create the order record.
Commit transaction.
A unique constraint on seat ownership ensures a seat can only be attached to one completed order.

This guarantees that two users cannot purchase the same seat.

6. Architecture
                   +----------------+
                   |     Users      |
                   +-------+--------+
                           |
                           v
                +-------------------+
                | Load Balancer     |
                +---------+---------+
                          |
          +---------------+---------------+
          |                               |
          v                               v
+-------------------+          +-------------------+
| Web/API Server 1  |          | Web/API Server 2  |
+---------+---------+          +---------+---------+
          |                              |
          +--------------+---------------+
                         |
                         v
                +-------------------+
                | Redis Cache       |
                +---------+---------+
                          |
                          v
                +-------------------+
                | Queue Service     |
                +---------+---------+
                          |
                          v
                +-------------------+
                | SQL Database      |
                +-------------------+
Component Explanation
Load Balancer distributes traffic across servers.
Web/API Servers process requests.
Redis Cache stores frequently requested data.
Queue Service handles high-demand ticket sales fairly.
SQL Database stores users, seats, events and orders.
Handling the Big Sale
When 200,000 users attempt to buy tickets:

Users enter a waiting room.
Requests are queued.
Event information is served from cache.
More application servers can be added.
Database transactions protect seat inventory.
7. Trade-Offs
Trade-Off 1: Strong Consistency vs Speed
Benefit:

Prevents double-booking.
Cost:

Slightly slower transactions.
Trade-Off 2: Queue System vs User Experience
Benefit:

Protects the system during traffic spikes.
Cost:

Users may wait before entering checkout.
Conclusion
TicketHub uses transactions, row-level locking, seat holds, and database constraints to ensure seats are never sold twice. Load balancing, caching, and queues allow the system to scale during major ticket releases.
