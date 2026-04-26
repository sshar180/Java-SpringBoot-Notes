# Java-SpringBoot-Notes


# ☕ Java — Backend Interview Prep

> Comprehensive Java notes for backend interview preparation. Covers core Java, collections, streams, concurrency, and Java 17/21 features. Each section includes code examples and high-signal interview Q&A.

---

## 📑 Table of Contents

1. [Java Fundamentals](#java-fundamentals)
2. [OOP & Access Modifiers](#oop--access-modifiers)
3. [String, StringBuilder, StringBuffer](#string-stringbuilder-stringbuffer)
4. [Wrapper Classes & Type Conversion](#wrapper-classes--type-conversion)
5. [Collections Framework](#collections-framework)
6. [Map Deep Dive](#map-deep-dive)
7. [Comparator & Comparable](#comparator--comparable)
8. [Streams API](#streams-api)
9. [Collectors](#collectors)
10. [Optional](#optional)
11. [Object Mapper (Jackson)](#object-mapper-jackson)
12. [Multithreading & Concurrency](#multithreading--concurrency)
13. [Java 17 Features](#java-17-features)
14. [Java 21 Features](#java-21-features)
15. [Top 50 Interview Q&A](#top-50-interview-qa)

---

## Java Fundamentals

### Why Java is Platform Independent

- The Java compiler (`javac`) compiles `.java` source files into `.class` files containing **bytecode**.
- Bytecode is platform-neutral.
- The **JVM** (which is platform-dependent) interprets/JIT-compiles bytecode into machine code for the host OS.
- **Each OS has its own JVM**, but all JVMs can run the same bytecode → "Write Once, Run Anywhere".

### JDK vs JRE vs JVM

| Component | What it is | Contains |
|-----------|------------|----------|
| **JVM** | Abstract machine that executes bytecode | Class loader, bytecode interpreter, JIT compiler, garbage collector |
| **JRE** | Java Runtime Environment | JVM + core libraries (`java.util`, `java.io`) + supporting files |
| **JDK** | Java Development Kit (superset of JRE) | JRE + dev tools (`javac`, `javadoc`, `jdb`, `jar`) |

**Visualize:** `JDK ⊃ JRE ⊃ JVM`

### JVM Responsibilities

1. **Loads bytecode** — reads & verifies `.class` files (Class Loader subsystem).
2. **Interprets/JITs bytecode** — converts to native machine code.
3. **Manages memory** — heap, stack, metaspace, garbage collection.
4. **Ensures security** — verifies bytecode for unsafe operations (Bytecode Verifier).

### `==` vs `.equals()`

```java
// == compares object references (memory addresses) for objects, values for primitives
int a = 5, b = 5;
System.out.println(a == b); // true (primitive value comparison)

String s1 = new String("Hello");
String s2 = new String("Hello");
System.out.println(s1 == s2);       // false (different references)
System.out.println(s1.equals(s2));  // true  (logical/content equality)
```

> **Rule:** Always override `equals()` AND `hashCode()` together. Two objects equal by `equals()` MUST have the same `hashCode()`.

---

## OOP & Access Modifiers

| Modifier | Same Class | Same Package | Subclass (other pkg) | Other Package |
|----------|:---:|:---:|:---:|:---:|
| `public` | ✅ | ✅ | ✅ | ✅ |
| `protected` | ✅ | ✅ | ✅ (only via inheritance) | ❌ |
| *default* (no modifier) | ✅ | ✅ | ❌ | ❌ |
| `private` | ✅ | ❌ | ❌ | ❌ |

### Accessing Protected from Different Package

A `protected` method in a different package is **only** accessible through inheritance, using `this` or via a subclass instance — never via the parent's reference.

```java
// package p1
public class A {
    protected void display() {
        System.out.println("Protected method in A");
    }
}

// package p2
import p1.A;
public class B extends A {
    public void callDisplay() {
        this.display();   // ✅ allowed via inheritance
        // new A().display(); // ❌ compile error
    }
}
```

### `final`, `finally`, `finalize`

| Keyword | Purpose |
|---------|---------|
| `final` | Make variable constant, prevent class inheritance, prevent method override |
| `finally` | Block that always runs after try/catch (regardless of exception) |
| `finalize()` | Object class method called by GC before collection (**deprecated since Java 9**) |

```java
// Modifying final reference variable — content can change, reference cannot
final StringBuilder sb = new StringBuilder("Hello");
sb.append(" World");   // ✅ allowed
// sb = new StringBuilder("Hi"); // ❌ error: cannot reassign final ref

final List<Integer> i = new ArrayList<>();
i.add(1); // ✅ allowed (mutating contents)
```

### Method Hiding vs Method Overriding

> **Static methods cannot be overridden — they are hidden.**

```java
class Parent {
    static void show() { System.out.println("parent"); }
}
class Child extends Parent {
    static void show() { System.out.println("child"); }
}
public class Test {
    public static void main(String[] args) {
        Parent p = new Child();
        p.show();  // prints "parent" — method hiding (resolved at compile time)
    }
}
```

For **instance** methods, the same scenario would print `"child"` (dynamic dispatch).

### Override Rules — Edge Cases

- **Override `final` method** → ❌ compile error
- **Override `private` method** → Not really overriding; it creates a new method in child. Parent reference cannot call child's method via the private name.
- **Override `static` method** → No error, but it's method hiding, not overriding.
- **Constructor as `final`** → ❌ compile error (constructors aren't inherited, so `final` is meaningless).
- **Abstract class with constructor** → ✅ allowed (called via subclass `super()`).
- **Instantiate abstract class directly** → ❌ no, but you can use **anonymous inner class**:

```java
abstract class Animal { abstract void sound(); }

Animal a = new Animal() {
    @Override
    void sound() { System.out.println("anonymous animal"); }
};
a.sound();
```

### Singleton Class

Ensures only **one instance** of a class exists, with a global access point. Used for: configuration, DB connections, loggers.

```java
public class Singleton {
    private static Singleton instance;     // private static
    private Singleton() {}                 // private constructor
    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

> ⚠️ The above is **not thread-safe**. For thread-safe singleton, use double-checked locking with `volatile`, or better — use enum singleton:
>
> ```java
> public enum Singleton { INSTANCE; public void doStuff() { ... } }
> ```

### try-catch-finally

You can have:
1. `try` + `catch`
2. `try` + `finally`
3. `try` + `catch` + `finally`

`try` alone is **not** valid.

---

## String, StringBuilder, StringBuffer

### Why String is Immutable

1. **Security** — strings often hold sensitive data (file paths, URLs, credentials).
2. **String pool optimization** — JVM caches string literals; if mutable, changing one literal would affect all references.
3. **Thread safety** — immutable objects are inherently thread-safe.
4. **HashCode consistency** — strings used as `HashMap` keys must have stable hashCodes; mutable keys would break the map.
5. **Memory efficiency** — strings can be safely shared across the program without defensive copying.

### Comparison Table

| Feature | `String` | `StringBuilder` | `StringBuffer` |
|---------|----------|-----------------|----------------|
| Mutability | Immutable | Mutable | Mutable |
| Thread Safety | Thread-safe (immutable) | ❌ Not thread-safe | ✅ Thread-safe (synchronized) |
| Performance | Slow (creates new objects) | **Fastest** | Slower (synchronization overhead) |
| Use Case | Constant strings | Single-threaded mutation | Multi-threaded mutation |

### Common String Methods

```java
"Hello".length();                      // 5
"Hello".charAt(1);                     // 'e'
"".isEmpty();                          // true
"   ".isBlank();                       // true (since Java 11)
"HELLO".toLowerCase();                 // "hello"
"hello".toUpperCase();                 // "HELLO"
"Hello World".indexOf("o");            // 4 (first occurrence)
"Hello World".indexOf("o", 5);         // 7
"Hello World".lastIndexOf("o");        // 7
"Hello World".contains("World");       // true
"Hello".startsWith("He");              // true
"Hello".endsWith("lo");                // true
"Hello".equals("hello");               // false (case-sensitive)
"Hello".equalsIgnoreCase("hello");     // true
"Hello".compareTo("World");            // -15 (negative -> "Hello" < "World")
"Hello World".substring(6);            // "World"
"  Hello  ".trim();                    // "Hello"
"  Hello  ".strip();                   // "Hello" (Unicode-aware, since Java 11)
"a,b,c,d".split(",");                  // ["a","b","c","d"]
String.join("-", "a", "b", "c");       // "a-b-c"
"Hello".toCharArray();                 // ['H','e','l','l','o']
```

---

## Wrapper Classes & Type Conversion

### Primitive ↔ Wrapper Mapping

| # | Primitive | Wrapper | Bits |
|---|-----------|---------|------|
| 1 | `byte` | `Byte` | 8 |
| 2 | `short` | `Short` | 16 |
| 3 | `int` | `Integer` | 32 |
| 4 | `long` | `Long` | 64 |
| 5 | `float` | `Float` | 32 |
| 6 | `double` | `Double` | 64 |
| 7 | `char` | `Character` | 16 |
| 8 | `boolean` | `Boolean` | 1 |

> All wrapper classes (and `String`) implement `Comparable<T>` → all have `compareTo(y)`.

### Autoboxing & Unboxing

```java
// Autoboxing: primitive → wrapper
int i = 10;
Integer obj = i;        // autoboxing

// Unboxing: wrapper → primitive
Integer obj2 = 20;
int j = obj2;           // unboxing
```

### Casting (Primitive → Primitive)

```java
int i = (int) 5.7f;          // explicit cast (narrowing)
float f = 10;                // implicit (widening)
float f2 = (float) 5.67;     // explicit (double→float)
int i2 = (int) 100000L;      // explicit (long→int)
```

### String → Wrapper / Primitive

```java
int i = Integer.parseInt("42");          // primitive
float f = Float.parseFloat("42.5");      // primitive
double d = Double.parseDouble("42.50");  // primitive

Integer obj = Integer.valueOf("49");     // wrapper object
```

### Wrapper Useful Methods

- `valueOf(String s)` — convert String to wrapper
- `parse<Type>(String s)` — convert String to primitive
- `toString()` — wrapper → String
- `compareTo(T o)` — `<0`, `0`, `>0`
- `<Type>.MAX_VALUE`, `<Type>.MIN_VALUE`
- `hashCode()`

### Character Class Helpers

```java
Character.isDigit('5');         // true
Character.isLetter('a');        // true
Character.isUpperCase('A');     // true
Character.isLowerCase('a');     // true
Character.toUpperCase('a');     // 'A'
Character.toLowerCase('A');     // 'a'
Character.getNumericValue('5'); // 5
Character.isWhitespace(' ');    // true
Character.isLetterOrDigit('a'); // true
```

---

## Collections Framework

### Hierarchy

```
                Iterable<T>
                    │
                Collection<T>
        ┌───────────┼───────────┐
       List         Set        Queue
        │           │           │
   ArrayList    HashSet      Deque
   LinkedList   LinkedHashSet (PriorityQueue)
   Vector/Stack TreeSet         │
                            ArrayDeque
                            LinkedList (also implements Deque)
```

### Common Collection Methods

```java
Collection<String> names = new ArrayList<>(List.of("Alice", "Bob"));
names.add("Charlie");
names.addAll(List.of("Dave", "Eve"));
names.contains("Alice");
names.containsAll(List.of("Alice", "Bob"));
names.remove("Bob");
names.removeAll(List.of("Dave"));
names.isEmpty();
names.size();
names.toArray(new String[0]);
names.stream();
```

### List Interface

| Method | Purpose |
|--------|---------|
| `get(index)` | Retrieve element at index |
| `set(index, value)` | Replace element |
| `add(index, value)` | Insert (shifts subsequent elements) |
| `remove(index)` | Remove and shift |
| `indexOf(obj)` | Find first index |
| `subList(from, to)` | Slice view |

### LinkedList (also implements `Deque`)

```java
LinkedList<String> list = new LinkedList<>();
list.addFirst("a");
list.addLast("z");
list.removeFirst();
list.removeLast();
list.getFirst();
list.getLast();
list.pollFirst();   // null on empty
list.pollLast();
list.peekFirst();
list.peekLast();
```

### Stack

```java
Stack<Integer> stack = new Stack<>();
stack.push(1);
stack.pop();        // remove top
stack.peek();       // view top
stack.empty();      // is empty
```

> 💡 **Prefer `Deque<Integer> stack = new ArrayDeque<>();`** over `Stack` (legacy synchronized class).

### Set

```
        Set
   ┌─────┼──────┐
HashSet  LinkedHashSet  TreeSet
```

- **HashSet** — O(1) operations, no order
- **LinkedHashSet** — O(1), insertion order preserved
- **TreeSet** — O(log n), sorted order (Red-Black tree)

### Queue

| Method | Behavior on empty/full |
|--------|------------------------|
| `add(e)` | Throws exception if capacity full |
| `offer(e)` | Returns `false` if capacity full |
| `remove()` | Throws exception if empty |
| `poll()` | Returns `null` if empty |
| `peek()` | Returns `null` if empty |

### Deque Methods

```java
Deque<Integer> dq = new ArrayDeque<>();
dq.addFirst(1); dq.addLast(2);
dq.removeFirst(); dq.removeLast();
dq.peekFirst(); dq.peekLast();
```

### List immutable vs mutable

```java
List<String> immutable = List.of("apple");           // immutable (Java 9+)
List<String> mutable = new ArrayList<>(immutable);   // mutable copy
```

### Iteration with Iterator

```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String fruit = it.next();
    // it.remove(); // safe removal during iteration
}
```

---

## Map Deep Dive

### Map Hierarchy

- **HashMap** — O(1), no order, allows one `null` key
- **LinkedHashMap** — O(1), insertion (or access) order
- **TreeMap** — O(log n), sorted by keys (Red-Black tree)
- **ConcurrentHashMap** — thread-safe, segmented locking
- **Hashtable** — legacy synchronized, no nulls allowed

### Core Map Methods

```java
Map<String, Integer> map = new HashMap<>();
map.put("a", 1);
map.putIfAbsent("a", 2);            // Adds only if key not present
map.get("a");
map.getOrDefault("z", 0);           // Default if key missing
map.containsKey("a");
map.containsValue(1);
map.remove("a");
map.remove("a", 1);                 // remove only if value matches
map.clear();
map.size();
map.keySet();                       // Set<K>
map.values();                       // Collection<V>
map.entrySet();                     // Set<Map.Entry<K,V>>
```

### Compute Methods (Java 8+)

```java
// compute(key, BiFunction<K,V,V>) — always called
map.compute("Alice", (k, v) -> (v == null ? 0 : v) + 5);

// computeIfAbsent(key, Function<K,V>) — only if key missing
map.computeIfAbsent("Bob", k -> 75);

// computeIfPresent(key, BiFunction<K,V,V>) — only if key exists
map.computeIfPresent("Ram", (k, v) -> v + 10);

// merge(key, value, BiFunction<V,V,V>) — useful for counters
map.merge("apple", 1, Integer::sum);
```

### Three Ways to Iterate a Map

```java
// 1. keySet
for (String key : map.keySet()) {
    System.out.println(map.get(key));
}

// 2. entrySet
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + " -> " + entry.getValue());
}

// 3. forEach (BiConsumer)
map.forEach((k, v) -> System.out.println(k + " -> " + v));
```

> 📝 **Note:** `map.stream()` is NOT valid. Use `map.entrySet().stream()`.

---

## Comparator & Comparable

### Comparable (natural order)

Implemented by the class itself — defines the **default sort order**.

```java
class Abc implements Comparable<Abc> {
    int age;
    @Override
    public int compareTo(Abc abc) {
        return Integer.compare(this.age, abc.age);
    }
}

PriorityQueue<Abc> pq = new PriorityQueue<>();
pq.add(new Abc(1)); pq.add(new Abc(39));
System.out.println(pq.poll()); // smallest first
```

### Comparator (custom order)

A separate functional interface — define order externally.

```java
@FunctionalInterface
public interface Comparator<T> {
    int compare(T o1, T o2);
}
```

**Common Comparator methods:**

| Method | Purpose |
|--------|---------|
| `compare(o1, o2)` | Core comparison |
| `comparing(keyExtractor)` | Sort by key |
| `thenComparing(...)` | Tie-breaker |
| `reversed()` | Flip order |
| `Comparator.reverseOrder()` | Static reverse natural order |
| `comparingInt/Long/Double` | Avoids boxing |

```java
// Custom order via lambda
PriorityQueue<Abc> pq = new PriorityQueue<>(
    (a, b) -> Integer.compare(a.age, b.age)
);

// Multi-field sort (chained)
Comparator<Abc> cmp = Comparator
    .comparingInt((Abc a) -> a.age)
    .thenComparing(a -> a.name)
    .thenComparing(a -> a.timestamp);

PriorityQueue<Abc> pq2 = new PriorityQueue<>(cmp);
pq2.poll();
```

---

## Streams API

### Stream Pipeline

```
Source → Intermediate Operations* → Terminal Operation
```

### Intermediate Operations (return `Stream`)

| Operation | Purpose |
|-----------|---------|
| `filter(Predicate)` | Keep elements matching predicate |
| `map(Function)` | Transform each element |
| `flatMap(Function)` | Flatten nested streams |
| `distinct()` | Remove duplicates |
| `sorted()` / `sorted(Comparator)` | Order |
| `limit(n)` | Keep first n |
| `skip(n)` | Drop first n |
| `peek(Consumer)` | Side-effect (debugging) |
| `Stream.concat(s1, s2)` | Concatenate streams |

### Terminal Operations

| Operation | Purpose |
|-----------|---------|
| `forEach(Consumer)` | Side-effect on each element |
| `collect(Collector)` | Reduce into a collection |
| `count()` | Number of elements |
| `findFirst()`, `findAny()` | First/any matching |
| `anyMatch`, `allMatch`, `noneMatch` | Boolean tests |
| `min(Comparator)`, `max(Comparator)` | Extremes |
| `reduce(...)` | Custom aggregation |
| `toArray()` | To array |

### `flatMap` — When to Use

If you have `List<List<String>>` and want `List<String>`, `flatMap` flattens it. The mapper function must return a `Stream`.

```java
List<List<String>> nested = List.of(
    List.of("a", "b"), List.of("c", "d")
);
List<String> flat = nested.stream()
    .flatMap(List::stream)
    .collect(Collectors.toList()); // [a, b, c, d]
```

### Get Unique Product Names from Orders

```java
Set<String> productNames = orders.stream()
    .collect(Collectors.mapping(Product::getName, Collectors.toSet()));
```

---

## Collectors

> Collectors internally use the `Collector` interface. Most common: `toList()`, `toSet()`, `toMap()`.

### `toMap()` Full Signature

```java
Collectors.toMap(keyMapper, valueMapper, mergeFunction, mapSupplier)
```

### Word Frequency

```java
List<String> words = Arrays.asList("apple", "banana", "apple", "cherry", "apple");

Map<String, Long> count = words.stream()
    .collect(Collectors.toMap(
        word -> word,                   // key
        word -> 1L,                     // initial value
        (prev, curr) -> prev + curr     // merge function
    ));
// {apple=3, banana=1, cherry=1}
```

### Handling Collisions (Multiple values per key)

```java
Map<Integer, List<String>> byLength = words.stream()
    .collect(Collectors.toMap(
        String::length,
        word -> new ArrayList<>(List.of(word)),
        (existing, replacement) -> {
            existing.addAll(replacement);
            return existing;
        },
        TreeMap::new                    // mapSupplier (TreeMap instead of default HashMap)
    ));
```

### `Collectors.toCollection()`

When you need a specific collection type (LinkedList, TreeSet, PriorityQueue):

```java
LinkedList<String> linked = words.stream()
    .collect(Collectors.toCollection(LinkedList::new));

ConcurrentLinkedQueue<String> queue = words.parallelStream()
    .collect(Collectors.toCollection(ConcurrentLinkedQueue::new));
```

### `Collectors.joining()`

```java
List<String> words = List.of("apple", "banana", "cherry");

words.stream().collect(Collectors.joining());                     // "applebananacherry"
words.stream().collect(Collectors.joining(", "));                 // "apple, banana, cherry"
words.stream().collect(Collectors.joining(", ", "[", "]"));       // "[apple, banana, cherry]"
```

### `Collectors.partitioningBy()` (Boolean partition)

```java
Map<Boolean, Long> partitioned = numbers.stream()
    .collect(Collectors.partitioningBy(num -> num % 2 == 0, Collectors.counting()));
// {false=5, true=4}
```

### `Collectors.groupingBy()` (multi-bucket)

```java
// By length
Map<Integer, List<String>> byLen = words.stream()
    .collect(Collectors.groupingBy(String::length));

// By department
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment));

// With downstream collector — count per department
Map<String, Long> deptCount = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment, Collectors.counting()));
```

### LeetCode #49 — Group Anagrams

```java
String[] strs = {"eat", "tea", "tan", "ate", "nat", "bat"};

Map<String, List<String>> grouped = Arrays.stream(strs)
    .collect(Collectors.groupingBy(s -> {
        char[] arr = s.toCharArray();
        Arrays.sort(arr);
        return new String(arr);
    }));

return new ArrayList<>(grouped.values());
```

---

## Optional

`Optional` is a container object that may or may not contain a non-null value. **Eliminates `NullPointerException`.**

### Creation

```java
Optional<String> opt = Optional.ofNullable("Hello World");
Optional<String> empty = Optional.ofNullable(null);
Optional<String> definite = Optional.of("must not be null"); // throws if null
```

### Common Methods

| # | Method | Description |
|---|--------|-------------|
| 1 | `isPresent()` | true if value present |
| 2 | `isEmpty()` | true if value absent (Java 11+) |
| 3 | `get()` | Get value (throws `NoSuchElementException` if empty) |
| 4 | `orElse(default)` | Value or default |
| 5 | `orElseGet(Supplier)` | Value or supplier-provided default (lazy) |
| 6 | `orElseThrow(Supplier)` | Value or throw custom exception |
| 7 | `filter(Predicate)` | Optional if matches, else empty |
| 8 | `map(Function)` | Transform value |
| 9 | `flatMap(Function)` | Transform when value is itself an Optional |
| 10 | `ifPresent(Consumer)` | Run if present |
| 11 | `ifPresentOrElse(Consumer, Runnable)` | Conditional fork (Java 9+) |
| 12 | `stream()` | Stream of 0 or 1 element |

```java
String name = userOpt
    .filter(u -> u.getAge() > 18)
    .map(User::getName)
    .orElse("Unknown");
```

---

## Object Mapper (Jackson)

Used to handle JSON serialization/deserialization between JSON and Java objects.

```java
ObjectMapper om = new ObjectMapper();

// Deserialization: JSON String → Java object
String json = "{\"name\":\"Alice\",\"age\":30}";
Person person = om.readValue(json, Person.class);

// Serialization: Java object → JSON String
String jsonOut = om.writeValueAsString(person);
```

### List of Objects (TypeReference)

```java
String json = "[{\"name\":\"A\"}, {\"name\":\"B\"}]";
List<Person> people = om.readValue(json, new TypeReference<List<Person>>() {});
```

### Useful Annotations

- `@JsonProperty("custom_name")` — rename JSON field
- `@JsonIgnore` — skip during ser/des
- `@JsonInclude(Include.NON_NULL)` — omit nulls in output

---

## Multithreading & Concurrency

> ⭐ **Critical for senior backend interviews.** Capital One, Microsoft, and most big-tech expect deep knowledge here.

### Thread Lifecycle

`NEW → RUNNABLE → RUNNING → (BLOCKED / WAITING / TIMED_WAITING) → TERMINATED`

### Creating Threads

```java
// 1. Extend Thread
class MyThread extends Thread {
    public void run() { System.out.println("running"); }
}

// 2. Implement Runnable (preferred — keeps inheritance free)
Runnable r = () -> System.out.println("running");
new Thread(r).start();

// 3. Implement Callable (returns a value, throws checked exceptions)
Callable<Integer> c = () -> 42;
```

### Runnable vs Callable

| Aspect | `Runnable` | `Callable<V>` |
|--------|------------|---------------|
| Return value | `void` | Returns `V` |
| Throws checked exception | ❌ | ✅ |
| Used with | `Thread`, `Executor` | `ExecutorService.submit()` |

### Future, Callable, ExecutorService

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

Future<Integer> future = executor.submit(() -> {
    Thread.sleep(1000);
    return 42;
});

// Blocks until result is ready (or timeout)
Integer result = future.get(2, TimeUnit.SECONDS);

// Cancel if needed
future.cancel(true);

executor.shutdown();
```

**Future limitations:**
- `get()` is **blocking**.
- No way to chain dependent computations.
- No callback support.

### CompletableFuture (Java 8+)

`CompletableFuture` solves all the limitations of `Future` — it's **non-blocking**, supports **chaining**, **composition**, **exception handling**, and **callbacks**.

```java
// Run async (no return value)
CompletableFuture.runAsync(() -> System.out.println("hello"));

// Supply async (returns value)
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> "Hello");

// Chaining
CompletableFuture<String> chained = cf
    .thenApply(s -> s + " World")            // transform (sync)
    .thenApplyAsync(s -> s.toUpperCase())    // transform (async)
    .thenAccept(System.out::println);        // consume (no return)

// Combining two futures
CompletableFuture<String> a = CompletableFuture.supplyAsync(() -> "Hello ");
CompletableFuture<String> b = CompletableFuture.supplyAsync(() -> "World");
CompletableFuture<String> combined = a.thenCombine(b, (x, y) -> x + y);

// Run multiple in parallel, wait for all
CompletableFuture<Void> all = CompletableFuture.allOf(future1, future2, future3);

// Wait for first to complete
CompletableFuture<Object> any = CompletableFuture.anyOf(future1, future2);

// Exception handling
cf.exceptionally(ex -> {
    System.err.println(ex.getMessage());
    return "fallback";
});

// Or handle (called whether success or failure)
cf.handle((result, ex) -> ex != null ? "fallback" : result);
```

> 💼 **Interview answer template:** "We use `CompletableFuture` for non-blocking async pipelines — chaining transformations with `thenApply`, combining results with `thenCombine`, and aggregating multiple downstream calls with `allOf`. Compared to `Future`, it gives us proper composition, exception handling via `exceptionally`/`handle`, and avoids blocking threads on `get()`."

### ExecutorService Types

| Factory | Use case |
|---------|----------|
| `newFixedThreadPool(n)` | Fixed pool of n threads |
| `newCachedThreadPool()` | Grows on demand, reuses idle threads |
| `newSingleThreadExecutor()` | Sequential execution |
| `newScheduledThreadPool(n)` | Schedule tasks (delay / fixed-rate) |
| `newWorkStealingPool()` | ForkJoinPool, parallelism = CPU cores |
| `newVirtualThreadPerTaskExecutor()` | **Java 21+ — virtual threads** |

### Atomic Variables

Lock-free thread-safe primitives in `java.util.concurrent.atomic`. Backed by **CAS (Compare-And-Swap)** CPU instructions.

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();   // ++counter, atomically
counter.getAndIncrement();   // counter++, atomically
counter.compareAndSet(5, 10); // if value==5, set to 10, return true

AtomicReference<User> user = new AtomicReference<>();
user.compareAndSet(oldUser, newUser);

// Java 8+: more efficient under high contention
LongAdder adder = new LongAdder();
adder.increment();
adder.sum();
```

### `synchronized` vs Atomic vs Lock

| Mechanism | When to use |
|-----------|-------------|
| `synchronized` | Simple critical sections, low contention |
| `volatile` | Single-variable visibility, no atomicity for compound ops (`i++`) |
| Atomic classes | Single-variable atomic ops (counters, flags) |
| `ReentrantLock` | Need `tryLock`, fairness, or interruptible lock |
| `ReadWriteLock` | Read-heavy workloads |
| `StampedLock` | Optimistic reads (Java 8+) |

### Producer–Consumer Pattern

```java
BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(10);

// Producer
new Thread(() -> {
    try {
        for (int i = 0; i < 100; i++) queue.put(i); // blocks if full
    } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
}).start();

// Consumer
new Thread(() -> {
    try {
        while (true) {
            Integer item = queue.take(); // blocks if empty
            process(item);
        }
    } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
}).start();
```

### Thread-Safe Collections

| Collection | Thread-safe? |
|------------|--------------|
| `ArrayList`, `HashMap`, `HashSet` | ❌ No |
| `Vector`, `Hashtable`, `Stack` | ✅ (legacy, fully synchronized) |
| `Collections.synchronizedList(list)` | ✅ (wrapper, lock entire collection) |
| `ConcurrentHashMap` | ✅ (segment locking, **preferred**) |
| `CopyOnWriteArrayList` | ✅ (read-heavy, expensive writes) |
| `ConcurrentLinkedQueue`, `LinkedBlockingQueue` | ✅ |

### Common Concurrency Pitfalls

- **Race conditions** — multiple threads modifying shared state without sync.
- **Deadlock** — two locks, two threads, opposite acquisition order. Avoid by acquiring locks in a consistent order.
- **Livelock** — threads keep responding to each other but make no progress.
- **Starvation** — low-priority threads never get CPU time.
- **Memory visibility** — without `volatile` / sync, one thread may not see another's writes due to CPU caching.

---

## Java 17 Features

Java 17 is an **LTS** release. Key features you'll see in interviews:

### 1. Sealed Classes (final in 17)

Restrict which classes can extend a class.

```java
public sealed class Shape permits Circle, Square, Triangle {}

public final class Circle extends Shape {}
public final class Square extends Shape {}
public final class Triangle extends Shape {}
```

> Useful for closed type hierarchies (algebraic data types). Pairs well with pattern matching.

### 2. Records (since 16, mature in 17)

Compact immutable data carrier classes.

```java
public record Point(int x, int y) {}

// Auto-generates: constructor, accessors x()/y(), equals, hashCode, toString
Point p = new Point(1, 2);
p.x(); // 1
```

> **Use case:** DTOs, value objects. Replaces Lombok `@Value` for many cases.

### 3. Pattern Matching for `instanceof`

```java
// Pre-Java 16
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

// Java 17
if (obj instanceof String s) {
    System.out.println(s.length()); // s is auto-cast
}
```

### 4. Switch Expressions (mature in 14, polished in 17)

```java
String day = switch (dayNum) {
    case 1, 2, 3, 4, 5 -> "Weekday";
    case 6, 7 -> "Weekend";
    default -> throw new IllegalArgumentException();
};
```

### 5. Text Blocks

```java
String json = """
    {
        "name": "Alice",
        "age": 30
    }
    """;
```

### 6. `var` (since Java 10) — Local-Variable Type Inference

```java
var list = new ArrayList<String>();   // inferred as ArrayList<String>
var map = Map.of("a", 1, "b", 2);
```

> **Don't overuse `var`** — only when type is obvious from RHS. Avoid for primitives like `var x = 10` (becomes `int`, but it hurts readability).

---

## Java 21 Features

Java 21 is the latest **LTS** (Sept 2023). Highly likely interview topic in 2026.

### 1. ⭐ Virtual Threads (Project Loom)

The biggest feature: lightweight threads scheduled by the JVM (not OS).

```java
// Old way: 1 platform thread = 1 OS thread (limited to ~10k)
ExecutorService old = Executors.newFixedThreadPool(200);

// New way: virtual threads — millions are cheap
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 1_000_000; i++) {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return null;
        });
    }
}

// Or directly:
Thread.startVirtualThread(() -> System.out.println("hello"));

Thread vt = Thread.ofVirtual().name("worker").start(runnable);
```

**Why this matters:**
- For I/O-bound apps (typical microservices), virtual threads remove the thread-pool sizing problem.
- A blocked virtual thread doesn't pin an OS thread → massive concurrency.
- **Same `Thread` API** — no rewrite needed.
- ⚠️ Don't use `synchronized` blocks that hold a lock during I/O — they pin the carrier thread (use `ReentrantLock` instead).

> 💼 **Interview talking point:** "Virtual threads let us write blocking-style code that scales like reactive code — without the cognitive overhead of `Mono`/`Flux`. For most I/O-bound services, this means we can stick with imperative code and still hit the throughput we'd previously have needed WebFlux for."

### 2. Pattern Matching for `switch` (final in 21)

```java
sealed interface Shape permits Circle, Square, Triangle {}

double area = switch (shape) {
    case Circle c    -> Math.PI * c.radius() * c.radius();
    case Square s    -> s.side() * s.side();
    case Triangle t  -> 0.5 * t.base() * t.height();
};
```

With **record patterns**:

```java
double area = switch (shape) {
    case Circle(double r)             -> Math.PI * r * r;
    case Square(double s)             -> s * s;
    case Triangle(double b, double h) -> 0.5 * b * h;
};
```

### 3. Sequenced Collections

New unified interfaces: `SequencedCollection`, `SequencedSet`, `SequencedMap`.

```java
List<String> list = List.of("a", "b", "c");
list.getFirst();    // "a"
list.getLast();     // "c"
list.reversed();    // reversed view
```

### 4. String Templates (Preview)

```java
String name = "Alice";
String greeting = STR."Hello \{name}!";
```

### 5. Other Notable Features

- **Scoped Values** (preview) — better than ThreadLocal for virtual threads.
- **Structured Concurrency** (preview) — group related tasks, fail-fast on errors.
- **Generational ZGC** — better GC pause times.

---

## Top 50 Interview Q&A

### 🟢 Core Java

**Q1. Why is Java platform-independent?**

Java compiles source to platform-neutral bytecode. JVMs (one per OS) interpret the same bytecode → "Write Once, Run Anywhere".

**Q2. JDK vs JRE vs JVM.**

JDK ⊃ JRE ⊃ JVM. JVM runs bytecode, JRE = JVM + libraries (runtime), JDK = JRE + dev tools (`javac`, `javadoc`).

**Q3. Difference between `==` and `.equals()`?**

`==` compares references for objects, values for primitives. `.equals()` compares logical content (must be overridden in custom classes; default Object.equals == ==).

**Q4. Why must `equals()` and `hashCode()` be overridden together?**

Hash-based collections (`HashMap`, `HashSet`) use `hashCode()` to find the bucket and `equals()` to compare within it. If two objects are equal but have different hashCodes, the map breaks.

**Q5. Why is String immutable?**

Security, string pool optimization, thread safety, hashCode consistency, memory efficiency.

**Q6. String vs StringBuilder vs StringBuffer.**

| | String | StringBuilder | StringBuffer |
|-|-|-|-|
| Mutability | ❌ | ✅ | ✅ |
| Thread Safe | ✅ | ❌ | ✅ |
| Speed | Slow | Fastest | Slower (sync) |

**Q7. Can you have `try` without `catch`?**

Yes — `try` with `finally` (no catch). But not `try` alone.

**Q8. Can a constructor be `final`?**

No. Constructors aren't inherited, so `final` is meaningless.

**Q9. Can a static method be overridden?**

No — static methods are **hidden**, not overridden. Resolved at compile time using the reference type.

**Q10. Override `private` method?**

Not really — child gets a separate method. Calling via parent reference invokes the parent's private method (compile error if accessed externally).

**Q11. What is method hiding?**

When a child class declares a static method with the same signature as the parent. The method called depends on the reference type, not the runtime object (no dynamic dispatch).

**Q12. Can an abstract class have a constructor?**

Yes. Used by subclasses via `super()`.

**Q13. Can we instantiate an abstract class?**

Not directly. But via:
1. Subclass instance.
2. Anonymous inner class providing implementations.

**Q14. How do you modify a `final` reference variable?**

You **can** mutate the object it points to (e.g., add to a final List), but you **cannot** reassign the reference.

**Q15. Singleton class — how to make thread-safe?**

Double-checked locking with `volatile`, or use enum singleton (best — handles serialization too):

```java
public enum Singleton { INSTANCE; }
```

### 🟢 Collections

**Q16. ArrayList vs LinkedList.**

- `ArrayList` — backed by array, O(1) random access, O(n) middle insert.
- `LinkedList` — doubly-linked, O(1) head/tail ops, O(n) random access. Implements `Deque`.

**Q17. HashMap internal working.**

Array of buckets indexed by `hash(key) & (n-1)`. Each bucket is a linked list (Java 7) or balanced tree (Java 8+ when bucket size ≥ 8). Default initial capacity 16, load factor 0.75 → resize doubles capacity and rehashes.

**Q18. HashMap vs Hashtable vs ConcurrentHashMap.**

| | HashMap | Hashtable | ConcurrentHashMap |
|-|-|-|-|
| Thread-safe | ❌ | ✅ (full sync) | ✅ (segment/bucket locks) |
| Null key | 1 allowed | ❌ | ❌ |
| Performance | Best | Worst | Good under contention |

**Q19. HashSet internal working.**

Backed by `HashMap` with a dummy `PRESENT` value.

**Q20. fail-fast vs fail-safe iterators?**

- **fail-fast** (`ArrayList`, `HashMap`) — throws `ConcurrentModificationException` if collection is modified during iteration.
- **fail-safe** (`CopyOnWriteArrayList`, `ConcurrentHashMap`) — iterates over a snapshot or supports concurrent modification.

**Q21. Comparable vs Comparator.**

- `Comparable` — implemented by class, defines natural order, `compareTo(this, that)`.
- `Comparator` — separate class/lambda, defines custom order, `compare(o1, o2)`.

### 🟢 Streams & Functional

**Q22. Difference between `map` and `flatMap`.**

`map` transforms 1→1. `flatMap` transforms 1→many and flattens.

**Q23. Intermediate vs terminal operations.**

Intermediate ops are **lazy** and return a Stream. Terminal ops trigger pipeline execution and return a result (or `void`).

**Q24. Why are streams lazy?**

Performance — operations are fused and short-circuited. `findFirst()` after `filter` won't process the entire stream.

**Q25. Can you reuse a stream?**

No. A stream can be consumed only once. `IllegalStateException: stream has already been operated upon or closed`.

**Q26. parallelStream() — when to use?**

For CPU-bound, large datasets, side-effect-free pipelines. Avoid for small data, I/O-bound work, or order-sensitive ops.

### 🟢 Concurrency

**Q27. Runnable vs Callable.**

`Runnable` returns void, can't throw checked exceptions. `Callable<V>` returns value, can throw checked.

**Q28. Future vs CompletableFuture.**

`Future.get()` blocks. `CompletableFuture` supports non-blocking chaining (`thenApply`, `thenCombine`), exception handling (`exceptionally`), and composition (`allOf`, `anyOf`).

**Q29. How would you call multiple downstream services in parallel?**

```java
CompletableFuture<User> u = CompletableFuture.supplyAsync(() -> userClient.get(id));
CompletableFuture<Account> a = CompletableFuture.supplyAsync(() -> accountClient.get(id));
CompletableFuture<Profile> p = CompletableFuture.supplyAsync(() -> profileClient.get(id));

CompletableFuture<Void> all = CompletableFuture.allOf(u, a, p);
all.join();

return new Combined(u.join(), a.join(), p.join());
```

For reactive code, use `Mono.zip(...)` (see Spring Boot notes).

**Q30. What is the difference between `volatile` and `synchronized`?**

`volatile` — visibility (writes are visible to all threads), no atomicity for compound ops.
`synchronized` — atomicity + visibility, but acquires monitor lock.

**Q31. Why use Atomic classes over `synchronized`?**

Atomic classes use **CAS** (lock-free) — better under high contention for simple counter/flag updates.

**Q32. What is a deadlock? How do you avoid it?**

Two threads waiting for each other's locks indefinitely. Avoid by:
- Acquiring locks in a consistent global order.
- Using `tryLock(timeout)`.
- Avoiding nested locks where possible.

**Q33. Thread pool sizing rule of thumb.**

- CPU-bound: `N_threads = N_cores + 1`
- I/O-bound: `N_threads = N_cores * (1 + waitTime/computeTime)`
- Java 21 virtual threads: don't size — use `newVirtualThreadPerTaskExecutor()`.

**Q34. What is ThreadLocal?**

Per-thread storage. Used for request context (e.g., user ID, transaction ID). **⚠️ Always remove after use** to avoid memory leaks in pooled threads.

### 🟢 Java 17 / 21

**Q35. What is a Record?**

Immutable data class with auto-generated constructor, accessors, equals/hashCode/toString. Cannot extend other classes (always extends `Record`).

**Q36. Sealed classes — why use them?**

Restrict inheritance to a known set. Enables exhaustive switch pattern matching → compiler errors when a new subtype is added but a switch isn't updated.

**Q37. What are virtual threads?**

JVM-managed lightweight threads (Project Loom, Java 21). Many virtual threads multiplex on few OS carrier threads. Cheap to create (millions feasible). Ideal for I/O-bound microservices.

**Q38. Virtual threads vs reactive (WebFlux)?**

Virtual threads let you keep imperative blocking code with reactive-level scalability. WebFlux is still useful for backpressure, streaming, and event-driven flows. For typical request/response I/O-bound services, virtual threads are simpler.

**Q39. Pattern matching for switch — example?**

```java
String result = switch (obj) {
    case Integer i when i > 0 -> "positive int: " + i;
    case Integer i           -> "non-positive int";
    case String s            -> "string: " + s;
    case null                -> "null";
    default                  -> "other";
};
```

**Q40. Text blocks — when to use?**

Multi-line strings (JSON, SQL, HTML). Preserves indentation relative to the closing `"""`.

### 🟢 Coding Patterns

**Q41. Group anagrams (LeetCode 49).**

Group strings by sorted-character signature using `Collectors.groupingBy`.

**Q42. Word frequency from a list.**

```java
words.stream().collect(Collectors.toMap(w -> w, w -> 1L, Long::sum));
// or:
words.stream().collect(Collectors.groupingBy(w -> w, Collectors.counting()));
```

**Q43. Find Nth highest salary.**

```java
employees.stream()
    .map(Employee::getSalary)
    .distinct()
    .sorted(Comparator.reverseOrder())
    .skip(n - 1)
    .findFirst();
```

**Q44. Sort employees by department then by name.**

```java
employees.sort(
    Comparator.comparing(Employee::getDepartment)
              .thenComparing(Employee::getName)
);
```

**Q45. Partition list of numbers into even & odd.**

```java
Map<Boolean, List<Integer>> parts = numbers.stream()
    .collect(Collectors.partitioningBy(n -> n % 2 == 0));
```

**Q46. Reverse a string with a Stream.**

```java
new StringBuilder(input).reverse().toString();
// or via stream of chars
```

**Q47. Find duplicates in a List.**

```java
Set<String> seen = new HashSet<>();
Set<String> dupes = words.stream()
    .filter(w -> !seen.add(w))
    .collect(Collectors.toSet());
```

**Q48. Implement an LRU cache.**

```java
class LRUCache<K,V> extends LinkedHashMap<K,V> {
    private final int capacity;
    LRUCache(int cap) { super(cap, 0.75f, true /*accessOrder*/); this.capacity = cap; }
    protected boolean removeEldestEntry(Map.Entry<K,V> e) { return size() > capacity; }
}
```

(Or build from `HashMap` + doubly-linked list for O(1) get/put.)

**Q49. Producer-consumer with BlockingQueue.**

See Concurrency section above. `BlockingQueue.put()` blocks producers when full; `take()` blocks consumers when empty.

**Q50. How would you shut down an ExecutorService cleanly?**

```java
executor.shutdown(); // refuse new tasks
try {
    if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
        executor.shutdownNow(); // interrupt running tasks
    }
} catch (InterruptedException e) {
    executor.shutdownNow();
    Thread.currentThread().interrupt();
}
```

---

## 🎯 Final Tips for Interviews

- **Always think aloud.** State the trade-offs (time/space, mutability, thread safety) before writing code.
- **Edge cases first** — null, empty, single element, very large input, concurrent access.
- **Big-O on the spot.** Have hashtable/tree/heap complexities memorized.
- **Concurrency war stories.** If you've used `CompletableFuture`, atomics, or thread pools in production, lead with that.
- **Java 21 virtual threads** — be ready to explain why your team would (or wouldn't) adopt them.

---

*Notes compiled from personal handwritten study material + interview prep additions. Last updated April 2026.*
