# 04

## Design an ATM System

An ATM (Automated Teller Machine) is a self-service banking terminal that allows customers to perform financial transactions without the need for a human teller. ATMs are commonly found in bank branches, shopping centers, airports, and other public locations. They provide 24/7 access to basic banking services such as cash withdrawals, deposits, balance inquiries, and fund transfers.

In this design, we'll focus on the core functionalities of an ATM system including card validation, PIN verification, cash withdrawal, and deposit operations. We'll also explore how the State pattern can be used to manage the different operational states of the ATM.

---

## Requirements Gathering

Before diving into the design, let's clarify the requirements through a Q&A format, simulating an interview scenario.

### Requirements Clarification

**Q: What types of transactions should the ATM support?**
A: The ATM should support basic transactions: cash withdrawals and cash deposits. For this design, we'll focus on these core operations.

**Q: Should the ATM support multiple account types?**
A: Yes, the system should support different account types such as Checking and Savings accounts.

**Q: How should the ATM handle authentication?**
A: Users authenticate using a card (identified by card number) and a PIN. The PIN should be securely stored using hashing (MD5 in our implementation).

**Q: What happens if a user enters an incorrect PIN?**
A: The system should display an error message and allow the user to retry. For simplicity, we won't implement a lockout mechanism after multiple failed attempts.

**Q: How should the ATM handle insufficient funds during withdrawal?**
A: The ATM should check the account balance before dispensing cash. If funds are insufficient, it should display an appropriate error message and return to the transaction selection screen.

**Q: Should the ATM track transaction history?**
A: For this design, we'll focus on the core transaction operations without persistent transaction logging.

**Q: Can users perform multiple transactions in a single session?**
A: Yes, after completing a transaction, users should return to the transaction selection screen where they can perform another transaction or eject their card.

**Q: What hardware components should the ATM have?**
A: The ATM should have:
- **Input devices**: Card processor, keypad, deposit box
- **Output devices**: Display screen, cash dispenser

### Requirements State

Based on the clarifications above, here are our finalized requirements:

| Requirement | Status |
|-------------|--------|
| Support cash withdrawal transactions | ✅ Confirmed |
| Support cash deposit transactions | ✅ Confirmed |
| Support multiple account types (Checking/Savings) | ✅ Confirmed |
| Card-based authentication with PIN | ✅ Confirmed |
| Secure PIN storage using hashing | ✅ Confirmed |
| Insufficient funds handling | ✅ Confirmed |
| Multiple transactions per session | ✅ Confirmed |
| Hardware abstraction (simulated components) | ✅ Confirmed |
| Transaction history/logging | ❌ Out of scope |
| Account lockout after failed attempts | ❌ Out of scope |
| Inter-bank transfers | ❌ Out of scope |

---

## Identify Core Objects

From the requirements, we can identify the following core objects:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              ATM SYSTEM OBJECTS                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │  ATMMachine │    │    Bank     │    │   Account   │    │ Transaction │  │
│  │             │    │             │    │             │    │             │  │
│  │ - state     │    │ - accounts  │    │ - balance   │    │ - account   │  │
│  │ - hardware  │    │ - cards     │    │ - cardNum   │    │ - amount    │  │
│  └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘  │
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │  ATMState   │    │CardProcessor│    │   Keypad    │    │   Display   │  │
│  │             │    │             │    │             │    │             │  │
│  │ - idle      │    │ - cardNum   │    │ - PIN entry │    │ - messages  │  │
│  │ - pin entry │    │ - insert    │    │ - amount    │    │ - show()    │  │
│  │ - transact  │    │ - eject     │    │ - select    │    │             │  │
│  └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘  │
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐                                        │
│  │CashDispenser│    │ DepositBox  │                                        │
│  │             │    │             │                                        │
│  │ - dispense  │    │ - accept    │                                        │
│  └─────────────┘    └─────────────┘                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Core objects identified:**

1. **ATMMachine**: The main controller that orchestrates all ATM operations and manages state transitions
2. **Bank**: Manages accounts and provides banking operations (validation, PIN check, fund operations)
3. **Account**: Represents a bank account with balance, card details, and PIN security
4. **Transaction**: Interface for different transaction types (Deposit, Withdraw)
5. **ATMState**: Abstract base for state pattern implementation (Idle, PinEntry, TransactionSelection, etc.)
6. **Hardware Components**: CardProcessor, Keypad, Display, CashDispenser, DepositBox

