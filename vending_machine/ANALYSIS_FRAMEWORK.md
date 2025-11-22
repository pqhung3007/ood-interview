# 09

## Design a Vending Machine

A Vending Machine is a self-service retail device that dispenses products after receiving payment. It's a classic OOD interview problem that tests understanding of **transaction management**, **state machine design**, **inventory tracking**, and **payment processing**. Real vending machines must handle numerous edge cases: insufficient funds, out-of-stock items, transaction cancellation, change calculation, and maintaining data integrity across all operations.

This design presents unique challenges around **atomic transaction processing** (all-or-nothing operations), **state transitions** (what actions are valid in each state), and **separation of concerns** (inventory vs payment vs transaction history)—making it excellent for testing a candidate's ability to model real-world business processes.

---

## Requirements Gathering

Before diving into the design, let's clarify the requirements through a Q&A format, simulating an interview scenario. Pay attention to the **transaction lifecycle** and **edge cases**.

### Requirements Clarification

**Q: What are the core operations of a vending machine?**
A: The primary flow is: Insert money → Select product → Confirm purchase → Dispense product → Return change. The machine must also support transaction cancellation at any point before dispensing.

**Q: How is the product inventory organized?**
A: Products are stored in **racks** (also called slots or spirals) identified by codes (e.g., "A1", "B2", "C3"). Each rack contains a single product type with a quantity count. This mirrors physical vending machines where each spiral/slot holds one product type. A rack with count=0 is "sold out."

**Q: Can multiple racks contain the same product?**
A: Yes, the same product (e.g., "Coca-Cola") could be in racks A1 and A2. Each rack is independent with its own inventory count.

**Q: What payment methods should be supported?**
A: For this design, we focus on cash/coins. The machine tracks a running balance as money is inserted. Users can insert money multiple times before selecting a product. The balance must be sufficient to cover the product price before dispensing.

**Q: How should change be handled?**
A: After a successful purchase, the machine returns the difference between the inserted amount and the product price. For simplicity, we assume the machine always has sufficient change (real systems would need to track coin/bill inventory).

**Q: What happens if the user inserts money but doesn't complete the purchase?**
A: The user can cancel the transaction at any time before confirming. All inserted money is returned, and the machine resets to its initial state ready for the next customer.

**Q: What validations should occur before dispensing a product?**
A: Three validations must pass:
1. **Product selection is valid**: The rack code exists
2. **Product is in stock**: Rack count > 0
3. **Sufficient funds**: Balance >= product price

If any validation fails, the transaction should be rejected with an appropriate error message.

**Q: Should the machine track transaction history?**
A: Yes, for auditing, revenue tracking, and troubleshooting. Each completed transaction should be recorded with product, rack, amount, and timestamp.

**Q: What states can the vending machine be in?**
A: The machine transitions through well-defined states:
- **No Money Inserted**: Initial state, waiting for payment
- **Money Inserted**: Has balance, waiting for product selection
- **Product Selected**: Ready to dispense, waiting for confirmation
- **Dispensing**: Processing the transaction (dispense + change)

**Q: Can users add more money after selecting a product?**
A: Yes, if they selected a product but don't have enough balance, they can insert more money before confirming.

**Q: What happens during a power failure mid-transaction?**
A: Out of scope for this design, but worth mentioning. Real systems need transaction recovery mechanisms.

### Requirements State

Based on the clarifications above, here are our finalized requirements:

| Requirement | Status | Business Impact |
|-------------|--------|-----------------|
| Rack-based inventory system | ✅ Confirmed | Physical layout modeling |
| Product with code, description, price | ✅ Confirmed | Product catalog |
| Insert money (accumulating balance) | ✅ Confirmed | Payment flow |
| Select product by rack code | ✅ Confirmed | User interaction |
| Validate rack exists | ✅ Confirmed | Error handling |
| Validate inventory before dispense | ✅ Confirmed | Prevents overselling |
| Validate sufficient funds | ✅ Confirmed | Prevents underpayment |
| Dispense product (decrement inventory) | ✅ Confirmed | Core functionality |
| Calculate and return change | ✅ Confirmed | Payment completion |
| Cancel transaction and full refund | ✅ Confirmed | User experience |
| Transaction history tracking | ✅ Confirmed | Auditing & reporting |
| State pattern for machine states | ✅ Confirmed (deep dive) | Clean state management |
| Multiple payment methods (card, mobile) | ❌ Out of scope | — |
| Change inventory tracking | ❌ Out of scope | — |
| Remote inventory monitoring | ❌ Out of scope | — |
| Transaction recovery | ❌ Out of scope | — |

