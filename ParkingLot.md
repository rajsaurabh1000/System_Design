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

def assign_spot(vehicle_type, preferences):
    # Get compatible spots
    compatible_spots = filter_by_vehicle_type(vehicle_type)
    
    # Check handicapped
    if preferences.is_handicapped:
        handicapped_spots = filter_by_type(compatible_spots, HANDICAPPED)
        if handicapped_spots:
            return find_nearest(handicapped_spots)
    
    # Check electric
    if preferences.is_electric:
        electric_spots = filter_by_type(compatible_spots, ELECTRIC)
        if electric_spots:
            return find_nearest(electric_spots)
    
    # Find nearest available spot
    available_spots = filter_by_status(compatible_spots, AVAILABLE)
    return find_nearest(available_spots)
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
  
  # Apply daily max cap
  if amount > pricing_config.daily_max_rate:
      amount = pricing_config.daily_max_rate
  
  return amount

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
```sql
-- Atomic update with version check
UPDATE parking_spots
SET status = 'OCCUPIED',
    vehicle_id = ?,
    version = version + 1
WHERE spot_id = ?
  AND status = 'AVAILABLE'
  AND version = ?;

-- If rows_affected = 0, spot was taken, retry
```

**Alternative**: Use Redis distributed lock
```python
def assign_spot_with_lock(spot_id, vehicle_id):
    lock_key = f"spot_lock:{spot_id}"
    lock = redis.set(lock_key, "locked", nx=True, ex=10)
    
    if not lock:
        return False  # Spot being assigned by another process
    
    try:
        # Assign spot
        update_spot(spot_id, vehicle_id)
        return True
    finally:
        redis.delete(lock_key)
```

### 6.2 Real-time Availability Updates

**Problem**: Display boards need real-time updates
**Solution**: Event-driven architecture with pub/sub

```python
# On spot status change
def update_spot_status(spot_id, new_status):
    # Update database
    db.update_spot(spot_id, new_status)
    
    # Publish event
    kafka.publish('spot-updates', {
        'spot_id': spot_id,
        'floor_id': floor_id,
        'status': new_status,
        'timestamp': now()
    })
    
    # Update Redis cache
    redis.hincrby(f"floor:{floor_id}:availability", spot_type, delta)

# Display board subscribes to events
def handle_spot_update(event):
    floor_availability = redis.hgetall(f"floor:{event.floor_id}:availability")
    update_led_display(floor_availability)
```

### 6.3 Caching Strategy

**Multi-Level Cache**:
1. **Redis**: Real-time availability per floor/type
2. **Local Cache** (Entry terminals): Recent spot assignments
3. **CDN**: Static resources for mobile app

**Cache Structure**:
```
# Floor availability (Hash)
floor:1:availability = {
  "COMPACT": 50,
  "REGULAR": 100,
  "LARGE": 20,
  "HANDICAPPED": 10,
  "ELECTRIC": 15
}

# Active tickets (Set)
active_tickets = {ticketId1, ticketId2, ...}

# Spot status (String)
spot:spot123:status = "OCCUPIED"
```

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
```sql
-- Create hypertable for occupancy data
CREATE TABLE occupancy_data (
    time TIMESTAMPTZ NOT NULL,
    lot_id UUID NOT NULL,
    floor_id UUID,
    spot_type VARCHAR(20),
    occupied_count INT,
    available_count INT
);

SELECT create_hypertable('occupancy_data', 'time');

-- Query: Peak hours in last month
SELECT 
    time_bucket('1 hour', time) AS hour,
    AVG(occupied_count) as avg_occupancy
FROM occupancy_data
WHERE lot_id = ?
  AND time > NOW() - INTERVAL '30 days'
GROUP BY hour
ORDER BY hour;
```

### 6.7 Dynamic Pricing (Surge Pricing)

**Concept**: Increase prices during peak hours or high occupancy

```python
def calculate_dynamic_price(base_rate, current_occupancy_rate, time_of_day):
    price = base_rate
    
    # Occupancy surge
    if current_occupancy_rate > 0.9:
        price *= 1.5
    elif current_occupancy_rate > 0.7:
        price *= 1.2
    
    # Time-based surge (rush hours)
    if is_rush_hour(time_of_day):
        price *= 1.3
    
    return price
```

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

### 7.2 Detailed Implementation

#### 7.2.1 Parking Service

