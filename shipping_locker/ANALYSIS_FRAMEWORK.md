# 10

## Design a Shipping Locker System

A Shipping Locker System (like Amazon Locker) provides secure, self-service package pickup locations. Packages are delivered to lockers, customers receive access codes, and pick up at their convenience. This system must handle **size-based locker assignment**, **time-based storage charges**, **access code management**, and **account policies**—making it a practical test of real-world business logic.

This design focuses on **resource allocation** (matching packages to lockers), **policy-based charging**, and **notification integration**.

---

## Requirements Gathering

### Requirements Clarification

**Q: What are the core entities in the system?**
A: Sites (physical locations) → Lockers (different sizes) → Packages → Accounts (customers with policies)

**Q: How are lockers sized and priced?**
A: Three sizes with different dimensions and daily charges:
- SMALL: 10×10×10 inches, $5/day
- MEDIUM: 20×20×20 inches, $10/day
- LARGE: 30×30×30 inches, $15/day

**Q: How is the appropriate locker size determined for a package?**
A: The package declares its required `LockerSize`. The system finds an available locker of that size. This is a **smallest-fit** problem similar to memory allocation.

**Q: How does the charging model work?**
A: Accounts have policies defining:
- **Free period**: Days of free storage (e.g., 3 days)
- **Maximum period**: Maximum allowed storage (e.g., 14 days)
- After free period: daily charge based on locker size
- After maximum period: package marked as EXPIRED

**Q: How do customers access their packages?**
A: A unique 6-digit access code is generated when the package is assigned. Customer enters this code to open the locker.

**Q: What happens when a package is picked up?**
A:
1. Access code verified
2. Storage charges calculated
3. Charges added to account
4. Package status → RETRIEVED
5. Locker released for reuse

**Q: How are customers notified?**
A: The system uses a `NotificationInterface` for sending pickup notifications—allowing email, SMS, or push notifications.

### Requirements State

| Requirement | Status | Business Impact |
|-------------|--------|-----------------|
| Multi-size lockers (S/M/L) | ✅ Confirmed | Physical constraints |
| Size-based daily charges | ✅ Confirmed | Revenue model |
| Package-to-locker assignment | ✅ Confirmed | Core operation |
| Access code generation | ✅ Confirmed | Security |
| Free storage period (policy) | ✅ Confirmed | Customer benefit |
| Maximum storage period | ✅ Confirmed | Locker turnover |
| Storage charge calculation | ✅ Confirmed | Billing |
| Account-based policies | ✅ Confirmed | Tiered service |
| Notification on delivery | ✅ Confirmed | Customer experience |
| Package status tracking | ✅ Confirmed | Visibility |
| Locker upgrade (S→M if S full) | ❌ Out of scope | — |
| Multi-site management | ❌ Out of scope | — |

---

