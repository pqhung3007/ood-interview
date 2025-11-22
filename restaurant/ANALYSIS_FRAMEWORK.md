# 07

## Design a Restaurant Management System

A Restaurant Management System handles the complex operations of running a restaurant: table reservations (both scheduled and walk-ins), menu management, order processing with kitchen coordination, and billing. Unlike simple CRUD systems, restaurant software must handle **real-time table availability**, **order lifecycle management**, and **optimal table assignment**—all while ensuring smooth customer experience during peak hours.

This design presents unique business challenges around **resource optimization** (table assignment), **state machine management** (order lifecycle), and **time-based availability**—making it excellent for testing a candidate's understanding of business domain modeling.

---

## Requirements Gathering

Before diving into the design, let's clarify the requirements through a Q&A format, simulating an interview scenario. Pay attention to the **business rules** and **edge cases**.

### Requirements Clarification

**Q: What are the main operations the restaurant system should support?**
A: The system should handle:
- Table management with varying capacities
- Reservations (both scheduled in advance and walk-ins)
- Menu management with categorized items
- Order processing with kitchen workflow
- Bill calculation

**Q: How should table assignment work for reservations?**
A: Tables should be assigned optimally—**always assign the smallest available table that can accommodate the party size**. This maximizes seating efficiency. For example, a party of 4 should get a 4-seat table, not a 10-seat table, even if both are available.

**Q: What's the time granularity for reservations?**
A: Reservations are slot-based with **hourly granularity**. A table reserved at 7 PM is blocked for the 7 PM slot. This simplifies availability checking but has implications for restaurant throughput.

**Q: How should the system handle walk-ins vs scheduled reservations?**
A: Walk-ins are essentially "reservations for now"—they follow the same table assignment logic but with the current time. The challenge is that walk-ins compete with scheduled reservations for the same time slot.

**Q: What happens if no table can accommodate a party?**
A: If no table with sufficient capacity is available at the requested time, the reservation cannot be created. The system should provide alternative available time slots.

**Q: What is the order lifecycle in a restaurant?**
A: Orders follow a state machine:
```
PENDING → SENT_TO_KITCHEN → DELIVERED
    ↓           ↓
 CANCELED   CANCELED (with restrictions)
```
- Items start as PENDING when ordered
- They move to SENT_TO_KITCHEN when transmitted to the kitchen
- Finally DELIVERED when served to the customer
- Cancellation is allowed in PENDING or SENT_TO_KITCHEN states (with potential charge for the latter)

**Q: Can customers cancel an order after it's been delivered?**
A: No, delivered items cannot be canceled. This is a business rule enforced in the order item state machine.

**Q: How should the bill be calculated?**
A: The bill is the sum of all order items at the table. Canceled items should NOT be included in the bill (assuming no cancellation fee).

**Q: Should the system support menu item categories?**
A: Yes, items should be categorized (APPETIZER, MAIN, DESSERT) for menu organization and potentially for kitchen routing (different stations prepare different categories).

**Q: What about splitting bills or separate checks?**
A: Out of scope for this design, but worth mentioning as a common real-world requirement.

**Q: Should the system support table combining for large parties?**
A: Out of scope—we assume a fixed table layout where each table has a fixed capacity.

### Requirements State

Based on the clarifications above, here are our finalized requirements:

| Requirement | Status | Business Impact |
|-------------|--------|-----------------|
| Table layout with varying capacities | ✅ Confirmed | Core infrastructure |
| Optimal table assignment (smallest fit) | ✅ Confirmed | **Revenue optimization** |
| Scheduled reservations (future time) | ✅ Confirmed | Customer service |
| Walk-in reservations (current time) | ✅ Confirmed | Flexibility |
| Find available time slots | ✅ Confirmed | Customer convenience |
| Reservation cancellation | ✅ Confirmed | Customer service |
| Menu with categorized items | ✅ Confirmed | Kitchen organization |
| Order lifecycle (state machine) | ✅ Confirmed | **Kitchen coordination** |
| Order cancellation with state rules | ✅ Confirmed | Business rules |
| Bill calculation (excluding canceled) | ✅ Confirmed | Revenue accuracy |
| Table combining for large parties | ❌ Out of scope | — |
| Split bills / separate checks | ❌ Out of scope | — |
| Waitstaff assignment | ❌ Out of scope | — |
| Kitchen display system integration | ❌ Out of scope | — |

