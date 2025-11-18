# Pizza Vending Machine - System Design

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

## 6. Optimizations & Approaches

### 6.1 Temperature Control Algorithm

**PID Controller for Oven**:
```python
class PIDController:
    def __init__(self, kp, ki, kd):
        self.kp = kp  # Proportional gain
        self.ki = ki  # Integral gain
        self.kd = kd  # Derivative gain
        
        self.previous_error = 0
        self.integral = 0
    
    def calculate(self, target_temp, current_temp, dt):
        # Calculate error
        error = target_temp - current_temp
        
        # Proportional term
        P = self.kp * error
        
        # Integral term
        self.integral += error * dt
        I = self.ki * self.integral
        
        # Derivative term
        derivative = (error - self.previous_error) / dt
        D = self.kd * derivative
        
        # Update previous error
        self.previous_error = error
        
        # Calculate control output (heating power %)
        output = P + I + D
        
        # Clamp output to 0-100%
        output = max(0, min(100, output))
        
        return output

# Usage
pid = PIDController(kp=2.0, ki=0.5, kd=1.0)

while cooking:
    current_temp = read_temperature()
    heating_power = pid.calculate(
        target_temp=220,
        current_temp=current_temp,
        dt=0.1  # 100ms update rate
    )
    
    set_heating_element_power(heating_power)
    time.sleep(0.1)
```

### 6.2 Predictive Inventory Management

**Forecast demand and auto-reorder**:
```python
def predict_inventory_needs(machine_id, days_ahead=7):
    # Get historical data
    history = get_sales_history(machine_id, days=30)
    
    # Calculate average daily sales per pizza type
    daily_avg = {}
    for pizza_id in get_pizza_types():
        sales = [day[pizza_id] for day in history]
        daily_avg[pizza_id] = np.mean(sales)
    
    # Get current inventory
    current_inventory = get_inventory(machine_id)
    
    # Predict when restocking needed
    restock_recommendations = []
    
    for pizza_id, avg_sales in daily_avg.items():
        current_qty = current_inventory[pizza_id]
        days_until_empty = current_qty / avg_sales
        
        if days_until_empty < days_ahead:
            # Need to restock
            recommended_qty = int(avg_sales * 14)  # 2 weeks supply
            restock_recommendations.append({
                'pizza_id': pizza_id,
                'current_qty': current_qty,
                'recommended_qty': recommended_qty,
                'urgency': 'HIGH' if days_until_empty < 3 else 'MEDIUM'
            })
    
    return restock_recommendations

# Run daily
schedule.every().day.at("03:00").do(
    lambda: generate_restock_orders(predict_inventory_needs)
)
```

### 6.3 Energy Efficiency

**Smart Oven Pre-heating**:
```python
def optimize_oven_heating(order_queue):
    """
    Pre-heat oven based on predicted orders
    Save energy by turning off during idle periods
    """
    
    if len(order_queue) > 0:
        # Orders in queue, keep oven hot
        set_oven_target_temp(220)
    
    elif is_peak_hour():
        # Peak hours (lunch/dinner), keep oven warm
        set_oven_target_temp(150)  # Standby temp
    
    else:
        # Off-peak, turn off oven
        set_oven_target_temp(0)
    
def is_peak_hour():
    hour = datetime.now().hour
    # 11am-2pm or 5pm-9pm
    return (11 <= hour <= 14) or (17 <= hour <= 21)
```

### 6.4 Payment Failure Handling

**Robust payment processing**:
```python
async def process_payment(order_id, payment_details):
    max_retries = 3
    retry_delay = 2  # seconds
    
    for attempt in range(max_retries):
        try:
            # Call payment gateway
            response = await payment_gateway.charge(
                amount=order.total_amount,
                payment_method=payment_details
            )
            
            if response.status == 'SUCCESS':
                # Payment successful
                update_order_status(order_id, 'PAID')
                return {'success': True, 'transaction_id': response.id}
            
            elif response.status == 'DECLINED':
                # Card declined, don't retry
                return {'success': False, 'error': 'Payment declined'}
            
            else:
                # Gateway error, retry
                if attempt < max_retries - 1:
                    await asyncio.sleep(retry_delay)
                    continue
                else:
                    return {'success': False, 'error': 'Payment processing failed'}
        
        except NetworkError:
            # Network issue, retry
            if attempt < max_retries - 1:
                await asyncio.sleep(retry_delay)
            else:
                # Fallback: Mark order as pending and notify support
                update_order_status(order_id, 'PAYMENT_PENDING')
                notify_support(f"Payment network error for order {order_id}")
                return {'success': False, 'error': 'Network error, order on hold'}
```

### 6.5 Queue Management (Multiple Orders)

