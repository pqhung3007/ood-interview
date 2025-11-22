# 9

## Design a Shipping Locker System

A Shipping Locker System enables package delivery and pickup through secure automated lockers. This is a practical problem that tests understanding of **resource allocation strategies**, **policy-based pricing**, **storage lifecycle management**, and **notification patterns**. Think of it as building Amazon Locker or similar last-mile delivery systems.

This design focuses on **locker allocation algorithms** (best-fit vs first-fit), **tiered account policies** (free periods, maximum storage), **charge calculation with BigDecimal precision**, and **Observer pattern for notifications**.

---

## Requirements Gathering

### Requirements Clarification

**Q: What types of lockers are available?**
A: Three standard sizes with different dimensions and pricing:
- `SMALL`: 10x10x10 inches, $5.00/day
- `MEDIUM`: 20x20x20 inches, $10.00/day
- `LARGE`: 30x30x30 inches, $15.00/day

**Q: How is package-to-locker matching done?**
A: The system finds the smallest locker that can fit the package dimensions. If a package is 15x15x15, it requires a MEDIUM locker (SMALL is too small at 10x10x10).

**Q: What happens if no suitable locker is available?**
A: The system throws `NoLockerAvailableException`. The delivery driver must try a different site or return the package.

**Q: How does the pricing model work?**
A: Pricing depends on account policy:
- **Free period**: First N days are free (e.g., 3 days for standard accounts)
- **Daily rate**: After free period, charged per day based on locker size
- **Maximum period**: Package expires after M days (e.g., 7 days), status becomes EXPIRED

**Q: Can different accounts have different policies?**
A: Yes, `AccountLockerPolicy` defines per-account rules:
- Standard: 3 free days, 7 max days
- Premium: 5 free days, 14 max days
- Business: 7 free days, 30 max days

**Q: How do customers access their packages?**
A: Via a 6-digit access code generated when package is placed. Customer enters code → system validates → locker opens → charges calculated and applied.

**Q: What happens if a package expires?**
A: Package status changes to `EXPIRED`, customer is still charged for the storage period, and the package may be returned to sender or held for additional fees.

**Q: How are customers notified?**
A: Through `NotificationInterface` which can be implemented as email, SMS, or push notification. Notifications are sent on:
- Package arrival (with access code)
- Reminder before expiration
- Package expiration

**Q: What package statuses exist?**
A: Five distinct states:
- `CREATED`: Package registered in system
- `IN_TRANSIT`: Package en route to locker
- `IN_LOCKER`: Package placed in locker
- `RETRIEVED`: Customer picked up package
- `EXPIRED`: Maximum storage period exceeded

**Q: Should we support multiple sites?**
A: Yes, each `Site` has its own pool of lockers. `LockerManager` operates per-site.

**Q: How do we prevent race conditions in locker assignment?**
A: For interview scope, we assume single-threaded operation. In production, you'd use optimistic locking or database transactions.

**Q: Should oversized packages be split across lockers?**
A: No, one package per locker. Oversized packages that don't fit any locker throw `PackageIncompatibleException`.

### Requirements State

| Requirement | Status | Pattern/Approach |
|-------------|--------|------------------|
| Three locker sizes (S/M/L) | ✅ Confirmed | Enum with dimensions |
| Package-to-locker size matching | ✅ Confirmed | First-fit algorithm |
| NoLockerAvailableException | ✅ Confirmed | Exception handling |
| Free period per account | ✅ Confirmed | Policy pattern |
| Daily charge after free period | ✅ Confirmed | BigDecimal calculation |
| Maximum storage period | ✅ Confirmed | Policy pattern |
| Account-specific policies | ✅ Confirmed | Strategy/Policy |
| 6-digit access code generation | ✅ Confirmed | Random generation |
| Access code validation | ✅ Confirmed | Map lookup |
| Notification on package arrival | ✅ Confirmed | Observer pattern |
| Package status lifecycle | ✅ Confirmed | State management |
| Multiple sites | ✅ Confirmed | Site per manager |
| Oversized package rejection | ✅ Confirmed | PackageIncompatibleException |
| Concurrent locker assignment | ❌ Out of scope | — |
| Return-to-sender workflow | ❌ Out of scope | — |
| Locker maintenance/repair | ❌ Out of scope | — |

---

