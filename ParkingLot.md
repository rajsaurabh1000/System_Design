# Parking Lot System - System Design

## 1. Functional Requirements (FR)

### Core Features
1. **Vehicle Entry**
   - Issue parking ticket at entry gate
   - Capture vehicle details (license plate, type)
   - Assign parking spot
   - Open entry gate automatically

2. **Vehicle Exit**
   - Scan ticket/license plate at exit
   - Calculate parking fee
   - Process payment (cash, card, mobile)
   - Open exit gate
   - Free up parking spot

3. **Parking Spot Management**
   - Multiple spot types (compact, regular, large, handicapped, electric)
   - Real-time availability tracking
   - Spot assignment algorithms
   - Reserved spots for specific users

4. **Payment System**
   - Hourly/daily pricing
   - Different rates for spot types
   - Multiple payment methods
   - Digital receipts

5. **Admin Operations**
   - Monitor occupancy in real-time
   - Generate reports (revenue, usage)
   - Manage pricing
   - Handle disputes and overrides

6. **User Features (Optional)**
   - Pre-booking via app
   - Find available spots
   - Navigate to assigned spot
   - Payment history

### Out of Scope
- Valet parking
- Car wash services
- EV charging management (can be extension)
- Security/surveillance integration

## 2. Non-Functional Requirements (NFR)

1. **Performance**
   - Entry/exit processing: < 5 seconds
   - Spot assignment: < 1 second
   - Payment processing: < 10 seconds
   - Real-time updates: < 1 second

2. **Scalability**
   - Support parking lots with 10,000+ spots
   - Handle 1000 entries/exits per hour during peak
   - Multiple parking lots per system

3. **Availability**
   - 99.99% uptime
   - Entry/exit gates must work even if network fails (degraded mode)
   - Local caching for critical operations

4. **Reliability**
   - No double-booking of spots
   - Accurate payment calculation
   - Data durability (no lost transactions)

5. **Consistency**
   - Strong consistency for spot allocation
   - Eventual consistency for analytics

6. **Security**
   - Secure payment processing (PCI DSS)
   - Audit logs for all transactions
   - Access control for admin operations

## 3. Core Entities

### 3.1 ParkingLot
```
ParkingLot {
  lotId: UUID
  name: string
  address: string
  totalFloors: int
  totalSpots: int
  availableSpots: int
  entryGates: List<EntryGate>
  exitGates: List<ExitGate>
  pricingConfig: PricingConfig
  createdAt: timestamp
}
```

### 3.2 ParkingFloor
```
ParkingFloor {
  floorId: UUID
  lotId: UUID
  floorNumber: int
  spots: List<ParkingSpot>
  displayBoard: DisplayBoard
}
```

### 3.3 ParkingSpot
```
ParkingSpot {
  spotId: UUID
  floorId: UUID
  spotNumber: string
  type: enum (COMPACT, REGULAR, LARGE, HANDICAPPED, ELECTRIC)
  status: enum (AVAILABLE, OCCUPIED, RESERVED, OUT_OF_SERVICE)
  vehicle: Vehicle (nullable)
  assignedAt: timestamp (nullable)
}
```

### 3.4 Vehicle
```
Vehicle {
  vehicleId: UUID
  licensePlate: string
  type: enum (MOTORCYCLE, CAR, SUV, TRUCK, VAN)
  color: string
  ownerName: string (optional)
}
```

### 3.5 ParkingTicket
```
ParkingTicket {
  ticketId: UUID
  ticketNumber: string
  lotId: UUID
  spotId: UUID
  vehicleId: UUID
  entryTime: timestamp
  exitTime: timestamp (nullable)
  status: enum (ACTIVE, PAID, LOST, CANCELLED)
  amount: decimal (nullable)
  paymentMethod: string (nullable)
  paidAt: timestamp (nullable)
}
```

### 3.6 Payment
```
Payment {
  paymentId: UUID
  ticketId: UUID
  amount: decimal
  paymentMethod: enum (CASH, CREDIT_CARD, DEBIT_CARD, MOBILE_PAY)
  status: enum (PENDING, COMPLETED, FAILED, REFUNDED)
  transactionId: string
  processedAt: timestamp
}
```