```java
@Service
public class ParkingService {
    private final ParkingLotRepository parkingLotRepo;
    private final ParkingSpotRepository spotRepo;
    private final ParkingTicketRepository ticketRepo;
    private final SpotAssignmentStrategy assignmentStrategy;
    private final RedisTemplate<String, String> redis;
    private final EventPublisher eventPublisher;
    
    @Transactional
    public ParkingTicket processEntry(
        UUID lotId,
        String licensePlate,
        VehicleType vehicleType
    ) {
        // 1. Check if parking lot has capacity
        int availableSpots = getAvailableSpotsCount(lotId);
        if (availableSpots == 0) {
            throw new ParkingFullException("No available spots");
        }
        
        // 2. Create or get vehicle
        Vehicle vehicle = vehicleRepo.findByLicensePlate(licensePlate)
            .orElseGet(() -> {
                Vehicle v = new Vehicle(UUID.randomUUID(), licensePlate, vehicleType);
                return vehicleRepo.save(v);
            });
        
        // 3. Find suitable parking spot
        ParkingSpot spot = findAndAssignSpot(lotId, vehicleType);
        if (spot == null) {
            throw new NoSuitableSpotException();
        }
        
        // 4. Assign spot to vehicle
        spot.setStatus(SpotStatus.OCCUPIED);
        spot.setVehicle(vehicle);
        spot.setAssignedAt(Instant.now());
        spotRepo.save(spot);
        
        // 5. Create parking ticket
        ParkingTicket ticket = ParkingTicket.builder()
            .ticketId(UUID.randomUUID())
            .ticketNumber(generateTicketNumber())
            .lotId(lotId)
            .spotId(spot.getSpotId())
            .vehicleId(vehicle.getVehicleId())
            .entryTime(Instant.now())
            .status(TicketStatus.ACTIVE)
            .build();
        
        ticketRepo.save(ticket);
        
        // 6. Update cache (availability)
        updateAvailabilityCache(lotId, spot.getType(), -1);
        
        // 7. Publish event
        eventPublisher.publish(new VehicleEnteredEvent(
            lotId, vehicle.getVehicleId(), spot.getSpotId()
        ));
        
        return ticket;
    }
    
    @Transactional
    public ExitResult processExit(String ticketNumber) {
        // 1. Find ticket
        ParkingTicket ticket = ticketRepo.findByTicketNumber(ticketNumber)
            .orElseThrow(() -> new TicketNotFoundException());
        
        if (ticket.getStatus() != TicketStatus.ACTIVE) {
            throw new InvalidTicketException("Ticket already processed");
        }
        
        // 2. Calculate parking fee
        ticket.setExitTime(Instant.now());
        Money fee = calculateParkingFee(ticket);
        
        // 3. Check if payment needed
        if (fee.getAmount() > 0) {
            return new ExitResult(
                ticket.getTicketId(),
                fee,
                true, // payment required
                null
            );
        }
        
        // 4. If free parking or already paid, release spot
        releaseSpot(ticket);
        
        return new ExitResult(
            ticket.getTicketId(),
            fee,
            false,
            "success"
        );
    }
    
    public Payment processPayment(
        UUID ticketId,
        Money amount,
        PaymentMethod method,
        Map<String, String> paymentDetails
    ) {
        ParkingTicket ticket = ticketRepo.findById(ticketId)
            .orElseThrow(() -> new TicketNotFoundException());
        
        // Verify amount
        Money calculatedFee = calculateParkingFee(ticket);
        if (!calculatedFee.equals(amount)) {
            throw new InvalidAmountException();
        }
        
        // Process payment via gateway
        Payment payment = paymentGateway.processPayment(
            amount,
            method,
            paymentDetails
        );
        
        if (payment.getStatus() == PaymentStatus.COMPLETED) {
            // Update ticket
            ticket.setStatus(TicketStatus.PAID);
            ticket.setPaidAt(Instant.now());
            ticketRepo.save(ticket);
            
            // Release spot
            releaseSpot(ticket);
        }
        
        return payment;
    }
    
    private ParkingSpot findAndAssignSpot(UUID lotId, VehicleType vehicleType) {
        // Get available spots for vehicle type
        List<ParkingSpot> availableSpots = spotRepo.findAvailableSpots(
            lotId,
            vehicleType
        );
        
        if (availableSpots.isEmpty()) {
            return null;
        }
        
        // Use assignment strategy to select best spot
        ParkingSpot selectedSpot = assignmentStrategy.selectSpot(
            availableSpots,
            vehicleType
        );
        
        // Try to reserve spot (optimistic locking)
        int updated = spotRepo.updateSpotStatus(
            selectedSpot.getSpotId(),
            SpotStatus.AVAILABLE,
            SpotStatus.OCCUPIED,
            selectedSpot.getVersion()
        );
        
        if (updated == 0) {
            // Spot was taken, retry
            return findAndAssignSpot(lotId, vehicleType);
        }
        
        return selectedSpot;
    }
    
    private Money calculateParkingFee(ParkingTicket ticket) {
        Duration duration = Duration.between(
            ticket.getEntryTime(),
            ticket.getExitTime()
        );
        
        // Round up to nearest hour
        long hours = (long) Math.ceil(duration.toMinutes() / 60.0);
        
        // Get pricing config
        PricingConfig pricing = pricingRepo.findByLotId(ticket.getLotId());
        
        ParkingSpot spot = spotRepo.findById(ticket.getSpotId()).orElseThrow();
        
        // Calculate fee
        BigDecimal baseRate = pricing.getBaseRate();
        BigDecimal typeMultiplier = pricing.getSpotTypePricing()
            .getOrDefault(spot.getType(), BigDecimal.ONE);
        
        BigDecimal amount = baseRate
            .multiply(BigDecimal.valueOf(hours))
            .multiply(typeMultiplier);
        
        // Apply daily max cap
        if (amount.compareTo(pricing.getDailyMaxRate()) > 0) {
            amount = pricing.getDailyMaxRate();
        }
        
        return new Money(amount, "USD");
    }
    
    private void releaseSpot(ParkingTicket ticket) {
        ParkingSpot spot = spotRepo.findById(ticket.getSpotId()).orElseThrow();
        
        // Release spot
        spot.setStatus(SpotStatus.AVAILABLE);
        spot.setVehicle(null);
        spot.setAssignedAt(null);
        spotRepo.save(spot);
        
        // Update cache
        updateAvailabilityCache(ticket.getLotId(), spot.getType(), 1);
        
        // Publish event
        eventPublisher.publish(new VehicleExitedEvent(
            ticket.getLotId(),
            ticket.getVehicleId(),
            spot.getSpotId()
        ));
    }
    
    private void updateAvailabilityCache(
        UUID lotId,
        SpotType spotType,
        int delta
    ) {
        String key = "lot:" + lotId + ":availability";
        redis.opsForHash().increment(key, spotType.name(), delta);
    }
    
    private int getAvailableSpotsCount(UUID lotId) {
        String key = "lot:" + lotId + ":availability";
        Map<Object, Object> availability = redis.opsForHash().entries(key);
        
        return availability.values().stream()
            .mapToInt(v -> Integer.parseInt(v.toString()))
            .sum();
    }
    
    private String generateTicketNumber() {
        // Format: YYYYMMDD-HHMM-XXXX
        String date = LocalDateTime.now().format(
            DateTimeFormatter.ofPattern("yyyyMMdd-HHmm")
        );
        String random = String.format("%04d", 
            ThreadLocalRandom.current().nextInt(10000)
        );
        return date + "-" + random;
    }
}
```

