# Ticket Booking System (BookMyShow) - System Design

## 1. Functional Requirements (FR)

### Core Features
1. **Event Management**
   - Browse events (movies, concerts, sports, plays, exhibitions)
   - Search and filter by category, location, date, price
   - View event details (description, venue, timings, reviews)
   - List upcoming and trending events

2. **Venue & Show Management**
   - List venues by location and type
   - View venue details (capacity, facilities, seating layout)
   - Show timings for each event
   - Real-time seat/ticket availability

3. **Ticket Selection & Booking**
   - View real-time availability
   - Select seats/tickets (single or multiple)
   - Temporary seat locking during booking
   - Handle concurrent seat selection conflicts

4. **Pricing & Payment**
   - Dynamic pricing based on demand
   - Calculate total with taxes and fees
   - Multiple payment methods (card, wallet, UPI, net banking)
   - Apply coupons and promotional codes
   - Split payment support

5. **User Management**
   - Registration and authentication
   - User profile and preferences
   - Booking history
   - Wishlist and notifications
   - Saved payment methods

6. **Booking Management**
   - View booking details and e-tickets
   - Cancel bookings with refund handling
   - Transfer tickets
   - Generate QR codes for entry
   - Send reminders and notifications

### Out of Scope (Extensions)
- Food & beverage ordering
- Parking reservation
- Reviews and ratings (can add later)
- Social sharing

---

## 2. Non-Functional Requirements (NFR)

### 2.1 Scalability
- Support 10M+ daily active users
- Handle 500K+ concurrent users during peak hours
- Process 5M+ bookings per day
- Scale horizontally across regions

### 2.2 Performance
- Event search: < 300ms
- Seat availability: < 200ms
- Booking flow: < 3 seconds end-to-end
- Payment processing: < 5 seconds
- API response time: p99 < 500ms

### 2.3 Availability
- 99.99% uptime (4.38 minutes downtime/month)
- Handle regional failures gracefully
- Zero data loss for confirmed bookings
- Graceful degradation during peak load

### 2.4 Consistency
- **Strong consistency** for seat booking (no double booking)
- **Eventual consistency** for event listings and search
- **ACID guarantees** for payment transactions
- Idempotent payment operations

### 2.5 Reliability
- Auto-release locked seats after timeout (10 mins)
- Payment retry mechanism with exponential backoff
- Automated refund processing
- Data backup and disaster recovery

### 2.6 Security
- PCI-DSS compliant payment processing
- End-to-end encryption for sensitive data
- Rate limiting and DDoS protection
- Authentication and authorization (OAuth 2.0)
- Audit logging for all transactions

---

## 3. Core Entities (Classes & Enums)

### 3.1 Enums

```java
enum EventCategory {
    MOVIE,
    CONCERT,
    SPORTS,
    THEATER,
    COMEDY_SHOW,
    EXHIBITION,
    WORKSHOP,
    OTHER
}

enum EventStatus {
    UPCOMING,
    LIVE,
    COMPLETED,
    CANCELLED,
    POSTPONED
}

enum SeatType {
    REGULAR,
    PREMIUM,
    VIP,
    RECLINER,
    WHEELCHAIR_ACCESSIBLE
}

enum SeatStatus {
    AVAILABLE,
    LOCKED,
    BOOKED,
    BLOCKED  // Maintenance or reserved
}

enum BookingStatus {
    PENDING,
    CONFIRMED,
    CANCELLED,
    REFUNDED,
    EXPIRED
}

enum PaymentMethod {
    CREDIT_CARD,
    DEBIT_CARD,
    UPI,
    NET_BANKING,
    WALLET,
    EMI
}

enum PaymentStatus {
    INITIATED,
    PENDING,
    SUCCESS,
    FAILED,
    REFUNDED
}

enum NotificationType {
    EMAIL,
    SMS,
    PUSH,
    WHATSAPP
}

enum RefundStatus {
    INITIATED,
    PROCESSING,
    COMPLETED,
    FAILED
}
```

### 3.2 Core Classes

