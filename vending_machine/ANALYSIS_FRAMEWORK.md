# 09

## Design a Vending Machine

A Vending Machine is a self-service retail device that dispenses products after receiving payment. It's a classic OOD problem that tests understanding of **transaction management**, **state machines**, **inventory tracking**, and **payment processing**. Real vending machines must handle edge cases like insufficient funds, out-of-stock items, and transaction cancellation—all while maintaining data integrity.

This design focuses on **transaction lifecycle management**, **separation of concerns** (inventory vs payment), and the **State Pattern** for handling different machine states.

---

## Requirements Gathering

### Requirements Clarification

**Q: What are the core operations of a vending machine?**
A: Insert money → Select product → Dispense product → Return change. The machine must also support transaction cancellation at any point before dispensing.

**Q: How is the product inventory organized?**
A: Products are stored in **racks** identified by codes (e.g., "A1", "B2"). Each rack contains a single product type with a quantity count. This is how physical vending machines work—each spiral/slot holds one product type.

**Q: What payment methods should be supported?**
A: For this design, we focus on cash/coins. The machine tracks a running balance as money is inserted. The balance must be sufficient to cover the product price.

**Q: What happens if the user inserts money but doesn't complete the purchase?**
A: The user can cancel the transaction, and all inserted money is returned. The machine resets to its initial state.

**Q: What validations should occur before dispensing?**
A: Three validations:
1. Product selection is valid (rack exists)
2. Product is in stock (rack count > 0)
3. Sufficient funds (balance >= product price)

**Q: Should the machine track transaction history?**
A: Yes, for auditing and revenue tracking. Each completed transaction is recorded.

**Q: What states can the vending machine be in?**
A: The machine transitions through states:
- **No Money Inserted**: Waiting for payment
- **Money Inserted**: Waiting for product selection
- **Dispense**: Processing the transaction

### Requirements State

| Requirement | Status | Business Impact |
|-------------|--------|-----------------|
| Rack-based inventory system | ✅ Confirmed | Physical layout modeling |
| Product with code, description, price | ✅ Confirmed | Product catalog |
| Insert money (accumulating balance) | ✅ Confirmed | Payment flow |
| Select product by rack code | ✅ Confirmed | User interaction |
| Validate inventory before dispense | ✅ Confirmed | Prevents overselling |
| Validate sufficient funds | ✅ Confirmed | Prevents underpayment |
| Dispense product and return change | ✅ Confirmed | Core functionality |
| Cancel transaction and refund | ✅ Confirmed | User experience |
| Transaction history | ✅ Confirmed | Auditing |
| State pattern for machine states | ✅ Confirmed (deep dive) | Clean state management |
| Multiple payment methods | ❌ Out of scope | — |
| Remote inventory monitoring | ❌ Out of scope | — |

---

## Identify Core Objects

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          VENDING MACHINE OBJECTS                                     │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌───────────────┐    ┌────────────────────┐    ┌────────────────────┐             │
│  │VendingMachine │───►│InventoryManager    │───►│      Rack          │             │
│  │   (Facade)    │    │                    │    │                    │             │
│  │               │    │- racks (Map)       │    │- rackCode          │             │
│  │- inventory    │    │                    │    │- product           │             │
│  │- payment      │    │+getProductInRack() │    │- count             │             │
│  │- transaction  │    │+dispenseFromRack() │    └────────────────────┘             │
│  │- history      │    └────────────────────┘              │                        │
│  └───────────────┘                                        ▼                        │
│         │                                         ┌────────────────────┐            │
│         │                                         │     Product        │            │
│         ▼                                         │                    │            │
│  ┌────────────────────┐                           │- productCode       │            │
│  │ PaymentProcessor   │                           │- description       │            │
│  │                    │                           │- unitPrice         │            │
│  │- currentBalance    │                           └────────────────────┘            │
│  │                    │                                                             │
│  │+addBalance()       │    ┌────────────────────┐                                   │
│  │+charge()           │    │   Transaction      │                                   │
│  │+returnChange()     │    │                    │                                   │
│  └────────────────────┘    │- rack              │                                   │
│                            │- product           │                                   │
│                            │- totalAmount       │                                   │
│                            └────────────────────┘                                   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                      STATE PATTERN (Deep Dive)                               │   │
│  │  ┌──────────────────────┐                                                   │   │
│  │  │ VendingMachineState  │◄──────────┬──────────────┬──────────────┐         │   │
│  │  │    <<interface>>     │           │              │              │         │   │
│  │  ├──────────────────────┤           │              │              │         │   │
│  │  │+insertMoney()        │      ┌────┴────┐   ┌────┴────┐   ┌────┴────┐     │   │
│  │  │+selectProduct()      │      │NoMoney  │   │ Money   │   │Dispense │     │   │
│  │  │+dispenseProduct()    │      │Inserted │   │Inserted │   │  State  │     │   │
│  │  └──────────────────────┘      │ State   │   │ State   │   │         │     │   │
│  │                                └─────────┘   └─────────┘   └─────────┘     │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Design Class Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                              TRANSACTION FLOW                                         │
└──────────────────────────────────────────────────────────────────────────────────────┘

    User                    VendingMachine           PaymentProcessor      InventoryManager
      │                          │                         │                      │
      │  insertMoney($5)         │                         │                      │
      │─────────────────────────►│   addBalance($5)        │                      │
      │                          │────────────────────────►│                      │
      │                          │                         │                      │
      │  chooseProduct("A1")     │                         │                      │
      │─────────────────────────►│   getProductInRack("A1")│                      │
      │                          │────────────────────────────────────────────────►│
      │                          │◄────────────────────────────────────────────────│
      │                          │                         │                      │
      │  confirmTransaction()    │                         │                      │
      │─────────────────────────►│                         │                      │
      │                          │   validateTransaction() │                      │
      │                          │   ├─ product != null?   │                      │
      │                          │   ├─ count > 0?         │                      │
      │                          │   └─ balance >= price?  │                      │
      │                          │                         │                      │
      │                          │   charge($2.50)         │                      │
      │                          │────────────────────────►│                      │
      │                          │                         │                      │
      │                          │   dispenseProductFromRack()                    │
      │                          │────────────────────────────────────────────────►│
      │                          │                         │                      │
      │                          │   returnChange()        │                      │
      │                          │────────────────────────►│                      │
      │   change: $2.50          │◄────────────────────────│                      │
      │◄─────────────────────────│                         │                      │