---

## Design Class Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                                  CLASS DIAGRAM                                        │
└──────────────────────────────────────────────────────────────────────────────────────┘

                                    ┌─────────────────┐
                                    │   ATMMachine    │
                                    ├─────────────────┤
                                    │ - state         │
                                    │ - cardProcessor │
                                    │ - depositBox    │
                                    │ - cashDispenser │
                                    │ - keypad        │
                                    │ - display       │
                                    │ - bank          │
                                    ├─────────────────┤
                                    │ + insertCard()  │
                                    │ + ejectCard()   │
                                    │ + enterPin()    │
                                    │ + withdraw()    │
                                    │ + deposit()     │
                                    │ + enterAmount() │
                                    │ + transition()  │
                                    └────────┬────────┘
                                             │
                 ┌───────────────────────────┼───────────────────────────┐
                 │                           │                           │
                 ▼                           ▼                           ▼
       ┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
       │    ATMState     │         │  BankInterface  │         │    Hardware     │
       │   (abstract)    │         │   <<interface>> │         │   Components    │
       ├─────────────────┤         ├─────────────────┤         └────────┬────────┘
       │ + processCard() │         │ + validateCard()│                  │
       │ + processPin()  │         │ + checkPin()    │    ┌─────────────┼─────────────┐
       │ + processEject()│         │ + getAccount()  │    │             │             │
       │ + processAmount │         │ + withdrawFunds │    ▼             ▼             ▼
       └────────┬────────┘         └────────┬────────┘  Input       Output      Interfaces
                │                           │          Devices     Devices
    ┌───────────┼───────────┐               │
    │     │     │     │     │               ▼
    ▼     ▼     ▼     ▼     ▼      ┌─────────────────┐
┌──────┐┌──────┐┌──────┐┌──────┐   │      Bank       │
│Idle  ││PinEn-││Trans-││Withdr│   ├─────────────────┤
│State ││try   ││action││aw/   │   │ - accounts      │
│      ││State ││Select││Deposit   │ - accountByCard │
└──────┘└──────┘└──────┘└──────┘   ├─────────────────┤
                                   │ + addAccount()  │
                                   │ + validateCard()│
                                   │ + checkPin()    │
                                   │ + withdrawFunds │
                                   └────────┬────────┘
                                            │
                                            ▼
                                   ┌─────────────────┐
                                   │     Account     │
                                   ├─────────────────┤
                                   │ - balance       │
                                   │ - accountNumber │
                                   │ - cardNumber    │
                                   │ - cardPinHash   │
                                   │ - accountType   │
                                   ├─────────────────┤
                                   │ + validatePin() │
                                   │ + getBalance()  │
                                   │ + updateBalance │
                                   └─────────────────┘

┌──────────────────────────────────────────────────────────────────────────────────────┐
│                               STATE TRANSITIONS                                       │
└──────────────────────────────────────────────────────────────────────────────────────┘

    ┌──────────┐   insert card    ┌──────────┐   correct PIN   ┌────────────┐
    │   Idle   │ ───────────────► │ PinEntry │ ──────────────► │Transaction │
    │  State   │                  │  State   │                 │ Selection  │
    └──────────┘                  └──────────┘                 └────────────┘
         ▲                             │                        │         │
         │                             │                        │         │
         │    eject card / cancel      │     eject card         │         │
         └─────────────────────────────┘                        │         │
         │                                                      │         │
         │              ┌───────────────────────────────────────┘         │
         │              │ select withdraw                   select deposit│
         │              ▼                                                 ▼
         │     ┌────────────────┐                            ┌────────────────┐
         │     │   Withdraw     │                            │    Deposit     │
         │     │  AmountEntry   │                            │   Collection   │
         │     └────────────────┘                            └────────────────┘
         │              │                                             │
         │              │  enter amount                 collect deposit│
         │              │  (success/fail)                (success)    │
         │              └──────────────┐    ┌─────────────────────────┘
         │                             │    │
         │                             ▼    ▼
         │                       ┌────────────┐
         │     eject card        │Transaction │
         └─────────────────────  │ Selection  │
                                 └────────────┘
