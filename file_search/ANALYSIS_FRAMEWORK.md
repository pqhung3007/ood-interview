# 11

## Design a File Search System

A File Search System allows users to find files based on various criteria—name patterns, size ranges, ownership, and more. This is a practical problem that tests understanding of **predicate composition**, **the Composite pattern**, **the Strategy pattern**, and **recursive tree traversal**. Think of it as building a simplified version of Unix `find` command.

This design focuses on **flexible query building** using predicates, **composable search criteria** (AND, OR, NOT), **extensible comparison operators**, and **iterative depth-first traversal** that avoids stack overflow.

---

## Requirements Gathering

### Requirements Clarification

**Q: What file attributes can be searched?**
A: The system supports four core attributes:
- `FILENAME`: String matching (exact, prefix, suffix, regex)
- `SIZE`: Numeric comparison (equals, greater than, less than)
- `OWNER`: String matching (exact match)
- `IS_DIRECTORY`: Boolean check (file vs directory)

**Q: What comparison operators should be supported?**
A: Multiple operators for flexibility:
- `EqualsOperator<T>`: Exact match for any type
- `GreaterThanOperator`: Numeric > comparison
- `LessThanOperator`: Numeric < comparison
- `RegexMatchOperator`: Pattern matching for strings

**Q: Can search criteria be combined?**
A: Yes, using logical operators that form a tree structure:
- `AND`: All predicates must match
- `OR`: At least one predicate must match
- `NOT`: Inverts the predicate result

**Q: Can predicates be nested arbitrarily?**
A: Yes, the Composite pattern allows unlimited nesting depth:
```
(size > 1000 AND owner = "admin") OR (filename matches "*.log")
```

**Q: How should the file system be traversed?**
A: Using **iterative depth-first traversal** with an explicit stack. This avoids `StackOverflowError` on deep directory structures that would occur with recursive traversal.

**Q: Should the search return directories, files, or both?**
A: Both by default. Use `IS_DIRECTORY` predicate to filter if needed.

**Q: How should type safety be handled with different attribute types?**
A: Using Java generics. `SimplePredicate<T>` ensures the operator and expected value are type-compatible at compile time.

**Q: Should we support file content search (grep-like)?**
A: No, out of scope. This system searches file metadata only, not contents.

**Q: How do we handle permission errors during traversal?**
A: For interview scope, we assume all files are readable. Production would catch `AccessDeniedException` and continue.

**Q: Should results be sorted?**
A: Out of scope. Results are returned in traversal order. Sorting can be applied by the caller.

**Q: How do we handle symbolic links?**
A: Out of scope for interview. Production would need cycle detection.

**Q: Can we add new operators without modifying existing code?**
A: Yes, the Strategy pattern for operators means adding `ContainsOperator` or `StartsWithOperator` requires no changes to `SimplePredicate`.

### Requirements State

| Requirement | Status | Pattern Used |
|-------------|--------|--------------|
| File attributes (name, size, owner, isDir) | ✅ Confirmed | Enum + extractor |
| SimplePredicate (attribute + operator + value) | ✅ Confirmed | Strategy |
| AndPredicate (all must match) | ✅ Confirmed | Composite |
| OrPredicate (any must match) | ✅ Confirmed | Composite |
| NotPredicate (invert result) | ✅ Confirmed | Decorator |
| EqualsOperator | ✅ Confirmed | Strategy |
| GreaterThan/LessThan operators | ✅ Confirmed | Strategy |
| RegexMatchOperator | ✅ Confirmed | Strategy |
| Iterative tree traversal | ✅ Confirmed | DFS with stack |
| Nested predicate composition | ✅ Confirmed | Composite |
| Type-safe generics | ✅ Confirmed | Java generics |
| Parallel search | ❌ Out of scope | — |
| File content search | ❌ Out of scope | — |
| Symbolic link handling | ❌ Out of scope | — |
| Result sorting | ❌ Out of scope | — |

---

