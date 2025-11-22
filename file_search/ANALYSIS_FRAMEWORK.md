# 11

## Design a File Search System

A File Search System allows users to find files based on various criteria—name patterns, size ranges, ownership, and more. This is a practical problem that tests understanding of **predicate composition**, **the Composite pattern**, and **recursive tree traversal**. Think of it as building a simplified version of Unix `find` command.

This design focuses on **flexible query building** using predicates, **composable search criteria** (AND, OR, NOT), and **extensible comparison operators**.

---

## Requirements Gathering

### Requirements Clarification

**Q: What file attributes can be searched?**
A: The system supports:
- `FILENAME`: String matching (exact, regex)
- `SIZE`: Numeric comparison (equals, greater than, less than)
- `OWNER`: String matching
- `IS_DIRECTORY`: Boolean check

**Q: What comparison operators should be supported?**
A: Multiple operators for flexibility:
- `EqualsOperator`: Exact match
- `GreaterThanOperator`: Numeric > comparison
- `LessThanOperator`: Numeric < comparison
- `RegexMatchOperator`: Pattern matching

**Q: Can search criteria be combined?**
A: Yes, using logical operators:
- `AND`: All predicates must match
- `OR`: At least one predicate must match
- `NOT`: Inverts the predicate result

**Q: How should the file system be traversed?**
A: Using iterative depth-first traversal with a stack. This avoids stack overflow on deep directory structures.

**Q: Can predicates be nested?**
A: Yes, the Composite pattern allows arbitrary nesting:
```
(size > 1000 AND owner = "admin") OR (filename matches "*.log")
```

### Requirements State

| Requirement | Status | Pattern Used |
|-------------|--------|--------------|
| File attributes (name, size, owner, isDir) | ✅ Confirmed | — |
| SimplePredicate (attribute + operator + value) | ✅ Confirmed | Strategy |
| AndPredicate (all must match) | ✅ Confirmed | Composite |
| OrPredicate (any must match) | ✅ Confirmed | Composite |
| NotPredicate (invert result) | ✅ Confirmed | Decorator |
| EqualsOperator | ✅ Confirmed | Strategy |
| GreaterThan/LessThan operators | ✅ Confirmed | Strategy |
| RegexMatchOperator | ✅ Confirmed | Strategy |
| Iterative tree traversal | ✅ Confirmed | — |
| Nested predicate composition | ✅ Confirmed | Composite |
| Parallel search | ❌ Out of scope | — |
| File content search | ❌ Out of scope | — |

---

## Identify Core Objects

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           FILE SEARCH SYSTEM OBJECTS                                 │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         SEARCH DOMAIN                                        │   │
│  │  ┌──────────────────┐    ┌──────────────────┐                               │   │
│  │  │   FileSearch     │───►│FileSearchCriteria│───► Predicate                 │   │
│  │  │                  │    │                  │                               │   │
│  │  │+search(root,     │    │- predicate       │    The criteria wraps a       │   │
│  │  │        criteria) │    │                  │    predicate for matching     │   │
│  │  │                  │    │+isMatch(file)    │                               │   │
│  │  └──────────────────┘    └──────────────────┘                               │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                      PREDICATE DOMAIN (Composite Pattern)                    │   │
│  │                                                                             │   │
│  │                         ┌──────────────────┐                                │   │
│  │                         │    Predicate     │                                │   │
│  │                         │  <<interface>>   │                                │   │
│  │                         ├──────────────────┤                                │   │
│  │                         │ +isMatch(File)   │                                │   │
│  │                         └────────┬─────────┘                                │   │
│  │                                  │                                          │   │
│  │            ┌─────────────────────┼─────────────────────┐                    │   │
│  │            │                     │                     │                    │   │
│  │            ▼                     ▼                     ▼                    │   │
│  │   ┌────────────────┐    ┌────────────────┐    ┌────────────────┐           │   │
│  │   │SimplePredicate │    │ AndPredicate   │    │  OrPredicate   │           │   │
│  │   │                │    │ (Composite)    │    │  (Composite)   │           │   │
│  │   │- attribute     │    │                │    │                │           │   │
│  │   │- operator      │    │- operands[]    │    │- operands[]    │           │   │
│  │   │- expectedValue │    │                │    │                │           │   │
│  │   │                │    │ allMatch()     │    │ anyMatch()     │           │   │
│  │   └────────────────┘    └────────────────┘    └────────────────┘           │   │
│  │            │                                                                │   │
│  │            ▼                                                                │   │
│  │   ┌────────────────┐                                                        │   │
│  │   │ComparisonOperator                                                       │   │
│  │   │  <<interface>> │                                                        │   │
│  │   ├────────────────┤                                                        │   │
│  │   │+isMatch(actual,│                                                        │   │
│  │   │        expected)                                                        │   │
│  │   └───────┬────────┘                                                        │   │
│  │           │                                                                 │   │
│  │     ┌─────┴─────┬─────────────┬─────────────┐                              │   │
│  │     │           │             │             │                              │   │
│  │     ▼           ▼             ▼             ▼                              │   │
│  │  ┌──────┐   ┌───────┐   ┌───────┐   ┌───────────┐                          │   │
│  │  │Equals│   │Greater│   │ Less  │   │RegexMatch │                          │   │
│  │  │  Op  │   │ThanOp │   │ThanOp │   │    Op     │                          │   │
│  │  └──────┘   └───────┘   └───────┘   └───────────┘                          │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                        FILESYSTEM DOMAIN                                     │   │
│  │  ┌──────────────────┐    ┌──────────────────┐                               │   │
│  │  │      File        │    │  FileAttribute   │                               │   │
│  │  │                  │    │     (enum)       │                               │   │
│  │  │- isDirectory     │    │                  │                               │   │
│  │  │- size            │    │ SIZE             │                               │   │
│  │  │- owner           │    │ OWNER            │                               │   │
│  │  │- filename        │    │ IS_DIRECTORY     │                               │   │
│  │  │- entries[]       │    │ FILENAME         │                               │   │
│  │  │                  │    │                  │                               │   │
│  │  │+extract(attr)    │    │                  │                               │   │
│  │  │+addEntry(file)   │    │                  │                               │   │
│  │  └──────────────────┘    └──────────────────┘                               │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Design Class Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                          QUERY COMPOSITION EXAMPLES                                   │
└──────────────────────────────────────────────────────────────────────────────────────┘

  Example 1: Find files larger than 1MB
  ═════════════════════════════════════

  SimplePredicate<Integer>(
      FileAttribute.SIZE,
      new GreaterThanOperator(),
      1_000_000
  )


  Example 2: Find log files owned by admin
  ═════════════════════════════════════════

  AndPredicate([
      SimplePredicate(FILENAME, RegexMatchOperator, ".*\\.log"),
      SimplePredicate(OWNER, EqualsOperator, "admin")
  ])


  Example 3: Complex query
  ═════════════════════════
  Find: (large files AND owned by admin) OR (any .tmp files)

                    ┌───────────┐
                    │    OR     │
                    └─────┬─────┘
                          │
            ┌─────────────┴─────────────┐
            │                           │
      ┌─────┴─────┐              ┌──────┴──────┐
      │    AND    │              │SimplePredicate
      └─────┬─────┘              │filename=*.tmp│
            │                    └──────────────┘
     ┌──────┴──────┐
     │             │
