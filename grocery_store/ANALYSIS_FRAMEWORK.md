# 05

## Design a Grocery Store System

A Grocery Store System is a software application that manages the day-to-day operations of a grocery store, including product catalog management, inventory tracking, order processing, discount campaigns, and receipt generation. Modern grocery systems need to handle various discount strategies, support multiple product categories, and provide accurate billing with applied promotions.

In this design, we'll focus on core functionalities including catalog management, inventory tracking, flexible discount systems using Strategy and Composite patterns, and checkout processing with receipt generation.

---

## Requirements Gathering

Before diving into the design, let's clarify the requirements through a Q&A format, simulating an interview scenario.

### Requirements Clarification

**Q: What are the core operations the grocery store system should support?**
A: The system should support catalog management (add/update/remove items), inventory tracking, order processing, discount application, and receipt generation.

**Q: How should products be identified and categorized?**
A: Each product should have a unique barcode identifier, a name, a category (e.g., Fruit, Candy, Pantry), and a price. Categories help in applying category-wide discounts.

**Q: What types of discounts should the system support?**
A: The system should support multiple discount types:
- Percentage-based discounts (e.g., 50% off)
- Fixed amount discounts (e.g., $5 off)
- Discounts should be stackable for advanced promotions

**Q: How should the system determine which items get discounts?**
A: Discounts should be applied based on flexible criteria:
- Category-based (all items in a category)
- Item-based (specific items by barcode)
- Composite criteria (combination of multiple conditions)

**Q: What happens when multiple discounts apply to the same item?**
A: The system should automatically select the better discount for the customer (higher discount amount).

**Q: How should monetary calculations be handled?**
A: All monetary values should use `BigDecimal` to ensure precision and avoid floating-point errors.

**Q: Should the system track inventory?**
A: Yes, the system should track stock levels and allow adding/reducing inventory quantities.

**Q: What information should the receipt include?**
A: The receipt should show all items with quantities, original prices, applied discounts, subtotal, and final total.

### Requirements State

Based on the clarifications above, here are our finalized requirements:

| Requirement | Status |
|-------------|--------|
| Product catalog with barcode, name, category, price | ✅ Confirmed |
| Inventory tracking (add/reduce stock) | ✅ Confirmed |
| Percentage-based discount strategy | ✅ Confirmed |
| Fixed amount discount strategy | ✅ Confirmed |
| Category-based discount criteria | ✅ Confirmed |
| Item-based discount criteria | ✅ Confirmed |
| Composite criteria for complex rules | ✅ Confirmed |
| Automatic best discount selection | ✅ Confirmed |
| Receipt generation with discount details | ✅ Confirmed |
| BigDecimal for monetary precision | ✅ Confirmed |
| Customer loyalty programs | ❌ Out of scope |
| Payment processing (card/cash handling) | ❌ Out of scope |
| Multi-store inventory sync | ❌ Out of scope |

---

## Identify Core Objects

From the requirements, we can identify the following core objects:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         GROCERY STORE SYSTEM OBJECTS                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │GroceryStoreSystem│  │     Catalog     │  │    Inventory    │             │
│  │                 │  │                 │  │                 │             │
│  │ - catalog       │  │ - items (map)   │  │ - stock (map)   │             │
│  │ - inventory     │  │ + updateItem()  │  │ + addStock()    │             │
│  │ - discounts     │  │ + removeItem()  │  │ + reduceStock() │             │
│  │ - checkout      │  │ + getItem()     │  │ + getStock()    │             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │      Item       │  │    OrderItem    │  │      Order      │             │
│  │                 │  │                 │  │                 │             │
│  │ - name          │  │ - item          │  │ - orderId       │             │
│  │ - barcode       │  │ - quantity      │  │ - items         │             │
│  │ - category      │  │ + calcPrice()   │  │ - discounts     │             │
│  │ - price         │  │ + calcWithDisc()│  │ + calcTotal()   │             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │    Checkout     │  │DiscountCampaign │  │     Receipt     │             │
│  │                 │  │                 │  │                 │             │
│  │ - currentOrder  │  │ - criteria      │  │ - order         │             │
│  │ - discounts     │  │ - strategy      │  │ - issueDate     │             │
│  │ + addItem()     │  │ + isApplicable()│  │ + printReceipt()│             │
│  │ + processPayment│  │ + calcDiscount()│  │                 │             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐                                  │
│  │DiscountCriteria │  │DiscountStrategy │                                  │
│  │  <<interface>>  │  │  <<interface>>  │                                  │
│  │                 │  │                 │                                  │
│  │ + isApplicable()│  │ + calcDiscount()│                                  │
│  └─────────────────┘  └─────────────────┘                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Core objects identified:**