---

## Identify Core Objects

From the requirements, we can identify the following core objects:

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          VENDING MACHINE SYSTEM OBJECTS                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         CORE DOMAIN                                          │   │
│  │                                                                             │   │
│  │  ┌───────────────────┐                                                      │   │
│  │  │  VendingMachine   │    Central controller (Facade)                       │   │
│  │  │                   │    Coordinates all operations                        │   │
│  │  │ - inventoryMgr    │                                                      │   │
│  │  │ - paymentProc     │                                                      │   │
│  │  │ - currentTxn      │                                                      │   │
│  │  │ - txnHistory      │                                                      │   │
│  │  │                   │                                                      │   │
│  │  │ + insertMoney()   │                                                      │   │
│  │  │ + chooseProduct() │                                                      │   │
│  │  │ + confirmTxn()    │                                                      │   │
│  │  │ + cancelTxn()     │                                                      │   │
│  │  └───────────────────┘                                                      │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                      INVENTORY DOMAIN                                        │   │
│  │                                                                             │   │
│  │  ┌───────────────────┐         ┌───────────────────┐                        │   │
│  │  │ InventoryManager  │────────►│       Rack        │                        │   │
│  │  │                   │   1:N   │                   │                        │   │
│  │  │ - racks (Map)     │         │ - rackCode        │                        │   │
│  │  │                   │         │ - product ────────┼──►┌─────────────┐      │   │
│  │  │ + getProductIn()  │         │ - count           │   │   Product   │      │   │
│  │  │ + dispenseFrom()  │         │                   │   │             │      │   │
│  │  │ + getRack()       │         │ + getProductCount │   │ - code      │      │   │
│  │  │ + updateRack()    │         │ + setCount()      │   │ - desc      │      │   │
│  │  └───────────────────┘         └───────────────────┘   │ - unitPrice │      │   │
│  │                                                        └─────────────┘      │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                      PAYMENT DOMAIN                                          │   │
│  │                                                                             │   │
│  │  ┌───────────────────┐                                                      │   │
│  │  │ PaymentProcessor  │    Handles all monetary operations                   │   │
│  │  │                   │                                                      │   │
│  │  │ - currentBalance  │    Uses BigDecimal for precision                     │   │
│  │  │                   │                                                      │   │
│  │  │ + addBalance()    │    Accumulates inserted money                        │   │
│  │  │ + charge()        │    Deducts product price                             │   │
│  │  │ + returnChange()  │    Returns remaining balance, resets to 0            │   │
│  │  │ + getBalance()    │                                                      │   │
│  │  └───────────────────┘                                                      │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                      TRANSACTION DOMAIN                                      │   │
│  │                                                                             │   │
│  │  ┌───────────────────┐                                                      │   │
│  │  │    Transaction    │    Captures complete purchase context                │   │
│  │  │                   │                                                      │   │
│  │  │ - rack            │    Which rack was accessed                           │   │
│  │  │ - product         │    What product was purchased                        │   │
│  │  │ - totalAmount     │    Change returned (for history)                     │   │
│  │  │                   │                                                      │   │
│  │  └───────────────────┘                                                      │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                      STATE PATTERN (Deep Dive)                               │   │
│  │                                                                             │   │
│  │  ┌───────────────────────┐                                                  │   │
│  │  │ VendingMachineState   │◄────────────┬──────────────┬──────────────┐      │   │
│  │  │    <<interface>>      │             │              │              │      │   │
│  │  ├───────────────────────┤             │              │              │      │   │
│  │  │ + insertMoney()       │        ┌────┴────┐   ┌────┴────┐   ┌────┴────┐  │   │
│  │  │ + selectProductByCode │        │ NoMoney │   │  Money  │   │Dispense │  │   │
│  │  │ + dispenseProduct()   │        │Inserted │   │Inserted │   │  State  │  │   │
│  │  │ + getStateDescription │        │  State  │   │  State  │   │         │  │   │
│  │  └───────────────────────┘        └─────────┘   └─────────┘   └─────────┘  │   │
│  │                                                                             │   │
│  │  Each state knows what actions are valid and handles transitions            │   │
│  │                                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