---

## Identify Core Objects

From the requirements, we can identify the following core objects:

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                       RESTAURANT MANAGEMENT SYSTEM OBJECTS                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                           TABLE DOMAIN                                       │   │
│  │  ┌──────────┐    ┌──────────┐                                               │   │
│  │  │  Layout  │───►│  Table   │    Layout manages the restaurant's tables     │   │
│  │  │          │ 1:N│          │    Tables have capacity and track             │   │
│  │  │- tables  │    │- tableId │    reservations and orders                    │   │
│  │  │- byCapacity   │- capacity│                                               │   │
│  │  └──────────┘    │- reservations                                            │   │
│  │                  │- orders  │                                               │   │
│  │                  └──────────┘                                               │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         RESERVATION DOMAIN                                   │   │
│  │  ┌──────────────────┐    ┌──────────────┐                                   │   │
│  │  │ReservationManager│───►│ Reservation  │                                   │   │
│  │  │                  │ 1:N│              │                                   │   │
│  │  │- layout          │    │- partyName   │   ReservationManager handles      │   │
│  │  │- reservations    │    │- partySize   │   finding slots, creating and     │   │
│  │  │                  │    │- time        │   canceling reservations          │   │
│  │  │+findTimeSlots()  │    │- table       │                                   │   │
│  │  │+createReservation│    └──────────────┘                                   │   │
│  │  └──────────────────┘                                                       │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                            MENU DOMAIN                                       │   │
│  │  ┌──────────┐    ┌──────────┐                                               │   │
│  │  │   Menu   │───►│ MenuItem │    Menu organizes items by name               │   │
│  │  │          │ 1:N│          │    MenuItem has category for kitchen routing  │   │
│  │  │- items   │    │- name    │                                               │   │
│  │  │          │    │- price   │    Categories: APPETIZER, MAIN, DESSERT       │   │
│  │  │+addItem()│    │- category│                                               │   │
│  │  │+getItem()│    └──────────┘                                               │   │
│  │  └──────────┘                                                               │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                           ORDER DOMAIN                                       │   │
│  │  ┌──────────┐    ┌──────────┐                                               │   │
│  │  │OrderItem │    │  Status  │    OrderItem wraps MenuItem with lifecycle    │   │
│  │  │          │    │  (enum)  │                                               │   │
│  │  │- item    │    │          │    Status: PENDING → SENT_TO_KITCHEN →        │   │
│  │  │- status  │───►│ PENDING  │            DELIVERED (or CANCELED)            │   │
│  │  │          │    │ SENT...  │                                               │   │
│  │  │+sendTo() │    │ DELIVERED│    State transitions enforce business rules   │   │
│  │  │+deliver()│    │ CANCELED │                                               │   │
│  │  │+cancel() │    └──────────┘                                               │   │
│  │  └──────────┘                                                               │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                      COMMAND PATTERN (Deep Dive)                             │   │
│  │  ┌──────────────┐                                                           │   │
│  │  │ OrderCommand │◄─────────────┬─────────────┬─────────────┐                │   │
│  │  │ <<interface>>│              │             │             │                │   │
│  │  ├──────────────┤              │             │             │                │   │
│  │  │ + execute()  │         ┌────┴───┐   ┌────┴───┐   ┌────┴───┐              │   │
│  │  └──────────────┘         │ SendTo │   │Deliver │   │ Cancel │              │   │
│  │                           │ Kitchen│   │Command │   │Command │              │   │
│  │  ┌──────────────┐         └────────┘   └────────┘   └────────┘              │   │
│  │  │ OrderManager │    Invoker that queues and executes commands              │   │
│  │  │  (Invoker)   │    Enables undo/redo, logging, and batch processing       │   │
│  │  └──────────────┘                                                           │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

