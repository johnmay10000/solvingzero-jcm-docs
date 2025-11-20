# Backward Compatibility Analysis: Optional Date Parameter

## Question
Is the recommended approach backward compatible when no date is provided in the top-level request?

## Answer: ✅ **YES** - Fully Backward Compatible

The multi-arity function approach ensures backward compatibility through the entire call chain.

## Call Chain Analysis

### Scenario 1: No Date Provided (Backward Compatible Path)

```
Firebase Function Call
  ↓
{data: {user-uid: "...", national-metering-id: "..."}}  // No date field
  ↓
Middleware (passes through unchanged)
  ↓
fetch-user-interval-handler
  ↓
Extracts: {:user-uid "...", :national-metering-id "..."}  // :date is nil/absent
  ↓
Calls: (interval-reads-for-user-uid user-uid national-metering-id)  // Single arity
  ↓
interval-reads-for-user-uid (single arity)
  ↓
Calls: (query-hourly-interval-reads national-metering-id)  // Single arity
  ↓
query-hourly-interval-reads (single arity)
  ↓
Uses: CURRENT_DATE in SQL query ✅
```

### Scenario 2: Date Provided (New Functionality)

```
Firebase Function Call
  ↓
{data: {user-uid: "...", national-metering-id: "...", date: "2024-11-20"}}
  ↓
Middleware (passes through unchanged)
  ↓
fetch-user-interval-handler
  ↓
Extracts: {:user-uid "...", :national-metering-id "...", :date "2024-11-20"}
  ↓
Calls: (interval-reads-for-user-uid user-uid national-metering-id date-str)  // Two arity
  ↓
interval-reads-for-user-uid (two arity)
  ↓
Calls: (query-hourly-interval-reads national-metering-id date-str)  // Two arity
  ↓
query-hourly-interval-reads (two arity)
  ↓
Uses: date-str in SQL query ($2::DATE) ✅
```

## Implementation Details

### Handler Implementation (Backward Compatible)

```clojure
(defn fetch-user-interval-handler [request _]
  (let [{:keys [user-uid national-metering-id date]} (:data request)]
    (->
     ;; Multi-arity call: if date is nil/absent, calls single arity
     ;; If date is present, calls two arity
     (if (some? date)
       (interval-reads-for-user-uid user-uid national-metering-id date)
       (interval-reads-for-user-uid user-uid national-metering-id))
     (p/catch (fn [err]
                (firestore/write-user-error
                 "admin/fetch-user-interval"
                 user-uid
                 err)
                (p/rejected err))))))
```

**Alternative (More Concise):**
```clojure
(defn fetch-user-interval-handler [request _]
  (let [{:keys [user-uid national-metering-id date]} (:data request)]
    (->
     ;; Clojure's multi-arity dispatch handles this automatically
     ;; If date is nil, we can still call two-arity with nil
     ;; But cleaner to use conditional or let multi-arity handle it
     (interval-reads-for-user-uid user-uid national-metering-id date)
     (p/catch (fn [err]
                (firestore/write-user-error
                 "admin/fetch-user-interval"
                 user-uid
                 err)
                (p/rejected err))))))
```

**Best Approach (Uses Multi-Arity Correctly):**
```clojure
(defn fetch-user-interval-handler [request _]
  (let [{:keys [user-uid national-metering-id date]} (:data request)]
    (->
     ;; Use multi-arity: if date is nil, call single arity
     ;; If date is present, call two arity
     (if (some? date)
       (interval-reads-for-user-uid user-uid national-metering-id date)
       (interval-reads-for-user-uid user-uid national-metering-id))
     (p/catch (fn [err]
                (firestore/write-user-error
                 "admin/fetch-user-interval"
                 user-uid
                 err)
                (p/rejected err))))))
```

### Business Logic Implementation (Backward Compatible)

```clojure
(defn interval-reads-for-user-uid
  "Queries timescale for daily interval reads.
   
   Args:
     user-uid - User identifier
     national-metering-id - NMI identifier
     date-str - Optional date string in YYYY-MM-DD format"
  ([user-uid national-metering-id]
   (interval-reads-for-user-uid user-uid national-metering-id nil))
  ([user-uid national-metering-id date-str]
   (p/let [{:keys [national-metering-ids]} (firestore/get-user user-uid)
           result (if (contains? (set national-metering-ids) national-metering-id)
                    ;; Multi-arity call: passes date-str (may be nil) to query function
                    (timescale/query-hourly-interval-reads national-metering-id date-str)
                    [])]
     (->> result
          (group-by :read_start)
          (map (fn [[date-time values]]
                 (merge {:date-time (tf/unparse (tf/formatters :date-time-no-ms) (DateTime. date-time))}
                        (zipmap (map :register_suffix values)
                                (map (comp numbers/round-3dp :sum) values))))
          (sort-by :date-time)
          (clj->js)))))
```