#### 3.2.1 User
```java
class User {
    // Fields
    private UUID userId;
    private String name;
    private String email;
    private String phoneNumber;
    private String passwordHash;
    private Address address;
    private List<PaymentMethod> savedPaymentMethods;
    private UserPreferences preferences;
    private LocalDateTime createdAt;
    private LocalDateTime lastLoginAt;
    
    // Methods
    public void register(String email, String password);
    public boolean authenticate(String email, String password);
    public void updateProfile(UserProfile profile);
    public List<Booking> getBookingHistory();
    public void addToWishlist(UUID eventId);
    public void enableNotifications(NotificationType type);
}

class UserPreferences {
    private Set<EventCategory> favoriteCategories;
    private String preferredCity;
    private List<String> preferredLanguages;
    private boolean enableEmailNotifications;
    private boolean enablePushNotifications;
}

class Address {
    private String street;
    private String city;
    private String state;
    private String pincode;
    private String country;
    private double latitude;
    private double longitude;
}
```

#### 3.2.2 Event
```java
class Event {
    // Fields
    private UUID eventId;
    private String title;
    private String description;
    private EventCategory category;
    private EventStatus status;
    private List<String> languages;
    private int durationMinutes;
    private LocalDate startDate;
    private LocalDate endDate;
    private String posterUrl;
    private String trailerUrl;
    private List<String> tags;
    private EventMetadata metadata;  // Category-specific data
    private double averageRating;
    private LocalDateTime createdAt;
    
    // Methods
    public List<Show> getShows(String city, LocalDate date);
    public EventDetails getDetails();
    public void updateStatus(EventStatus newStatus);
    public boolean isActive();
    public List<Venue> getVenues();
}

class EventMetadata {
    // For Movies
    private String genre;
    private String director;
    private List<String> cast;
    private String certificate;
    
    // For Concerts/Sports
    private String artist;
    private String team;
    private String league;
    
    // For all
    private Map<String, String> additionalInfo;
}
```

#### 3.2.3 Venue
```java
class Venue {
    // Fields
    private UUID venueId;
    private String name;
    private String address;
    private String city;
    private Location location;  // lat, long
    private VenueType type;  // CINEMA, STADIUM, THEATER, ARENA
    private List<Section> sections;  // Seating sections
    private Map<String, String> amenities;
    private int totalCapacity;
    private boolean isPremium;
    private LocalDateTime createdAt;
    
    // Methods
    public List<Show> getUpcomingShows();
    public SeatLayout getSeatLayout();
    public boolean hasAmenity(String amenity);
    public double getDistanceFrom(Location location);
}

class Location {
    private double latitude;
    private double longitude;
    
    public double distanceTo(Location other);
}
```

#### 3.2.4 Section (Seating Area)
```java
class Section {
    // Fields
    private UUID sectionId;
    private UUID venueId;
    private String name;  // "Screen 1", "Section A", "VIP Box"
    private SectionType type;  // SCREEN, SECTION, BOX
    private int totalSeats;
    private SeatLayout layout;
    
    // Methods
    public List<Seat> getSeats();
    public SeatLayout getLayout();
}

class SeatLayout {
    private int rows;
    private int columns;
    private List<List<Seat>> grid;  // 2D grid
    private List<Coordinate> aisles;
    
    public Seat getSeatAt(int row, int col);
    public boolean isAisle(int row, int col);
}
```

#### 3.2.5 Seat
```java
class Seat {
    // Fields
    private UUID seatId;
    private UUID sectionId;
    private String seatNumber;  // "A1", "Row 5 Seat 12"
    private SeatType type;
    private int rowNumber;
    private int columnNumber;
    private boolean isActive;  // Can be disabled for maintenance
    
    // Methods
    public String getDisplayName();
    public boolean isAccessible();
}
```

#### 3.2.6 Show
```java
class Show {
    // Fields
    private UUID showId;
    private UUID eventId;
    private UUID sectionId;
    private LocalDateTime startTime;
    private LocalDateTime endTime;
    private Map<SeatType, BigDecimal> basePricing;
    private ShowStatus status;
    private int availableSeatsCount;
    private int bookedSeatsCount;
    private LocalDateTime createdAt;
    
    // Methods
    public List<ShowSeat> getAvailableSeats();
    public BigDecimal getPriceForSeat(UUID seatId, UUID userId);
    public double getOccupancyRate();
    public boolean isFull();
    public void updateAvailability();
    public boolean canBook();
}

enum ShowStatus {
    SCHEDULED,
    BOOKING_OPEN,
    ALMOST_FULL,
    HOUSEFULL,
    COMPLETED,
    CANCELLED
}
```