1. **GroceryStoreSystem**: Main facade coordinating all store operations
2. **Catalog**: Manages the product catalog with barcode-to-item mapping
3. **Inventory**: Tracks stock quantities for each product
4. **Item**: Represents a product with name, barcode, category, and price
5. **Order**: Contains order items and tracks applied discounts
6. **OrderItem**: Represents an item with quantity in an order
7. **Checkout**: Handles the checkout process and discount application
8. **DiscountCampaign**: Combines criteria and strategy for discount logic
9. **DiscountCriteria**: Interface for determining discount eligibility
10. **DiscountCalculationStrategy**: Interface for calculating discounted prices
11. **Receipt**: Generates formatted receipt output

---

## Design Class Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                                  CLASS DIAGRAM                                        │
└──────────────────────────────────────────────────────────────────────────────────────┘

                              ┌─────────────────────┐
                              │  GroceryStoreSystem │
                              │      (Facade)       │
                              ├─────────────────────┤
                              │ - catalog           │
                              │ - inventory         │
                              │ - activeDiscounts   │
                              │ - checkout          │
                              ├─────────────────────┤
                              │ + addOrUpdateItem() │
                              │ + updateInventory() │
                              │ + addDiscountCampaign()│
                              │ + getCheckout()     │
                              └──────────┬──────────┘
                                         │
         ┌───────────────┬───────────────┼───────────────┬───────────────┐
         │               │               │               │               │
         ▼               ▼               ▼               ▼               ▼
   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────┐  ┌──────────┐
   │ Catalog  │   │Inventory │   │ Checkout │   │DiscountCampaign│ │  Item   │
   ├──────────┤   ├──────────┤   ├──────────┤   ├──────────────┤  ├──────────┤
   │- items   │   │- stock   │   │- order   │   │- criteria    │  │- name    │
   │          │   │          │   │- discounts│  │- strategy    │  │- barcode │
   ├──────────┤   ├──────────┤   ├──────────┤   ├──────────────┤  │- category│
   │+update() │   │+addStock()│  │+addItem()│   │+isApplicable()│ │- price   │
   │+remove() │   │+reduce() │   │+process()│   │+calcDiscount()│ └──────────┘
   │+get()    │   │+get()    │   │+getReceipt│  └──────┬───────┘
   └──────────┘   └──────────┘   └────┬─────┘          │
                                      │                │
                                      ▼          ┌─────┴─────┐
                                ┌──────────┐     │           │
                                │  Order   │     ▼           ▼
                                ├──────────┤  ┌────────────────┐  ┌────────────────────┐
                                │- orderId │  │DiscountCriteria│  │DiscountCalculation │
                                │- items   │  │ <<interface>>  │  │    Strategy        │
                                │- discounts│ ├────────────────┤  │   <<interface>>    │
                                ├──────────┤  │+isApplicable() │  ├────────────────────┤
                                │+addItem()│  └───────┬────────┘  │+calcDiscountedPrice│
                                │+calcTotal│          │           └─────────┬──────────┘
                                └────┬─────┘          │                     │
                                     │       ┌────────┼────────┐    ┌───────┼───────┐
                                     ▼       │        │        │    │       │       │
                              ┌──────────┐   ▼        ▼        ▼    ▼       ▼       ▼
                              │OrderItem │ ┌────┐  ┌────┐  ┌────┐ ┌────┐ ┌────┐ ┌────┐
                              ├──────────┤ │Cat-│  │Item│  │Comp│ │ %  │ │Amt │ │Deco│
                              │- item    │ │egory│ │Based│ │osite│ │Based│ │Based│ │rator│
                              │- quantity│ │Based│ │Crit│  │Crit│ │Strat│ │Strat│ │    │
                              ├──────────┤ └────┘  └────┘  └────┘ └────┘ └────┘ └────┘
                              │+calcPrice│
                              └──────────┘


