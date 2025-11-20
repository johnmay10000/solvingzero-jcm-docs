# SolvingZero Repository Review
**Date:** November 20, 2025  
**Reviewer:** AI Assistant  
**Branch:** SOL-165

## Executive Summary

SolvingZero is a ClojureScript-based energy management platform built on Firebase Cloud Functions, integrating with Australian energy data standards (CDR/Fiskil), NREL PVWatts, and various energy partners. The codebase follows a monorepo structure with three main applications: Cloud Functions, Frontend, and Admin tools.

## Repository Structure

### Monorepo Organization
- **Root**: npm workspaces managing three sub-projects
- **app/functions**: Firebase Cloud Functions (ClojureScript → Node.js)
- **app/frontend**: User-facing web application (ClojureScript → Browser)
- **app/admin**: Admin dashboard (ClojureScript → Browser)

### Key Directories
```
solvingzero/
├── app/
│   ├── functions/          # Cloud Functions backend
│   │   ├── src/           # ClojureScript source
│   │   │   ├── admin/     # Admin functions
│   │   │   ├── fiskil/    # Fiskil/CDR integration
│   │   │   ├── middleware/# Auth & middleware
│   │   │   ├── partners/  # Partner integrations
│   │   │   ├── plans/     # Energy plan logic
│   │   │   └── timescaledb/ # Time-series DB
│   │   └── test/          # Test files
│   ├── frontend/          # User-facing UI
│   └── admin/             # Admin dashboard
├── resources/             # Static assets (logos, images)
├── docs/                  # Documentation (new)
└── shadow-cljs.edn        # Build configuration
```

## Technology Stack

### Core Technologies
- **Language**: ClojureScript 1.11.132
- **Build Tool**: ShadowCLJS 2.28.15
- **Runtime**: Node.js 20 (Functions), Browser (Frontend/Admin)
- **Cloud Platform**: Google Cloud Platform / Firebase
- **Database**: 
  - Firestore (primary)
  - TimescaleDB (time-series data)
- **UI Framework**: Reagent (React wrapper)
- **Routing**: Reitit 0.7.2

### Key Dependencies
- **Firebase**: Admin SDK, Functions v2, Firestore
- **Energy APIs**: Fiskil (CDR), NREL PVWatts
- **Monitoring**: Sentry
- **Email**: SendGrid
- **Data Processing**: Promesa (async), Malli (validation)

## Architecture Overview

### Cloud Functions Architecture

The functions are organized around middleware composition:

```clojure
Handler → Auth Middleware → Fiskil Token Middleware → Business Logic
```

**Function Types:**
1. **onCall** (HTTP callable): User registration, data fetching, calculations
2. **onRequest** (HTTP): Webhooks, admin endpoints
3. **onSchedule** (Cron): Scheduled jobs (plan updates, bulk syncs)
4. **onMessagePublished** (Pub/Sub): Async processing (CSV generation)

**Key Functions:**
- `registerUser`: User onboarding with Fiskil integration
- `fetchUserBilling`: Retrieve billing data via CDR
- `fetchAllUserUsageData`: Time-series usage data
- `calculateCurrentPlanPrice`: Energy cost calculations
- `findRightPlansForUser`: Plan recommendation engine
- `bulkUserDataUpdate`: Scheduled sync (2 AM daily)
- `fetchAvailablePlans`: Scheduled plan updates (4 AM daily)

### Middleware Pattern

The codebase uses composable middleware:
- **auth/wrap-authentication**: User authentication
- **auth/wrap-admin-authentication**: Admin-only access
- **fiskil/token-middleware**: Fiskil API token management
- **misc/wrap-onrequest**: Request logging/error handling

### Data Flow

1. **User Registration**: 
   - User signs up → `registerUser` → Fiskil consent → Account linking
   
2. **Data Sync**:
   - Scheduled jobs fetch energy data → Store in TimescaleDB → Calculate insights
   
3. **Plan Recommendations**:
   - Usage data → Plan eligibility → Cost calculations → Recommendations

## Key Integrations

### Fiskil (Consumer Data Right)
- Energy account management
- Billing data retrieval
- Usage data synchronization
- DER (Distributed Energy Resources) data
- Webhook handling for real-time updates

### Partners
- **Solahart**: Solar installation partner
- **Spendwatt**: Energy management partner
- **Zeco**: Partner registration flow

### External Services
- **NREL PVWatts**: Solar generation calculations
- **SendGrid**: Email notifications
- **Google Cloud**: Vertex AI, Storage, Pub/Sub
- **Slack**: Notifications (mentioned in code)

## Development Workflow

### Branch Strategy
- Feature branches from `dev`
- PR to `dev` → Auto-deploy to Firebase dev (`solvingzero-db-62acd`)
- PR to `main` → Auto-deploy to Firebase prod (`solvingzero-db`)

