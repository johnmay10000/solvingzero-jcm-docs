# Source Code Changes: Optional Date Parameter Implementation

## Overview

This document details all source code changes made to implement the optional date parameter feature for `fetchUserIntervalReads`.

## Files Modified

### 1. `app/functions/src/timescaledb/timescale.cljs`

**Purpose**: Database layer - Add optional date parameter support to SQL queries

#### Changes Made

**Added Import**:
```clojure
[clojure.string :as string]  ; Added for string operations
```

**Added Helper Function** (lines 89-106):
```clojure
(defn- build-date-clause
  "Builds SQL date clause and parameters.
   Returns [clause-string params-vector]
   
   Args:
     date-str - Optional date string in YYYY-MM-DD format.
                If nil, uses CURRENT_DATE.
   
   Returns:
     [date-clause-string date-params-vector]"
  [date-str]
  (if (some? date-str)
    [(str "AND read_start_date >= $2::DATE - INTERVAL '2 days' - INTERVAL '365 days' "
          "AND read_start_date < $2::DATE - INTERVAL '2 days'")
     [date-str]]
    [(str "AND read_start_date >= CURRENT_DATE - INTERVAL '2 days' - INTERVAL '365 days' "
          "AND read_start_date < CURRENT_DATE - INTERVAL '2 days'")
     []]))
```

**Modified Function**: `query-hourly-interval-reads` (lines 108-135)

**Before**:
```clojure
(defn query-hourly-interval-reads
  "Given an NMI,
   Return a list of rows for the last 365 days, one per aggregated hour usage,
   each row contains an hourly `sum` of usage"
  [national-meter-id]
  (let [query (str "SELECT date_trunc('hour', read_start_date) AS read_start, register_suffix, sum(read_value) AS sum "
                   "FROM interval_read "
                   "WHERE national_metering_id = $1 "
                   "AND read_start_date >= CURRENT_DATE - INTERVAL '2 days' - INTERVAL '365 days' "
                   "AND read_start_date < CURRENT_DATE - INTERVAL '2 days' "
                   "GROUP BY 1,2 "
                   "ORDER BY read_start ASC")]
    (-> (query! query [national-meter-id])
        (p/catch (fn [err]
                   (throw (ex-info (str "Failed to query hourly interval reads" err)
                                   {:national-metering-id national-meter-id
                                    :error err})))))))
```