#### 7.2.2 Spot Assignment Strategies

```java
// Strategy Interface
public interface SpotAssignmentStrategy {
    ParkingSpot selectSpot(List<ParkingSpot> availableSpots, VehicleType vehicleType);
}

// Nearest to Entry Strategy
@Component
public class NearestToEntryStrategy implements SpotAssignmentStrategy {
    
    @Override
    public ParkingSpot selectSpot(
        List<ParkingSpot> availableSpots,
        VehicleType vehicleType
    ) {
        // Sort by floor (ascending) and then by distance to entry
        return availableSpots.stream()
            .sorted(Comparator
                .comparingInt((ParkingSpot s) -> s.getFloor().getFloorNumber())
                .thenComparingInt(s -> s.getDistanceToEntry()))
            .findFirst()
            .orElse(null);
    }
}

// Sequential Fill Strategy (energy efficient)
@Component
public class SequentialFillStrategy implements SpotAssignmentStrategy {
    
    @Override
    public ParkingSpot selectSpot(
        List<ParkingSpot> availableSpots,
        VehicleType vehicleType
    ) {
        // Fill one floor at a time
        // Benefit: Can turn off lights/HVAC on empty floors
        
        Map<Integer, List<ParkingSpot>> spotsByFloor = availableSpots.stream()
            .collect(Collectors.groupingBy(s -> s.getFloor().getFloorNumber()));
        
        // Find the floor with most occupied spots
        Integer mostOccupiedFloor = spotsByFloor.keySet().stream()
            .min(Comparator.comparingInt(floor -> {
                return getAvailableCountOnFloor(floor);
            }))
            .orElse(null);
        
        if (mostOccupiedFloor != null) {
            return spotsByFloor.get(mostOccupiedFloor).get(0);
        }
        
        return availableSpots.get(0);
    }
    
    private int getAvailableCountOnFloor(int floorNumber) {
        // Get from cache or database
        return 0; // placeholder
    }
}
```