### Build Targets
- `functions-local`: Development with REPL
- `functions-dev`: Dev deployment build
- `functions-prod`: Production build (optimized)
- `frontend-dev/prod`: Frontend builds
- `admin-dev/prod`: Admin builds
- `test`: Test compilation

### Local Development
1. Start TimescaleDB: `npm run start:timescale`
2. Watch functions: `npm run watch:functions`
3. Run compiled JS: `npm run start:functions`
4. Start emulators: `npm run start:emulators`

### Testing
- Test command: `npm test` (compiles and runs node tests)
- Test files in `app/functions/test/`
- Uses ShadowCLJS `:node-test` target

## Code Quality Observations

### Strengths
1. **Clear separation of concerns**: Middleware, handlers, business logic well-separated
2. **Consistent naming**: Functions follow clear patterns
3. **Composable architecture**: Middleware composition enables flexibility
4. **Comprehensive README**: Good documentation for setup and workflows
5. **Environment management**: Proper use of secrets and environment variables

### Areas for Improvement

1. **Missing CODE_STYLE.md**: User rules mention this file but it doesn't exist
   - Should document: functional style, immutable functions, ADTs, pattern matching

2. **Test Coverage**: Limited test files visible
   - Only `core_test.cljs` and `register_user_test.cljs` found
   - Consider expanding test coverage

3. **Type Safety**: Using Malli for validation but could benefit from more schema definitions

4. **Error Handling**: Sentry integration present, but error handling patterns could be more consistent

5. **Documentation**: 
   - Some modules have markdown docs (FISKIL.md, PARTNERS.md)
   - Could benefit from more inline documentation

6. **Code Organization**:
   - Some large files (e.g., `core.cljs` with 146 lines of exports)
   - Consider splitting large modules

## Security Considerations

### Current Practices
- ✅ Authentication middleware for protected endpoints
- ✅ Admin authentication for sensitive operations
- ✅ Secrets management via Firebase Functions secrets
- ✅ Sentry for error tracking
- ✅ Environment-based configuration

### Recommendations
- Review secret rotation policies
- Ensure all user inputs are validated (Malli schemas)
- Audit admin endpoints for proper authorization
- Review webhook security (admin-passcode)

## Performance Considerations

### Current Optimizations
- Memory allocation per function (256MiB - 2GiB)
- Timeout configuration (300-900 seconds)
- Scheduled jobs for heavy operations
- TimescaleDB for time-series data (efficient queries)

### Observations
- Some functions use 1-2GiB memory (heavy computations)
- Scheduled jobs run during off-peak hours (2 AM, 4 AM)
- Consider caching strategies for frequently accessed data

## Deployment Pipeline

### CI/CD
- GitHub Actions workflow: `.github/workflows/deploy-functions.yml`
- Auto-deploy on merge to `dev` and `main`
- Environment-specific deployments

### Deployment Process
1. Build ClojureScript → JavaScript
2. Deploy to Firebase Functions
3. Functions auto-scale based on load

## Recommendations

### Immediate Actions
1. **Create CODE_STYLE.md**: Document coding standards (functional style, ADTs, pattern matching)
2. **Expand test coverage**: Add tests for critical paths
3. **Add .gitignore entry for docs/**: Ensure daily work folders aren't committed

### Short-term Improvements
1. **Documentation**: Add more inline docs for complex functions
2. **Error handling**: Standardize error handling patterns
3. **Type schemas**: Expand Malli schema usage for validation
4. **Code splitting**: Break down large modules

### Long-term Considerations
1. **Monitoring**: Enhanced observability (beyond Sentry)
2. **Performance**: Caching layer for frequently accessed data
3. **Testing**: Integration tests for critical flows
4. **Documentation**: API documentation for functions

## Questions & Notes

1. **Flowstorm**: Mentioned in deps.edn but not actively used - consider removing or documenting
2. **Mixed TypeScript/ClojureScript**: Frontend has some `.tsx` files - migration in progress?
3. **Energy Manager Pilot**: Special registration flow for beta users
4. **Partner flows**: Multiple registration endpoints suggest white-label capabilities

## Conclusion

The SolvingZero codebase demonstrates a well-structured ClojureScript application with clear separation of concerns and good use of middleware patterns. The integration with Australian energy data standards (CDR/Fiskil) is well-organized. The main areas for improvement are documentation (especially CODE_STYLE.md), test coverage, and potentially breaking down larger modules.

The development workflow is streamlined with good CI/CD practices and clear deployment processes. The codebase appears production-ready with proper security and monitoring in place.

---

**Next Steps:**
- Review specific modules as needed
- Address missing CODE_STYLE.md
- Expand test coverage
- Consider architectural improvements based on team priorities

