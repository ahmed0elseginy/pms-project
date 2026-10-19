# Local test accounts (v2)

Test-only accounts in the local verify database (`pms-phase0-verify-mysql`, port 3307), workspace 2 "Meridian Park Co". Never use them outside localhost.

| Role tab | URL | Email | Password | Membership |
|---|---|---|---|---|
| Manager | http://manager.localhost:3001 | v2.backend.test@pms.local | V2test@12345 | MANAGER |
| Staff | http://staff.localhost:3001 | v2.staff.test@pms.local | V2staff-Test-2026! | STAFF |
| Resident | http://resident.localhost:3001 | v2.booking.resident@pms.local | V2test@12345 | RESIDENT |

Each role uses its own host so the three tabs keep separate sessions (cookies and localStorage are per host). The backend must allow these origins: start it with launch config `pms-backend` in `.claude/launch.json`, which adds them to `com.pms.library.rest.cors-allowed-origins`.

The v1 seed users (`manager@pms.local`, `super@pms.local`, `renter@pms.local`, `ahmed@pms.local`) no longer accept their seeded password in this database.
