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
<img width="718" height="693" alt="Screenshot 2025-12-29 at 22 09 31" src="https://github.com/user-attachments/assets/e794e19b-ec42-41b8-8b80-d8217d791514" />


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

### 6.2 SCAN Algorithm Implementation

### 6.3 Destination Dispatch System (Advanced)

**Concept**: Pre-assign elevator when user enters destination at lobby kiosk
**Benefits**: 
- Reduce travel time by grouping passengers with similar destinations
- More efficient than traditional up/down buttons

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

### 6.8 Wait Time Estimation

### 6.9 Preventing Elevator Bunching

**Problem**: Multiple elevators end up at same location
**Solution**: 
- Monitor elevator distribution
- Assign idle elevators to sparse areas
- Use cost function that penalizes clustering

### 6.10 Handling Overweight

**Detection**:
- Weight sensors in elevator floor
- Compare against capacity

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