**Core objects identified:**

1. **VendingMachine**: Main facade coordinating all operations
2. **InventoryManager**: Manages rack-to-product mappings
3. **Rack**: Physical slot with product and count
4. **Product**: Item with code, description, and price (immutable)
5. **PaymentProcessor**: Handles balance accumulation and change
6. **Transaction**: Records completed purchases
7. **VendingMachineState**: Interface for state pattern (deep dive)
8. **InvalidTransactionException**: Business rule violation

---

## Design Class Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                                  CLASS DIAGRAM                                        │
└──────────────────────────────────────────────────────────────────────────────────────┘

                            ┌─────────────────────┐
                            │   VendingMachine    │
                            │      (Facade)       │
                            ├─────────────────────┤
                            │ - transactionHistory│
                            │ - inventoryManager  │
                            │ - paymentProcessor  │
                            │ - currentTransaction│
                            ├─────────────────────┤
                            │ + setRack()         │
                            │ + insertMoney()     │
                            │ + chooseProduct()   │
                            │ + confirmTransaction│
                            │ + cancelTransaction │
                            │ + getHistory()      │
                            └──────────┬──────────┘
                                       │
         ┌─────────────────────────────┼─────────────────────────────┐
         │                             │                             │
         ▼                             ▼                             ▼
  ┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
  │InventoryManager │      │PaymentProcessor  │      │   Transaction    │
  ├──────────────────┤      ├──────────────────┤      ├──────────────────┤
  │- racks (Map)     │      │- currentBalance  │      │- rack            │
  ├──────────────────┤      │  (BigDecimal)    │      │- product         │
  │+getProductInRack │      ├──────────────────┤      │- totalAmount     │
  │+dispenseFromRack │      │+addBalance()     │      ├──────────────────┤
  │+updateRack()     │      │+charge()         │      │+setProduct()     │
  │+getRack()        │      │+returnChange()   │      │+setRack()        │
  └────────┬─────────┘      │+getCurrentBalance│      │+setTotalAmount() │
           │                └──────────────────┘      └──────────────────┘
           │ manages
           ▼
    ┌──────────────┐         ┌──────────────┐
    │     Rack     │────────►│   Product    │
    ├──────────────┤  holds  ├──────────────┤
    │- rackCode    │         │- productCode │
    │- product     │         │- description │
    │- count       │         │- unitPrice   │
    ├──────────────┤         │  (BigDecimal)│
    │+getProduct() │         ├──────────────┤
    │+getCount()   │         │+getCode()    │
    │+setCount()   │         │+getPrice()   │
    └──────────────┘         └──────────────┘


