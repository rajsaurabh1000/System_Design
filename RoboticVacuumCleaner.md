# Robotic Vacuum Cleaner - System Design

<img width="4609" height="8874" alt="Robotic_Vacuum_Cleaner" src="https://github.com/user-attachments/assets/a1622ade-971c-42f1-bf47-40f369171b1a" />


## 1. Functional Requirements (FR)

### Core Features
1. **Autonomous Navigation**
   - Map the room/house
   - Navigate around obstacles
   - Avoid stairs and hazards
   - Return to charging dock

2. **Cleaning Operations**
   - Vacuum floor surfaces
   - Adjustable suction power
   - Multiple cleaning modes (auto, spot, edge, scheduled)
   - Detect and handle different floor types (carpet, hardwood, tile)

3. **Mapping & Path Planning**
   - Create floor plan map
   - Optimize cleaning path
   - Remember cleaned areas
   - Multi-room navigation
   - Save multiple floor plans

4. **Obstacle Detection & Avoidance**
   - Detect walls, furniture, objects
   - Detect stairs/drop-offs
   - Handle dynamic obstacles (pets, people)
   - Anti-collision sensors

5. **User Control**
   - Mobile app control
   - Start/stop/pause cleaning
   - Schedule cleaning times
   - Set no-go zones
   - Remote monitoring

6. **Maintenance & Monitoring**
   - Battery level monitoring
   - Dustbin full alert
   - Brush wear detection
   - Error notifications
   - Cleaning history and statistics

### Out of Scope
- Mopping functionality (can be extension)
- Voice assistant deep integration
- Multi-robot coordination
- Advanced AI object recognition

## 2. Non-Functional Requirements (NFR)

1. **Performance**
   - Cleaning speed: 20-30 m²/hour
   - Navigation latency: < 100ms
   - Mapping update: Real-time
   - Battery life: 90-120 minutes
   - Charging time: 3-4 hours

2. **Reliability**
   - Successfully return to dock: 99%
   - Complete cleaning without getting stuck: 95%
   - Uptime: 99% (operational when not charging)
   - MTBF (Mean Time Between Failures): 6 months

3. **Safety**
   - Never fall down stairs
   - Gentle collision with obstacles
   - Safe for pets and children
   - Auto-shutdown on malfunction

4. **Usability**
   - Easy setup (< 5 minutes)
   - Intuitive mobile app
   - Minimal user intervention
   - Clear error messages

5. **Efficiency**
   - Optimal path coverage
   - Minimize redundant cleaning
   - Energy efficient
   - Complete coverage: 95% of accessible area

6. **Scalability**
   - Support homes up to 300 m²
   - Handle complex layouts (10+ rooms)
   - Save up to 5 different floor plans

## 3. Core Entities

### 3.1 Robot
```
Robot {
  robotId: UUID
  modelNumber: string
  firmwareVersion: string
  batteryLevel: int (0-100%)
  state: enum (IDLE, CLEANING, RETURNING, CHARGING, ERROR)
  position: {x: float, y: float, theta: float}
  dustbinLevel: int (0-100%)
  brushWear: int (0-100%)
  filterWear: int (0-100%)
  totalRuntime: int (hours)
  totalCleanedArea: float (m²)
}
```

### 3.2 Map
```
Map {
  mapId: UUID
  name: string
  dimensions: {width: float, height: float}
  resolution: float (cm per cell)
  grid: int[][] (0=unknown, 1=free, 2=obstacle, 3=no-go)
  rooms: List<Room>
  dockPosition: {x: float, y: float}
  createdAt: timestamp
  updatedAt: timestamp
}
```

### 3.3 Room
```
Room {
  roomId: UUID
  name: string
  type: enum (LIVING_ROOM, BEDROOM, KITCHEN, BATHROOM, etc.)
  boundaries: Polygon
  floorType: enum (CARPET, HARDWOOD, TILE)
  isAccessible: boolean
}
```

### 3.4 CleaningTask
```
CleaningTask {
  taskId: UUID
  robotId: UUID
  mapId: UUID
  mode: enum (AUTO, SPOT, EDGE, ROOM_SPECIFIC)
  selectedRooms: List<UUID> (nullable)
  suctionPower: enum (LOW, MEDIUM, HIGH, AUTO)
  status: enum (SCHEDULED, IN_PROGRESS, COMPLETED, FAILED)
  startTime: timestamp
  endTime: timestamp (nullable)
  cleanedArea: float (m²)
  cleaningPath: List<Position>
}
```

### 3.5 Schedule
```
Schedule {
  scheduleId: UUID
  robotId: UUID
  daysOfWeek: Set<DayOfWeek>
  time: string (HH:MM)
  mode: enum
  selectedRooms: List<UUID> (nullable)
  enabled: boolean
}
```

