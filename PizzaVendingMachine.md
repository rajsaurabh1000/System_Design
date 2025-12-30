# Pizza Vending Machine - System Design

<img width="4583" height="8777" alt="Pizza_Vending_Machine2" src="https://github.com/user-attachments/assets/3ac0ecb0-70d5-4335-8e3f-43bd29972c2b" />


## 1. Functional Requirements (FR)

### Core Features
1. **Pizza Selection & Ordering**
   - Browse available pizzas (types, toppings, sizes)
   - Select pizza and customize (if supported)
   - View nutritional information and allergens
   - Check preparation time
   - Select payment method

2. **Payment Processing**
   - Accept multiple payment methods (card, mobile pay, cash)
   - Process payment securely
   - Issue receipt (digital/printed)
   - Handle refunds for failures

3. **Pizza Preparation**
   - Retrieve frozen pizza from storage
   - Defrost (if needed)
   - Bake/cook pizza to proper temperature
   - Monitor cooking process
   - Notify when ready

4. **Pizza Dispensing**
   - Package pizza safely
   - Dispense to pickup area
   - Ensure proper temperature
   - Handle pickup verification

5. **Inventory Management**
   - Track inventory levels
   - Alert when stock low
   - Manage expiration dates
   - Reorder supplies automatically

6. **Machine Monitoring**
   - Temperature monitoring (oven, storage)
   - Equipment status (oven, dispenser, payment)
   - Error detection and alerts
   - Usage analytics

### Out of Scope
- Custom pizza building (use pre-made pizzas)
- Delivery service
- Loyalty program integration
- Multi-language support (can be extension)

## 2. Non-Functional Requirements (NFR)

1. **Performance**
   - Pizza preparation time: 3-5 minutes
   - Payment processing: < 10 seconds
   - UI response time: < 500ms
   - Order processing: < 30 seconds (selection to start cooking)

2. **Availability**
   - 99% uptime (24/7 operation)
   - Graceful degradation (cash-only if card reader fails)
   - Remote monitoring 24/7
   - Maintenance mode support

3. **Reliability**
   - Payment success rate: 99.9%
   - Pizza quality consistency
   - Proper cooking temperature (guaranteed food safety)
   - Accurate inventory tracking

4. **Safety**
   - Food safety compliance (HACCP)
   - Electrical safety
   - Fire safety systems
   - Emergency stop mechanism
   - Safe for public access (no pinch points, hot surfaces protected)

5. **Security**
   - PCI DSS compliance for payments
   - Secure communication (TLS/SSL)
   - Physical security (tamper-proof)
   - Access control for maintenance

6. **Scalability**
   - Support fleet of 100+ machines
   - Handle peak loads (lunch/dinner rush)
   - Centralized monitoring and management

## 3. Core Entities

### 3.1 Machine
```
Machine {
  machineId: UUID
  location: {
    address: string,
    latitude: float,
    longitude: float
  }
  status: enum (OPERATIONAL, MAINTENANCE, ERROR, OFFLINE)
  temperature: {
    storage: float,
    oven: float,
    ambient: float
  }
  inventory: List<InventoryItem>
  lastMaintenance: timestamp
  softwareVersion: string
}
```

### 3.2 Pizza
```
Pizza {
  pizzaId: UUID
  name: string
  description: string
  size: enum (SMALL, MEDIUM, LARGE)
  toppings: List<string>
  price: decimal
  calories: int
  allergens: List<string>
  imageUrl: string
  cookingTime: int (seconds)
  storageSlot: int
}
```

### 3.3 Order
```
Order {
  orderId: UUID
  machineId: UUID
  pizzaId: UUID
  status: enum (PENDING, PAYMENT_PROCESSING, PAID, PREPARING, 
                READY, DISPENSED, CANCELLED, REFUNDED)
  customerEmail: string (optional)
  totalAmount: decimal
  paymentMethod: enum
  paymentTransactionId: string
  placedAt: timestamp
  readyAt: timestamp (nullable)
  dispensedAt: timestamp (nullable)
  pickupCode: string (4-digit)
}
```

