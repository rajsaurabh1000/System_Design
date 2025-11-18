# Robotic Vacuum Cleaner - System Design

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

**Dynamic Replanning**:
```python
while cleaning:
    if new_obstacle_detected():
        replan_path(current_position, goal)
    
    if battery_low():
        return_to_dock()
    
    move_along_path()
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

### 6.1 SLAM Optimization

**FastSLAM Algorithm**:
```python
class FastSLAM:
    def __init__(self, num_particles=100):
        self.particles = [Particle() for _ in range(num_particles)]
    
    def update(self, sensor_data, odometry):
        # 1. Prediction: Update particle poses based on odometry
        for particle in self.particles:
            particle.predict(odometry)
        
        # 2. Update: Weight particles based on sensor measurements
        for particle in self.particles:
            particle.weight = self.calculate_weight(
                particle,
                sensor_data
            )
        
        # 3. Resampling: Keep high-weight particles
        self.particles = self.resample(self.particles)
        
        # 4. Map update: Update map for each particle
        for particle in self.particles:
            particle.update_map(sensor_data)
    
    def get_best_estimate(self):
        # Return particle with highest weight
        return max(self.particles, key=lambda p: p.weight)
```

### 6.2 Coverage Path Planning

**Efficient Coverage Algorithm**:
```python
def coverage_path_planning(occupancy_grid):
    """
    Boustrophedon cell decomposition
    """
    # 1. Decompose free space into cells
    cells = decompose_into_cells(occupancy_grid)
    
    # 2. Build adjacency graph of cells
    graph = build_cell_graph(cells)
    
    # 3. Find optimal order to visit cells (TSP-like)
    cell_order = find_optimal_cell_order(graph)
    
    # 4. Generate zigzag path within each cell
    path = []
    for cell in cell_order:
        zigzag_path = generate_zigzag_path(cell)
        path.extend(zigzag_path)
    
    return path

def generate_zigzag_path(cell):
    """Generate back-and-forth pattern"""
    path = []
    y = cell.min_y
    direction = 1  # 1 = right, -1 = left
    
    while y <= cell.max_y:
        if direction == 1:
            # Move right
            for x in range(cell.min_x, cell.max_x + 1):
                path.append((x, y))
        else:
            # Move left
            for x in range(cell.max_x, cell.min_x - 1, -1):
                path.append((x, y))
        
        y += 1
        direction *= -1
    
    return path
```

### 6.3 Battery Management

**Smart Battery Algorithm**:
```python
def should_return_to_dock(robot):
    # Estimate remaining battery needed
    current_position = robot.get_position()
    dock_position = robot.get_dock_position()
    
    # Distance to dock
    distance_to_dock = calculate_distance(current_position, dock_position)
    
    # Estimate energy needed to return
    energy_to_return = distance_to_dock * ENERGY_PER_METER
    energy_margin = 0.2  # 20% safety margin
    
    required_energy = energy_to_return * (1 + energy_margin)
    
    # Check if current battery is enough
    if robot.battery_level < required_energy:
        return True
    
    # Also consider remaining cleaning area
    uncleaned_area = calculate_uncleaned_area(robot.map)
    energy_to_clean = uncleaned_area * ENERGY_PER_SQM
    
    total_required = energy_to_return + energy_to_clean
    
    if robot.battery_level < total_required:
        # Not enough to finish, return now
        return True
    
    return False

def resume_after_charging(robot):
    """Resume cleaning from where it left off"""
    if robot.battery_level >= 0.8:  # 80% charged
        # Get last position before docking
        last_position = robot.get_last_cleaning_position()
        
        # Navigate back
        robot.navigate_to(last_position)
        
        # Resume cleaning
        robot.resume_cleaning()
```

### 6.4 Multi-Room Sequencing

**Room Ordering Strategy**:
```python
def optimize_room_sequence(rooms, current_position):
    """
    Order rooms to minimize travel distance
    Similar to TSP (Traveling Salesman Problem)
    """
    # Use greedy nearest-neighbor heuristic
    unvisited = set(rooms)
    sequence = []
    current_pos = current_position
    
    while unvisited:
        # Find nearest unvisited room
        nearest_room = min(unvisited, 
            key=lambda r: distance(current_pos, r.center))
        
        sequence.append(nearest_room)
        unvisited.remove(nearest_room)
        current_pos = nearest_room.center
    
    return sequence