```

---

## Code

The ATM system is implemented using the **State Pattern** to manage different operational states and clean separation of concerns between hardware, banking logic, and state management.

### Deep Dive Topics

#### What happens if a user tries to insert a card when already in a transaction?

The State pattern elegantly handles this. Each state class only responds to relevant actions. If a card insertion is attempted while in `TransactionSelectionState`, the default `ATMState` implementation displays "Invalid action, please try again." This prevents invalid state transitions.

**Reference:** `atm/states/ATMState.java:14-21`

#### How does the PIN validation work securely?

PINs are never stored in plain text. When an account is created, the PIN is immediately hashed using MD5:

```java
// Account.java - PIN is hashed during account creation
public Account(final String accountNumber, final AccountType type,
               final String cardNumber, final String pin) {
    this.cardPinHash = calculateMd5(pin);  // PIN is hashed for security
    // ...
}

// PIN validation compares hashes, not plain text
public boolean validatePin(String pinNumber) {
    byte[] entryPinHash = calculateMd5(pinNumber);
    return Arrays.equals(cardPinHash, entryPinHash);
}
```

**Reference:** `atm/bank/Account.java:13-42`

---

### Parking at Code

The following describes the code structure:

```
atm/
├── ATMMachine.java              # Main ATM controller, orchestrates all operations
├── ATMTest.java                 # Comprehensive test suite
├── bank/                        # Banking domain
│   ├── Account.java             # Account entity with secure PIN handling
│   ├── Bank.java                # Bank implementation
│   ├── BankInterface.java       # Contract for bank operations
│   ├── Transaction.java         # Transaction interface
│   ├── DepositTransaction.java  # Deposit operation
│   ├── WithdrawTransaction.java # Withdrawal operation
│   └── enums/
│       ├── AccountType.java     # CHECKING, SAVING
│       └── TransactionType.java # WITHDRAW, DEPOSIT
├── hardware/                    # Hardware abstractions
│   ├── input/                   # Input devices
│   │   ├── CardProcessor.java   # Card reading interface
│   │   ├── DepositBox.java      # Cash deposit interface
│   │   ├── Keypad.java          # User input interface
│   │   └── Simulated*.java      # Test implementations
│   └── output/                  # Output devices
│       ├── CashDispenser.java   # Cash dispensing interface
│       ├── Display.java         # Screen interface
│       └── Simulated*.java      # Test implementations
└── states/                      # State pattern implementation
    ├── ATMState.java            # Base state with default behaviors
    ├── IdleState.java           # Waiting for card
    ├── PinEntryState.java       # PIN verification
    ├── TransactionSelectionState.java  # Choose operation
    ├── WithdrawAmountEntryState.java   # Withdrawal flow
    └── DepositCollectionState.java     # Deposit flow
```

---

### ATMMachine

The `ATMMachine` class is the central controller that coordinates all ATM operations. It follows the **Facade Pattern** to provide a simplified interface to the complex subsystem of hardware components and banking operations.

```java
public class ATMMachine {
    private ATMState state;
    private final CardProcessor cardProcessor;
    private final DepositBox depositBox;
    private final CashDispenser cashDispenser;
    private final Keypad keypad;
    private final Display display;
    private final Bank bank;
}
```

**Key design decisions:**

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| State management | Delegation to `ATMState` | Enables clean state transitions without conditional logic |
| Hardware coupling | Interface-based injection | Allows swapping real hardware with simulated components for testing |
| Bank operations | Through `BankInterface` | Decouples ATM from specific bank implementation |

**Reference:** `atm/ATMMachine.java:16-113`

---

### ATMState (State Pattern)

The `ATMState` class defines the abstract base for all ATM states. Each concrete state handles specific user actions and transitions to appropriate next states.

```java
public class ATMState {
    // Default implementations - display error for invalid actions
    public void processCardInsertion(ATMMachine atmMachine, String cardNumber) {
        atmMachine.getDisplay().showMessage("Invalid action, please try again.");
    }