### 3.4 InventoryItem
```
InventoryItem {
  itemId: UUID
  machineId: UUID
  pizzaId: UUID
  quantity: int
  storageSlot: int
  expirationDate: date
  addedAt: timestamp
}
```

### 3.5 Payment
```
Payment {
  paymentId: UUID
  orderId: UUID
  amount: decimal
  method: enum (CARD, MOBILE_PAY, CASH)
  status: enum (PENDING, COMPLETED, FAILED, REFUNDED)
  transactionId: string
  gatewayResponse: JSON
  processedAt: timestamp
}
```

### 3.6 MaintenanceLog
```
MaintenanceLog {
  logId: UUID
  machineId: UUID
  type: enum (ROUTINE, REPAIR, CLEANING, RESTOCKING)
  performedBy: string
  description: text
  partsReplaced: List<string>
  cost: decimal
  startedAt: timestamp
  completedAt: timestamp
}
```

### 3.7 Alert
```
Alert {
  alertId: UUID
  machineId: UUID
  type: enum (LOW_INVENTORY, EQUIPMENT_FAILURE, 
              TEMPERATURE_ISSUE, PAYMENT_ISSUE)
  severity: enum (INFO, WARNING, CRITICAL)
  message: string
  resolved: boolean
  createdAt: timestamp
  resolvedAt: timestamp (nullable)
}
```

### 3.8 CookingSession
```
CookingSession {
  sessionId: UUID
  orderId: UUID
  machineId: UUID
  startTemperature: float
  targetTemperature: float
  currentTemperature: float
  estimatedCompletion: timestamp
  status: enum (HEATING, COOKING, COMPLETE, ERROR)
  startedAt: timestamp
  completedAt: timestamp (nullable)
}
```

## 4. API Design

### 4.1 Customer-Facing APIs (Machine UI)
```
GET /api/v1/machine/{machineId}/menu
Response: {
  pizzas: [
    {
      pizzaId, name, description, size,
      toppings, price, calories, allergens,
      imageUrl, cookingTime, available: boolean
    }
  ]
}

POST /api/v1/machine/{machineId}/order
Request: {
  pizzaId: UUID,
  customerEmail?: string
}
Response: {
  orderId: UUID,
  totalAmount: decimal,
  estimatedTime: int (seconds),
  pickupCode: string
}

POST /api/v1/order/{orderId}/payment
Request: {
  method: enum,
  paymentDetails: {...}
}
Response: {
  paymentId: UUID,
  status: enum,
  receiptUrl?: string
}

GET /api/v1/order/{orderId}/status
Response: {
  orderId: UUID,
  status: enum,
  estimatedReadyTime: timestamp,
  pickupCode: string
}

POST /api/v1/order/{orderId}/pickup
Request: {
  pickupCode: string
}
Response: {
  success: boolean,
  message: string
}
```

### 4.2 Machine Control APIs
```
POST /api/v1/machine/{machineId}/cook
Request: {
  orderId: UUID,
  pizzaId: UUID,
  storageSlot: int
}
Response: {
  sessionId: UUID,
  estimatedCompletion: timestamp
}

GET /api/v1/machine/{machineId}/cooking-status
Response: {
  sessionId: UUID,
  currentTemperature: float,
  targetTemperature: float,
  timeRemaining: int,
  status: enum
}

POST /api/v1/machine/{machineId}/dispense
Request: {
  orderId: UUID
}
Response: {
  success: boolean,
  message: string
}

POST /api/v1/machine/{machineId}/cancel-order
Request: {
  orderId: UUID,
  reason: string
}
Response: {
  success: boolean,
  refundInitiated: boolean
}
```

