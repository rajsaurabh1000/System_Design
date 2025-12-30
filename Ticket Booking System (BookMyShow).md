
# Ticket Booking System (BookMyShow) - System Design

## 1. Functional Requirements (FR)

### Core Features
1. **Event Management**
   - Browse events (movies, concerts, sports, plays)
   - Search and filter by category, location, date, price
   - View event details and show timings

2. **Venue & Show Management**
   - List venues by location
   - View seat layouts and availability
   - Real-time seat status updates

3. **Booking Flow**
   - Select seats with temporary locking (10 minutes)
   - Handle concurrent seat selection
   - Prevent double booking

4. **Payment Processing**
   - Multiple payment methods (card, UPI, wallet)
   - Apply promo codes
   - Generate booking confirmation and QR code

5. **User Management**
   - Registration and authentication
   - Booking history
   - Saved payment methods

6. **Notifications**
   - Booking confirmation (email/SMS)
   - Show reminders
   - Cancellation notices

### Out of Scope
- Food & beverage ordering
- Reviews and ratings
- Social sharing

---

## 2. Non-Functional Requirements (NFR)

### Scalability
- Support 10M daily active users
- Handle 500K concurrent users during peak
- Process 5M bookings per day

### Performance
- Search results: < 300ms
- Seat availability: < 200ms
- Booking completion: < 3 seconds
- Payment processing: < 5 seconds

### Availability
- 99.99% uptime
- Zero data loss for confirmed bookings
- Graceful degradation during peak load

### Consistency
- **Strong consistency** for seat booking (no double booking)
- **Eventual consistency** for event listings
- **ACID guarantees** for payment transactions

### Security
- PCI-DSS compliant payment processing
- OAuth 2.0 authentication
- Rate limiting and DDoS protection

---

## 3. Core Entities

### 3.1 Enums

```java
enum EventCategory {
    MOVIE, CONCERT, SPORTS, THEATER, COMEDY_SHOW, EXHIBITION
}

enum SeatStatus {
    AVAILABLE, LOCKED, BOOKED, BLOCKED
}

enum BookingStatus {
    PENDING, CONFIRMED, CANCELLED, REFUNDED, EXPIRED
}

enum PaymentStatus {
    INITIATED, PENDING, SUCCESS, FAILED, REFUNDED
}

enum PaymentMethod {
    CREDIT_CARD, DEBIT_CARD, UPI, NET_BANKING, WALLET
}
```

### 3.2 Core Classes

```java
class User {
    UUID userId;
    String name;
    String email;
    String phoneNumber;
    String passwordHash;
    List<PaymentMethod> savedPaymentMethods;
    LocalDateTime createdAt;
}

class Event {
    UUID eventId;
    String title;
    String description;
    EventCategory category;
    List<String> languages;
    int durationMinutes;
    LocalDate startDate;
    LocalDate endDate;
    String posterUrl;
    double averageRating;
}

class Venue {
    UUID venueId;
    String name;
    String address;
    String city;
    double latitude;
    double longitude;
    List<Section> sections;
    int totalCapacity;
}

class Section {
    UUID sectionId;
    UUID venueId;
    String name;  // "Screen 1", "Section A"
    int totalSeats;
    SeatLayout layout;
}

class Seat {
    UUID seatId;
    UUID sectionId;
    String seatNumber;  // "A1", "B5"
    SeatType type;  // REGULAR, PREMIUM, VIP
    int rowNumber;
    int columnNumber;
}

class Show {
    UUID showId;
    UUID eventId;
    UUID sectionId;
    LocalDateTime startTime;
    LocalDateTime endTime;
    Map<SeatType, BigDecimal> basePricing;
    ShowStatus status;
    int availableSeatsCount;
}

class ShowSeat {
    UUID showSeatId;
    UUID showId;
    UUID seatId;
    SeatStatus status;
    BigDecimal currentPrice;
    UUID lockedByUserId;
    LocalDateTime lockExpiry;
    Long version;  // Optimistic locking
}

class Booking {
    UUID bookingId;
    UUID userId;
    UUID showId;
    List<ShowSeat> seats;
    BookingStatus status;
    BigDecimal totalAmount;
    BigDecimal discount;
    String promoCode;
    String qrCode;
    LocalDateTime bookedAt;
    LocalDateTime expiryTime;
}

class Payment {
    UUID paymentId;
    UUID bookingId;
    BigDecimal amount;
    PaymentMethod method;
    PaymentStatus status;
    String transactionId;
    String idempotencyKey;
    LocalDateTime processedAt;
}
```