┌──────────────────────────────────────────────────────────────────────────────────────┐
│                              TRANSACTION FLOW                                         │
└──────────────────────────────────────────────────────────────────────────────────────┘

    Customer               VendingMachine           PaymentProcessor    InventoryManager
        │                        │                         │                   │
        │  insertMoney($5)       │                         │                   │
        │───────────────────────►│   addBalance($5)        │                   │
        │                        │────────────────────────►│                   │
        │                        │   balance = $5          │                   │
        │                        │◄────────────────────────│                   │
        │                        │                         │                   │
        │  insertMoney($2)       │                         │                   │
        │───────────────────────►│   addBalance($2)        │                   │
        │                        │────────────────────────►│                   │
        │                        │   balance = $7          │                   │
        │                        │◄────────────────────────│                   │
        │                        │                         │                   │
        │  chooseProduct("A1")   │                         │                   │
        │───────────────────────►│   getProductInRack("A1")│                   │
        │                        │────────────────────────────────────────────►│
        │                        │   product (Chips, $2.50)│                   │
        │                        │◄────────────────────────────────────────────│
        │                        │                         │                   │
        │                        │   getRack("A1")         │                   │
        │                        │────────────────────────────────────────────►│
        │                        │   rack (A1, count=5)    │                   │
        │                        │◄────────────────────────────────────────────│
        │                        │                         │                   │
        │                        │   currentTransaction.setProduct(Chips)      │
        │                        │   currentTransaction.setRack(A1)            │
        │                        │                         │                   │
        │  confirmTransaction()  │                         │                   │
        │───────────────────────►│                         │                   │
        │                        │  ┌─────────────────────────────────────┐    │
        │                        │  │ validateTransaction()               │    │
        │                        │  │  ├─ product != null?     ✓         │    │
        │                        │  │  ├─ rack.count > 0?      ✓ (5>0)   │    │
        │                        │  │  └─ balance >= price?    ✓ ($7>=$2.50)  │
        │                        │  └─────────────────────────────────────┘    │
        │                        │                         │                   │
        │                        │   charge($2.50)         │                   │
        │                        │────────────────────────►│                   │
        │                        │   balance = $4.50       │                   │
        │                        │◄────────────────────────│                   │
        │                        │                         │                   │
        │                        │   dispenseProductFromRack(A1)               │
        │                        │────────────────────────────────────────────►│
        │                        │   rack.count = 4        │                   │
        │                        │◄────────────────────────────────────────────│
        │                        │                         │                   │
        │                        │   returnChange()        │                   │
        │                        │────────────────────────►│                   │
        │   change = $4.50       │   $4.50                 │                   │
        │◄───────────────────────│◄────────────────────────│                   │
        │                        │                         │                   │
        │                        │   transactionHistory.add(txn)               │
        │                        │   currentTransaction = new Transaction()    │


┌──────────────────────────────────────────────────────────────────────────────────────┐
│                           STATE PATTERN TRANSITIONS                                   │
└──────────────────────────────────────────────────────────────────────────────────────┘

                              ┌─────────────────────┐
                              │   NoMoneyInserted   │
                              │       State         │
                              │                     │
                              │ insertMoney() → OK  │
                              │ selectProduct()→ERR │
                              │ dispense() → ERR    │
                              └──────────┬──────────┘
                                         │
                                         │ insertMoney()
                                         ▼
                              ┌─────────────────────┐
                              │   MoneyInserted     │
              insertMoney()   │       State         │
            ┌────────────────►│                     │
            │                 │ insertMoney() → OK  │◄────────┐
            │                 │ selectProduct()→OK  │         │
            │                 │ dispense() → ERR    │         │
            │                 └──────────┬──────────┘         │
            │                            │                    │
            │                            │ selectProduct()    │
            │                            ▼                    │
            │                 ┌─────────────────────┐         │
            │                 │    DispenseState    │         │
            │                 │                     │         │
            │                 │ insertMoney() → OK  │─────────┘
            │                 │ selectProduct()→ERR │   (add more money)
            │                 │ dispense() → OK     │
            │                 └──────────┬──────────┘
            │                            │
            │                            │ dispense() success
            │                            ▼
            │                 ┌─────────────────────┐
            └─────────────────│   NoMoneyInserted   │
               (reset after   │       State         │
                dispense)     └─────────────────────┘


  INVALID ACTION HANDLING:
  ════════════════════════

  ┌────────────────────────┬─────────────────────┬─────────────────────────────────┐
  │ Current State          │ Action Attempted    │ Result                          │
  ├────────────────────────┼─────────────────────┼─────────────────────────────────┤
  │ NoMoneyInserted        │ selectProduct()     │ InvalidStateException:          │
  │                        │                     │ "Cannot select without money"   │
  ├────────────────────────┼─────────────────────┼─────────────────────────────────┤
  │ NoMoneyInserted        │ dispense()          │ InvalidStateException:          │
  │                        │                     │ "Cannot dispense without money" │
  ├────────────────────────┼─────────────────────┼─────────────────────────────────┤
  │ MoneyInserted          │ dispense()          │ InvalidStateException:          │
  │                        │                     │ "Please select a product first" │
  ├────────────────────────┼─────────────────────┼─────────────────────────────────┤
  │ DispenseState          │ selectProduct()     │ InvalidStateException:          │
  │                        │                     │ "Product already selected"      │
  └────────────────────────┴─────────────────────┴─────────────────────────────────┘
