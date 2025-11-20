# Unified Implementation Plan: Optional Date Parameter

## Overview

This document provides a complete, step-by-step implementation plan for adding optional date parameter support to the `fetchUserIntervalReads` function chain. The implementation uses **multi-arity functions with `some?` checks** (idiomatic Clojure) and is **fully backward compatible**.

## Implementation Approach

- **Multi-arity functions** - Type-safe optional parameters
- **`some?` checks** - Explicit nil handling (no fragile nil passing)
- **Helper functions** - Clear separation of concerns
- **Backward compatible** - Existing code continues to work unchanged

## Complete Call Chain

```
Firebase Function: fetchUserIntervalReads
  ↓
Middleware: auth/wrap-admin-authentication → fiskil/token-middleware
  ↓
Handler: fetch-user-interval-handler (admin/core.cljs)
  ↓
Business Logic: interval-reads-for-user-uid (admin/core.cljs)
  ↓
Database Query: query-hourly-interval-reads (timescale.cljs)
  ↓
TimescaleDB SQL Query
```

## Phase 1: Database Layer Implementation

### File: `app/functions/src/timescaledb/timescale.cljs`

#### Step 1.1: Add Helper Function

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

#### Step 1.2: Update `query-hourly-interval-reads`

**Replace existing function (lines 88-104) with:**

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

**Key Changes:**
- ✅ Multi-arity function (single and two arity)
- ✅ Uses `build-date-clause` helper
- ✅ `some?` check for explicit nil handling
- ✅ Backward compatible (single arity calls two arity with nil)

## Phase 2: Business Logic Layer Implementation

### File: `app/functions/src/admin/core.cljs`

#### Step 2.1: Update `interval-reads-for-user-uid`

**Replace existing function (lines 20-43) with:**

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
                                (map (comp numbers/round-3dp :sum) values))))
          (sort-by :date-time)
          (clj->js)))))
```

**Key Changes:**
- ✅ Multi-arity function (single and two arity)
- ✅ Passes `date-str` through to `query-hourly-interval-reads`
- ✅ Backward compatible (single arity calls two arity with nil)

#### Step 2.2: Update `fetch-user-interval-handler`

**Replace existing function (lines 45-54) with:**

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

**Key Changes:**
- ✅ Extracts optional `date` from `(:data request)`
- ✅ Normalizes empty string to nil
- ✅ Uses conditional to call correct arity
- ✅ Backward compatible (handles missing date field)

## Phase 3: Middleware Verification

### File: `app/functions/src/core.cljs`

**No Changes Required** ✅

The middleware chain already handles optional data fields correctly:
- `auth/wrap-admin-authentication` - passes `:data` through unchanged
- `fiskil/token-middleware` - preserves all `:data` fields including optional `date`

**Verification:**
- Lines 79-81: Handler wrapper already set up correctly
- Line 143: Function export already configured correctly

## Phase 4: Test Implementation

### Test File 1: `app/functions/test/timescaledb/timescale_test.cljs`

**Create new file:**

```clojure
(ns functions.test.timescaledb.timescale-test
  (:require [functions.src.timescaledb.timescale :as sut]
            [functions.src.timescaledb.timescale :as timescale]
            [cljs.test :refer [deftest is testing use-fixtures]]
            [promesa.core :as p]))

(def mock-national-meter-id "NMI123")
(def mock-date-str "2024-11-20")

(def mock-query-results
  [{:read_start "2024-11-20T00:00:00.000Z"
    :register_suffix "E1"
    :sum 0.5}
   {:read_start "2024-11-20T01:00:00.000Z"
    :register_suffix "B1"
    :sum -0.2}])

(def captured-query (atom nil))
(def captured-params (atom nil))

(defn mock-fixture [f]
  (reset! captured-query nil)
  (reset! captured-params nil)
  (with-redefs [timescale/query! (fn [query params]
                                    (reset! captured-query query)
                                    (reset! captured-params params)
                                    (p/resolved mock-query-results))]
    (f)))

(use-fixtures :each mock-fixture)

(deftest test-query-hourly-interval-reads-without-date
  (testing "Queries without date parameter use CURRENT_DATE"
    (-> (sut/query-hourly-interval-reads mock-national-meter-id)
        (p/then
         (fn [result]
           (is (= result mock-query-results))
           (is (some? @captured-query))
           (is (string/includes? @captured-query "CURRENT_DATE"))
           (is (= @captured-params [mock-national-meter-id])))))))