#### 3.2.7 ShowSeat
```java
class ShowSeat {
    // Fields
    private UUID showSeatId;
    private UUID showId;
    private UUID seatId;
    private SeatStatus status;
    private BigDecimal currentPrice;  // Dynamic pricing
    private UUID lockedByUserId;
    private LocalDateTime lockExpiry;
    private Long version;  // For optimistic locking
    private LocalDateTime updatedAt;
    
    // Methods
    public boolean isAvailable();
    public boolean lock(UUID userId, Duration duration);
    public void release();
    public void confirm();
    public boolean isLockExpired();
}
```

#### 3.2.8 Booking
```java
class Booking {
    // Fields
    private UUID bookingId;
    private UUID userId;
    private UUID showId;
    private List<ShowSeat> seats;
    private BookingStatus status;
    private BigDecimal totalAmount;
    private BigDecimal convenienceFee;
    private BigDecimal taxes;
    private BigDecimal discount;
    private String promoCode;
    private Payment payment;
    private String qrCode;
    private LocalDateTime bookedAt;
    private LocalDateTime expiryTime;
    private LocalDateTime confirmedAt;
    
    // Methods
    public void initiate(UUID userId, UUID showId, List<UUID> seatIds);
    public BigDecimal calculateTotal();
    public void applyPromoCode(String code);
    public boolean confirm(Payment payment);
    public void cancel(String reason);
    public boolean isExpired();
    public String generateQRCode();
    public Ticket getTicket();
}
```

#### 3.2.9 Payment
```java
class Payment {
    // Fields
    private UUID paymentId;
    private UUID bookingId;
    private BigDecimal amount;
    private PaymentMethod method;
    private PaymentStatus status;
    private String transactionId;
    private String gatewayReference;
    private String idempotencyKey;
    private int retryCount;
    private LocalDateTime initiatedAt;
    private LocalDateTime processedAt;
    private String failureReason;
    
    // Methods
    public PaymentStatus process(PaymentDetails details);
    public void retry();
    public boolean isSuccess();
    public Refund initiateRefund(BigDecimal amount, String reason);
    public void updateStatus(PaymentStatus newStatus);
}

class PaymentDetails {
    private PaymentMethod method;
    private String cardNumber;  // Tokenized
    private String cvv;
    private String upiId;
    private String walletId;
    private Map<String, String> additionalInfo;
}
```

#### 3.2.10 Refund
```java
class Refund {
    // Fields
    private UUID refundId;
    private UUID paymentId;
    private UUID bookingId;
    private BigDecimal amount;
    private RefundStatus status;
    private String reason;
    private LocalDateTime initiatedAt;
    private LocalDateTime processedAt;
    private String gatewayReference;
    
    // Methods
    public void initiate(String reason);
    public void process();
    public void updateStatus(RefundStatus status);
}
```

#### 3.2.11 Notification
```java
class Notification {
    // Fields
    private UUID notificationId;
    private UUID userId;
    private NotificationType type;
    private String subject;
    private String message;
    private Map<String, String> templateData;
    private boolean isSent;
    private LocalDateTime sentAt;
    private LocalDateTime createdAt;
    
    // Methods
    public void send();
    public void sendBookingConfirmation(Booking booking);
    public void sendCancellationNotice(Booking booking);
    public void sendReminder(Show show, int hoursBefore);
}
```

#### 3.2.12 PromoCode
```java
class PromoCode {
    // Fields
    private UUID promoId;
    private String code;
    private String description;
    private DiscountType type;  // PERCENTAGE, FLAT, CASHBACK
    private BigDecimal value;
    private BigDecimal minBookingAmount;
    private BigDecimal maxDiscount;
    private LocalDateTime validFrom;
    private LocalDateTime validUntil;
    private int maxUsageCount;
    private int currentUsageCount;
    private List<EventCategory> applicableCategories;
    private boolean isActive;
    
    // Methods
    public boolean isValid();
    public BigDecimal calculateDiscount(BigDecimal amount);
    public boolean canApply(Booking booking);
    public void incrementUsage();
}

enum DiscountType {
    PERCENTAGE,
    FLAT_AMOUNT,
    CASHBACK,
    BUY_X_GET_Y
}
```

---

## 4. API Design

### 4.1 User APIs

```
POST   /api/v1/users/register
POST   /api/v1/users/login
GET    /api/v1/users/{userId}/profile
PUT    /api/v1/users/{userId}/profile
GET    /api/v1/users/{userId}/bookings
GET    /api/v1/users/{userId}/wishlist
POST   /api/v1/users/{userId}/wishlist
DELETE /api/v1/users/{userId}/wishlist/{eventId}
```

### 4.2 Event APIs

