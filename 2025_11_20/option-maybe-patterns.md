# Option/Maybe Patterns in Clojure for Optional Date Parameter

## Problem Statement

Using `nil` for optional values is fragile because:
- `nil` can represent both "not provided" and "explicitly set to nil"
- No type safety - easy to forget nil checks
- No compile-time guarantees about handling optional values
- Can lead to `NullPointerException`-like errors

Using an Option/Maybe type provides:
- Explicit encoding of optionality in the type system
- Compile-time safety
- Functional composition patterns (map, flatMap, etc.)
- Clear intent in function signatures

## Clojure Options Analysis

### Option 1: Idiomatic Clojure with `some?` and Conditionals

**Pros:**
- ✅ No dependencies
- ✅ Idiomatic Clojure
- ✅ Simple and readable
- ✅ Works well with multi-arity functions

**Cons:**
- ❌ No type safety
- ❌ Still uses nil internally
- ❌ Easy to forget nil checks

**Example:**
```clojure
(defn query-hourly-interval-reads
  ([national-meter-id]
   (query-hourly-interval-reads national-meter-id nil))
  ([national-meter-id date-str]
   (let [query (if (some? date-str)
                 (str "SELECT ... WHERE ... AND read_start_date >= $2::DATE - INTERVAL '2 days' - INTERVAL '365 days' "
                      "AND read_start_date < $2::DATE - INTERVAL '2 days'")
                 (str "SELECT ... WHERE ... AND read_start_date >= CURRENT_DATE - INTERVAL '2 days' - INTERVAL '365 days' "
                      "AND read_start_date < CURRENT_DATE - INTERVAL '2 days'"))
         params (if (some? date-str)
                  [national-meter-id date-str]
                  [national-meter-id])]
     (query! query params))))
```

### Option 2: Simple Custom Option Type

**Pros:**
- ✅ No external dependencies
- ✅ Type-safe and explicit
- ✅ Can use pattern matching
- ✅ Lightweight

**Cons:**
- ❌ Custom implementation to maintain
- ❌ Not standard Clojure idiom
- ❌ Requires helper functions

**Implementation:**
```clojure
;; Simple Option type
(defprotocol Option
  (is-some? [this])
  (value [this])
  (or-else [this default]))

(defrecord Some [value]
  Option
  (is-some? [_] true)
  (value [_] value)
  (or-else [_ _] value))

(defrecord None []
  Option
  (is-some? [_] false)
  (value [_] nil)
  (or-else [_ default] default))

(defn some [x] (->Some x))
(defn none [] (->None))

;; Usage
(defn query-hourly-interval-reads
  ([national-meter-id]
   (query-hourly-interval-reads national-meter-id (none)))
  ([national-meter-id date-option]
   (let [date-str (if (is-some? date-option)
                    (value date-option)
                    nil)
         query (if (some? date-str)
                 (str "... $2::DATE ...")
                 (str "... CURRENT_DATE ..."))
         params (if (some? date-str)
                  [national-meter-id date-str]
                  [national-meter-id])]
     (query! query params))))

;; Pattern matching version (Clojure 1.11+)
(defn query-hourly-interval-reads
  ([national-meter-id]
   (query-hourly-interval-reads national-meter-id (none)))
  ([national-meter-id date-option]
   (match date-option
     (->Some date-str) (query! (build-query-with-date date-str) 
                                [national-meter-id date-str])
     (->None)          (query! (build-query-with-current-date) 
                                [national-meter-id]))))
```

### Option 3: Using Clojure 1.11+ Pattern Matching

**Pros:**
- ✅ Built-in to Clojure 1.11+
- ✅ No dependencies
- ✅ Expressive pattern matching
- ✅ Type-safe with records

**Cons:**
- ❌ Requires Clojure 1.11+ (you have 1.11.1 ✅)
- ❌ Still need to define Option type or use nil with patterns

**Implementation:**
```clojure
(require '[clojure.core.match :refer [match]])

(defn query-hourly-interval-reads
  ([national-meter-id]
   (query-hourly-interval-reads national-meter-id nil))
  ([national-meter-id date-str]
   (match [date-str]
     [nil] (query! (build-query-with-current-date) [national-meter-id])
     [(date-str :guard some?)] (query! (build-query-with-date date-str) 
                                        [national-meter-id date-str]))))
```

### Option 4: Using Malli (Already in Dependencies)

**Pros:**
- ✅ Already in your dependencies (`metosin/malli`)
- ✅ Schema validation
- ✅ Can encode optionality in schema
- ✅ Runtime validation

**Cons:**
- ❌ More verbose
- ❌ Runtime validation (not compile-time)
- ❌ Overkill for simple optional parameter

