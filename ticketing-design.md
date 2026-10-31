# TicketHub: Event Ticketing System Design

## 1. System Requirements

TicketHub is an online event ticketing platform that allows customers to discover events, select seats, reserve tickets and make payments. It is designed to support both everyday ticket purchases and high-demand concert releases.

### Functional Requirements

The system should allow customers to:

1. Register accounts and log in securely.
2. Browse, search and filter upcoming events.
3. View event details, venues, dates and ticket prices.
4. View available, temporarily held and sold seats.
5. Join a virtual waiting room during popular ticket sales.
6. Temporarily reserve selected seats while completing checkout.
7. Pay for tickets using supported payment methods.
8. Receive booking confirmations and digital tickets with unique QR codes.
9. View their purchased tickets and booking history.
10. Cancel eligible bookings according to the event's cancellation policy.

### Non-Functional Requirements

- Performance: Event pages should load within approximately two seconds under normal conditions.
- Scalability: The platform should handle sudden increases in traffic without becoming unavailable.
- Correctness: A seat must never be confirmed as sold to more than one customer.
- Security: User accounts and payment information must be protected through authentication, authorization and HTTPS.
- Fairness: A virtual waiting room should control access to popular sales and prevent automated scripts from bypassing the queue.
- Reliability: The system should continue operating if an individual application server fails.
- Usability: The booking process should work smoothly on mobile phones and computers.

## 2. Traffic and Capacity Estimates

TicketHub has 2 million registered users. The system must support normal daily activity as well as major ticket releases that attract large numbers of customers.

### Normal Daily Traffic

Given:

- Registered users: 2,000,000
- Daily visitors: 50,000
- Pages viewed per visitor: 10
- Tickets sold daily: 5,000

Daily page views:

50,000 × 10 = 500,000 page views per day.

Average page requests per second:

500,000 ÷ 86,400 = approximately 5.8 requests per second.

Average ticket sales per second:

5,000 ÷ 86,400 = approximately 0.058 tickets per second.

This represents relatively steady browsing activity, with purchases happening at a much lower rate than page views.

### Popular Concert Sale

For a major concert release:

- People attempting to purchase tickets: 200,000
- Available seats: 20,000
- Sale period: 10 minutes or 600 seconds.

Purchase attempts per second:

200,000 ÷ 600 = approximately 333 attempts per second.

Percentage of customers who can obtain a seat:

20,000 ÷ 200,000 × 100 = 10%.

Only one out of every ten people attempting to purchase a ticket can obtain one if everyone is competing for a single seat.

### Comparing Normal and Peak Traffic

On a normal day, TicketHub receives approximately 5.8 page requests per second. During the popular concert sale, it must manage approximately 333 purchase attempts per second.

This represents a major increase in demand within a short period. The system must therefore control incoming traffic, reduce unnecessary database requests and process bookings carefully.

These figures assume requests are evenly distributed. In practice, customers may arrive in sudden bursts, creating even greater pressure.

## 3. API Design

TicketHub uses a REST API to connect the frontend application with the backend services. The API uses JSON for sending and receiving information and HTTPS to secure communication.

Base URL: https://api.tickethub.example/v1

The main API endpoints are:

1. GET /events - Retrieves upcoming events and supports searching and filtering. Returns 200 OK.

2. GET /events/{eventId} - Retrieves details of a selected event, including its venue and date. Returns 200 OK.

3. GET /events/{eventId}/seats - Retrieves available, held and sold seats for an event. Returns 200 OK.

4. POST /events/{eventId}/holds - Temporarily holds selected seats for a customer. Returns 201 Created.

5. POST /orders/{orderId}/payments - Submits payment for an existing order. Returns 200 OK when payment is successfully verified.

6. GET /users/me/tickets - Retrieves tickets belonging to the authenticated customer. Returns 200 OK.

7. DELETE /holds/{holdId} - Releases a temporary seat reservation. Returns 204 No Content.

### Example: Requesting Available Seats

Request:

GET /events/205/seats

Example response:

{
  "eventId": 205,
  "currency": "KES",
  "seats": [
    {
      "seatId": 501,
      "seatNumber": "A12",
      "price": 3500,
      "status": "available"
    },
    {
      "seatId": 502,
      "seatNumber": "A13",
      "price": 3500,
      "status": "held"
    },
    {
      "seatId": 503,
      "seatNumber": "A14",
      "price": 3500,
      "status": "sold"
    }
  ]
}

### Example: Holding a Seat

Request:

POST /events/205/holds

Request body:

{
  "seatIds": [501],
  "queueToken": "valid-queue-token"
}

Example successful response:

{
  "holdId": 901,
  "eventId": 205,
  "seatIds": [501],
  "status": "held",
  "expiresAt": "2026-11-15T14:10:00Z"
}

The queue token confirms that the customer has entered the booking process through the virtual waiting room. The temporary hold gives the customer five minutes to complete payment. If the hold expires without successful payment, the seat is released.