```

---

## Code

The Vending Machine is implemented using the **Facade Pattern** for coordination, **careful transaction management** for data integrity, and the **State Pattern** (deep dive) for clean state transitions.

### Deep Dive Topics

#### Transaction Validation: Defense in Depth

The `validateTransaction()` method performs three critical checks in a specific order before any money or inventory changes:

```java
private void validateTransaction() throws InvalidTransactionException {
    // 1. Product selection validation - cheapest check first
    if (currentTransaction.getProduct() == null) {
        throw new InvalidTransactionException("Invalid product selection");
    }

    // 2. Inventory validation - requires rack lookup
    if (currentTransaction.getRack().getProductCount() == 0) {
        throw new InvalidTransactionException("Insufficient inventory for product.");
    }

    // 3. Payment validation - requires balance comparison
    if (paymentProcessor.getCurrentBalance()
            .compareTo(currentTransaction.getProduct().getUnitPrice()) < 0) {
        throw new InvalidTransactionException("Insufficient fund");
    }
}
```

**Why this order matters:**

| Check | Cost | Fail Fast Benefit |
|-------|------|-------------------|
| Product null check | O(1), no I/O | Catches programming errors |
| Inventory count check | O(1), memory access | Prevents "sold out" attempts |
| Balance comparison | O(1), BigDecimal compare | Prevents insufficient fund attempts |

**The order follows the "fail fast" principle:** cheapest validations first, so we don't waste resources on expensive checks if a cheap one fails.

**Reference:** `vendingmachine/VendingMachine.java:59-68`

---

#### The Confirm Transaction Flow: Atomic Operations

The `confirmTransaction()` method demonstrates careful ordering of operations:

```java
Transaction confirmTransaction() throws InvalidTransactionException {
    // Step 1: Validate BEFORE any state changes
    validateTransaction();

    // Step 2: Charge the customer (deduct from balance)
    paymentProcessor.charge(currentTransaction.getProduct().getUnitPrice());

    // Step 3: Dispense the product (decrement inventory)
    inventoryManager.dispenseProductFromRack(currentTransaction.getRack());

    // Step 4: Calculate and return change
    currentTransaction.setTotalAmount(paymentProcessor.returnChange());

    // Step 5: Record the transaction
    transactionHistory.add(currentTransaction);
    Transaction completedTransaction = currentTransaction;

    // Step 6: Reset for next customer
    currentTransaction = new Transaction();
    return completedTransaction;
}
```

**Critical ordering:**
1. **Validate first**: No side effects if validation fails
2. **Charge before dispense**: Money is secured before releasing product
3. **Change after dispense**: Complete the monetary transaction
4. **Record before reset**: Don't lose transaction data

**What could go wrong with wrong ordering:**

| Wrong Order | Problem |
|-------------|---------|
| Dispense before charge | Product gone, payment fails = loss |
| Reset before record | Transaction data lost |
| Charge before validate | Money taken, but can't dispense |

**Reference:** `vendingmachine/VendingMachine.java:37-57`

---

#### Cancel Transaction: Clean State Reset

The cancellation flow demonstrates proper cleanup:

```java
public void cancelTransaction() {
    // Return all inserted money
    paymentProcessor.returnChange();

    // Reset the current transaction (discards product selection)
    currentTransaction = new Transaction();
}
```

**Important:** This works because:
1. `returnChange()` returns the full balance and resets to zero
2. Creating a new `Transaction` discards any product/rack references
3. No inventory was touched (validation hadn't passed)

**Reference:** `vendingmachine/VendingMachine.java:75-78`

---

#### State Pattern: Eliminating Conditional Complexity (Deep Dive)

Without the State Pattern, the vending machine would have scattered conditionals:

```java
// BAD: Without State Pattern - conditionals everywhere
public void selectProduct(String code) {
    if (state == WAITING_FOR_MONEY) {
        throw new Exception("Insert money first");
    } else if (state == MONEY_INSERTED) {
        // process selection
        state = PRODUCT_SELECTED;
    } else if (state == DISPENSING) {
        throw new Exception("Transaction in progress");
    }
}