## Identify Core Objects

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                          SHIPPING LOCKER SYSTEM OBJECTS                                      │
├─────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│  │                           MANAGEMENT DOMAIN                                          │   │
│  │                                                                                      │   │
│  │  ┌────────────────────┐         ┌────────────────────┐                              │   │
│  │  │   LockerManager    │────────►│       Site         │                              │   │
│  │  │                    │         │                    │                              │   │
│  │  │ - site             │         │ - lockers Map      │                              │   │
│  │  │ - accounts Map     │         │                    │                              │   │
│  │  │ - accessCodeMap    │         │ +findAvailableLocker()                            │   │
│  │  │ - notification     │         │ +placePackage()    │                              │   │
│  │  │                    │         │                    │                              │   │
│  │  │ +assignPackage()   │         └────────────────────┘                              │   │
│  │  │ +pickUpPackage()   │                                                             │   │
│  │  │ +getAccount()      │         Orchestrates package placement                      │   │
│  │  └────────────────────┘         and customer pickup operations                      │   │
│  │           │                                                                          │   │
│  │           │ uses                                                                     │   │
│  │           ▼                                                                          │   │
│  │  ┌────────────────────┐                                                             │   │
│  │  │NotificationInterface                                                             │   │
│  │  │   <<interface>>    │                                                             │   │
│  │  │                    │                                                             │   │
│  │  │+sendNotification() │                                                             │   │
│  │  └────────────────────┘                                                             │   │
│  │           ▲                                                                          │   │
│  │           │ implements                                                               │   │
│  │  ┌────────┴────────┬────────────────┐                                               │   │
│  │  │                 │                │                                               │   │
│  │  ▼                 ▼                ▼                                               │   │
│  │ Email            SMS             Push                                               │   │
│  │ Notification     Notification    Notification                                       │   │
│  │                                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│  │                            LOCKER DOMAIN                                             │   │
│  │                                                                                      │   │
│  │  ┌────────────────────┐         ┌────────────────────┐                              │   │
│  │  │      Locker        │────────►│    LockerSize      │                              │   │
│  │  │                    │         │      (enum)        │                              │   │
│  │  │ - size             │         │                    │                              │   │
│  │  │ - currentPackage   │         │ SMALL  ($5/day)    │                              │   │
│  │  │ - assignmentDate   │         │ MEDIUM ($10/day)   │                              │   │
│  │  │ - accessCode       │         │ LARGE  ($15/day)   │                              │   │
│  │  │                    │         │                    │                              │   │
│  │  │ +assignPackage()   │         │ - sizeName         │                              │   │
│  │  │ +releaseLocker()   │         │ - dailyCharge      │                              │   │
│  │  │ +calculateCharges()│         │ - width/height/depth                              │   │
│  │  │ +isAvailable()     │         │                    │                              │   │
│  │  │ +checkAccessCode() │         └────────────────────┘                              │   │
│  │  └────────────────────┘                                                             │   │
│  │                                                                                      │   │
│  │  Physical locker with state management and charge calculation                       │   │
│  │                                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│  │                           ACCOUNT DOMAIN                                             │   │
│  │                                                                                      │   │
│  │  ┌────────────────────┐         ┌────────────────────┐                              │   │
│  │  │      Account       │────────►│AccountLockerPolicy │                              │   │
│  │  │                    │         │                    │                              │   │
│  │  │ - accountId        │         │ - freePeriodDays   │                              │   │
│  │  │ - ownerName        │         │ - maximumPeriodDays│                              │   │
│  │  │ - lockerPolicy     │         │                    │                              │   │
│  │  │ - usageCharges     │         │ +getFreePeriodDays()                              │   │
│  │  │                    │         │ +getMaximumPeriodDays()                           │   │
│  │  │ +addUsageCharge()  │         │                    │                              │   │
│  │  │ +getUsageCharges() │         └────────────────────┘                              │   │
│  │  │ +getLockerPolicy() │                                                             │   │
│  │  └────────────────────┘         Policy defines free days and max storage            │   │
│  │                                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│  │                           PACKAGE DOMAIN                                             │   │
│  │                                                                                      │   │
│  │  ┌────────────────────┐         ┌────────────────────┐                              │   │
│  │  │  ShippingPackage   │         │   ShippingStatus   │                              │   │
│  │  │   <<interface>>    │────────►│      (enum)        │                              │   │
│  │  │                    │         │                    │                              │   │
│  │  │ +getOrderId()      │         │ CREATED            │                              │   │
│  │  │ +getUser()         │         │ IN_TRANSIT         │                              │   │
│  │  │ +getDimensions()   │         │ IN_LOCKER          │                              │   │
│  │  │ +getLockerSize()   │         │ RETRIEVED          │                              │   │
│  │  │ +updateStatus()    │         │ EXPIRED            │                              │   │
│  │  └────────┬───────────┘         └────────────────────┘                              │   │
│  │           │                                                                          │   │
│  │           │ implements                                                               │   │
│  │           ▼                                                                          │   │
│  │  ┌────────────────────┐                                                             │   │
│  │  │BasicShippingPackage│         Determines required locker size                     │   │
│  │  │                    │         from package dimensions                             │   │
│  │  │ - orderId          │                                                             │   │
│  │  │ - user             │                                                             │   │
│  │  │ - width/height/depth                                                             │   │
│  │  │ - status           │                                                             │   │
│  │  │                    │                                                             │   │
│  │  │ +getLockerSize()   │◄──── First-fit: finds smallest locker                       │   │
│  │  └────────────────────┘      that fits all dimensions                               │   │
│  │                                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│  │                         EXCEPTION DOMAIN                                             │   │
│  │                                                                                      │   │
│  │  ┌────────────────────────┐  ┌─────────────────────────────────┐                    │   │
│  │  │NoLockerAvailableException│ │MaximumStoragePeriodExceededException                │   │
│  │  │                        │  │                                 │                    │   │
│  │  │ No locker of required  │  │ Package exceeded max days       │                    │   │
│  │  │ size is available      │  │ (e.g., 7 days for standard)     │                    │   │
│  │  └────────────────────────┘  └─────────────────────────────────┘                    │   │
│  │                                                                                      │   │
│  │  ┌────────────────────────┐                                                         │   │
│  │  │PackageIncompatibleException                                                      │   │
│  │  │                        │                                                         │   │
│  │  │ Package too large for  │                                                         │   │
│  │  │ any available locker   │                                                         │   │
│  │  └────────────────────────┘                                                         │   │
│  │                                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Design Class Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                            PACKAGE ASSIGNMENT FLOW                                            │
└──────────────────────────────────────────────────────────────────────────────────────────────┘

  ┌─────────┐     ┌─────────────┐     ┌──────┐     ┌────────┐     ┌────────────┐
  │ Carrier │     │LockerManager│     │ Site │     │ Locker │     │Notification│
  └────┬────┘     └──────┬──────┘     └──┬───┘     └───┬────┘     └─────┬──────┘
       │                 │               │             │                │
       │ assignPackage() │               │             │                │
       │────────────────►│               │             │                │
       │                 │               │             │                │
       │                 │placePackage() │             │                │
       │                 │──────────────►│             │                │
       │                 │               │             │                │
       │                 │               │findAvailableLocker()         │
       │                 │               │─────────────►                │
       │                 │               │             │                │
       │                 │               │   locker    │                │
       │                 │               │◄────────────│                │
       │                 │               │             │                │
       │                 │               │assignPackage()               │
       │                 │               │─────────────►                │
       │                 │               │             │                │
       │                 │               │  generates  │                │
       │                 │               │  access code│                │
       │                 │               │             │                │
       │                 │    locker     │             │                │
       │                 │◄──────────────│             │                │
       │                 │               │             │                │
       │                 │ store accessCode in map     │                │
       │                 │               │             │                │
       │                 │ sendNotification("Package arrived, code: XXX")
       │                 │─────────────────────────────────────────────►│
       │                 │               │             │                │
       │    locker       │               │             │                │
       │◄────────────────│               │             │                │
       │                 │               │             │                │


┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                            PACKAGE PICKUP FLOW                                                │
└──────────────────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────┐     ┌─────────────┐     ┌────────┐     ┌─────────┐
  │ Customer │     │LockerManager│     │ Locker │     │ Account │
  └────┬─────┘     └──────┬──────┘     └───┬────┘     └────┬────┘
       │                  │                │               │
       │pickUpPackage(code)               │               │
       │─────────────────►│                │               │
       │                  │                │               │
       │                  │ lookup code in │               │
       │                  │ accessCodeMap  │               │
       │                  │                │               │
       │                  │checkAccessCode()               │
       │                  │───────────────►│               │
       │                  │                │               │
       │                  │calculateStorageCharges()       │
       │                  │───────────────►│               │
       │                  │                │               │
       │                  │  ┌─────────────┴──────────────┐│
       │                  │  │ 1. Get user's policy       ││
       │                  │  │ 2. Calculate days stored   ││
       │                  │  │ 3. Check max period        ││
       │                  │  │ 4. Subtract free days      ││
       │                  │  │ 5. Multiply by daily rate  ││
       │                  │  └─────────────┬──────────────┘│
       │                  │                │               │
       │                  │    charge      │               │
       │                  │◄───────────────│               │
       │                  │                │               │
       │                  │ releaseLocker()│               │
       │                  │───────────────►│               │
       │                  │                │               │
       │                  │ addUsageCharge(charge)         │
       │                  │───────────────────────────────►│
       │                  │                │               │
       │                  │ updateStatus(RETRIEVED)        │
       │                  │                │               │
       │    locker        │                │               │
       │◄─────────────────│                │               │


┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                          LOCKER SIZE SELECTION ALGORITHM                                      │
└──────────────────────────────────────────────────────────────────────────────────────────────┘

  Package: 15x12x8 inches

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                       First-Fit Algorithm                                │
  │                                                                         │
  │   for each LockerSize in [SMALL, MEDIUM, LARGE]:                        │
  │       if locker.width >= pkg.width AND                                  │
  │          locker.height >= pkg.height AND                                │
  │          locker.depth >= pkg.depth:                                     │
  │           return this size                                              │
  │                                                                         │
  │   throw PackageIncompatibleException                                    │
  └─────────────────────────────────────────────────────────────────────────┘

  Evaluation:
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                                                                         │
  │   SMALL (10x10x10):                                                     │
  │     10 >= 15? ❌ NO → Skip                                              │
  │                                                                         │
  │   MEDIUM (20x20x20):                                                    │
  │     20 >= 15? ✅ YES                                                    │
  │     20 >= 12? ✅ YES                                                    │
  │     20 >= 8?  ✅ YES → SELECTED                                         │
  │                                                                         │
  │   LARGE (30x30x30): Not evaluated (already found fit)                   │
  │                                                                         │
  └─────────────────────────────────────────────────────────────────────────┘


┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                          CHARGE CALCULATION FORMULA                                           │
└──────────────────────────────────────────────────────────────────────────────────────────────┘

  Given:
    - assignmentDate: 2024-01-01
    - pickupDate: 2024-01-06 (5 days stored)
    - lockerSize: MEDIUM ($10.00/day)
    - accountPolicy: freePeriodDays=3, maximumPeriodDays=7

  Calculation:
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                                                                         │
  │   totalDaysUsed = (pickupDate - assignmentDate) / (24 * 60 * 60 * 1000) │
  │                 = 5 days                                                │
  │                                                                         │
  │   if (totalDaysUsed > maximumPeriodDays):                               │
  │       throw MaximumStoragePeriodExceededException                       │
  │                                                                         │
  │   chargeableDays = max(0, totalDaysUsed - freePeriodDays)               │
  │                  = max(0, 5 - 3)                                        │
  │                  = 2 days                                               │
  │                                                                         │
  │   charge = chargeableDays × dailyCharge                                 │
  │          = 2 × $10.00                                                   │
  │          = $20.00                                                       │
  │                                                                         │
  └─────────────────────────────────────────────────────────────────────────┘


┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                          PACKAGE STATUS LIFECYCLE                                             │
└──────────────────────────────────────────────────────────────────────────────────────────────┘

                    ┌─────────────────────────────────────────────────────┐
                    │                                                     │
                    ▼                                                     │
              ┌───────────┐                                               │
              │  CREATED  │──── Package registered in system              │
              └─────┬─────┘                                               │
                    │                                                     │
                    │ carrier picks up                                    │
                    ▼                                                     │
              ┌───────────┐                                               │
              │IN_TRANSIT │──── En route to locker site                   │
              └─────┬─────┘                                               │
                    │                                                     │
                    │ placed in locker                                    │
                    ▼                                                     │
              ┌───────────┐                                               │
              │ IN_LOCKER │──── Stored, awaiting pickup                   │
              └─────┬─────┘                                               │
                    │                                                     │
         ┌──────────┴──────────┐                                          │
         │                     │                                          │
         │ customer picks up   │ max period exceeded                      │
         ▼                     ▼                                          │
   ┌───────────┐         ┌───────────┐                                    │
   │ RETRIEVED │         │  EXPIRED  │──── May trigger return-to-sender   │
   └───────────┘         └───────────┘     (out of scope)                 │
                               │                                          │
                               │ if return-to-sender enabled              │
                               └──────────────────────────────────────────┘
```

---

## Code

### Deep Dive Topics

#### Deep Dive 1: Locker Size Matching - First-Fit vs Best-Fit

The current implementation uses **first-fit**: iterate through sizes in order (SMALL → MEDIUM → LARGE) and return the first one that fits. This is simple but has trade-offs:

```java
// First-Fit Implementation (Current)
@Override
public LockerSize getLockerSize() {
    for (LockerSize size : LockerSize.values()) {
        if (size.getWidth().compareTo(width) >= 0 &&
            size.getHeight().compareTo(height) >= 0 &&
            size.getDepth().compareTo(depth) >= 0) {
            return size;
        }
    }
    throw new PackageIncompatibleException("No locker size available for the package");
}
```

**Why first-fit works here:**
1. Enum values are ordered SMALL → MEDIUM → LARGE
2. Iterating in order guarantees we find the smallest fit
3. Simple and efficient O(n) where n = number of sizes

**Alternative: Best-Fit with Volume Calculation**

```java
// Best-Fit Alternative (minimizes wasted space)
public LockerSize getLockerSizeBestFit() {
    BigDecimal packageVolume = width.multiply(height).multiply(depth);

    LockerSize bestFit = null;
    BigDecimal minWaste = null;

    for (LockerSize size : LockerSize.values()) {
        if (fitsInLocker(size)) {
            BigDecimal lockerVolume = size.getWidth()
                .multiply(size.getHeight())
                .multiply(size.getDepth());
            BigDecimal waste = lockerVolume.subtract(packageVolume);

            if (minWaste == null || waste.compareTo(minWaste) < 0) {
                minWaste = waste;
                bestFit = size;
            }
        }
    }

    if (bestFit == null) {
        throw new PackageIncompatibleException("Package too large");
    }
    return bestFit;
}
```

**When to use best-fit:**
- When locker sizes have irregular dimensions
- When optimizing for locker utilization is critical
- When sizes aren't naturally ordered smallest-to-largest

**Reference:** `shippinglocker/pkg/BasicShippingPackage.java:76-85`

---

#### Deep Dive 2: Charge Calculation with BigDecimal Precision

The system uses `BigDecimal` for all monetary calculations. This is critical for financial accuracy:

```java
// Locker.java - Charge calculation
public BigDecimal calculateStorageCharges() {
    if (currentPackage == null || assignmentDate == null) {
        return BigDecimal.ZERO;
    }

    AccountLockerPolicy policy = currentPackage.getUser().getLockerPolicy();
    long totalDaysUsed = (new Date().getTime() - assignmentDate.getTime())
        / (1000 * 60 * 60 * 24);

    // Check maximum period
    if (totalDaysUsed > policy.getMaximumPeriodDays()) {
        currentPackage.updateShippingStatus(ShippingStatus.EXPIRED);
        throw new MaximumStoragePeriodExceededException(
            "Package has exceeded maximum allowed storage period");
    }

    // Calculate chargeable days (excluding free period)
    long chargeableDays = Math.max(0, totalDaysUsed - policy.getFreePeriodDays());
    return size.dailyCharge.multiply(new BigDecimal(chargeableDays));
}
```

**Why BigDecimal instead of double?**

```java
// WRONG: Floating point errors
double charge = 10.00 * 3;  // Might be 29.999999999999996