┌──────────────────────────────────────────────────────────────────────────────────────┐
│                              DISCOUNT SYSTEM DESIGN                                   │
└──────────────────────────────────────────────────────────────────────────────────────┘

  CRITERIA (Who gets the discount?)          STRATEGY (How much discount?)
  ─────────────────────────────────          ─────────────────────────────

  ┌───────────────────┐                      ┌─────────────────────────┐
  │  DiscountCriteria │                      │DiscountCalculationStrategy│
  │   <<interface>>   │                      │      <<interface>>      │
  ├───────────────────┤                      ├─────────────────────────┤
  │ +isApplicable()   │                      │ +calcDiscountedPrice()  │
  └─────────┬─────────┘                      └───────────┬─────────────┘
            │                                            │
   ┌────────┼────────┐                       ┌───────────┼───────────┐
   │        │        │                       │           │           │
   ▼        ▼        ▼                       ▼           ▼           ▼
┌──────┐ ┌──────┐ ┌──────┐              ┌──────┐    ┌──────┐    ┌──────┐
│Categ-│ │Item  │ │Compo-│              │ % -  │    │Amount│    │Decor-│
│ory   │ │Based │ │site  │              │Based │    │Based │    │ator  │
│Based │ │      │ │      │              │      │    │      │    │      │
└──────┘ └──────┘ └──────┘              └──────┘    └──────┘    └──────┘
   │        │        │                      │           │           │
   │        │        │                      │           │           │
"Fruit" "barcode" combines              50% off    $5 off     stacks
category  matches   multiple                                discounts
```

---

## Code

The Grocery Store system is implemented using multiple design patterns including **Strategy Pattern** for discount calculations, **Composite Pattern** for combining criteria, and **Decorator Pattern** for stacking discounts.

### Deep Dive Topics

#### How does the system handle multiple discounts on the same item?

When adding an item to an order, the `Checkout` class iterates through all active discounts. If multiple discounts apply to the same item, it compares the discounted prices and keeps the one that gives the customer a better deal (lower final price):

```java
// Checkout.java - Best discount selection logic
for (DiscountCampaign newDiscount : activeDiscounts) {
    if (newDiscount.isApplicable(item)) {
        if (currentOrder.getAppliedDiscounts().containsKey(orderItem)) {
            DiscountCampaign existingDiscount = currentOrder.getAppliedDiscounts().get(orderItem);
            // Compare: keep discount that results in lower price
            if (orderItem.calculatePriceWithDiscount(newDiscount)
                    .compareTo(orderItem.calculatePriceWithDiscount(existingDiscount)) > 0) {
                currentOrder.applyDiscount(orderItem, newDiscount);
            }
        } else {
            currentOrder.applyDiscount(orderItem, newDiscount);
        }
    }
}
```

**Reference:** `grocery/Checkout.java:36-49`

#### How do the Criteria and Strategy patterns work together?

The `DiscountCampaign` class combines both patterns:
- **Criteria** determines *eligibility* (which items qualify)
- **Strategy** determines *calculation* (how much discount)

```java
public class DiscountCampaign {
    private final DiscountCriteria criteria;           // WHO gets the discount
    private final DiscountCalculationStrategy calculationStrategy;  // HOW MUCH

    public boolean isApplicable(Item item) {
        return criteria.isApplicable(item);  // Delegates to criteria
    }

    public BigDecimal calculateDiscount(OrderItem item) {
        return calculationStrategy.calculateDiscountedPrice(item.calculatePrice());
    }
}
```

**Reference:** `grocery/discount/DiscountCampaign.java:10-49`

---

### Parking at Code

The following describes the code structure:

```
grocery_store/
├── grocery/
│   ├── GroceryStoreSystem.java      # Main facade coordinating operations
│   ├── GroceryStoreSystemTest.java  # Comprehensive test suite
│   ├── Catalog.java                 # Product catalog management
│   ├── Inventory.java               # Stock tracking
│   ├── Item.java                    # Product entity
│   ├── Order.java                   # Customer order with discounts
│   ├── OrderItem.java               # Line item in an order
│   ├── Checkout.java                # Checkout process and discount logic
│   ├── Receipt.java                 # Receipt generation
│   ├── ItemBasedCriteria.java       # Criteria for specific items
│   └── discount/
│       ├── DiscountCampaign.java    # Combines criteria + strategy
│       ├── criteria/
│       │   ├── DiscountCriteria.java      # Interface for eligibility
│       │   └── CategoryBasedCriteria.java # Category-based eligibility
│       └── strategy/
│           ├── DiscountCalculationStrategy.java  # Interface for calculation
│           ├── PercentageBasedStrategy.java      # Percentage discount
│           └── AmountBasedStrategy.java          # Fixed amount discount
└── grocery_store_deep_dive/
    ├── CompositeCriteria.java        # Combines multiple criteria (AND)
    ├── FixedDiscountDecorator.java   # Stacks fixed discount on strategy
    └── PercentageDiscountDecorator.java  # Stacks percentage on strategy