```
GET    /api/v1/events?city={city}&category={category}&date={date}
GET    /api/v1/events/{eventId}
GET    /api/v1/events/{eventId}/shows?city={city}&date={date}
GET    /api/v1/events/trending?city={city}
GET    /api/v1/events/search?query={query}&filters={filters}
```

### 4.3 Show APIs

```
GET    /api/v1/shows/{showId}
GET    /api/v1/shows/{showId}/seats
GET    /api/v1/shows/{showId}/availability
GET    /api/v1/shows/{showId}/pricing
```

### 4.4 Booking APIs

```
POST   /api/v1/bookings/initiate
POST   /api/v1/bookings/{bookingId}/lock-seats
GET    /api/v1/bookings/{bookingId}
POST   /api/v1/bookings/{bookingId}/confirm
POST   /api/v1/bookings/{bookingId}/cancel
POST   /api/v1/bookings/{bookingId}/apply-promo
GET    /api/v1/bookings/{bookingId}/ticket
```

### 4.5 Payment APIs

```
POST   /api/v1/payments/process
GET    /api/v1/payments/{paymentId}/status
POST   /api/v1/payments/webhook
POST   /api/v1/payments/{paymentId}/refund
```

### 4.6 Venue APIs

```
GET    /api/v1/venues?city={city}
GET    /api/v1/venues/{venueId}
GET    /api/v1/venues/{venueId}/shows
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

### 5.2 Component Responsibilities

#### User Service
- User registration and authentication
- Profile management
- Wishlist and preferences
- Booking history

#### Event Service
- Event catalog management
- Event details and metadata
- Event status updates
- Category-based filtering

#### Show Service
- Show scheduling
- Seat layout management
- Real-time availability
- Dynamic pricing

#### Booking Service
- Booking initiation and management
- Seat locking mechanism
- Booking confirmation
- Cancellation and refund

#### Payment Service
- Payment processing
- Payment gateway integration
- Idempotency handling
- Refund processing

#### Search Service
- Full-text search across events
- Geo-based search
- Filters and facets
- Autocomplete

---

## 6. Low-Level Design (LLD)

### 6.1 Booking Flow Sequence

```
User          API Gateway    Booking Service    Show Service    Lock Manager    Payment Service
 │                 │               │                 │               │                │
 │─────Search─────>│               │                 │               │                │
 │                 │──────────────>│                 │               │                │
 │                 │               │────Get Shows───>│               │                │
 │<────Results─────│<──────────────│<────────────────│               │                │
 │                 │               │                 │               │                │
 │───Select Show──>│               │                 │               │                │
 │                 │──Get Seats───>│                 │               │                │
 │                 │               │───Get Layout───>│               │                │
 │<────Seats───────│<──────────────│<────────────────│               │                │
 │                 │               │                 │               │                │
 │──Select Seats──>│               │                 │               │                │
 │                 │───Lock Seats──>│                │               │                │
 │                 │               │────────────────────>Lock(Redis)│                │
 │                 │               │<────────────────────Acquired────│                │
 │                 │               │──Update DB─────>│               │                │
 │<──Booking ID────│<──────────────│<────────────────│               │                │
 │                 │               │                 │               │                │
 │───Pay Amount───>│               │                 │               │                │
 │                 │──────────────────────────────────────────────────>Process Payment│
 │                 │<─────────────────────────────────────────────────Success─────────│
 │                 │───Confirm────>│                 │               │                │
 │                 │               │──────────────────────>Confirm───│                │
 │                 │               │──Update Status─>│               │                │
 │<───Ticket───────│<──────────────│<────────────────│               │                │
```

### 6.2 Seat Locking Mechanism

```java
interface SeatLockManager {
    /**
     * Attempts to lock seats for a user
     * Returns list of successfully locked seats
     * Uses distributed Redis locks + DB updates
     */
    List<ShowSeat> lockSeats(
        UUID showId, 
        List<UUID> seatIds, 
        UUID userId, 
        Duration lockDuration
    );
    
    /**
     * Releases locked seats back to available pool
     * Called on booking expiry or cancellation
     */
    void releaseSeats(UUID showId, List<UUID> seatIds);
    
    /**
     * Confirms seats as booked (final state)
     * Called after successful payment
     */
    void confirmSeats(UUID showId, List<UUID> seatIds);
    
    /**
     * Background job to release expired locks
     * Runs every minute
     */
    void releaseExpiredLocks();
}