```

### 6.5 Dynamic Obstacle Handling

**Scenario**: Pet or person walks in front of robot

**Approach**:
```python
def handle_dynamic_obstacle():
    if obstacle_detected():
        # 1. Stop immediately
        robot.stop()
        
        # 2. Wait for obstacle to move
        wait_time = 0
        max_wait = 30  # seconds
        
        while obstacle_still_present() and wait_time < max_wait:
            time.sleep(1)
            wait_time += 1
        
        if wait_time >= max_wait:
            # 3. Try to navigate around
            alternate_path = find_alternate_path()
            if alternate_path:
                robot.follow_path(alternate_path)
            else:
                # 4. Mark area as temporarily inaccessible
                mark_area_blocked(current_area)
                move_to_next_area()
        else:
            # Obstacle moved, continue
            robot.resume()
```

### 6.6 Carpet Detection & Boost

**Implementation**:
```python
def detect_floor_type():
    # Use motor current draw as proxy
    # Carpet requires more power
    
    current_draw = motor.get_current()
    
    if current_draw > CARPET_THRESHOLD:
        return FLOOR_TYPE_CARPET
    else:
        return FLOOR_TYPE_HARD
    
def adjust_suction_power():
    floor_type = detect_floor_type()
    
    if floor_type == FLOOR_TYPE_CARPET:
        # Boost suction on carpet
        robot.set_suction_power(HIGH)
    else:
        # Normal suction on hard floors
        robot.set_suction_power(MEDIUM)
```

### 6.7 Error Recovery

**Common Errors & Recovery**:
```python
class ErrorRecovery:
    def handle_error(self, error_code):
        if error_code == ERROR_STUCK:
            return self.recover_from_stuck()
        elif error_code == ERROR_WHEEL_SUSPENDED:
            return self.recover_from_suspension()
        elif error_code == ERROR_DUSTBIN_FULL:
            return self.notify_user_dustbin_full()
        elif error_code == ERROR_BRUSH_JAMMED:
            return self.recover_from_jammed_brush()
    
    def recover_from_stuck(self):
        # Try multiple strategies
        strategies = [
            self.reverse_and_turn,
            self.wiggle_movement,
            self.call_for_help
        ]
        
        for strategy in strategies:
            success = strategy()
            if success:
                return True
        
        return False
    
    def reverse_and_turn(self):
        robot.move_backward(duration=2)
        robot.turn(angle=45)
        robot.move_forward(duration=1)
        return not robot.is_stuck()
    
    def wiggle_movement(self):
        for _ in range(5):
            robot.turn(angle=15)
            robot.turn(angle=-30)
            robot.turn(angle=15)
            if not robot.is_stuck():
                return True
        return False
    
    def call_for_help(self):
        robot.notify_user("Robot is stuck, please assist")
        robot.play_sound("help_needed.wav")
        return False
```

### 6.8 Map Storage Optimization

**Compression**:
```python
def compress_map(occupancy_grid):
    """
    Use run-length encoding for sparse maps
    """
    compressed = []
    current_value = occupancy_grid[0][0]
    count = 1
    
    for row in occupancy_grid:
        for cell in row:
            if cell == current_value:
                count += 1
            else:
                compressed.append((current_value, count))
                current_value = cell
                count = 1
    
    compressed.append((current_value, count))
    return compressed

# Typical map: 100x100 = 10,000 cells
# Uncompressed: 10,000 bytes
# Compressed (sparse): ~1,000 bytes (10x reduction)
```

### 6.9 Localization Without GPS

**Monte Carlo Localization (Particle Filter)**:
```python
def localize(sensor_data, map):
    # Use particle filter for localization
    
    # 1. Initialize particles across possible positions
    particles = initialize_particles(map, num=1000)
    
    # 2. Update based on sensor data
    for particle in particles:
        # Predict: apply motion model
        particle.predict(odometry_data)
        
        # Update: compare particle's expected sensor data
        # with actual sensor data
        expected_reading = simulate_sensor(particle.pose, map)
        particle.weight = similarity(expected_reading, sensor_data)
    
    # 3. Resample: keep high-probability particles
    particles = resample_particles(particles)
    
    # 4. Estimate: weighted average of particles
    estimated_pose = compute_weighted_average(particles)
    
    return estimated_pose
