# 06

## Design a Movie Ticket Booking System

A Movie Ticket Booking System is a software application that allows users to browse movies, view available screenings, select seats, and purchase tickets. These systems must handle concurrent bookings, prevent double-booking of seats, manage dynamic pricing based on seat tiers, and provide real-time availability updates.

This design presents unique challenges around **concurrency**, **race conditions**, and **distributed state management**—making it an excellent topic for testing a candidate's understanding of system design beyond basic OOP.

---

## Requirements Gathering

Before diving into the design, let's clarify the requirements through a Q&A format, simulating an interview scenario. Pay attention to the **edge cases** and **scalability concerns** raised.

### Requirements Clarification

**Q: What are the core entities in the system?**
A: The system should manage Cinemas (with multiple Rooms), Movies, Screenings (a movie shown at a specific time in a specific room), Seats, Tickets, and Orders.

**Q: How should seating be organized?**
A: Each Room has a Layout defining rows and columns of Seats. Seats should be accessible both by position (row, column) and by a unique seat number. The layout is fixed per room but different rooms can have different layouts.

**Q: How should pricing work?**
A: Pricing should be flexible and seat-based. Different seats can have different prices (Normal, Premium, VIP tiers). The price should be determined by a pricing strategy attached to each seat.

**Q: Can the same seat have different prices for different screenings?**
A: In this design, pricing is seat-based (tied to the physical seat). For time-based pricing (matinee vs evening), we would need a more complex pricing model combining seat tier and time slot—consider this out of scope for now.

**Q: What happens when two users try to book the same seat simultaneously?**
A: This is a critical concurrency challenge. The system must prevent double-booking. We need to consider:
- **Optimistic locking**: Check-then-book with conflict detection
- **Pessimistic locking**: Lock the seat during selection
- **Temporary seat holds**: Lock seats during checkout with expiration

**Q: How long should a seat be held during checkout?**
A: Typically 10-15 minutes. If the user doesn't complete payment, the lock expires and the seat becomes available again. This requires a **TTL (Time-To-Live)** mechanism.

**Q: Should the system support bulk booking (multiple seats in one order)?**
A: Yes, users should be able to select multiple seats and book them in a single transaction. This introduces **atomicity requirements**—either all seats are booked or none.

**Q: What happens if a screening is cancelled?**
A: All tickets for that screening should be refunded. This is out of scope for this design but worth mentioning for a complete system.

**Q: How do we handle the "seat map" display—should it show real-time availability?**
A: Ideally yes, but this has performance implications. We'll discuss eventual consistency vs strong consistency trade-offs in the deep dive.

**Q: Should the system support multiple cinemas in different locations?**
A: Yes, the system should be multi-cinema. This affects how we structure lookups (e.g., find screenings by location).

### Requirements State

Based on the clarifications above, here are our finalized requirements:

| Requirement | Status | Complexity |
|-------------|--------|------------|
| Multi-cinema, multi-room support | ✅ Confirmed | Medium |
| Flexible room layouts (rows × columns) | ✅ Confirmed | Medium |
| Seat-based tiered pricing (Normal/Premium/VIP) | ✅ Confirmed | Low |
| Movie and screening management | ✅ Confirmed | Low |
| Seat availability tracking per screening | ✅ Confirmed | High |
| Prevent double-booking (concurrency) | ✅ Confirmed | **Critical** |
| Temporary seat holds with TTL | ✅ Confirmed | High |
| Bulk seat booking (multiple seats per order) | ✅ Confirmed | Medium |
| Order management with total calculation | ✅ Confirmed | Low |
| Time-based pricing (matinee/evening) | ❌ Out of scope | — |
| Refunds and cancellations | ❌ Out of scope | — |
| Payment processing | ❌ Out of scope | — |
| User accounts and authentication | ❌ Out of scope | — |

---

## Identify Core Objects

