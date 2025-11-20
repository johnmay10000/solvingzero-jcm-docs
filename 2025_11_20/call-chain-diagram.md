# Complete Call Chain Diagram: fetchUserIntervalReads

## Visual Call Flow

```
┌─────────────────────────────────────────────────────────────┐
│ Firebase Cloud Function Call                                │
│ fetchUserIntervalReads({                                    │
│   data: {                                                    │
│     user-uid: "...",                                         │
│     national-metering-id: "...",                             │
│     date: "2024-11-20"  // Optional                         │
│   }                                                          │
│ })                                                           │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ core.cljs:143                                                │
│ :fetchUserIntervalReads                                      │
│   (create-oncall-handler                                     │
│    fetch-user-interval-reads-wrapped-handler                │
│    [token/client-secret timescale/password]                 │
│    "256MiB")                                                 │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ core.cljs:79-81                                             │
│ fetch-user-interval-reads-wrapped-handler                  │
│   (-> admin-fn/fetch-user-interval-handler                │
│       auth/wrap-admin-authentication                        │
│       fiskil/token-middleware)                              │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ MIDDLEWARE LAYER                                            │
│                                                             │
│ 1. auth/wrap-admin-authentication                          │
│    - Validates admin token                                  │
│    - Converts JS request → Clojure map                     │
│    - Passes :data through unchanged                        │
│                                                             │
│ 2. fiskil/token-middleware                                 │
│    - Fetches Fiskil API token                              │
│    - Adds :token to request                                │
│    - Converts :data from JS → Clojure                      │
│    - Preserves all :data fields (including date)           │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ admin/core.cljs:45-54                                       │
│ fetch-user-interval-handler                                 │
│                                                             │
│ Input: {:data {:user-uid "...",                            │
│                :national-metering-id "...",                 │
│                :date "2024-11-20"}  ; Optional             │
│        :token "..."}                                       │
│                                                             │
│ Extracts: user-uid, national-metering-id, date (optional)  │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ admin/core.cljs:20-43                                       │
│ interval-reads-for-user-uid                                 │
│                                                             │
│ - Validates user has access to NMI                         │
│ - Calls query-hourly-interval-reads                       │
│ - Transforms results (groups, formats)                     │
│ - Returns JavaScript-compatible format                     │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ timescale.cljs:88-104                                       │
│ query-hourly-interval-reads                                 │
│                                                             │
│ - Builds SQL query                                         │
│ - Uses CURRENT_DATE or provided date-str                   │
│ - Executes parameterized query                            │
│ - Returns raw database results                             │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ TimescaleDB                                                 │
│                                                             │
│ SELECT date_trunc('hour', read_start_date) AS read_start, │
│        register_suffix,                                     │
│        sum(read_value) AS sum                              │
│ FROM interval_read                                          │
│ WHERE national_metering_id = $1                            │
│ AND read_start_date >= $2::DATE - INTERVAL '2 days'       │
│      - INTERVAL '365 days'                                 │
│ AND read_start_date < $2::DATE - INTERVAL '2 days'        │
│ GROUP BY 1,2                                                │
│ ORDER BY read_start ASC                                     │
└─────────────────────────────────────────────────────────────┘
```

## Data Flow Through Middleware

### Input (JavaScript)
```javascript
{
  data: {
    user-uid: "user123",
    national-metering-id: "NMI123",
    date: "2024-11-20"  // Optional
  }
}
```

### After auth/wrap-admin-authentication (Clojure)
```clojure
{
  :data {
    :user-uid "user123"
    :national-metering-id "NMI123"
    :date "2024-11-20"  ; Optional, preserved
  }
  :auth {...}
}
```

### After fiskil/token-middleware (Clojure)
```clojure
{
  :data {
    :user-uid "user123"
    :national-metering-id "NMI123"
    :date "2024-11-20"  ; Optional, preserved
  }
  :token "fiskil-api-token-xyz"
}
```

### Handler Receives
```clojure
{
  :data {
    :user-uid "user123"
    :national-metering-id "NMI123"
    :date "2024-11-20"  ; Can extract this
  }
  :token "fiskil-api-token-xyz"
}
```

## Key Points

1. **Middleware Preserves Data**: The `date` parameter flows through middleware unchanged
2. **No Middleware Changes Needed**: Existing middleware already handles this correctly
3. **Extraction Point**: Date is extracted in `fetch-user-interval-handler` from `(:data request)`
4. **Backward Compatible**: If `date` is not provided, `nil` flows through and defaults to `CURRENT_DATE`
   - Uses `some?` checks for explicit nil handling (see `option-maybe-patterns.md` for alternatives)
   - Multi-arity functions provide type-safe optional parameters

## Implementation Impact

- ✅ **core.cljs**: No changes required - middleware handles data flow
- ✅ **admin/core.cljs**: Extract date from `(:data request)` and pass through
- ✅ **timescale.cljs**: Accept optional date parameter and use in SQL

