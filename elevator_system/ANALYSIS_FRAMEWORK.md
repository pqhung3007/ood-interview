# 08

## Design an Elevator System

An Elevator System is one of the classic system design problems that tests a candidate's understanding of **scheduling algorithms**, **real-time state management**, and **optimization trade-offs**. Unlike simple request-response systems, elevators must make continuous decisions about which requests to serve next while physically moving through space—a problem directly analogous to disk scheduling in operating systems.

This design presents unique challenges around **dispatching algorithms** (which elevator serves which request), **direction optimization** (minimizing travel time), and **fairness vs efficiency trade-offs**—making it excellent for testing algorithmic thinking in a system design context.

---

## Requirements Gathering

Before diving into the design, let's clarify the requirements through a Q&A format. Pay attention to the **algorithmic trade-offs** and **edge cases**.

### Requirements Clarification

**Q: What are the basic components of an elevator system?**
A: The system consists of:
- Multiple elevator cars, each with current floor, direction, and destination queue
- Hallway button panels on each floor (UP/DOWN buttons)
- In-car button panels (floor selection buttons)
- A central dispatch controller that assigns requests to elevators

**Q: What triggers an elevator request?**
A: Two types of requests:
1. **Hall calls**: Person on floor X presses UP or DOWN button
2. **Car calls**: Person inside elevator presses destination floor button