From the requirements, we can identify the following core objects:

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        MOVIE TICKET BOOKING SYSTEM OBJECTS                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                           LOCATION DOMAIN                                    │   │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐              │   │
│  │  │  Cinema  │───►│   Room   │───►│  Layout  │───►│   Seat   │              │   │
│  │  │          │ 1:N│          │ 1:1│          │ 1:N│          │              │   │
│  │  │- name    │    │- roomNum │    │- rows    │    │- seatNum │              │   │
│  │  │- location│    │- layout  │    │- columns │    │- pricing │              │   │
│  │  └──────────┘    └──────────┘    │- seatMap │    │          │              │   │
│  │                                  └──────────┘    └──────────┘              │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                           SHOWING DOMAIN                                     │   │
│  │  ┌──────────┐    ┌──────────┐                                               │   │
│  │  │  Movie   │◄───│Screening │    Screening = Movie + Room + TimeSlot        │   │
│  │  │          │ N:1│          │                                               │   │
│  │  │- title   │    │- movie   │    A movie can have many screenings           │   │
│  │  │- genre   │    │- room    │    A screening is in exactly one room         │   │
│  │  │- duration│    │- start   │                                               │   │
│  │  └──────────┘    │- end     │                                               │   │
│  │                  └──────────┘                                               │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                           BOOKING DOMAIN                                     │   │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐                               │   │
│  │  │  Ticket  │◄───│  Order   │    │Screening │                               │   │
│  │  │          │ N:1│          │    │ Manager  │  Coordinates bookings         │   │
│  │  │- screening│   │- tickets │    ├──────────┤                               │   │
│  │  │- seat    │    │- date    │    │- movies  │  Tracks seat availability     │   │
│  │  │- price   │    │- total   │    │- tickets │  Prevents double-booking      │   │
│  │  └──────────┘    └──────────┘    └──────────┘                               │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                           PRICING DOMAIN                                     │   │
│  │  ┌────────────────┐                                                         │   │
│  │  │PricingStrategy │◄─────────────┬─────────────┬─────────────┐              │   │
│  │  │  <<interface>> │              │             │             │              │   │
│  │  ├────────────────┤              │             │             │              │   │
│  │  │ + getPrice()   │         ┌────┴───┐   ┌────┴───┐   ┌────┴───┐           │   │
│  │  └────────────────┘         │ Normal │   │Premium │   │  VIP   │           │   │
│  │                             │  Rate  │   │  Rate  │   │  Rate  │           │   │
│  │                             └────────┘   └────────┘   └────────┘           │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                      CONCURRENCY DOMAIN (Deep Dive)                          │   │
│  │  ┌──────────────┐    ┌──────────┐                                           │   │
│  │  │ SeatLock     │    │ SeatLock │    Temporary holds during checkout        │   │
│  │  │   Manager    │───►│          │                                           │   │
│  │  │              │    │- userId  │    TTL-based expiration                   │   │
│  │  │- lockedSeats │    │- expiry  │    Prevents race conditions               │   │
│  │  └──────────────┘    └──────────┘                                           │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

**Core objects identified:**

1. **Cinema**: Physical location containing multiple rooms
2. **Room**: A theater room with a specific seating layout
3. **Layout**: Defines the seat arrangement (rows × columns) with dual-access (by position and by number)
4. **Seat**: Individual seat with associated pricing strategy
5. **Movie**: Film with title, genre, and duration
6. **Screening**: A specific showing of a movie in a room at a time
7. **Ticket**: Proof of booking linking screening, seat, and price
8. **Order**: Collection of tickets for a single transaction
9. **ScreeningManager**: Coordinates movies, screenings, and ticket bookings
10. **PricingStrategy**: Interface for seat pricing (Normal, Premium, VIP)
11. **SeatLockManager**: Handles temporary seat holds (concurrency control)

---