### 3.7 PricingConfig
```
PricingConfig {
  configId: UUID
  lotId: UUID
  baseRate: decimal (per hour)
  spotTypePricing: Map<SpotType, decimal>
  dailyMaxRate: decimal
  validFrom: timestamp
  validTo: timestamp
}
```

### 3.8 EntryGate / ExitGate
```
Gate {
  gateId: UUID
  lotId: UUID
  gateNumber: string
  type: enum (ENTRY, EXIT)
  status: enum (OPEN, CLOSED, OUT_OF_SERVICE)
  lastUsedAt: timestamp
}
```

### 3.9 DisplayBoard
```
DisplayBoard {
  boardId: UUID
  floorId: UUID
  availableSpots: Map<SpotType, int>
  lastUpdated: timestamp
}
```

## 4. API Design

### 4.1 Entry/Exit APIs
```
POST /api/v1/parking/entry
Request: {
  lotId: UUID,
  gateId: UUID,
  vehicleType: enum,
  licensePlate: string
}
Response: {
  ticketId: UUID,
  ticketNumber: string,
  assignedSpot: {
    spotNumber: string,
    floor: int,
    directions: string
  },
  entryTime: timestamp
}

POST /api/v1/parking/exit
Request: {
  ticketNumber: string,
  exitGateId: UUID
}
Response: {
  ticketId: UUID,
  parkingDuration: int (minutes),
  amount: decimal,
  paymentDue: boolean
}

POST /api/v1/parking/payment
Request: {
  ticketId: UUID,
  paymentMethod: enum,
  amount: decimal,
  paymentDetails: {...}
}
Response: {
  paymentId: UUID,
  status: enum,
  receiptUrl: string
}
```

### 4.2 Availability APIs
```
GET /api/v1/parking/lots/{lotId}/availability
Response: {
  lotId: UUID,
  totalSpots: int,
  availableSpots: int,
  spotsByType: {
    COMPACT: { total: int, available: int },
    REGULAR: { total: int, available: int },
    LARGE: { total: int, available: int },
    HANDICAPPED: { total: int, available: int },
    ELECTRIC: { total: int, available: int }
  },
  floorAvailability: [
    {
      floor: int,
      availableSpots: int
    }
  ]
}

GET /api/v1/parking/spots/find
Query: { lotId: UUID, vehicleType: enum }
Response: {
  availableSpots: [
    {
      spotId: UUID,
      spotNumber: string,
      floor: int,
      type: enum,
      distance: int (meters from entry)
    }
  ]
}
```

### 4.3 Admin APIs
```
GET /api/v1/admin/parking/lots/{lotId}/occupancy
Headers: { Authorization: Bearer <admin-token> }
Response: {
  currentOccupancy: int,
  occupancyRate: float,
  peakHours: [...],
  floorWiseOccupancy: [...]
}

GET /api/v1/admin/parking/lots/{lotId}/revenue
Headers: { Authorization: Bearer <admin-token> }
Query: { startDate, endDate }
Response: {
  totalRevenue: decimal,
  totalVehicles: int,
  averageParking Duration: int,
  revenueByDay: [...]
}

PUT /api/v1/admin/parking/spots/{spotId}
Headers: { Authorization: Bearer <admin-token> }
Request: {
  status: enum (AVAILABLE, OUT_OF_SERVICE)
}
Response: { success: boolean }

POST /api/v1/admin/parking/pricing
Headers: { Authorization: Bearer <admin-token> }
Request: {
  lotId: UUID,
  baseRate: decimal,
  spotTypePricing: {...},
  validFrom: timestamp
}
Response: { configId: UUID }
```

### 4.4 User App APIs (Optional)
```
POST /api/v1/user/parking/reserve
Headers: { Authorization: Bearer <token> }
Request: {
  lotId: UUID,
  vehicleType: enum,
  startTime: timestamp,
  duration: int (hours)
}
Response: {
  reservationId: UUID,
  spotNumber: string,
  qrCode: string
}

GET /api/v1/user/parking/history
Headers: { Authorization: Bearer <token> }
Response: {
  parkingHistory: [
    {
      ticketId, lotName, entryTime, exitTime,
      duration, amount, spotNumber
    }
  ]
}
```

