# Optional Date Parameter Implementation Plan

## Overview
Add support for an optional `date` parameter to `query-hourly-interval-reads` and `interval-reads-for-user-uid` functions. When provided, this date will replace `CURRENT_DATE` in the SQL query, allowing queries for historical date ranges.

## Current Implementation Analysis

### Complete Function Call Chain
```
core.cljs:143
  :fetchUserIntervalReads (Firebase Function Export)
    ↓
core.cljs:79-81
  fetch-user-interval-reads-wrapped-handler
    (Middleware: auth/wrap-admin-authentication → fiskil/token-middleware)
    ↓
admin/core.cljs:45-54
  fetch-user-interval-handler
    ↓
admin/core.cljs:20-43
  interval-reads-for-user-uid
    ↓
timescale.cljs:88-104
  query-hourly-interval-reads
    ↓
TimescaleDB SQL Query (uses CURRENT_DATE)
```

### Middleware Behavior
- **`auth/wrap-admin-authentication`**: Validates admin token, converts JS request to Clojure map
- **`fiskil/token-middleware`**: Adds Fiskil token to request, converts `:data` to Clojure map
- **Result**: Handler receives Clojure map with `:data` (containing user-uid, national-metering-id) and `:token`
- **Date Parameter Flow**: Will be in `(:data request)` and flows through middleware unchanged

### Current SQL Query Logic
```sql
WHERE national_metering_id = $1 
AND read_start_date >= CURRENT_DATE - INTERVAL '2 days' - INTERVAL '365 days' 
AND read_start_date < CURRENT_DATE - INTERVAL '2 days'
```

This queries:
- **Lower bound**: 365 days before 2 days ago
- **Upper bound**: 2 days ago (exclusive)

### Date Format in Codebase
- Dates are passed as strings in `YYYY-MM-DD` format (see `melbourne-date-today` in `usagesync.cljs`)
- Uses `cljs-time` library for date manipulation
- SQL expects date values in PostgreSQL date format

## Design Decision: Separate Functions vs Optional Parameters

### Recommendation: **Multi-Arity Functions (Optional Parameters)**

**Rationale:**
1. **Backward Compatibility**: Existing code continues to work without changes
2. **Clojure Idioms**: Multi-arity functions are standard practice for optional parameters
3. **Minimal Code Duplication**: Single implementation with conditional logic
4. **Clean API**: One function name, clear intent

**Alternative Considered:**
- Separate functions (e.g., `query-hourly-interval-reads` and `query-hourly-interval-reads-for-date`)
- **Rejected because**: Creates API surface area bloat, requires maintaining two similar functions

## Implementation Plan

### Step 1: Update `query-hourly-interval-reads` (timescale.cljs)

**Changes:**
- Add multi-arity function signature
- Accept optional `date-str` parameter (string in `YYYY-MM-DD` format)
- When `date-str` is provided, use it in SQL; otherwise use `CURRENT_DATE`
- Convert date string to PostgreSQL date format for SQL parameterization

**Function Signature:**
```clojure
;; Single arity (backward compatible)
(query-hourly-interval-reads national-meter-id)

;; Two arity (with optional date)
(query-hourly-interval-reads national-meter-id date-str)
```

**SQL Changes:**
- Use parameterized query: `$2` for the date when provided
- Conditional SQL generation based on whether date is provided
- Format: `date-str::DATE` in PostgreSQL

### Step 2: Update `interval-reads-for-user-uid` (admin/core.cljs)

**Changes:**
- Add optional `date-str` parameter
- Pass `date-str` through to `query-hourly-interval-reads`
- Maintain backward compatibility

**Function Signature:**
```clojure
;; Single arity (backward compatible)
(interval-reads-for-user-uid user-uid national-metering-id)

;; Two arity (with optional date)
(interval-reads-for-user-uid user-uid national-metering-id date-str)
```

### Step 3: Update `fetch-user-interval-handler` (admin/core.cljs)

**Changes:**
- Extract optional `date` from `(:data request)`
- Pass `date` to `interval-reads-for-user-uid` if present
- Handle date validation (optional - can be added later)