## Identify Core Objects

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                           FILE SEARCH SYSTEM OBJECTS                                         │
├─────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│  │                         SEARCH DOMAIN                                                │   │
│  │                                                                                      │   │
│  │  ┌────────────────────┐         ┌────────────────────┐                              │   │
│  │  │    FileSearch      │────────►│FileSearchCriteria  │                              │   │
│  │  │                    │         │                    │                              │   │
│  │  │ +search(root,      │         │ - predicate        │                              │   │
│  │  │         criteria)  │         │                    │                              │   │
│  │  │                    │         │ +isMatch(file)     │◄──── Wraps a Predicate       │   │
│  │  └────────────────────┘         └────────────────────┘      for cleaner API         │   │
│  │           │                              │                                           │   │
│  │           │ uses                         │ delegates to                              │   │
│  │           ▼                              ▼                                           │   │
│  │  ┌────────────────────┐         ┌────────────────────┐                              │   │
│  │  │  ArrayDeque<File>  │         │     Predicate      │                              │   │
│  │  │     (stack)        │         │   <<interface>>    │                              │   │
│  │  │                    │         │                    │                              │   │
│  │  │  Iterative DFS     │         │ +isMatch(File)     │                              │   │
│  │  │  traversal         │         │                    │                              │   │
│  │  └────────────────────┘         └────────────────────┘                              │   │
│  │                                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│  │                      PREDICATE DOMAIN (Composite Pattern)                            │   │
│  │                                                                                      │   │
│  │                              ┌──────────────────┐                                    │   │
│  │                              │    Predicate     │                                    │   │
│  │                              │  <<interface>>   │                                    │   │
│  │                              ├──────────────────┤                                    │   │
│  │                              │ +isMatch(File)   │                                    │   │
│  │                              └────────┬─────────┘                                    │   │
│  │                                       │                                              │   │
│  │               ┌───────────────────────┼───────────────────────┐                      │   │
│  │               │                       │                       │                      │   │
│  │               ▼                       ▼                       ▼                      │   │
│  │      ┌────────────────┐      ┌────────────────┐      ┌────────────────┐             │   │
│  │      │SimplePredicate │      │ AndPredicate   │      │  OrPredicate   │             │   │
│  │      │     <T>        │      │ (Composite)    │      │  (Composite)   │             │   │
│  │      │                │      │                │      │                │             │   │
│  │      │- attribute     │      │- operands[]    │      │- operands[]    │             │   │
│  │      │- operator      │      │                │      │                │             │   │
│  │      │- expectedValue │      │ allMatch()     │      │ anyMatch()     │             │   │
│  │      └───────┬────────┘      └────────────────┘      └────────────────┘             │   │
│  │              │                        │                                              │   │
│  │              │                        │                                              │   │
│  │              │              ┌─────────┴─────────┐                                    │   │
│  │              │              │   NotPredicate    │                                    │   │
│  │              │              │   (Decorator)     │                                    │   │
│  │              │              │                   │                                    │   │
│  │              │              │ - operand         │                                    │   │
│  │              │              │ !operand.isMatch()│                                    │   │
│  │              │              └───────────────────┘                                    │   │
│  │              │                                                                       │   │
│  │              │ uses                                                                  │   │
│  │              ▼                                                                       │   │
│  │     ┌─────────────────────┐                                                         │   │
│  │     │ ComparisonOperator  │                                                         │   │
│  │     │    <<interface>>    │                                                         │   │
│  │     │        <T>          │                                                         │   │
│  │     ├─────────────────────┤                                                         │   │
│  │     │+isMatch(actual, expected)                                                     │   │
│  │     └──────────┬──────────┘                                                         │   │
│  │                │                                                                     │   │
│  │      ┌─────────┼─────────┬─────────────┬─────────────┐                              │   │
│  │      │         │         │             │             │                              │   │
│  │      ▼         ▼         ▼             ▼             ▼                              │   │
│  │  ┌───────┐ ┌───────┐ ┌───────┐ ┌───────────┐ ┌───────────┐                          │   │
│  │  │Equals │ │Greater│ │ Less  │ │ Regex     │ │ Contains  │                          │   │
│  │  │  Op   │ │ThanOp │ │ThanOp │ │ MatchOp   │ │    Op     │                          │   │
│  │  │  <T>  │ │       │ │       │ │           │ │ (future)  │                          │   │
│  │  └───────┘ └───────┘ └───────┘ └───────────┘ └───────────┘                          │   │
│  │                                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│  │                        FILESYSTEM DOMAIN                                             │   │
│  │                                                                                      │   │
│  │  ┌────────────────────┐         ┌────────────────────┐                              │   │
│  │  │       File         │         │   FileAttribute    │                              │   │
│  │  │                    │         │      (enum)        │                              │   │
│  │  │ - isDirectory      │         │                    │                              │   │
│  │  │ - size             │         │ SIZE          (int)│                              │   │
│  │  │ - owner            │         │ OWNER      (String)│                              │   │
│  │  │ - filename         │         │ IS_DIRECTORY (bool)│                              │   │
│  │  │ - entries[]        │         │ FILENAME   (String)│                              │   │
│  │  │                    │         │                    │                              │   │
│  │  │ +extract(attr)     │◄────────│ Maps enum to field │                              │   │
│  │  │ +getEntries()      │         │                    │                              │   │
│  │  │ +addEntry(file)    │         └────────────────────┘                              │   │
│  │  └────────────────────┘                                                             │   │
│  │                                                                                      │   │
│  │  File can be a directory (has entries) or regular file (no entries)                 │   │
│  │                                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Design Class Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                          PREDICATE COMPOSITION EXAMPLES                                       │
└──────────────────────────────────────────────────────────────────────────────────────────────┘

  Example 1: Find files larger than 1MB
  ═════════════════════════════════════

  SimplePredicate<Integer>(
      FileAttribute.SIZE,
      new GreaterThanOperator(),
      1_000_000
  )

  ┌─────────────────────────────────────────────────────┐
  │                   SimplePredicate                    │
  │                                                     │
  │  attribute: SIZE                                    │
  │  operator:  GreaterThanOperator                     │
  │  expected:  1000000                                 │
  │                                                     │
  │  isMatch(file) → file.size > 1000000               │
  └─────────────────────────────────────────────────────┘


  Example 2: Find log files owned by admin
  ════════════════════════════════════════

  AndPredicate([
      SimplePredicate(FILENAME, RegexMatchOperator, ".*\\.log"),
      SimplePredicate(OWNER, EqualsOperator, "admin")
  ])

              ┌───────────────────────────────────────┐
              │            AndPredicate                │
              │                                       │
              │  allMatch(predicates) must be true    │
              └────────────────┬──────────────────────┘
                               │
              ┌────────────────┴────────────────┐
              │                                 │
              ▼                                 ▼
  ┌───────────────────────┐       ┌───────────────────────┐
  │    SimplePredicate    │       │    SimplePredicate    │
  │                       │       │                       │
  │  attr: FILENAME       │       │  attr: OWNER          │
  │  op:   RegexMatch     │       │  op:   Equals         │
  │  val:  ".*\\.log"     │       │  val:  "admin"        │
  └───────────────────────┘       └───────────────────────┘


  Example 3: Complex nested query
  ═══════════════════════════════
  Find: (large files AND owned by admin) OR (any .tmp files)

                         ┌───────────────────────────────────────┐
                         │             OrPredicate                │
                         │                                       │
                         │  anyMatch(predicates) must be true    │
                         └────────────────┬──────────────────────┘
                                          │
                        ┌─────────────────┴─────────────────┐
                        │                                   │
                        ▼                                   ▼
            ┌───────────────────────┐           ┌───────────────────────┐
            │     AndPredicate      │           │    SimplePredicate    │
            │                       │           │                       │
            │  allMatch() = true    │           │  attr: FILENAME       │
            └───────────┬───────────┘           │  op:   RegexMatch     │
                        │                       │  val:  ".*\\.tmp"     │
           ┌────────────┴────────────┐          └───────────────────────┘
           │                         │
           ▼                         ▼
  ┌────────────────┐       ┌────────────────┐
  │SimplePredicate │       │SimplePredicate │
  │ size > 1MB     │       │ owner = admin  │
  └────────────────┘       └────────────────┘


┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                          NOT PREDICATE (DECORATOR)                                            │
└──────────────────────────────────────────────────────────────────────────────────────────────┘

  Find all files that are NOT directories:

  NotPredicate(
      SimplePredicate(IS_DIRECTORY, EqualsOperator, true)
  )

              ┌───────────────────────────────────────┐
              │           NotPredicate                 │
              │                                       │
              │  !operand.isMatch(file)               │
              └────────────────┬──────────────────────┘
                               │
                               │ wraps
                               ▼
              ┌───────────────────────────────────────┐
              │         SimplePredicate               │
              │                                       │
              │  attr: IS_DIRECTORY                   │
              │  op:   EqualsOperator                 │
              │  val:  true                           │
              │                                       │
              │  isMatch → file.isDirectory == true   │
              └───────────────────────────────────────┘

  Result: NotPredicate inverts → matches when file.isDirectory == false


┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                          SEARCH TRAVERSAL ALGORITHM                                           │
└──────────────────────────────────────────────────────────────────────────────────────────────┘

  Using iterative DFS with explicit stack (avoids recursion stack overflow):

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                                                                         │
  │  public List<File> search(File root, FileSearchCriteria criteria) {     │
  │      List<File> result = new ArrayList<>();                             │
  │      ArrayDeque<File> stack = new ArrayDeque<>();                       │
  │                                                                         │
  │      stack.push(root);                                                  │
  │                                                                         │
  │      while (!stack.isEmpty()) {                                         │
  │          File current = stack.pop();                                    │
  │                                                                         │
  │          if (criteria.isMatch(current)) {                               │
  │              result.add(current);                                       │
  │          }                                                              │
  │                                                                         │
  │          // Add children for further exploration                        │
  │          for (File entry : current.getEntries()) {                      │
  │              stack.push(entry);                                         │
  │          }                                                              │
  │      }                                                                  │
  │                                                                         │
  │      return result;                                                     │
  │  }                                                                      │
  │                                                                         │
  └─────────────────────────────────────────────────────────────────────────┘


  Visual traversal example:
  ═════════════════════════

  File system:
                          /root
                         /      \
                     /dir1      /dir2
                    /    \          \
                file1   file2      file3

  Stack operations:
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                                                                         │
  │  Step 1: push(root)          Stack: [root]                              │
  │                                                                         │
  │  Step 2: pop → root          Stack: []                                  │
  │          check match(root)                                              │
  │          push(dir2, dir1)    Stack: [dir1, dir2]                        │
  │                                                                         │
  │  Step 3: pop → dir1          Stack: [dir2]                              │
  │          check match(dir1)                                              │
  │          push(file2, file1)  Stack: [file1, file2, dir2]                │
  │                                                                         │
  │  Step 4: pop → file1         Stack: [file2, dir2]                       │
  │          check match(file1)                                             │
  │          no children                                                    │
  │                                                                         │
  │  Step 5: pop → file2         Stack: [dir2]                              │
  │          check match(file2)                                             │
  │          no children                                                    │
  │                                                                         │
  │  Step 6: pop → dir2          Stack: []                                  │
  │          check match(dir2)                                              │
  │          push(file3)         Stack: [file3]                             │
  │                                                                         │
  │  Step 7: pop → file3         Stack: []                                  │
  │          check match(file3)                                             │
  │          no children                                                    │
  │                                                                         │
  │  Step 8: stack empty → return results                                   │
  │                                                                         │
  └─────────────────────────────────────────────────────────────────────────┘


┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                          FILE ATTRIBUTE EXTRACTION                                            │
└──────────────────────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                                                                         │
  │  File                                 FileAttribute (enum)              │
  │  ─────                                ─────────────────────             │
  │  - filename: "report.log"             SIZE                              │
  │  - size: 1024                         OWNER                             │
  │  - owner: "admin"                     IS_DIRECTORY                      │
  │  - isDirectory: false                 FILENAME                          │
  │                                                                         │
  │                                                                         │
  │  file.extract(SIZE)         → 1024                                      │
  │  file.extract(OWNER)        → "admin"                                   │
  │  file.extract(FILENAME)     → "report.log"                              │
  │  file.extract(IS_DIRECTORY) → false                                     │
  │                                                                         │
  └─────────────────────────────────────────────────────────────────────────┘
```

---

## Code

### Deep Dive Topics

#### Deep Dive 1: SimplePredicate - The Building Block

The `SimplePredicate` is the leaf node in our predicate tree. It combines an attribute, operator, and expected value:

```java
public class SimplePredicate<T> implements Predicate {
    private final FileAttribute attributeName;
    private final ComparisonOperator<T> operator;
    private final T expectedValue;

    public SimplePredicate(
            final FileAttribute attributeName,
            final ComparisonOperator<T> operator,
            final T expectedValue) {
        this.attributeName = attributeName;
        this.operator = operator;
        this.expectedValue = expectedValue;
    }

    @Override
    @SuppressWarnings("unchecked")
    public boolean isMatch(final File inputFile) {
        // Extract the actual value from the file
        Object actualValue = inputFile.extract(attributeName);

        // Type check before comparison
        if (expectedValue.getClass().isInstance(actualValue)) {
            return operator.isMatch((T) actualValue, expectedValue);
        }
        return false;
    }
}
```

**Why generics matter here:**

```java
// Type-safe at compile time
SimplePredicate<Integer> sizePred = new SimplePredicate<>(
    FileAttribute.SIZE,
    new GreaterThanOperator(),  // expects Integer
    1000                         // Integer value
);

// This would be a compile error:
// new SimplePredicate<Integer>(SIZE, new RegexMatchOperator(), 1000)
// RegexMatchOperator expects String, not Integer
```

**The runtime type check:**

```java
if (expectedValue.getClass().isInstance(actualValue)) {
    return operator.isMatch((T) actualValue, expectedValue);
}
return false;
```

This handles the case where file attributes might have unexpected types at runtime, preventing `ClassCastException`.

**Reference:** `filesearch/clause/SimplePredicate.java:8-40`

---

#### Deep Dive 2: Composite Pattern - AND and OR Predicates

The composite predicates treat collections of predicates uniformly, enabling tree-like structures:

```java
public class AndPredicate implements CompositePredicate {
    private final List<Predicate> operands;

    public AndPredicate(final List<Predicate> operands) {
        this.operands = operands;
    }

    @Override
    public boolean isMatch(final File inputFile) {
        return operands.stream()
            .allMatch(predicate -> predicate.isMatch(inputFile));
    }
}

public class OrPredicate implements CompositePredicate {
    private final List<Predicate> operands;

    public OrPredicate(final List<Predicate> operands) {
        this.operands = operands;
    }

