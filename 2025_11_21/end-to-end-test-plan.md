# End-to-End Test Plan: fetchUserIntervalReads

## Overview

Create an end-to-end test that verifies the complete call chain from the Firebase function entry point through middleware, handlers, business logic, and database, returning the expected response schema.

## Current Test Coverage Gap

**Existing Tests:**
- ✅ Unit tests for `query-hourly-interval-reads` (database layer)
- ✅ Unit tests for `interval-reads-for-user-uid` (business logic)
- ✅ Unit tests for `fetch-user-interval-handler` (handler layer)
- ❌ **Missing**: End-to-end test covering the full call chain

## Complete Call Chain

```
Firebase Function: :fetchUserIntervalReads
  ↓
create-oncall-handler wrapper
  ↓
Middleware Chain:
  1. fiskil/token-middleware (adds Fiskil token)
  2. auth/wrap-admin-authentication (checks admin flag)
  ↓
Handler: fetch-user-interval-handler
  ↓
Business Logic: interval-reads-for-user-uid
  ↓
Database: query-hourly-interval-reads
  ↓
Response: JavaScript array of interval reads
```

## Test Requirements

### What to Test

1. **Full Call Chain**
   - Firebase function wrapper execution
   - Middleware execution (token-middleware, wrap-admin-authentication)
   - Handler execution
   - Business logic execution
   - Database query execution
   - Response transformation

2. **Request/Response Schema**
   - Request format matches Firebase onCall format
   - Response format matches expected JavaScript array format
   - Data transformation through each layer

3. **With and Without Date Parameter**
   - Test backward compatibility (no date)
   - Test new functionality (with date)

4. **Error Scenarios**
   - Middleware failures (missing token, not admin)
   - Firestore errors
   - Database errors
   - Error propagation through layers

## Implementation Approach

### Option 1: Test the Wrapped Handler Directly (Recommended)

**Pros:**
- Tests actual middleware chain
- Tests actual handler
- No need to mock Firebase SDK
- Faster execution
- More reliable

**Cons:**
- Doesn't test Firebase SDK integration
- Requires mocking Firebase Admin SDK

**Implementation:**
- Create the wrapped handler exactly as in `core.cljs`
- Mock external dependencies (Firestore, TimescaleDB, token service)
- Test the wrapped handler function directly

### Option 2: Test via Firebase Emulator

**Pros:**
- Tests actual Firebase SDK integration
- Most realistic test

**Cons:**
- Requires Firebase emulator setup
- Slower execution
- More complex setup
- Harder to isolate failures

**Implementation:**
- Use Firebase emulator
- Make actual HTTP calls
- More integration test than unit test

## Recommended Approach: Option 1

Test the wrapped handler directly by:
1. Recreating the middleware chain in test
2. Mocking all external dependencies
3. Verifying request/response flow
4. Validating response schema

## Test Structure

### Test File: `app/functions/test/core/fetch_user_interval_reads_e2e_test.cljs`

**Test Cases:**

1. **test-fetch-user-interval-reads-e2e-without-date**
   - Full call chain without date parameter
   - Verifies middleware execution
   - Verifies handler execution
   - Verifies response schema

2. **test-fetch-user-interval-reads-e2e-with-date**
   - Full call chain with date parameter
   - Verifies date parameter flows through all layers
   - Verifies response schema

3. **test-fetch-user-interval-reads-e2e-middleware-token-failure**
   - Token middleware fails (no token available)
   - Verifies error response

4. **test-fetch-user-interval-reads-e2e-middleware-auth-failure**
   - Admin authentication fails (not admin)
   - Verifies error response

5. **test-fetch-user-interval-reads-e2e-firestore-error**
   - Firestore error propagates correctly
   - Verifies error handling

6. **test-fetch-user-interval-reads-e2e-database-error**
   - Database error propagates correctly
   - Verifies error handling

## Response Schema Validation

### Expected Response Format

```javascript
[
  {
    "date-time": "2024-11-20T00:00:00.000Z",
    "E1": 0.5,
    "B1": -0.2
  },
  {
    "date-time": "2024-11-20T01:00:00.000Z",
    "E1": 0.4
  }
]
```

**Validation Points:**
- Array format (JavaScript array)
- Each item has `date-time` field
- Each item has register suffix keys (E1, B1, etc.)
- Values are numbers (rounded to 3 decimal places)
- Sorted by date-time ascending

## Mocking Strategy

### External Dependencies to Mock

1. **Firebase Admin SDK**
   - Not needed if testing wrapped handler directly

2. **Fiskil Token Service** (`token/get-token`)
   - Mock to return valid token

3. **Firestore** (`firestore/get-user`, `firestore/write-user-error`)
   - Mock user data
   - Mock error scenarios

4. **TimescaleDB** (`timescale/query-hourly-interval-reads` or `timescale/query!`)
   - Mock database query results
   - Mock database errors

## Implementation Checklist

- [ ] Create test file structure
- [ ] Set up middleware chain recreation
- [ ] Create mock fixtures for all dependencies
- [ ] Implement test for successful flow without date
- [ ] Implement test for successful flow with date
- [ ] Implement test for middleware failures
- [ ] Implement test for Firestore errors
- [ ] Implement test for database errors
- [ ] Add response schema validation
- [ ] Verify all tests pass
- [ ] Document test approach

## Benefits of End-to-End Test

1. **Confidence**: Verifies entire flow works together
2. **Regression Prevention**: Catches integration issues
3. **Documentation**: Shows how components work together
4. **Schema Validation**: Ensures response format is correct
5. **Middleware Testing**: Verifies middleware chain works correctly

## Challenges

1. **Firebase SDK Mocking**: May need to mock Firebase Admin initialization
2. **Middleware Chain**: Need to recreate exact middleware chain
3. **Promise Chains**: Complex promise chains need careful testing
4. **Response Format**: Need to validate JavaScript array format

## Solution

The implementation will:
- Recreate the middleware chain exactly as in `core.cljs`
- Mock all external dependencies
- Test the wrapped handler function directly
- Validate request/response flow and schema
- Cover both success and error scenarios

