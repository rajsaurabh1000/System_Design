# Elevator System - System Design

## 1. Functional Requirements (FR)

### Core Features
1. **User Request Handling**
   - Call elevator from any floor (up/down buttons)
   - Select destination floor inside elevator
   - Request multiple floors
   - Cancel requests

2. **Elevator Movement**
   - Move up and down
   - Open and close doors
   - Stop at requested floors
   - Emergency stop functionality

3. **Request Scheduling**
   - Optimize elevator assignment to requests
   - Minimize wait time
   - Minimize travel time
   - Handle multiple simultaneous requests

4. **Safety Features**
   - Door sensors (don't close on people)
   - Weight limit enforcement
   - Emergency button (call for help)
   - Power failure handling

5. **Display & Notifications**
   - Current floor indicator
   - Direction indicator (up/down arrows)
   - Estimated wait time (optional)
   - Audio/visual notifications

6. **Multiple Elevators**
   - Coordinate multiple elevator cars
   - Load balancing
   - Avoid bunching (all elevators on same floor)

### Out of Scope
- Maintenance scheduling
- Video surveillance
- Fire safety protocols (can be discussed as extension)
- IoT integration

## 2. Non-Functional Requirements (NFR)

1. **Performance**
   - Response time: < 1 second for button press
   - Average wait time: < 60 seconds
   - Door operation: 3-5 seconds
   - Floor travel time: 3-5 seconds per floor

2. **Availability**
   - 99.99% uptime
   - Failover for control system
   - Manual override capability
   - At least one elevator operational at all times

3. **Scalability**
   - Support buildings with 100+ floors
   - Support 10+ elevator cars
   - Handle 1000+ requests per hour

4. **Safety**
   - Zero accidents due to system failure
   - Immediate response to emergency button
   - Graceful handling of power loss
   - Door safety sensors

5. **Efficiency**
   - Minimize energy consumption
   - Optimize for peak vs off-peak hours
   - Reduce waiting time
   - Reduce travel time

6. **Reliability**
   - Precise floor alignment
   - Accurate weight sensing
   - Reliable door mechanism

## 3. Core Entities

### 3.1 Elevator
```
Elevator {
  elevatorId: UUID
  currentFloor: int
  direction: enum (UP, DOWN, IDLE)
  state: enum (MOVING, STOPPED, DOOR_OPEN, DOOR_CLOSED, MAINTENANCE, EMERGENCY)
  capacity: int (kg)
  currentLoad: int (kg)
  destinationFloors: Set<int>
  doorStatus: enum (OPEN, CLOSED, OPENING, CLOSING)
}
```

### 3.2 Request
```
Request {
  requestId: UUID
  floor: int
  direction: enum (UP, DOWN)
  timestamp: long
  status: enum (PENDING, ASSIGNED, COMPLETED, CANCELLED)
  assignedElevatorId: UUID (nullable)
}
```

### 3.3 DestinationRequest
```
DestinationRequest {
  requestId: UUID
  elevatorId: UUID
  fromFloor: int
  toFloor: int
  timestamp: long
  status: enum (PENDING, IN_PROGRESS, COMPLETED)
}
```

### 3.4 Building
```
Building {
  buildingId: UUID
  name: string
  totalFloors: int
  groundFloor: int (usually 0 or 1)
  elevators: List<Elevator>
  floorHeight: double (meters)
}
```

### 3.5 Floor
```
Floor {
  floorNumber: int
  hasUpButton: boolean
  hasDownButton: boolean
  upButtonPressed: boolean
  downButtonPressed: boolean
  waitingPassengers: int (estimate)
}
```

### 3.6 ElevatorController
```
ElevatorController {
  controllerId: UUID
  building: Building
  scheduler: RequestScheduler
  activeRequests: Queue<Request>
  algorithm: enum (FCFS, SCAN, LOOK, SSTF)
}
```

## 4. API Design

### 4.1 External Request APIs (Floor Buttons)
```
POST /api/v1/elevator/request
Request: {
  floor: int,
  direction: enum (UP, DOWN)
}
Response: {
  requestId: UUID,
  estimatedWaitTime: int (seconds),
  assignedElevator: int (elevator number)
}

DELETE /api/v1/elevator/request/{requestId}
Response: { success: boolean }
```

### 4.2 Internal Request APIs (Elevator Buttons)
```
POST /api/v1/elevator/{elevatorId}/destination
Request: {
  floor: int
}
Response: {
  requestId: UUID,
  estimatedArrivalTime: int (seconds)
}

GET /api/v1/elevator/{elevatorId}/status
Response: {
  elevatorId: UUID,
  currentFloor: int,
  direction: enum,
  destinationFloors: [int],
  doorStatus: enum,
  load: int,
  capacity: int
}
```

### 4.3 Control APIs
```
POST /api/v1/elevator/{elevatorId}/door/open
Response: { success: boolean }

POST /api/v1/elevator/{elevatorId}/door/close
Response: { success: boolean }

POST /api/v1/elevator/{elevatorId}/emergency
Response: { success: boolean, emergencyResponse: string }

POST /api/v1/elevator/{elevatorId}/stop
Response: { success: boolean }
```

### 4.4 Admin APIs
```
GET /api/v1/admin/elevators/status
Headers: { Authorization: Bearer <token> }
Response: {
  elevators: [
    {
      elevatorId, currentFloor, direction,
      state, load, activeRequests: int
    }
  ]
}

PUT /api/v1/admin/elevator/{elevatorId}/maintenance
Headers: { Authorization: Bearer <token> }
Request: { enable: boolean }
Response: { success: boolean }

GET /api/v1/admin/analytics
Headers: { Authorization: Bearer <token> }
Query: { startDate, endDate }
Response: {
  totalTrips: int,
  averageWaitTime: float,
  averageTravelTime: float,
  peakHours: [...],
  efficiency: float
}
```

### 4.5 Real-time WebSocket API
```
WS /api/v1/elevator/updates

// Server -> Client: Elevator position updates
{
  type: "ELEVATOR_UPDATE",
  elevatorId: UUID,
  currentFloor: int,
  direction: enum,
  doorStatus: enum
}

// Server -> Client: Request status updates
{
  type: "REQUEST_UPDATE",
  requestId: UUID,
  status: enum,
  assignedElevator: int,
  estimatedArrivalTime: int
}
```

## 5. High-Level Design (HLD)

### 5.1 Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                   PHYSICAL LAYER                             │
│                                                              │
│  Floor Buttons    Elevator Car      Door Sensors            │
│  ┌──────────┐    ┌──────────┐      ┌──────────┐            │
│  │ Up/Down  │    │ Keypad   │      │ IR/Laser │            │
│  │ Buttons  │    │ Display  │      │ Sensors  │            │
│  └────┬─────┘    └────┬─────┘      └────┬─────┘            │
│       │               │                  │                  │
│       └───────────────┼──────────────────┘                  │
│                       │                                     │
│               ┌───────▼────────┐                            │
│               │  PLC/MCU       │                            │
│               │  (Hardware     │                            │
│               │   Controller)  │                            │
│               └───────┬────────┘                            │
└───────────────────────┼──────────────────────────────────────┘
                        │
                        │ Serial/Network
                        │
┌───────────────────────┼──────────────────────────────────────┐
│              CONTROL SYSTEM LAYER                            │
│               ┌───────▼────────┐                             │
│               │   Elevator     │                             │
│               │   Controller   │                             │
│               │   Service      │                             │
│               └───────┬────────┘                             │
│                       │                                      │
│       ┌───────────────┼───────────────┐                     │
│       │               │               │                     │
│   ┌───▼────┐    ┌────▼────┐    ┌────▼─────┐                │
│   │Request │    │ Elevator│    │Scheduler │                │
│   │Manager │    │ State   │    │ (SCAN/   │                │
│   │        │    │ Machine │    │  LOOK)   │                │
│   └────────┘    └─────────┘    └──────────┘                │
│                                                              │
└──────────────────────┬───────────────────────────────────────┘
                       │
┌──────────────────────┼───────────────────────────────────────┐
│                 DATA LAYER                                   │
│    ┌──────────┐  ┌──▼─────────┐  ┌──────────────┐          │
│    │ Redis    │  │ PostgreSQL │  │  TimeSeries  │          │
│    │ (State,  │  │ (Config,   │  │  DB          │          │
│    │ Requests)│  │  Logs)     │  │ (Analytics)  │          │
│    └──────────┘  └────────────┘  └──────────────┘          │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                  MONITORING LAYER                            │
│         ┌────────────────┐      ┌─────────────┐             │
│         │  Monitoring    │      │  Alert      │             │
│         │  Dashboard     │      │  System     │             │
│         └────────────────┘      └─────────────┘             │
└──────────────────────────────────────────────────────────────┘
```

### 5.2 Component Descriptions

**Physical Layer:**
- Floor buttons (up/down calls)
- Elevator car buttons (destination selection)
- Door sensors (safety)
- Weight sensors
- Motor and drive system
- PLC/MCU for low-level control

**Control System:**
- **Elevator Controller**: Main orchestrator
- **Request Manager**: Handles incoming requests
- **Scheduler**: Assigns elevators to requests
- **State Machine**: Manages elevator states and transitions
- **Safety Monitor**: Checks safety conditions

**Data Layer:**
- **Redis**: Real-time state, pending requests, elevator positions
- **PostgreSQL**: Configuration, audit logs, maintenance records
- **TimeSeries DB**: Analytics data (wait times, trips, etc.)

**Monitoring:**
- Real-time dashboard for operators
- Alert system for failures
- Performance metrics

### 5.3 Elevator State Machine

```
┌──────────────────────────────────────────────────────────────┐
│                   Elevator State Diagram                     │
│                                                              │
│                        ┌──────┐                              │
│                        │ IDLE │                              │
│                        └───┬──┘                              │
│                            │                                 │
│                  Request Received                            │
│                            │                                 │
│                    ┌───────▼────────┐                        │
│                    │ DOOR_OPENING   │                        │
│                    └───────┬────────┘                        │
│                            │                                 │
│                     Doors Fully Open                         │
│                            │                                 │
│                    ┌───────▼────────┐                        │
│              ┌────>│   DOOR_OPEN    │<─────┐                │
│              │     └───────┬────────┘      │                │
│              │             │               │                │
│        Door Sensor    Timer Expires    Obstacle             │
│        Blocked             │                                 │
│              │     ┌───────▼────────┐      │                │
│              │     │ DOOR_CLOSING   │──────┘                │
│              │     └───────┬────────┘                        │
│              │             │                                 │
│              │      Doors Fully Closed                       │
│              │             │                                 │
│              │     ┌───────▼────────┐                        │
│              └─────┤   MOVING       │                        │
│                    └───────┬────────┘                        │
│                            │                                 │
│                   Reached Destination                        │
│                            │                                 │
│                    ┌───────▼────────┐                        │
│                    │   STOPPED      │                        │
│                    └───────┬────────┘                        │
│                            │                                 │
│                   ┌────────┴─────────┐                       │
│              No More        More Destinations                │
│              Requests              │                         │
│                   │                │                         │
│              ┌────▼────┐           │                         │
│              │  IDLE   │<──────────┘                         │
│              └─────────┘                                     │
│                                                              │
│            Emergency Button: Any State -> EMERGENCY          │
└──────────────────────────────────────────────────────────────┘
```

### 5.4 Request Flow

```
User presses     Controller      Scheduler      Elevator
Floor Button     receives                       
     │              │               │              │
     │  Request     │               │              │
     ├─────────────>│               │              │
     │              │ Find Best     │              │
     │              │ Elevator      │              │
     │              ├──────────────>│              │
     │              │               │ Evaluate     │
     │              │               │ Elevators    │
     │              │               │ (cost calc)  │
     │              │               │              │
     │              │ Elevator ID   │              │
     │              │<──────────────┤              │
     │              │               │              │
     │              │ Assign Request│              │
     │              ├──────────────────────────────>│
     │              │               │              │
     │              │               │  Add to Queue│
     │              │               │  Update Route│
     │              │               │              │
     │  Estimated   │               │              │
     │  Wait Time   │               │              │
     │<─────────────┤               │              │
     │              │               │              │
     │              │               │    Move to   │
     │              │               │    Floor     │
     │              │               │<─────────────┤
     │              │               │              │
     │   Elevator   │               │              │
     │   Arrived    │               │              │
     │<─────────────┼───────────────┼──────────────┤
     │              │               │              │
```

### 5.5 Scheduling Algorithms

**1. FCFS (First Come First Serve)**
```
Simple: Serve requests in order received
Pro: Fair
Con: Inefficient, high wait times
```

**2. SCAN (Elevator Algorithm)**
```
Move in one direction, serve all requests in that direction
When no more requests, reverse direction
Pro: Efficient, predictable
Con: Can have longer wait for opposite direction
```

**3. LOOK (Optimized SCAN)**
```
Like SCAN but reverse when no more requests (don't go to extremes)
Pro: More efficient than SCAN
Con: Still directional bias
```

**4. Shortest Seek Time First (SSTF)**
```
Serve nearest request first
Pro: Minimizes travel distance
Con: Can starve distant floors
```

**5. SCAN with Multiple Elevators (Recommended)**
```
Each elevator runs SCAN independently
Dispatcher assigns new requests to elevator with lowest cost
Cost = distance + estimated_time + load_factor
```

## 6. Optimizations & Approaches

### 6.1 Multi-Elevator Scheduling

**Problem**: With multiple elevators, how to assign requests?

**Solution**: Cost-based assignment
```python
def calculate_cost(elevator, request):
    cost = 0
    
    # Distance cost
    distance = abs(elevator.current_floor - request.floor)
    cost += distance * 10
    
    # Direction cost (prefer same direction)
    if elevator.direction == request.direction:
        cost += 0
    elif elevator.direction == IDLE:
        cost += 20
    else:
        cost += 50  # opposite direction, high cost
    
    # Load cost (prefer less loaded elevator)
    load_factor = elevator.current_load / elevator.capacity
    cost += load_factor * 30
    
    # Queue length cost
    cost += len(elevator.destination_floors) * 5
    
    return cost

def assign_request(request, elevators):
    available_elevators = [e for e in elevators 
                           if e.state != MAINTENANCE]
    
    best_elevator = min(available_elevators, 
                       key=lambda e: calculate_cost(e, request))
    
    return best_elevator
```

### 6.2 SCAN Algorithm Implementation

```python
class ElevatorController:
    def __init__(self):
        self.current_floor = 0
        self.direction = UP
        self.destination_floors = set()
        
    def add_destination(self, floor):
        self.destination_floors.add(floor)
    
    def get_next_floor(self):
        if not self.destination_floors:
            return None
        
        if self.direction == UP:
            # Get floors above current
            above = [f for f in self.destination_floors 
                     if f > self.current_floor]
            
            if above:
                return min(above)  # Next floor going up
            else:
                # No floors above, change direction
                self.direction = DOWN
                return max(self.destination_floors)
        
        else:  # DOWN
            # Get floors below current
            below = [f for f in self.destination_floors 
                     if f < self.current_floor]
            
            if below:
                return max(below)  # Next floor going down
            else:
                # No floors below, change direction
                self.direction = UP
                return min(self.destination_floors)
    
    def move_to_next_floor(self):
        next_floor = self.get_next_floor()
        
        if next_floor is None:
            self.direction = IDLE
            return
        
        # Move one floor at a time
        if next_floor > self.current_floor:
            self.current_floor += 1
        elif next_floor < self.current_floor:
            self.current_floor -= 1
        
        # Check if reached destination
        if self.current_floor in self.destination_floors:
            self.destination_floors.remove(self.current_floor)
            self.stop_at_floor()
    
    def stop_at_floor(self):
        self.open_doors()
        time.sleep(5)  # Wait for passengers
        self.close_doors()
```

### 6.3 Destination Dispatch System (Advanced)

**Concept**: Pre-assign elevator when user enters destination at lobby kiosk
**Benefits**: 
- Reduce travel time by grouping passengers with similar destinations
- More efficient than traditional up/down buttons

**Implementation**:
```python
def destination_dispatch(requests):
    """
    Group requests by destination zone
    Assign elevators to serve specific zones
    """
    
    # Divide building into zones
    zones = {
        'LOW': range(1, 11),      # Floors 1-10
        'MID': range(11, 21),     # Floors 11-20
        'HIGH': range(21, 31)     # Floors 21-30
    }
    
    # Group requests by zone
    requests_by_zone = {zone: [] for zone in zones}
    for req in requests:
        for zone_name, floor_range in zones.items():
            if req.destination_floor in floor_range:
                requests_by_zone[zone_name].append(req)
    
    # Assign elevators to zones
    assignments = []
    for zone, zone_requests in requests_by_zone.items():
        if zone_requests:
            elevator = find_best_elevator_for_zone(zone)
            assignments.append((elevator, zone_requests))
    
    return assignments
```

### 6.4 Energy Optimization

**Strategies**:
1. **Idle Parking**: Return to most-requested floor when idle (usually lobby)
2. **Sleep Mode**: Reduce power when idle for extended period
3. **Regenerative Braking**: Capture energy when descending
4. **Load Balancing**: Distribute elevators across floors during peak

### 6.5 Peak Hour Handling

**Morning Peak** (everyone going up):
- Keep more elevators at ground floor
- Express elevators for high floors

**Evening Peak** (everyone going down):
- Distribute elevators across all floors
- Group destinations going down

**Implementation**:
```python
class PeakModeController:
    def detect_peak_mode(self):
        current_hour = datetime.now().hour
        
        if 7 <= current_hour <= 9:
            return MORNING_PEAK
        elif 17 <= current_hour <= 19:
            return EVENING_PEAK
        else:
            return NORMAL
    
    def adjust_strategy(self, mode):
        if mode == MORNING_PEAK:
            # Return idle elevators to ground floor
            for elevator in self.elevators:
                if elevator.direction == IDLE and elevator.current_floor != 0:
                    elevator.add_destination(0)
        
        elif mode == EVENING_PEAK:
            # Spread elevators across building
            floors = range(self.building.total_floors)
            idle_elevators = [e for e in self.elevators if e.direction == IDLE]
            
            for i, elevator in enumerate(idle_elevators):
                target_floor = floors[i % len(floors)]
                elevator.add_destination(target_floor)
```

### 6.6 Emergency Handling

**Emergency Button Pressed**:
1. Stop at next available floor
2. Open doors
3. Call emergency services
4. Keep doors open
5. Disable further movement

**Power Failure**:
1. Backup battery power activates
2. Move to nearest floor
3. Open doors
4. Wait for power restoration

**Fire Alarm**:
1. Return all elevators to ground floor (non-stop)
2. Open doors
3. Disable elevator calls
4. Display "Out of Service"

### 6.7 Door Safety

**Sensors**:
- Infrared beam across doorway
- Load sensors detect obstruction
- Acoustic sensors

**Logic**:
```python
def close_doors():
    while not doors_fully_closed():
        if door_sensor_blocked():
            open_doors()
            wait(3_seconds)
            retry_count += 1
            
            if retry_count > 3:
                # Alert maintenance
                trigger_alarm()
                return False
        else:
            continue_closing()
    
    return True
```

### 6.8 Wait Time Estimation

**Algorithm**:
```python
def estimate_wait_time(request, assigned_elevator):
    # Calculate time for elevator to reach request floor
    
    # 1. Time to complete current destinations before request
    time = 0
    current_floor = assigned_elevator.current_floor
    
    for dest in assigned_elevator.destination_floors_in_order():
        if reached_request_floor(dest, request.floor):
            break
        
        # Travel time
        time += abs(dest - current_floor) * FLOOR_TRAVEL_TIME
        
        # Stop time (door open/close + passenger exchange)
        time += STOP_TIME
        
        current_floor = dest
    
    # 2. Time to reach request floor
    time += abs(current_floor - request.floor) * FLOOR_TRAVEL_TIME
    
    return time

FLOOR_TRAVEL_TIME = 3  # seconds per floor
STOP_TIME = 10  # seconds for door operation + passenger exchange
```

### 6.9 Preventing Elevator Bunching

**Problem**: Multiple elevators end up at same location
**Solution**: 
- Monitor elevator distribution
- Assign idle elevators to sparse areas
- Use cost function that penalizes clustering

```python
def anti_bunching_cost(elevator, floor):
    # Check proximity to other elevators
    proximity_cost = 0
    
    for other_elevator in self.elevators:
        if other_elevator.id != elevator.id:
            distance = abs(other_elevator.current_floor - floor)
            if distance < 3:  # Within 3 floors
                proximity_cost += 100 / (distance + 1)
    
    return proximity_cost
```

### 6.10 Handling Overweight

**Detection**:
- Weight sensors in elevator floor
- Compare against capacity

**Action**:
```python
def check_weight():
    if current_load > capacity:
        # Play warning sound
        play_audio("Overweight, please exit")
        
        # Keep doors open
        prevent_door_close()
        
        # Flash warning light
        enable_warning_indicator()
        
        # Don't move until weight is reduced
        while current_load > capacity:
            wait(1_second)
        
        # Resume normal operation
        disable_warning_indicator()
```

## 7. Low-Level Design (LLD)

### 7.1 Class Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                        Building                              │
├──────────────────────────────────────────────────────────────┤
│ - buildingId: UUID                                           │
│ - totalFloors: int                                           │
│ - elevators: List<Elevator>                                  │
│ - controller: ElevatorController                             │
├──────────────────────────────────────────────────────────────┤
│ + addElevator(elevator: Elevator): void                      │
│ + requestElevator(floor: int, direction: Direction): void    │
│ + getStatus(): BuildingStatus                                │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                        Elevator                              │
├──────────────────────────────────────────────────────────────┤
│ - elevatorId: UUID                                           │
│ - currentFloor: int                                          │
│ - direction: Direction                                       │
│ - state: ElevatorState                                       │
│ - destinationFloors: SortedSet<int>                          │
│ - capacity: int                                              │
│ - currentLoad: int                                           │
│ - door: Door                                                 │
├──────────────────────────────────────────────────────────────┤
│ + move(): void                                               │
│ + stop(): void                                               │
│ + addDestination(floor: int): void                           │
│ + openDoors(): void                                          │
│ + closeDoors(): void                                         │
│ + emergency(): void                                          │
│ + getNextFloor(): int                                        │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                   ElevatorController                         │
├──────────────────────────────────────────────────────────────┤
│ - elevators: List<Elevator>                                  │
│ - scheduler: Scheduler                                       │
│ - requestQueue: Queue<Request>                               │
├──────────────────────────────────────────────────────────────┤
│ + handleRequest(request: Request): void                      │
│ + assignElevator(request: Request): Elevator                 │
│ + monitorElevators(): void                                   │
│ + handleEmergency(elevatorId: UUID): void                    │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                      Scheduler                               │
│                    <<interface>>                             │
├──────────────────────────────────────────────────────────────┤
│ + selectElevator(request, elevators): Elevator               │
│ + optimize(elevators): void                                  │
└──────────────────────────────────────────────────────────────┘
           △                              △
           │                              │
     ┌─────┴──────┐               ┌──────┴────────┐
     │    SCAN    │               │     LOOK      │
     │ Scheduler  │               │  Scheduler    │
     └────────────┘               └───────────────┘

┌──────────────────────────────────────────────────────────────┐
│                         Request                              │
├──────────────────────────────────────────────────────────────┤
│ - requestId: UUID                                            │
│ - floor: int                                                 │
│ - direction: Direction                                       │
│ - timestamp: long                                            │
│ - status: RequestStatus                                      │
├──────────────────────────────────────────────────────────────┤
│ + assignToElevator(elevator: Elevator): void                 │
│ + cancel(): void                                             │
│ + complete(): void                                           │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                          Door                                │
├──────────────────────────────────────────────────────────────┤
│ - status: DoorStatus                                         │
│ - sensor: DoorSensor                                         │
├──────────────────────────────────────────────────────────────┤
│ + open(): void                                               │
│ + close(): boolean                                           │
│ + isObstructed(): boolean                                    │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                      DoorSensor                              │
├──────────────────────────────────────────────────────────────┤
│ - sensorType: enum (IR, ACOUSTIC, PRESSURE)                  │
├──────────────────────────────────────────────────────────────┤
│ + detectObstruction(): boolean                               │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                      ButtonPanel                             │
├──────────────────────────────────────────────────────────────┤
│ - buttons: Map<int, Button>                                  │
│ - display: Display                                           │
├──────────────────────────────────────────────────────────────┤
│ + pressButton(floor: int): void                              │
│ + updateDisplay(floor: int, direction: Direction): void      │
└──────────────────────────────────────────────────────────────┘
```

### 7.2 Implementation

#### 7.2.1 Elevator Class

```java
public class Elevator implements Runnable {
    private final UUID elevatorId;
    private int currentFloor;
    private Direction direction;
    private ElevatorState state;
    private TreeSet<Integer> destinationFloors;
    private final int capacity;
    private int currentLoad;
    private Door door;
    
    public Elevator(UUID id, int capacity) {
        this.elevatorId = id;
        this.currentFloor = 0;
        this.direction = Direction.IDLE;
        this.state = ElevatorState.STOPPED;
        this.destinationFloors = new TreeSet<>();
        this.capacity = capacity;
        this.currentLoad = 0;
        this.door = new Door();
    }
    
    @Override
    public void run() {
        while (true) {
            try {
                if (state == ElevatorState.MAINTENANCE) {
                    Thread.sleep(1000);
                    continue;
                }
                
                if (destinationFloors.isEmpty()) {
                    direction = Direction.IDLE;
                    Thread.sleep(100);
                    continue;
                }
                
                Integer nextFloor = getNextFloor();
                if (nextFloor == null) {
                    continue;
                }
                
                moveToFloor(nextFloor);
                
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
    
    public synchronized void addDestination(int floor) {
        if (floor < 0 || floor >= building.getTotalFloors()) {
            throw new IllegalArgumentException("Invalid floor");
        }
        
        destinationFloors.add(floor);
        
        // Set direction if idle
        if (direction == Direction.IDLE) {
            direction = (floor > currentFloor) ? Direction.UP : Direction.DOWN;
        }
    }
    
    private Integer getNextFloor() {
        if (destinationFloors.isEmpty()) {
            return null;
        }
        
        // SCAN algorithm
        if (direction == Direction.UP) {
            // Get next floor going up
            SortedSet<Integer> above = destinationFloors.tailSet(currentFloor);
            if (!above.isEmpty()) {
                return above.first();
            } else {
                // Change direction
                direction = Direction.DOWN;
                return destinationFloors.last();
            }
        } else if (direction == Direction.DOWN) {
            // Get next floor going down
            SortedSet<Integer> below = destinationFloors.headSet(currentFloor + 1);
            if (!below.isEmpty()) {
                return below.last();
            } else {
                // Change direction
                direction = Direction.UP;
                return destinationFloors.first();
            }
        }
        
        return null;
    }
    
    private void moveToFloor(int targetFloor) throws InterruptedException {
        state = ElevatorState.MOVING;
        
        while (currentFloor != targetFloor) {
            // Move one floor
            if (targetFloor > currentFloor) {
                currentFloor++;
                direction = Direction.UP;
            } else {
                currentFloor--;
                direction = Direction.DOWN;
            }
            
            // Notify observers
            notifyFloorChange(currentFloor);
            
            // Simulate travel time
            Thread.sleep(3000); // 3 seconds per floor
        }
        
        // Reached destination
        stopAtFloor(targetFloor);
    }
    
    private void stopAtFloor(int floor) throws InterruptedException {
        state = ElevatorState.STOPPED;
        destinationFloors.remove(floor);
        
        // Open doors
        openDoors();
        
        // Wait for passengers
        Thread.sleep(5000); // 5 seconds
        
        // Close doors
        closeDoors();
        
        // Ready to move again
        if (!destinationFloors.isEmpty()) {
            state = ElevatorState.MOVING;
        } else {
            direction = Direction.IDLE;
        }
    }
    
    public synchronized void openDoors() throws InterruptedException {
        state = ElevatorState.DOOR_OPENING;
        door.open();
        state = ElevatorState.DOOR_OPEN;
        notifyDoorStatus(DoorStatus.OPEN);
    }
    
    public synchronized void closeDoors() throws InterruptedException {
        state = ElevatorState.DOOR_CLOSING;
        
        boolean closed = door.close();
        
        if (!closed) {
            // Door sensor blocked, reopen
            openDoors();
            Thread.sleep(3000);
            closeDoors();  // Retry
        } else {
            state = ElevatorState.DOOR_CLOSED;
            notifyDoorStatus(DoorStatus.CLOSED);
        }
    }
    
    public synchronized void emergency() {
        state = ElevatorState.EMERGENCY;
        
        // Stop movement
        destinationFloors.clear();
        
        // Move to nearest floor
        try {
            // Emergency protocol
            openDoors();
            // Alert emergency services
            notifyEmergency();
        } catch (Exception e) {
            // Log error
        }
    }
    
    public boolean isOverweight() {
        return currentLoad > capacity;
    }
    
    // Getters and notification methods
    public int getCurrentFloor() { return currentFloor; }
    public Direction getDirection() { return direction; }
    public ElevatorState getState() { return state; }
    public Set<Integer> getDestinationFloors() { 
        return new TreeSet<>(destinationFloors); 
    }
    
    private void notifyFloorChange(int floor) {
        // Publish event or update UI
    }
    
    private void notifyDoorStatus(DoorStatus status) {
        // Publish event or update UI
    }
    
    private void notifyEmergency() {
        // Alert emergency services
    }
}
```

#### 7.2.2 Elevator Controller

```java
@Service
public class ElevatorController {
    private final List<Elevator> elevators;
    private final Scheduler scheduler;
    private final Queue<Request> requestQueue;
    private final ExecutorService executorService;
    
    public ElevatorController(int numElevators, int capacity, Scheduler scheduler) {
        this.elevators = new ArrayList<>();
        this.scheduler = scheduler;
        this.requestQueue = new ConcurrentLinkedQueue<>();
        this.executorService = Executors.newFixedThreadPool(numElevators);
        
        // Initialize elevators
        for (int i = 0; i < numElevators; i++) {
            Elevator elevator = new Elevator(UUID.randomUUID(), capacity);
            elevators.add(elevator);
            executorService.submit(elevator);
        }
        
        // Start request processor
        executorService.submit(this::processRequests);
    }
    
    public UUID handleRequest(int floor, Direction direction) {
        Request request = new Request(UUID.randomUUID(), floor, direction);
        requestQueue.offer(request);
        return request.getRequestId();
    }
    
    private void processRequests() {
        while (true) {
            try {
                Request request = requestQueue.poll();
                
                if (request == null) {
                    Thread.sleep(100);
                    continue;
                }
                
                // Find best elevator
                Elevator assignedElevator = scheduler.selectElevator(
                    request,
                    getAvailableElevators()
                );
                
                if (assignedElevator != null) {
                    // Assign request
                    assignedElevator.addDestination(request.getFloor());
                    request.setAssignedElevator(assignedElevator.getElevatorId());
                    request.setStatus(RequestStatus.ASSIGNED);
                    
                    // Notify user
                    notifyUserAssignment(request, assignedElevator);
                } else {
                    // No available elevator, requeue
                    requestQueue.offer(request);
                    Thread.sleep(1000);
                }
                
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
    
    private List<Elevator> getAvailableElevators() {
        return elevators.stream()
            .filter(e -> e.getState() != ElevatorState.MAINTENANCE &&
                        e.getState() != ElevatorState.EMERGENCY)
            .collect(Collectors.toList());
    }
    
    public List<ElevatorStatus> getSystemStatus() {
        return elevators.stream()
            .map(this::getElevatorStatus)
            .collect(Collectors.toList());
    }
    
    private ElevatorStatus getElevatorStatus(Elevator elevator) {
        return new ElevatorStatus(
            elevator.getElevatorId(),
            elevator.getCurrentFloor(),
            elevator.getDirection(),
            elevator.getState(),
            elevator.getDestinationFloors().size()
        );
    }
    
    public void handleEmergency(UUID elevatorId) {
        elevators.stream()
            .filter(e -> e.getElevatorId().equals(elevatorId))
            .findFirst()
            .ifPresent(Elevator::emergency);
    }
    
    public void shutdown() {
        executorService.shutdownNow();
    }
}
```

#### 7.2.3 SCAN Scheduler

```java
public class SCANScheduler implements Scheduler {
    
    @Override
    public Elevator selectElevator(Request request, List<Elevator> elevators) {
        if (elevators.isEmpty()) {
            return null;
        }
        
        // Calculate cost for each elevator
        Elevator bestElevator = elevators.stream()
            .min(Comparator.comparingInt(e -> calculateCost(e, request)))
            .orElse(null);
        
        return bestElevator;
    }
    
    private int calculateCost(Elevator elevator, Request request) {
        int cost = 0;
        
        // 1. Distance cost
        int distance = Math.abs(elevator.getCurrentFloor() - request.getFloor());
        cost += distance * 10;
        
        // 2. Direction alignment cost
        Direction elevatorDir = elevator.getDirection();
        Direction requestDir = request.getDirection();
        
        if (elevatorDir == requestDir) {
            // Same direction
            if (elevatorDir == Direction.UP && 
                request.getFloor() >= elevator.getCurrentFloor()) {
                // On the way
                cost += 0;
            } else if (elevatorDir == Direction.DOWN && 
                       request.getFloor() <= elevator.getCurrentFloor()) {
                // On the way
                cost += 0;
            } else {
                // Not on the way, need to come back
                cost += 100;
            }
        } else if (elevatorDir == Direction.IDLE) {
            // Idle elevator, moderate cost
            cost += 20;
        } else {
            // Opposite direction, high cost
            cost += 150;
        }
        
        // 3. Load cost
        int queueLength = elevator.getDestinationFloors().size();
        cost += queueLength * 5;
        
        // 4. Current load cost
        if (elevator.isOverweight()) {
            cost += 1000; // Don't assign to overweight elevator
        }
        
        return cost;
    }
    
    @Override
    public void optimize(List<Elevator> elevators) {
        // Anti-bunching: spread idle elevators
        List<Elevator> idleElevators = elevators.stream()
            .filter(e -> e.getDirection() == Direction.IDLE)
            .collect(Collectors.toList());
        
        if (idleElevators.size() <= 1) {
            return;
        }
        
        // Distribute idle elevators across building
        int totalFloors = 30; // Example
        int spacing = totalFloors / idleElevators.size();
        
        for (int i = 0; i < idleElevators.size(); i++) {
            Elevator elevator = idleElevators.get(i);
            int targetFloor = spacing * i;
            
            if (elevator.getCurrentFloor() != targetFloor) {
                elevator.addDestination(targetFloor);
            }
        }
    }
}
```

## 🧩 Design Patterns Used

| Pattern             | Where Used               | Why                                  |
|---------------------|--------------------------|--------------------------------------|
| **Strategy**        | SCAN / LOOK / SSTF       | Plug different scheduling algorithms |
| **State**           | ElevatorState, DoorState | Clean and deterministic state changes|
| **Command**         | Request objects          | Encapsulate user actions             |
| **Observer**        | UI / Monitoring          | Push real-time updates               |
| **Facade**          | ElevatorController       | Single entry point for system ops    |
| **Factory**         | SchedulerFactory         | Create scheduling strategies         |
| **Singleton**       | ElevatorController       | Ensure single orchestrator           |
| **Adapter**         | PLCAdapter               | Bridge software and hardware         |
| **Template Method** | Scheduler base class     | Share common scheduling flow         |

## 📦 Data Structures Summary (Interview Friendly)

| Component            | Data Structure | Reason                         |
|---------------------|----------------|--------------------------------|
| Elevator queues     | `TreeSet`      | Maintains ordered floors       |
| Requests            | `Queue`        | FIFO intake handling           |
| Elevators           | `List`         | Easy iteration and traversal   |
| States              | `Enum`         | Safe and explicit transitions  |
| Scheduler lookup    | Strategy       | Extensible algorithm selection |
| Sensors             | DTO            | Simple data transport object   |



### 7.3 Scalability Calculations

**Building Specifications**:
- 30 floors
- 6 elevators
- Capacity: 10 people per elevator
- Average trip: 10 floors (30 seconds)

**Peak Hour Traffic**:
- 1000 people entering building in 1 hour
- Each elevator: 1000 / 6 / 10 = ~17 trips
- Trip time: 30 seconds + 10 seconds (loading) = 40 seconds
- Capacity: 6 elevators × 90 trips/hour = 540 people/hour

**Wait Time**:
- Average wait: 60 seconds (acceptable)
- With optimizations: < 30 seconds

**System Resources**:
- CPU: Minimal (simple algorithms)
- Memory: < 1MB (state of 6 elevators)
- Database: Minimal logging

---

## Interview Discussion Points

### Clarifying Questions:
1. Building size: Floors? Elevators?
2. Traffic patterns: Office? Residential? Mixed?
3. Special requirements: Express elevators? Service elevators?
4. Budget constraints?
5. Existing vs new installation?

### Trade-offs:
1. **Simple vs Complex Algorithm**: FCFS vs SCAN
2. **Number of Elevators**: Cost vs performance
3. **Features vs Reliability**: Advanced features add complexity

### Extensions:
1. **Double-Decker Elevators**: Two cabins per shaft
2. **Destination Dispatch**: Kiosk-based assignment
3. **AI-Powered**: Machine learning for pattern prediction
4. **IoT Integration**: Predictive maintenance
5. **Access Control**: Security badges, floor restrictions

This comprehensive elevator system design handles real-world scenarios efficiently!