### 4.3 Admin/Monitoring APIs
```
GET /api/v1/admin/machines
Headers: { Authorization: Bearer <token> }
Query: { status?, location? }
Response: {
  machines: [
    {
      machineId, location, status,
      temperature, inventoryLevels,
      lastMaintenance, alerts: int
    }
  ]
}

GET /api/v1/admin/machine/{machineId}/status
Headers: { Authorization: Bearer <token> }
Response: {
  machineId, status, temperature,
  inventory: [...],
  activeOrders: [...],
  recentAlerts: [...],
  uptime: float
}

GET /api/v1/admin/machine/{machineId}/inventory
Headers: { Authorization: Bearer <token> }
Response: {
  items: [
    {
      pizzaId, pizzaName, quantity,
      storageSlot, expirationDate
    }
  ],
  lowStockItems: [...]
}

POST /api/v1/admin/machine/{machineId}/maintenance-mode
Headers: { Authorization: Bearer <token> }
Request: { enable: boolean, reason: string }
Response: { success: boolean }

GET /api/v1/admin/analytics
Headers: { Authorization: Bearer <token> }
Query: { startDate, endDate, machineId? }
Response: {
  totalOrders: int,
  revenue: decimal,
  averageOrderTime: float,
  popularPizzas: [...],
  machineUptime: {...}
}
```

### 4.4 Inventory Management APIs
```
POST /api/v1/admin/machine/{machineId}/restock
Headers: { Authorization: Bearer <token> }
Request: {
  items: [
    {
      pizzaId: UUID,
      quantity: int,
      storageSlot: int,
      expirationDate: date
    }
  ]
}
Response: { success: boolean }

GET /api/v1/admin/low-inventory
Headers: { Authorization: Bearer <token> }
Response: {
  machines: [
    {
      machineId, location,
      lowStockItems: [
        { pizzaId, pizzaName, currentQuantity, minimumQuantity }
      ]
    }
  ]
}
```

### 4.5 Real-time Monitoring (WebSocket)
```
WS /api/v1/machine/{machineId}/live

// Server -> Client: Order status updates
{
  type: "ORDER_UPDATE",
  orderId: UUID,
  status: enum,
  timestamp: long
}

// Server -> Client: Temperature alerts
{
  type: "TEMPERATURE_ALERT",
  zone: enum (STORAGE, OVEN),
  currentTemp: float,
  targetTemp: float,
  timestamp: long
}

// Server -> Client: Equipment errors
{
  type: "EQUIPMENT_ERROR",
  component: enum (OVEN, DISPENSER, PAYMENT, STORAGE),
  errorCode: string,
  message: string,
  timestamp: long
}

// Server -> Client: Cooking progress
{
  type: "COOKING_PROGRESS",
  orderId: UUID,
  temperature: float,
  timeRemaining: int,
  timestamp: long
}
```

## 5. High-Level Design (HLD)

### 5.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                  VENDING MACHINE HARDWARE                   │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ Touchscreen  │  │  Card Reader │  │ Cash Acceptor│     │
│  │  Display     │  │  (NFC/EMV)   │  │              │     │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │
│         │                  │                  │            │
│         └──────────────────┼──────────────────┘            │
│                            │                               │
│  ┌──────────────┐  ┌───────▼──────┐  ┌──────────────┐     │
│  │   Receipt    │  │ Main Control │  │  WiFi/4G     │     │
│  │   Printer    │  │  Computer    │  │   Modem      │     │
│  └──────┬───────┘  │(Raspberry Pi)│  └──────┬───────┘     │
│         │          └───────┬──────┘         │            │
│         └──────────────────┼────────────────┘            │
│                            │                               │
│  ┌─────────────────────────┼──────────────────┐           │
│  │                         │                  │           │
│  │  ┌──────────────┐  ┌────▼─────┐  ┌───────▼───────┐    │
│  │  │  Storage     │  │  Oven    │  │  Dispenser    │    │
│  │  │  Carousel    │  │ (Conveyor│  │  Mechanism    │    │
│  │  │ (Refrigerated│  │  Oven)   │  │ (Robotic Arm) │    │
│  │  │  -5°C)       │  │ 200-250°C│  │               │    │
│  │  └──────┬───────┘  └────┬─────┘  └───────┬───────┘    │
│  │         │               │                │            │
│  │  ┌──────▼───────────────▼────────────────▼───────┐    │
│  │  │         PLC (Programmable Logic Controller)    │    │
│  │  │         - Motor control                        │    │
│  │  │         - Temperature sensors                  │    │
│  │  │         - Safety interlocks                    │    │
│  │  └────────────────────────────────────────────────┘    │
│  │                                                        │
│  └────────────────────────────────────────────────────────┘
└─────────────────────────────────────────────────────────────┘
                             │
                             │ Internet
                             │
                ┌────────────▼─────────────┐
                │   Load Balancer          │
                └────────────┬─────────────┘
                             │