## 5. High-Level Design (HLD)

### 5.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                  PHYSICAL LAYER                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Entry    │  │  Exit    │  │ Display  │  │ Payment  │   │
│  │ Terminal │  │ Terminal │  │  Board   │  │ Terminal │   │
│  │ (Camera, │  │ (Camera, │  │ (LED)    │  │ (POS)    │   │
│  │  RFID)   │  │  Barrier)│  │          │  │          │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
└───────┼─────────────┼─────────────┼─────────────┼──────────┘
        │             │             │             │
        └─────────────┼─────────────┼─────────────┘
                      │             │
              ┌───────▼─────────────▼────┐
              │   Edge Controllers       │
              │   (Local Processing)     │
              └───────┬──────────────────┘
                      │
                      │ Internet
                      │
              ┌───────▼──────────────────┐
              │   Load Balancer          │
              └───────┬──────────────────┘
                      │
┌─────────────────────┼──────────────────────────────────────┐
│              APPLICATION LAYER                              │
│         ┌───────────▼──────────────┐                        │
│         │     API Gateway          │                        │
│         └───────────┬──────────────┘                        │
│                     │                                       │
│    ┌────────────────┼────────────────┐                     │
│    │                │                │                     │
│ ┌──▼──────┐  ┌──────▼─────┐  ┌──────▼─────┐               │
│ │ Parking │  │  Payment   │  │   Admin    │               │
│ │ Service │  │  Service   │  │  Service   │               │
│ └──┬──────┘  └──────┬─────┘  └──────┬─────┘               │
│    │                │                │                     │
│ ┌──▼──────┐  ┌──────▼─────┐  ┌──────▼─────┐               │
│ │  Spot   │  │ Analytics  │  │ Notification│              │
│ │Assignment│ │  Service   │  │  Service   │               │
│ │ Service │  └────────────┘  └────────────┘               │
│ └─────────┘                                                │
└────────────────────┬───────────────────────────────────────┘
                     │
┌────────────────────┼───────────────────────────────────────┐
│              DATA LAYER                                     │
│    ┌───────────────▼────────┐  ┌──────────────┐            │
│    │      PostgreSQL        │  │    Redis     │            │
│    │  (Tickets, Spots,      │  │  (Cache,     │            │
│    │   Payments, Config)    │  │  Real-time   │            │
│    └────────────────────────┘  │  Availability│            │
│                                └──────────────┘            │
│    ┌────────────────────────┐  ┌──────────────┐            │
│    │      TimescaleDB       │  │    Kafka     │            │
│    │  (Time-series data,    │  │  (Event      │            │
│    │   Analytics)           │  │   Stream)    │            │
│    └────────────────────────┘  └──────────────┘            │
└────────────────────────────────────────────────────────────┘
```

### 5.2 Component Descriptions

**Physical Layer:**
- **Entry Terminal**: License plate camera, RFID reader, ticket printer, barrier
- **Exit Terminal**: Ticket scanner, payment terminal, barrier
- **Display Board**: LED displays showing available spots per floor
- **Payment Terminal**: POS system at exit or payment stations

**Edge Controllers:**
- Local processing for gate operations
- Offline capability (critical for entry/exit)
- Syncs with cloud when online

**Application Layer:**
- **Parking Service**: Core parking operations (entry, exit, spot management)
- **Spot Assignment Service**: Algorithms for optimal spot allocation
- **Payment Service**: Payment processing integration
- **Admin Service**: Management and reporting
- **Analytics Service**: Usage patterns, occupancy trends
- **Notification Service**: SMS/email for reservations, receipts

**Data Layer:**
- **PostgreSQL**: Transactional data (tickets, spots, payments)
- **Redis**: Real-time availability, session management, caching
- **TimescaleDB**: Time-series analytics data
- **Kafka**: Event streaming for real-time updates

### 5.3 Vehicle Entry Flow

```
┌────────┐      ┌───────┐      ┌────────┐      ┌────────┐
│ Entry  │      │ Edge  │      │Parking │      │  Spot  │
│Terminal│      │Ctrl   │      │Service │      │Assign  │
└───┬────┘      └───┬───┘      └───┬────┘      └───┬────┘
    │               │              │               │
    │ Scan License  │              │               │
    │ Plate         │              │               │
    ├──────────────>│              │               │
    │               │              │               │
    │               │ Check        │               │
    │               │ Availability │               │
    │               ├─────────────>│               │
    │               │              │ Find Spot     │
    │               │              ├──────────────>│
    │               │              │               │
    │               │              │ Optimal Spot  │
    │               │              │<──────────────┤
    │               │              │               │
    │               │ Reserve Spot │               │
    │               │<─────────────┤               │
    │               │              │               │
    │               │ Create Ticket│               │
    │               ├─────────────>│               │
    │               │              │               │
    │ Print Ticket  │              │               │
    │<──────────────┤              │               │
    │               │              │               │
    │ Open Barrier  │              │               │
    │<──────────────┤              │               │
    │               │              │               │
```

### 5.4 Spot Assignment Algorithm

```
Decision Tree for Spot Assignment:

1. Filter by vehicle type:
   - Motorcycle -> COMPACT spots
   - Car -> COMPACT or REGULAR
   - SUV -> REGULAR or LARGE
   - Truck -> LARGE

2. Filter by availability:
   - Only AVAILABLE spots

3. Apply preferences:
   - Handicapped users -> HANDICAPPED spots (priority)
   - Electric vehicles -> ELECTRIC spots (if available)

4. Optimization criteria (choose best):
   a) Nearest to entry/exit
   b) Lowest floor (for quick exit)
   c) Fill floors sequentially (energy efficiency)
   d) Balanced distribution (even wear)

