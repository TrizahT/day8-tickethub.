# TicketHub: Event Ticketing System Design

## 1. Requirements

TicketHub is an online event ticketing platform where users can discover concerts and other events, view available seats, reserve tickets and complete payments. The system should provide a smooth booking experience while remaining reliable during high-demand ticket releases.

### Functional Requirements

The system should allow users to:

1. Create accounts and log in securely.
2. Browse upcoming concerts and events.
3. Search and filter events by name, date, location and category.
4. View event details, ticket prices and available seats.
5. Join a virtual waiting room when a popular event goes on sale.
6. Temporarily hold selected seats while completing checkout.
7. Pay for tickets using supported payment methods.
8. Receive booking confirmation and a digital ticket after successful payment.
9. View their purchased tickets and booking history.
10. Cancel bookings where the event's cancellation policy allows it.

### Non-Functional Requirements

- **Performance:** Event pages and seat information should load quickly, ideally within two seconds under normal conditions.
- **Scalability:** The system should handle sudden increases in traffic during popular ticket releases without becoming unavailable.
- **Reliability:** Users should be able to browse events and complete bookings with minimal service interruptions.
- **Correctness:** A seat must never be confirmed as sold to more than one customer.
- **Security:** User accounts, personal information and payment transactions must be protected through authentication, authorization and encrypted communication.
- **Fairness:** A virtual waiting room should manage high-demand releases fairly, using a clearly communicated queue process rather than allowing automated requests to overwhelm the booking system.
- **Availability:** The system should remain accessible even when individual application servers fail.
- **Usability:** The booking process should be simple, responsive and accessible on both mobile devices and desktop computers.

## 2. Traffic and Storage Estimates

TicketHub has 2 million registered users. The system must support ordinary daily activity as well as unusually high demand when popular concerts go on sale.

### A. Normal Traffic

Given:

- Registered users: 2,000,000
- Daily visitors: 50,000
- Pages viewed per visitor: 10
- Tickets sold per day: 5,000

**Daily page views**

50,000 × 10 = 500,000 page views per day.

**Average page requests per second**

500,000 ÷ 86,400 = approximately 5.8 requests per second.

**Average ticket sales per second**

5,000 ÷ 86,400 = approximately 0.058 tickets per second.

This means the platform handles a moderate amount of browsing activity on a normal day, with ticket purchases happening less frequently than page views.

### B. Big Concert Sale

When a popular concert goes on sale:

- People attempting to purchase tickets: 200,000
- Available seats: 20,000
- Time available: 10 minutes (600 seconds)

**Purchase attempts per second**

200,000 ÷ 600 = approximately 333 purchase attempts per second.

**Ticket availability**

20,000 ÷ 200,000 × 100 = 10%.

Only 10% of the people attempting to purchase tickets can obtain a seat if every person is competing for one ticket.

The system must therefore handle approximately 333 purchase attempts per second while protecting the limited number of available seats.

### C. Comparison

| Measurement | Normal day | Popular concert sale |
|---|---:|---:|
| Visitors or purchase attempts | 50,000 visitors/day | 200,000 attempts/10 min |
| Page views | 500,000/day | Depends on browsing activity |
| Average relevant request rate | 5.8 page requests/sec | 333 purchase attempts/sec |
| Ticket availability | 5,000 sold/day | 20,000 seats |
| Main system concern | Consistent performance | Traffic spikes and booking accuracy |

The big sale creates a much higher demand in a very short period. TicketHub should therefore use caching, a virtual waiting room and additional application capacity to manage the traffic. The estimates assume requests are evenly distributed over the stated periods; actual traffic may arrive in much larger bursts.

## 3. API Design

TicketHub uses a REST API to allow the frontend application to communicate with the backend. The API uses JSON for data exchange and HTTPS to protect communication.

Base URL: `https://api.tickethub.example/v1`

| Method | Endpoint | Description | Success status |
|---|---|---|---|
| GET | `/events` | Retrieve and browse available events | 200 OK |
| GET | `/events/{eventId}` | View details of a specific event | 200 OK |
| GET | `/events/{eventId}/seats` | Retrieve seat availability and prices | 200 OK |
| POST | `/events/{eventId}/holds` | Temporarily reserve selected seats | 201 Created |
| POST | `/orders/{orderId}/payments` | Submit payment for an order | 200 OK |
| GET | `/users/me/tickets` | View the logged-in user's purchased tickets | 200 OK |
| DELETE | `/holds/{holdId}` | Release a temporary seat hold | 204 No Content |

### Example: Viewing Available Seats

**Request**

`GET /events/205/seats`

**Response**

```json
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
```

### Example: Holding a Seat

**Request**

`POST /events/205/holds`

```json
{
  "seatIds": [501],
  "queueToken": "valid-queue-token"
}
```

**Successful Response**

```json
{
  "holdId": 901,
  "eventId": 205,
  "seatIds": [501],
  "status": "held",
  "expiresAt": "2026-11-15T14:10:00Z"
}
```

