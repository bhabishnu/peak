# Peak — frontend

FitConnect / Peak campus fitness app. React + Vite + TypeScript, MapLibre GL, Framer Motion, Recharts, Supabase.
Scope is frozen by `docs/contract.md` (local, not committed).

## Run

```bash
npm install
npm run dev          # http://localhost:5173 — mock backend by default
npm run dev:phone    # HTTPS on your LAN, for DeviceMotion / GPS / camera on a real phone
```

Phones only expose motion, location and camera in a secure context. Open the `https://<lan-ip>:5173`
URL from `dev:phone`, accept the self-signed certificate, then tap Start (iOS asks for motion access on that tap).

## Modes

| `.env.local` | Behaviour |
|---|---|
| nothing / `VITE_USE_MOCKS=true` | In-memory mock backend with simulated realtime. State persists in localStorage and syncs **between tabs of one browser** (student tab + organiser tab). |
| `VITE_USE_MOCKS=false` + `VITE_SUPABASE_URL` + `VITE_SUPABASE_ANON_KEY` | Real backend via `src/api/supabase/supabaseApi.ts`. Cross-device realtime. |
| `VITE_DEMO_EMAIL` / `VITE_DEMO_PASSWORD` | Enables "Continue as demo student" in Supabase mode. Keep these in `.env.local` only. |

See `.env.example`.

## Architecture

```
src/api/types.ts                 frontend domain types (the UI's contract)
src/api/client.ts                Api interface — the only thing screens call
src/api/mock/                    mock backend (rules marked MOCK RULE stand in for backend logic)
src/api/supabase/                translation layer: backend rows/RPCs → domain types
  database.types.ts              copied verbatim from the backend's generated types
src/api/useLive.ts               fetch + refetch on realtime topics
src/sensors/                     step detector, GPS sampling + route filter, wake lock, simulator
src/tracking/sessionStore.ts     live session: start → push every 5 s → finish/discard
src/components/summit/           weekly 53,000-step Everest journey
src/screens/                     Home · Track · Play · Compete · You (+ hidden organiser route)
```

- The backend owns the schema; the frontend adapts in `supabaseApi.ts`.
- The UI never computes MovePoints, ranks, streaks or eligibility. It renders what the API returns.
- Navigation is frozen at five tabs. `/organiser/event/:eventId` is a hidden deep link.
  
## Demo Video

https://github.com/user-attachments/assets/c21195e9-57bc-4ed9-8f7a-69fc8d176432

## Demo script

**Mock (one laptop or one phone, two tabs):**
1. Track → walk (or "Run a simulated session" indoors). Home/Compete in another tab climb live.
2. You → Demo → open an organiser link in a new tab.
3. Play → Scan event QR → scan the organiser tab's QR (or "Enter code instead", or open its URL).
4. Organiser tab → Confirm → the student tab flips to **Verified**, with points and badge toasts.
5. You → Demo → Reset demo data before presenting.

**Supabase (phone + laptop):** follow the backend handoff (re-seed shortly before). Phone signs in
as the demo student, laptop opens `/organiser/event/<event-id>` signed in as the organiser. Scan with
the in-app scanner: the backend QR payload is `peak:checkin:CODE`, which phone camera apps won't open as a link.
Two events are live for ~6 h after a re-seed, so one can back up the other (organiser `organiser@peak.test`):
Futsal `/organiser/event/00000000-0000-4000-c000-000000000001` (`FUTSAL26`) and
Pickleball `/organiser/event/00000000-0000-4000-c000-000000000003` (`PICKLE26`).

## Known limitations

- Browsers pause DeviceMotion and Geolocation when the screen locks. Tracking holds a Screen Wake Lock
  and tells the user to keep the screen on. There is no native/background workaround.
- A session in progress is not resumed after a page reload. The backend auto-completes a lingering live
  activity on the next start.
- Supabase mode: Home's feed shows your own ledger, because the backend has no campus-wide feed.
- The map uses OpenFreeMap's free tiles and needs network. If tiles fail, the route still records.
- `src/config.ts` has a placeholder campus centre (map fallback + simulator loop). Set the real one.