// CORRECT: Exact decimal arithmetic
BigDecimal charge = new BigDecimal("10.00").multiply(new BigDecimal("3"));
// Always exactly 30.00
```

**Key BigDecimal operations used:**

| Operation | Method | Example |
|-----------|--------|---------|
| Addition | `add()` | `usageCharges.add(amount)` |
| Multiplication | `multiply()` | `dailyCharge.multiply(days)` |
| Comparison | `compareTo()` | `width.compareTo(lockerWidth) >= 0` |
| Zero check | `BigDecimal.ZERO` | `return BigDecimal.ZERO` |

**Edge case: Day calculation boundary**

```java
// Issue: Integer division truncates
// Package stored from 1:00 AM to 11:00 PM same day = 0 days (22 hours)
long totalDaysUsed = (now - assignment) / (1000 * 60 * 60 * 24);

// Consider: Should partial days count?
// Option 1: Ceiling (any partial day = 1 day)
long totalDaysUsed = (long) Math.ceil(
    (double)(now - assignment) / (1000 * 60 * 60 * 24)
);

// Option 2: Grace period (first 24 hours = day 0)
// Current implementation uses floor (truncation)
```

**Reference:** `shippinglocker/locker/Locker.java:43-61`

---

#### Deep Dive 3: Access Code Security and Lookup

The system generates 6-digit codes and maintains a lookup map:

```java
// Code generation
private String generateAccessCode() {
    Random random = new Random();
    int accessCode = 100000 + random.nextInt(900000);  // 100000-999999
    return String.valueOf(accessCode);
}

// Code storage (LockerManager)
private final Map<String, Locker> accessCodeMap = new HashMap<>();

// On assignment
public Locker assignPackage(ShippingPackage pkg, Date date) {
    Locker locker = site.placePackage(pkg, date);
    if (locker != null) {
        accessCodeMap.put(locker.getAccessCode(), locker);  // Store mapping
        notificationService.sendNotification(
            "Package assigned to locker " + locker.getAccessCode(),
            pkg.getUser()
        );
    }
    return locker;
}
```

**Security considerations:**

| Issue | Current State | Production Fix |
|-------|---------------|----------------|
| Predictable Random | `java.util.Random` | Use `SecureRandom` |
| Code collision | Not handled | Check uniqueness before assigning |
| Brute force | No protection | Rate limiting, lockout |
| Code reuse | Released on pickup | Add expiration timestamp |

**Improved secure implementation:**

```java
private String generateSecureAccessCode() {
    SecureRandom secureRandom = new SecureRandom();
    int maxAttempts = 10;

    for (int i = 0; i < maxAttempts; i++) {
        int code = 100000 + secureRandom.nextInt(900000);
        String codeStr = String.valueOf(code);

        // Ensure uniqueness
        if (!accessCodeMap.containsKey(codeStr)) {
            return codeStr;
        }
    }
    throw new RuntimeException("Could not generate unique access code");
}
```

**Reference:** `shippinglocker/locker/Locker.java:74-78`, `shippinglocker/locker/LockerManager.java:33-39`

---

#### Deep Dive 4: Policy Pattern for Account Tiers

The `AccountLockerPolicy` class encapsulates account-specific rules:

```java
public class AccountLockerPolicy {
    final int freePeriodDays;      // Days before charging starts
    final int maximumPeriodDays;   // Days before package expires

    public AccountLockerPolicy(int freePeriodDays, int maximumPeriodDays) {
        this.freePeriodDays = freePeriodDays;
        this.maximumPeriodDays = maximumPeriodDays;
    }
}
```

**Why this is Policy Pattern (variant of Strategy):**

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│              Account                                                   │
│                 │                                                      │
│                 │ uses                                                 │
│                 ▼                                                      │
│        AccountLockerPolicy                                             │
│                 │                                                      │
│     ┌───────────┼───────────┐                                          │
│     │           │           │                                          │
│     ▼           ▼           ▼                                          │
│  Standard    Premium    Business                                       │
│  (3d/7d)     (5d/14d)   (7d/30d)                                       │
│                                                                        │
│  The Locker doesn't know about account tiers—                          │
│  it just uses whatever policy is attached to the user.                 │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

**Usage in charge calculation:**

```java
// The Locker doesn't know about account tiers
// It just uses the policy attached to the user
AccountLockerPolicy policy = currentPackage.getUser().getLockerPolicy();
long chargeableDays = Math.max(0, totalDaysUsed - policy.getFreePeriodDays());
```

**Benefits:**
1. **Decoupling**: Locker doesn't need to know account types
2. **Extensibility**: New tiers just need new policy instances
3. **Testability**: Easy to inject different policies for testing

**Extended policy example:**

```java
// More sophisticated policy with additional rules
public class ExtendedLockerPolicy extends AccountLockerPolicy {
    private final BigDecimal discountPercentage;  // Volume discount
    private final int simultaneousPackageLimit;    // Max packages at once
    private final boolean priorityNotifications;   // SMS vs email

    @Override
    public BigDecimal applyDiscount(BigDecimal baseCharge) {
        return baseCharge.multiply(
            BigDecimal.ONE.subtract(discountPercentage)
        );
    }
}
```

**Reference:** `shippinglocker/account/AccountLockerPolicy.java:4-25`

---

#### Deep Dive 5: Site Resource Management

The `Site` class manages locker inventory using a Map structure:

```java
public class Site {
    // Map of locker sizes to sets of lockers of that size
    final Map<LockerSize, Set<Locker>> lockers = new HashMap<>();