### 3.3 Key Interfaces

```java
interface SeatLockManager {
    List<ShowSeat> lockSeats(UUID showId, List<UUID> seatIds, UUID userId, Duration lockDuration);
    void releaseSeats(UUID showId, List<UUID> seatIds);
    void confirmSeats(UUID showId, List<UUID> seatIds);
    void releaseExpiredLocks();
}

interface PaymentProcessor {
    Payment processPayment(UUID bookingId, BigDecimal amount, PaymentDetails details, String idempotencyKey);
    void handleWebhook(PaymentWebhookEvent event);
    Refund initiateRefund(UUID paymentId, BigDecimal amount, String reason);
}

interface PricingEngine {
    BigDecimal calculatePrice(UUID showId, SeatType seatType, UUID userId);
    Map<SeatType, BigDecimal> getShowPricing(UUID showId, UUID userId);
}

interface SearchService {
    SearchResult search(SearchQuery query);
    List<String> autocomplete(String prefix, int limit);
    List<Event> searchNearby(Location location, double radiusKm);
}

interface NotificationService {
    void sendBookingConfirmation(Booking booking);
    void sendShowReminder(Booking booking, int hoursBefore);
    void sendCancellation(Booking booking);
}
```

---

## 4. API Design

### User APIs
```
POST   /api/v1/users/register
POST   /api/v1/users/login
GET    /api/v1/users/{userId}/profile
GET    /api/v1/users/{userId}/bookings
```

### Event APIs
```
GET    /api/v1/events?city={city}&category={category}&date={date}
GET    /api/v1/events/{eventId}
GET    /api/v1/events/{eventId}/shows?city={city}&date={date}
GET    /api/v1/events/search?query={query}
```

### Show APIs
```
GET    /api/v1/shows/{showId}
GET    /api/v1/shows/{showId}/seats
GET    /api/v1/shows/{showId}/pricing
```

### Booking APIs
```
POST   /api/v1/bookings/initiate
POST   /api/v1/bookings/{bookingId}/lock-seats
GET    /api/v1/bookings/{bookingId}
POST   /api/v1/bookings/{bookingId}/confirm
POST   /api/v1/bookings/{bookingId}/cancel
```

### Payment APIs
```
POST   /api/v1/payments/process
GET    /api/v1/payments/{paymentId}/status
POST   /api/v1/payments/webhook
```

---

## 5. High-Level Design (HLD)

### 5.1 System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Layer                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ Web App  │  │ Mobile   │  │  Partner │  │  Admin   │       │
│  │          │  │   App    │  │   Apps   │  │  Portal  │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
└───────────────────────┬─────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                      CDN / Load Balancer                         │
│                    (CloudFront / CloudFlare)                     │
└───────────────────────┬─────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                       API Gateway                                │
│         (Rate Limiting, Auth, Request Routing)                   │
└───────┬───────┬──────────┬──────────┬─────────┬─────────────────┘
        │       │          │          │         │
        ▼       ▼          ▼          ▼         ▼
┌───────────────────────────────────────────────────────────────┐
│                    Microservices Layer                         │
├─────────┬─────────┬─────────┬─────────┬─────────┬────────────┤
│  User   │  Event  │  Show   │ Booking │ Payment │   Search   │
│ Service │ Service │ Service │ Service │ Service │  Service   │
└─────────┴─────────┴─────────┴─────────┴─────────┴────────────┘
     │         │         │         │         │          │
     └─────────┴─────────┴─────────┴─────────┴──────────┘
                        │
        ┌───────────────┼───────────────┬───────────────┐
        ▼               ▼               ▼               ▼
┌─────────────┐  ┌──────────┐  ┌─────────────┐ ┌──────────────┐
│   Redis     │  │PostgreSQL│  │ Elasticsearch│ │  Message     │
│  (Cache +   │  │ (Primary │  │  (Search)   │ │  Queue       │
│   Locks)    │  │   DB)    │  │             │ │ (Kafka/SQS)  │
└─────────────┘  └──────────┘  └─────────────┘ └──────────────┘
        │               │               │               │
        └───────────────┴───────────────┴───────────────┘
                        │
        ┌───────────────┼───────────────┬───────────────┐
        ▼               ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────┐