**After**:
```clojure
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

**Key Changes**:
- ✅ Converted to multi-arity function (2 arities: with and without date)
- ✅ Uses `build-date-clause` helper to construct SQL clause dynamically
- ✅ Adds date parameter to query params when provided
- ✅ Includes `date-str` in error context for debugging
- ✅ Maintains backward compatibility (single arity calls two-arity with `nil`)

### 2. `app/functions/src/admin/core.cljs`

**Purpose**: Business logic and handler layer - Add optional date parameter support

#### Changes Made

**Modified Function**: `interval-reads-for-user-uid` (lines 20-51)

**Before**:
```clojure
(defn interval-reads-for-user-uid
  "Queries timescale for daily interval reads for the specified `date`.
   Returns up to 24 rows, each containing an aggregated sums for all register suffixes
   existing for the `national-metering-id`.
   Each returning row is a map of 'date-time' inst and a key for each register suffix

   Result example: [{:date-time #inst\"2024-10-15T21:00:00.000-00:00\", :E1 0.4, :B1 -0.4}]

   - If `user-uid` does not contain `national-metering-id`, returns an empty list.
   - Results are ordered by date-time
   "
  [user-uid national-metering-id]
  (p/let [{:keys [national-metering-ids]} (firestore/get-user user-uid)
           result (if (contains? (set national-metering-ids) national-metering-id)
                    (timescale/query-hourly-interval-reads national-metering-id)
                    [])]
    (->> result
         (group-by :read_start)
         (map (fn [[date-time values]]
                (merge {:date-time (tf/unparse (tf/formatters :date-time-no-ms) (DateTime. date-time))}
                       (zipmap (map :register_suffix values)
                               (map (comp numbers/round-3dp :sum) values)))))
         (sort-by :date-time)
         (clj->js))))
```

**After**:
```clojure
(defn interval-reads-for-user-uid
  "Queries timescale for daily interval reads for the specified date.
   Returns up to 24 rows, each containing aggregated sums for all register suffixes
   existing for the `national-metering-id`.
   Each returning row is a map of 'date-time' inst and a key for each register suffix

   Result example: [{:date-time #inst\"2024-10-15T21:00:00.000-00:00\", :E1 0.4, :B1 -0.4}]

   Args:
     user-uid - User identifier (required)
     national-metering-id - NMI identifier (required)
     date-str - Optional date string in YYYY-MM-DD format.
                If nil or not provided, uses CURRENT_DATE.

   - If `user-uid` does not contain `national-metering-id`, returns an empty list.
   - Results are ordered by date-time
   "
  ([user-uid national-metering-id]
   (interval-reads-for-user-uid user-uid national-metering-id nil))
  ([user-uid national-metering-id date-str]
   (p/let [{:keys [national-metering-ids]} (firestore/get-user user-uid)
           result (if (contains? (set national-metering-ids) national-metering-id)
                    (timescale/query-hourly-interval-reads national-metering-id date-str)
                    [])]
    (->> result
         (group-by :read_start)
         (map (fn [[date-time values]]
                (merge {:date-time (tf/unparse (tf/formatters :date-time-no-ms) (DateTime. date-time))}
                       (zipmap (map :register_suffix values)
                               (map (comp numbers/round-3dp :sum) values)))))
         (sort-by :date-time)
         (clj->js)))))
```

**Key Changes**:
- ✅ Converted to multi-arity function (2 arities: with and without date)
- ✅ Passes `date-str` to `query-hourly-interval-reads` when provided
- ✅ Updated docstring to document optional parameter
- ✅ Maintains backward compatibility

**Modified Function**: `fetch-user-interval-handler` (lines 53-67)

**Before**:
```clojure
(defn fetch-user-interval-handler [request _]
  (let [{:keys [user-uid national-metering-id]} (:data request)]
    (->
     (interval-reads-for-user-uid user-uid national-metering-id)
     (p/catch (fn [err]
                (firestore/write-user-error
                 "admin/fetch-user-interval"
                 user-uid
                 err)
                (p/rejected err))))))
```

**After**:
```clojure
(defn fetch-user-interval-handler [request _]
  (let [{:keys [user-uid national-metering-id date]} (:data request)
        ;; Normalize: treat empty string as nil for safety
        date-str (when (and (some? date) (not= date "")) date)]
    (->
     ;; Use multi-arity: call single arity if no date, two arity if date present
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

**Key Changes**:
- ✅ Extracts optional `date` from request data
- ✅ Normalizes empty string to `nil` for safety
- ✅ Uses explicit `some?` check before calling multi-arity function
- ✅ Calls appropriate arity based on date presence
- ✅ Maintains error handling unchanged

## Code Quality

### Design Decisions

1. **Multi-Arity Functions**: Chosen over separate functions for:
   - Type safety (compiler enforces arity)
   - Single function interface (easier to maintain)
   - Idiomatic Clojure approach
   - No breaking changes

2. **Helper Function**: `build-date-clause` extracted for:
   - Separation of concerns
   - Easier testing
   - Better maintainability
   - Clearer code intent

3. **Explicit Nil Handling**: Uses `some?` checks for:
   - Explicit optionality in code
   - Better readability
   - Catches bugs earlier
   - Follows Clojure best practices

4. **Empty String Normalization**: Treats empty string as `nil` for:
   - Safety (defensive programming)
   - Consistent behavior
   - Better UX (empty string = use default)

### Backward Compatibility

✅ **Fully backward compatible**:
- Single-arity calls still work (delegate to two-arity with `nil`)
- Existing code unchanged
- No breaking changes
- Same behavior when date not provided

### Security

✅ **SQL Injection Protection**:
- Uses parameterized queries (`$2::DATE`)
- Date string passed as parameter, not concatenated
- PostgreSQL validates date format

✅ **Input Validation**:
- Empty strings normalized to `nil`
- Invalid dates handled by PostgreSQL
- Access control still enforced

## Impact Analysis

### Performance Impact
- ✅ **No performance impact**: Same query performance
- ✅ **Index usage**: Queries use existing indexes
- ✅ **Query plan caching**: Parameterized queries enable caching

### Breaking Changes
- ✅ **None**: Fully backward compatible

### Dependencies
- ✅ **No new dependencies**: Uses existing libraries
- ✅ **No dependency updates**: Uses existing versions

## Testing

All source code changes are covered by:
- Unit tests (database layer)
- Integration tests (business logic layer)
- End-to-end tests (full call chain)

See `TEST_CODE_CHANGES.md` for detailed test coverage.