    public Site(Map<LockerSize, Integer> lockerCounts) {
        for (Map.Entry<LockerSize, Integer> entry : lockerCounts.entrySet()) {
            Set<Locker> lockerSet = new HashSet<>();
            for (int i = 0; i < entry.getValue(); i++) {
                lockerSet.add(new Locker(entry.getKey()));
            }
            this.lockers.put(entry.getKey(), lockerSet);
        }
    }
}
```

**Data structure choice analysis:**

| Structure | Used | Alternative | Trade-off |
|-----------|------|-------------|-----------|
| `Map<LockerSize, Set<Locker>>` | ✅ | `List<Locker>` | O(1) size lookup vs O(n) filtering |
| `HashSet<Locker>` | ✅ | `LinkedList<Locker>` | O(1) add/remove vs O(n) |
| `HashMap` | ✅ | `EnumMap` | `EnumMap` is more efficient for enum keys |

**Optimized version with EnumMap:**

```java
public class OptimizedSite {
    // EnumMap is more efficient for enum keys
    private final EnumMap<LockerSize, Deque<Locker>> availableLockers;
    private final EnumMap<LockerSize, Set<Locker>> occupiedLockers;

    public OptimizedSite(Map<LockerSize, Integer> lockerCounts) {
        availableLockers = new EnumMap<>(LockerSize.class);
        occupiedLockers = new EnumMap<>(LockerSize.class);

        for (LockerSize size : LockerSize.values()) {
            availableLockers.put(size, new ArrayDeque<>());
            occupiedLockers.put(size, new HashSet<>());
        }

        for (Map.Entry<LockerSize, Integer> entry : lockerCounts.entrySet()) {
            for (int i = 0; i < entry.getValue(); i++) {
                availableLockers.get(entry.getKey()).add(new Locker(entry.getKey()));
            }
        }
    }

    // O(1) locker retrieval
    public Locker findAvailableLocker(LockerSize size) {
        return availableLockers.get(size).pollFirst();
    }

    // O(1) locker release
    public void releaseLocker(Locker locker) {
        occupiedLockers.get(locker.getSize()).remove(locker);
        availableLockers.get(locker.getSize()).addLast(locker);
    }
}
```

**Locker upgrade strategy:**

When exact size isn't available, should we offer a larger locker?

```java
public Locker findAvailableLockerWithUpgrade(LockerSize requestedSize) {
    // Try exact match first
    Locker locker = findAvailableLocker(requestedSize);
    if (locker != null) {
        return locker;
    }

    // Try larger sizes (upgrade)
    for (LockerSize size : LockerSize.values()) {
        if (size.ordinal() > requestedSize.ordinal()) {
            locker = findAvailableLocker(size);
            if (locker != null) {
                // Charge at requested size, not upgraded size (customer benefit)
                return locker;
            }
        }
    }

    throw new NoLockerAvailableException("No suitable locker available");
}
```

**Reference:** `shippinglocker/Site.java:12-49`

---

#### Deep Dive 6: Observer Pattern for Notifications

The `NotificationInterface` decouples the locker system from specific notification channels:

```java
public interface NotificationInterface {
    void sendNotification(String message, Account user);
}

// LockerManager uses it
public class LockerManager {
    private final NotificationInterface notificationService;

    public Locker assignPackage(ShippingPackage pkg, Date date) {
        Locker locker = site.placePackage(pkg, date);
        if (locker != null) {
            notificationService.sendNotification(
                "Package assigned to locker " + locker.getAccessCode(),
                pkg.getUser()
            );
        }
        return locker;
    }
}
```

**Multiple notification channels:**

```java
// Email implementation
public class EmailNotification implements NotificationInterface {
    private final EmailService emailService;

    @Override
    public void sendNotification(String message, Account user) {
        emailService.send(user.getEmail(), "Locker Update", message);
    }
}

// SMS implementation
public class SMSNotification implements NotificationInterface {
    private final SMSGateway smsGateway;

    @Override
    public void sendNotification(String message, Account user) {
        smsGateway.sendSMS(user.getPhone(), message);
    }
}

// Composite: send to multiple channels
public class MultiChannelNotification implements NotificationInterface {
    private final List<NotificationInterface> channels;

    @Override
    public void sendNotification(String message, Account user) {
        for (NotificationInterface channel : channels) {
            try {
                channel.sendNotification(message, user);
            } catch (Exception e) {
                // Log failure, continue with other channels
            }
        }
    }
}
```

**Full Observer Pattern implementation:**

```java
// Observer interface
public interface LockerEventObserver {
    void onPackageAssigned(Locker locker, ShippingPackage pkg);
    void onPackageRetrieved(Locker locker, ShippingPackage pkg);
    void onPackageExpired(Locker locker, ShippingPackage pkg);
}

// Subject (Observable)
public class ObservableLockerManager extends LockerManager {
    private final List<LockerEventObserver> observers = new ArrayList<>();

    public void addObserver(LockerEventObserver observer) {
        observers.add(observer);
    }

    @Override
    public Locker assignPackage(ShippingPackage pkg, Date date) {
        Locker locker = super.assignPackage(pkg, date);

        // Notify all observers
        for (LockerEventObserver observer : observers) {
            observer.onPackageAssigned(locker, pkg);
        }

        return locker;
    }
}