public void insertMoney(double amount) {
    if (state == WAITING_FOR_MONEY) {
        balance += amount;
        state = MONEY_INSERTED;
    } else if (state == MONEY_INSERTED) {
        balance += amount;
        // stay in same state
    } else if (state == DISPENSING) {
        balance += amount;
        // stay in same state
    }
}
// ... repeated for every operation
```

**With State Pattern:**

```java
// State interface - each state handles its own logic
public interface VendingMachineState {
    void insertMoney(VendingMachine VM, double amount);
    void selectProductByCode(VendingMachine VM, String productCode) throws InvalidStateException;
    void dispenseProduct(VendingMachine VM) throws InvalidStateException;
    String getStateDescription();
}

// NoMoneyInsertedState - initial state
public class NoMoneyInsertedState implements VendingMachineState {
    @Override
    public void insertMoney(VendingMachine VM, double amount) {
        VM.addBalance(amount);
        VM.setState(new MoneyInsertedState());  // Transition!
    }

    @Override
    public void selectProductByCode(VendingMachine VM, String code) throws InvalidStateException {
        throw new InvalidStateException("Cannot select a product without inserting money.");
    }

    @Override
    public void dispenseProduct(VendingMachine VM) throws InvalidStateException {
        throw new InvalidStateException("Cannot dispense product without inserting money.");
    }

    @Override
    public String getStateDescription() {
        return "No Money Inserted State - Please insert money to proceed";
    }
}

// MoneyInsertedState - has balance, waiting for selection
public class MoneyInsertedState implements VendingMachineState {
    @Override
    public void insertMoney(VendingMachine VM, double amount) {
        VM.getPaymentProcessor().processPayment(new BigDecimal(amount));
        // Stay in MoneyInsertedState
    }

    @Override
    public void selectProductByCode(VendingMachine VM, String productCode) {
        try {
            VM.getInventoryManager().getProductInRack(productCode);
            VM.setSelectedProduct(productCode);
            VM.setState(new DispenseState());  // Transition!
        } catch (Exception e) {
            throw new InvalidStateException("Invalid product selection: " + e.getMessage());
        }
    }

    @Override
    public void dispenseProduct(VendingMachine VM) {
        throw new InvalidStateException("Please select a product first.");
    }

    @Override
    public String getStateDescription() {
        return "Money Inserted State - Please select a product";
    }
}
```

**Benefits of State Pattern:**

| Aspect | Without State Pattern | With State Pattern |
|--------|----------------------|-------------------|
| Adding new state | Modify every method | Add new class |
| Understanding state behavior | Scattered across methods | Localized in state class |
| Testing | Hard to isolate | Test each state independently |
| Transitions | Easy to miss cases | Explicit in each state |
| State-specific messages | Conditionals | Natural per-state |

**Reference:** `vending_machine_deep_dive/VendingMachineState.java`, `vending_machine_deep_dive/NoMoneyInsertedState.java`, `vending_machine_deep_dive/MoneyInsertedState.java`

---

#### BigDecimal for Money: Why Not double?

The system uses `BigDecimal` for all monetary values:

```java
public class PaymentProcessor {
    private BigDecimal currentBalance = BigDecimal.ZERO;

    public void charge(BigDecimal amount) {
        currentBalance = currentBalance.subtract(amount);
    }
}
```

**Why BigDecimal over double:**

```java
// PROBLEM with double
double price = 0.10;
double balance = 1.00;
for (int i = 0; i < 10; i++) {
    balance -= price;
}
System.out.println(balance);  // Expected: 0.0, Actual: 1.3877787807814457E-16