```

---

### GroceryStoreSystem

The `GroceryStoreSystem` class serves as the main **Facade** for the grocery store operations. It coordinates between the catalog, inventory, discount campaigns, and checkout system.

```java
public class GroceryStoreSystem {
    private final Catalog catalog;
    private final Inventory inventory;
    private List<DiscountCampaign> activeDiscounts = new ArrayList<>();
    private final Checkout checkout;
}
```

**Key responsibilities:**

| Method | Purpose |
|--------|---------|
| `addOrUpdateItem(Item)` | Add or update product in catalog |
| `updateInventory(barcode, count)` | Adjust stock levels |
| `addDiscountCampaign(DiscountCampaign)` | Register new promotion |
| `getCheckout()` | Get checkout system for order processing |
| `getItemByBarcode(barcode)` | Retrieve item from catalog |
| `removeItem(barcode)` | Remove item from catalog |

**Reference:** `grocery/GroceryStoreSystem.java:1-66`

---

### Catalog and Inventory

The **Catalog** and **Inventory** classes manage product information and stock levels respectively, using HashMap for O(1) lookups by barcode.

```java
// Catalog - Product information management
public class Catalog {
    private final Map<String, Item> items = new HashMap<>();

    public void updateItem(Item item) { items.put(item.getBarcode(), item); }
    public void removeItem(String barcode) { items.remove(barcode); }
    public Item getItem(String barcode) { return items.get(barcode); }
}

// Inventory - Stock quantity management
public class Inventory {
    private final Map<String, Integer> stock = new HashMap<>();

    public void addStock(String barcode, int count) {
        stock.put(barcode, stock.getOrDefault(barcode, 0) + count);
    }
    public void reduceStock(String barcode, int count) {
        stock.put(barcode, stock.getOrDefault(barcode, 0) - count);
    }
}
```

**Design note:** Separating Catalog and Inventory follows the **Single Responsibility Principle**—catalog handles product metadata, inventory handles quantities.

**Reference:** `grocery/Catalog.java:1-24`, `grocery/Inventory.java:1-24`

---

### Item and OrderItem

**Item** represents a product in the catalog, while **OrderItem** represents an item with quantity in a specific order.

```java
// Item - Immutable product entity
public class Item {
    private final String name;
    private final String barcode;
    private final String category;
    private final BigDecimal price;
}

// OrderItem - Item with quantity, calculates prices
public class OrderItem {
    private final Item item;
    private final int quantity;

    public BigDecimal calculatePrice() {
        return item.getPrice().multiply(BigDecimal.valueOf(quantity));
    }

    public BigDecimal calculatePriceWithDiscount(DiscountCampaign discount) {
        return discount.calculateDiscount(this);
    }
}
```

**Reference:** `grocery/Item.java:1-42`, `grocery/OrderItem.java:1-36`

---

### Order

The **Order** class represents a customer's order, tracking items and their applied discounts.

```java
public class Order {
    private final String orderId;
    private final List<OrderItem> items = new ArrayList<>();
    private final Map<OrderItem, DiscountCampaign> appliedDiscounts = new HashMap<>();
    private BigDecimal paymentAmount = BigDecimal.ZERO;
}
```

**Key calculations:**

| Method | Description |
|--------|-------------|
| `calculateSubtotal()` | Sum of all items without discounts |
| `calculateTotal()` | Sum with applied discounts |
| `calculateChange()` | Payment amount minus total |

**Reference:** `grocery/Order.java:1-79`

---

### Checkout

The **Checkout** class manages the checkout process, including discount application logic.

```java
public class Checkout {
    private Order currentOrder;
    private final List<DiscountCampaign> activeDiscounts;