**Core objects identified:**

1. **Restaurant**: Main facade coordinating all operations
2. **Layout**: Manages tables with capacity-based lookup for optimal assignment
3. **Table**: Physical table with capacity, reservations, and current orders
4. **ReservationManager**: Handles availability checking and reservation CRUD
5. **Reservation**: Links party to table at a specific time
6. **Menu**: Collection of menu items by name
7. **MenuItem**: Product with name, description, price, and category
8. **OrderItem**: Wraps MenuItem with lifecycle status
9. **Status**: Enum representing order item state
10. **OrderCommand**: Interface for command pattern (deep dive)
11. **OrderManager**: Invoker for command execution

---

## Design Class Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                                  CLASS DIAGRAM                                        │
└──────────────────────────────────────────────────────────────────────────────────────┘

                              ┌─────────────────────┐
                              │     Restaurant      │
                              │      (Facade)       │
                              ├─────────────────────┤
                              │ - name              │
                              │ - menu              │
                              │ - layout            │
                              │ - reservationMgr    │
                              ├─────────────────────┤
                              │ + findTimeSlots()   │
                              │ + createReservation()│
                              │ + createWalkIn()    │
                              │ + orderItem()       │
                              │ + cancelItem()      │
                              │ + calculateBill()   │
                              └──────────┬──────────┘
                                         │
         ┌───────────────────────────────┼───────────────────────────────┐
         │                               │                               │
         ▼                               ▼                               ▼
   ┌──────────┐                   ┌──────────────────┐            ┌──────────┐
   │   Menu   │                   │ReservationManager│            │  Layout  │
   ├──────────┤                   ├──────────────────┤            ├──────────┤
   │- items   │                   │- layout          │            │- byId    │
   │ (Map)    │                   │- reservations    │            │- byCap   │◄─── SortedMap!
   ├──────────┤                   ├──────────────────┤            ├──────────┤
   │+addItem()│                   │+findTimeSlots()  │            │+findTable│
   │+getItem()│                   │+createReservation│            └────┬─────┘
   └────┬─────┘                   │+removeReservation│                 │
        │                         └────────┬─────────┘                 │
        │ 1:N                              │                           │ 1:N
        ▼                                  │                           ▼
   ┌──────────┐                            │                      ┌──────────┐
   │ MenuItem │                            │                      │  Table   │
   ├──────────┤                            │                      ├──────────┤
   │- name    │                            │ creates              │- tableId │
   │- descript│                            │                      │- capacity│
   │- price   │                            ▼                      │- reservations
   │- category│                     ┌──────────────┐              │- orders  │
   └──────────┘                     │ Reservation  │              ├──────────┤
                                    ├──────────────┤              │+addOrder()│
                                    │- partyName   │◄─────────────│+removeOrder│
                                    │- partySize   │   assigned   │+calcBill()│
                                    │- time        │              │+isAvailable│
                                    │- table ──────┼──────────────┘          │
                                    └──────────────┘                          │
                                                                              │
                                                                              │ 1:N
                                                                              ▼
                                                                        ┌──────────┐
                                                                        │OrderItem │
                                                                        ├──────────┤
                                                                        │- item    │
                                                                        │- status  │
                                                                        ├──────────┤
                                                                        │+sendTo() │
                                                                        │+deliver()│
                                                                        │+cancel() │
                                                                        └──────────┘