### 3.6 NoGoZone
```
NoGoZone {
  zoneId: UUID
  mapId: UUID
  type: enum (NO_GO, NO_MOP)
  boundaries: Polygon
  createdAt: timestamp
}
```

### 3.7 ObstacleDetection
```
ObstacleDetection {
  detectionId: UUID
  timestamp: long
  type: enum (WALL, FURNITURE, DROP_OFF, UNKNOWN)
  position: {x: float, y: float}
  size: {width: float, height: float}
  confidence: float
}
```

### 3.8 User
```
User {
  userId: UUID
  email: string
  name: string
  robots: List<Robot>
  homes: List<Home>
}
```

### 3.9 Home
```
Home {
  homeId: UUID
  name: string
  address: string
  maps: List<Map>
}
```

## 4. API Design

### 4.1 Robot Control APIs
```
POST /api/v1/robot/{robotId}/start
Headers: { Authorization: Bearer <token> }
Request: {
  mode: enum (AUTO, SPOT, EDGE),
  roomIds?: List<UUID>,
  suctionPower?: enum
}
Response: {
  taskId: UUID,
  status: "started"
}

POST /api/v1/robot/{robotId}/pause
Headers: { Authorization: Bearer <token> }
Response: { status: "paused" }

POST /api/v1/robot/{robotId}/stop
Headers: { Authorization: Bearer <token> }
Response: { status: "stopped" }

POST /api/v1/robot/{robotId}/dock
Headers: { Authorization: Bearer <token> }
Response: { status: "returning_to_dock" }

GET /api/v1/robot/{robotId}/status
Headers: { Authorization: Bearer <token> }
Response: {
  robotId: UUID,
  state: enum,
  batteryLevel: int,
  position: {x, y, theta},
  currentTask: {...},
  dustbinLevel: int
}
```

### 4.2 Mapping APIs
```
GET /api/v1/robot/{robotId}/map
Headers: { Authorization: Bearer <token> }
Response: {
  mapId: UUID,
  name: string,
  dimensions: {...},
  grid: string (base64 encoded),
  rooms: [...],
  dockPosition: {x, y}
}

POST /api/v1/robot/{robotId}/map/room
Headers: { Authorization: Bearer <token> }
Request: {
  name: string,
  boundaries: [...],
  type: enum,
  floorType: enum
}
Response: {
  roomId: UUID,
  name: string
}

POST /api/v1/robot/{robotId}/map/no-go-zone
Headers: { Authorization: Bearer <token> }
Request: {
  boundaries: [...],
  type: enum
}
Response: {
  zoneId: UUID
}

POST /api/v1/robot/{robotId}/map/reset
Headers: { Authorization: Bearer <token> }
Response: { success: boolean }
```

### 4.3 Schedule APIs
```
POST /api/v1/robot/{robotId}/schedule
Headers: { Authorization: Bearer <token> }
Request: {
  daysOfWeek: [MONDAY, WEDNESDAY, FRIDAY],
  time: "09:00",
  mode: enum,
  roomIds?: [...]
}
Response: {
  scheduleId: UUID
}

GET /api/v1/robot/{robotId}/schedules
Headers: { Authorization: Bearer <token> }
Response: {
  schedules: [
    { scheduleId, daysOfWeek, time, mode, enabled }
  ]
}

PUT /api/v1/robot/{robotId}/schedule/{scheduleId}
Headers: { Authorization: Bearer <token> }
Request: { enabled: boolean }
Response: { success: boolean }

DELETE /api/v1/robot/{robotId}/schedule/{scheduleId}
Headers: { Authorization: Bearer <token> }
Response: { success: boolean }
```

### 4.4 History & Stats APIs
```
GET /api/v1/robot/{robotId}/history
Headers: { Authorization: Bearer <token> }
Query: { startDate, endDate, limit }
Response: {
  tasks: [
    {
      taskId, startTime, endTime, duration,
      cleanedArea, mode, status
    }
  ]
}

GET /api/v1/robot/{robotId}/statistics
Headers: { Authorization: Bearer <token> }
Query: { period: "week" | "month" | "year" }
Response: {
  totalCleaningTime: int,
  totalCleanedArea: float,
  averageCleaningDuration: int,
  cleaningsByDay: {...}
}
```

### 4.5 Maintenance APIs
```
GET /api/v1/robot/{robotId}/maintenance
Headers: { Authorization: Bearer <token> }
Response: {
  dustbinLevel: int,
  filterWear: int,
  brushWear: int,
  lastMaintenance: timestamp,
  recommendations: [
    "Clean dustbin",
    "Replace filter in 30 days"
  ]
}

POST /api/v1/robot/{robotId}/maintenance/reset
Headers: { Authorization: Bearer <token> }
Request: {
  component: enum (FILTER, BRUSH, SENSOR)
}
Response: { success: boolean }
```