┌──────────────────────────────────────────────────────────────────────────────────────┐
│                            STATE TRANSITIONS                                          │
└──────────────────────────────────────────────────────────────────────────────────────┘

    ┌────────────────┐    insertMoney()    ┌────────────────┐
    │  NoMoney       │────────────────────►│  MoneyInserted │
    │  Inserted      │                     │     State      │
    │    State       │◄────────────────────│                │
    └────────────────┘    cancel()         └───────┬────────┘
           ▲                                       │
           │                                       │ selectProduct()
           │                                       ▼
           │                               ┌────────────────┐
           │         dispenseComplete()    │   Dispense     │
           └───────────────────────────────│     State      │
                                           └────────────────┘
```

---

## Code

### Deep Dive Topics

#### Transaction Validation: Defense in Depth

The `validateTransaction()` method performs three critical checks:

```java
private void validateTransaction() throws InvalidTransactionException {
    // 1. Product selection validation
    if (currentTransaction.getProduct() == null) {
        throw new InvalidTransactionException("Invalid product selection");
    }
    // 2. Inventory validation
    if (currentTransaction.getRack().getProductCount() == 0) {
        throw new InvalidTransactionException("Insufficient inventory for product.");
    }
    // 3. Payment validation
    if (paymentProcessor.getCurrentBalance()
            .compareTo(currentTransaction.getProduct().getUnitPrice()) < 0) {
        throw new InvalidTransactionException("Insufficient fund");
    }
}
```

**Why this order matters:**
1. Check product first (cheapest validation)
2. Check inventory (requires rack lookup)
3. Check payment last (requires balance comparison)

**Reference:** `vendingmachine/VendingMachine.java:59-68`

---

#### State Pattern: Eliminating Conditional Complexity

The deep dive implementation uses the State Pattern to avoid scattered conditionals:

```java
// Without State Pattern (problematic)
public void selectProduct(String code) {
    if (state == WAITING_FOR_MONEY) {
        throw new Exception("Insert money first");
    } else if (state == MONEY_INSERTED) {
        // process selection
    } else if (state == DISPENSING) {
        throw new Exception("Transaction in progress");
    }
}

// With State Pattern (clean)
public interface VendingMachineState {
    void insertMoney(VendingMachine VM, double amount);
    void selectProductByCode(VendingMachine VM, String code) throws InvalidStateException;
    void dispenseProduct(VendingMachine VM) throws InvalidStateException;
}

public class NoMoneyInsertedState implements VendingMachineState {
    @Override
    public void selectProductByCode(VendingMachine VM, String code) {
        throw new InvalidStateException("Cannot select a product without inserting money.");
    }
}
```

**Benefits:**
- Each state encapsulates its own behavior
- Adding new states doesn't modify existing code
- State transitions are explicit

**Reference:** `vending_machine_deep_dive/VendingMachineState.java`, `vending_machine_deep_dive/NoMoneyInsertedState.java`

---

### Parking at Code

```
vending_machine/
├── vendingmachine/
│   ├── VendingMachine.java          # Main facade
│   ├── Product.java                 # Product entity
│   ├── Rack.java                    # Physical slot with product and count
│   ├── InventoryManager.java        # Rack management
│   ├── PaymentProcessor.java        # Balance and change handling
│   ├── Transaction.java             # Transaction record
│   ├── InvalidTransactionException.java
│   └── VendingMachineTest.java
└── vending_machine_deep_dive/
    ├── VendingMachineState.java     # State interface
    ├── NoMoneyInsertedState.java    # Initial state
    ├── MoneyInsertedState.java      # After payment
    ├── DispenseState.java           # During dispensing
    └── InvalidStateException.java
```

---

## Wrap Up

Key takeaways:

1. **Separation of Concerns**: `InventoryManager` handles stock, `PaymentProcessor` handles money—single responsibility
2. **Transaction as First-Class Object**: Captures the complete purchase context
3. **Validation Before Action**: All checks before dispensing prevents partial transactions
4. **State Pattern** (deep dive): Cleanly models machine states without conditionals
5. **Immutable Product**: Products don't change; only rack counts change

---

## Further Reading

### State Pattern

The State pattern allows an object to alter its behavior when its internal state changes.

```
┌─────────────────────┐
│  VendingMachine     │
│                     │
│  + state ───────────┼─────► VendingMachineState
│                     │              │
│  + insertMoney()    │       ┌──────┴──────┐
│  + selectProduct()  │       │             │
└─────────────────────┘  NoMoney    MoneyInserted
                         State        State
```

**When to use:**
- Object behavior depends on state
- Operations have complex conditionals based on state
- State transitions are well-defined