┌──────────────────────────────────────────────────────────────────────────────────────┐
│                         TABLE ASSIGNMENT OPTIMIZATION                                 │
└──────────────────────────────────────────────────────────────────────────────────────┘

  THE BUSINESS PROBLEM:
  ═══════════════════

    Restaurant has tables: [4-seat], [4-seat], [6-seat], [6-seat], [10-seat]

    Party of 4 arrives. Which table to assign?

    ❌ NAIVE APPROACH: First available table
       → Might assign 10-seat table
       → Party of 8 arrives later, no table fits!
       → Lost revenue: $XXX

    ✅ OPTIMAL APPROACH: Smallest available table that fits
       → Assign 4-seat table
       → 10-seat remains available for larger party
       → Maximized seating capacity utilization


  IMPLEMENTATION USING SortedMap:
  ═══════════════════════════════

    Layout {
        tablesByCapacity: SortedMap<Integer, Set<Table>>

        // TreeMap maintains keys in sorted order (4, 6, 10)
        // tailMap(partySize) returns entries >= partySize
    }

    findAvailableTable(partySize=4, time):
        tablesByCapacity.tailMap(4)  →  {4: [T1, T2], 6: [T3, T4], 10: [T5]}

        Iterate from smallest to largest:
        - Check T1 (4-seat): Available? YES → Return T1 ✓

    findAvailableTable(partySize=7, time):
        tablesByCapacity.tailMap(7)  →  {10: [T5]}

        Iterate:
        - Check T5 (10-seat): Available? YES → Return T5 ✓


┌──────────────────────────────────────────────────────────────────────────────────────┐
│                           ORDER LIFECYCLE STATE MACHINE                               │
└──────────────────────────────────────────────────────────────────────────────────────┘

                              ┌─────────────┐
                              │   PENDING   │  Customer orders item
                              │  (initial)  │
                              └──────┬──────┘
                                     │
                   ┌─────────────────┴─────────────────┐
                   │ sendToKitchen()                   │ cancel()
                   ▼                                   ▼
           ┌───────────────┐                   ┌─────────────┐
           │SENT_TO_KITCHEN│                   │  CANCELED   │
           │               │                   │   (final)   │
           └───────┬───────┘                   └─────────────┘
                   │                                   ▲
                   │ deliverToCustomer()              │ cancel()
                   │               ┌──────────────────┘
                   │               │ (allowed with restrictions)
                   ▼               │
           ┌───────────────┐       │
           │   DELIVERED   │───────┘ ✗ cancel() NOT allowed
           │    (final)    │         (item already served)
           └───────────────┘


  STATE TRANSITION RULES (Business Logic):
  ════════════════════════════════════════

    ┌──────────────────┬───────────────────┬─────────────────────────────────┐
    │ Current State    │ Allowed Actions   │ Business Rationale              │
    ├──────────────────┼───────────────────┼─────────────────────────────────┤
    │ PENDING          │ sendToKitchen()   │ Normal flow                     │
    │                  │ cancel()          │ No cost, not started            │
    ├──────────────────┼───────────────────┼─────────────────────────────────┤
    │ SENT_TO_KITCHEN  │ deliverToCustomer │ Normal flow                     │
    │                  │ cancel()          │ May incur charge (food wasted)  │
    ├──────────────────┼───────────────────┼─────────────────────────────────┤
    │ DELIVERED        │ (none)            │ Item consumed, cannot undo      │
    ├──────────────────┼───────────────────┼─────────────────────────────────┤
    │ CANCELED         │ (none)            │ Terminal state                  │
    └──────────────────┴───────────────────┴─────────────────────────────────┘
```

---

## Code

The Restaurant Management System is implemented using the **Facade Pattern** for coordination, **State Pattern** (implicit) for order lifecycle, and **Command Pattern** for order operations in the deep dive.

### Deep Dive Topics

#### The Table Assignment Optimization Problem

Efficient table assignment directly impacts revenue. The system uses a `SortedMap<Integer, Set<Table>>` to enable O(log N) lookup of the smallest available table:

```java
public class Layout {
    private final Map<Integer, Table> tablesById = new HashMap<>();
    // Key insight: TreeMap maintains keys in sorted order (smallest to largest)
    private final SortedMap<Integer, Set<Table>> tablesByCapacity = new TreeMap<>();