### API Error Handling

The API should return appropriate HTTP status codes when a request cannot be completed.

- 400 Bad Request: The customer submits an invalid seat selection.
- 401 Unauthorized: The customer needs to authenticate.
- 403 Forbidden: The customer attempts to access an order they do not own.
- 404 Not Found: The requested event or seat does not exist.
- 409 Conflict: The selected seat has already been held or sold.
- 500 Internal Server Error: An unexpected server error occurs.

Example error response:

{
  "error": {
    "code": "SEAT_UNAVAILABLE",
    "message": "This seat is no longer available. Please select another seat."
  }
}

## 4. Database Design

TicketHub uses PostgreSQL, a relational database, because ticket bookings involve structured information and relationships that require accurate transactions.

The database contains five main tables.

### 4.1 Users Table

Stores information about registered customers.

- user_id: INT, primary key.
- full_name: VARCHAR(100), customer's full name.
- email: VARCHAR(150), unique email address.
- password_hash: VARCHAR(255), securely stored password hash.
- created_at: TIMESTAMP, account creation date.

### 4.2 Events Table

Stores information about concerts and other events.

- event_id: INT, primary key.
- event_name: VARCHAR(150), name of the event.
- venue: VARCHAR(150), event location.
- event_date: TIMESTAMP, date and time of the event.
- sale_start: TIMESTAMP, time ticket sales begin.
- status: VARCHAR(20), event status such as scheduled, on_sale or completed.

### 4.3 Seats Table

Stores the seats available for each event.

- seat_id: INT, primary key.
- event_id: INT, foreign key referencing Events.
- seat_number: VARCHAR(20), identifies the seat.
- price: DECIMAL(10,2), ticket price.
- status: VARCHAR(20), available, held or sold.
- held_by: INT, foreign key referencing Users when a seat is temporarily held.
- hold_expires_at: TIMESTAMP, time when the temporary hold expires.

A unique constraint on event_id and seat_number ensures that the same seat cannot be registered twice for one event.

### 4.4 Orders Table

Stores customer booking information.

- order_id: INT, primary key.
- user_id: INT, foreign key referencing Users.
- order_date: TIMESTAMP, date the order was created.
- total_amount: DECIMAL(10,2), total booking amount.
- status: VARCHAR(20), pending, paid, failed or expired.
- payment_reference: VARCHAR(100), unique payment provider reference.

### 4.5 Order_Items Table

Stores the individual seats included in each order.

- order_item_id: INT, primary key.
- order_id: INT, foreign key referencing Orders.
- seat_id: INT, foreign key referencing Seats.
- ticket_price: DECIMAL(10,2), price of the seat at the time of purchase.

A unique constraint on seat_id in active booking records prevents the same seat from being assigned to multiple active orders. Expired and cancelled bookings must be released correctly so that seats can be purchased again.

### Relationships Between Tables

- One user can create many orders. This is a one-to-many relationship.
- One event can contain many seats. This is a one-to-many relationship.
- One order can contain many order items. This is a one-to-many relationship.
- Order_Items connects Orders and Seats, allowing a customer to purchase multiple seats in one order.

These relationships help TicketHub maintain organized booking records and enforce data consistency.

## 5. Preventing Double-Booking

Double-booking occurs when two or more customers successfully purchase the same seat. This is a major risk during popular concert releases because many customers may select the same seat almost simultaneously.

TicketHub prevents this using database transactions, row-level locking and conditional updates.

### How Seat Reservation Works

1. A customer selects a seat and sends a request to temporarily reserve it.
2. The backend starts a database transaction.
3. The database locks the selected seat row using SELECT FOR UPDATE. This prevents another transaction from modifying that same seat while the first transaction is processing.
4. The system checks whether the seat is available or whether an earlier temporary hold has expired.
5. If available, the system changes the seat status to held and records the customer and hold expiration time.
6. If the seat is already held or sold, the request is rejected with a 409 Conflict response.
7. The transaction is committed, and the customer receives confirmation of the temporary hold.
8. After payment is successfully verified, the order is marked as paid and the seat status changes to sold.
9. If payment fails or the hold expires, the seat is released and becomes available again.

### Example of the Database Operation

The following simplified PostgreSQL statements show how a seat can be locked and conditionally updated:

BEGIN;

SELECT seat_id, status, hold_expires_at
FROM seats
WHERE seat_id = 501
FOR UPDATE;

UPDATE seats
SET status = 'held',
    held_by = 25,
    hold_expires_at = CURRENT_TIMESTAMP + INTERVAL '5 minutes'
WHERE seat_id = 501
AND (
    status = 'available'
    OR (
        status = 'held'
        AND hold_expires_at < CURRENT_TIMESTAMP
    )
);

COMMIT;