┌────────────────────────────┼─────────────────────────────────┐
│                   CLOUD BACKEND                              │
│               ┌────────────▼─────────────┐                   │
│               │   API Gateway            │                   │
│               └────────────┬─────────────┘                   │
│                            │                                 │
│    ┌───────────────────────┼───────────────────┐            │
│    │                       │                   │            │
│ ┌──▼──────┐  ┌─────────────▼──────┐  ┌────────▼────────┐   │
│ │ Order   │  │    Payment         │  │   Inventory     │   │
│ │ Service │  │    Service         │  │   Service       │   │
│ └──┬──────┘  └─────────┬──────────┘  └────────┬────────┘   │
│    │                   │                       │            │
│ ┌──▼──────┐  ┌─────────▼──────┐  ┌────────────▼────────┐   │
│ │ Machine │  │   Analytics    │  │   Notification      │   │
│ │ Control │  │   Service      │  │   Service           │   │
│ │ Service │  └────────────────┘  └─────────────────────┘   │
│ └──┬──────┘                                                 │
│    │                                                        │
│ ┌──▼──────────────┐                                         │
│ │   Maintenance   │                                         │
│ │   Service       │                                         │
│ └─────────────────┘                                         │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────┼────────────────────────────────────────┐
│              DATA LAYER                                     │
│    ┌───────────────▼────────┐  ┌──────────────┐            │
│    │      PostgreSQL        │  │    Redis     │            │
│    │  (Orders, Inventory,   │  │  (Cache,     │            │
│    │   Machines, Payments)  │  │   Sessions)  │            │
│    └────────────────────────┘  └──────────────┘            │
│    ┌────────────────────────┐  ┌──────────────┐            │
│    │    TimeSeries DB       │  │    Kafka     │            │
│    │  (Temperature logs,    │  │  (Events,    │            │
│    │   Machine metrics)     │  │   Alerts)    │            │
│    └────────────────────────┘  └──────────────┘            │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│                MONITORING & ADMIN DASHBOARD                │
│         ┌──────────────────────────────┐                   │
│         │   Web Dashboard              │                   │
│         │  - Fleet monitoring          │                   │
│         │  - Analytics & Reports       │                   │
│         │  - Inventory management      │                   │
│         │  - Maintenance scheduling    │                   │
│         └──────────────────────────────┘                   │
└────────────────────────────────────────────────────────────┘
```

### 5.2 Order Flow Sequence

```
Customer        Machine UI      Order Service   Payment Svc   Machine Control
   │                │                │               │              │
   │ Select Pizza   │                │               │              │
   ├───────────────>│                │               │              │
   │                │ Create Order   │               │              │
   │                ├───────────────>│               │              │
   │                │  Order Created │               │              │
   │  Show Payment  │<───────────────┤               │              │
   │  Options       │                │               │              │
   │<───────────────┤                │               │              │
   │                │                │               │              │
   │ Insert Card    │                │               │              │
   ├───────────────>│                │               │              │
   │                │  Process       │               │              │
   │                │  Payment       │               │              │
   │                ├────────────────┼──────────────>│              │
   │                │                │  Payment OK   │              │
   │  Payment OK    │                │<──────────────┤              │
   │<───────────────┤<───────────────┤               │              │
   │                │                │               │              │
   │                │  Start Cooking │               │              │
   │                ├────────────────┼───────────────┼─────────────>│
   │                │                │               │  Retrieve    │
   │                │                │               │  Pizza from  │
   │                │                │               │  Storage     │
   │                │                │               │<─────────────┤
   │                │                │               │              │
   │                │                │               │  Start Oven  │
   │                │                │               │<─────────────┤
   │  "Preparing    │                │               │              │
   │   Your Pizza"  │                │               │              │
   │<───────────────┤                │               │              │
   │                │                │               │              │
   │                │                │               │  Monitor     │
   │                │                │               │  Cooking     │
   │                │                │               │              │
   │                │                │               │  (3-5 min)   │
   │                │                │               │              │
   │                │                │               │  Pizza Ready │
   │                │  Update Status │               │<─────────────┤
   │  "Pizza Ready" │<───────────────┤<──────────────┼──────────────┤
   │  Pickup Code   │                │               │              │
   │<───────────────┤                │               │  Dispense    │
   │                │                │               │  to Pickup   │
   │                │                │               │  Area        │
   │ Enter Code     │                │               │<─────────────┤
   ├───────────────>│                │               │              │
   │                │  Verify Code   │               │              │
   │                ├───────────────>│               │              │
   │                │  Code Valid    │               │              │
   │                │<───────────────┤               │              │
   │                │                │               │  Open Door   │
   │  Door Opens    │                │               │<─────────────┤
   │<───────────────┼────────────────┼───────────────┼──────────────┤
   │                │                │               │              │
   │ Take Pizza     │                │               │              │
   │                │                │               │              │