## Design Class Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                                  CLASS DIAGRAM                                        │
└──────────────────────────────────────────────────────────────────────────────────────┘

                            ┌─────────────────────┐
                            │ MovieBookingSystem  │
                            │      (Facade)       │
                            ├─────────────────────┤
                            │ - movies            │
                            │ - cinemas           │
                            │ - screeningManager  │
                            ├─────────────────────┤
                            │ + addMovie()        │
                            │ + addCinema()       │
                            │ + addScreening()    │
                            │ + getAvailableSeats()│
                            │ + bookTicket()      │
                            └──────────┬──────────┘
                                       │
          ┌────────────────────────────┼────────────────────────────┐
          │                            │                            │
          ▼                            ▼                            ▼
    ┌──────────┐              ┌────────────────┐             ┌──────────┐
    │  Cinema  │              │ScreeningManager│             │  Movie   │
    ├──────────┤              ├────────────────┤             ├──────────┤
    │- name    │              │-screeningsByMovie            │- title   │
    │- location│              │-ticketsByScreening           │- genre   │
    │- rooms   │              ├────────────────┤             │- duration│
    ├──────────┤              │+addScreening() │             └──────────┘
    │+addRoom()│              │+addTicket()    │
    │+getRooms()              │+getAvailable() │
    └────┬─────┘              │+bookOptimistic()│◄─── Concurrency!
         │                    └────────────────┘
         │ 1:N
         ▼
    ┌──────────┐         ┌──────────┐         ┌──────────┐
    │   Room   │────────►│  Layout  │────────►│   Seat   │
    ├──────────┤   1:1   ├──────────┤   1:N   ├──────────┤
    │- roomNum │         │- rows    │         │- seatNum │
    │- layout  │         │- columns │         │- pricing │◄─── Strategy
    └──────────┘         │- seatsByNumber     ├──────────┤     Pattern
                         │- seatsByPosition   │+getPrice()│
                         ├──────────┤         └────┬─────┘
                         │+getSeat()|              │
                         │+getAllSeats()           │
                         └──────────┘              │
                                                   ▼
                                          ┌────────────────┐
                                          │PricingStrategy │
                                          │ <<interface>>  │
                                          ├────────────────┤
                                          │ + getPrice()   │
                                          └───────┬────────┘
                                                  │
                                    ┌─────────────┼─────────────┐
                                    │             │             │
                                    ▼             ▼             ▼
                              ┌──────────┐ ┌──────────┐ ┌──────────┐
                              │NormalRate│ │PremiumRate│ │ VIPRate │
                              │ ($10)    │ │  ($15)   │ │  ($25)  │
                              └──────────┘ └──────────┘ └──────────┘


┌──────────────────────────────────────────────────────────────────────────────────────┐
│                            BOOKING FLOW & RELATIONSHIPS                               │
└──────────────────────────────────────────────────────────────────────────────────────┘

    ┌──────────┐                    ┌──────────┐
    │  Movie   │◄───────────────────│Screening │
    │          │       N:1          │          │
    │          │                    │- movie   │
    └──────────┘                    │- room    │────────────► Room
                                    │- startTime
                                    │- endTime │
                                    └────┬─────┘
                                         │
                                         │ 1:N (tickets per screening)
                                         ▼
                                    ┌──────────┐         ┌──────────┐
                                    │  Ticket  │◄────────│  Order   │
                                    ├──────────┤   N:1   ├──────────┤
                                    │-screening│         │- tickets │
                                    │- seat    │         │- orderDate│
                                    │- price   │         ├──────────┤
                                    └──────────┘         │+addTicket()│
                                                         │+calcTotal()│
                                                         └──────────┘