    public Table findAvailableTable(int partySize, LocalDateTime reservationTime) {
        // tailMap(partySize) returns all entries with capacity >= partySize
        // Since TreeMap is sorted, we iterate from smallest sufficient capacity first
        for (Set<Table> tables : tablesByCapacity.tailMap(partySize).values()) {
            for (Table table : tables) {
                if (table.isAvailableAt(reservationTime)) {
                    return table;  // First available = smallest sufficient
                }
            }
        }
        return null;  // No available table
    }
}
```

**Complexity analysis:**
- `tailMap(partySize)`: O(log N) where N = number of distinct capacities
- Iteration: O(T) where T = number of tables with sufficient capacity
- **Total: O(log N + T)**, much better than O(all tables) naive approach

**Business impact simulation:**

```
Scenario: Friday evening, 5 tables (4, 4, 6, 6, 10 seats)

Naive assignment (first available):
  18:00 - Party of 2 → Table 10 ✗ (wasted 8 seats)
  18:30 - Party of 8 → No table! ✗ (lost booking)
  Revenue lost: ~$200

Optimal assignment (smallest fit):
  18:00 - Party of 2 → Table 4 ✓
  18:30 - Party of 8 → Table 10 ✓
  All parties seated!
```

**Reference:** `restaurant/table/Layout.java:22-31`

---

#### Time Slot Availability: The Hourly Granularity Trade-off

The system uses hourly slots for simplicity, but this has business implications:

```java
public LocalDateTime[] findAvailableTimeSlots(LocalDateTime rangeStart,
                                               LocalDateTime rangeEnd,
                                               int partySize) {
    LocalDateTime current = rangeStart;
    List<LocalDateTime> possibleReservations = new ArrayList<>();

    while (!current.isAfter(rangeEnd)) {
        Table availableTable = layout.findAvailableTable(partySize, current);
        if (availableTable != null) {
            possibleReservations.add(current);
        }
        current = current.plusHours(1);  // Hourly granularity
    }
    return possibleReservations.toArray(new LocalDateTime[0]);
}
```

**Trade-off analysis:**

| Granularity | Pros | Cons |
|-------------|------|------|
| Hourly | Simple, predictable | Under-utilizes tables, limits throughput |
| 30-minute | Better utilization | More complex, may rush customers |
| Variable (based on party size) | Optimal utilization | Complex, unpredictable for customers |

**Real-world consideration:** Most fine dining uses 2-hour windows, casual dining 1-hour. The system could be enhanced with configurable slot duration:

```java
// Enhanced version (not implemented)
public class ReservationManager {
    private final Duration slotDuration;  // Configurable

    public ReservationManager(Layout layout, Duration slotDuration) {
        this.slotDuration = slotDuration;
    }
}
```

**Reference:** `restaurant/reservation/ReservationManager.java:24-36`

---

#### Order Item State Machine: Enforcing Business Rules

The `OrderItem` class implements an implicit state machine with guarded transitions:

```java
public class OrderItem {
    private final MenuItem item;
    private Status status = Status.PENDING;

    public void sendToKitchen() {
        if (status == Status.PENDING) {
            status = Status.SENT_TO_KITCHEN;
        }
        // Silent failure if wrong state - could throw exception instead
    }

    public void deliverToCustomer() {
        if (status == Status.SENT_TO_KITCHEN) {
            status = Status.DELIVERED;
        }
    }