│   Payment    │ │ Notification │ │   Analytics  │ │  CDN/S3  │
│   Gateway    │ │   Service    │ │   Service    │ │ (Media)  │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────┘
```

### 5.2 Data Flow - Booking Process

```
User → Search Events → Select Show → Choose Seats
  ↓
Lock Seats (Redis + DB)
  ↓
Initiate Payment (Idempotency Key)
  ↓
Payment Gateway → Success/Failure
  ↓
Confirm Booking + Generate QR Code
  ↓
Send Notification (Kafka → Email/SMS)
```

---

## 6. Low-Level Design (LLD)

<img width="890" height="872" alt="image" src="https://github.com/user-attachments/assets/f7beab27-4baf-44e4-81e7-7a13a2809847" />


### 6.2 Sequence Diagram - Complete Booking Flow

```
User    API Gateway  BookingSvc  ShowSvc  LockMgr  PaymentSvc  NotifySvc
 │           │           │          │         │         │           │
 │ Search    │           │          │         │         │           │
 ├──────────>│           │          │         │         │           │
 │           ├──────────>│          │         │         │           │
 │           │           ├─────────>│         │         │           │
 │<──────────────────────┴──────────┘         │         │           │
 │           │           │          │         │         │           │
 │ Select    │           │          │         │         │           │
 │ Seats     │           │          │         │         │           │
 ├──────────>│           │          │         │         │           │
 │           ├──────────>│          │         │         │           │
 │           │           ├─────────────────────>│       │           │
 │           │           │          │    Lock  │       │           │
 │           │           │          │   (Redis)│       │           │
 │           │           │<─────────────────────┤       │           │
 │           │           ├─────────>│         │         │           │
 │           │           │   Update │         │         │           │
 │           │           │   DB     │         │         │           │
 │<──────────────────────┴──────────┘         │         │           │
 │ BookingID │           │          │         │         │           │
 │           │           │          │         │         │           │
 │ Payment   │           │          │         │         │           │
 ├──────────>│           │          │         │         │           │
 │           ├──────────>│          │         │         │           │
 │           │           ├─────────────────────┴────────>│           │
 │           │           │          │         │  Process│           │
 │           │           │          │         │  (Gateway)          │
 │           │           │<────────────────────┴─────────┤           │
 │           │           │          │         │ Success │           │
 │           │           ├─────────────────────>│        │           │
 │           │           │          │    Confirm        │           │
 │           │           │          │    Seats │        │           │
 │           │           ├─────────>│         │         │           │
 │           │           │   Update │         │         │           │
 │<──────────────────────┴──────────┘         │         │           │
 │ E-Ticket  │           │          │         │         │           │
 │ + QR Code │           ├────────────────────────────────────────────>│
 │           │           │          │         │         │    Notify │
 │           │           │          │         │         │   (Kafka) │
```

### 6.3 Seat Locking Mechanism

```
┌─────────────────────────────────────────────────────┐
│         TWO-PHASE DISTRIBUTED LOCKING               │
└─────────────────────────────────────────────────────┘

Phase 1: Redis Lock (Fast, Distributed)
┌──────────────────────────────────────┐
│ Key: "seat_lock:{showId}:{seatId}"   │
│ Value: {userId}                      │
│ TTL: 600 seconds (10 minutes)       │
│                                      │
│ Command: SET key userId NX EX 600   │
│ (Atomic operation)                  │
│                                      │
│ Result: true/false                  │
└──────────────────────────────────────┘
        │
        ▼ (if true)
Phase 2: Database Update (Persistent)
┌──────────────────────────────────────┐
│ UPDATE show_seats                    │
│ SET status = 'LOCKED',               │
│     locked_by_user_id = :userId,     │
│     lock_expiry = NOW() + 10min,     │
│     version = version + 1            │
│ WHERE show_seat_id = :seatId         │
│   AND status = 'AVAILABLE'           │
│   AND version = :currentVersion      │
│                                      │
│ (Optimistic Locking)                 │
└──────────────────────────────────────┘
```

### 6.4 Payment Idempotency

```
┌─────────────────────────────────────────────────────┐
│         IDEMPOTENT PAYMENT PROCESSING               │
└─────────────────────────────────────────────────────┘