┌──────────────────────────────────────────────────────────────────────────────────────┐
│                         CONCURRENCY CONTROL (Deep Dive)                               │
└──────────────────────────────────────────────────────────────────────────────────────┘

  THE RACE CONDITION PROBLEM:
  ═══════════════════════════

    User A                           User B
      │                                │
      │  1. Check seat available       │
      │  ───────────────────────►      │
      │        ✓ Available             │  2. Check seat available
      │                                │  ───────────────────────►
      │                                │        ✓ Available
      │  3. Book seat                  │
      │  ───────────────────────►      │
      │        ✓ Booked                │  4. Book seat
      │                                │  ───────────────────────►
      │                                │        ✓ ALSO BOOKED! 💥
      │                                │
      └────── DOUBLE BOOKING! ─────────┘


  SOLUTION 1: PESSIMISTIC LOCKING (SeatLockManager)
  ════════════════════════════════════════════════

    ┌──────────────────┐
    │  SeatLockManager │
    ├──────────────────┤         ┌──────────────┐
    │ - lockedSeats    │────────►│   SeatLock   │
    │   (ConcurrentMap)│         ├──────────────┤
    │ - lockDuration   │         │ - userId     │
    ├──────────────────┤         │ - expiration │
    │ + lockSeat()     │         ├──────────────┤
    │ + isLocked()     │         │ + isExpired()│
    │ + unlock()       │         └──────────────┘
    └──────────────────┘

    Flow:
    1. User selects seat → lockSeat(screening, seat, userId)
    2. Lock created with TTL (e.g., 10 minutes)
    3. Other users see seat as "held" (yellow on UI)
    4. User completes payment → actual booking + unlock
    5. OR: TTL expires → automatic unlock


  SOLUTION 2: OPTIMISTIC LOCKING (ScreeningManager)
  ═════════════════════════════════════════════════

    public synchronized Ticket bookSeatOptimistically(...) {
        if (isSeatBooked(screening, seat)) {
            throw new IllegalStateException("Seat already booked");
        }
        // Proceed with booking
        return ticket;
    }

    Trade-off: Less overhead, but user only discovers conflict
               at final booking step (bad UX for long checkout flows)