**Implementation:**
```clojure
(require '[malli.core :as m])

(def DateOption
  [:maybe [:string {:min 10 :max 10}]])

(defn query-hourly-interval-reads
  ([national-meter-id]
   (query-hourly-interval-reads national-meter-id nil))
  ([national-meter-id date-str]
   (when-not (m/validate DateOption date-str)
     (throw (ex-info "Invalid date option" {:date-str date-str})))
   (if (some? date-str)
     (query! (build-query-with-date date-str) [national-meter-id date-str])
     (query! (build-query-with-current-date) [national-meter-id]))))
```

### Option 5: Lightweight Option Library (cats or similar)

**Pros:**
- ✅ Well-tested library
- ✅ Rich API (map, flatMap, etc.)
- ✅ Functional composition

**Cons:**
- ❌ Additional dependency
- ❌ May be overkill for this use case
- ❌ Not currently in your dependencies

**Would require adding:** `funcool/cats` or similar

## Recommended Approach: Hybrid Pattern Matching + Nil Handling

Given your codebase constraints and Clojure 1.11.1, I recommend a **hybrid approach** that:
1. Uses multi-arity functions (idiomatic Clojure)
2. Uses `some?` for nil checks (simple and clear)
3. Uses pattern matching where it adds clarity
4. Documents the optionality clearly

**✅ Backward Compatibility Guaranteed**: This approach is fully backward compatible. When no date is provided in the request, the call chain correctly defaults to `CURRENT_DATE` at every layer. See `backward-compatibility-analysis.md` for detailed call chain analysis.

### Recommended Implementation

```clojure
(defn build-date-query-clause
  "Builds the date clause for SQL query.
   Returns [query-fragment params] where params may be empty or contain date."
  [date-str]
  (if (some? date-str)
    ["AND read_start_date >= $2::DATE - INTERVAL '2 days' - INTERVAL '365 days' 
      AND read_start_date < $2::DATE - INTERVAL '2 days'"
     [date-str]]
    ["AND read_start_date >= CURRENT_DATE - INTERVAL '2 days' - INTERVAL '365 days' 
      AND read_start_date < CURRENT_DATE - INTERVAL '2 days'"
     []]))

(defn query-hourly-interval-reads
  "Queries hourly interval reads for a national meter ID.
   
   Args:
     national-meter-id - Required NMI string
     date-str - Optional date string in YYYY-MM-DD format. If nil, uses CURRENT_DATE.
   
   Returns:
     Promise resolving to vector of interval read rows"
  ([national-meter-id]
   (query-hourly-interval-reads national-meter-id nil))
  ([national-meter-id date-str]
   (let [[date-clause date-params] (build-date-query-clause date-str)
         query (str "SELECT date_trunc('hour', read_start_date) AS read_start, "
                    "register_suffix, sum(read_value) AS sum "
                    "FROM interval_read "
                    "WHERE national_metering_id = $1 "
                    date-clause
                    " GROUP BY 1,2 "
                    "ORDER BY read_start ASC")
         params (into [national-meter-id] date-params)]
     (-> (query! query params)
         (p/catch (fn [err]
                    (throw (ex-info (str "Failed to query hourly interval reads" err)
                                    {:national-metering-id national-meter-id
                                     :date-str date-str
                                     :error err}))))))))
```

### Alternative: Explicit Option Type with Pattern Matching

If you want more type safety, here's a lightweight Option implementation:

```clojure
(ns functions.src.utils.option
  "Simple Option/Maybe type for optional values")

(defprotocol Option
  "Protocol for optional values"
  (is-some? [this] "Returns true if value is present")
  (value [this] "Returns the value if present, nil otherwise")
  (map-option [this f] "Maps function over value if present")
  (or-else [this default] "Returns value if present, default otherwise"))

(defrecord Some [v]
  Option
  (is-some? [_] true)
  (value [_] v)
  (map-option [_ f] (->Some (f v)))
  (or-else [_ _] v))

(defrecord None []
  Option
  (is-some? [_] false)
  (value [_] nil)
  (map-option [_ _] (->None))
  (or-else [_ default] default))

(defn some [x] (->Some x))
(defn none [] (->None))
(defn option [x] (if (some? x) (some x) (none)))

;; Helper for pattern matching
(defn match-option [opt some-fn none-fn]
  (if (is-some? opt)
    (some-fn (value opt))
    (none-fn)))

;; Usage example
(defn query-hourly-interval-reads
  ([national-meter-id]
   (query-hourly-interval-reads national-meter-id (none)))
  ([national-meter-id date-option]
   (match-option date-option
     (fn [date-str]
       (query! (build-query-with-date date-str) [national-meter-id date-str]))
     (fn []
       (query! (build-query-with-current-date) [national-meter-id])))))
```