### 4.6 Real-time Updates (WebSocket)
```
WS /api/v1/robot/{robotId}/live

// Server -> Client: Position updates
{
  type: "POSITION_UPDATE",
  position: {x, y, theta},
  timestamp: long
}

// Server -> Client: State changes
{
  type: "STATE_CHANGE",
  oldState: enum,
  newState: enum,
  timestamp: long
}

// Server -> Client: Battery updates
{
  type: "BATTERY_UPDATE",
  batteryLevel: int,
  timestamp: long
}

// Server -> Client: Error notifications
{
  type: "ERROR",
  errorCode: string,
  message: string,
  timestamp: long
}
```

## 5. High-Level Design (HLD)

### 5.1 Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                    ROBOT HARDWARE                            │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  LIDAR   │  │ Cameras  │  │  Cliff   │  │  Bumper  │   │
│  │  Sensor  │  │  (Visual)│  │ Sensors  │  │  Sensors │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
│       │             │             │             │          │
│       └─────────────┼─────────────┼─────────────┘          │
│                     │             │                        │
│  ┌──────────┐  ┌────▼─────┐  ┌───▼──────┐  ┌──────────┐   │
│  │ Wheels & │  │  MCU     │  │  IMU     │  │ Battery  │   │
│  │  Motors  │  │(Control) │  │(Gyro/    │  │Management│   │
│  └────┬─────┘  └────┬─────┘  │Accel)    │  └──────────┘   │
│       │             │        └──────────┘                  │
│       └─────────────┼────────────────────────────┐         │
│                     │                            │         │
└─────────────────────┼────────────────────────────┼─────────┘
                      │                            │
              ┌───────▼────────────────────────────▼─────┐
              │    Embedded Computer (Raspberry Pi /     │
              │         Jetson Nano)                     │
              │                                          │
              │  ┌──────────────┐  ┌──────────────┐     │
              │  │   SLAM       │  │    Path      │     │
              │  │  Algorithm   │  │   Planning   │     │
              │  └──────────────┘  └──────────────┘     │
              │  ┌──────────────┐  ┌──────────────┐     │
              │  │  Obstacle    │  │   Control    │     │
              │  │  Detection   │  │   System     │     │
              │  └──────────────┘  └──────────────┘     │
              └───────┬──────────────────────┬───────────┘
                      │                      │
                      │ WiFi / BLE           │
                      │                      │
┌─────────────────────┼──────────────────────┼──────────────┐
│               CLOUD BACKEND                                │
│               ┌─────▼──────────────────────▼─────┐         │
│               │   Load Balancer / API Gateway    │         │
│               └─────┬────────────────────────────┘         │
│                     │                                      │
│    ┌────────────────┼────────────────┐                    │
│    │                │                │                    │
│ ┌──▼─────┐    ┌─────▼────┐    ┌─────▼────┐               │
│ │ Robot  │    │   Map    │    │Schedule  │               │
│ │Service │    │ Service  │    │ Service  │               │
│ └──┬─────┘    └─────┬────┘    └─────┬────┘               │
│    │                │                │                    │
│ ┌──▼─────┐    ┌─────▼────┐    ┌─────▼────┐               │
│ │Analytics│   │  User    │    │  Notif.  │               │
│ │ Service│    │ Service  │    │ Service  │               │
│ └────────┘    └──────────┘    └──────────┘               │
│                                                           │
└──────────────────────┬────────────────────────────────────┘
                       │
┌──────────────────────┼────────────────────────────────────┐
│                 DATA LAYER                                │
│    ┌──────────┐  ┌──▼─────────┐  ┌──────────────┐        │
│    │PostgreSQL│  │   Redis    │  │    S3        │        │
│    │  (Main   │  │  (Cache,   │  │  (Map Data,  │        │
│    │   DB)    │  │  Sessions) │  │   Logs)      │        │
│    └──────────┘  └────────────┘  └──────────────┘        │
│    ┌──────────────────────────┐  ┌──────────────┐        │
│    │    TimeSeries DB         │  │    Kafka     │        │
│    │  (Sensor Data, Metrics)  │  │  (Events)    │        │
│    └──────────────────────────┘  └──────────────┘        │
└───────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────┐
│                   MOBILE APP                              │
│         ┌─────────────────────────────┐                   │
│         │   iOS / Android App         │                   │
│         │  - Robot Control            │                   │
│         │  - Map Visualization        │                   │
│         │  - Scheduling               │                   │
│         └─────────────────────────────┘                   │
└───────────────────────────────────────────────────────────┘
```

### 5.2 SLAM (Simultaneous Localization and Mapping)

**Algorithm**: Typically uses one of:
- **Grid-based SLAM**: Occupancy grid
- **Graph-based SLAM**: Pose graphs
- **Visual SLAM**: Camera-based
- **Lidar SLAM**: Laser-based (most common for robot vacuums)

**Process**:
```
1. Sensor Reading (LIDAR, Camera, IMU)
     │
     ▼