```

---

## Code

The Movie Ticket Booking System is implemented using the **Strategy Pattern** for pricing, **Facade Pattern** for system coordination, and advanced concurrency patterns for handling simultaneous bookings.

### Deep Dive Topics

#### The Double-Booking Problem: A Concurrency Deep Dive

The most critical challenge in a ticket booking system is preventing two users from booking the same seat. This is a classic **race condition** problem.

**Scenario without protection:**
```
Time    User A                      User B                    Seat State
────────────────────────────────────────────────────────────────────────
T1      getAvailableSeats()         -                         Available
T2      -                           getAvailableSeats()       Available
T3      sees seat A1 available      -                         Available
T4      -                           sees seat A1 available    Available
T5      bookTicket(A1)              -                         Available→Booked
T6      ✓ Success                   -                         Booked
T7      -                           bookTicket(A1)            Booked
T8      -                           ✓ Success (BUG!)          DOUBLE BOOKED!
```

**The root cause:** The "check-then-act" sequence is not atomic. Between checking availability (T2/T4) and booking (T5/T7), the state can change.

**Reference:** `movie_ticket_deep_dive/SeatLockManager.java`

---

#### Optimistic vs Pessimistic Locking: Trade-offs

| Aspect | Optimistic Locking | Pessimistic Locking |
|--------|-------------------|---------------------|
| **When to use** | Low contention (few users per screening) | High contention (popular movies, limited seats) |
| **Lock duration** | Microseconds (during transaction) | Minutes (during checkout flow) |
| **User experience** | Conflict discovered at booking time | Seat shown as "held" immediately |
| **Complexity** | Lower | Higher (need TTL, cleanup) |
| **Starvation risk** | Yes (repeated conflicts) | Yes (locks held too long) |
| **Implementation** | `synchronized` block | `SeatLockManager` with TTL |

**Optimistic locking implementation:**
```java
// ScreeningManager (deep dive version)
public synchronized Ticket bookSeatOptimistically(Screening screening, Seat seat) {
    // Check-then-act is now atomic due to synchronized
    if (isSeatBooked(screening, seat)) {
        throw new IllegalStateException("Seat is already booked");
    }

    BigDecimal price = seat.getPricingStrategy().getPrice();
    Ticket ticket = new Ticket(screening, seat, price);
    ticketsByScreening.computeIfAbsent(screening, k -> new ArrayList<>()).add(ticket);
    return ticket;
}
```

**Pessimistic locking implementation:**
```java
// SeatLockManager
public synchronized boolean lockSeat(Screening screening, Seat seat, String userId) {
    String lockKey = generateLockKey(screening, seat);
    cleanupLockIfExpired(lockKey);

    if (isLocked(screening, seat)) {
        return false;  // Someone else has the lock
    }

    SeatLock lock = new SeatLock(userId, LocalDateTime.now().plus(lockDuration));
    lockedSeats.put(lockKey, lock);
    return true;
}
```

**Reference:** `movie_ticket_deep_dive/ScreeningManager.java:58-81`, `movie_ticket_deep_dive/SeatLockManager.java:14-28`

---

#### Why Layout Uses Two Maps

The `Layout` class maintains two data structures for seat lookup:

```java
public class Layout {
    private final Map<String, Seat> seatsByNumber;           // "A1" → Seat
    private final Map<Integer, Map<Integer, Seat>> seatsByPosition;  // [row][col] → Seat
}
```

**Rationale:**

| Use Case | Preferred Lookup | Time Complexity |
|----------|------------------|-----------------|
| Display seat map on UI | `seatsByPosition` (iterate by row/col) | O(rows × cols) |
| User clicks specific seat | `seatsByNumber` (direct lookup) | O(1) |
| Booking API (seat ID in request) | `seatsByNumber` | O(1) |
| Seat availability check | Either (both O(1) for single seat) | O(1) |

This is a classic **space-time trade-off**: we use extra memory (two maps) to achieve O(1) lookup for both access patterns.

**Reference:** `movie_ticket/location/Layout.java:13-16`

---

#### The Seat Availability Calculation Problem

The current implementation recalculates available seats by iterating through all booked tickets:

```java
public List<Seat> getAvailableSeats(Screening screening) {
    List<Seat> allSeats = screening.getRoom().getLayout().getAllSeats();
    List<Ticket> bookedTickets = getTicketsForScreening(screening);

    List<Seat> availableSeats = new ArrayList<>(allSeats);
    for (Ticket ticket : bookedTickets) {
        availableSeats.remove(ticket.getSeat());  // O(n) per removal!
    }
    return availableSeats;
}
```

**Performance analysis:**
- `getAllSeats()`: O(S) where S = total seats
- Loop through tickets: O(T) where T = booked tickets
- `remove()` on ArrayList: O(S) per call
- **Total: O(T × S)** — problematic for large rooms with many bookings

**Improved approach using Set:**
```java
public List<Seat> getAvailableSeatsOptimized(Screening screening) {
    Set<Seat> bookedSeats = getTicketsForScreening(screening).stream()
            .map(Ticket::getSeat)
            .collect(Collectors.toSet());

    return screening.getRoom().getLayout().getAllSeats().stream()
            .filter(seat -> !bookedSeats.contains(seat))
            .collect(Collectors.toList());
}
// Time: O(S + T) — much better!
```

**Reference:** `movie_ticket/ticket/ScreeningManager.java:47-56`

---

### Parking at Code

The following describes the code structure:

```
movie_ticket/
├── movie_ticket/
│   ├── MovieBookingSystem.java      # Main facade coordinating all operations
│   ├── MovieBookingSystemTest.java  # Test suite
│   ├── location/                    # Physical location domain
│   │   ├── Cinema.java              # Cinema with multiple rooms
│   │   ├── Room.java                # Room with layout
│   │   ├── Layout.java              # Seat arrangement (dual-map design)
│   │   └── Seat.java                # Individual seat with pricing
│   ├── showing/                     # Movie and screening domain
│   │   ├── Movie.java               # Film entity
│   │   └── Screening.java           # Movie + Room + TimeSlot
│   ├── ticket/                      # Booking domain
│   │   ├── Ticket.java              # Proof of booking
│   │   ├── Order.java               # Collection of tickets
│   │   └── ScreeningManager.java    # Coordinates bookings
│   └── rate/                        # Pricing strategy domain
│       ├── PricingStrategy.java     # Interface
│       ├── NormalRate.java          # Standard pricing
│       ├── PremiumRate.java         # Middle tier
│       └── VIPRate.java             # Premium tier
└── movie_ticket_deep_dive/
    ├── ScreeningManager.java        # With optimistic locking
    └── SeatLockManager.java         # Pessimistic locking implementation