    @Override
    public boolean isMatch(final File inputFile) {
        return operands.stream()
            .anyMatch(predicate -> predicate.isMatch(inputFile));
    }
}
```

**Why Composite Pattern works here:**

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│   Predicate (interface)                                                │
│        │                                                               │
│        ├── SimplePredicate      (leaf - no children)                   │
│        │                                                               │
│        ├── AndPredicate         (composite - contains Predicate[])     │
│        │                                                               │
│        ├── OrPredicate          (composite - contains Predicate[])     │
│        │                                                               │
│        └── NotPredicate         (decorator - contains one Predicate)   │
│                                                                        │
│   Key insight: Both leaf and composite implement the same interface,   │
│   so they can be used interchangeably and nested arbitrarily.          │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

**Building complex queries:**

```java
// (size > 1MB AND owner = "admin") OR (filename matches "*.tmp")
Predicate query = new OrPredicate(List.of(
    new AndPredicate(List.of(
        new SimplePredicate<>(SIZE, new GreaterThanOperator(), 1_000_000),
        new SimplePredicate<>(OWNER, new EqualsOperator<>(), "admin")
    )),
    new SimplePredicate<>(FILENAME, new RegexMatchOperator(), ".*\\.tmp")
));
```

**Reference:** `filesearch/clause/AndPredicate.java`, `filesearch/clause/OrPredicate.java`

---

#### Deep Dive 3: Strategy Pattern - Comparison Operators

Operators are interchangeable comparison strategies that define **how** to compare values:

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

public class LessThanOperator implements ComparisonOperator<Integer> {
    @Override
    public boolean isMatch(Integer actual, Integer expected) {
        return actual < expected;
    }
}

public class RegexMatchOperator implements ComparisonOperator<String> {
    @Override
    public boolean isMatch(String actual, String pattern) {
        return actual.matches(pattern);
    }
}
```

**Why Strategy Pattern:**

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  SimplePredicate doesn't know HOW to compare—it delegates to the       │
│  operator. This means:                                                 │
│                                                                        │
│  1. SimplePredicate is closed for modification                         │
│  2. New operators can be added without changing SimplePredicate        │
│  3. Each operator encapsulates one comparison strategy                 │
│                                                                        │
│                    SimplePredicate                                     │
│                          │                                             │
│                          │ uses                                        │
│                          ▼                                             │
│                ComparisonOperator<T>                                   │
│                          │                                             │
│          ┌───────────────┼───────────────┐                            │
│          │               │               │                            │
│          ▼               ▼               ▼                            │
│       Equals        GreaterThan      Regex                            │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

**Adding new operators (Open/Closed Principle):**

```java
// Adding a new operator requires NO changes to SimplePredicate
public class StartsWithOperator implements ComparisonOperator<String> {
    @Override
    public boolean isMatch(String actual, String prefix) {
        return actual.startsWith(prefix);
    }
}

public class ContainsOperator implements ComparisonOperator<String> {
    @Override
    public boolean isMatch(String actual, String substring) {
        return actual.contains(substring);
    }
}

public class BetweenOperator implements ComparisonOperator<Integer> {
    private final int min;
    private final int max;

    public BetweenOperator(int min, int max) {
        this.min = min;
        this.max = max;
    }

    @Override
    public boolean isMatch(Integer actual, Integer ignored) {
        return actual >= min && actual <= max;
    }
}
```

**Reference:** `filesearch/operator/*.java`

---

#### Deep Dive 4: NotPredicate - The Decorator

`NotPredicate` wraps another predicate and inverts its result:

```java
public class NotPredicate implements CompositePredicate {
    private final Predicate operand;

    public NotPredicate(final Predicate operand) {
        this.operand = operand;
    }

    @Override
    public boolean isMatch(final File inputFile) {
        return !operand.isMatch(inputFile);
    }
}
```

**Why it's a Decorator, not a Composite:**

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  Composite Pattern: Contains COLLECTION of same-type objects           │
│  - AndPredicate has List<Predicate>                                    │
│  - OrPredicate has List<Predicate>                                     │
│                                                                        │
│  Decorator Pattern: Wraps SINGLE object, adds behavior                 │
│  - NotPredicate has ONE Predicate                                      │
│  - Adds inversion behavior: !operand.isMatch()                         │
│                                                                        │
│  The distinction matters for:                                          │
│  - Documentation clarity                                               │
│  - Understanding intent (one vs many)                                  │
│  - Pattern recognition in interviews                                   │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

**Usage examples:**

```java
// Find files that are NOT directories
Predicate notDirectory = new NotPredicate(
    new SimplePredicate<>(IS_DIRECTORY, new EqualsOperator<>(), true)
);

// Find files NOT owned by root
Predicate notRoot = new NotPredicate(
    new SimplePredicate<>(OWNER, new EqualsOperator<>(), "root")
);

// Complex: NOT (large AND owned by admin)
Predicate complex = new NotPredicate(
    new AndPredicate(List.of(
        new SimplePredicate<>(SIZE, new GreaterThanOperator(), 1_000_000),
        new SimplePredicate<>(OWNER, new EqualsOperator<>(), "admin")
    ))
);
```

**Reference:** `filesearch/clause/NotPredicate.java`

---

#### Deep Dive 5: File Attribute Extraction

The `File` class uses a switch-based extractor to retrieve attribute values:

```java
public class File {
    private final boolean isDirectory;
    private final int size;
    private final String owner;
    private final String filename;
    private final List<File> entries;

    public Object extract(final FileAttribute attributeName) {
        switch (attributeName) {
            case SIZE -> { return size; }
            case OWNER -> { return owner; }
            case IS_DIRECTORY -> { return isDirectory; }
            case FILENAME -> { return filename; }
        }
        throw new IllegalArgumentException("Invalid filter criteria type");
    }
}
```