// Concrete observers
public class NotificationObserver implements LockerEventObserver {
    @Override
    public void onPackageAssigned(Locker locker, ShippingPackage pkg) {
        // Send notification
    }
}

public class AnalyticsObserver implements LockerEventObserver {
    @Override
    public void onPackageAssigned(Locker locker, ShippingPackage pkg) {
        // Track metrics
    }
}
```

**Reference:** `shippinglocker/NotificationInterface.java`, `shipping_locker_deep_dive/LockerEventObserver.java`

---

#### Deep Dive 7: Pickup Flow - Validation and Charge Application

The pickup process involves multiple steps with specific ordering:

```java
public Locker pickUpPackage(String accessCode) {
    // Step 1: Find locker by access code
    Locker locker = accessCodeMap.get(accessCode);

    if (locker != null && locker.checkAccessCode(accessCode)) {
        try {
            // Step 2: Calculate charges (may throw if expired)
            BigDecimal charge = locker.calculateStorageCharges();

            // Step 3: Get package reference before release
            ShippingPackage pkg = locker.getPackage();

            // Step 4: Release the locker
            locker.releaseLocker();

            // Step 5: Apply charges to account
            pkg.getUser().addUsageCharge(charge);

            // Step 6: Update package status
            pkg.updateShippingStatus(ShippingStatus.RETRIEVED);

            return locker;
        } catch (MaximumStoragePeriodExceededException e) {
            // Package expired - still release locker
            locker.releaseLocker();
            return locker;
        }
    }
    return null;
}
```

**Why this ordering matters:**

```
┌────────────────────────────────────────────────────────────────────────┐
│                    CORRECT ORDER                                        │
│                                                                        │
│  1. Calculate charges    ← Must happen BEFORE release (needs package)  │
│  2. Get package ref      ← Must happen BEFORE release (cleared after)  │
│  3. Release locker       ← Clears package, code, date                  │
│  4. Apply charges        ← Uses package ref saved in step 2            │
│  5. Update status        ← Uses package ref saved in step 2            │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                    WRONG ORDER (Bug!)                                   │
│                                                                        │
│  1. Release locker       ← Clears currentPackage = null                │
│  2. Calculate charges    ← Returns BigDecimal.ZERO (no package!)       │
│  3. Get package ref      ← Returns null!                               │
│  4. Apply charges        ← NullPointerException!                       │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

**Transactional approach:**

```java
public Locker pickUpPackageTransactional(String accessCode) {
    Locker locker = accessCodeMap.get(accessCode);

    if (locker == null || !locker.checkAccessCode(accessCode)) {
        return null;
    }

    // Capture all data needed BEFORE any mutations
    ShippingPackage pkg = locker.getPackage();
    Account user = pkg.getUser();
    BigDecimal charge;

    try {
        charge = locker.calculateStorageCharges();
    } catch (MaximumStoragePeriodExceededException e) {
        pkg.updateShippingStatus(ShippingStatus.EXPIRED);
        locker.releaseLocker();
        accessCodeMap.remove(accessCode);
        return locker;
    }

    // All validations passed - now perform mutations
    locker.releaseLocker();
    accessCodeMap.remove(accessCode);  // Don't forget to clean up!
    user.addUsageCharge(charge);
    pkg.updateShippingStatus(ShippingStatus.RETRIEVED);

    return locker;
}
```

**Reference:** `shippinglocker/locker/LockerManager.java:43-59`

---

### Parking at Code

```
shipping_locker/
├── shippinglocker/
│   ├── Site.java                    # Physical location with locker pools
│   ├── NotificationInterface.java   # Notification abstraction
│   ├── ShippingLockerTest.java      # Integration tests
│   │
│   ├── account/
│   │   ├── Account.java             # User account with charges
│   │   └── AccountLockerPolicy.java # Per-account storage rules
│   │
│   ├── locker/
│   │   ├── Locker.java              # Individual locker with state
│   │   ├── LockerSize.java          # Enum: SMALL/MEDIUM/LARGE
│   │   └── LockerManager.java       # Orchestrates assign/pickup
│   │
│   └── pkg/
│       ├── ShippingPackage.java     # Package interface
│       ├── BasicShippingPackage.java # Standard implementation
│       ├── ShippingStatus.java      # Enum: CREATED/IN_TRANSIT/etc.
│       ├── NoLockerAvailableException.java
│       ├── PackageIncompatibleException.java
│       └── MaximumStoragePeriodExceededException.java
│
└── shipping_locker_deep_dive/
    ├── LockerEventObserver.java     # Observer pattern interface
    ├── LockerFactory.java           # Factory for locker creation
    ├── EmailNotification.java       # Notification implementation
    └── LockerManagerChange.java     # Extended manager with observers
```

### Class-by-Class Analysis