```

---

### MovieBookingSystem

The `MovieBookingSystem` class serves as the **Facade**, providing a simplified interface for all booking operations.

```java
public class MovieBookingSystem {
    private final List<Movie> movies;
    private final List<Cinema> cinemas;
    private final ScreeningManager screeningManager;

    public void bookTicket(Screening screening, Seat seat) {
        BigDecimal price = seat.getPricingStrategy().getPrice();
        Ticket ticket = new Ticket(screening, seat, price);
        screeningManager.addTicket(screening, ticket);
    }
}
```

**Design critique:** The current `bookTicket` method doesn't validate seat availability before booking. In a production system, this should either:
1. Throw an exception if seat is already booked
2. Return a result object indicating success/failure
3. Use the optimistic locking version from the deep dive

**Reference:** `movie_ticket/MovieBookingSystem.java:1-77`

---

### Layout (Dual-Map Design)

The `Layout` class demonstrates a thoughtful data structure design for different access patterns:

```java
public class Layout {
    private final int rows;
    private final int columns;
    private final Map<String, Seat> seatsByNumber;            // O(1) by ID
    private final Map<Integer, Map<Integer, Seat>> seatsByPosition;  // O(1) by position

    public Seat getSeatByNumber(String seatNumber) {
        return seatsByNumber.get(seatNumber);
    }

    public Seat getSeatByPosition(int row, int column) {
        Map<Integer, Seat> rowSeats = seatsByPosition.get(row);
        return (rowSeats != null) ? rowSeats.get(column) : null;
    }
}
```

**Interview follow-up questions:**
- *"Why not just use one map?"* — Different access patterns require different key structures
- *"What's the memory overhead?"* — Each seat is stored once, but referenced twice. Overhead is O(S) pointers, negligible vs seat objects
- *"How would you handle a non-rectangular layout (e.g., curved IMAX)?"* — The position-based map handles gaps naturally; non-existent positions simply return null

**Reference:** `movie_ticket/location/Layout.java:1-58`

---

### Screening and ScreeningManager

**Screening** represents a specific showing, while **ScreeningManager** coordinates the many-to-many relationships:

```java
// Screening - immutable value object
public class Screening {
    private final Movie movie;
    private final Room room;
    private final LocalDateTime startTime;
    private final LocalDateTime endTime;
}

// ScreeningManager - handles relationships
public class ScreeningManager {
    private final Map<Movie, List<Screening>> screeningsByMovie;
    private final Map<Screening, List<Ticket>> ticketsByScreening;
}
```

**Key relationship insight:** A single `Screening` object is shared across both maps. This is intentional—`Screening` acts as the "join point" between the movie schedule and ticket sales.

**Reference:** `movie_ticket/showing/Screening.java:1-41`, `movie_ticket/ticket/ScreeningManager.java:1-57`

---

### PricingStrategy (Strategy Pattern)

The pricing system uses the **Strategy Pattern** for flexibility:

```java
public interface PricingStrategy {
    BigDecimal getPrice();
}

public class NormalRate implements PricingStrategy {
    private final BigDecimal price;

    @Override
    public BigDecimal getPrice() { return price; }
}

// Usage: seat.setPricingStrategy(new VIPRate(new BigDecimal("25.00")));
```

**Why Strategy over Inheritance:**

| Approach | Pros | Cons |
|----------|------|------|
| Inheritance (`VIPSeat extends Seat`) | Simple | Seat tier is static; can't change at runtime |
| Strategy (`Seat` has `PricingStrategy`) | Flexible; can change pricing | Slightly more complex |

**Real-world extension:** The strategy could be enhanced to consider additional factors:

```java
// Enhanced strategy for time-based pricing (future extension)
public interface EnhancedPricingStrategy {
    BigDecimal getPrice(Screening screening);  // Consider showtime
}