class RedisLockManager implements SeatLockManager {
    private RedisTemplate redis;
    private ShowSeatRepository repository;
    
    public List<ShowSeat> lockSeats(...) {
        List<ShowSeat> locked = new ArrayList<>();
        
        for (UUID seatId : seatIds) {
            String lockKey = buildLockKey(showId, seatId);
            
            // Atomic Redis operation
            boolean acquired = redis.setNX(
                lockKey, 
                userId, 
                lockDuration
            );
            
            if (acquired) {
                // Update database
                ShowSeat seat = repository.findById(seatId);
                if (seat.isAvailable()) {
                    seat.lock(userId, lockDuration);
                    repository.save(seat);
                    locked.add(seat);
                } else {
                    redis.delete(lockKey);
                }
            }
        }
        
        return locked;
    }
}
```

### 6.3 Payment Processing with Idempotency

```java
interface PaymentProcessor {
    /**
     * Process payment with idempotency guarantee
     * Same idempotency key returns same result
     */
    Payment processPayment(
        UUID bookingId,
        BigDecimal amount,
        PaymentDetails details,
        String idempotencyKey
    );
    
    /**
     * Handles payment gateway webhook
     * Updates payment status asynchronously
     */
    void handleWebhook(PaymentWebhookEvent event);
    
    /**
     * Initiates refund for cancelled booking
     */
    Refund initiateRefund(UUID paymentId, BigDecimal amount, String reason);
}

class PaymentService implements PaymentProcessor {
    private PaymentGateway gateway;
    private PaymentRepository repository;
    private BookingService bookingService;
    
    public Payment processPayment(...) {
        // Check idempotency
        Payment existing = repository.findByIdempotencyKey(idempotencyKey);
        if (existing != null) {
            return existing;  // Return cached result
        }
        
        // Create new payment
        Payment payment = new Payment();
        payment.setIdempotencyKey(idempotencyKey);
        payment.setStatus(PaymentStatus.INITIATED);
        repository.save(payment);
        
        try {
            // Call payment gateway
            GatewayResponse response = gateway.charge(amount, details);
            
            if (response.isSuccess()) {
                payment.setStatus(PaymentStatus.SUCCESS);
                payment.setTransactionId(response.getTxnId());
                
                // Confirm booking
                bookingService.confirmBooking(bookingId, payment);
            } else {
                payment.setStatus(PaymentStatus.FAILED);
                payment.setFailureReason(response.getError());
            }
            
        } catch (Exception e) {
            payment.setStatus(PaymentStatus.FAILED);
            payment.setFailureReason(e.getMessage());
        }
        
        repository.save(payment);
        return payment;
    }
}
```

### 6.4 Dynamic Pricing Engine

```java
interface PricingEngine {
    /**
     * Calculate dynamic price for seat based on demand
     * Returns price adjusted by multiplier (0.5x to 3x)
     */
    BigDecimal calculatePrice(UUID showId, SeatType seatType, UUID userId);
    
    /**
     * Get pricing for all seat types in a show
     */
    Map<SeatType, BigDecimal> getShowPricing(UUID showId, UUID userId);
}

class DynamicPricingEngine implements PricingEngine {
    private ShowService showService;
    private EventService eventService;
    private UserService userService;
    
    public BigDecimal calculatePrice(...) {
        Show show = showService.getShow(showId);
        Event event = eventService.getEvent(show.getEventId());
        
        // Base price
        BigDecimal basePrice = show.getBasePricing().get(seatType);
        
        // Calculate multiplier based on factors
        double multiplier = 1.0;
        
        // Factor 1: Occupancy (higher = higher price)
        double occupancy = show.getOccupancyRate();
        if (occupancy > 0.8) multiplier += 0.5;
        else if (occupancy > 0.6) multiplier += 0.3;
        else if (occupancy < 0.2) multiplier -= 0.2;
        
        // Factor 2: Time to show
        long hoursToShow = Duration.between(now(), show.getStartTime()).toHours();
        if (hoursToShow < 3) multiplier += 0.3;  // Last minute
        else if (hoursToShow > 168) multiplier -= 0.2;  // Early bird
        
        // Factor 3: Event popularity
        if (event.getAverageRating() > 4.5) multiplier += 0.2;
        
        // Factor 4: Day of week
        if (show.getStartTime().getDayOfWeek() == SATURDAY || 
            show.getStartTime().getDayOfWeek() == SUNDAY) {
            multiplier += 0.1;
        }
        
        // Apply constraints
        multiplier = Math.max(0.7, Math.min(2.5, multiplier));
        
        return basePrice.multiply(BigDecimal.valueOf(multiplier))
                        .setScale(2, RoundingMode.HALF_UP);
    }
}
```

### 6.5 Search Service

```java
interface SearchService {
    /**
     * Full-text search across events
     */
    SearchResult search(SearchQuery query);
    