### Database Query Implementation (Backward Compatible)

```clojure
(defn query-hourly-interval-reads
  "Queries hourly interval reads for the last 365 days.
   
   Args:
     national-meter-id - National Metering Identifier (required)
     date-str - Optional date string in YYYY-MM-DD format.
                If nil or not provided, uses CURRENT_DATE."
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

## Backward Compatibility Guarantees

### ✅ Existing Code Continues to Work

**Before (Current Implementation):**
```clojure
;; Handler
(interval-reads-for-user-uid user-uid national-metering-id)

;; Business Logic
(timescale/query-hourly-interval-reads national-metering-id)

;; Database Query
;; Uses CURRENT_DATE in SQL
```

**After (With Optional Date):**
```clojure
;; Handler - No date provided
(interval-reads-for-user-uid user-uid national-metering-id)
;; ↓ Calls single arity
;; ↓ Calls single arity
;; ↓ Uses CURRENT_DATE ✅

;; Handler - Date provided
(interval-reads-for-user-uid user-uid national-metering-id date-str)
;; ↓ Calls two arity
;; ↓ Calls two arity  
;; ↓ Uses date-str ✅
```

### ✅ Multi-Arity Function Dispatch

Clojure's multi-arity function dispatch ensures:
- **Single arity call** → Single arity implementation → Uses CURRENT_DATE
- **Two arity call with nil** → Two arity implementation → `some?` check → Uses CURRENT_DATE
- **Two arity call with value** → Two arity implementation → `some?` check → Uses date value

### ✅ Nil Handling

The `some?` check in `build-date-clause` handles nil correctly:
```clojure
(defn- build-date-clause [date-str]
  (if (some? date-str)  ; nil? returns false, so uses CURRENT_DATE
    [(str "... $2::DATE ...") [date-str]]
    [(str "... CURRENT_DATE ...") []]))
```

## Edge Cases Handled

1. **Date field absent from request** ✅
   - `(:data request)` returns `nil` for missing key
   - Handler calls single arity → Uses CURRENT_DATE

2. **Date field explicitly nil** ✅
   - Handler can call two arity with `nil`
   - `some?` check returns `false` → Uses CURRENT_DATE

3. **Date field empty string** ⚠️
   - `some? ""` returns `true` (empty string is truthy)
   - Would pass empty string to SQL (PostgreSQL would error)
   - **Recommendation**: Add validation or treat empty string as nil

4. **Date field invalid format** ⚠️
   - Would pass invalid string to SQL
   - PostgreSQL would handle validation
   - **Recommendation**: Add format validation

## Recommended Handler Implementation

```clojure
(defn fetch-user-interval-handler [request _]
  (let [{:keys [user-uid national-metering-id date]} (:data request)
        ;; Normalize: treat empty string as nil
        date-str (when (and (some? date) (not= date "")) date)]
    (->
     ;; Use multi-arity correctly
     (if (some? date-str)
       (interval-reads-for-user-uid user-uid national-metering-id date-str)
       (interval-reads-for-user-uid user-uid national-metering-id))
     (p/catch (fn [err]
                (firestore/write-user-error
                 "admin/fetch-user-interval"
                 user-uid
                 err)
                (p/rejected err))))))
```

## Verification Checklist

- ✅ Existing calls without date parameter work unchanged
- ✅ New calls with date parameter work correctly
- ✅ Nil values handled correctly through call chain
- ✅ Multi-arity dispatch ensures correct function selection
- ✅ SQL query generation handles both cases
- ✅ No breaking changes to function signatures
- ✅ Backward compatible at every layer

## Conclusion

**The recommended approach is fully backward compatible** because:

1. **Multi-arity functions** provide two entry points:
   - Single arity (existing code path) → Uses CURRENT_DATE
   - Two arity (new code path) → Uses provided date or CURRENT_DATE if nil

2. **Nil handling** is explicit and safe:
   - `some?` checks distinguish between nil and values
   - Helper function (`build-date-clause`) centralizes logic

3. **Call chain** preserves backward compatibility:
   - Handler can call single or two arity based on date presence
   - Business logic passes through date (or nil) correctly
   - Database query handles both cases

4. **No breaking changes**:
   - Existing function signatures unchanged
   - New signatures added (not replacing)
   - Default behavior (CURRENT_DATE) preserved

The implementation ensures that **existing code continues to work exactly as before**, while new code can optionally provide a date parameter.