    public void cancel() {
        if (status == Status.PENDING || status == Status.SENT_TO_KITCHEN) {
            status = Status.CANCELED;
        }
        // Cannot cancel DELIVERED items - enforced by condition
    }
}
```

**Design critique:** The current implementation silently ignores invalid transitions. A more robust approach:

```java
// Alternative: Explicit state validation with exceptions
public void cancel() {
    switch (status) {
        case PENDING:
        case SENT_TO_KITCHEN:
            status = Status.CANCELED;
            break;
        case DELIVERED:
            throw new IllegalStateException("Cannot cancel delivered item");
        case CANCELED:
            throw new IllegalStateException("Item already canceled");
    }
}
```

**Reference:** `restaurant/reservation/OrderItem.java:19-34`

---

#### Bill Calculation: Handling Edge Cases

The bill calculation must account for order item states:

```java
// Current implementation - counts ALL items
public BigDecimal calculateBillAmount() {
    return orderedItems.values().stream()
            .flatMap(List::stream)
            .map(OrderItem::getItem)
            .map(MenuItem::getPrice)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
}
```

**Problem:** This includes CANCELED items in the bill! A correct implementation:

```java
// Corrected implementation - excludes canceled items
public BigDecimal calculateBillAmount() {
    return orderedItems.values().stream()
            .flatMap(List::stream)
            .filter(orderItem -> orderItem.getStatus() != Status.CANCELED)
            .map(OrderItem::getItem)
            .map(MenuItem::getPrice)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
}
```

**Business extension:** For items canceled after being sent to kitchen, some restaurants charge a percentage:

```java
public BigDecimal calculateBillAmountWithCancellationFee() {
    BigDecimal total = BigDecimal.ZERO;
    for (List<OrderItem> items : orderedItems.values()) {
        for (OrderItem orderItem : items) {
            switch (orderItem.getStatus()) {
                case DELIVERED:
                case PENDING:
                case SENT_TO_KITCHEN:
                    total = total.add(orderItem.getItem().getPrice());
                    break;
                case CANCELED:
                    // Check if it was canceled after being sent to kitchen
                    // (would need to track this in OrderItem)
                    // If yes, charge 50% cancellation fee
                    break;
            }
        }
    }
    return total;
}
```

**Reference:** `restaurant/table/Table.java:29-36`

---

### Parking at Code

The following describes the code structure:

```
restaurant/
├── restaurant/
│   ├── Restaurant.java              # Main facade coordinating operations
│   ├── RestaurantTest.java          # Test suite
│   ├── table/
│   │   ├── Table.java               # Physical table with orders & reservations
│   │   └── Layout.java              # Table collection with optimal assignment
│   ├── menu/
│   │   ├── Menu.java                # Menu item collection
│   │   └── MenuItem.java            # Product with category
│   └── reservation/
│       ├── Reservation.java         # Party-table-time binding
│       ├── ReservationManager.java  # Availability & reservation CRUD
│       ├── OrderItem.java           # MenuItem with lifecycle status
│       └── Status.java              # Order item state enum
└── restaurant_deep_dive/
    ├── command/
    │   ├── OrderCommand.java        # Command interface
    │   ├── SendToKitchenCommand.java
    │   ├── DeliverCommand.java
    │   └── CancelCommand.java
    ├── OrderManager.java            # Command invoker with queue
    └── RestaurantWithCommands.java  # Enhanced restaurant with command pattern
```

---

### Restaurant (Facade)

The `Restaurant` class serves as the **Facade**, providing a unified interface for all restaurant operations:

```java
public class Restaurant {
    private final String name;
    private final Menu menu;
    private final Layout layout;
    private final ReservationManager reservationManager;

    // Reservation operations delegate to ReservationManager
    public Reservation createScheduledReservation(String partyName, int partySize, LocalDateTime time) {
        return reservationManager.createReservation(partyName, partySize, time);
    }

    // Walk-in is just a reservation for "now"
    public Reservation createWalkInReservation(String partyName, int partySize) {
        return reservationManager.createReservation(partyName, partySize, LocalDateTime.now());
    }

    // Order operations delegate to Table
    public void orderItem(Table table, MenuItem item) {
        table.addOrder(item);
    }
}
```

**Reference:** `restaurant/Restaurant.java:1-78`

---

### Layout (Optimal Table Assignment)

The `Layout` class uses a `SortedMap` to enable efficient smallest-fit table lookup:

```java
public class Layout {
    private final Map<Integer, Table> tablesById = new HashMap<>();
    private final SortedMap<Integer, Set<Table>> tablesByCapacity = new TreeMap<>();