    public void addItemToOrder(Item item, int quantity) {
        OrderItem orderItem = new OrderItem(item, quantity);
        currentOrder.addItem(orderItem);

        // Apply best applicable discount
        for (DiscountCampaign discount : activeDiscounts) {
            if (discount.isApplicable(item)) {
                // Compare and keep better discount if one already exists
                // ...
            }
        }
    }
}
```

**Key design decision:** The checkout automatically applies the best available discount for each item, ensuring customers always get the best deal without manual intervention.

**Reference:** `grocery/Checkout.java:1-61`

---

### DiscountCampaign

The **DiscountCampaign** class combines the **Criteria** (who qualifies) and **Strategy** (how much discount) patterns.

```java
public class DiscountCampaign {
    private final String discountId;
    private final String name;
    private final DiscountCriteria criteria;              // Eligibility check
    private final DiscountCalculationStrategy calculationStrategy;  // Calculation

    public boolean isApplicable(Item item) {
        return criteria.isApplicable(item);
    }

    public BigDecimal calculateDiscount(OrderItem item) {
        return calculationStrategy.calculateDiscountedPrice(item.calculatePrice());
    }
}
```

**Usage example:**
```java
// Create a "50% off all fruits" campaign
DiscountCampaign fruitSale = new DiscountCampaign(
    "1",
    "Fruit Half Price",
    new CategoryBasedCriteria("Fruit"),      // Criteria: fruit items only
    new PercentageBasedStrategy(new BigDecimal("50"))  // Strategy: 50% off
);
```

**Reference:** `grocery/discount/DiscountCampaign.java:1-49`

---

### DiscountCriteria Implementations

The **Strategy Pattern** is used for discount criteria, allowing flexible eligibility rules.

| Criteria | Description | Example |
|----------|-------------|---------|
| `CategoryBasedCriteria` | Matches items by category | All "Fruit" items |
| `ItemBasedCriteria` | Matches specific item by barcode | Item with barcode "123" |
| `CompositeCriteria` | Combines multiple criteria (AND logic) | "Fruit" AND "Organic" |

```java
// CategoryBasedCriteria - Matches by category
public class CategoryBasedCriteria implements DiscountCriteria {
    private final String category;

    @Override
    public boolean isApplicable(Item item) {
        return item.getCategory().equals(category);
    }
}

// CompositeCriteria - Combines multiple criteria (AND)
public class CompositeCriteria implements DiscountCriteria {
    private final List<DiscountCriteria> criteriaList;

    @Override
    public boolean isApplicable(Item item) {
        return criteriaList.stream().allMatch(c -> c.isApplicable(item));
    }
}
```

**Reference:** `grocery/discount/criteria/CategoryBasedCriteria.java:1-20`, `grocery_store_deep_dive/CompositeCriteria.java:1-33`

---

### DiscountCalculationStrategy Implementations

The **Strategy Pattern** is also used for discount calculations.

| Strategy | Description | Calculation |
|----------|-------------|-------------|
| `PercentageBasedStrategy` | Percentage discount | `price * (1 - percentage/100)` |
| `AmountBasedStrategy` | Fixed amount off | `max(price - amount, 0)` |

```java
// PercentageBasedStrategy - e.g., 50% off
public class PercentageBasedStrategy implements DiscountCalculationStrategy {
    private final BigDecimal percentage;

    @Override
    public BigDecimal calculateDiscountedPrice(BigDecimal originalPrice) {
        return originalPrice.multiply(
            BigDecimal.ONE.subtract(percentage.divide(BigDecimal.valueOf(100)))
        );
    }
}

// AmountBasedStrategy - e.g., $5 off
public class AmountBasedStrategy implements DiscountCalculationStrategy {
    private final BigDecimal discountAmount;