**Request Format (from Firebase Function):**
```javascript
{
  data: {
    user-uid: "...",
    national-metering-id: "...",
    date: "2024-11-20"  // Optional, YYYY-MM-DD format
  }
}
```

**After Middleware Processing:**
The middleware chain in `core.cljs` converts this to a Clojure map:
```clojure
{
  :data {
    :user-uid "..."
    :national-metering-id "..."
    :date "2024-11-20"  ; Optional
  }
  :token "fiskil-api-token"
}
```

### Step 4: Verify Middleware Chain (core.cljs)

**No Changes Required** ✅
- The middleware chain already passes `:data` through unchanged
- The `date` parameter will automatically flow through:
  1. `auth/wrap-admin-authentication` - passes `:data` through
  2. `fiskil/token-middleware` - adds `:token`, preserves `:data`
  3. Handler receives complete request with optional `date` in `:data`

## Technical Considerations

### Date Handling
1. **Input Format**: Accept `YYYY-MM-DD` string format (consistent with codebase)
2. **SQL Format**: PostgreSQL accepts `YYYY-MM-DD` format directly
3. **Timezone**: Current implementation uses `CURRENT_DATE` (server timezone). When providing date, should we:
   - Use as-is (assume UTC or server timezone)
   - Convert to Melbourne timezone (like `melbourne-date-today`)
   - **Recommendation**: Use as-is for now, document timezone behavior

### SQL Parameterization
- Use `$2` parameter for date when provided
- Ensure proper SQL injection prevention (already using parameterized queries)

### Error Handling
- If invalid date format provided, let PostgreSQL handle validation
- Consider adding date format validation in ClojureScript (optional enhancement)

### Testing Considerations
- Test with date parameter
- Test without date parameter (backward compatibility)
- Test with invalid date formats
- Test edge cases (future dates, very old dates)

## Code Style Guidelines

Following Clojure best practices:
1. **Functional Style**: Pure functions, immutable data
2. **Multi-Arity**: Use function overloading for optional parameters
3. **Documentation**: Update docstrings to reflect new parameter
4. **Naming**: Use descriptive names (`date-str` to indicate string format)
5. **Optional Value Handling**: Use `some?` for explicit nil checks (see `option-maybe-patterns.md` for Option/Maybe alternatives)

### Optional Value Pattern

Instead of fragile `nil` handling, we use:
- **Multi-arity functions** - Type-safe optional parameters
- **`some?` checks** - Explicit nil handling
- **Helper functions** - Separate query building logic

See `option-maybe-patterns.md` for detailed analysis of Option/Maybe patterns in Clojure.

## Backward Compatibility

✅ **Fully Backward Compatible**
- All existing calls without date parameter continue to work
- No breaking changes to function signatures
- Default behavior (using CURRENT_DATE) preserved

## Files to Modify

1. `app/functions/src/timescaledb/timescale.cljs`
   - Update `query-hourly-interval-reads` function

2. `app/functions/src/admin/core.cljs`
   - Update `interval-reads-for-user-uid` function
   - Update `fetch-user-interval-handler` function

3. `app/functions/src/core.cljs`
   - **No changes required** - middleware already handles data flow correctly
   - Document that date parameter flows through middleware chain

## Testing Strategy

### Test Files to Create
1. **`app/functions/test/timescaledb/timescale_test.cljs`**
   - Tests for `query-hourly-interval-reads` function
   - Tests with and without date parameter
   - Tests error handling

2. **`app/functions/test/admin/core_test.cljs`**
   - Tests for `interval-reads-for-user-uid` function
   - Tests for `fetch-user-interval-handler` function
   - Tests with and without date parameter
   - Tests result transformation logic
   - Tests error handling

### Test Coverage
- ✅ Backward compatibility (without date parameter)
- ✅ New functionality (with date parameter)
- ✅ Error scenarios (database errors, missing data)
- ✅ Edge cases (invalid dates, empty results, boundary dates)
- ✅ Result transformation (grouping, formatting, sorting)

### Test Execution
```bash
npm test
```

See `test-implementation-plan.md` for detailed test implementation plan.

## Future Enhancements (Out of Scope)

- Date range queries (start-date, end-date)
- Timezone conversion utilities
- Date validation middleware
- Date format normalization