```

### 5.3 Cooking Process Flow

```
┌─────────────────────────────────────────────────────────┐
│                   Cooking State Machine                 │
│                                                         │
│                      ┌──────────┐                       │
│                      │  IDLE    │                       │
│                      └────┬─────┘                       │
│                           │                             │
│                    Order Received                       │
│                           │                             │
│                   ┌───────▼───────┐                     │
│                   │   RETRIEVING  │                     │
│                   │   (Get pizza  │                     │
│                   │   from slot)  │                     │
│                   └───────┬───────┘                     │
│                           │                             │
│                    Pizza Retrieved                      │
│                           │                             │
│                   ┌───────▼───────┐                     │
│                   │   HEATING     │                     │
│                   │ (Preheat oven │                     │
│                   │  to 220°C)    │                     │
│                   └───────┬───────┘                     │
│                           │                             │
│                    Temp Reached                         │
│                           │                             │
│                   ┌───────▼───────┐                     │
│                   │   COOKING     │                     │
│                   │ (3-5 minutes) │                     │
│                   │ Monitor temp  │                     │
│                   └───────┬───────┘                     │
│                           │                             │
│                    Cooking Complete                     │
│                           │                             │
│                   ┌───────▼───────┐                     │
│                   │   PACKAGING   │                     │
│                   │ (Transfer to  │                     │
│                   │  pickup area) │                     │
│                   └───────┬───────┘                     │
│                           │                             │
│                    Packaging Done                       │
│                           │                             │
│                   ┌───────▼───────┐                     │
│                   │   READY       │                     │
│                   │ (Awaiting     │                     │
│                   │  pickup)      │                     │
│                   └───────┬───────┘                     │
│                           │                             │
│                    Pizza Picked Up                      │
│                           │                             │
│                   ┌───────▼───────┐                     │
│                   │   COMPLETE    │                     │
│                   └───────┬───────┘                     │
│                           │                             │
│                        Return to                        │
│                         IDLE                            │
│                                                         │
│              Error at any stage -> ERROR_STATE          │
│              -> Refund customer -> Return to IDLE       │
└─────────────────────────────────────────────────────────┘
```

### 6. Optimizations & Approaches (Concise)

6.1 Temperature Control (PID)

Uses a PID controller to continuously adjust heating power based on error between target and current temperature, ensuring stable and efficient cooking.

6.2 Predictive Inventory Management

Uses historical sales averages to forecast demand and auto-generate restock recommendations before inventory runs out.

6.3 Energy Optimization

Dynamically adjusts oven temperature based on order queue and peak hours to reduce power consumption during idle periods.

6.4 Payment Failure Handling

Implements retry with backoff, fallback states, and deferred processing to safely handle gateway or network failures.

6.5 Queue Management (Order Scheduling)

Uses a priority-based queue to process orders FIFO with VIP prioritization while ensuring only one active cooking job.

6.6 Anomaly Detection

Analyzes temperature variance, cooking duration, and payment failures to detect abnormal machine behavior early.

6.7 Food Safety Compliance (HACCP)

Tracks time spent in unsafe temperature ranges and automatically discards expired or unsafe items.

6.8 Offline Mode Support

Allows limited offline operation using cached inventory and deferred sync once connectivity is restored.

## 7. Low-Level Design (LLD)

<img width="2391" height="842" alt="image" src="https://github.com/user-attachments/assets/1311e962-283d-4509-a7d5-a914a0114ce6" />

### Design Patterns Used (Interview-Friendly)

Strategy → Swappable algorithms (path planning, inventory forecasting, scheduling)

State → Machine, oven, and order lifecycle management

Command → Encapsulates order actions (start, cancel, cook, refund)

Factory → Creates payments, orders, or strategy implementations

Singleton → Central controllers (MachineManager, ConfigManager)

Facade → VendingMachine exposes a simple interface over subsystems

Observer / Pub-Sub → Alerts, monitoring, UI updates

Adapter → Payment gateways, sensors, external APIs

Template Method → Cooking, validation, and processing workflows

Repository → Abstracts data persistence logic

Builder → Constructs complex orders or configurations safely

Chain of Responsibility → Validation → payment → cooking → dispensing

Decorator → Adds behavior like logging, retries, or safety checks

Flyweight → Reuse immutable objects like configurations and constants

### Data Structures Used (1 line each)

Map / HashMap → Fast lookup for inventory, robots, and configurations

PriorityQueue → Order scheduling based on priority and arrival time

Queue / Deque → Sequential order execution and buffering

List / ArrayList → Store history, metrics, logs, and cleaning paths

Circular Buffer → Sliding window for sensor and telemetry data

Set / HashSet → Track processed or unique items (dependencies, alerts)

Time-series List → Store metrics like temperature, runtime, failures

Graph (Adjacency List) → Dependency or workflow relationships

LRU Cache → Cache frequently accessed configs or inventory

Enum → Represent safe state transitions (machine, order, oven states)

Heap / Priority Queue → Scheduling, anomaly severity ranking

Append-only Log → Auditing, telemetry, and event replay

Polygon / Geometry List → Define no-go zones or spatial boundaries

### 7.2 Scalability Calculations

**Single Machine Capacity**:
- Cooking time: 4 minutes per pizza
- Max throughput: 15 pizzas/hour
- Operating hours: 18 hours/day (6am-12am)
- Daily capacity: 270 pizzas
- Storage: 40-60 pizzas

**Fleet Management**:
- 100 machines
- Total daily capacity: 27,000 pizzas
- Peak hour demand: ~10% = 2,700 pizzas/hour
- System handles: 100 machines × 1 order/minute = 100 orders/minute

**Backend Scalability**:
- Database: PostgreSQL handles 10K orders/minute
- API: Load balanced, auto-scaling
- Real-time monitoring: WebSocket connections for 100 machines

**Data Volume**:
- Order: ~1KB
- Daily orders: 27,000 × 1KB = 27MB
- Annual: ~10GB
- Logs and metrics: ~50GB/year

---

## Interview Discussion Points

### Clarifying Questions:
1. Location: Indoor (mall) or outdoor (street)?
2. Power: Standard electrical or special requirements?
3. Menu: Fixed pizzas or customizable?
4. Target market: Students, office workers, general public?
5. Price point: Premium or budget?

### Trade-offs:
1. **Fresh vs Frozen**: Quality vs shelf life
2. **Speed vs Quality**: Fast cooking vs perfect pizza
3. **Complexity vs Reliability**: Features vs failure modes
4. **Cash vs Cashless**: Accessibility vs maintenance

### Extensions:
1. **Multiple Pizza Types**: Pasta, sandwiches
2. **Customization**: Build-your-own pizza
3. **Loyalty Program**: Points and rewards
4. **Mobile Pre-ordering**: Order ahead via app
5. **Delivery Robot**: Integrate with delivery robots

This comprehensive pizza vending machine design covers hardware, software, and operational aspects for a production-ready system!