**Design trade-offs:**

| Approach | Pros | Cons |
|----------|------|------|
| Switch statement | Simple, fast, familiar | Violates Open/Closed |
| Map<Attribute, Supplier> | Extensible | More complex setup |
| Reflection | Very extensible | Slow, fragile |

**Alternative: Map-based extractor**

```java
public class File {
    private final Map<FileAttribute, Supplier<Object>> extractors;

    public File(String filename, int size, String owner, boolean isDirectory) {
        this.filename = filename;
        this.size = size;
        this.owner = owner;
        this.isDirectory = isDirectory;

        this.extractors = Map.of(
            SIZE, () -> this.size,
            OWNER, () -> this.owner,
            IS_DIRECTORY, () -> this.isDirectory,
            FILENAME, () -> this.filename
        );
    }

    public Object extract(FileAttribute attr) {
        Supplier<Object> extractor = extractors.get(attr);
        if (extractor == null) {
            throw new IllegalArgumentException("Unknown attribute: " + attr);
        }
        return extractor.get();
    }
}
```

**Reference:** `filesearch/filesystem/File.java:26-42`

---

#### Deep Dive 6: Iterative DFS vs Recursive DFS

The search uses iterative depth-first search with an explicit stack:

```java
public List<File> search(final File root, final FileSearchCriteria criteria) {
    final List<File> result = new ArrayList<>();
    final ArrayDeque<File> recursionStack = new ArrayDeque<>();

    recursionStack.add(root);

    while (!recursionStack.isEmpty()) {
        File next = recursionStack.pop();

        if (criteria.isMatch(next)) {
            result.add(next);
        }

        for (File entry : next.getEntries()) {
            recursionStack.push(entry);
        }
    }

    return result;
}
```

**Why iterative over recursive?**

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  Recursive DFS (problematic):                                          │
│  ────────────────────────────                                          │
│                                                                        │
│  public void searchRecursive(File current) {                           │
│      if (criteria.isMatch(current)) result.add(current);               │
│      for (File child : current.getEntries()) {                         │
│          searchRecursive(child);  // Each call adds to JVM stack!      │
│      }                                                                 │
│  }                                                                     │
│                                                                        │
│  Problem: Deep directory (e.g., 10,000 levels) → StackOverflowError    │
│           JVM stack is typically limited to ~1MB                       │
│                                                                        │
│                                                                        │
│  Iterative DFS (solution):                                             │
│  ─────────────────────────                                             │
│                                                                        │
│  ArrayDeque<File> stack = new ArrayDeque<>();  // Lives on heap        │
│                                                                        │
│  - Heap is much larger than stack (GBs vs MBs)                         │
│  - Can handle millions of files/directories                            │
│  - Easy to add features (pagination, early termination)                │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

**DFS vs BFS comparison:**

```java
// DFS with Stack (current implementation) - LIFO
ArrayDeque<File> stack = new ArrayDeque<>();
stack.push(root);
while (!stack.isEmpty()) {
    File current = stack.pop();      // Last In, First Out
    for (File child : current.getEntries()) {
        stack.push(child);
    }
}

// BFS with Queue - FIFO
ArrayDeque<File> queue = new ArrayDeque<>();
queue.add(root);
while (!queue.isEmpty()) {
    File current = queue.poll();     // First In, First Out
    for (File child : current.getEntries()) {
        queue.add(child);
    }
}
```

| Algorithm | Order | Use Case |
|-----------|-------|----------|
| DFS | Deep-first (explores one branch fully before backtracking) | Finding any match quickly, memory efficient |
| BFS | Level-by-level | Finding closest matches, shortest paths |

**Reference:** `filesearch/FileSearch.java:8-29`

---

#### Deep Dive 7: Type Safety and Generic Constraints

The system uses generics to ensure type safety between operators and values:

```java
// The generic type T flows through the system:

public interface ComparisonOperator<T> {
    boolean isMatch(T attributeValue, T expectedValue);
}

public class SimplePredicate<T> implements Predicate {
    private final ComparisonOperator<T> operator;
    private final T expectedValue;  // Must match operator's type
}
```

**What the type system prevents:**

```java
// COMPILE ERROR: Type mismatch
SimplePredicate<Integer> wrongType = new SimplePredicate<>(
    FileAttribute.SIZE,
    new RegexMatchOperator(),  // Expects String, not Integer!
    1000
);

// CORRECT: Types align
SimplePredicate<Integer> correct = new SimplePredicate<>(
    FileAttribute.SIZE,
    new GreaterThanOperator(),  // Expects Integer
    1000                         // Integer value
);
```

**The runtime type check handles edge cases:**

```java
@Override
public boolean isMatch(final File inputFile) {
    Object actualValue = inputFile.extract(attributeName);

    // Runtime check: is actualValue compatible with expectedValue's type?
    if (expectedValue.getClass().isInstance(actualValue)) {
        return operator.isMatch((T) actualValue, expectedValue);
    }
    return false;  // Type mismatch at runtime → no match
}
```