    /**
     * Autocomplete suggestions
     */
    List<String> autocomplete(String prefix, int limit);
    
    /**
     * Geo-based search for nearby events
     */
    List<Event> searchNearby(Location location, double radiusKm);
}

class ElasticsearchService implements SearchService {
    private RestHighLevelClient client;
    
    public SearchResult search(SearchQuery query) {
        // Build Elasticsearch query
        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
        
        // Text search
        if (query.hasText()) {
            boolQuery.must(
                QueryBuilders.multiMatchQuery(query.getText())
                    .field("title", 3.0f)     // Boost title
                    .field("description")
                    .field("tags", 2.0f)       // Boost tags
            );
        }
        
        // Filters
        if (query.hasCategory()) {
            boolQuery.filter(
                QueryBuilders.termQuery("category", query.getCategory())
            );
        }
        
        if (query.hasCity()) {
            boolQuery.filter(
                QueryBuilders.termQuery("city", query.getCity())
            );
        }
        
        if (query.hasDateRange()) {
            boolQuery.filter(
                QueryBuilders.rangeQuery("startDate")
                    .gte(query.getStartDate())
                    .lte(query.getEndDate())
            );
        }
        
        // Geo search
        if (query.hasLocation()) {
            boolQuery.filter(
                QueryBuilders.geoDistanceQuery("location")
                    .point(query.getLatitude(), query.getLongitude())
                    .distance(query.getRadius(), DistanceUnit.KILOMETERS)
            );
        }
        
        // Execute search
        SearchRequest request = new SearchRequest("events");
        request.source().query(boolQuery);
        
        SearchResponse response = client.search(request);
        return mapToSearchResult(response);
    }
}
```

### 6.6 Notification Service

```java
interface NotificationService {
    /**
     * Send notification using multiple channels
     */
    void send(Notification notification);
    
    /**
     * Send booking confirmation
     */
    void sendBookingConfirmation(Booking booking);
    
    /**
     * Send show reminder
     */
    void sendShowReminder(Booking booking, int hoursBefore);
    
    /**
     * Send cancellation notice
     */
    void sendCancellation(Booking booking);
}

class AsyncNotificationService implements NotificationService {
    private EmailService emailService;
    private SMSService smsService;
    private PushService pushService;
    private MessageQueue queue;
    
    public void send(Notification notification) {
        // Async dispatch to queue
        queue.publish("notifications", notification);
    }
    