## Identify Core Objects

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         SHIPPING LOCKER SYSTEM OBJECTS                               │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                           LOCATION DOMAIN                                    │   │
│  │  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐       │   │
│  │  │   LockerManager  │───►│      Site        │───►│     Locker       │       │   │
│  │  │                  │    │                  │    │                  │       │   │
│  │  │- site            │    │- lockers (by size)    │- size            │       │   │
│  │  │- accounts        │    │                  │    │- currentPackage  │       │   │
│  │  │- accessCodeMap   │    │+findAvailable()  │    │- assignmentDate  │       │   │
│  │  │- notification    │    │+placePackage()   │    │- accessCode      │       │   │
│  │  │                  │    │                  │    │                  │       │   │
│  │  │+assignPackage()  │    └──────────────────┘    │+assignPackage()  │       │   │
│  │  │+pickUpPackage()  │                            │+releaseLocker()  │       │   │
│  │  └──────────────────┘                            │+calculateCharges()       │   │
│  │                                                  └──────────────────┘       │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                           PACKAGE DOMAIN                                     │   │
│  │  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐       │   │
│  │  │ ShippingPackage  │    │BasicShippingPkg  │    │  ShippingStatus  │       │   │
│  │  │  <<interface>>   │◄───│                  │    │     (enum)       │       │   │
│  │  │                  │    │- orderId         │    │                  │       │   │
│  │  │+getOrderId()     │    │- user            │    │ PENDING          │       │   │
│  │  │+getUser()        │    │- dimensions      │    │ IN_LOCKER        │       │   │
│  │  │+getLockerSize()  │    │- status          │    │ RETRIEVED        │       │   │
│  │  │+updateStatus()   │    │                  │    │ EXPIRED          │       │   │
│  │  └──────────────────┘    └──────────────────┘    └──────────────────┘       │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                           ACCOUNT DOMAIN                                     │   │
│  │  ┌──────────────────┐    ┌──────────────────────┐                           │   │
│  │  │     Account      │───►│ AccountLockerPolicy  │                           │   │
│  │  │                  │    │                      │                           │   │
│  │  │- accountId       │    │- freePeriodDays      │  Different tiers:         │   │
│  │  │- ownerName       │    │- maximumPeriodDays   │  Basic: 3 free, 7 max     │   │
│  │  │- usageCharges    │    │                      │  Prime: 7 free, 14 max    │   │
│  │  │- lockerPolicy    │    │                      │                           │   │
│  │  │                  │    │                      │                           │   │
│  │  │+addUsageCharge() │    │                      │                           │   │
│  │  └──────────────────┘    └──────────────────────┘                           │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                        LOCKER SIZE (Enum)                                    │   │
│  │  ┌──────────────────────────────────────────────────────────────────────┐   │   │
│  │  │  SMALL   │  10×10×10  │  $5/day   │                                  │   │   │
│  │  │  MEDIUM  │  20×20×20  │  $10/day  │                                  │   │   │
│  │  │  LARGE   │  30×30×30  │  $15/day  │                                  │   │   │
│  │  └──────────────────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Design Class Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                           STORAGE CHARGE CALCULATION                                  │
└──────────────────────────────────────────────────────────────────────────────────────┘

  Account Policy: freePeriodDays = 3, maximumPeriodDays = 14
  Locker Size: MEDIUM ($10/day)

  Timeline:
  ═════════════════════════════════════════════════════════════════════════

  Day 0        Day 3              Day 14                    Day N
    │            │                  │                         │
    ▼            ▼                  ▼                         ▼
  ┌─────────────┬─────────────────┬─────────────────────────┬──────────
  │  FREE       │   CHARGEABLE    │        EXPIRED          │
  │  PERIOD     │    PERIOD       │  (MaximumStorageExceededException)
  │  (3 days)   │   (11 days)     │                         │
  └─────────────┴─────────────────┴─────────────────────────┴──────────

  Example: Package picked up on Day 5
  ─────────────────────────────────
  Total days stored: 5
  Free days: 3
  Chargeable days: 5 - 3 = 2
  Charge: 2 × $10 = $20


  Example: Package picked up on Day 15
  ─────────────────────────────────────
  Total days stored: 15 > maximum (14)
  Status: EXPIRED
  Throws: MaximumStoragePeriodExceededException


┌──────────────────────────────────────────────────────────────────────────────────────┐
│                              PACKAGE LIFECYCLE                                        │
└──────────────────────────────────────────────────────────────────────────────────────┘

    ┌──────────┐    assignPackage()    ┌──────────┐    pickUp()    ┌──────────┐
    │ PENDING  │──────────────────────►│IN_LOCKER │───────────────►│RETRIEVED │
    │          │                       │          │                │          │
    └──────────┘                       └────┬─────┘                └──────────┘
                                            │
                                            │ maxPeriod exceeded
                                            ▼
                                       ┌──────────┐
                                       │ EXPIRED  │
                                       │          │
                                       └──────────┘