    public void processCardEjection(ATMMachine atmMachine) { /* default */ }
    public void processPinEntry(ATMMachine atmMachine, String pin) { /* default */ }
    public void processWithdrawalRequest(ATMMachine atmMachine) { /* default */ }
    public void processDepositRequest(ATMMachine atmMachine) { /* default */ }
    public void processAmountEntry(ATMMachine atmMachine, BigDecimal amount) { /* default */ }
    public void processDepositCollection(ATMMachine atmMachine, BigDecimal amount) { /* default */ }
}
```

**State implementations:**

| State | Valid Actions | Transitions To |
|-------|---------------|----------------|
| `IdleState` | Insert card | `PinEntryState` (if card valid) |
| `PinEntryState` | Enter PIN, Eject card | `TransactionSelectionState` (PIN correct), `IdleState` (eject) |
| `TransactionSelectionState` | Withdraw, Deposit, Eject | Respective states |
| `WithdrawAmountEntryState` | Enter amount, Eject | `TransactionSelectionState` (after operation) |
| `DepositCollectionState` | Collect deposit, Eject | `TransactionSelectionState` (after operation) |

**Reference:** `atm/states/ATMState.java:1-52`

---

### Bank and BankInterface

The `Bank` class implements the `BankInterface` and manages all account-related operations. It uses two HashMaps for efficient lookups:

```java
public class Bank implements BankInterface {
    private final Map<String, Account> accounts = new HashMap<>();      // by account number
    private final Map<String, Account> accountByCard = new HashMap<>(); // by card number

    public boolean validateCard(String cardNumber);
    public boolean checkPin(String cardNumber, String pinNumber);
    public boolean withdrawFunds(Account account, BigDecimal amount);
}
```

**Design consideration:** The dual-map approach allows O(1) lookups both by account number (for internal operations) and by card number (for ATM authentication flow).

**Reference:** `atm/bank/Bank.java:1-62`

---

### Account

The `Account` class represents a bank account with secure PIN handling:

```java
public class Account {
    private BigDecimal balance;
    private final String accountNumber;
    private final String cardNumber;
    private final byte[] cardPinHash;      // Stored as MD5 hash, never plain text
    private final AccountType accountType;
}
```

**Security note:** The PIN is immediately hashed upon account creation and never stored in plain text. Validation is performed by comparing hashes.

**Reference:** `atm/bank/Account.java:1-70`

---

### Transaction Interface and Implementations

Transactions follow the **Strategy Pattern**, with a common interface and specific implementations:

```java
public interface Transaction {
    TransactionType getType();
    boolean validateTransaction();
    void executeTransaction();
}
```

| Transaction Type | Validation | Execution |
|-----------------|------------|-----------|
| `DepositTransaction` | Always valid | Add amount to balance |
| `WithdrawTransaction` | Check sufficient funds | Subtract amount from balance |

**Reference:** `atm/bank/Transaction.java:1-12`, `atm/bank/DepositTransaction.java:1-34`, `atm/bank/WithdrawTransaction.java:1-39`

---

### Hardware Interfaces

Hardware components are abstracted through interfaces, enabling testability:

**Input Devices:**

| Interface | Purpose | Key Methods |
|-----------|---------|-------------|
| `CardProcessor` | Card insertion/ejection | `handleCardInsertion()`, `handleCardEjection()`, `getCardNumber()` |
| `Keypad` | User input | `handlePinEntry()`, `handleAmountEntry()`, `handleSelectTransaction()` |
| `DepositBox` | Cash deposits | `acceptDeposit()` |

**Output Devices:**

| Interface | Purpose | Key Methods |
|-----------|---------|-------------|
| `Display` | Screen messages | `showMessage()`, `getDisplayedMessage()` |
| `CashDispenser` | Cash withdrawal | `dispenseCash()` |

**Reference:** `atm/hardware/input/`, `atm/hardware/output/`

---

### Adding a New Transaction Type

To add a new transaction type (e.g., Balance Inquiry or Transfer), follow these steps:

1. **Add enum value** in `TransactionType.java`:
```java
public enum TransactionType {
    WITHDRAW,
    DEPOSIT,
    BALANCE_INQUIRY  // New type
}
```

2. **Create transaction class** implementing `Transaction`:
```java
public class BalanceInquiryTransaction implements Transaction {
    @Override
    public TransactionType getType() { return TransactionType.BALANCE_INQUIRY; }
    @Override
    public boolean validateTransaction() { return true; }
    @Override
    public void executeTransaction() { /* Display balance */ }
}
```

3. **Add new state** for the transaction flow (if needed)

4. **Update `TransactionSelectionState`** to handle the new request:
```java
@Override
public void processBalanceInquiry(ATMMachine atmMachine) {
    atmMachine.getDisplay().showMessage("Your balance is: " + balance);
    // Either stay in TransactionSelectionState or create BalanceDisplayState
}
```

---

### Faster State Management

The State pattern in this design provides several benefits:

1. **Eliminates complex conditionals**: Instead of checking current state in every method, behavior is encapsulated in state classes

2. **Single Responsibility**: Each state class handles only its specific concerns

3. **Open/Closed Principle**: New states can be added without modifying existing code

**State transition example:**

```java
// In IdleState - handles card insertion
@Override
public void processCardInsertion(ATMMachine atmMachine, String cardNumber) {
    if (atmMachine.getBankInterface().validateCard(cardNumber)) {
        atmMachine.getDisplay().showMessage("Please enter your PIN");
        atmMachine.transitionToState(new PinEntryState());  // Clean transition
    } else {
        atmMachine.getDisplay().showMessage("Invalid card. Please try again.");
        // Stays in IdleState - no transition needed
    }
}
```

**Anti-pattern avoided:** The design avoids the "state flag" anti-pattern where a single class checks state variables to determine behavior:

```java
// BAD: State flag anti-pattern (NOT used in this design)
public void processAction() {
    if (state == STATE_IDLE) { /* ... */ }
    else if (state == STATE_PIN_ENTRY) { /* ... */ }
    else if (state == STATE_TRANSACTION) { /* ... */ }
    // ... many more conditionals
}
```

---

## Wrap Up

In this design, we created an ATM system that handles core banking operations through a well-structured object-oriented approach. Key takeaways:

1. **State Pattern** for managing ATM operational states, eliminating complex conditionals and enabling clean state transitions

2. **Strategy Pattern** for transactions, allowing different transaction types to share a common interface while implementing specific behaviors

3. **Facade Pattern** in `ATMMachine`, providing a simplified interface to coordinate hardware components and banking operations

4. **Interface-based design** for hardware components, enabling dependency injection and testability with simulated implementations

5. **Security considerations** with hashed PIN storage and validation

---

## Further Reading: Strategy and State Design Patterns

### State Design Pattern

The State pattern allows an object to alter its behavior when its internal state changes. The object will appear to change its class.

**When to use:**
- Object behavior depends on its state and must change at runtime
- Operations have large conditional statements based on state
- State transitions need to be explicit and well-defined

**Structure in this design:**

```
┌─────────────┐         ┌─────────────────┐
│ ATMMachine  │────────►│    ATMState     │
│             │         │   (abstract)    │
│ + state     │         ├─────────────────┤
│             │         │ + processCard() │
└─────────────┘         │ + processPin()  │
                        │ + processEject()│
                        └────────┬────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            │                    │                    │
            ▼                    ▼                    ▼
     ┌───────────┐        ┌───────────┐        ┌───────────┐
     │ IdleState │        │ PinEntry  │        │Transaction│
     │           │        │   State   │        │ Selection │
     └───────────┘        └───────────┘        └───────────┘