## Comparison Matrix

| Approach | Type Safety | Dependencies | Complexity | Idiomatic | Recommended |
|----------|-------------|-------------|------------|-----------|-------------|
| `some?` + conditionals | ⚠️ Runtime | None | Low | ✅ Yes | ✅ **Yes** |
| Custom Option + Pattern Match | ✅ High | None | Medium | ⚠️ Partial | ⚠️ Maybe |
| Pattern Match (nil) | ⚠️ Runtime | None | Low | ✅ Yes | ✅ Yes |
| Malli validation | ⚠️ Runtime | Already have | Medium | ✅ Yes | ⚠️ Overkill |
| Option library | ✅ High | New dep | Medium | ⚠️ Partial | ❌ No |

## Final Recommendation

**Use Option 1 (Idiomatic Clojure with `some?`) with clear documentation:**

1. **Multi-arity functions** - Standard Clojure pattern
2. **`some?` checks** - Clear and idiomatic
3. **Helper function** - `build-date-query-clause` to separate concerns
4. **Clear docstrings** - Document optionality
5. **Pattern matching** - Use where it adds clarity (optional)

**Why this approach:**
- ✅ No new dependencies
- ✅ Follows existing codebase patterns (I see `some?`, `when`, `nil?` already used)
- ✅ Simple and maintainable
- ✅ Clear intent
- ✅ Easy to test
- ✅ Backward compatible

**When to consider Option type:**
- If you find yourself passing optional values through many layers
- If you want compile-time guarantees
- If you're building a larger functional programming style system

## Implementation Example

```clojure
;; timescale.cljs

(defn- build-date-clause
  "Builds SQL date clause and parameters.
   Returns [clause-string params-vector]"
  [date-str]
  (if (some? date-str)
    [(str "AND read_start_date >= $2::DATE - INTERVAL '2 days' - INTERVAL '365 days' "
          "AND read_start_date < $2::DATE - INTERVAL '2 days'")
     [date-str]]
    [(str "AND read_start_date >= CURRENT_DATE - INTERVAL '2 days' - INTERVAL '365 days' "
          "AND read_start_date < CURRENT_DATE - INTERVAL '2 days'")
     []]))

(defn query-hourly-interval-reads
  "Queries hourly interval reads for the last 365 days.
   
   Args:
     national-meter-id - National Metering Identifier (required)
     date-str - Optional date string in YYYY-MM-DD format.
                If nil or not provided, uses CURRENT_DATE.
   
   Returns:
     Promise resolving to vector of {:read_start, :register_suffix, :sum}"
  ([national-meter-id]
   (query-hourly-interval-reads national-meter-id nil))
  ([national-meter-id date-str]
   (let [[date-clause date-params] (build-date-clause date-str)
         query (str "SELECT date_trunc('hour', read_start_date) AS read_start, "
                    "register_suffix, sum(read_value) AS sum "
                    "FROM interval_read "
                    "WHERE national_metering_id = $1 "
                    date-clause
                    " GROUP BY 1,2 "
                    "ORDER BY read_start ASC")
         params (into [national-meter-id] date-params)]
     (-> (query! query params)
         (p/catch (fn [err]
                    (throw (ex-info (str "Failed to query hourly interval reads" err)
                                    {:national-metering-id national-meter-id
                                     :date-str date-str
                                     :error err}))))))))
```

This approach:
- ✅ Explicitly handles optionality
- ✅ Clear separation of concerns (`build-date-clause`)
- ✅ Type-safe at runtime with `some?`
- ✅ Easy to understand and maintain
- ✅ No new dependencies
- ✅ Follows Clojure idioms

## Pattern Matching Alternative (If Desired)

If you want to use pattern matching for clarity:

```clojure
(require '[clojure.core.match :refer [match]])

(defn query-hourly-interval-reads
  ([national-meter-id]
   (query-hourly-interval-reads national-meter-id nil))
  ([national-meter-id date-str]
   (match [(some? date-str)]
     [true]  (let [query (build-query-with-date date-str)
                   params [national-meter-id date-str]]
               (query! query params))
     [false] (let [query (build-query-with-current-date)
                   params [national-meter-id]]
               (query! query params)))))
```

This is more explicit but adds a dependency on `clojure.core.match` (though it's built-in to Clojure 1.11+).

## Conclusion

For this codebase, **use `some?` with multi-arity functions** - it's the most idiomatic, requires no changes to dependencies, and is easy to understand. The optionality is clear from the function signature and docstring, and `some?` provides runtime safety.

If you want more type safety in the future, consider the lightweight Option type implementation provided above, but start simple with `some?`.