5. Return assigned spot

Algorithm:
┌──────────────────────────────────────────┐
│ Spot Assignment Strategy: NEAREST_FIRST  │
└──────────────────────────────────────────┘
```

### 5.5 Payment Calculation

```
Calculate Parking Fee:

Input: Entry Time, Exit Time, Spot Type
Output: Amount Due

Formula:
  duration = exit_time - entry_time (in hours, rounded up)
  base_rate = pricing_config.base_rate
  type_multiplier = pricing_config.spot_type_pricing[spot_type]
  
  amount = duration * base_rate * type_multiplier

Example:
  Entry: 9:00 AM
  Exit: 3:30 PM
  Duration: 6.5 hours -> 7 hours (rounded up)
  Base Rate: $5/hour
  Spot Type: REGULAR (1.0x multiplier)
  
  Amount = 7 * $5 * 1.0 = $35
```

## 6. Optimizations & Approaches

### 6.1 Concurrent Spot Assignment (Race Condition)

**Problem**: Multiple vehicles trying to book same spot simultaneously

**Solution**: Optimistic locking with database constraints

**Alternative**: Use Redis distributed lock

### 6.2 Real-time Availability Updates

**Problem**: Display boards need real-time updates
**Solution**: Event-driven architecture with pub/sub

### 6.3 Caching Strategy

**Multi-Level Cache**:
1. **Redis**: Real-time availability per floor/type
2. **Local Cache** (Entry terminals): Recent spot assignments
3. **CDN**: Static resources for mobile app


### 6.4 Offline Mode (Edge Computing)

**Problem**: Internet connection failure shouldn't stop parking operations

**Solution**: Edge controllers with local database
- Maintain local copy of spot availability
- Issue tickets locally (offline queue)
- Sync to cloud when connection restored
- Use sequential ticket IDs with prefix (ENTRY_GATE_1_0001)

### 6.5 Payment Processing

**Integration**: Stripe, PayPal, or Payment Gateway
**Security**: PCI DSS compliance
**Flow**:
```
1. Calculate amount
2. Create payment intent
3. Process payment via gateway
4. Receive callback/webhook
5. Update ticket status
6. Open exit gate
7. Send receipt
```

### 6.6 Analytics Optimization

**Time-Series Data**: Use TimescaleDB for efficient queries

### 6.7 Dynamic Pricing (Surge Pricing)

**Concept**: Increase prices during peak hours or high occupancy

### 6.8 Load Balancing for Entry/Exit

**Problem**: Uneven load across gates
**Solution**: 
- Direct vehicles to less busy gates via app/signage
- Monitor queue length at each gate
- Predictive routing based on historical patterns

### 6.9 Spot Reservation Timeout

**Problem**: Reserved spots not used
**Solution**: 
- Reservation expires after 15 minutes
- Auto-release if vehicle doesn't arrive
- Grace period notification

### 6.10 Lost Ticket Handling

**Scenario**: Customer loses parking ticket
**Solution**:
- Charge maximum daily rate (policy)
- Or lookup by license plate (if captured)
- Admin override capability

## 7. Low-Level Design (LLD)

### 7.1 Class Diagram

```
┌──────────────────────────────────────────────────────────┐
│                    ParkingLot                            │
├──────────────────────────────────────────────────────────┤
│ - lotId: UUID                                            │
│ - name: string                                           │
│ - floors: List<ParkingFloor>                             │
│ - entryGates: List<EntryGate>                            │
│ - exitGates: List<ExitGate>                              │
│ - pricingConfig: PricingConfig                           │
├──────────────────────────────────────────────────────────┤
│ + getAvailableSpots(): Map<SpotType, int>                │
│ + findSpot(vehicleType: VehicleType): ParkingSpot        │
│ + processEntry(vehicle: Vehicle): ParkingTicket          │
│ + processExit(ticket: ParkingTicket): Payment            │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                   ParkingFloor                           │
├──────────────────────────────────────────────────────────┤
│ - floorId: UUID                                          │
│ - floorNumber: int                                       │
│ - spots: List<ParkingSpot>                               │
│ - displayBoard: DisplayBoard                             │
├──────────────────────────────────────────────────────────┤
│ + getAvailableSpots(type: SpotType): List<ParkingSpot>   │
│ + updateDisplayBoard(): void                             │
│ + getOccupancyRate(): float                              │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                   ParkingSpot                            │
├──────────────────────────────────────────────────────────┤
│ - spotId: UUID                                           │
│ - spotNumber: string                                     │
│ - type: SpotType                                         │
│ - status: SpotStatus                                     │
│ - vehicle: Vehicle                                       │
├──────────────────────────────────────────────────────────┤
│ + isAvailable(): boolean                                 │
│ + assign(vehicle: Vehicle): void                         │
│ + release(): void                                        │
│ + canFitVehicle(vehicleType: VehicleType): boolean       │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                     Vehicle                              │
├──────────────────────────────────────────────────────────┤
│ - vehicleId: UUID                                        │
│ - licensePlate: string                                   │
│ - type: VehicleType                                      │
│ - color: string                                          │
├──────────────────────────────────────────────────────────┤
│ + getRequiredSpotType(): SpotType                        │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                  ParkingTicket                           │
├──────────────────────────────────────────────────────────┤
│ - ticketId: UUID                                         │
│ - ticketNumber: string                                   │
│ - vehicle: Vehicle                                       │
│ - spot: ParkingSpot                                      │
│ - entryTime: Instant                                     │
│ - exitTime: Instant                                      │
│ - status: TicketStatus                                   │
├──────────────────────────────────────────────────────────┤
│ + getDuration(): Duration                                │
│ + calculateFee(pricing: PricingConfig): Money            │
│ + markPaid(payment: Payment): void                       │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│              SpotAssignmentStrategy                      │
│                 <<interface>>                            │
├──────────────────────────────────────────────────────────┤
│ + findSpot(vehicleType, available): ParkingSpot          │
└──────────────────────────────────────────────────────────┘
           △                        △
           │                        │
    ┌──────┴────────┐        ┌─────┴──────────┐
    │   Nearest     │        │  Sequential    │
    │   Strategy    │        │    Strategy    │
    └───────────────┘        └────────────────┘