```

### Strategy Design Pattern

The Strategy pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

**Application in this design:**

```
┌─────────────────┐
│   Transaction   │
│  <<interface>>  │
├─────────────────┤
│ + getType()     │
│ + validate()    │
│ + execute()     │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌────────┐ ┌────────┐
│Deposit │ │Withdraw│
│  Txn   │ │  Txn   │
└────────┘ └────────┘
```

**Key differences:**

| Aspect | State Pattern | Strategy Pattern |
|--------|--------------|------------------|
| Intent | Change behavior based on internal state | Choose algorithm at runtime |
| State awareness | States know about each other (transitions) | Strategies are independent |
| Trigger | Internal state change | Client selection |
| In this design | ATM operational flow | Transaction types |

### Facade Design Pattern

The Facade pattern provides a unified interface to a set of interfaces in a subsystem. It defines a higher-level interface that makes the subsystem easier to use.

**Application in `ATMMachine`:**

```
                    ┌─────────────────┐
                    │   ATMMachine    │
     Client ───────►│    (Facade)     │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │         │         │         │         │
         ▼         ▼         ▼         ▼         ▼
    ┌────────┐┌────────┐┌────────┐┌────────┐┌────────┐
    │ Card   ││ Keypad ││Display ││ Cash   ││ Bank   │
    │Processor│        │        ││Dispenser│        │
    └────────┘└────────┘└────────┘└────────┘└────────┘
```

The `ATMMachine` facade:
- Hides complexity of coordinating multiple hardware components
- Provides simple methods like `insertCard()`, `enterPin()`, `withdrawRequest()`
- Allows hardware components to be swapped without affecting client code