    @MessageListener("notifications")
    public void processNotification(Notification notification) {
        User user = userService.getUser(notification.getUserId());
        
        // Send via enabled channels
        if (user.isEmailEnabled()) {
            emailService.send(
                user.getEmail(),
                notification.getSubject(),
                notification.getMessage()
            );
        }
        
        if (user.isSMSEnabled()) {
            smsService.send(
                user.getPhoneNumber(),
                notification.getMessage()
            );
        }
        
        if (user.isPushEnabled()) {
            pushService.send(
                user.getDeviceToken(),
                notification.getMessage()
            );
        }
    }
}
```

---

## 7. Optimizations & Approaches

### 7.1 Caching Strategy
- **Redis cache** for hot data (events, shows, seat availability) with TTL 30-60 seconds
- **CDN caching** for static assets (posters, images) with long TTL
- **Application-level cache** for user sessions and frequently accessed data

### 7.2 Database Optimization
- **Read replicas** for read-heavy operations (search, browse)
- **Sharding** by city/region for horizontal scaling
- **Indexing** on frequently queried columns (eventId, showId, userId, city, date)

### 7.3 Seat Lock Optimization
- **Two-phase locking**: Redis for fast lock + DB for persistence
- **Lock expiry**: Auto-release after 10 minutes using TTL
- **Optimistic locking**: Version field to prevent race conditions

### 7.4 Payment Reliability
- **Idempotency keys** prevent duplicate charges
- **Webhook reconciliation** handles async payment updates
- **Background jobs** retry failed payments and sync status

### 7.5 Search Optimization
- **Elasticsearch** for full-text and geo-spatial search
- **Autocomplete** using edge n-grams and prefix queries
- **Faceted search** for filters (category, price, rating)

### 7.6 Scalability Patterns
- **Microservices architecture** for independent scaling
- **Event-driven communication** using message queues (Kafka)
- **CQRS pattern** separate read/write models for booking

### 7.7 Availability Patterns
- **Circuit breaker** for payment gateway failures
- **Fallback mechanisms** for search and recommendations
- **Graceful degradation** during peak load (disable non-critical features)

### 7.8 Performance Optimization
- **Connection pooling** for database connections
- **Batch processing** for bulk operations (seat updates)
- **Async processing** for notifications and analytics

### 7.9 Security Measures
- **Rate limiting** per user/IP to prevent abuse
- **Token-based auth** (JWT) with refresh tokens
- **PCI compliance** for payment data (never store CVV)

### 7.10 Monitoring & Observability
- **Distributed tracing** (Jaeger/Zipkin) for request flow
- **Metrics collection** (Prometheus) for system health
- **Log aggregation** (ELK stack) for debugging

---

## 8. Design Patterns Used

### 8.1 Creational Patterns

**Factory Pattern**
- Used in: Creating different event types (Movie, Concert, Sports)
- Benefit: Encapsulates object creation logic

**Builder Pattern**
- Used in: Building complex queries (SearchQuery, BookingRequest)
- Benefit: Readable and flexible object construction

**Singleton Pattern**
- Used in: Database connection pool, Redis client
- Benefit: Single shared instance for resources

### 8.2 Structural Patterns

**Adapter Pattern**
- Used in: Payment gateway integration (different gateway APIs)
- Benefit: Unified interface for multiple gateways

**Facade Pattern**
- Used in: BookingFacade simplifies complex booking flow
- Benefit: Single interface for multiple subsystems

**Proxy Pattern**
- Used in: Caching proxy for database queries
- Benefit: Transparent caching layer

### 8.3 Behavioral Patterns

**Strategy Pattern**
- Used in: Pricing strategies (dynamic, static, promotional)
- Benefit: Runtime selection of pricing algorithm

**Observer Pattern**
- Used in: Event-driven notifications and webhooks
- Benefit: Loose coupling between components

**Command Pattern**
- Used in: Booking operations with undo capability (cancellation)
- Benefit: Encapsulates actions as objects

**State Pattern**
- Used in: Booking state transitions (Pending → Confirmed → Cancelled)
- Benefit: Clean state management

**Chain of Responsibility**
- Used in: Payment processing pipeline (validation → charge → confirm)
- Benefit: Flexible request processing chain

**Template Method**
- Used in: Common booking flow with category-specific steps
- Benefit: Code reuse with customization points

---

## 9. Data Structures Used

### 9.1 Core Data Structures

**HashMap/Map**
- Used for: In-memory caching, seat availability lookup (O(1) access)
- Example: `Map<UUID, ShowSeat>` for fast seat lookup

**TreeMap/SortedMap**
- Used for: Time-based show listings, price ranges
- Benefit: Sorted order for range queries

**HashSet/Set**
- Used for: Unique seat IDs, user wishlists, applied promo codes
- Benefit: O(1) membership testing

**ArrayList/List**
- Used for: Seat lists, booking history, search results
- Benefit: Fast iteration and indexed access

**LinkedList**
- Used for: Queue of pending bookings, notification queue
- Benefit: O(1) insertion/deletion at ends

**Priority Queue (Heap)**
- Used for: Processing bookings by priority (VIP users, expiry time)
- Benefit: O(log n) insertion with priority ordering

**Trie**
- Used for: Autocomplete suggestions in search
- Benefit: Efficient prefix matching

**Graph**
- Used for: Venue seat layout representation
- Benefit: Model relationships between adjacent seats

**Bitmap**
- Used for: Compact seat availability representation
- Benefit: Memory-efficient (1 bit per seat)

**B+ Tree (Database Index)**
- Used for: Database indexes on showId, eventId, userId
- Benefit: Fast range queries and lookups

---

## 10. Database Schema

```sql
-- Users
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_email (email),
    INDEX idx_phone (phone)
);

-- Events
CREATE TABLE events (
    event_id UUID PRIMARY KEY,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    category VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL,
    duration_minutes INT,
    start_date DATE,
    end_date DATE,
    poster_url VARCHAR(500),
    average_rating DECIMAL(3,2),
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_category (category),
    INDEX idx_dates (start_date, end_date),
    INDEX idx_status (status)
);

