# Test Implementation Plan: Optional Date Parameter

## Overview
Comprehensive test suite for the optional date parameter functionality, following existing test patterns in `app/functions/test/`.

## Test File Structure

Following the existing pattern, create:
- `app/functions/test/admin/core_test.cljs` - Tests for admin functions
- `app/functions/test/timescaledb/timescale_test.cljs` - Tests for TimescaleDB functions

## Test Patterns Observed

### Existing Test Patterns
1. **Namespace**: `functions.test.<module-path>-test`
2. **Aliases**: Use `sut` (system under test) for the module being tested
3. **Mock Data**: Define at top level with descriptive names
4. **Fixtures**: Use `use-fixtures` with `with-redefs` for shared mocking
5. **Async Testing**: Use `promesa.core` with `p/then` for async assertions
6. **Test Structure**: `deftest` with `testing` blocks for organization
7. **Error Testing**: Test both success and failure paths

## Test Coverage Plan

### 1. Tests for `query-hourly-interval-reads` (timescale.cljs)

**File**: `app/functions/test/timescaledb/timescale_test.cljs`

#### Test Cases:

1. **test-query-hourly-interval-reads-without-date**
   - Calls function with only `national-meter-id`
   - Verifies SQL query uses `CURRENT_DATE`
   - Verifies correct parameters passed to `query!`

2. **test-query-hourly-interval-reads-with-date**
   - Calls function with `national-meter-id` and `date-str`
   - Verifies SQL query uses parameterized date (`$2`)
   - Verifies date parameter is passed correctly

3. **test-query-hourly-interval-reads-database-error**
   - Mocks database error
   - Verifies exception is properly wrapped with context

4. **test-query-hourly-interval-reads-empty-results**
   - Mocks empty result set
   - Verifies function returns empty list

5. **test-query-hourly-interval-reads-with-sample-data**
   - Mocks sample database results
   - Verifies results are returned correctly

### 2. Tests for `interval-reads-for-user-uid` (admin/core.cljs)

**File**: `app/functions/test/admin/core_test.cljs`

#### Test Cases:

1. **test-interval-reads-for-user-uid-without-date**
   - Calls function without date parameter
   - Mocks `firestore/get-user` to return user with NMI
   - Mocks `query-hourly-interval-reads` to return sample data
   - Verifies results are transformed correctly (grouped, formatted)
   - Verifies JavaScript-compatible format returned

2. **test-interval-reads-for-user-uid-with-date**
   - Calls function with date parameter
   - Verifies date parameter is passed to `query-hourly-interval-reads`
   - Verifies results are transformed correctly

3. **test-interval-reads-for-user-uid-user-no-access**
   - User doesn't have access to the NMI
   - Verifies empty list is returned
   - Verifies `query-hourly-interval-reads` is not called

4. **test-interval-reads-for-user-uid-firestore-error**
   - Mocks Firestore error
   - Verifies error is propagated correctly

5. **test-interval-reads-for-user-uid-database-error**
   - Mocks database query error
   - Verifies error is propagated correctly

6. **test-interval-reads-for-user-uid-result-transformation**
   - Tests result transformation logic
   - Multiple register suffixes (E1, B1)
   - Verifies grouping by date-time
   - Verifies sorting by date-time
   - Verifies rounding to 3 decimal places

### 3. Tests for `fetch-user-interval-handler` (admin/core.cljs)

**File**: `app/functions/test/admin/core_test.cljs` (same file as above)

#### Test Cases:

1. **test-fetch-user-interval-handler-without-date**
   - Calls handler without date in request data
   - Verifies handler extracts `user-uid` and `national-metering-id`
   - Verifies calls `interval-reads-for-user-uid` without date
   - Verifies successful response

2. **test-fetch-user-interval-handler-with-date**
   - Calls handler with date in request data
   - Verifies handler extracts date from `(:data request)`
   - Verifies calls `interval-reads-for-user-uid` with date
   - Verifies successful response

3. **test-fetch-user-interval-handler-missing-user-uid**
   - Request missing `user-uid`
   - Verifies error handling

4. **test-fetch-user-interval-handler-missing-national-metering-id**
   - Request missing `national-metering-id`
   - Verifies error handling

5. **test-fetch-user-interval-handler-database-error**
   - Mocks database error
   - Verifies error is written to Firestore via `write-user-error`
   - Verifies error is propagated