The application must check whether the UPDATE affected exactly one row. If no row was updated, the seat was not available and the customer must receive an error instead of a successful hold.

For bookings containing multiple seats, all selected seats should be reserved within one transaction. If any requested seat is unavailable, the transaction should be rolled back so that the customer does not receive only part of the requested booking.

The database transaction ensures that competing requests cannot both successfully reserve the same seat. The unique constraints provide an additional layer of protection against duplicate assignments.

Payment should be verified with the payment provider before the order is confirmed. Unique payment references and idempotency checks also help prevent duplicate charges when payment requests are retried.

This approach makes the database responsible for the final booking decision instead of relying on the seat information displayed in the customer's browser, which may become outdated.

## 6. System Architecture

TicketHub uses a multi-layer architecture to separate browsing, seat booking, payment processing and background tasks. This helps the platform remain responsive during sudden increases in traffic.

### Architecture Diagram

CUSTOMERS
↓
DNS SERVICE
↓
CDN / EDGE CACHE
↓
LOAD BALANCER
↓
VIRTUAL WAITING ROOM
↓
APPLICATION SERVERS (SERVER 1 AND SERVER 2)
↓
REDIS CACHE
↓
BOOKING SERVICE
↓
PRIMARY DATABASE
↓
READ REPLICA

APPLICATION SERVERS
↓
MESSAGE QUEUE
↓
BACKGROUND WORKERS
↓
EMAIL SERVICE AND DIGITAL TICKET GENERATION

### Component Explanations

- Customers: Use mobile phones or computers to browse events and purchase tickets.
- DNS Service: Directs customers to the TicketHub platform through its domain name.
- CDN / Edge Cache: Delivers static resources such as images, stylesheets and scripts from locations closer to customers.
- Load Balancer: Distributes incoming requests across multiple application servers to prevent one server from handling all the traffic.
- Virtual Waiting Room: Controls entry into popular ticket sales and manages customer queue positions to reduce sudden pressure on the booking system.
- Application Servers: Handle user requests, authentication, event browsing and communication with backend services.
- Redis Cache: Temporarily stores frequently requested information such as event details to reduce unnecessary database reads.
- Booking Service: Manages seat holds, order creation and booking rules while checking seat availability before accepting reservations.
- Primary Database: Stores the main user, event, seat and order records and processes booking transactions.
- Read Replica: Handles suitable read requests to reduce the workload on the primary database, while final booking decisions are checked against the primary database.
- Message Queue: Stores background tasks so that activities such as sending emails do not delay customer checkout.
- Background Workers: Process queued tasks, including generating digital tickets and sending booking confirmations.
- Email Service: Sends customers booking confirmations and ticket notifications.
- Digital Ticket Generation: Creates tickets containing unique references or QR codes that can be verified at the event entrance.

### How the Architecture Handles a Popular Sale

When a popular concert goes on sale, many customers may visit the platform simultaneously. The CDN handles static resources while the virtual waiting room controls how many customers enter the booking process at a time.

The load balancer distributes admitted requests across the available application servers. Redis handles frequently requested event information, reducing repeated queries to the database.

When customers attempt to reserve seats, the booking service communicates with the primary database. Transactions and row-level locking ensure that seats cannot be reserved by multiple customers at the same time.

After payment is verified, the system sends ticket generation and notification tasks to the message queue. Background workers process these tasks separately, allowing the customer-facing application to remain responsive.

Additional application servers can be added when traffic increases. If one server fails, the load balancer directs requests to the remaining healthy servers. Database backups, replication and monitoring also support recovery and reliability.

## 7. Design Trade-Offs

### 7.1 Fairness Versus Speed

A virtual waiting room helps organize high-demand ticket releases and prevents the booking system from receiving too many requests simultaneously. However, customers may have to wait before accessing the booking page.

This creates a trade-off between providing immediate access and maintaining fair, controlled access. TicketHub prioritizes fairness and system stability during popular releases.

### 7.2 Strong Consistency Versus Performance

Database transactions and row-level locking prevent two customers from purchasing the same seat. However, they may increase response times when many customers compete for the same seats.

TicketHub prioritizes booking correctness because selling one seat to multiple customers would damage customer trust. Caching is used to improve browsing performance, while final seat availability is always verified against the primary database.

### 7.3 Automatic Scaling Versus Cost

Adding more application servers allows TicketHub to handle sudden increases in traffic. However, additional servers and infrastructure increase operating costs.

The platform can use automatic scaling to add capacity during major ticket releases and reduce it when demand returns to normal. This balances performance with infrastructure expenses.

## Conclusion

TicketHub is designed to provide a reliable and fair ticket purchasing experience during both normal activity and major concert releases. Its use of a virtual waiting room, scalable application servers, caching, database transactions and background processing helps manage high traffic without compromising booking accuracy.

Most importantly, seat availability is verified and controlled by the database, ensuring that only one customer can successfully reserve and purchase a particular seat.