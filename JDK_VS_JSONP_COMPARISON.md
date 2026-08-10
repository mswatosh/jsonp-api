# JDK Incubator JSON API vs. Jakarta JSON-P: Comparison & Contrast

> **Source A:** `jdk.incubator.json` — JDK 28 incubator module (draft)  
> **Source B:** `jakarta.json` — Jakarta JSON Processing (JSON-P) API in this repository  
> **Branch:** `jdkson`  
> **Standard:** Both target [RFC 8259](https://www.rfc-editor.org/rfc/rfc8259) (JSON data interchange format)

---

## 1. Design Philosophy

| Dimension | `jdk.incubator.json` | `jakarta.json` (JSON-P) |
|---|---|---|
| **Scope** | Intentionally minimal: parse, navigate, construct, serialise only | Comprehensive: streaming, object model, readers/writers, builders, pointer/patch operations |
| **Target use case** | Simple, dependency-free JSON access from within the JDK | Full-featured JSON processing for Jakarta EE and standalone applications |
| **API surface size** | 9 types total (6 value types + `Json` + 2 exceptions) | 37+ types across 3 packages (`jakarta.json`, `jakarta.json.spi`, `jakarta.json.stream`) |
| **Configuration** | None — no configuration model | `JsonConfig` with `KEY_STRATEGY` enum; provider-level config maps on all factories |
| **Provider model** | No SPI — single JDK-internal implementation | `JsonProvider` SPI with discovery via system property, `ServiceLoader`, or OSGi |
| **Module** | `jdk.incubator.json` (incubator, must be added with `--add-modules`) | `jakarta.json` (standard module — see [`module-info.java`](api/src/main/java/module-info.java:20)); currently requires `jdk.incubator.json` on this branch |

---

## 2. Type Hierarchy Comparison

### JDK Incubator JSON

```
JsonValue              (sealed interface — root)
├── JsonObject         (non-sealed interface)
├── JsonArray          (non-sealed interface)
├── JsonString         (non-sealed interface)
├── JsonNumber         (non-sealed interface)
├── JsonBoolean        (non-sealed interface)
└── JsonNull           (non-sealed interface)

Json                   (final class — parse / display entry points)
JsonParseException     (final class extends RuntimeException)
JsonValueException     (final class extends RuntimeException)
```

### Jakarta JSON-P

```
JsonValue              (interface — root)
├── JsonStructure      (interface — container supertype)
│   ├── JsonObject     (interface extends JsonStructure, Map<String,JsonValue>)
│   └── JsonArray      (interface extends JsonStructure, List<JsonValue>)
├── JsonString         (interface)
└── JsonNumber         (interface)
    [TRUE / FALSE / NULL represented as JsonValue constants]

Json                   (final class — factory/utility entry points)
JsonProvider           (abstract class — SPI root)
JsonObjectBuilder      (interface)
JsonArrayBuilder       (interface)
JsonBuilderFactory     (interface)
JsonReader             (interface extends Closeable)
JsonWriter             (interface extends Closeable)
JsonReaderFactory      (interface)
JsonWriterFactory      (interface)
JsonPointer            (interface — RFC 6901)
JsonPatch              (interface — RFC 6902)
JsonPatchBuilder       (interface)
JsonMergePatch         (interface — RFC 7396)
JsonConfig             (final class — config constants)
JsonException          (class extends RuntimeException)

jakarta.json.stream:
├── JsonParser         (interface extends Closeable)
├── JsonGenerator      (interface extends Flushable, Closeable)
├── JsonParserFactory  (interface)
├── JsonGeneratorFactory (interface)
├── JsonLocation       (interface)
├── JsonCollectors     (final class — Stream Collector utilities)
├── JsonParsingException (class extends JsonException)
└── JsonGenerationException (class extends JsonException)
```

### Key Structural Differences

**Boolean and Null representation:**  
`jdk.incubator.json` has dedicated [`JsonBoolean`](JDK_JSON_API_RESEARCH.md) and [`JsonNull`](JDK_JSON_API_RESEARCH.md) interfaces as first-class members of the sealed hierarchy. JSON-P represents `true`, `false`, and `null` as three static singleton constants on [`JsonValue`](api/src/main/java/jakarta/json/JsonValue.java:89): `JsonValue.TRUE`, `JsonValue.FALSE`, and `JsonValue.NULL`. There are no `JsonBoolean` or `JsonNull` types in JSON-P.

**Container hierarchy:**  
JSON-P introduces [`JsonStructure`](api/src/main/java/jakarta/json/JsonStructure.java:23) as an intermediate interface between `JsonValue` and the two container types, adding JSON Pointer navigation (`getValue(String jsonPointer)`). The JDK API has no equivalent intermediate type.

**Sealed vs. open hierarchy:**  
`jdk.incubator.json` uses a `sealed interface` for `JsonValue`, enabling exhaustive pattern-matching in `switch` expressions — a direct benefit of modern Java language features. JSON-P predates sealing and uses a plain interface, discriminated by a `ValueType` enum returned from [`getValueType()`](api/src/main/java/jakarta/json/JsonValue.java:106).

**`Map`/`List` relationship:**  
JSON-P's [`JsonObject`](api/src/main/java/jakarta/json/JsonObject.java:104) extends `Map<String, JsonValue>` and [`JsonArray`](api/src/main/java/jakarta/json/JsonArray.java:93) extends `List<JsonValue>`, making them directly usable anywhere a `Map` or `List` is expected (mutations throw `UnsupportedOperationException`). The JDK API's types do not implement these collection interfaces; they only expose values through [`asMap()`](JDK_JSON_API_RESEARCH.md) and [`asList()`](JDK_JSON_API_RESEARCH.md), which return unmodifiable collection views.

---

## 3. Value Model API Comparison

### 3.1 `JsonString`

| Feature | `jdk.incubator.json.JsonString` | `jakarta.json.JsonString` |
|---|---|---|
| Get Java value | [`asString()`](JDK_JSON_API_RESEARCH.md) | [`getString()`](api/src/main/java/jakarta/json/JsonString.java:29) |
| CharSequence access | Not available | [`getChars()`](api/src/main/java/jakarta/json/JsonString.java:37) |
| `toString()` behavior | Returns **JSON-encoded** quoted+escaped form | Returns JSON-encoded form (same) |
| Construction | [`JsonString.of(String)`](JDK_JSON_API_RESEARCH.md) | [`Json.createValue(String)`](api/src/main/java/jakarta/json/Json.java:433) |
| `equals`/`hashCode` | Not specified in the sealed interface | Defined on content: `getString()` equality |

### 3.2 `JsonNumber`

| Feature | `jdk.incubator.json.JsonNumber` | `jakarta.json.JsonNumber` |
|---|---|---|
| `int` extraction | [`asInt()`](JDK_JSON_API_RESEARCH.md) — throws `JsonValueException` if fractional or out-of-range | [`intValue()`](api/src/main/java/jakarta/json/JsonNumber.java:69) (lossy) / [`intValueExact()`](api/src/main/java/jakarta/json/JsonNumber.java:79) (throws `ArithmeticException`) |
| `long` extraction | [`asLong()`](JDK_JSON_API_RESEARCH.md) — throws on fraction or overflow | [`longValue()`](api/src/main/java/jakarta/json/JsonNumber.java:89) (lossy) / [`longValueExact()`](api/src/main/java/jakarta/json/JsonNumber.java:99) (throws) |
| `double` extraction | [`asDouble()`](JDK_JSON_API_RESEARCH.md) — throws if result is `±Infinity` | [`doubleValue()`](api/src/main/java/jakarta/json/JsonNumber.java:133) — may lose precision, no overflow throw |
| `BigDecimal` extraction | Via `toString()` + `new BigDecimal(…)` (not a direct method) | [`bigDecimalValue()`](api/src/main/java/jakarta/json/JsonNumber.java:140) (direct) |
| `BigInteger` extraction | Not provided | [`bigIntegerValue()`](api/src/main/java/jakarta/json/JsonNumber.java:111) / [`bigIntegerValueExact()`](api/src/main/java/jakarta/json/JsonNumber.java:121) |
| `Number` extraction | Not provided | [`numberValue()`](api/src/main/java/jakarta/json/JsonNumber.java:149) (since 1.1; default throws `UnsupportedOperationException`) |
| Integral check | Not provided | [`isIntegral()`](api/src/main/java/jakarta/json/JsonNumber.java:59) |
| Internal representation | Raw string (arbitrary precision, lazy conversion) | `BigDecimal`-based semantics |
| Construction | [`JsonNumber.of(int/long/double/String)`](JDK_JSON_API_RESEARCH.md) | [`Json.createValue(int/long/double/BigDecimal/BigInteger/Number)`](api/src/main/java/jakarta/json/Json.java:445) |
| `equals`/`hashCode` | Not specified in interface | Defined on `bigDecimalValue()` equality |

> **Key difference:** JSON-P provides both lossless (`exact`) and lossy extraction paths, plus direct `BigDecimal` and `BigInteger` accessors. The JDK API's `asInt()`/`asLong()` are effectively "exact-only" (throw on imprecision) while `asDouble()` allows precision loss but guards against infinities. JSON-P has no built-in infinity guard on `doubleValue()`.

### 3.3 `JsonObject` — Access Patterns

JSON-P provides rich, typed accessors directly on [`JsonObject`](api/src/main/java/jakarta/json/JsonObject.java) for the common case:

```java
// JSON-P: typed accessors with null-safe defaults
String name = jsonObject.getString("name");                  // throws NPE if absent
String name = jsonObject.getString("name", "unknown");       // returns default if absent
int    age  = jsonObject.getInt("age", 0);
boolean ok  = jsonObject.getBoolean("ok", false);
boolean nil = jsonObject.isNull("field");
```

The JDK API uses a uniform navigation model on [`JsonValue`](JDK_JSON_API_RESEARCH.md) with a split between throwing and `Optional`-returning variants:

```java
// JDK: explicit navigation + terminal conversion
String name = root.get("name").asString();                   // throws on absent
Optional<String> maybeName = root.tryGet("name")
                                 .map(v -> v.asString());    // Optional on absent
```

JSON-P's default-value accessors (`getString(name, defaultValue)`) handle both absent keys and wrong types silently. The JDK API has no equivalent — `get(String)` always throws on absence and any type mismatch throws `JsonValueException`.

### 3.4 `JsonArray` — Access Patterns

JSON-P's [`JsonArray`](api/src/main/java/jakarta/json/JsonArray.java) extends `List<JsonValue>` directly, making it compatible with all `java.util.List` APIs and Stream operations. It additionally offers:
- [`getValuesAs(Class<T>)`](api/src/main/java/jakarta/json/JsonArray.java:155) — typed list view (unsafe cast, `ClassCastException` deferred)
- [`getValuesAs(Function<K,T>)`](api/src/main/java/jakarta/json/JsonArray.java:177) — transform via function (since 1.1)
- Typed index accessors (`getJsonObject(int)`, `getString(int)`, etc.)

The JDK API's `JsonArray` is not a `List`. Access is only through `get(int)` (throws on bounds) or `asList()` which returns an unmodifiable `List<JsonValue>`.

---

## 4. Construction API Comparison

### JDK Incubator JSON — Static `of(…)` Factories

Construction uses static factory methods directly on value types. There are no builder objects:

```java
JsonObject doc = JsonObject.of(Map.of(
    "name",    JsonString.of("Alice"),
    "age",     JsonNumber.of(30),
    "active",  JsonBoolean.of(true),
    "note",    JsonNull.of()
));
JsonArray arr = JsonArray.of(List.of(JsonString.of("a"), JsonNumber.of(1)));
```

**Constraint:** Since `Map.of(…)` with more than 10 entries requires `Map.entry(…)` or `Map.ofEntries(…)`, constructing large objects is less ergonomic. There is also no incremental/builder approach.

### Jakarta JSON-P — Builder Pattern

JSON-P uses dedicated builder interfaces ([`JsonObjectBuilder`](api/src/main/java/jakarta/json/JsonObjectBuilder.java), [`JsonArrayBuilder`](api/src/main/java/jakarta/json/JsonArrayBuilder.java)) with fluent `add(…)` chains. The builders accept raw Java types directly, eliminating explicit wrapping:

```java
// JSON-P: no explicit JsonString.of / JsonNumber.of wrapping needed
JsonObject doc = Json.createObjectBuilder()
    .add("name",   "Alice")
    .add("age",    30)
    .add("active", true)
    .addNull("note")
    .build();

JsonArray arr = Json.createArrayBuilder()
    .add("a")
    .add(1)
    .build();
```

JSON-P builders also support:
- Nesting builders: `add("child", Json.createObjectBuilder().add(…))`
- Initialising from an existing `JsonObject`/`JsonArray` (since 1.1): [`createObjectBuilder(JsonObject)`](api/src/main/java/jakarta/json/Json.java:289)
- Initialising from a `Map<String,?>` or `Collection<?>` (since 1.1): [`createObjectBuilder(Map)`](api/src/main/java/jakarta/json/Json.java:305)
- Positional `add(int, …)`, `set(int, …)`, `remove(int)` on [`JsonArrayBuilder`](api/src/main/java/jakarta/json/JsonArrayBuilder.java) (since 1.1)
- `remove(String)` on [`JsonObjectBuilder`](api/src/main/java/jakarta/json/JsonObjectBuilder.java:269) (since 1.1)
- `addAll(JsonObjectBuilder)` / `addAll(JsonArrayBuilder)` (since 1.1)

> **Key difference:** JSON-P builders are more ergonomic for incremental and complex construction, especially when mixing in existing structures. The JDK API's `of(…)` approach is more concise for simple, known-at-construction-time documents.

---

## 5. Parsing and I/O Comparison

### 5.1 Parsing

| Feature | `jdk.incubator.json` | `jakarta.json` (JSON-P) |
|---|---|---|
| Parse from `String` | [`Json.parse(String)`](JDK_JSON_API_RESEARCH.md) | Not directly — use `JsonReader` wrapping a `StringReader` |
| Parse from `char[]` | [`Json.parse(char[])`](JDK_JSON_API_RESEARCH.md) | Not directly |
| Parse from `Reader` | Not available | [`Json.createReader(Reader)`](api/src/main/java/jakarta/json/Json.java:191) → `JsonReader.readValue()` |
| Parse from `InputStream` | Not available | [`Json.createReader(InputStream)`](api/src/main/java/jakarta/json/Json.java:203) with RFC 7159 auto-detection |
| Parse from `InputStream` with explicit charset | Not available | [`JsonReaderFactory.createReader(InputStream, Charset)`](api/src/main/java/jakarta/json/JsonReaderFactory.java:78) |
| Streaming / event-based | Not available | [`JsonParser`](api/src/main/java/jakarta/json/stream/JsonParser.java) — pull parser with 10-event model |
| Duplicate key handling | Always throws [`JsonParseException`](JDK_JSON_API_RESEARCH.md) | Configurable via [`JsonConfig.KEY_STRATEGY`](api/src/main/java/jakarta/json/JsonConfig.java:31): `FIRST`, `LAST`, or `NONE` (throws) |

### 5.2 Serialization / Writing

| Feature | `jdk.incubator.json` | `jakarta.json` (JSON-P) |
|---|---|---|
| Compact serialization | [`JsonValue.toString()`](JDK_JSON_API_RESEARCH.md) | [`JsonWriter.write(JsonValue)`](api/src/main/java/jakarta/json/JsonWriter.java) / `toString()` on value types |
| Pretty-print serialization | [`Json.toDisplayString(JsonValue, int)`](JDK_JSON_API_RESEARCH.md) | [`JsonGeneratorFactory`](api/src/main/java/jakarta/json/stream/JsonGeneratorFactory.java) with `JsonGenerator.PRETTY_PRINTING` config |
| Write to `Writer` | Not available | [`Json.createWriter(Writer)`](api/src/main/java/jakarta/json/Json.java:168) → `JsonWriter` |
| Write to `OutputStream` | Not available | [`Json.createWriter(OutputStream)`](api/src/main/java/jakarta/json/Json.java:181) (UTF-8) |
| Write to `OutputStream` with charset | Not available | [`JsonWriterFactory.createWriter(OutputStream, Charset)`](api/src/main/java/jakarta/json/JsonWriterFactory.java:81) |
| Streaming / incremental write | Not available | [`JsonGenerator`](api/src/main/java/jakarta/json/stream/JsonGenerator.java) — push writer with fluent chaining |
| One-write enforcement | N/A (value objects can be serialized repeatedly) | `JsonWriter` enforces single-write-per-instance; second call throws `IllegalStateException` |

> **Key difference:** The JDK API produces JSON text as a `String` only. JSON-P can write directly to any `Writer`, `OutputStream`, or byte stream with charset control. The JDK API has no streaming writer.

---

## 6. Error Handling Comparison

### Exception Hierarchy

| Category | `jdk.incubator.json` | `jakarta.json` (JSON-P) |
|---|---|---|
| Base exception | N/A — no shared base beyond `RuntimeException` | [`JsonException`](api/src/main/java/jakarta/json/JsonException.java) extends `RuntimeException` |
| Parse failure | [`JsonParseException`](JDK_JSON_API_RESEARCH.md) (final) — carries line + position | [`JsonParsingException`](api/src/main/java/jakarta/json/stream/JsonParsingException.java) extends `JsonException` — carries [`JsonLocation`](api/src/main/java/jakarta/json/stream/JsonLocation.java) |
| Navigation / type failure | [`JsonValueException`](JDK_JSON_API_RESEARCH.md) (final) | `ClassCastException` (JSON-P uses casts at access boundaries) or `NullPointerException` (absent key) |
| Write / generation failure | N/A (no writer) | [`JsonGenerationException`](api/src/main/java/jakarta/json/stream/JsonGenerationException.java) extends `JsonException` |
| Checked vs unchecked | All unchecked | All unchecked (via `JsonException` → `RuntimeException`) |
| Pointer / patch failure | N/A | `JsonException` from [`JsonPointer`](api/src/main/java/jakarta/json/JsonPointer.java) operations |

### Location Information

Both APIs provide location information on parse failures:

- **JDK:** [`JsonParseException.getErrorLine()`](JDK_JSON_API_RESEARCH.md) (0-based) and [`getErrorPosition()`](JDK_JSON_API_RESEARCH.md) (0-based, UTF-16 code units)
- **JSON-P:** [`JsonParsingException.getLocation()`](api/src/main/java/jakarta/json/stream/JsonParsingException.java:73) returns a [`JsonLocation`](api/src/main/java/jakarta/json/stream/JsonLocation.java) with `getLineNumber()` (1-based), `getColumnNumber()` (1-based), and `getStreamOffset()` (byte or character offset)

> **Key difference:** JSON-P's `JsonLocation` is richer (line + column + stream offset vs. line + position). JSON-P line numbers are 1-based; JDK line numbers are 0-based. JSON-P wraps all failures under a common `JsonException` base; the JDK has no shared base.

### Navigation Failure Behavior

JSON-P uses Java's standard `ClassCastException` / `NullPointerException` / `IndexOutOfBoundsException` to signal type mismatches and absent values — this is consistent with the `Map`/`List` inheritance model. The JDK API introduces a dedicated `JsonValueException` for all such failures (type mismatch, absent key, out-of-bounds index, numeric overflow), creating a more uniform error surface but requiring callers to distinguish failure cause via the exception message alone.

---

## 7. Duplicate Key Handling

A notable policy divergence:

| Behavior | `jdk.incubator.json` | `jakarta.json` (JSON-P) |
|---|---|---|
| Parse-time duplicate keys | Always throws `JsonParseException` — no override | Configurable: `JsonConfig.KEY_STRATEGY.FIRST`, `LAST`, or `NONE` (throws) |
| Construct-time duplicate keys | `JsonObject.of(Map)` throws `IllegalArgumentException` (inherits from `Map.of` contract) | Not enforced — `JsonObjectBuilder.add(name, …)` silently replaces the prior mapping for the same name |

The JDK API treats duplicate keys as a hard error at all levels. JSON-P defaults to lenient (last-wins in most implementations) with an opt-in strict mode.

---

## 8. JSON Standards Coverage

| Standard | `jdk.incubator.json` | `jakarta.json` (JSON-P) |
|---|---|---|
| [RFC 8259](https://www.rfc-editor.org/rfc/rfc8259) — JSON | ✅ Full compliance | ✅ Full compliance |
| [RFC 6901](https://tools.ietf.org/html/rfc6901) — JSON Pointer | ❌ Not supported | ✅ [`JsonPointer`](api/src/main/java/jakarta/json/JsonPointer.java) + [`Json.createPointer()`](api/src/main/java/jakarta/json/Json.java:324) + [`Json.encodePointer()`](api/src/main/java/jakarta/json/Json.java:506) / [`decodePointer()`](api/src/main/java/jakarta/json/Json.java:519) |
| [RFC 6902](https://tools.ietf.org/html/rfc6902) — JSON Patch | ❌ Not supported | ✅ [`JsonPatch`](api/src/main/java/jakarta/json/JsonPatch.java), [`JsonPatchBuilder`](api/src/main/java/jakarta/json/JsonPatchBuilder.java), [`Json.createDiff()`](api/src/main/java/jakarta/json/Json.java:377) |
| [RFC 7396](https://tools.ietf.org/html/rfc7396) — JSON Merge Patch | ❌ Not supported | ✅ [`JsonMergePatch`](api/src/main/java/jakarta/json/JsonMergePatch.java), [`Json.createMergePatch()`](api/src/main/java/jakarta/json/Json.java:390), [`Json.createMergeDiff()`](api/src/main/java/jakarta/json/Json.java:405) |

---

## 9. Navigation Model Comparison

### JDK Incubator JSON — Uniform Chain Navigation

All navigation goes through a single interface (`JsonValue`), which provides `get(String)` and `get(int)` as default methods that delegate to `asMap()` / `asList()`. Every node in the tree participates in the same navigation API, and there is an `Optional`-returning twin for each throwing method:

```java
// Throws on any absent key or type mismatch
String city = root.get("address").get("city").asString();

// Optional chain — returns empty if anything is missing or null
Optional<String> city = root.tryGet("address")
    .flatMap(a -> a.tryGet("city"))
    .flatMap(v -> v.tryValue())
    .map(v -> v.asString());
```

### Jakarta JSON-P — Type-Specific Accessor Methods

Navigation in JSON-P is done through typed accessors on `JsonObject` and typed index-accessors on `JsonArray`. There is no unified default-method navigation chain; callers must cast to the appropriate container type first:

```java
// JSON-P: explicit cast + typed accessors
JsonObject address = root.getJsonObject("address");   // returns null if absent
String city = address.getString("city");              // throws NPE if address is null

// Null-safe pattern requires explicit null checks
JsonObject address = root.getJsonObject("address");
String city = (address != null) ? address.getString("city", "unknown") : "unknown";
```

JSON-P offers `getString(name, defaultValue)` overloads that handle absent keys gracefully, but there is no equivalent of `tryGet()`/`tryValue()` — absent member accessors return `null` from typed accessors (`getJsonObject`, `getJsonArray`, etc.) but throw `NullPointerException` from the primitive-returning accessors (`getString`, `getInt`, `getBoolean`).

> **Key difference:** The JDK API has a more composable, null-safe navigation model using `Optional`. JSON-P relies on Java's existing `null`-return convention from `Map` lookups, which can cause `NullPointerException` on chained access unless callers insert explicit null checks.

---

## 10. Streaming API Comparison

JSON-P has a full-featured streaming tier that the JDK API entirely lacks:

### [`JsonParser`](api/src/main/java/jakarta/json/stream/JsonParser.java) (Pull Parser)
- Event-based pull model: `hasNext()` / `next()` → `Event` enum (10 values: `START_OBJECT`, `END_OBJECT`, `START_ARRAY`, `END_ARRAY`, `KEY_NAME`, `VALUE_STRING`, `VALUE_NUMBER`, `VALUE_TRUE`, `VALUE_FALSE`, `VALUE_NULL`)
- Critical for processing JSON documents **too large to fit in memory**
- Value accessors: `getString()`, `getInt()`, `getLong()`, `getBigDecimal()`
- Partial model materialisation (since 1.1): `getObject()`, `getArray()`, `getValue()`
- Stream integration (since 1.1): `getArrayStream()`, `getObjectStream()`, `getValueStream()`
- Skip helpers (since 1.1): `skipArray()`, `skipObject()`
- Can parse from `Reader`, `InputStream`, or directly from an existing `JsonObject`/`JsonArray`

### [`JsonGenerator`](api/src/main/java/jakarta/json/stream/JsonGenerator.java) (Push Writer)
- Context-aware streaming writer: tracks object/array/field context and validates call sequence
- Method chaining: all write methods return `this`
- Supports `PRETTY_PRINTING` configuration
- Extends both `Flushable` and `Closeable`; validates that the output is a complete JSON structure before closing
- Writes to `Writer`, `OutputStream` (UTF-8), or `OutputStream` + `Charset`

> The JDK API has no streaming parser or streaming writer. All processing happens through in-memory tree construction and `String`-based serialization.

---

## 11. `java.util.stream` Integration

### JDK Incubator JSON
No native `Stream` integration. Callers compose streams manually using `asList()` and `asMap()`:

```java
// Manual streaming from JDK API
root.get("tags").asList().stream()
    .map(v -> v.asString())
    .forEach(System.out::println);
```

### Jakarta JSON-P
JSON-P provides two levels of Stream integration:

1. **`JsonParser` streaming methods** (since 1.1): `getArrayStream()`, `getObjectStream()`, `getValueStream()` for lazy event-stream processing
2. **[`JsonCollectors`](api/src/main/java/jakarta/json/stream/JsonCollectors.java)** (since 1.1): Collectors for accumulating stream elements back into JSON structures:
   - `toJsonArray()` — `Collector<JsonValue, …, JsonArray>`
   - `toJsonObject(Function, Function)` — mapping + reduction to `JsonObject`
   - `groupingBy(Function, Collector)` — group `JsonValue` elements into a `JsonObject` of `JsonArray`s

```java
// JSON-P: collectors integration
JsonArray filtered = jsonParser.getArrayStream()
    .filter(v -> v.asJsonObject().getString("active").equals("true"))
    .collect(JsonCollectors.toJsonArray());
```

---

## 12. SPI and Extensibility

| Dimension | `jdk.incubator.json` | `jakarta.json` (JSON-P) |
|---|---|---|
| Custom implementation | Value subtype interfaces are `non-sealed` — custom impls can implement `JsonObject` etc. | Implementations plug in via `JsonProvider` SPI |
| Provider discovery | No SPI — single built-in JDK impl | System property `jakarta.json.provider`, then `ServiceLoader`, then OSGi, then `org.eclipse.parsson.JsonProviderImpl` (default) |
| Thread-safety of factory | N/A | All factory interfaces are documented thread-safe |
| Multiple instances | N/A | Factory pattern (`JsonBuilderFactory`, `JsonReaderFactory`, etc.) with shared config recommended for multiple instances |

---


## 13. Summary Comparison Table

| Capability | `jdk.incubator.json` | `jakarta.json` (JSON-P) |
|---|---|---|
| Parse JSON text | ✅ `Json.parse(String/char[])` | ✅ via `JsonReader` (Reader/InputStream) |
| Parse from InputStream | ❌ | ✅ |
| Streaming parse | ❌ | ✅ `JsonParser` |
| Build JSON programmatically | ✅ Static `of(…)` factories | ✅ Mutable builder pattern |
| Serialize to String | ✅ `toString()` / `toDisplayString()` | ✅ via `JsonWriter` or `toString()` |
| Serialize to OutputStream/Writer | ❌ | ✅ |
| Streaming write | ❌ | ✅ `JsonGenerator` |
| Pretty printing | ✅ `Json.toDisplayString(value, indent)` | ✅ `PRETTY_PRINTING` config |
| `Map`/`List` compatibility | ❌ (separate `asMap()`/`asList()`) | ✅ (`JsonObject` extends `Map`, `JsonArray` extends `List`) |
| Optional-chain navigation | ✅ `tryGet()` / `tryValue()` | ❌ (null-return pattern) |
| Null-safe default accessors | ❌ | ✅ `getString(name, default)` etc. |
| BigDecimal access | ❌ (via `toString()` only) | ✅ `bigDecimalValue()` |
| BigInteger access | ❌ | ✅ `bigIntegerValue()` |
| Exhaustive type switch | ✅ (`sealed` hierarchy) | ❌ (`ValueType` enum dispatch) |
| JSON Pointer (RFC 6901) | ❌ | ✅ |
| JSON Patch (RFC 6902) | ❌ | ✅ |
| JSON Merge Patch (RFC 7396) | ❌ | ✅ |
| Duplicate key config | ❌ (always throws) | ✅ `JsonConfig.KEY_STRATEGY` |
| Stream collectors | ❌ | ✅ `JsonCollectors` |
| `java.util.stream` integration | Manual | ✅ Native via parser + collectors |
| SPI / pluggable implementation | ❌ | ✅ `JsonProvider` |
| Dependencies | `java.base` only | `java.logging`, SPI provider (Parsson by default) |

---

## 14. Key Locations Reference

| File | Description |
|---|---|
| [`api/src/main/java/jakarta/json/JsonValue.java`](api/src/main/java/jakarta/json/JsonValue.java) | Root of JSON-P type hierarchy; `ValueType` enum; `TRUE`/`FALSE`/`NULL` singletons |
| [`api/src/main/java/jakarta/json/JsonObject.java`](api/src/main/java/jakarta/json/JsonObject.java) | JSON object — extends `Map<String,JsonValue>`; typed accessors; null-returning `getJsonXxx()` |
| [`api/src/main/java/jakarta/json/JsonArray.java`](api/src/main/java/jakarta/json/JsonArray.java) | JSON array — extends `List<JsonValue>`; typed accessors; `getValuesAs()` |
| [`api/src/main/java/jakarta/json/JsonNumber.java`](api/src/main/java/jakarta/json/JsonNumber.java) | JSON number — full `BigDecimal`/`BigInteger`/exact conversion suite |
| [`api/src/main/java/jakarta/json/JsonString.java`](api/src/main/java/jakarta/json/JsonString.java) | JSON string — `getString()` + `getChars()` |
| [`api/src/main/java/jakarta/json/JsonStructure.java`](api/src/main/java/jakarta/json/JsonStructure.java) | Container supertype — adds `getValue(jsonPointer)` |
| [`api/src/main/java/jakarta/json/Json.java`](api/src/main/java/jakarta/json/Json.java) | Central factory class — all creation entry points |
| [`api/src/main/java/jakarta/json/JsonObjectBuilder.java`](api/src/main/java/jakarta/json/JsonObjectBuilder.java) | Fluent builder for `JsonObject` |
| [`api/src/main/java/jakarta/json/JsonArrayBuilder.java`](api/src/main/java/jakarta/json/JsonArrayBuilder.java) | Fluent builder for `JsonArray`; add/set/remove at index |
| [`api/src/main/java/jakarta/json/spi/JsonProvider.java`](api/src/main/java/jakarta/json/spi/JsonProvider.java) | SPI root; provider discovery logic (system property → ServiceLoader → OSGi → Parsson) |
| [`api/src/main/java/jakarta/json/JsonConfig.java`](api/src/main/java/jakarta/json/JsonConfig.java) | `KEY_STRATEGY` config constant and enum |
| [`api/src/main/java/jakarta/json/JsonReader.java`](api/src/main/java/jakarta/json/JsonReader.java) | High-level reader; single-use; `read()`, `readObject()`, `readArray()`, `readValue()` |
| [`api/src/main/java/jakarta/json/JsonWriter.java`](api/src/main/java/jakarta/json/JsonWriter.java) | High-level writer; single-use; ⚠️ contains `jdk.incubator.json` reference on this branch |
| [`api/src/main/java/jakarta/json/stream/JsonParser.java`](api/src/main/java/jakarta/json/stream/JsonParser.java) | Streaming pull parser; `Event` enum; stream/skip methods |
| [`api/src/main/java/jakarta/json/stream/JsonGenerator.java`](api/src/main/java/jakarta/json/stream/JsonGenerator.java) | Streaming push writer; fluent; `PRETTY_PRINTING` constant |
| [`api/src/main/java/jakarta/json/JsonPointer.java`](api/src/main/java/jakarta/json/JsonPointer.java) | RFC 6901 — add/remove/replace/get/containsValue |
| [`api/src/main/java/jakarta/json/JsonPatch.java`](api/src/main/java/jakarta/json/JsonPatch.java) | RFC 6902 — `apply()`, `toJsonArray()`; `Operation` enum |
| [`api/src/main/java/jakarta/json/JsonMergePatch.java`](api/src/main/java/jakarta/json/JsonMergePatch.java) | RFC 7396 — `apply()`, `toJsonValue()` |
| [`api/src/main/java/jakarta/json/stream/JsonCollectors.java`](api/src/main/java/jakarta/json/stream/JsonCollectors.java) | Stream collectors: `toJsonArray()`, `toJsonObject()`, `groupingBy()` |
| [`api/src/main/java/module-info.java`](api/src/main/java/module-info.java) | Module descriptor — currently `requires jdk.incubator.json` on this branch |

---

## 15. Gaps and Assumptions


1. **JDK API implementation details:** The JDK incubator API's internal implementation (lazy parsing, memory model) was not available for direct inspection — findings come from Javadoc only.
2. **Data binding:** Neither API covers data binding (POJO ↔ JSON). That is handled by Jakarta JSON Binding (JSON-B) on the Jakarta side, and by no JDK equivalent.
3. **Thread-safety of value objects:** Both APIs describe their JSON value objects as immutable, but neither specifies memory visibility guarantees (e.g., `final` field publication). Assumed safe for concurrent read across threads given immutability.
4. **`JsonConfig.KEY_STRATEGY` implementation coverage:** The config key is defined in [`JsonConfig`](api/src/main/java/jakarta/json/JsonConfig.java:31), but whether a given `JsonProvider` implementation honors it is implementation-dependent. The `NONE` strategy (throws on duplicate) is the closest match to the JDK API's behavior.