Request: processPayment(bookingId, amount, idempotencyKey)
    ↓
Check Cache/DB: findByIdempotencyKey(key)
    ↓
    ├─ Found? → Return Cached Result
    │
    └─ Not Found:
        ↓
        Create Payment Record (status: INITIATED)
        ↓
        Call Payment Gateway
        ↓
        Update Payment Record (status: SUCCESS/FAILED)
        ↓
        Store with Idempotency Key
        ↓
        Return Result

Idempotency Key = Hash(bookingId + userId + timestamp)
Stored for 24 hours in Redis + Database
```

### 6.5 Booking State Machine

```
    ┌──────────┐
    │INITIATED │
    └─────┬────┘
          │
          ▼
    ┌──────────┐
    │  LOCKED  │──────timeout────> EXPIRED
    └─────┬────┘
          │
          ├──processPayment──>┌───────────────┐
          │                   │PAYMENT_PENDING│
          │                   └───────┬───────┘
          │                           │
          │                   ┌───────┴───────┐
          │                   │               │
          │              success          failure
          │                   │               │
          │                   ▼               ▼
          │              ┌──────────┐    ┌────────┐
          │              │CONFIRMED │    │ FAILED │
          │              └─────┬────┘    └────────┘
          │                    │
          └──cancel────>  ┌────┴────┐
                         │CANCELLED │
                         └─────┬────┘
                               │
                               ▼
                         ┌──────────┐
                         │ REFUNDED │
                         └──────────┘