**Q: How should the system decide which elevator serves a hall call?**
A: This is the core dispatching problem. We need a **dispatching strategy** that considers:
- Current position of each elevator
- Current direction (is the elevator heading toward the request?)
- Current load/queue of each elevator
- Fairness (don't starve any request)

**Q: What dispatching strategies should we support?**
A: The system should be extensible to multiple strategies:
1. **First Come First Serve (FCFS)**: Simple, fair, but inefficient
2. **Shortest Seek Time First (SSTF)**: Minimize distance, but risks starvation
3. **SCAN/LOOK (Elevator Algorithm)**: Move in one direction until no more requests, then reverse

**Q: What happens when an elevator is already moving in a direction?**
A: This is critical for efficiency. An elevator moving UP should preferably:
- Pick up hall calls going UP that are on its path
- NOT reverse direction for a DOWN call (inefficient detour)

**Q: Should all elevators serve all floors?**
A: Not necessarily. Large buildings often have:
- Low-rise elevators (floors 1-20)
- High-rise elevators (floors 20-50)
- Express elevators (lobby to sky lobby)

This requires **accessible floor constraints** per elevator.

**Q: What about elevator capacity?**
A: Out of scope for this design, but worth mentioning. Real systems track passenger count or weight to skip floors when full.

**Q: How does the system handle concurrent requests?**
A: Multiple hall calls can arrive simultaneously. The dispatcher must handle this without race conditions.

### Requirements State

Based on the clarifications above, here are our finalized requirements:

| Requirement | Status | Complexity |
|-------------|--------|------------|
| Multiple elevator cars with state tracking | ✅ Confirmed | Medium |
| Hall call requests (floor + direction) | ✅ Confirmed | Low |
| Car call requests (destination floor) | ✅ Confirmed | Low |
| Pluggable dispatching strategies | ✅ Confirmed | **High** |
| First Come First Serve (FCFS) strategy | ✅ Confirmed | Low |
| Shortest Seek Time First (SSTF) strategy | ✅ Confirmed | Medium |
| Direction-aware dispatching | ✅ Confirmed | **High** |
| Accessible floor constraints per elevator | ✅ Confirmed | Medium |
| Observer pattern for button events | ✅ Confirmed | Medium |
| Elevator capacity tracking | ❌ Out of scope | — |
| SCAN/LOOK algorithm | ❌ Out of scope (extension) | — |
| Emergency stop / fire mode | ❌ Out of scope | — |

---

## Identify Core Objects

From the requirements, we can identify the following core objects:

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           ELEVATOR SYSTEM OBJECTS                                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         CORE COMPONENTS                                      │   │
│  │  ┌────────────────┐    ┌────────────────┐    ┌────────────────┐             │   │
│  │  │ ElevatorSystem │───►│  ElevatorCar   │    │ ElevatorStatus │             │   │
│  │  │    (Facade)    │ 1:N│                │───►│   (Immutable)  │             │   │
│  │  │                │    │- status        │    │                │             │   │
│  │  │- elevators     │    │- targetFloors  │    │- currentFloor  │             │   │
│  │  │- dispatcher    │    │- accessible    │    │- direction     │             │   │
│  │  │                │    │                │    │                │             │   │
│  │  │+requestElevator│    │+addFloorRequest│    │                │             │   │
│  │  │+selectFloor    │    │+moveOneStep    │    │                │             │   │
│  │  └────────────────┘    └────────────────┘    └────────────────┘             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                      DISPATCHING DOMAIN (Strategy Pattern)                   │   │
│  │  ┌──────────────────┐    ┌──────────────────────┐                           │   │
│  │  │ ElevatorDispatch │───►│ DispatchingStrategy  │                           │   │
│  │  │                  │    │    <<interface>>     │                           │   │
│  │  │- strategy       │    ├──────────────────────┤                           │   │
│  │  │                  │    │ +selectElevator()    │                           │   │
│  │  │+dispatchCar()   │    └──────────┬───────────┘                           │   │
│  │  └──────────────────┘               │                                       │   │
│  │                          ┌──────────┴──────────┐                            │   │
│  │                          │                     │                            │   │
│  │                          ▼                     ▼                            │   │
│  │                   ┌────────────┐        ┌────────────┐                      │   │
│  │                   │    FCFS    │        │    SSTF    │                      │   │
│  │                   │  Strategy  │        │  Strategy  │                      │   │
│  │                   └────────────┘        └────────────┘                      │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                      OBSERVER PATTERN (Button Events)                        │   │
│  │  ┌──────────────────┐    ┌──────────────────┐                               │   │
│  │  │HallwayButtonPanel│───►│ ElevatorObserver │                               │   │
│  │  │   (Subject)      │ 1:N│  <<interface>>   │                               │   │
│  │  │                  │    ├──────────────────┤                               │   │
│  │  │- floor           │    │ +update(floor,   │                               │   │
│  │  │- observers       │    │         direction)│                              │   │
│  │  │                  │    └────────┬─────────┘                               │   │
│  │  │+pressButton()    │             │                                         │   │
│  │  │+addObserver()    │             ▼                                         │   │
│  │  └──────────────────┘    ┌──────────────────┐                               │   │
│  │                          │DispatchController│                               │   │
│  │                          │  (Observer Impl) │                               │   │
│  │                          └──────────────────┘                               │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         DIRECTION (State)                                    │   │
│  │                    ┌────────────────────┐                                   │   │
│  │                    │     Direction      │                                   │   │
│  │                    │      (enum)        │                                   │   │
│  │                    ├────────────────────┤                                   │   │
│  │                    │  UP | DOWN | IDLE  │                                   │   │
│  │                    └────────────────────┘                                   │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

**Core objects identified:**

1. **ElevatorSystem**: Facade coordinating elevators and dispatching
2. **ElevatorCar**: Physical elevator with position, direction, and request queue
3. **ElevatorStatus**: Immutable snapshot of elevator state (floor + direction)
4. **Direction**: Enum (UP, DOWN, IDLE) representing elevator movement state
5. **ElevatorDispatch**: Applies strategy to assign requests to elevators
6. **DispatchingStrategy**: Interface for pluggable algorithms
7. **FirstComeFirstServeStrategy**: Simple FCFS implementation
8. **ShortestSeekTimeFirstStrategy**: Distance-optimized implementation
9. **HallwayButtonPanel**: Observable subject for floor buttons
10. **ElevatorObserver**: Observer interface for button events
11. **ElevatorDispatchController**: Observer that handles button press events

---

## Design Class Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                                  CLASS DIAGRAM                                        │
└──────────────────────────────────────────────────────────────────────────────────────┘

                            ┌─────────────────────┐
                            │   ElevatorSystem    │
                            │      (Facade)       │
                            ├─────────────────────┤
                            │ - elevators         │
                            │ - dispatchController│
                            ├─────────────────────┤
                            │ + getAllStatuses()  │
                            │ + requestElevator() │
                            │ + selectFloor()     │
                            └──────────┬──────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
                    ▼                  ▼                  ▼
            ┌─────────────┐   ┌────────────────┐   ┌─────────────┐
            │ ElevatorCar │   │ElevatorDispatch│   │   Direction │
            ├─────────────┤   ├────────────────┤   │   (enum)    │
            │- status     │   │- strategy      │   ├─────────────┤
            │- targetQueue│   ├────────────────┤   │ UP          │
            │- accessible │   │+dispatchCar()  │   │ DOWN        │
            ├─────────────┤   └───────┬────────┘   │ IDLE        │
            │+addRequest()│           │            └─────────────┘
            │+moveOneStep()           │
            │+isIdle()    │           │ uses
            │+isAtDest()  │           ▼
            └──────┬──────┘   ┌────────────────────┐
                   │          │DispatchingStrategy │
                   │          │   <<interface>>    │
                   │          ├────────────────────┤
                   │          │+selectElevator()   │
                   │          └─────────┬──────────┘
                   │                    │
                   │         ┌──────────┴──────────┐
                   │         │                     │
                   │         ▼                     ▼
                   │   ┌───────────┐        ┌───────────┐
                   │   │   FCFS    │        │   SSTF    │
                   │   │ Strategy  │        │ Strategy  │
                   │   └───────────┘        └───────────┘
                   │
                   ▼
            ┌─────────────────┐
            │ ElevatorStatus  │
            │  (immutable)    │
            ├─────────────────┤
            │- currentFloor   │
            │- direction      │
            └─────────────────┘


┌──────────────────────────────────────────────────────────────────────────────────────┐
│                      DISPATCHING ALGORITHM COMPARISON                                 │
└──────────────────────────────────────────────────────────────────────────────────────┘

  SCENARIO: Elevator at floor 5, requests at floors 2, 8, 3, 9

  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │                    FIRST COME FIRST SERVE (FCFS)                                 │
  ├─────────────────────────────────────────────────────────────────────────────────┤
  │                                                                                 │
  │  Queue order: [2, 8, 3, 9]                                                      │
  │                                                                                 │
  │  Movement:  5 → 2 → 8 → 3 → 9                                                   │
  │  Distance:  3  + 6  + 5  + 6  = 20 floors traveled                              │
  │                                                                                 │
  │  ✓ Fair: serves in request order                                               │
  │  ✗ Inefficient: lots of direction changes                                       │
  │                                                                                 │
  └─────────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │                  SHORTEST SEEK TIME FIRST (SSTF)                                 │
  ├─────────────────────────────────────────────────────────────────────────────────┤
  │                                                                                 │
  │  At floor 5, nearest is 3 (distance 2)                                          │
  │  At floor 3, nearest is 2 (distance 1)                                          │
  │  At floor 2, nearest is 8 (distance 6)                                          │
  │  At floor 8, nearest is 9 (distance 1)                                          │
  │                                                                                 │
  │  Movement:  5 → 3 → 2 → 8 → 9                                                   │
  │  Distance:  2  + 1  + 6  + 1  = 10 floors traveled                              │
  │                                                                                 │
  │  ✓ Efficient: minimizes total travel                                           │
  │  ✗ Starvation risk: far requests may wait indefinitely                          │
  │                                                                                 │
  └─────────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │                      SCAN / ELEVATOR ALGORITHM                                   │
  ├─────────────────────────────────────────────────────────────────────────────────┤
  │                                                                                 │
  │  Currently moving UP from floor 5                                               │
  │                                                                                 │
  │  Phase 1 (UP): 5 → 8 → 9 (serve all UP requests)                                │
  │  Phase 2 (DOWN): 9 → 3 → 2 (reverse, serve DOWN requests)                       │
  │                                                                                 │
  │  Movement:  5 → 8 → 9 → 3 → 2                                                   │
  │  Distance:  3  + 1  + 6  + 1  = 11 floors traveled                              │
  │                                                                                 │
  │  ✓ Balanced: efficient and fair                                                │
  │  ✓ No starvation: bounded wait time                                            │
  │  ✓ Predictable: passengers know elevator is coming                              │
  │                                                                                 │
  └─────────────────────────────────────────────────────────────────────────────────┘


┌──────────────────────────────────────────────────────────────────────────────────────┐
│                           OBSERVER PATTERN FOR BUTTONS                                │
└──────────────────────────────────────────────────────────────────────────────────────┘

    ┌────────────────┐         ┌────────────────┐
    │    Subject     │         │   Observer     │
    │ (Observable)   │         │  (Interface)   │
    ├────────────────┤         ├────────────────┤
    │+addObserver()  │         │+update()       │
    │+notify()       │         └───────┬────────┘
    └───────┬────────┘                 │
            │                          │
            ▼                          ▼
    ┌────────────────┐         ┌────────────────┐
    │HallwayButton   │────────►│DispatchController
    │    Panel       │ notifies│  (Observer)    │
    ├────────────────┤         ├────────────────┤
    │- floor         │         │+update(floor,  │
    │- observers     │         │        dir)    │
    │                │         └────────────────┘
    │+pressButton(dir│
    └────────────────┘

    Flow:
    1. User presses UP button on floor 5
    2. HallwayButtonPanel.pressButton(UP)
    3. Panel notifies all observers: update(5, UP)
    4. DispatchController receives notification
    5. Controller invokes dispatch logic
```

---

## Code

The Elevator System is implemented using the **Strategy Pattern** for dispatching algorithms, **Observer Pattern** for button events, and careful state management for elevator movement.

### Deep Dive Topics

#### The Elevator Scheduling Problem: A Classic CS Challenge

Elevator scheduling is directly analogous to disk scheduling in operating systems. The "elevator" moves a read/write head (or car) to different positions (floors/tracks) to serve requests.

**Key insight:** The problem is fundamentally about **minimizing total seek time** while ensuring **fairness** (no request waits forever).

| Disk Scheduling | Elevator Scheduling |
|-----------------|---------------------|
| Read/write head position | Current floor |
| Track number | Floor number |
| Seek time | Travel time |
| Disk arm movement | Elevator direction |

**Reference:** This is why the SCAN algorithm is also called the "Elevator Algorithm" in OS literature.

---

#### FCFS Strategy: Simple But Inefficient

The First Come First Serve strategy is the simplest approach:

```java
public class FirstComeFirstServeStrategy implements DispatchingStrategy {
    @Override
    public ElevatorCar selectElevator(List<ElevatorCar> elevators, int floor, Direction direction) {
        for (ElevatorCar elevator : elevators) {
            // Prefer idle elevators or those going the same direction
            if (elevator.isIdle() || elevator.getCurrentDirection() == direction) {
                return elevator;
            }
        }
        // Fallback: random selection
        return elevators.get((int) (Math.random() * elevators.size()));
    }
}
```

**Analysis:**

| Metric | FCFS Performance |
|--------|------------------|
| Time complexity | O(n) where n = number of elevators |
| Fairness | High (serves in order) |
| Efficiency | Low (ignores distance) |
| Starvation risk | None |
| Best for | Low-traffic buildings |

**Problem:** The random fallback when no suitable elevator exists can lead to unpredictable behavior.

**Reference:** `elevator/dispatch/FirstComeFirstServeStrategy.java:8-18`

---

#### SSTF Strategy: Optimized But Risky

The Shortest Seek Time First strategy optimizes for distance:

```java
public class ShortestSeekTimeFirstStrategy implements DispatchingStrategy {
    @Override
    public ElevatorCar selectElevator(List<ElevatorCar> elevators, int floor, Direction direction) {
        ElevatorCar bestElevator = null;
        int shortestDistance = Integer.MAX_VALUE;

        for (ElevatorCar elevator : elevators) {
            int distance = Math.abs(elevator.getCurrentFloor() - floor);

            // Must be idle or going same direction
            if ((elevator.isIdle() || elevator.getCurrentDirection() == direction)
                && distance < shortestDistance) {
                bestElevator = elevator;
                shortestDistance = distance;
            }
        }
        return bestElevator;
    }
}
```

**Analysis:**

| Metric | SSTF Performance |
|--------|------------------|
| Time complexity | O(n) where n = number of elevators |
| Fairness | Low (can starve distant requests) |
| Efficiency | High (minimizes immediate travel) |
| Starvation risk | **High** |
| Best for | Moderate traffic, spread-out requests |

**The Starvation Problem:**

```
Elevator at floor 10, continuous requests at floors 9, 11, 10, 12...

Floor 1 request waits indefinitely because closer requests keep arriving!

Solution: Add "aging" to increase priority of waiting requests over time.
```

**Reference:** `elevator/dispatch/ShortestSeekTimeFirstStrategy.java:8-24`

---

#### Direction-Aware Dispatching: The Key Optimization

Both strategies include direction awareness:

```java
if (elevator.isIdle() || elevator.getCurrentDirection() == direction) {
    // This elevator can serve the request
}
```

**Why this matters:**

```
Elevator at floor 5, moving UP, currently heading to floor 10

Request: Floor 3, going DOWN

Without direction check:
  ✗ Assign to this elevator → Must reverse, go to 3, then back up
  Total detour: 5 floors down + 7 floors up = 12 extra floors

With direction check:
  ✓ Skip this elevator → Let another idle elevator handle it
  This elevator continues to 10 → No detour
```

**Business impact:** Direction-aware dispatching can reduce average wait time by 30-40% in high-traffic buildings.

---

#### Accessible Floors: Multi-Zone Elevators

The deep dive implementation adds floor restrictions:

```java
public class ElevatorCar {
    private final Set<Integer> accessibleFloors;

    public void addFloorRequest(int floor) {
        // Only accept requests for accessible floors
        if (accessibleFloors.contains(floor) && !targetFloors.contains(floor)) {
            targetFloors.offer(floor);
            updateDirection(floor);
        }
    }
}
```

**Real-world scenario:**

```
50-floor building with 6 elevators:

Elevators 1-2: Floors 1-20 (low-rise)
Elevators 3-4: Floors 1, 20-40 (mid-rise, express to 20)
Elevators 5-6: Floors 1, 40-50 (high-rise, express to 40)

Benefits:
- Reduces travel time for high floors
- Prevents low-floor traffic from delaying high-floor passengers
- Common in skyscrapers (Empire State Building, Burj Khalifa)
```

**Strategy must also be aware:**

```java
// Enhanced SSTF with floor accessibility
if ((elevator.isIdle() || elevator.getCurrentDirection() == direction)
    && elevator.getAccessibleFloors().contains(floor)  // NEW CHECK
    && distance < shortestDistance) {
    bestElevator = elevator;
}
```

**Reference:** `elevator_deep_dive/ElevatorCar.java:22-27`, `elevator_deep_dive/ShortestSeekTimeFirstStrategy.java:17`

---

#### Observer Pattern: Decoupling Button Events

The system uses Observer pattern to decouple button panels from dispatch logic:

```java
// Subject (Observable)
public class HallwayButtonPanel {
    private final int floor;
    private final List<ElevatorObserver> observers;

    public void pressButton(Direction direction) {
        notifyObservers(direction);
    }

    private void notifyObservers(Direction direction) {
        for (ElevatorObserver observer : observers) {
            observer.update(floor, direction);
        }
    }
}

// Observer interface
public interface ElevatorObserver {
    void update(int floor, Direction direction);
}

// Concrete observer
public class ElevatorDispatchController implements ElevatorObserver {
    @Override
    public void update(int floor, Direction direction) {
        // Dispatch logic here
    }
}
```

**Benefits:**
- Button panels don't know about dispatch logic
- Multiple observers can react to same button press (e.g., logging, display update)
- Easy to add new behaviors without modifying button panel

**Reference:** `elevator/components/HallwayButtonPanel.java`, `elevator/observer/ElevatorObserver.java`

---

### Parking at Code

The following describes the code structure:

```
elevator_system/
├── elevator/
│   ├── ElevatorSystem.java           # Main facade
│   ├── ElevatorSystemTest.java       # Test suite
│   ├── components/
│   │   ├── ElevatorCar.java          # Elevator with state and queue
│   │   ├── ElevatorStatus.java       # Immutable status snapshot
│   │   ├── Direction.java            # UP, DOWN, IDLE enum
│   │   └── HallwayButtonPanel.java   # Observable button panel
│   ├── dispatch/
│   │   ├── ElevatorDispatch.java     # Strategy context
│   │   ├── DispatchingStrategy.java  # Strategy interface
│   │   ├── FirstComeFirstServeStrategy.java
│   │   ├── ShortestSeekTimeFirstStrategy.java
│   │   └── ElevatorDispatchController.java  # Observer implementation
│   └── observer/
│       ├── ElevatorObserver.java     # Observer interface
│       └── ObserverPatternExample.java
└── elevator_deep_dive/
    ├── ElevatorCar.java              # With accessible floors
    └── ShortestSeekTimeFirstStrategy.java  # With floor constraints
```

---

### ElevatorSystem (Facade)

The `ElevatorSystem` provides a unified interface for elevator operations:

```java
public class ElevatorSystem {
    private final List<ElevatorCar> elevators;
    private final ElevatorDispatch dispatchController;

    public void requestElevator(int currentFloor, Direction direction) {
        // Hall call: delegate to dispatcher
        dispatchController.dispatchElevatorCar(currentFloor, direction, elevators);
    }

    public void selectFloor(ElevatorCar car, int destinationFloor) {
        // Car call: directly add to elevator's queue
        car.addFloorRequest(destinationFloor);
    }
}
```

**Key distinction:**
- `requestElevator()`: Hall call → needs dispatching decision
- `selectFloor()`: Car call → directly handled by specific elevator

**Reference:** `elevator/ElevatorSystem.java:1-39`

---

### ElevatorCar (State Management)

The `ElevatorCar` manages its own state and movement:

```java
public class ElevatorCar {
    private ElevatorStatus status;
    private final Queue<Integer> targetFloors;

    public void moveOneStep() {
        if (targetFloors.isEmpty()) return;

        int nextFloor = targetFloors.peek();
        if (status.getCurrentFloor() < nextFloor) {
            status = new ElevatorStatus(status.getCurrentFloor() + 1, Direction.UP);
        } else if (status.getCurrentFloor() > nextFloor) {
            status = new ElevatorStatus(status.getCurrentFloor() - 1, Direction.DOWN);
        } else {
            targetFloors.poll();  // Arrived, remove from queue
            if (targetFloors.isEmpty()) {
                status = new ElevatorStatus(status.getCurrentFloor(), Direction.IDLE);
            }
        }
    }
}
```

**Design note:** `ElevatorStatus` is immutable—each state change creates a new instance. This is safer for concurrent access and debugging.

**Reference:** `elevator/components/ElevatorCar.java:1-80`

---

### Implementing SCAN Algorithm (Extension)

The SCAN (Elevator) algorithm serves all requests in one direction before reversing:

```java
public class ScanStrategy implements DispatchingStrategy {
    @Override
    public ElevatorCar selectElevator(List<ElevatorCar> elevators, int floor, Direction direction) {
        ElevatorCar bestElevator = null;
        int bestScore = Integer.MAX_VALUE;

        for (ElevatorCar elevator : elevators) {
            int score = calculateScore(elevator, floor, direction);
            if (score < bestScore) {
                bestElevator = elevator;
                bestScore = score;
            }
        }
        return bestElevator;
    }

    private int calculateScore(ElevatorCar elevator, int floor, Direction direction) {
        int currentFloor = elevator.getCurrentFloor();
        Direction currentDir = elevator.getCurrentDirection();

        if (elevator.isIdle()) {
            return Math.abs(currentFloor - floor);  // Simple distance
        }

        // Elevator moving in same direction AND will pass the floor
        if (currentDir == direction) {
            if (currentDir == Direction.UP && currentFloor <= floor) {
                return floor - currentFloor;  // Will pick up on the way
            }
            if (currentDir == Direction.DOWN && currentFloor >= floor) {
                return currentFloor - floor;  // Will pick up on the way
            }
        }

        // Must wait for elevator to reverse
        return Integer.MAX_VALUE - 1;  // Low priority
    }
}
```

---

## Wrap Up

In this design, we created an Elevator System that addresses both basic functionality and advanced scheduling challenges. Key takeaways:

1. **Strategy Pattern** for pluggable dispatching algorithms (FCFS, SSTF, SCAN), allowing runtime algorithm selection

2. **Observer Pattern** for button panel events, decoupling UI components from dispatch logic

3. **Facade Pattern** in `ElevatorSystem`, providing clean API for hall calls and car calls

4. **Immutable Status Objects** (`ElevatorStatus`) for safe state management

5. **Algorithm Trade-offs:**
   - FCFS: Fair but inefficient
   - SSTF: Efficient but risks starvation
   - SCAN: Balanced fairness and efficiency

6. **Direction-Aware Dispatching** to avoid unnecessary reversals

7. **Accessible Floor Constraints** for multi-zone elevator systems

---

## Further Reading: Strategy Pattern and Scheduling Algorithms

### Strategy Design Pattern

The Strategy pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable.

**Application in Elevator System:**

```
              ┌──────────────────────┐
              │ DispatchingStrategy  │
              │    <<interface>>     │
              ├──────────────────────┤
              │ +selectElevator()    │
              └──────────┬───────────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
    ┌──────────┐  ┌──────────┐  ┌──────────┐
    │   FCFS   │  │   SSTF   │  │   SCAN   │
    │ Strategy │  │ Strategy │  │ Strategy │
    └──────────┘  └──────────┘  └──────────┘
```

**Benefits:**
- Add new algorithms without modifying existing code
- Switch algorithms at runtime
- Easy to test each algorithm in isolation

### Observer Design Pattern

The Observer pattern defines a one-to-many dependency between objects.

**Application:**

```
    ┌──────────────────┐         ┌──────────────────┐
    │ HallwayButton    │────────►│ ElevatorObserver │
    │    Panel         │ notifies│   <<interface>>  │
    │  (Subject)       │         └────────┬─────────┘
    └──────────────────┘                  │
                                          │
                               ┌──────────┴──────────┐
                               │                     │
                               ▼                     ▼
                        ┌────────────┐        ┌────────────┐
                        │ Dispatch   │        │  Logger    │
                        │ Controller │        │ (future)   │
                        └────────────┘        └────────────┘
```

### Scheduling Algorithm Comparison

| Algorithm | Fairness | Efficiency | Starvation Risk | Best Use Case |
|-----------|----------|------------|-----------------|---------------|
| FCFS | ⭐⭐⭐⭐⭐ | ⭐⭐ | None | Low traffic |
| SSTF | ⭐⭐ | ⭐⭐⭐⭐⭐ | High | Spread requests |
| SCAN | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | None | High traffic |
| LOOK | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | None | Most buildings |
| C-SCAN | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | None | Uniform load |

### Real-World Elevator Systems

Modern elevator systems use sophisticated algorithms beyond simple scheduling:

1. **Destination Dispatch**: Passengers enter destination before boarding; system groups passengers going to same floors

2. **Traffic Pattern Learning**: AI learns building traffic patterns (morning rush up, evening rush down)

3. **Predictive Positioning**: Idle elevators position themselves based on expected demand

4. **Energy Optimization**: Consider energy cost of acceleration/deceleration

These are all variations and extensions of the core Strategy pattern—the architecture remains the same, only the algorithm changes.