    public Layout(List<Integer> tableCapacities) {
        for (int i = 0; i < tableCapacities.size(); i++) {
            int capacity = tableCapacities.get(i);
            Table table = new Table(i, capacity);
            tablesById.put(i, table);
            tablesByCapacity.computeIfAbsent(capacity, k -> new HashSet<>()).add(table);
        }
    }
}
```

**Why two data structures:**
- `tablesById`: O(1) lookup by ID for specific table operations
- `tablesByCapacity`: O(log N) for optimal assignment, maintains capacity order

**Reference:** `restaurant/table/Layout.java:1-32`

---

### Table (Order and Reservation Management)

The `Table` class manages both reservations (time-based) and orders (current session):

```java
public class Table {
    private final int tableId;
    private final int capacity;
    private final Map<LocalDateTime, Reservation> reservations = new HashMap<>();
    private final Map<MenuItem, List<OrderItem>> orderedItems = new HashMap<>();

    // Time-based availability check
    public boolean isAvailableAt(LocalDateTime reservationTime) {
        return !reservations.containsKey(reservationTime);
    }

    // Order management with quantity support
    public void addOrder(MenuItem item, int quantity) {
        for (int i = 0; i < quantity; i++) {
            addOrder(item);
        }
    }
}
```

**Design note:** Orders are keyed by `MenuItem` for easy lookup, with a `List<OrderItem>` to handle multiple orders of the same item.

**Reference:** `restaurant/table/Table.java:1-107`

---

### Command Pattern for Order Operations (Deep Dive)

The deep dive implements the **Command Pattern** for order operations, enabling:
- Queuing of operations
- Potential undo/redo (not implemented but structure supports it)
- Logging and auditing
- Batch processing

```java
// Command interface
public interface OrderCommand {
    void execute();
}

// Concrete commands
public class SendToKitchenCommand implements OrderCommand {
    private final OrderItem orderItem;

    @Override
    public void execute() {
        orderItem.sendToKitchen();
    }
}

// Invoker
public class OrderManager {
    private final List<OrderCommand> commandQueue = new ArrayList<>();

    public void addCommand(OrderCommand command) {
        commandQueue.add(command);
    }

    public void executeCommands() {
        for (OrderCommand command : commandQueue) {
            command.execute();
        }
        commandQueue.clear();
    }
}
```

**Usage in RestaurantWithCommands:**

```java
public void orderItem(Table table, MenuItem item) {
    table.addOrder(item);
    OrderItem lastOrder = /* get last order */;

    // Use command pattern instead of direct method call
    OrderCommand sendToKitchen = new SendToKitchenCommand(lastOrder);
    orderManager.addCommand(sendToKitchen);
    orderManager.executeCommands();
}
```

**Extension for undo:** Each command could implement an `undo()` method:

```java
public interface OrderCommand {
    void execute();
    void undo();  // For rollback capability
}

public class SendToKitchenCommand implements OrderCommand {
    private Status previousStatus;  // Store for undo

    @Override
    public void execute() {
        previousStatus = orderItem.getStatus();
        orderItem.sendToKitchen();
    }

