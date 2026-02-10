# Why Enums Matter in Rust - Part 1

Rust is often considered a hard language to master. My goal here is to make it accessible — one small bite at a time.

## Java Enums
If you’re coming from a Java background, this is how you might define an enum:

```
enum Shape {
    CIRCLE(5.0),
    RECTANGLE(5.0, 4.0);

    private final double width;
    private final double height;
    private final double radius;

    Shape(double radius) {
        this.radius = radius;
        this.width = 0;
        this.height = 0;
    }

    Shape(double width, double height) {
        this.width = width;
        this.height = height;
        this.radius = 0;
    }
}
```

At first glance, this looks perfectly reasonable. But let’s pause and look at what Java is _actually_ doing.

### Some quick observations:
1. **Each enum value is an object**
   - `CIRCLE` and `RECTANGLE` are `public static final` singleton instances of `Shape`.
   - If an enum constant defines its own behavior (for example, a custom `area()` method), it becomes an instance of an anonymous subclass of `Shape`.

2. **Enum instances live on the heap**
   - Like all objects in Java, enum instances are allocated on the heap.
   - They are created when the enum class is initialized and live for the lifetime of the class loader.

3. **All fields exist on every enum object**
   - Even though `width` and `height` have no meaning for a circle, your `CIRCLE` object still stores them — unnecessarily.

So `Shape.CIRCLE` is not just a value.  It is a full object with a fixed memory layout.
***
### The System Cost: Object Bloat

#### What does “object” really mean?
When Java creates an object, it doesn’t just store your data. It also stores bookkeeping information that the JVM needs to manage that object.

This includes things like:
- metadata for garbage collection
- information about which class this object belongs to
- data used for locking and synchronization

All of this lives along with your actual fields and cannot be removed.

#### The "Kitchen Sink" Problem:
Now let's come back to enums.

Even though `CIRCLE` has no use for `width` and `height`, it carries them around anyway. You asked for a glass of water but Java just handed you the entire kitchen sink.

**Estimated Size of a Java `CIRCLE`:**
On a typical 64-bit JVM, a single enum instance looks like this:

| Component         | Size (Approx) | Notes                               |
| ----------------- | ------------- | ----------------------------------- |
| **Object Header** | 12 bytes      | JVM bookkeeping (GC, locking, type) |
| **radius**        | 8 bytes       | Useful data                         |
| **width**         | 8 bytes       | **WASTED** (0.0)                    |
| **height**        | 8 bytes       | **WASTED** (0.0)                    |
| **Padding**       | 4 bytes       | To align to 8-byte boundary         |
| **Total**         | **40 Bytes**  |                                     |
**Note:** Exact sizes vary by JVM, flags, and architecture.

#### What this means at runtime 
To access this data, the CPU has to:
1. Load a reference from the stack
2. Chase that reference to the heap (potential cache miss)
3. Read a ~40-byte object, 16 bytes of which are zeros that mean nothing

This implies:
- extra memory per enum instance
- standard object overhead (headers, alignment)
- pointer indirection before the actual data can be read

As we can see, Java enums are powerful — but they are heavy.
***

## Rust Enums

### Rust enums: same word, different idea

Now let’s look at Rust.

The syntax may look familiar, but the underlying model is completely different.

```
enum Shape {
    Circle(f64),
    Rectangle { width: f64, height: f64 },
}
```

So what changed?
#### How Rust enums are different (one idea at a time)

1. **Enums are value-based, not object-based**
   - A `Shape` is a single value with a specific variant like `Circle`.  
   - There are no pre-existing objects for `Circle` or `Rectangle`.

2. **Rust enums are sum types (also called tagged unions)**
    A _sum type_ means:
    - a value can be one of several variants
    - but only one variant at a time
    Think:
    > “This value is A OR B OR C — never more than one.”
    
3. **Each enum value stores only the data for its active variant**  
   - A `Circle` variant stores only a radius.  
   - It does not carry `width` or `height`.

No wasted fields. No object bloat.
***
#### How Rust enums use memory
Rust asks a simple question: *“What is the largest variant?”*

In this case:
- `Circle(f64)` → 8 bytes
- `Rectangle { f64, f64 }` → 16 bytes

Rust then:
- allocates enough space for the largest variant
- adds a small discriminant (a tag to identify which variant is active)
- applies alignment and padding as needed

So the enum’s size is roughly:
`max(variant size) + discriminant + padding`

On a typical 64-bit system, this usually becomes:
- 1 byte for the tag
- 7 bytes of padding
- 16 bytes for `Rectangle` data

**If it's a `Rectangle` (Largest Variant):**

```
| Tag (1 byte) | Padding | width (8 bytes) | height (8 bytes) |
|--------------|---------|-----------------|------------------|
|      1       | ....... |    width_val    | height_val  |
```

_Total Size:_ 24 Bytes.

**If it's a `Circle` (Smaller Variant):**

```
| Tag (1 byte) | Padding | radius (8 bytes) | UNUSED (8 bytes) |
|--------------|---------|------------------|------------------|
|      0       | ....... |    radius_val    |   (raw garbage)  |
```

_Total Size:_ 24 Bytes.

(**Note**: Exact size is platform- and compiler-dependent.)

Importantly, this data is stored inline:
- if `Shape` is a local variable, it lives on the stack
- if it’s a field in a struct, it’s stored inline in that struct
- if it’s inside a `Vec<Shape>`, the enum values are stored inline in the vector’s contiguous buffer

Heap allocation only occurs if the data inside a variant allocates (for example `Box`, `String`, or `Vec`), not because the enum itself requires it.

***
#### Rust enums are 'sealed'
Rust enums are *closed* (often described as *sealed*). In this sense, they’re similar to Java enums.

This means:
1. All possible variants are defined in one place
2. No new variants can be added elsewhere
3. The compiler knows every possible variant

This property enables something that’s central to Rust—but much stronger than what Java enforces.

**Pattern matching**

```
match my_shape {
     Shape::Circle(r) => println!("Area is {}", 3.14 * r * r),
     Shape::Rectangle { width, height } => println!("Area is {}", width * height), 
}
```

Rust forces exhaustive pattern matching. If you forget a variant, the program does not compile:

```
match my_shape {
     Shape::Circle(r) => println!("Area {}", r * r * 3.14),
}
```

Rust refuses to guess.

In Java, similar logic often compiles and fails later at runtime. Rust moves this check to compile time. You can refactor fearlessly.

#### Why all this matters?

1. **Memory efficiency**  
    Only the data that matters exists.  
    No unused fields. No extra baggage.
2. **Fewer indirections**  
    Data is stored directly, often inline.  
    This is cache-friendly and predictable.
3. **Safety at compile time**  
    Missing cases are caught early.  
    No silent fall-throughs. No runtime surprises.

So now, the big idea:
- Java enums are objects with behavior.
- Rust enums are precise descriptions of data.

Same word. Very different philosophy.