**Handle concurrent orders**:
```python
class OrderQueue:
    def __init__(self):
        self.queue = PriorityQueue()
        self.current_order = None
        self.max_concurrent = 1  # Only 1 pizza at a time
    
    def add_order(self, order):
        # Calculate priority (FIFO but VIP can have higher priority)
        priority = order.placed_at.timestamp()
        if order.is_vip:
            priority -= 1000  # Higher priority
        
        self.queue.put((priority, order))
        
        # Estimate wait time
        position = self.queue.qsize()
        estimated_wait = position * 300  # 5 min per pizza
        
        return estimated_wait
    
    def get_next_order(self):
        if self.queue.empty():
            return None
        
        priority, order = self.queue.get()
        self.current_order = order
        return order
    
    def complete_current_order(self):
        self.current_order = None

# In main loop
queue = OrderQueue()

while True:
    if machine.is_idle() and not queue.is_empty():
        order = queue.get_next_order()
        machine.prepare_pizza(order)
    
    time.sleep(1)
```

### 6.6 Anomaly Detection

**Detect equipment issues early**:
```python
def detect_anomalies(machine_id):
    # Get recent metrics
    metrics = get_machine_metrics(machine_id, hours=24)
    
    anomalies = []
    
    # Check oven temperature stability
    oven_temps = [m['oven_temp'] for m in metrics]
    temp_std = np.std(oven_temps)
    if temp_std > 15:  # High variance
        anomalies.append({
            'type': 'OVEN_INSTABILITY',
            'severity': 'WARNING',
            'message': 'Oven temperature fluctuating abnormally'
        })
    
    # Check cooking times
    cook_times = [m['cook_time'] for m in metrics if m['cook_time']]
    avg_cook_time = np.mean(cook_times)
    if avg_cook_time > 360:  # 6 minutes (normal is 4)
        anomalies.append({
            'type': 'SLOW_COOKING',
            'severity': 'WARNING',
            'message': 'Cooking times longer than normal'
        })
    
    # Check payment success rate
    total_payments = len([m for m in metrics if m['payment_attempt']])
    failed_payments = len([m for m in metrics if m['payment_failed']])
    failure_rate = failed_payments / total_payments if total_payments > 0 else 0
    
    if failure_rate > 0.1:  # >10% failure
        anomalies.append({
            'type': 'PAYMENT_ISSUES',
            'severity': 'CRITICAL',
            'message': f'High payment failure rate: {failure_rate*100:.1f}%'
        })
    
    # Create alerts
    for anomaly in anomalies:
        create_alert(machine_id, anomaly)
    
    return anomalies

# Run every hour
schedule.every().hour.do(lambda: detect_anomalies_all_machines())
```

### 6.7 Food Safety Compliance

**HACCP monitoring**:
```python
class FoodSafetyMonitor:
    # Temperature danger zone: 4°C to 60°C
    DANGER_ZONE_MIN = 4
    DANGER_ZONE_MAX = 60
    MAX_DANGER_ZONE_TIME = 120  # minutes (2 hours)
    
    def __init__(self):
        self.time_in_danger_zone = {}
    
    def monitor_pizza(self, pizza_id, current_temp):
        if self.DANGER_ZONE_MIN < current_temp < self.DANGER_ZONE_MAX:
            # Pizza in danger zone
            if pizza_id not in self.time_in_danger_zone:
                self.time_in_danger_zone[pizza_id] = 0
            
            self.time_in_danger_zone[pizza_id] += 1  # Increment minutes
            
            if self.time_in_danger_zone[pizza_id] > self.MAX_DANGER_ZONE_TIME:
                # Exceeded safe time, discard pizza
                self.discard_pizza(pizza_id, reason="Exceeded safe temperature time")
                return False
        else:
            # Reset timer (either frozen or hot)
            self.time_in_danger_zone[pizza_id] = 0
        
        return True
    
    def check_expiration_dates(self, machine_id):
        inventory = get_inventory(machine_id)
        expired = []
        
        for item in inventory:
            if item.expiration_date <= date.today():
                expired.append(item)
                self.discard_pizza(item.pizza_id, reason="Expired")
        
        if expired:
            notify_operator(f"{len(expired)} expired items discarded")
    
    def discard_pizza(self, pizza_id, reason):
        # Mark as discarded in inventory
        remove_from_inventory(pizza_id)
        
        # Log for compliance
        log_food_safety_event({
            'pizza_id': pizza_id,
            'action': 'DISCARDED',
            'reason': reason,
            'timestamp': datetime.now()
        })
```

### 6.8 Offline Mode