```

---

## 7. Optimizations & Approaches

### 7.1 Caching Strategy
**Multi-layer caching (CDN, Redis, Application)**
Reduces database load by 70%, improves response time from 500ms to 50ms for event listings and show availability.

### 7.2 Database Optimization
**Read replicas + Connection pooling + Composite indexes**
Handles 10K reads/sec with 3 read replicas, connection pool size 50, indexes on (showId, seatId) and (userId, bookedAt).

### 7.3 Distributed Locking
**Redis SETNX + Database optimistic locking (version field)**
Prevents double booking with 99.99% success rate, handles 1000 concurrent lock attempts per second atomically.

### 7.4 Payment Reliability
**Idempotency keys + Webhook reconciliation + Retry with exponential backoff**
Zero duplicate charges, 99.9% success rate, webhooks handle async updates, background jobs reconcile stuck payments.

### 7.5 Search Performance
**Elasticsearch with inverted index + Edge n-grams + Geo-spatial index**
Sub-200ms search across millions of events, typo-tolerant fuzzy matching, distance-based venue sorting.

### 7.6 Horizontal Scaling
**Microservices + Auto-scaling + Database sharding by city**
Scales to 10M DAU, Kubernetes HPA scales services 2-50 instances based on CPU/memory, city-based sharding for data locality.

### 7.7 High Availability
**Multi-region deployment + Circuit breaker + Health checks**
99.99% uptime, automatic failover in <30 seconds, circuit breaker trips after 5 failures, graceful degradation during peak.

### 7.8 Async Processing
**Message queue (Kafka) for notifications + Event-driven architecture**
Decouples services, handles 10K events/sec, retry mechanism for failed messages, enables event sourcing and audit logs.

### 7.9 Rate Limiting
**Redis-based sliding window counter per user/IP**
Prevents abuse with 100 requests/min per user, distributed rate limiting across services, protects against DDoS attacks.

### 7.10 Monitoring
**Distributed tracing + Metrics + Centralized logging**
Jaeger for request tracing with correlation IDs, Prometheus + Grafana for metrics, ELK for logs, PagerDuty for alerts.

---

## 8. Design Patterns Used

### 8.1 Creational Patterns

**Factory Pattern**
- Creates event types (MovieEvent, ConcertEvent, SportsEvent)
- EventFactory.create(category) returns appropriate event instance

**Builder Pattern**
- Builds complex objects (SearchQuery, BookingRequest)
- SearchQuery.builder().city("Mumbai").category(CONCERT).build()

**Singleton Pattern**
- Database connection pool, Redis client, Configuration manager
- Ensures single shared instance for expensive resources

### 8.2 Structural Patterns

**Adapter Pattern**
- Unified interface for multiple payment gateways (Stripe, Razorpay, PayPal)
- Each gateway adapter implements PaymentGateway interface

**Facade Pattern**
- BookingFacade simplifies complex booking flow
- Single interface coordinates Show, Lock, Payment, Notification services

**Proxy Pattern**
- Caching proxy intercepts database queries
- Checks cache before hitting database, transparent to client

**Decorator Pattern**
- Adds features to bookings (insurance, cancellation protection)
- Wraps base booking with additional functionality

### 8.3 Behavioral Patterns

**Strategy Pattern**
- Different pricing strategies (DynamicPricing, StaticPricing, PromoPricing)
- Selected at runtime based on configuration

**Observer Pattern**
- Event-driven notifications for booking events
- BookingSubject notifies EmailObserver, SMSObserver, AnalyticsObserver

**Command Pattern**
- Encapsulates booking operations (CreateBooking, CancelBooking)
- Enables queuing, logging, and undo capability

**State Pattern**
- Manages booking state transitions
- Each state (Pending, Confirmed, Cancelled) handles allowed transitions

**Chain of Responsibility**
- Payment processing pipeline (Validate → Charge → Confirm → Notify)
- Each handler processes request and passes to next

**Template Method**
- Abstract booking flow with category-specific customization
- MovieBookingService extends AbstractBookingService

---

## 9. Data Structures Used

### 9.1 Core Data Structures

**HashMap / Map**
- **Use**: Seat availability lookup, in-memory caching, pricing map
- **Benefit**: O(1) average case lookup, fast seat status checks
- **Example**: Map<UUID, ShowSeat> for quick seat retrieval

**TreeMap / SortedMap**
- **Use**: Time-ordered show listings, price ranges
- **Benefit**: O(log n) operations with sorted order
- **Example**: Shows sorted by start time for range queries

**HashSet / Set**
- **Use**: Unique seat IDs, applied promo codes, user wishlists
- **Benefit**: O(1) membership testing, no duplicates
- **Example**: Set<UUID> for selected seat IDs

**ArrayList / List**
- **Use**: Booking history, search results, seat lists
- **Benefit**: O(1) indexed access, fast iteration
- **Example**: List<Booking> for user booking history

**LinkedList**
- **Use**: Queue of pending bookings, notification queue
- **Benefit**: O(1) insertion/deletion at ends
- **Example**: Queue for processing bookings in order

**Priority Queue (Heap)**
- **Use**: Processing bookings by priority (VIP users, expiry time)
- **Benefit**: O(log n) insertion with priority ordering
- **Example**: PriorityQueue<Booking> ordered by expiry time

**Trie**
- **Use**: Autocomplete suggestions in search
- **Benefit**: Efficient prefix matching, O(k) where k = key length
- **Example**: Event title autocomplete with prefix search

**Graph**
- **Use**: Venue seat layout (adjacent seats, aisles)
- **Benefit**: Models relationships between seats
- **Example**: Graph where nodes are seats, edges are adjacency

**Bitmap**
- **Use**: Compact seat availability representation
- **Benefit**: 1 bit per seat, memory efficient for large venues
- **Example**: BitSet for 1000 seats uses only 125 bytes

**B+ Tree (Database Index)**
- **Use**: Database indexes on showId, eventId, userId
- **Benefit**: O(log n) range queries, efficient disk access
- **Example**: Index on (showId, seatId) for fast seat lookups

---

## 10. Database Schema

```sql
-- Users
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_email (email),
    INDEX idx_phone (phone)
);

-- Events
CREATE TABLE events (
    event_id UUID PRIMARY KEY,
    title VARCHAR(500) NOT NULL,
    category VARCHAR(50) NOT NULL,
    description TEXT,
    duration_minutes INT,
    start_date DATE,
    end_date DATE,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_category (category),
    INDEX idx_dates (start_date, end_date)
);

-- Venues
CREATE TABLE venues (
    venue_id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    latitude DECIMAL(10,8),
    longitude DECIMAL(11,8),
    INDEX idx_city (city),
    INDEX idx_location (latitude, longitude)
);