// SOLUTION with BigDecimal
BigDecimal price = new BigDecimal("0.10");
BigDecimal balance = new BigDecimal("1.00");
for (int i = 0; i < 10; i++) {
    balance = balance.subtract(price);
}
System.out.println(balance);  // Correctly: 0.00
```

**Reference:** `vendingmachine/PaymentProcessor.java:6`, `vendingmachine/Product.java:8`

---

### Parking at Code

The following describes the code structure:

```
vending_machine/
├── vendingmachine/
│   ├── VendingMachine.java          # Main facade coordinating operations
│   ├── VendingMachineTest.java      # Comprehensive test suite
│   ├── Product.java                 # Immutable product entity
│   ├── Rack.java                    # Physical slot with product and count
│   ├── InventoryManager.java        # Rack management and dispensing
│   ├── PaymentProcessor.java        # Balance and change handling
│   ├── Transaction.java             # Transaction record for history
│   └── InvalidTransactionException.java  # Business rule violation
└── vending_machine_deep_dive/
    ├── VendingMachineState.java     # State interface
    ├── NoMoneyInsertedState.java    # Initial state
    ├── MoneyInsertedState.java      # After payment received
    ├── DispenseState.java           # Ready to dispense
    └── InvalidStateException.java   # Invalid action for current state
```

---

### VendingMachine (Facade)

The `VendingMachine` class serves as the **Facade**, coordinating inventory, payment, and transaction management:

```java
class VendingMachine {
    private final List<Transaction> transactionHistory;
    private final InventoryManager inventoryManager;
    private final PaymentProcessor paymentProcessor;
    private Transaction currentTransaction;

    public VendingMachine() {
        transactionHistory = new ArrayList<>();
        currentTransaction = new Transaction();
        inventoryManager = new InventoryManager();
        paymentProcessor = new PaymentProcessor();
    }
}
```

**Design decisions:**

| Component | Responsibility | Why Separate |
|-----------|---------------|--------------|
| `InventoryManager` | Product/rack management | Single responsibility |
| `PaymentProcessor` | Money handling | Isolate financial logic |
| `Transaction` | Purchase record | Immutable history entry |
| `transactionHistory` | Audit trail | Reporting/debugging |

**Reference:** `vendingmachine/VendingMachine.java:9-21`

---

### Rack and Product

The `Rack` represents a physical slot, while `Product` is an immutable value object:

```java
public class Rack {
    private final String rackCode;
    private final Product product;
    private int count;  // Mutable - decrements on dispense

    public void setCount(int count) {
        this.count = count;
    }
}

public class Product {
    private final String productCode;    // Immutable
    private final String description;    // Immutable
    private final BigDecimal unitPrice;  // Immutable
}
```

**Key insight:** Product is immutable because prices shouldn't change mid-transaction. Rack count is mutable because inventory changes.

**Reference:** `vendingmachine/Rack.java:1-33`, `vendingmachine/Product.java:1-29`

---

### PaymentProcessor

Handles all monetary operations with precision:

```java
public class PaymentProcessor {
    private BigDecimal currentBalance = BigDecimal.ZERO;

    public void addBalance(BigDecimal amount) {
        currentBalance = currentBalance.add(amount);
    }

    public void charge(BigDecimal amount) {
        currentBalance = currentBalance.subtract(amount);
    }

    public BigDecimal returnChange() {
        BigDecimal change = currentBalance;
        currentBalance = BigDecimal.ZERO;  // Reset for next customer
        return change;
    }

    public BigDecimal getCurrentBalance() {
        return currentBalance;
    }
}
```

**Thread-safety note:** This implementation is not thread-safe. A production system would need synchronization or use `AtomicReference<BigDecimal>`.

**Reference:** `vendingmachine/PaymentProcessor.java:1-23`

---

### Adding a New Product Category

To add a new product category (e.g., cold drinks vs snacks with different dispensing logic):

1. **Extend Product** with category:
```java
public class Product {
    private final ProductCategory category;  // SNACK, COLD_DRINK, HOT_DRINK

    public enum ProductCategory {
        SNACK,
        COLD_DRINK,
        HOT_DRINK
    }
}
```

2. **Create dispensing strategies**:
```java
public interface DispensingStrategy {
    void dispense(Rack rack);
}

public class SnackDispensingStrategy implements DispensingStrategy {
    @Override
    public void dispense(Rack rack) {
        rack.setCount(rack.getProductCount() - 1);
        // Spiral mechanism
    }
}