(deftest test-query-hourly-interval-reads-with-date
  (testing "Queries with date parameter use provided date"
    (-> (sut/query-hourly-interval-reads mock-national-meter-id mock-date-str)
        (p/then
         (fn [result]
           (is (= result mock-query-results))
           (is (some? @captured-query))
           (is (string/includes? @captured-query "$2::DATE"))
           (is (= @captured-params [mock-national-meter-id mock-date-str])))))))

(deftest test-query-hourly-interval-reads-empty-results
  (testing "Handles empty result set"
    (with-redefs [timescale/query! (fn [_ _] (p/resolved []))]
      (-> (sut/query-hourly-interval-reads mock-national-meter-id)
          (p/then
           (fn [result]
             (is (= result []))))))))

(deftest test-query-hourly-interval-reads-database-error
  (testing "Handles database errors correctly"
    (with-redefs [timescale/query! (fn [_ _] (p/rejected (js/Error. "Database error")))]
      (-> (sut/query-hourly-interval-reads mock-national-meter-id)
          (p/catch
           (fn [err]
             (is (some? err))
             (is (= (:national-metering-id (ex-data err)) mock-national-meter-id))))))))
```

### Test File 2: `app/functions/test/admin/core_test.cljs`

**Create new file:**

```clojure
(ns functions.test.admin.core-test
  (:require [functions.src.admin.core :as sut]
            [functions.src.google.firebase.firestore :as firestore]
            [functions.src.timescaledb.timescale :as timescale]
            [cljs.test :refer [deftest is testing use-fixtures]]
            [promesa.core :as p]))

(def mock-user-uid "user-123")
(def mock-national-metering-id "NMI123")
(def mock-date-str "2024-11-20")

(def mock-user-data
  {:national-metering-ids [mock-national-metering-id "NMI456"]})

(def mock-interval-reads-results
  [{:read_start "2024-11-20T00:00:00.000Z"
    :register_suffix "E1"
    :sum 0.5}
   {:read_start "2024-11-20T00:00:00.000Z"
    :register_suffix "B1"
    :sum -0.2}
   {:read_start "2024-11-20T01:00:00.000Z"
    :register_suffix "E1"
    :sum 0.4}])

(def captured-query-args (atom nil))

(defn mock-fixture [f]
  (reset! captured-query-args nil)
  (with-redefs [firestore/get-user (fn [_] (p/resolved mock-user-data))
                timescale/query-hourly-interval-reads (fn [nmi & [date]]
                                                         (reset! captured-query-args [nmi date])
                                                         (p/resolved mock-interval-reads-results))]
    (f)))

(use-fixtures :each mock-fixture)

(deftest test-interval-reads-for-user-uid-without-date
  (testing "Queries without date parameter"
    (-> (sut/interval-reads-for-user-uid mock-user-uid mock-national-metering-id)
        (p/then
         (fn [result]
           (is (array? result))
           (is (> (.-length result) 0))
           (is (= (first @captured-query-args) mock-national-metering-id))
           (is (nil? (second @captured-query-args))))))))

(deftest test-interval-reads-for-user-uid-with-date
  (testing "Queries with date parameter"
    (-> (sut/interval-reads-for-user-uid mock-user-uid mock-national-metering-id mock-date-str)
        (p/then
         (fn [result]
           (is (array? result))
           (is (= (first @captured-query-args) mock-national-metering-id))
           (is (= (second @captured-query-args) mock-date-str)))))))