┌────┴────┐  ┌────┴────┐
│size>1MB │  │owner=   │
│         │  │ admin   │
└─────────┘  └─────────┘


┌──────────────────────────────────────────────────────────────────────────────────────┐
│                           TRAVERSAL ALGORITHM                                         │
└──────────────────────────────────────────────────────────────────────────────────────┘

  Using iterative DFS with explicit stack (avoids recursion stack overflow):

  public List<File> search(File root, FileSearchCriteria criteria) {
      List<File> result = new ArrayList<>();
      ArrayDeque<File> stack = new ArrayDeque<>();

      stack.push(root);

      while (!stack.isEmpty()) {
          File current = stack.pop();

          if (criteria.isMatch(current)) {
              result.add(current);
          }

          for (File entry : current.getEntries()) {
              stack.push(entry);
          }
      }

      return result;
  }

  Why iterative over recursive?
  ─────────────────────────────
  - Deep directory structures can cause StackOverflowError
  - Explicit stack size is limited only by heap memory
  - Easier to add early termination or pagination
```

---

## Code

### Deep Dive Topics

#### SimplePredicate: The Building Block

The `SimplePredicate` combines an attribute, operator, and expected value:

```java
public class SimplePredicate<T> implements Predicate {
    private final FileAttribute attributeName;
    private final ComparisonOperator<T> operator;
    private final T expectedValue;

    @Override
    public boolean isMatch(final File inputFile) {
        Object actualValue = inputFile.extract(attributeName);

        if (expectedValue.getClass().isInstance(actualValue)) {
            return operator.isMatch((T) actualValue, expectedValue);
        }
        return false;
    }
}
```

**Type safety:** The generic type `T` ensures the operator and expected value are compatible.

**Reference:** `filesearch/clause/SimplePredicate.java:8-40`

---

#### Composite Pattern: AND and OR Predicates

The composite predicates treat collections of predicates uniformly:

```java
public class AndPredicate implements CompositePredicate {
    private final List<Predicate> operands;

    @Override
    public boolean isMatch(final File inputFile) {
        return operands.stream()
            .allMatch(predicate -> predicate.isMatch(inputFile));
    }
}

public class OrPredicate implements CompositePredicate {
    private final List<Predicate> operands;

    @Override
    public boolean isMatch(final File inputFile) {
        return operands.stream()
            .anyMatch(predicate -> predicate.isMatch(inputFile));
    }
}
```

**Key insight:** Both `SimplePredicate` and composite predicates implement `Predicate`, allowing unlimited nesting.

**Reference:** `filesearch/clause/AndPredicate.java`, `filesearch/clause/OrPredicate.java`

---

#### Strategy Pattern: Comparison Operators

Operators are interchangeable comparison strategies:

```java
public interface ComparisonOperator<T> {
    boolean isMatch(T attributeValue, T expectedValue);
}