| Class | Purpose | Key Methods | Pattern |
|-------|---------|-------------|---------|
| `LockerManager` | Orchestrates operations | `assignPackage()`, `pickUpPackage()` | Facade |
| `Site` | Manages locker pools | `findAvailableLocker()`, `placePackage()` | — |
| `Locker` | Individual locker state | `assignPackage()`, `calculateStorageCharges()` | — |
| `LockerSize` | Size enum with pricing | `getWidth()`, `dailyCharge` | Enum |
| `Account` | User with policy | `addUsageCharge()`, `getLockerPolicy()` | — |
| `AccountLockerPolicy` | Storage rules | `getFreePeriodDays()`, `getMaximumPeriodDays()` | Policy/Strategy |
| `BasicShippingPackage` | Package with dimensions | `getLockerSize()`, `updateShippingStatus()` | — |
| `ShippingStatus` | Package lifecycle | CREATED, IN_LOCKER, RETRIEVED, EXPIRED | Enum |
| `NotificationInterface` | Channel abstraction | `sendNotification()` | Strategy |

---

## Extension Guide: Adding a New Locker Size

**Scenario:** Add an EXTRA_LARGE locker (40x40x40, $20/day)

**Step 1: Update LockerSize enum**

```java
public enum LockerSize {
    SMALL("Small", new BigDecimal("5.00"),
          new BigDecimal("10.00"), new BigDecimal("10.00"), new BigDecimal("10.00")),
    MEDIUM("Medium", new BigDecimal("10.00"),
           new BigDecimal("20.00"), new BigDecimal("20.00"), new BigDecimal("20.00")),
    LARGE("Large", new BigDecimal("15.00"),
          new BigDecimal("30.00"), new BigDecimal("30.00"), new BigDecimal("30.00")),
    // NEW SIZE
    EXTRA_LARGE("Extra Large", new BigDecimal("20.00"),
                new BigDecimal("40.00"), new BigDecimal("40.00"), new BigDecimal("40.00"));
    // ... rest unchanged
}
```

**Step 2: Update Site configuration**

```java
Map<LockerSize, Integer> lockerConfig = new HashMap<>();
lockerConfig.put(LockerSize.SMALL, 10);
lockerConfig.put(LockerSize.MEDIUM, 8);
lockerConfig.put(LockerSize.LARGE, 5);
lockerConfig.put(LockerSize.EXTRA_LARGE, 3);  // Add new size

Site site = new Site(lockerConfig);
```

**Step 3: No changes needed in:**
- `BasicShippingPackage.getLockerSize()` - automatically iterates all enum values
- `Locker` - works with any `LockerSize`
- `LockerManager` - size-agnostic

**This is Open/Closed Principle in action:** The system is open for extension (new sizes) but closed for modification (existing code unchanged).

---

## Wrap Up

### Key Takeaways

1. **Policy Pattern** for account tiers: `AccountLockerPolicy` encapsulates storage rules, allowing different pricing for different customer segments without changing core logic

2. **First-Fit Algorithm** for locker selection: Simple, efficient, and works well when sizes are naturally ordered

3. **BigDecimal for Money**: Critical for financial calculations—never use `double` or `float` for currency

4. **Observer Pattern** for notifications: Decouples locker events from notification channels, enabling email, SMS, push without code changes

5. **Validation Order Matters**: Calculate charges before releasing locker; capture references before mutations

6. **Enum with Data**: `LockerSize` enum carries dimensions and pricing, making the type system work for us

---

## Further Reading

### Policy Pattern

The Policy pattern (a specialization of Strategy) encapsulates business rules:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│              Account                                                    │
│                 │                                                       │
│                 │ uses                                                  │
│                 ▼                                                       │
│        AccountLockerPolicy                                              │
│                 │                                                       │
│     ┌───────────┼───────────┐                                          │
│     │           │           │                                          │
│     ▼           ▼           ▼                                          │
│  Standard    Premium    Business                                        │
│  (3d/7d)     (5d/14d)   (7d/30d)                                       │
│                                                                         │
│  The Locker doesn't know about account tiers—                          │
│  it just uses whatever policy is attached to the user.                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Benefits:**
- Business rules are encapsulated and testable
- New tiers require no changes to Locker or LockerManager
- Easy to A/B test different pricing strategies

### Observer Pattern

The Observer pattern for event-driven notifications:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│              LockerManager (Subject)                                    │
│                     │                                                   │
│                     │ notifies                                          │
│                     ▼                                                   │
│           ┌─────────────────┐                                          │
│           │LockerEventObserver                                         │
│           │  <<interface>>  │                                          │
│           └────────┬────────┘                                          │
│                    │                                                    │
│        ┌───────────┼───────────┐                                       │
│        │           │           │                                       │
│        ▼           ▼           ▼                                       │
│   Notification  Analytics   Audit                                       │
│    Observer     Observer   Observer                                     │
│                                                                         │
│  Each observer reacts to events independently.                         │
│  Adding new observers requires no changes to LockerManager.            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**When to use Observer:**
- Multiple independent reactions to the same event
- Event sources shouldn't know about event consumers
- Adding new consumers should be easy

### Factory Pattern (Extension)

For creating lockers with consistent initialization:

```java
public class LockerFactory {
    public static Locker createLocker(LockerSize size) {
        Locker locker = new Locker(size);
        // Additional initialization
        // - Set locker ID
        // - Register with monitoring system
        // - Initialize sensors
        return locker;
    }

    public static Set<Locker> createLockerPool(LockerSize size, int count) {
        Set<Locker> pool = new HashSet<>();
        for (int i = 0; i < count; i++) {
            pool.add(createLocker(size));
        }
        return pool;
    }
}
```

**Benefits:**
- Centralizes locker creation logic
- Easy to add initialization steps
- Consistent locker setup across the system