**Why this check is needed:**

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  Compile time: We know T, but extract() returns Object                 │
│                                                                        │
│  file.extract(SIZE) returns Object (could be Integer, String, etc.)   │
│                                                                        │
│  Without runtime check:                                                │
│  - If file.extract(SIZE) returns String "1000" (corrupted data)        │
│  - Casting to Integer would throw ClassCastException                   │
│                                                                        │
│  With runtime check:                                                   │
│  - Check if "1000" instanceof Integer → false                          │
│  - Return false instead of crashing                                    │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

### Parking at Code

```
file_search/
├── filesearch/
│   ├── FileSearch.java              # Main search engine with DFS traversal
│   ├── FileSearchCriteria.java      # Wraps predicate for search API
│   ├── FileSearchTest.java          # Unit tests
│   │
│   ├── clause/
│   │   ├── Predicate.java           # Base interface: isMatch(File)
│   │   ├── SimplePredicate.java     # Leaf: Attribute + Operator + Value
│   │   ├── CompositePredicate.java  # Marker interface for composites
│   │   ├── AndPredicate.java        # Composite: All must match
│   │   ├── OrPredicate.java         # Composite: Any must match
│   │   └── NotPredicate.java        # Decorator: Invert result
│   │
│   ├── operator/
│   │   ├── ComparisonOperator.java  # Strategy interface
│   │   ├── EqualsOperator.java      # Exact match (generic)
│   │   ├── GreaterThanOperator.java # Numeric >
│   │   ├── LessThanOperator.java    # Numeric <
│   │   └── RegexMatchOperator.java  # Pattern matching
│   │
│   └── filesystem/
│       ├── File.java                # File/directory with attributes
│       └── FileAttribute.java       # Enum: SIZE, OWNER, FILENAME, IS_DIRECTORY
```

### Class-by-Class Analysis

| Class | Purpose | Key Methods | Pattern |
|-------|---------|-------------|---------|
| `FileSearch` | Orchestrates traversal | `search(root, criteria)` | Facade |
| `FileSearchCriteria` | Wraps predicate | `isMatch(file)` | Adapter |
| `Predicate` | Match interface | `isMatch(file)` | Interface |
| `SimplePredicate<T>` | Leaf predicate | `isMatch(file)` | Leaf (Composite) |
| `AndPredicate` | All must match | `allMatch()` | Composite |
| `OrPredicate` | Any must match | `anyMatch()` | Composite |
| `NotPredicate` | Invert result | `!operand.isMatch()` | Decorator |
| `ComparisonOperator<T>` | Comparison strategy | `isMatch(actual, expected)` | Strategy |
| `File` | Filesystem node | `extract(attr)`, `getEntries()` | — |
| `FileAttribute` | Attribute enum | SIZE, OWNER, FILENAME, IS_DIRECTORY | Enum |

---

## Extension Guide: Adding a New Operator

**Scenario:** Add a `ContainsOperator` for substring matching

**Step 1: Create the operator**

```java
public class ContainsOperator implements ComparisonOperator<String> {
    @Override
    public boolean isMatch(String actual, String substring) {
        return actual.contains(substring);
    }
}
```

**Step 2: Use it (no other changes needed!)**

```java
// Find files with "report" in the filename
Predicate containsReport = new SimplePredicate<>(
    FileAttribute.FILENAME,
    new ContainsOperator(),
    "report"
);
```

**This is Open/Closed Principle:** `SimplePredicate`, `AndPredicate`, `OrPredicate` are all unchanged.

---

## Extension Guide: Adding a New File Attribute

**Scenario:** Add `CREATED_DATE` attribute

**Step 1: Update FileAttribute enum**

```java
public enum FileAttribute {
    SIZE,
    OWNER,
    IS_DIRECTORY,
    FILENAME,
    CREATED_DATE  // New!
}
```

**Step 2: Update File class**

```java
public class File {
    private final LocalDateTime createdDate;  // New field

    public Object extract(final FileAttribute attributeName) {
        switch (attributeName) {
            case SIZE -> { return size; }
            case OWNER -> { return owner; }
            case IS_DIRECTORY -> { return isDirectory; }
            case FILENAME -> { return filename; }
            case CREATED_DATE -> { return createdDate; }  // New case
        }
        throw new IllegalArgumentException("Invalid attribute");
    }
}
```

**Step 3: Create date comparison operators**

```java
public class BeforeOperator implements ComparisonOperator<LocalDateTime> {
    @Override
    public boolean isMatch(LocalDateTime actual, LocalDateTime expected) {
        return actual.isBefore(expected);
    }
}

public class AfterOperator implements ComparisonOperator<LocalDateTime> {
    @Override
    public boolean isMatch(LocalDateTime actual, LocalDateTime expected) {
        return actual.isAfter(expected);
    }
}
```

**Step 4: Use the new attribute**