public class EqualsOperator<T> implements ComparisonOperator<T> {
    @Override
    public boolean isMatch(T actual, T expected) {
        return actual.equals(expected);
    }
}

public class GreaterThanOperator implements ComparisonOperator<Integer> {
    @Override
    public boolean isMatch(Integer actual, Integer expected) {
        return actual > expected;
    }
}

public class RegexMatchOperator implements ComparisonOperator<String> {
    @Override
    public boolean isMatch(String actual, String pattern) {
        return actual.matches(pattern);
    }
}
```

**Extensibility:** New operators (e.g., `ContainsOperator`, `StartsWithOperator`) can be added without modifying existing code.

**Reference:** `filesearch/operator/*.java`

---

#### File Attribute Extraction

The `File` class uses a switch-based extractor:

```java
public Object extract(final FileAttribute attributeName) {
    switch (attributeName) {
        case SIZE -> { return size; }
        case OWNER -> { return owner; }
        case IS_DIRECTORY -> { return isDirectory; }
        case FILENAME -> { return filename; }
    }
    throw new IllegalArgumentException("invalid filter criteria type");
}
```

**Alternative design:** Could use a `Map<FileAttribute, Supplier>` for more flexibility:

```java
private final Map<FileAttribute, Supplier<Object>> extractors = Map.of(
    SIZE, () -> this.size,
    OWNER, () -> this.owner,
    IS_DIRECTORY, () -> this.isDirectory,
    FILENAME, () -> this.filename
);

public Object extract(FileAttribute attr) {
    return extractors.get(attr).get();
}
```

**Reference:** `filesearch/filesystem/File.java:26-42`

---

### Parking at Code

```
file_search/
├── filesearch/
│   ├── FileSearch.java              # Main search engine with DFS traversal
│   ├── FileSearchCriteria.java      # Wraps predicate for search
│   ├── FileSearchTest.java
│   ├── clause/
│   │   ├── Predicate.java           # Base interface
│   │   ├── SimplePredicate.java     # Attribute + Operator + Value
│   │   ├── CompositePredicate.java  # Marker for composites
│   │   ├── AndPredicate.java        # All must match
│   │   ├── OrPredicate.java         # Any must match
│   │   └── NotPredicate.java        # Invert result
│   ├── operator/
│   │   ├── ComparisonOperator.java  # Strategy interface
│   │   ├── EqualsOperator.java
│   │   ├── GreaterThanOperator.java
│   │   ├── LessThanOperator.java
│   │   └── RegexMatchOperator.java
│   └── filesystem/
│       ├── File.java                # File/directory with attributes
│       └── FileAttribute.java       # Enum of searchable attributes
```

---

## Wrap Up

Key takeaways:

1. **Composite Pattern** for predicates: `AndPredicate` and `OrPredicate` contain lists of `Predicate`, allowing arbitrary nesting

2. **Strategy Pattern** for operators: `ComparisonOperator` interface with multiple implementations (Equals, GreaterThan, Regex)

3. **Separation of Concerns**:
   - `Predicate`: What to match
   - `ComparisonOperator`: How to compare
   - `FileSearch`: How to traverse

4. **Iterative DFS**: Avoids stack overflow on deep directory trees

5. **Type-Safe Generics**: `SimplePredicate<T>` ensures operator and value compatibility

---

## Further Reading

### Composite Pattern

The Composite pattern composes objects into tree structures:

```
              ┌──────────────┐
              │   Predicate  │
              │ <<interface>>│
              └──────┬───────┘
                     │
         ┌───────────┼───────────┐
         │           │           │
         ▼           ▼           ▼
   ┌──────────┐ ┌─────────┐ ┌─────────┐
   │  Simple  │ │   AND   │ │   OR    │
   │Predicate │ │Predicate│ │Predicate│
   │  (Leaf)  │ │(Composite)│(Composite)
   └──────────┘ └────┬────┘ └────┬────┘
                     │           │
               contains      contains
               Predicate[]   Predicate[]
```

**Benefits:**
- Uniform treatment of simple and composite objects
- Recursive composition to arbitrary depth
- Easy to add new composite types (XOR, NAND, etc.)

### Strategy Pattern

The Strategy pattern for operators:

```
SimplePredicate ────► ComparisonOperator<T>
                             │
                    ┌────────┼────────┐
                    │        │        │
                    ▼        ▼        ▼
               Equals   GreaterThan  Regex
```

### Interpreter Pattern (Related)

The predicate structure resembles the Interpreter pattern—each predicate class "interprets" whether a file matches. For complex query languages, a full parser could build predicate trees from string queries:

```
"size > 1000 AND owner = 'admin'"
          ↓ parse
    AndPredicate([
        SimplePredicate(SIZE, >, 1000),
        SimplePredicate(OWNER, =, "admin")
    ])
```