    @Override
    public void undo() {
        // Restore previous status (simplified - real impl would be more complex)
    }
}
```

**Reference:** `restaurant_deep_dive/command/*.java`, `restaurant_deep_dive/OrderManager.java`

---

## Wrap Up

In this design, we created a Restaurant Management System that addresses both functional requirements and business optimization challenges. Key takeaways:

1. **Optimal Table Assignment** using `SortedMap.tailMap()` to always assign the smallest available table, maximizing seating efficiency and revenue

2. **Time Slot Management** with hourly granularity, trading simplicity for potential under-utilization (configurable in production)

3. **Order Lifecycle State Machine** enforcing business rules (can't cancel delivered items) through guarded state transitions

4. **Facade Pattern** in `Restaurant` class, providing a clean API while coordinating menu, layout, and reservation subsystems

5. **Command Pattern** (deep dive) for order operations, enabling queuing, logging, and potential undo/redo functionality

6. **Bill Calculation** considerations for handling canceled items and potential cancellation fees

---

## Further Reading: Command Pattern and State Machine Patterns

### Command Design Pattern

The Command pattern encapsulates a request as an object, allowing parameterization, queuing, logging, and undo operations.

**Structure:**

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│    Client    │      │   Invoker    │      │   Receiver   │
│              │      │              │      │              │
│ Restaurant   │─────►│ OrderManager │─────►│  OrderItem   │
│              │      │              │      │              │
└──────────────┘      └──────────────┘      └──────────────┘
                             │
                             │ executes
                             ▼
                      ┌──────────────┐
                      │   Command    │
                      │ <<interface>>│
                      ├──────────────┤
                      │ + execute()  │
                      └──────┬───────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
           ▼                 ▼                 ▼
    ┌────────────┐    ┌────────────┐    ┌────────────┐
    │SendToKitchen    │  Deliver   │    │   Cancel   │
    │  Command   │    │  Command   │    │  Command   │
    └────────────┘    └────────────┘    └────────────┘
```

**Benefits:**
- **Decoupling**: Invoker doesn't know about receiver details
- **Queuing**: Commands can be stored and executed later
- **Logging**: Command history enables audit trails
- **Undo/Redo**: Commands can implement reverse operations
- **Macro commands**: Composite commands for batch operations

### State Machine Pattern (Implicit in OrderItem)

The OrderItem class implements an implicit state machine. An explicit implementation would look like:

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                          EXPLICIT STATE PATTERN                                       │
└──────────────────────────────────────────────────────────────────────────────────────┘

              ┌──────────────┐
              │  OrderState  │
              │ <<interface>>│
              ├──────────────┤
              │+sendToKitchen│
              │+deliver()    │
              │+cancel()     │
              └──────┬───────┘
                     │
     ┌───────────────┼───────────────┬───────────────┐
     │               │               │               │
     ▼               ▼               ▼               ▼
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ Pending │    │  Sent   │    │Delivered│    │Canceled │
│  State  │    │  State  │    │  State  │    │  State  │
├─────────┤    ├─────────┤    ├─────────┤    ├─────────┤
│sendTo→  │    │deliver→ │    │ (all    │    │ (all    │
│  Sent   │    │Delivered│    │ no-op)  │    │ no-op)  │
│cancel→  │    │cancel→  │    │         │    │         │
│Canceled │    │Canceled │    │         │    │         │
└─────────┘    └─────────┘    └─────────┘    └─────────┘


  Benefits of Explicit State Pattern:
  ═══════════════════════════════════

  1. Each state is a class - Single Responsibility
  2. Adding new states doesn't modify existing code - Open/Closed
  3. State-specific behavior is encapsulated
  4. Transitions are explicit and testable
```

### Facade Design Pattern (Restaurant Class)

The Facade provides a simplified interface to the complex restaurant subsystem:

```
                         ┌─────────────────┐
          Client ───────►│   Restaurant    │
                         │    (Facade)     │
                         └────────┬────────┘
                                  │
       ┌──────────────────────────┼──────────────────────────┐
       │              │           │           │              │
       ▼              ▼           ▼           ▼              ▼
   ┌───────┐    ┌───────────┐ ┌───────┐  ┌────────┐   ┌───────┐
   │ Menu  │    │Reservation│ │Layout │  │ Table  │   │ Order │
   │       │    │  Manager  │ │       │  │        │   │ Item  │
   └───────┘    └───────────┘ └───────┘  └────────┘   └───────┘
```

### Pattern Comparison for Restaurant Domain

| Pattern | Purpose | Application in Restaurant |
|---------|---------|---------------------------|
| **Facade** | Simplify subsystem access | `Restaurant` coordinates Menu, Layout, Reservations |
| **Command** | Encapsulate operations | Order lifecycle operations (send, deliver, cancel) |
| **State** | Behavior based on state | OrderItem status transitions |
| **Strategy** | Interchangeable algorithms | Could be used for pricing strategies (happy hour, etc.) |
| **Observer** | Notify on changes | Could notify kitchen display when order sent |