(deftest test-interval-reads-for-user-uid-user-no-access
  (testing "Returns empty list when user doesn't have access to NMI"
    (with-redefs [firestore/get-user (fn [_] (p/resolved {:national-metering-ids ["OTHER_NMI"]}))]
      (-> (sut/interval-reads-for-user-uid mock-user-uid mock-national-metering-id)
          (p/then
           (fn [result]
             (is (= result #js []))
             (is (nil? @captured-query-args))))))))

(deftest test-fetch-user-interval-handler-without-date
  (testing "Handler without date in request"
    (let [request {:data {:user-uid mock-user-uid
                           :national-metering-id mock-national-metering-id}}]
      (-> (sut/fetch-user-interval-handler request nil)
          (p/then
           (fn [result]
             (is (array? result))
             (is (nil? (second @captured-query-args)))))))))

(deftest test-fetch-user-interval-handler-with-date
  (testing "Handler with date in request"
    (let [request {:data {:user-uid mock-user-uid
                          :national-metering-id mock-national-metering-id
                          :date mock-date-str}}]
      (-> (sut/fetch-user-interval-handler request nil)
          (p/then
           (fn [result]
             (is (array? result))
             (is (= (second @captured-query-args) mock-date-str))))))))

(deftest test-fetch-user-interval-handler-empty-string-date
  (testing "Handler normalizes empty string date to nil"
    (let [request {:data {:user-uid mock-user-uid
                          :national-metering-id mock-national-metering-id
                          :date ""}}]
      (-> (sut/fetch-user-interval-handler request nil)
          (p/then
           (fn [result]
             (is (array? result))
             (is (nil? (second @captured-query-args)))))))))
```

## Phase 5: Testing & Verification

### Step 5.1: Run Tests

```bash
npm test
```

**Expected Results:**
- ✅ All new tests pass
- ✅ Existing tests still pass (backward compatibility)
- ✅ No test failures

### Step 5.2: Manual Testing

#### Test 1: Backward Compatibility (No Date)

```javascript
// Firebase Function Call
fetchUserIntervalReads({
  data: {
    userUid: "user-123",
    nationalMeteringId: "NMI123"
    // No date field
  }
})
```

**Expected:** Returns data using CURRENT_DATE ✅

#### Test 2: New Functionality (With Date)

```javascript
// Firebase Function Call
fetchUserIntervalReads({
  data: {
    userUid: "user-123",
    nationalMeteringId: "NMI123",
    date: "2024-11-20"
  }
})
```

**Expected:** Returns data using provided date ✅

#### Test 3: Edge Cases

- Empty string date → Should normalize to nil → Uses CURRENT_DATE
- Invalid date format → PostgreSQL will handle validation
- Future date → Should work (PostgreSQL handles)
- Very old date → Should work (PostgreSQL handles)

## Implementation Checklist

### Code Changes
- [ ] **Phase 1.1**: Add `build-date-clause` helper function to `timescale.cljs`
- [ ] **Phase 1.2**: Update `query-hourly-interval-reads` with multi-arity
- [ ] **Phase 2.1**: Update `interval-reads-for-user-uid` with multi-arity
- [ ] **Phase 2.2**: Update `fetch-user-interval-handler` to extract and pass date
- [ ] **Phase 3**: Verify middleware chain (no changes needed)

### Tests
- [ ] **Phase 4.1**: Create `timescale_test.cljs` with all test cases
- [ ] **Phase 4.2**: Create `admin/core_test.cljs` with all test cases
- [ ] **Phase 5.1**: Run `npm test` and verify all tests pass
- [ ] **Phase 5.2**: Manual testing via Firebase emulator

### Documentation
- [ ] Update function docstrings with new parameter documentation
- [ ] Verify all docstrings are clear and accurate
- [ ] Document date format requirements (YYYY-MM-DD)

### Verification
- [ ] Backward compatibility verified (existing calls work)
- [ ] New functionality verified (date parameter works)
- [ ] Error handling verified (database errors handled)
- [ ] Edge cases verified (empty string, nil, etc.)

## Code Style Guidelines

Following Clojure best practices:
1. ✅ **Functional Style**: Pure functions, immutable data
2. ✅ **Multi-Arity**: Function overloading for optional parameters
3. ✅ **`some?` Checks**: Explicit nil handling (not fragile nil passing)
4. ✅ **Helper Functions**: Clear separation of concerns (`build-date-clause`)
5. ✅ **Documentation**: Clear docstrings explaining optionality
6. ✅ **Backward Compatible**: No breaking changes

## Backward Compatibility Guarantee

✅ **Fully Backward Compatible**

- Existing calls without date parameter → Single arity → Uses CURRENT_DATE
- New calls with date parameter → Two arity → Uses provided date
- Multi-arity dispatch ensures correct function selection
- No breaking changes to function signatures
- Default behavior (CURRENT_DATE) preserved

## Error Handling

- **Database Errors**: Wrapped with context (national-metering-id, date-str, error)
- **Firestore Errors**: Propagated through promise chain
- **Invalid Dates**: PostgreSQL handles validation (can add ClojureScript validation later)
- **Missing Parameters**: Handler extracts safely (nil if missing)

## Future Enhancements (Out of Scope)

- Date range queries (start-date, end-date)
- Date format validation middleware
- Timezone conversion utilities
- Option/Maybe type implementation (if desired for more type safety)

## Summary

This implementation plan provides:
- ✅ Complete code changes with examples
- ✅ Comprehensive test suite
- ✅ Backward compatibility guarantee
- ✅ Clear step-by-step instructions
- ✅ Verification checklist

**Estimated Implementation Time:** 2-3 hours
**Risk Level:** Low (backward compatible, well-tested)