-- Venues
CREATE TABLE venues (
    venue_id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    address TEXT NOT NULL,
    city VARCHAR(100) NOT NULL,
    latitude DECIMAL(10,8),
    longitude DECIMAL(11,8),
    type VARCHAR(50),
    total_capacity INT,
    is_premium BOOLEAN DEFAULT FALSE,
    INDEX idx_city (city),
    INDEX idx_location (latitude, longitude)
);

-- Sections (Screens/Areas)
CREATE TABLE sections (
    section_id UUID PRIMARY KEY,
    venue_id UUID NOT NULL,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(50),
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
    is_active BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (section_id) REFERENCES sections(section_id),
    UNIQUE KEY uk_section_seat (section_id, seat_number),
    INDEX idx_section (section_id)
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
    booked_seats INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (event_id) REFERENCES events(event_id),
    FOREIGN KEY (section_id) REFERENCES sections(section_id),
    INDEX idx_event_time (event_id, start_time),
    INDEX idx_section_time (section_id, start_time)
);

-- Show Seats (availability per show)
CREATE TABLE show_seats (
    show_seat_id UUID PRIMARY KEY,
    show_id UUID NOT NULL,
    seat_id UUID NOT NULL,
    status VARCHAR(20) DEFAULT 'AVAILABLE',
    current_price DECIMAL(10,2),
    locked_by_user_id UUID,
    lock_expiry TIMESTAMP,
    version BIGINT DEFAULT 0,
    updated_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (show_id) REFERENCES shows(show_id),
    FOREIGN KEY (seat_id) REFERENCES seats(seat_id),
    FOREIGN KEY (locked_by_user_id) REFERENCES users(user_id),
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
    convenience_fee DECIMAL(10,2),
    taxes DECIMAL(10,2),
    discount DECIMAL(10,2) DEFAULT 0,
    promo_code VARCHAR(50),
    qr_code VARCHAR(500),
    booked_at TIMESTAMP DEFAULT NOW(),
    expiry_time TIMESTAMP,
    confirmed_at TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (show_id) REFERENCES shows(show_id),
    INDEX idx_user (user_id),
    INDEX idx_show (show_id),
    INDEX idx_status (status),
    INDEX idx_expiry (expiry_time)
);

-- Booking Seats (junction)
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
    gateway_reference VARCHAR(255),
    idempotency_key VARCHAR(255) UNIQUE,
    retry_count INT DEFAULT 0,
    initiated_at TIMESTAMP DEFAULT NOW(),
    processed_at TIMESTAMP,
    failure_reason TEXT,
    FOREIGN KEY (booking_id) REFERENCES bookings(booking_id),
    INDEX idx_booking (booking_id),
    INDEX idx_idempotency (idempotency_key),
    INDEX idx_transaction (transaction_id),
    INDEX idx_status (status)
);

-- Promo Codes
CREATE TABLE promo_codes (
    promo_id UUID PRIMARY KEY,
    code VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    discount_type VARCHAR(20) NOT NULL,
    value DECIMAL(10,2) NOT NULL,
    min_booking_amount DECIMAL(10,2),
    max_discount DECIMAL(10,2),
    valid_from TIMESTAMP NOT NULL,
    valid_until TIMESTAMP NOT NULL,
    max_usage_count INT,
    current_usage_count INT DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    INDEX idx_code (code),
    INDEX idx_validity (valid_from, valid_until)
);
```

---

## 11. Key Takeaways

### Critical Design Decisions

1. **Seat Locking**: Two-phase (Redis + DB) ensures no double booking
2. **Payment Idempotency**: Prevents duplicate charges with idempotency keys
3. **Dynamic Pricing**: Revenue optimization based on demand and occupancy
4. **Microservices**: Independent scaling of booking vs search workloads
5. **Event-Driven**: Async processing for notifications and analytics

### Scalability Strategies

- Horizontal scaling of stateless services
- Database sharding by region/city
- Read replicas for query-heavy operations
- Caching at multiple layers (CDN, Redis, Application)

### Reliability Measures

- Circuit breakers for external dependencies
- Webhook + reconciliation for payment confirmation
- Background jobs for expired lock cleanup
- Automated refund processing

### Performance Optimizations

- Elasticsearch for sub-second search
- Redis for seat availability (< 200ms)
- Connection pooling and batch operations
- CDN for static content delivery

This design handles 10M+ users, prevents common booking issues (double booking, payment failures), and scales horizontally for peak traffic!