The hold gives the customer a limited period, for example five minutes, to complete payment. If the time expires without successful payment, the seat becomes available again.

### API Error Handling

The API should return clear error messages and appropriate HTTP status codes.

| Status code | Meaning | Example |
|---|---|---|
| 400 | Bad Request | Invalid seat selection |
| 401 | Unauthorized | User needs to log in |
| 403 | Forbidden | User is not allowed to access the order |
| 404 | Not Found | Event or seat does not exist |
| 409 | Conflict | Seat is no longer available |
| 500 | Internal Server Error | Unexpected server failure |

Example error response:

```json
{
  "error": {
    "code": "SEAT_UNAVAILABLE",
    "message": "This seat is no longer available. Please select another seat."
  }
}
```

## 4. Data Model

TicketHub uses a relational database such as PostgreSQL because booking information has clear relationships and requires accurate transactions.

The main tables are users, events, seats, orders and order_items.

### A. Users Table

Stores information about registered customers.

| Column | Data type | Description |
|---|---|---|
| user_id | INT | Primary key |
| full_name | VARCHAR(100) | Customer's name |
| email | VARCHAR(150) | Unique email address |
| password_hash | VARCHAR(255) | Encrypted password hash |
| created_at | TIMESTAMP | Account creation date |

### B. Events Table

Stores details about concerts and other events.

| Column | Data type | Description |
|---|---|---|
| event_id | INT | Primary key |
| event_name | VARCHAR(150) | Name of the event |
| venue | VARCHAR(150) | Event location |
| event_date | TIMESTAMP | Date and time of the event |
| sale_start | TIMESTAMP | Time ticket sales begin |
| status | VARCHAR(20) | Scheduled, on_sale or completed |

### C. Seats Table

Stores the seats available for each event and their current booking status.

| Column | Data type | Description |
|---|---|---|
| seat_id | INT | Primary key |
| event_id | INT | Foreign key referencing events |
| seat_number | VARCHAR(20) | Seat identifier |
| price | DECIMAL(10,2) | Ticket price |
| status | VARCHAR(20) | Available, held or sold |
| held_by | INT | User currently holding the seat, if any |
| hold_expires_at | TIMESTAMP | Time when the temporary hold expires |

A unique constraint on `(event_id, seat_number)` ensures that the same seat is not created twice for one event.

### D. Orders Table

Stores customer booking and payment information.

| Column | Data type | Description |
|---|---|---|
| order_id | INT | Primary key |
| user_id | INT | Foreign key referencing users |
| order_date | TIMESTAMP | Date the order was created |
| total_amount | DECIMAL(10,2) | Total order amount |
| status | VARCHAR(20) | Pending, paid, failed or expired |
| payment_reference | VARCHAR(100) | Payment provider reference |

### E. Order_Items Table

Connects orders to the individual seats being purchased. This allows one order to contain multiple tickets.

| Column | Data type | Description |
|---|---|---|
| order_item_id | INT | Primary key |
| order_id | INT | Foreign key referencing orders |
| seat_id | INT | Foreign key referencing seats |
| ticket_price | DECIMAL(10,2) | Price of the seat at purchase time |

A unique constraint on `seat_id` in active booking records, together with the seat reservation transaction, prevents the same seat from being assigned to multiple active orders. Expired or cancelled booking records must be handled carefully so that a released seat can be purchased again.

### Relationships

- One user can create many orders. This is a one-to-many relationship.
- One event can have many seats. This is a one-to-many relationship.
- One order can contain many order items. This is a one-to-many relationship.
- One seat can appear in an order item when it is reserved or sold. The booking process controls when it can be assigned again.

The order_items table connects orders and seats, making it possible to purchase several seats in one checkout.

## 5. Preventing Double-Booking

Preventing double-booking is one of the most important requirements of TicketHub. During a popular concert sale, several customers may select the same seat within milliseconds. The system must ensure that only one customer can successfully hold and purchase it.

TicketHub uses database transactions, row-level locking, conditional updates and database constraints to protect seat availability.

### How the Process Works

1. A customer selects an available seat and requests a temporary hold.
2. The backend starts a database transaction and locks the selected seat row so another booking request cannot modify it at the same time.
3. The system checks whether the seat is still available or whether an earlier hold has expired.
4. If the seat is available, its status changes to `held`, and the customer receives a hold expiration time.
5. If another customer has already held or purchased the seat, the request is rejected with a `409 Conflict` response.
6. The customer completes payment before the hold expires.
7. After payment is verified, the system updates the order to `paid` and the seat to `sold`.
8. If payment fails or the hold expires, the seat is released for another customer.

### Database Transaction Example

A simplified SQL transaction for holding one seat could look like this:

```sql
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

-- The application checks that exactly one row was updated.
-- If zero rows were updated, the seat is unavailable.

COMMIT;
```

The application must check the update result before confirming the hold. If the update affects zero rows, it must reject the request instead of creating a valid reservation. For a booking involving several seats, all selected seats should be held within one transaction so the customer does not receive only part of the requested booking.