public class ColdDrinkDispensingStrategy implements DispensingStrategy {
    @Override
    public void dispense(Rack rack) {
        rack.setCount(rack.getProductCount() - 1);
        // Drop mechanism + keep cold
    }
}
```

---

## Wrap Up

In this design, we created a Vending Machine system that handles the complete purchase lifecycle with careful attention to data integrity. Key takeaways:

1. **Separation of Concerns**: `InventoryManager` handles stock, `PaymentProcessor` handles money—each with single responsibility

2. **Transaction as First-Class Object**: Captures complete purchase context for history and auditing

3. **Validation Before Action**: All checks complete before any state changes, ensuring atomicity

4. **Careful Operation Ordering**: Charge before dispense, record before reset—order matters for correctness

5. **State Pattern** (deep dive): Cleanly models machine states, eliminating scattered conditionals

6. **BigDecimal for Money**: Avoids floating-point precision errors in financial calculations

7. **Immutable Products**: Prices don't change mid-transaction; only inventory counts change

---

## Further Reading: State Pattern and Transaction Management

### State Design Pattern

The State pattern allows an object to alter its behavior when its internal state changes. The object will appear to change its class.

**Structure:**

```
┌─────────────────────┐         ┌─────────────────────┐
│   VendingMachine    │────────►│ VendingMachineState │
│     (Context)       │         │    <<interface>>    │
├─────────────────────┤         ├─────────────────────┤
│ - state             │         │ + insertMoney()     │
│                     │         │ + selectProduct()   │
│ + insertMoney()     │         │ + dispenseProduct() │
│ + selectProduct()   │         └──────────┬──────────┘
│ + dispense()        │                    │
└─────────────────────┘         ┌──────────┴──────────┐
                                │          │          │
                                ▼          ▼          ▼
                          ┌─────────┐┌─────────┐┌─────────┐
                          │NoMoney  ││ Money   ││Dispense │
                          │Inserted ││Inserted ││ State   │
                          │ State   ││ State   ││         │
                          └─────────┘└─────────┘└─────────┘
```

**When to use:**
- Object behavior depends on its state
- Operations have large conditional statements based on state
- State transitions are well-defined and should be explicit

### Transaction Management Patterns

The vending machine demonstrates key transaction management principles:

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          TRANSACTION LIFECYCLE                                       │
└─────────────────────────────────────────────────────────────────────────────────────┘

    ┌──────────┐    validate    ┌──────────┐    execute    ┌──────────┐
    │  PENDING │───────────────►│ VALIDATED│──────────────►│COMPLETED │
    │          │                │          │               │          │
    └──────────┘                └──────────┘               └──────────┘
         │                           │                          │
         │ cancel                    │ validation fails         │ record
         ▼                           ▼                          ▼
    ┌──────────┐                ┌──────────┐               ┌──────────┐
    │ CANCELLED│                │ REJECTED │               │ HISTORY  │
    │ (refund) │                │  (error) │               │          │
    └──────────┘                └──────────┘               └──────────┘


  Key Principles:
  ═══════════════

  1. VALIDATE BEFORE MODIFY: Check all preconditions before any state change
  2. ATOMIC OPERATIONS: All-or-nothing execution
  3. ORDERED OPERATIONS: Charge before dispense, record before reset
  4. CLEAN CANCELLATION: Full refund, state reset
  5. AUDIT TRAIL: Record all completed transactions
```

### Facade Design Pattern

The `VendingMachine` demonstrates the Facade pattern:

```
                         ┌───────────────────┐
          Customer ─────►│  VendingMachine   │
                         │     (Facade)      │
                         └─────────┬─────────┘
                                   │
         ┌─────────────────────────┼─────────────────────────┐
         │                         │                         │
         ▼                         ▼                         ▼
  ┌──────────────┐        ┌──────────────┐        ┌──────────────┐
  │  Inventory   │        │   Payment    │        │ Transaction  │
  │   Manager    │        │  Processor   │        │   History    │
  └──────────────┘        └──────────────┘        └──────────────┘
```

**Benefits:**
- Customer interacts with one simple interface
- Internal complexity is hidden
- Subsystems can be changed independently