    @Override
    public BigDecimal calculateDiscountedPrice(BigDecimal originalPrice) {
        return originalPrice.subtract(discountAmount).max(BigDecimal.ZERO);
    }
}
```

**Reference:** `grocery/discount/strategy/PercentageBasedStrategy.java:1-20`, `grocery/discount/strategy/AmountBasedStrategy.java:1-20`

---

### Decorator Pattern for Stacking Discounts

The **Decorator Pattern** allows stacking multiple discounts on top of each other.

```java
// Stack discounts: 50% off, then additional $2 off
DiscountCalculationStrategy baseStrategy = new PercentageBasedStrategy(new BigDecimal("50"));
DiscountCalculationStrategy stackedStrategy = new FixedDiscountDecorator(baseStrategy, new BigDecimal("2"));

// For a $100 item:
// Step 1: 50% off -> $50
// Step 2: $2 off  -> $48
```

```java
// FixedDiscountDecorator - Adds fixed amount to existing discount
public class FixedDiscountDecorator implements DiscountCalculationStrategy {
    private final DiscountCalculationStrategy strategy;
    private final BigDecimal fixedAmount;

    @Override
    public BigDecimal calculateDiscountedPrice(BigDecimal originalPrice) {
        return strategy.calculateDiscountedPrice(originalPrice).subtract(fixedAmount);
    }
}

// PercentageDiscountDecorator - Adds percentage to existing discount
public class PercentageDiscountDecorator implements DiscountCalculationStrategy {
    private final DiscountCalculationStrategy strategy;
    private final BigDecimal additionalPercentage;

    @Override
    public BigDecimal calculateDiscountedPrice(BigDecimal originalPrice) {
        BigDecimal baseDiscountedPrice = strategy.calculateDiscountedPrice(originalPrice);
        return baseDiscountedPrice.multiply(
            BigDecimal.ONE.subtract(additionalPercentage.divide(BigDecimal.valueOf(100)))
        );
    }
}
```

**Reference:** `grocery_store_deep_dive/FixedDiscountDecorator.java:1-23`, `grocery_store_deep_dive/PercentageDiscountDecorator.java:1-24`

---

### Adding a New Discount Type

To add a new discount type (e.g., "Buy 2 Get 1 Free"):

1. **Create new strategy** implementing `DiscountCalculationStrategy`:
```java
public class BuyNGetMFreeStrategy implements DiscountCalculationStrategy {
    private final int buyQuantity;
    private final int freeQuantity;

    @Override
    public BigDecimal calculateDiscountedPrice(BigDecimal originalPrice) {
        // Calculate: pay for buyQuantity out of (buyQuantity + freeQuantity) items
        BigDecimal totalItems = BigDecimal.valueOf(buyQuantity + freeQuantity);
        BigDecimal paidItems = BigDecimal.valueOf(buyQuantity);
        return originalPrice.multiply(paidItems).divide(totalItems, RoundingMode.HALF_UP);
    }
}
```

2. **Use in DiscountCampaign**:
```java
DiscountCampaign buy2Get1 = new DiscountCampaign(
    "B2G1",
    "Buy 2 Get 1 Free",
    new CategoryBasedCriteria("Candy"),
    new BuyNGetMFreeStrategy(2, 1)
);
```

---

### Adding a New Criteria Type

To add complex criteria (e.g., "Items priced over $10"):

1. **Create new criteria** implementing `DiscountCriteria`:
```java
public class PriceThresholdCriteria implements DiscountCriteria {
    private final BigDecimal minPrice;

    public PriceThresholdCriteria(BigDecimal minPrice) {
        this.minPrice = minPrice;
    }