The row lock ensures that competing requests cannot successfully change the same seat at the same time. The conditional update checks availability again inside the database rather than relying on possibly outdated information displayed in the browser.

Payment should be verified with the payment provider before an order is marked as paid. The payment process should also use unique transaction references and idempotency checks to avoid charging a customer twice if a request is retried.

This approach makes the database, rather than the frontend, responsible for the final booking decision.

## 6. System Architecture

TicketHub uses a scalable, multi-layer architecture to separate browsing, booking, payment and background processing. This allows the system to handle ordinary activity and sudden increases in demand during popular ticket releases.

### Architecture Diagram

```text
                  CUSTOMERS
                      |
                      v
                 DNS SERVICE
                      |
                      v
                CDN / EDGE CACHE
                      |
                      v
               LOAD BALANCER
                      |
              VIRTUAL WAITING ROOM
                      |
             +--------+--------+
             |                 |
             v                 v
        APP SERVER 1      APP SERVER 2
             |                 |
             +--------+--------+
                      |
             +--------+--------+
             |                 |
             v                 v
          REDIS CACHE      BOOKING SERVICE
             |                 |
             |                 v
             |           PRIMARY DATABASE
             |                 |
             |                 v
             |           READ REPLICA
             |                 |
             +-----------------+
                      |
                      v
                 MESSAGE QUEUE
                      |
                      v
              BACKGROUND WORKERS
                      |
             +--------+--------+
             |                 |
             v                 v
        EMAIL SERVICE     TICKET GENERATION
```

### Component Explanations

- **Customers:** Use phones or computers to browse events and purchase tickets.
- **DNS Service:** Directs users to the TicketHub platform using its domain name.
- **CDN / Edge Cache:** Delivers static files such as images, stylesheets and scripts from locations closer to users, reducing requests to the main servers.
- **Load Balancer:** Distributes incoming requests across multiple application servers to prevent one server from handling all the traffic.
- **Virtual Waiting Room:** Controls the number of customers entering the booking process during high-demand sales and manages their place in the queue.
- **Application Servers:** Handle user requests, event browsing, authentication and communication with backend services.
- **Redis Cache:** Temporarily stores frequently requested information, such as event details and seat availability, to reduce database reads.
- **Booking Service:** Handles seat holds, order creation and booking rules, ensuring that seat availability is checked before a reservation is accepted.
- **Primary Database:** Stores the main records for users, events, seats and orders, and processes booking transactions.
- **Read Replica:** Handles suitable read operations to reduce the workload on the primary database; booking decisions always use the primary database to avoid relying on stale availability data.
- **Message Queue:** Stores background tasks so that activities such as email delivery and ticket generation do not slow down the checkout process.
- **Background Workers:** Process queued tasks, including sending booking confirmations and generating digital tickets.
- **Email Service:** Sends customers payment confirmations and ticket notifications.
- **Ticket Generation:** Creates digital tickets with unique booking references or QR codes that can be checked at the event entrance.

### How the Architecture Handles a Popular Sale

When a popular concert goes on sale, many customers may arrive at almost the same time. The CDN handles static content, while the virtual waiting room controls entry into the booking process. This prevents all 200,000 customers from sending booking requests directly to the application servers at once.

The load balancer distributes admitted traffic across multiple application servers. The Redis cache handles frequently requested event information, reducing unnecessary database queries.

Actual seat reservations are processed by the booking service and primary database. Transactions and row-level locking protect the seats from being sold more than once. Once payment is confirmed, background workers handle ticket generation and notification tasks through the message queue.

Additional application servers can be added as demand increases. If one application server fails, the load balancer can direct requests to the remaining healthy servers. Database backups, replication and monitoring help protect the stored information and improve recovery.

## 7. Design Trade-Offs

### A. Fairness vs Speed During Ticket Sales

A virtual waiting room helps control traffic and gives customers an organized way to enter a popular sale. However, it introduces an extra step and may increase the time customers spend waiting before they can purchase tickets.

This is a trade-off between maintaining fair access and providing the fastest possible checkout experience. TicketHub prioritizes controlled access and system stability during high-demand sales.

### B. Strong Consistency vs Performance

Using database transactions and row-level locking protects seat availability and prevents double-booking. However, these operations can slow down when many customers compete for the same seats.

TicketHub prioritizes correctness for bookings because selling the same seat twice would create serious customer and business problems. Caching can improve browsing performance, but final seat availability must always be verified against the primary database.

### C. Automatic Scaling vs Operating Cost

Adding more application servers allows TicketHub to handle sudden increases in traffic. However, running additional servers and supporting services increases infrastructure costs.

The platform can use automatic scaling to add capacity during major ticket releases and reduce it when demand returns to normal. This balances performance with operating expenses.

### Conclusion

TicketHub is designed to provide a reliable and fair event ticketing experience while supporting both normal daily activity and high-demand concert releases. Its use of a virtual waiting room, scalable application servers, database transactions and background processing helps maintain performance without compromising booking accuracy. Most importantly, the database controls seat reservations so that each seat can only be successfully sold once.