```java
// Find files created before 2024
Predicate oldFiles = new SimplePredicate<>(
    FileAttribute.CREATED_DATE,
    new BeforeOperator(),
    LocalDateTime.of(2024, 1, 1, 0, 0)
);
```

---

## Wrap Up

### Key Takeaways

1. **Composite Pattern** for predicates: `AndPredicate` and `OrPredicate` contain lists of `Predicate`, allowing arbitrary nesting of search criteria

2. **Strategy Pattern** for operators: `ComparisonOperator<T>` interface with multiple implementations (Equals, GreaterThan, Regex) makes adding new comparison logic trivial

3. **Decorator Pattern** for NOT: `NotPredicate` wraps a single predicate and inverts its result

4. **Separation of Concerns**:
   - `Predicate`: **What** to match (the criteria)
   - `ComparisonOperator`: **How** to compare values
   - `FileSearch`: **Where** to search (traversal)

5. **Iterative DFS**: Avoids stack overflow on deep directory trees by using explicit heap-based stack

6. **Type-Safe Generics**: `SimplePredicate<T>` ensures operator and value compatibility at compile time

7. **Runtime Type Check**: Handles edge cases where file attributes might have unexpected types

---

## Further Reading

### Composite Pattern

The Composite pattern composes objects into tree structures:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│              ┌──────────────┐                                          │
│              │   Predicate  │  ← Component interface                   │
│              │ <<interface>>│                                          │
│              └──────┬───────┘                                          │
│                     │                                                   │
│         ┌───────────┼───────────┐                                      │
│         │           │           │                                      │
│         ▼           ▼           ▼                                      │
│   ┌──────────┐ ┌─────────┐ ┌─────────┐                                │
│   │  Simple  │ │   AND   │ │   OR    │                                │
│   │Predicate │ │Predicate│ │Predicate│                                │
│   │  (Leaf)  │ │(Composite)│(Composite)                               │
│   └──────────┘ └────┬────┘ └────┬────┘                                │
│                     │           │                                      │
│               contains      contains                                   │
│               Predicate[]   Predicate[]                                │
│                                                                         │
│  Benefits:                                                             │
│  - Uniform treatment of simple and composite objects                   │
│  - Recursive composition to arbitrary depth                            │
│  - Easy to add new composite types (XOR, NAND, etc.)                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Strategy Pattern

The Strategy pattern for operators:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  SimplePredicate ────uses────► ComparisonOperator<T>                   │
│       (Context)                      (Strategy)                        │
│                                          │                             │
│                                 ┌────────┼────────┐                    │
│                                 │        │        │                    │
│                                 ▼        ▼        ▼                    │
│                             Equals   GreaterThan  Regex                │
│                                                                         │
│  The Context (SimplePredicate) doesn't know HOW to compare.            │
│  It delegates to the Strategy (ComparisonOperator).                    │
│                                                                         │
│  Benefits:                                                             │
│  - Algorithm can be selected at runtime                                │
│  - New algorithms added without modifying context                      │
│  - Each algorithm is encapsulated and testable                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Interpreter Pattern (Related)

The predicate structure resembles the Interpreter pattern—each predicate class "interprets" whether a file matches the criteria:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  Query String:  "size > 1000 AND owner = 'admin'"                      │
│                                                                         │
│           │                                                            │
│           │ parse (hypothetical)                                       │
│           ▼                                                            │
│                                                                         │
│      AndPredicate([                                                    │
│          SimplePredicate(SIZE, GreaterThan, 1000),                     │
│          SimplePredicate(OWNER, Equals, "admin")                       │
│      ])                                                                │
│                                                                         │
│           │                                                            │
│           │ interpret (evaluate against file)                          │
│           ▼                                                            │
│                                                                         │
│      true / false                                                      │
│                                                                         │
│  For complex query languages, a parser could build predicate trees     │
│  from string queries, implementing a full Interpreter pattern.         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Visitor Pattern (Alternative)

An alternative to the current design would use Visitor for operations:

```java
interface PredicateVisitor {
    void visit(SimplePredicate predicate);
    void visit(AndPredicate predicate);
    void visit(OrPredicate predicate);
    void visit(NotPredicate predicate);
}

// Visitor for matching
class MatchVisitor implements PredicateVisitor {
    private final File file;
    private boolean result;

    @Override
    public void visit(SimplePredicate p) {
        result = p.getOperator().isMatch(
            file.extract(p.getAttribute()),
            p.getExpectedValue()
        );
    }
}

// Visitor for printing query tree
class PrintVisitor implements PredicateVisitor {
    @Override
    public void visit(AndPredicate p) {
        System.out.println("AND(");
        for (Predicate child : p.getOperands()) {
            child.accept(this);
        }
        System.out.println(")");
    }
}
```

**When to use Visitor vs current approach:**
- **Current (polymorphism)**: Simple, works well when operations are stable
- **Visitor**: Better when you frequently add new operations but rarely add new predicate types