┌──────────────────────────────────────────────────────────┐
│                  ParkingService                          │
├──────────────────────────────────────────────────────────┤
│ - parkingLot: ParkingLot                                 │
│ - spotAssignment: SpotAssignmentStrategy                 │
│ - paymentService: PaymentService                         │
├──────────────────────────────────────────────────────────┤
│ + processEntry(vehicle: Vehicle): ParkingTicket          │
│ + processExit(ticketNumber: string): ExitResult          │
│ + getAvailability(): AvailabilityInfo                    │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                  PaymentService                          │
├──────────────────────────────────────────────────────────┤
│ - paymentGateway: PaymentGateway                         │
│ - pricingCalculator: PricingCalculator                   │
├──────────────────────────────────────────────────────────┤
│ + calculateFee(ticket: ParkingTicket): Money             │
│ + processPayment(amount, method): Payment                │
│ + generateReceipt(payment: Payment): Receipt             │
└──────────────────────────────────────────────────────────┘
```

## Design Patterns Used

| Pattern | Where Used | Purpose |
|--------|------------|---------|
| **Strategy** | SpotAssignmentStrategy (Nearest, Sequential, Priority) | Plug different spot-allocation algorithms |
| **Factory** | PaymentFactory, SpotStrategyFactory | Create objects without exposing instantiation logic |
| **Singleton** | ParkingService, PricingConfigManager | Ensure single coordinator instance |
| **Facade** | ParkingService | Unified interface for entry/exit workflows |
| **Command** | ParkingTicket, Payment | Encapsulate user actions |
| **State** | TicketStatus, SpotStatus, GateState | Controlled state transitions |
| **Observer** | DisplayBoard, AnalyticsService | Push real-time updates |
| **Adapter** | PaymentGatewayAdapter | Integrate external payment providers |
| **Template Method** | BaseSpotAssignmentStrategy | Shared algorithm skeleton |
| **Repository** | ParkingRepository, TicketRepository | Abstract persistence |
| **DTO** | Request/Response objects | Data transport only |
| **Builder (optional)** | ParkingTicketBuilder | Construct complex objects safely |


## Data Structures Summary

| Component | Data Structure | Reason |
|----------|----------------|--------|
| Parking floors | List<ParkingFloor> | Ordered traversal |
| Parking spots per floor | Map<SpotType, Set<ParkingSpot>> | Fast lookup by type |
| Available spots | TreeSet / PriorityQueue | Nearest / ordered selection |
| Occupied spots | HashMap<SpotId, ParkingSpot> | O(1) lookup |
| Tickets | HashMap<TicketId, ParkingTicket> | Fast retrieval |
| Active tickets | HashMap<VehicleId, Ticket> | Prevent duplicates |
| Requests queue | Queue<Request> | FIFO processing |
| Pricing rules | Map<SpotType, BigDecimal> | Fast rate lookup |
| Payments | List / Map | Audit & reporting |
| Display cache | ConcurrentHashMap | Thread-safe reads |
| Event subscribers | List<Observer> | Publish-subscribe |
| State enums | Enum | Safe transitions |
| Time-series data | Append-only list / DB | Analytics |


### 7.4 Scalability Calculations

**Parking Lot Size**: 2000 spots across 5 floors
**Traffic**: 1000 entries/exits per hour (peak)
**Database Load**:
- 1000 inserts/hour for tickets
- 1000 updates/hour for spot status
- Manageable with PostgreSQL

**Redis Memory**:
- Availability data: ~1KB per lot
- 100 lots = 100KB
- Minimal memory footprint

**Concurrent Users**: 10 entry gates, 10 exit gates
- 20 concurrent operations max
- No scaling issues

**Analytics Data**:
- 1 entry per minute = 1440 records/day
- 1 year = 525,600 records
- TimescaleDB handles this efficiently

---

## Interview Discussion Points

### Clarifying Questions:
1. Size: How many spots? Floors?
2. Traffic: Peak hour volume?
3. Payment: Online only or cash?
4. Features: Reservations? App integration?
5. Multiple lots: Centralized system?

### Trade-offs:
1. **Real-time vs Batch**: Availability updates
2. **Consistency vs Performance**: Spot assignment
3. **Complexity vs Features**: Simple vs dynamic pricing

### Extensions:
1. **Valet Parking**: Key management, car retrieval
2. **EV Charging**: Charging station integration
3. **IoT Sensors**: Real-time spot detection
4. **Mobile App**: Navigation, reservations
5. **License Plate Recognition**: Camera-based entry

This design provides a comprehensive parking lot management system that's scalable, reliable, and handles real-world scenarios!