    @Override
    public boolean isApplicable(Item item) {
        return item.getPrice().compareTo(minPrice) >= 0;
    }
}
```

2. **Combine with CompositeCriteria** for complex rules:
```java
// Discount for premium fruits (Fruit category AND price > $5)
CompositeCriteria premiumFruits = new CompositeCriteria(List.of(
    new CategoryBasedCriteria("Fruit"),
    new PriceThresholdCriteria(new BigDecimal("5"))
));
```

---

## Wrap Up

In this design, we created a Grocery Store System that handles product management, inventory tracking, and flexible discount application. Key takeaways:

1. **Strategy Pattern** for discount calculations, allowing different discount types (percentage, fixed amount) without modifying existing code

2. **Strategy Pattern** (again) for discount criteria, enabling flexible eligibility rules (category-based, item-based, price-based)

3. **Composite Pattern** for combining multiple criteria with AND logic, supporting complex discount rules

4. **Decorator Pattern** for stacking discounts, allowing promotions like "50% off + additional $5 off"

5. **Facade Pattern** in `GroceryStoreSystem`, providing a simplified interface to coordinate catalog, inventory, discounts, and checkout

6. **Single Responsibility Principle** with separate `Catalog` and `Inventory` classes

---

## Further Reading: Strategy, Composite, and Decorator Patterns

### Strategy Design Pattern

The Strategy pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable. It lets the algorithm vary independently from clients that use it.

**Application in this design:**

```
                        ┌──────────────────────────────┐
                        │  DiscountCalculationStrategy │
                        │        <<interface>>         │
                        ├──────────────────────────────┤
                        │  +calculateDiscountedPrice() │
                        └──────────────┬───────────────┘
                                       │
              ┌────────────────────────┼────────────────────────┐
              │                        │                        │
              ▼                        ▼                        ▼
     ┌────────────────┐      ┌────────────────┐      ┌────────────────┐
     │ PercentageBased│      │  AmountBased   │      │  BuyNGetMFree  │
     │    Strategy    │      │    Strategy    │      │   (future)     │
     └────────────────┘      └────────────────┘      └────────────────┘
```

**Benefits:**
- Add new discount types without changing existing code (Open/Closed Principle)
- Easy to test each strategy in isolation
- Runtime selection of discount calculation method

### Composite Design Pattern

The Composite pattern composes objects into tree structures to represent part-whole hierarchies. It lets clients treat individual objects and compositions uniformly.

**Application in this design:**

```
              ┌───────────────────┐
              │  DiscountCriteria │
              │   <<interface>>   │
              ├───────────────────┤
              │  +isApplicable()  │
              └─────────┬─────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
   ┌─────────┐    ┌─────────┐    ┌──────────────┐
   │Category │    │  Item   │    │  Composite   │
   │ Based   │    │  Based  │    │   Criteria   │◄────────┐
   └─────────┘    └─────────┘    ├──────────────┤         │
                                 │- criteriaList│─────────┘
                                 │  (children)  │  contains
                                 └──────────────┘  more criteria
```

**Benefits:**
- Combine simple criteria into complex rules
- Treat single criteria and composite criteria uniformly
- Easy to build "AND" (or "OR") combinations

### Decorator Design Pattern

The Decorator pattern attaches additional responsibilities to an object dynamically. It provides a flexible alternative to subclassing for extending functionality.

**Application in this design:**

```
                      ┌──────────────────────────────┐
                      │  DiscountCalculationStrategy │
                      │        <<interface>>         │
                      └──────────────┬───────────────┘
                                     │
           ┌─────────────────────────┼─────────────────────────┐
           │                         │                         │
           ▼                         ▼                         ▼
  ┌────────────────┐       ┌────────────────┐       ┌────────────────┐
  │ PercentageBased│       │   FixedDiscount│       │  Percentage    │
  │    Strategy    │       │    Decorator   │       │   Discount     │
  │   (concrete)   │       │                │       │   Decorator    │
  └────────────────┘       ├────────────────┤       ├────────────────┤
                           │ - strategy ────┼───────│ - strategy ────┼──► wraps
                           │ - fixedAmount  │       │ - percentage   │    another
                           └────────────────┘       └────────────────┘    strategy

  Example: 50% off + $2 off
  ┌────────────────┐     wraps     ┌────────────────┐
  │ FixedDiscount  │──────────────►│ PercentageBased│
  │ Decorator($2)  │               │  Strategy(50%) │
  └────────────────┘               └────────────────┘

  $100 item → 50% → $50 → -$2 → $48
```

**Benefits:**
- Stack multiple discounts dynamically
- Add behavior without modifying existing strategies
- Combine discounts in any order

### Pattern Comparison

| Aspect | Strategy | Composite | Decorator |
|--------|----------|-----------|-----------|
| Intent | Choose algorithm | Build tree structure | Add responsibilities |
| Structure | Interface + implementations | Uniform component interface | Wrapper chain |
| In this design | Discount calculation methods | Combine eligibility criteria | Stack discount modifiers |
| Example | Percentage vs Amount discount | Category AND Price criteria | 50% off + $2 off |