public class PeakHourRate implements EnhancedPricingStrategy {
    private final BigDecimal basePrice;
    private final BigDecimal peakMultiplier;

    @Override
    public BigDecimal getPrice(Screening screening) {
        if (isPeakHour(screening.getStartTime())) {
            return basePrice.multiply(peakMultiplier);
        }
        return basePrice;
    }
}
```

**Reference:** `movie_ticket/rate/PricingStrategy.java:1-7`, `movie_ticket/rate/VIPRate.java:1-16`

---

### SeatLockManager (Pessimistic Concurrency Control)

The **SeatLockManager** implements pessimistic locking with TTL-based expiration:

```java
public class SeatLockManager {
    private final Map<String, SeatLock> lockedSeats = new ConcurrentHashMap<>();
    private final Duration lockDuration;

    public synchronized boolean lockSeat(Screening screening, Seat seat, String userId) {
        String lockKey = generateLockKey(screening, seat);
        cleanupLockIfExpired(lockKey);  // Lazy cleanup

        if (isLocked(screening, seat)) {
            return false;
        }

        SeatLock lock = new SeatLock(userId, LocalDateTime.now().plus(lockDuration));
        lockedSeats.put(lockKey, lock);
        return true;
    }

    private static class SeatLock {
        private final String userId;
        private final LocalDateTime expirationTime;

        public boolean isExpired() {
            return LocalDateTime.now().isAfter(expirationTime);
        }
    }
}
```

**Design decisions:**

| Decision | Rationale |
|----------|-----------|
| `ConcurrentHashMap` | Thread-safe without full synchronization |
| `synchronized` on methods | Ensures atomic check-then-lock |
| Lazy cleanup (`cleanupLockIfExpired`) | Avoids background thread overhead |
| Lock key = `screeningId-seatNumber` | Unique identifier for each seat-in-screening |

**Trade-off:** Lazy cleanup means expired locks may briefly appear as "held" until the next access. For real-time accuracy, a background cleanup thread could be added.

**Reference:** `movie_ticket_deep_dive/SeatLockManager.java:1-69`

---

### Adding Dynamic Pricing Based on Demand

To implement surge pricing based on seat availability:

```java
public class DemandBasedPricingStrategy implements PricingStrategy {
    private final BigDecimal basePrice;
    private final ScreeningManager screeningManager;
    private final Screening screening;

    @Override
    public BigDecimal getPrice() {
        int availableSeats = screeningManager.getAvailableSeats(screening).size();
        int totalSeats = screening.getRoom().getLayout().getAllSeats().size();
        double occupancyRate = 1.0 - ((double) availableSeats / totalSeats);

        // Price increases as occupancy increases
        if (occupancyRate > 0.9) {
            return basePrice.multiply(new BigDecimal("1.5"));  // 50% surge
        } else if (occupancyRate > 0.7) {
            return basePrice.multiply(new BigDecimal("1.25")); // 25% surge
        }
        return basePrice;
    }
}
```

**Complexity note:** This requires the pricing strategy to have access to booking state, which increases coupling. Consider using an event-driven approach where prices are recalculated and cached when bookings change.

---

## Wrap Up

In this design, we created a Movie Ticket Booking System that addresses both basic OOP design and advanced concurrency challenges. Key takeaways:

1. **Strategy Pattern** for flexible seat pricing, allowing different tiers without modifying seat logic

2. **Facade Pattern** in `MovieBookingSystem`, providing a clean API while hiding the complexity of coordinating cinemas, movies, screenings, and bookings

3. **Dual-Map Design** in `Layout`, optimizing for different access patterns (by seat number vs by position)

4. **Concurrency Control** with two approaches:
   - **Optimistic locking**: Simple, suitable for low contention
   - **Pessimistic locking**: More complex, better UX for high contention with TTL-based seat holds

5. **Race Condition Prevention** through atomic check-then-act operations using `synchronized` blocks

6. **Performance Considerations** in seat availability calculation (ArrayList removal vs Set-based filtering)

---

## Further Reading: Concurrency Patterns and Strategy Pattern

### Strategy Design Pattern

The Strategy pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable.

**Application in this design:**

```
              ┌────────────────┐
              │PricingStrategy │
              │ <<interface>>  │
              ├────────────────┤
              │ + getPrice()   │
              └───────┬────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Normal   │  │ Premium  │  │   VIP    │
  │  Rate    │  │   Rate   │  │   Rate   │
  │  $10     │  │   $15    │  │   $25    │
  └──────────┘  └──────────┘  └──────────┘