-- Sections
CREATE TABLE sections (
    section_id UUID PRIMARY KEY,
    venue_id UUID NOT NULL,
    name VARCHAR(100) NOT NULL,
    total_seats INT NOT NULL,
    FOREIGN KEY (venue_id) REFERENCES venues(venue_id),
    INDEX idx_venue (venue_id)
);

-- Seats
CREATE TABLE seats (
    seat_id UUID PRIMARY KEY,
    section_id UUID NOT NULL,
    seat_number VARCHAR(20) NOT NULL,
    seat_type VARCHAR(20) NOT NULL,
    row_number INT,
    column_number INT,
    FOREIGN KEY (section_id) REFERENCES sections(section_id),
    UNIQUE KEY uk_section_seat (section_id, seat_number)
);

-- Shows
CREATE TABLE shows (
    show_id UUID PRIMARY KEY,
    event_id UUID NOT NULL,
    section_id UUID NOT NULL,
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP NOT NULL,
    status VARCHAR(20) NOT NULL,
    available_seats INT NOT NULL,
    FOREIGN KEY (event_id) REFERENCES events(event_id),
    FOREIGN KEY (section_id) REFERENCES sections(section_id),
    INDEX idx_event_time (event_id, start_time),
    INDEX idx_section_time (section_id, start_time)
);

-- Show Seats (per show availability)
CREATE TABLE show_seats (
    show_seat_id UUID PRIMARY KEY,
    show_id UUID NOT NULL,
    seat_id UUID NOT NULL,
    status VARCHAR(20) DEFAULT 'AVAILABLE',
    current_price DECIMAL(10,2),
    locked_by_user_id UUID,
    lock_expiry TIMESTAMP,
    version BIGINT DEFAULT 0,  -- Optimistic locking
    FOREIGN KEY (show_id) REFERENCES shows(show_id),
    FOREIGN KEY (seat_id) REFERENCES seats(seat_id),
    UNIQUE KEY uk_show_seat (show_id, seat_id),
    INDEX idx_show_status (show_id, status),
    INDEX idx_lock_expiry (lock_expiry)
);

-- Bookings
CREATE TABLE bookings (
    booking_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    show_id UUID NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    total_amount DECIMAL(10,2) NOT NULL,
    discount DECIMAL(10,2) DEFAULT 0,
    qr_code VARCHAR(500),
    booked_at TIMESTAMP DEFAULT NOW(),
    expiry_time TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (show_id) REFERENCES shows(show_id),
    INDEX idx_user (user_id, booked_at DESC),
    INDEX idx_show (show_id),
    INDEX idx_status (status)
);

-- Booking Seats (junction table)
CREATE TABLE booking_seats (
    booking_id UUID NOT NULL,
    show_seat_id UUID NOT NULL,
    PRIMARY KEY (booking_id, show_seat_id),
    FOREIGN KEY (booking_id) REFERENCES bookings(booking_id),
    FOREIGN KEY (show_seat_id) REFERENCES show_seats(show_seat_id)
);

-- Payments
CREATE TABLE payments (
    payment_id UUID PRIMARY KEY,
    booking_id UUID NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    payment_method VARCHAR(50) NOT NULL,
    status VARCHAR(20) DEFAULT 'INITIATED',
    transaction_id VARCHAR(255),
    idempotency_key VARCHAR(255) UNIQUE,
    processed_at TIMESTAMP,
    FOREIGN KEY (booking_id) REFERENCES bookings(booking_id),
    INDEX idx_booking (booking_id),
    INDEX idx_idempotency (idempotency_key),
    INDEX idx_transaction (transaction_id)
);
```

---

## Summary

### Key Technical Decisions

1. **Seat Locking**: Two-phase (Redis + DB) ensures no double booking with atomic operations
2. **Payment Idempotency**: UUID-based keys prevent duplicate charges, stored 24 hours
3. **Search**: Elasticsearch enables sub-second search across millions of events
4. **Scalability**: Microservices + auto-scaling + database sharding handles 10M DAU
5. **Reliability**: Circuit breakers + webhooks + reconciliation ensure 99.9% payment success

### System Guarantees

✓ No double booking (distributed locks)
✓ No duplicate payments (idempotency keys)
✓ Sub-second search (Elasticsearch)
✓ 99.99% uptime (multi-region)
✓ Handles 10M DAU with 5x traffic spikes