```

### 6.10 Firmware Updates (OTA)

**Safe Update Strategy**:
```python
def update_firmware(new_firmware_url):
    # 1. Check battery level (must be > 50%)
    if robot.battery_level < 0.5:
        return "Battery too low for update"
    
    # 2. Check if docked (safe location)
    if not robot.is_docked():
        robot.return_to_dock()
        robot.wait_until_docked()
    
    # 3. Download firmware
    firmware_data = download_firmware(new_firmware_url)
    
    # 4. Verify checksum
    if not verify_checksum(firmware_data):
        return "Firmware verification failed"
    
    # 5. Create backup of current firmware
    backup_current_firmware()
    
    # 6. Install new firmware
    try:
        install_firmware(firmware_data)
        robot.reboot()
        
        # 7. Verify installation
        if verify_installation():
            return "Update successful"
        else:
            rollback_firmware()
            return "Update failed, rolled back"
    
    except Exception as e:
        rollback_firmware()
        return f"Update failed: {str(e)}"
```

## 7. Low-Level Design (LLD)

### 7.1 Class Diagram

```
┌──────────────────────────────────────────────────────────┐
│                      Robot                               │
├──────────────────────────────────────────────────────────┤
│ - robotId: UUID                                          │
│ - state: RobotState                                      │
│ - batteryLevel: float                                    │
│ - position: Pose                                         │
│ - navigator: Navigator                                   │
│ - mapper: Mapper                                         │
│ - cleaner: CleanerModule                                 │
├──────────────────────────────────────────────────────────┤
│ + startCleaning(mode: CleaningMode): void                │
│ + stopCleaning(): void                                   │
│ + returnToDock(): void                                   │
│ + updatePosition(pose: Pose): void                       │
│ + handleObstacle(obstacle: Obstacle): void               │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                    Navigator                             │
├──────────────────────────────────────────────────────────┤
│ - pathPlanner: PathPlanner                               │
│ - obstacleAvoider: ObstacleAvoider                       │
│ - currentPath: List<Pose>                                │
├──────────────────────────────────────────────────────────┤
│ + planPath(start: Pose, goal: Pose): List<Pose>          │
│ + followPath(path: List<Pose>): void                     │
│ + avoidObstacle(obstacle: Obstacle): void                │
│ + replanPath(): void                                     │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                     Mapper (SLAM)                        │
├──────────────────────────────────────────────────────────┤
│ - occupancyGrid: OccupancyGrid                           │
│ - slamAlgorithm: SLAMAlgorithm                           │
│ - loopClosureDetector: LoopClosureDetector               │
├──────────────────────────────────────────────────────────┤
│ + updateMap(sensorData: SensorData): void                │
│ + localize(sensorData: SensorData): Pose                 │
│ + saveMap(): void                                        │
│ + loadMap(mapId: UUID): void                             │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                 SensorFusion                             │
├──────────────────────────────────────────────────────────┤
│ - lidarSensor: LidarSensor                               │
│ - camera: Camera                                         │
│ - imu: IMU                                               │
│ - cliffSensors: List<CliffSensor>                        │
│ - bumper: BumperSensor                                   │
├──────────────────────────────────────────────────────────┤
│ + getFusedData(): SensorData                             │
│ + detectObstacles(): List<Obstacle>                      │
│ + detectCliff(): boolean                                 │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                  PathPlanner                             │
├──────────────────────────────────────────────────────────┤
│ - map: OccupancyGrid                                     │
│ - algorithm: PathPlanningAlgorithm                       │
├──────────────────────────────────────────────────────────┤
│ + planCoveragePath(): List<Pose>                         │
│ + planPointToPoint(start, goal): List<Pose>              │
│ + generateZigzagPath(room: Room): List<Pose>             │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                 CleanerModule                            │
├──────────────────────────────────────────────────────────┤
│ - vacuumMotor: VacuumMotor                               │
│ - brush: Brush                                           │
│ - suctionPower: SuctionLevel                             │
├──────────────────────────────────────────────────────────┤
│ + startVacuum(): void                                    │
│ + stopVacuum(): void                                     │
│ + setSuctionPower(level: SuctionLevel): void             │
│ + isDustbinFull(): boolean                               │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                  MotionController                        │
├──────────────────────────────────────────────────────────┤
│ - leftWheel: Motor                                       │
│ - rightWheel: Motor                                      │
│ - currentVelocity: Velocity                              │
├──────────────────────────────────────────────────────────┤
│ + move(velocity: Velocity): void                         │
│ + turn(angle: float): void                               │
│ + stop(): void                                           │
│ + getOdometry(): Odometry                                │
└──────────────────────────────────────────────────────────┘
```

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