2. Feature Extraction
     │
     ▼
3. Data Association (match with existing map)
     │
     ▼
4. Pose Estimation (where am I?)
     │
     ▼
5. Map Update (update occupancy grid)
     │
     ▼
6. Loop Closure Detection (recognize revisited places)
```

### 5.3 Path Planning

**Algorithm**: Typically uses:
- **A\***: For optimal path from A to B
- **RRT (Rapidly-exploring Random Tree)**: For complex spaces
- **Coverage Path Planning**: For complete area coverage

**Coverage Patterns**:
```
┌─────────────────────────────────┐
│  Boustrophedon (Zigzag) Pattern │
│                                 │
│  ═══════════════════════>       │
│       <═══════════════════      │
│  ═══════════════════════>       │
│       <═══════════════════      │
│  ═══════════════════════>       │
└─────────────────────────────────┘

Benefits:
- Complete coverage
- Minimal overlap
- Efficient
```

### 5.4 Obstacle Avoidance

**Sensors**:
- LIDAR: 360° range finding
- Cameras: Visual identification
- Cliff sensors: Detect stairs/drops
- Bumper sensors: Physical contact
- Ultrasonic: Close-range detection

**Reactive vs Deliberative**:
```
Reactive (Real-time):
- Immediate response to obstacles
- Bumper hit -> reverse and turn
- Cliff detected -> stop immediately

Deliberative (Planning):
- Anticipate obstacles from map
- Plan path around known obstacles
- Update map with new obstacles
```

## 6. Optimizations & Approaches

1. SLAM (Simultaneous Localization and Mapping) Optimization: 
 Use FastSLAM with particle filters to localize efficiently while updating the map in real time.

2. Coverage Path Planning:
 Use boustrophedon (zig-zag) or A*-based coverage to minimize overlap and ensure full area coverage.

3. Battery Management:
 Predict remaining runtime and trigger auto-docking before battery drops below a safe threshold.

4. Multi-Room Sequencing: 
Order rooms using shortest-path or nearest-neighbor heuristic to reduce travel distance.

5. Dynamic Obstacle Handling:
 Combine real-time sensor detection with reactive avoidance and path replanning.

6. Carpet Detection & Boost:
 Detect carpet via sensor feedback and dynamically increase suction power.

7. Error Recovery:
 Use retry, fallback behaviors, and safe-state transitions to recover from navigation or hardware errors.

8. Map Storage Optimization: 
 Store maps as sparse grids with compression to reduce memory footprint.

9. Localization Without GPS:
 Use SLAM with LIDAR/IMU sensor fusion to estimate pose indoors.

10. Firmware Updates (OTA):
 Perform staged, checksum-verified updates with rollback support on failure.



## 7. Low-Level Design (LLD)

<img width="2297" height="825" alt="image" src="https://github.com/user-attachments/assets/7357a6ea-4fbc-46b3-858b-111766056d9d" />

### 7.2 Scalability Calculations

**Computational Requirements**:
- **SLAM Processing**: 10-20 Hz (50-100ms per update)
- **Path Planning**: 1-5 Hz (200ms-1s)
- **Obstacle Detection**: 20-50 Hz (20-50ms)
- **Motor Control**: 50-100 Hz (10-20ms)

**Memory Requirements**:
- Map (100x100 grid): 10KB
- Particle filter (1000 particles): 50KB
- Path (1000 waypoints): 12KB
- Total: ~100KB (easily fits in embedded system)

**Storage**:
- Map data: 10KB per floor
- Cleaning history (1 year): ~10MB
- Firmware: ~100MB

**Network**:
- Status updates: 1KB/s
- Map sync: 10KB (occasional)
- Firmware update: 100MB (rare)

---

## Interview Discussion Points

### Clarifying Questions:
1. Home size: Single room? Multi-room? Multiple floors?
2. Obstacle complexity: Simple or cluttered environment?
3. Budget: Consumer vs industrial grade?
4. Special features: Mopping? UV sterilization?
5. Connectivity: WiFi only? Offline operation?

### Trade-offs:
1. **LIDAR vs Camera SLAM**: Cost vs accuracy
2. **Random vs Systematic cleaning**: Simple vs efficient
3. **Cloud vs Local processing**: Latency vs power
4. **Battery size vs Weight**: Runtime vs maneuverability

### Extensions:
1. **Multi-Robot Coordination**: Multiple robots working together
2. **Object Recognition**: AI to identify and avoid specific objects
3. **Voice Control**: Integration with Alexa/Google Home
4. **Mopping**: Wet cleaning capability
5. **Self-Emptying**: Auto-empty dustbin at dock

This comprehensive design covers the essential aspects of a robotic vacuum cleaner system from hardware to cloud backend!