### 7.3 Database Schema

```sql
-- Parking lots table
CREATE TABLE parking_lots (
    lot_id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    address TEXT NOT NULL,
    total_floors INT NOT NULL,
    total_spots INT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Parking floors table
CREATE TABLE parking_floors (
    floor_id UUID PRIMARY KEY,
    lot_id UUID NOT NULL,
    floor_number INT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    FOREIGN KEY (lot_id) REFERENCES parking_lots(lot_id),
    UNIQUE KEY uk_lot_floor (lot_id, floor_number)
);

-- Parking spots table
CREATE TABLE parking_spots (
    spot_id UUID PRIMARY KEY,
    floor_id UUID NOT NULL,
    spot_number VARCHAR(50) NOT NULL,
    type VARCHAR(20) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'AVAILABLE',
    vehicle_id UUID,
    assigned_at TIMESTAMP,
    version INT NOT NULL DEFAULT 0,
    FOREIGN KEY (floor_id) REFERENCES parking_floors(floor_id),
    INDEX idx_floor_status (floor_id, status),
    INDEX idx_type_status (type, status)
);

-- Vehicles table
CREATE TABLE vehicles (
    vehicle_id UUID PRIMARY KEY,
    license_plate VARCHAR(50) NOT NULL UNIQUE,
    type VARCHAR(20) NOT NULL,
    color VARCHAR(50),
    owner_name VARCHAR(255),
    INDEX idx_license (license_plate)
);

-- Parking tickets table
CREATE TABLE parking_tickets (
    ticket_id UUID PRIMARY KEY,
    ticket_number VARCHAR(50) NOT NULL UNIQUE,
    lot_id UUID NOT NULL,
    spot_id UUID NOT NULL,
    vehicle_id UUID NOT NULL,
    entry_time TIMESTAMP NOT NULL,
    exit_time TIMESTAMP,
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    paid_at TIMESTAMP,
    FOREIGN KEY (lot_id) REFERENCES parking_lots(lot_id),
    FOREIGN KEY (spot_id) REFERENCES parking_spots(spot_id),
    FOREIGN KEY (vehicle_id) REFERENCES vehicles(vehicle_id),
    INDEX idx_ticket_number (ticket_number),
    INDEX idx_status (status),
    INDEX idx_entry_time (entry_time)
);

-- Payments table
CREATE TABLE payments (
    payment_id UUID PRIMARY KEY,
    ticket_id UUID NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    payment_method VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL,
    transaction_id VARCHAR(255),
    processed_at TIMESTAMP NOT NULL DEFAULT NOW(),
    FOREIGN KEY (ticket_id) REFERENCES parking_tickets(ticket_id),
    INDEX idx_ticket (ticket_id)
);

-- Pricing configuration table
CREATE TABLE pricing_config (
    config_id UUID PRIMARY KEY,
    lot_id UUID NOT NULL,
    base_rate DECIMAL(10, 2) NOT NULL,
    compact_rate DECIMAL(10, 2),
    regular_rate DECIMAL(10, 2),
    large_rate DECIMAL(10, 2),
    handicapped_rate DECIMAL(10, 2),
    electric_rate DECIMAL(10, 2),
    daily_max_rate DECIMAL(10, 2),
    valid_from TIMESTAMP NOT NULL,
    valid_to TIMESTAMP,
    FOREIGN KEY (lot_id) REFERENCES parking_lots(lot_id)
);

-- Entry/Exit gates table
CREATE TABLE gates (
    gate_id UUID PRIMARY KEY,
    lot_id UUID NOT NULL,
    gate_number VARCHAR(50) NOT NULL,
    type VARCHAR(10) NOT NULL, -- ENTRY or EXIT
    status VARCHAR(20) NOT NULL DEFAULT 'CLOSED',
    last_used_at TIMESTAMP,
    FOREIGN KEY (lot_id) REFERENCES parking_lots(lot_id)
);
```

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