**Handle network outages**:
```python
class OfflineMode:
    def __init__(self):
        self.offline = False
        self.pending_orders = []
        self.local_inventory = None
    
    def detect_network_status(self):
        try:
            # Ping backend
            response = requests.get(BACKEND_URL + '/health', timeout=5)
            self.offline = False
        except:
            self.offline = True
    
    def process_order_offline(self, order):
        # Check local inventory cache
        if not self.check_inventory_local(order.pizza_id):
            return {'success': False, 'error': 'Out of stock'}
        
        # Accept cash only in offline mode
        if order.payment_method != 'CASH':
            return {'success': False, 'error': 'Only cash accepted (offline mode)'}
        
        # Create order locally
        order.status = 'PENDING_SYNC'
        self.pending_orders.append(order)
        
        # Proceed with cooking
        machine.prepare_pizza(order)
        
        return {'success': True, 'offline_mode': True}
    
    def sync_when_online(self):
        if not self.offline and self.pending_orders:
            # Upload pending orders
            for order in self.pending_orders:
                try:
                    api.create_order(order)
                    self.pending_orders.remove(order)
                except:
                    pass  # Will retry next time
```

## 7. Low-Level Design (LLD)

### 7.1 Class Diagram

```
┌──────────────────────────────────────────────────────────┐
│                   VendingMachine                         │
├──────────────────────────────────────────────────────────┤
│ - machineId: UUID                                        │
│ - status: MachineStatus                                  │
│ - storage: StorageUnit                                   │
│ - oven: Oven                                             │
│ - dispenser: Dispenser                                   │
│ - paymentProcessor: PaymentProcessor                     │
│ - orderQueue: OrderQueue                                 │
├──────────────────────────────────────────────────────────┤
│ + processOrder(order: Order): void                       │
│ + checkInventory(pizzaId: UUID): boolean                 │
│ + cookPizza(order: Order): void                          │
│ + dispensePizza(order: Order): void                      │
│ + enterMaintenanceMode(): void                           │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                      Oven                                │
├──────────────────────────────────────────────────────────┤
│ - currentTemp: float                                     │
│ - targetTemp: float                                      │
│ - status: enum (IDLE, HEATING, COOKING, ERROR)           │
│ - pidController: PIDController                           │
├──────────────────────────────────────────────────────────┤
│ + preheat(targetTemp: float): void                       │
│ + startCooking(pizza: Pizza): CookingSession             │
│ + monitorTemperature(): float                            │
│ + stopCooking(): void                                    │
│ + selfTest(): boolean                                    │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                   StorageUnit                            │
├──────────────────────────────────────────────────────────┤
│ - slots: Map<int, InventoryItem>                         │
│ - temperature: float                                     │
│ - capacity: int                                          │
├──────────────────────────────────────────────────────────┤
│ + retrievePizza(pizzaId: UUID): Pizza                    │
│ + addPizza(pizza: Pizza, slot: int): void                │
│ + getInventoryLevel(): Map<Pizza, int>                   │
│ + checkExpiration(): List<InventoryItem>                 │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                   PaymentProcessor                       │
├──────────────────────────────────────────────────────────┤
│ - cardReader: CardReader                                 │
│ - cashAcceptor: CashAcceptor                             │
│ - paymentGateway: PaymentGateway                         │
├──────────────────────────────────────────────────────────┤
│ + processPayment(amount, method): PaymentResult          │
│ + refund(transactionId): boolean                         │
│ + checkConnectivity(): boolean                           │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                    Dispenser                             │
├──────────────────────────────────────────────────────────┤
│ - roboticArm: RoboticArm                                 │
│ - pickupArea: PickupArea                                 │
├──────────────────────────────────────────────────────────┤
│ + transferPizza(from: Location, to: Location): void      │
│ + openPickupDoor(): void                                 │
│ + closePickupDoor(): void                                │
│ + verifyPickup(): boolean                                │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                   OrderController                        │
├──────────────────────────────────────────────────────────┤
│ - orderService: OrderService                             │
│ - machine: VendingMachine                                │
├──────────────────────────────────────────────────────────┤
│ + createOrder(pizzaId, customer): Order                  │
│ + processPayment(order, payment): PaymentResult          │
│ + startCooking(order): void                              │
│ + completeOrder(order): void                             │
│ + cancelOrder(order, reason): void                       │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                 InventoryManager                         │
├──────────────────────────────────────────────────────────┤
│ - inventoryService: InventoryService                     │
│ - thresholds: Map<Pizza, int>                            │
├──────────────────────────────────────────────────────────┤
│ + checkStock(pizzaId): int                               │
│ + updateInventory(pizzaId, delta: int): void             │
│ + generateRestockOrder(): RestockOrder                   │
│ + removeExpiredItems(): List<Item>                       │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│               MaintenanceMonitor                         │
├──────────────────────────────────────────────────────────┤
│ - alertService: AlertService                             │
│ - metricCollector: MetricCollector                       │
├──────────────────────────────────────────────────────────┤
│ + monitorTemperature(): void                             │
│ + checkEquipmentStatus(): List<Alert>                    │
│ + logMetrics(): void                                     │
│ + detectAnomalies(): List<Anomaly>                       │
└──────────────────────────────────────────────────────────┘
```

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