6. **test-fetch-user-interval-handler-invalid-date-format**
   - Request with invalid date format
   - Verifies behavior (may pass through or validate)

## Mock Data Structure

### Mock Database Results
```clojure
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
```

### Mock User Data
```clojure
(def mock-user-data
  {:national-metering-ids ["NMI123" "NMI456"]})

(def mock-request-data
  {:user-uid "user-123"
   :national-metering-id "NMI123"
   :date "2024-11-20"})  ; Optional
```

## Test Implementation Details

### Test File: `app/functions/test/timescaledb/timescale_test.cljs`

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
    :sum 0.5}])

;; Mock the query! function
(defn mock-fixture [f]
  (with-redefs [timescale/query! (fn [query params]
                                    (p/resolved mock-query-results))]
    (f)))

(use-fixtures :each mock-fixture)

(deftest test-query-hourly-interval-reads-without-date
  (testing "Queries without date parameter use CURRENT_DATE"
    (-> (sut/query-hourly-interval-reads mock-national-meter-id)
        (p/then
         (fn [result]
           (is (= result mock-query-results)))))))

(deftest test-query-hourly-interval-reads-with-date
  (testing "Queries with date parameter use provided date"
    (-> (sut/query-hourly-interval-reads mock-national-meter-id mock-date-str)
        (p/then
         (fn [result]
           (is (= result mock-query-results)))))))
```

### Test File: `app/functions/test/admin/core_test.cljs`

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
    :sum -0.2}])

(defn mock-fixture [f]
  (with-redefs [firestore/get-user (fn [_] (p/resolved mock-user-data))
                timescale/query-hourly-interval-reads (fn [nmi & [date]]
                                                         (p/resolved mock-interval-reads-results))]
    (f)))

(use-fixtures :each mock-fixture)

(deftest test-interval-reads-for-user-uid-without-date
  (testing "Queries without date parameter"
    (-> (sut/interval-reads-for-user-uid mock-user-uid mock-national-metering-id)
        (p/then
         (fn [result]
           (is (array? result))
           (is (> (.-length result) 0)))))))

(deftest test-interval-reads-for-user-uid-with-date
  (testing "Queries with date parameter"
    (-> (sut/interval-reads-for-user-uid mock-user-uid mock-national-metering-id mock-date-str)
        (p/then
         (fn [result]
           (is (array? result)))))))

(deftest test-fetch-user-interval-handler-without-date
  (testing "Handler without date in request"
    (let [request {:data {:user-uid mock-user-uid
                          :national-metering-id mock-national-metering-id}}]
      (-> (sut/fetch-user-interval-handler request nil)
          (p/then
           (fn [result]
             (is (array? result))))))))

(deftest test-fetch-user-interval-handler-with-date
  (testing "Handler with date in request"
    (let [request {:data {:user-uid mock-user-uid
                          :national-metering-id mock-national-metering-id
                          :date mock-date-str}}]
      (-> (sut/fetch-user-interval-handler request nil)
          (p/then
           (fn [result]
             (is (array? result))))))))
```

## Edge Cases to Test

1. **Date Format Validation**
   - Valid: `"2024-11-20"`
   - Invalid formats: `"11/20/2024"`, `"2024-11-20T00:00:00Z"`, `"invalid"`
   - Empty string: `""`
   - Nil: `nil`

2. **Boundary Dates**
   - Very old dates (before data exists)
   - Future dates
   - Today's date
   - Date exactly 365 days ago

3. **Empty Results**
   - No data for the NMI
   - No data for the date range
   - User has no NMIs

4. **Error Scenarios**
   - Database connection errors
   - Firestore errors
   - Invalid NMI format
   - Missing required parameters

## Test Execution

### Running Tests
```bash
npm test
```

This compiles and runs all tests via ShadowCLJS `:test` build target.

### Test Organization
- Group related tests with `testing` blocks
- Use descriptive test names
- Follow existing naming conventions (`test-<function-name>-<scenario>`)

## Integration with Existing Tests

- Follow same patterns as `register_user_test.cljs`
- Use same mocking strategies
- Maintain consistency with existing test structure
- Add to existing test suite without breaking changes

## Success Criteria

✅ All tests pass  
✅ Backward compatibility verified (tests without date parameter)  
✅ New functionality verified (tests with date parameter)  
✅ Error cases handled correctly  
✅ Edge cases covered  
✅ Code coverage for all modified functions  
✅ Tests follow existing code style and patterns