```

**Benefits:**
- Pricing logic is isolated from seat logic
- New pricing tiers can be added without modifying existing code
- Pricing can be changed at runtime (e.g., promotions)

### Facade Design Pattern

The Facade pattern provides a unified interface to a set of interfaces in a subsystem.

**Application in `MovieBookingSystem`:**

```
                         ┌──────────────────┐
          Client ───────►│MovieBookingSystem│
                         │     (Facade)     │
                         └────────┬─────────┘
                                  │
       ┌──────────────────────────┼──────────────────────────┐
       │              │           │           │              │
       ▼              ▼           ▼           ▼              ▼
   ┌───────┐    ┌───────┐   ┌─────────┐  ┌────────┐   ┌──────────┐
   │Cinema │    │ Movie │   │Screening│  │ Ticket │   │ Screening│
   │       │    │       │   │         │  │        │   │ Manager  │
   └───────┘    └───────┘   └─────────┘  └────────┘   └──────────┘
```

### Concurrency Patterns

#### Optimistic Locking

Assumes conflicts are rare. Check for conflicts at commit time.

```
┌─────────────────────────────────────────────────────────────┐
│                    OPTIMISTIC LOCKING                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Read current state                                      │
│  2. Perform operation (assuming no conflict)                │
│  3. At commit: verify state hasn't changed                  │
│  4. If changed: retry or fail                               │
│                                                             │
│  Pros: Low overhead, high throughput for low contention     │
│  Cons: Retry storms under high contention                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### Pessimistic Locking

Assumes conflicts are likely. Lock resources before operating.

```
┌─────────────────────────────────────────────────────────────┐
│                    PESSIMISTIC LOCKING                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Acquire lock on resource                                │
│  2. Perform operation while holding lock                    │
│  3. Release lock                                            │
│                                                             │
│  Pros: Predictable behavior, no retry logic needed          │
│  Cons: Lock contention, deadlock risk, reduced throughput   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### TTL-Based Locks (Used in SeatLockManager)

Combines pessimistic locking with automatic expiration to prevent indefinite holds.

```
┌─────────────────────────────────────────────────────────────┐
│                     TTL-BASED LOCKS                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  User A acquires lock at T=0 (TTL=10 min)                   │
│         │                                                   │
│         ▼                                                   │
│  ┌──────────────────┐                                       │
│  │ Lock: Seat A1    │                                       │
│  │ User: A          │                                       │
│  │ Expires: T+10min │                                       │
│  └──────────────────┘                                       │
│         │                                                   │
│         ▼                                                   │
│  T+5: User A completes payment → Lock released manually     │
│         OR                                                  │
│  T+10: Lock expires → Automatic release, seat available     │
│                                                             │
│  Pros: Prevents abandoned locks, good UX                    │
│  Cons: User loses seat if checkout takes too long           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Pattern Comparison

| Aspect | Strategy | Facade | Optimistic Lock | Pessimistic Lock |
|--------|----------|--------|-----------------|------------------|
| **Purpose** | Encapsulate algorithms | Simplify subsystem access | Handle concurrent access | Prevent concurrent access |
| **When to use** | Multiple interchangeable behaviors | Complex subsystem coordination | Low conflict probability | High conflict probability |
| **In this design** | Seat pricing tiers | `MovieBookingSystem` API | Quick booking scenarios | Seat selection during checkout |
| **Trade-off** | Flexibility vs complexity | Simplicity vs power | Throughput vs consistency | Safety vs performance |