```

---

## Code

### Deep Dive Topics

#### Storage Charge Calculation: Policy-Based Billing

The charge calculation considers account policy (free period, max period) and locker size:

```java
public BigDecimal calculateStorageCharges() {
    if (currentPackage == null || assignmentDate == null) {
        return BigDecimal.ZERO;
    }

    AccountLockerPolicy policy = currentPackage.getUser().getLockerPolicy();
    long totalDaysUsed = (new Date().getTime() - assignmentDate.getTime())
                         / (1000 * 60 * 60 * 24);

    // Check maximum period violation
    if (totalDaysUsed > policy.getMaximumPeriodDays()) {
        currentPackage.updateShippingStatus(ShippingStatus.EXPIRED);
        throw new MaximumStoragePeriodExceededException(
            "Package exceeded maximum storage of " + policy.getMaximumPeriodDays() + " days");
    }

    // Calculate chargeable days (after free period)
    long chargeableDays = Math.max(0, totalDaysUsed - policy.getFreePeriodDays());
    return size.dailyCharge.multiply(new BigDecimal(chargeableDays));
}
```

**Business logic:**
- Days 1 to `freePeriodDays`: Free
- Days `freePeriodDays+1` to `maximumPeriodDays`: Charged at daily rate
- Beyond `maximumPeriodDays`: Package expires

**Reference:** `shippinglocker/locker/Locker.java:43-61`

---

#### Access Code Security

The system generates 6-digit random codes for pickup authentication:

```java
private String generateAccessCode() {
    Random random = new Random();
    int accessCode = 100000 + random.nextInt(900000);  // 100000-999999
    return String.valueOf(accessCode);
}
```

**Security considerations:**
- 6 digits = 900,000 possible combinations
- Code is single-use (cleared on pickup)
- Could enhance with: expiration, attempt limits, hashing

**Reference:** `shippinglocker/locker/Locker.java:74-78`

---

#### Notification Integration

The system uses interface-based notification for flexibility:

```java
public interface NotificationInterface {
    void sendNotification(String message, Account user);
}

// In LockerManager
public Locker assignPackage(ShippingPackage pkg, Date date) {
    Locker locker = site.placePackage(pkg, date);
    if (locker != null) {
        accessCodeMap.put(locker.getAccessCode(), locker);
        notificationService.sendNotification(
            "Package assigned to locker " + locker.getAccessCode(),
            pkg.getUser()
        );
    }
    return locker;
}
```

**Benefits:**
- Email, SMS, push can all implement the interface
- Easy to add new notification channels
- Testable with mock implementation

**Reference:** `shippinglocker/NotificationInterface.java`, `shippinglocker/locker/LockerManager.java:33-40`

---

### Parking at Code

```
shipping_locker/
├── shippinglocker/
│   ├── Site.java                    # Physical location with lockers
│   ├── NotificationInterface.java   # Notification contract
│   ├── ShippingLockerTest.java
│   ├── locker/
│   │   ├── Locker.java              # Individual locker with package
│   │   ├── LockerManager.java       # Locker operations coordinator
│   │   └── LockerSize.java          # Size enum with dimensions/charges
│   ├── pkg/
│   │   ├── ShippingPackage.java     # Package interface
│   │   ├── BasicShippingPackage.java
│   │   ├── ShippingStatus.java      # Status enum
│   │   ├── NoLockerAvailableException.java
│   │   ├── MaximumStoragePeriodExceededException.java
│   │   └── PackageIncompatibleException.java
│   └── account/
│       ├── Account.java             # User with policy and charges
│       └── AccountLockerPolicy.java # Free/max period rules
└── shipping_locker_deep_dive/
    ├── LockerFactory.java           # Factory for creating lockers
    ├── LockerEventObserver.java     # Observer for locker events
    └── EmailNotification.java       # Notification implementation
```

---

## Wrap Up

Key takeaways:

1. **Size-Based Resource Allocation**: Matching packages to appropriate locker sizes
2. **Policy Pattern**: `AccountLockerPolicy` allows different rules for different account tiers
3. **Time-Based Charging**: Free period → chargeable → expired lifecycle
4. **Interface for Notifications**: Decouples notification logic from locker management
5. **Access Code Security**: Single-use codes for secure pickup
6. **Exception-Driven Flow**: `MaximumStoragePeriodExceededException` for business rule violations

---

## Further Reading

### Strategy Pattern (via Policy)

The `AccountLockerPolicy` acts as a strategy for calculating charges:

```
Account ────► AccountLockerPolicy
                    │
              ┌─────┴─────┐
              │           │
         BasicPolicy  PrimePolicy
         (3 free,     (7 free,
          7 max)       14 max)
```

### Factory Pattern (Deep Dive)

`LockerFactory` can create lockers with appropriate configurations:

```java
public class LockerFactory {
    public Locker createLocker(LockerSize size) {
        return new Locker(size);
    }
}
```
