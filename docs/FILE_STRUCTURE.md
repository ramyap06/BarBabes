# BarBabes — File Structure & Architecture

This document explains how the repo is laid out and how the pieces fit together, so you can rebuild, extend, or onboard onto the project quickly.

> **Note:** this repo was recently cleaned up — duplicate/dead code (four separate Widmark BAC implementations, two separate Gemini integrations, an unused CRUD module), orphaned frontend screens, a stray root npm manifest, and a broken group-creation navigation path were removed or fixed. See "Cleanup log" at the bottom for exactly what changed.

## 1. High-level architecture

BarBabes is a **two-tier app**: a React Native (Expo) mobile client and a Python FastAPI backend, backed by MongoDB (live data) and, optionally, Snowflake (long-term archive) and the Google Gemini API (AI sobriety assessment).

```
┌─────────────────────┐        HTTP/JSON        ┌──────────────────────┐
│  frontend/ (Expo)    │ ───────────────────────▶│  backend/ (FastAPI)  │
│  React Native app    │◀─────────────────────── │  app/                │
└─────────────────────┘                          └──────────┬───────────┘
                                                              │
                                        ┌─────────────────────┼─────────────────────┐
                                        ▼                     ▼                     ▼
                                   MongoDB (Motor)      Gemini API (httpx)    Snowflake (archive,
                                   users/drinks/groups   sobriety scoring      via scripts/)
```

- The frontend never talks to MongoDB, Gemini, or Snowflake directly — everything goes through the FastAPI backend.
- `EXPO_PUBLIC_API_URL` (frontend `.env`) points the app at the backend's base URL (defaults to `http://localhost:8000`).
- `MONGODB_URI` / `GOOGLE_GEMINI_API_KEY` / `SNOWFLAKE_*` (backend `.env`) configure the backend's outbound connections.

## 2. Top-level layout

```
BarBabes/
├── backend/          FastAPI service (Python)
├── frontend/          Expo / React Native app (JavaScript)
├── scripts/            Standalone ops scripts (Snowflake archival)
├── docs/                Project documentation (this file lives here)
└── .gitignore
```

### `backend/`

```
backend/
├── app/
│   ├── main.py              FastAPI app factory: lifespan (Mongo connect + index creation +
│   │                        shared httpx client), CORS middleware, router registration,
│   │                        root "/" and "/health" endpoints.
│   ├── config.py            pydantic-settings Settings loaded from backend/.env
│   │                        (mongodb_uri, google_gemini_api_key, snowflake_*).
│   ├── db.py                Motor (async MongoDB) client factory + get_db() FastAPI dependency
│   │                        that yields request.app.state.db (the "saferound" database).
│   ├── models.py            All Pydantic request/response schemas: users, BAC, sobriety,
│   │                        groups, drink validation.
│   ├── routers/
│   │   ├── users.py           POST /users, GET /users/{id}, PUT/PATCH /users/{id},
│   │   │                     PATCH /users/{id}/cut-off — user CRUD against db.users.
│   │   ├── bac.py               POST /bac/estimate — Widmark BAC calculation. This is now the
│   │   │                       single source of truth for the formula (see Cleanup log — three
│   │   │                       other copies were removed).
│   │   ├── sobriety.py          POST /sobriety/assess, POST /sobriety/recommendation —
│   │   │                       delegates to services/gemini_sobriety.py.
│   │   ├── groups.py             GET /groups/list, POST /groups, POST /groups/join,
│   │   │                       GET /groups/{id}/members, POST /groups/notify — 6-digit
│   │   │                       group codes, membership, mock push notification. (Note:
│   │   │                       UserContext.js calls POST /groups/leave, which has no
│   │   │                       matching route here — see Known Issues.)
│   │   └── drinks.py             POST /validate-drink — cooldown + cut-off gated drink
│   │                            logging via services/drink_validation.py. Not included in
│   │                            main.py's router list, so this endpoint is currently dead
│   │                            (unreachable) despite being fully implemented — left as-is
│   │                            (registering it is a functional/product decision, not a
│   │                            structural cleanup one; see Known Issues).
│   └── services/
│       ├── drink_validation.py  Drink cooldown/cut-off business logic (used by routers/drinks.py).
│       ├── gemini_sobriety.py    Calls the Gemini API (tries gemini-2.5-flash → 2.0-flash →
│       │                        2.0-flash-001 in order), builds prompts from telemetry/BAC,
│       │                        parses JSON out of the model response, and provides safe
│       │                        BAC-only fallbacks if Gemini is unavailable/unconfigured. This
│       │                        is now the single Gemini integration (see Cleanup log).
│       └── indexes.py            create_indexes() — unique index on users.user_id, compound
│                                index on drinks (user_id, timestamp), unique index on
│                                groups.code. Run once at startup via main.py's lifespan.
├── seed_data.py                One-off script: inserts/upserts a demo user directly via
│                             pymongo (synchronous), for manual local testing.
├── requirements.txt            pip dependencies (fastapi, uvicorn, motor, pydantic,
│                             pydantic-settings, python-dotenv, httpx, snowflake-connector-python).
├── pyproject.toml              Hatchling project metadata, name "saferound-backend" — mirrors
│                             requirements.txt (package name still uses the app's old
│                             "SafeRound" working title).
└── .env.example                 Template for backend/.env.
```

**Request flow example (BAC estimate):**
`Dashboard.jsx` → `POST {API_BASE}/bac/estimate` → `routers/bac.py:estimate_bac()` → `widmark_bac()` → returns `{bac, status, notify_guardian}` → frontend renders `BACRing` and calls `/sobriety/recommendation` for a plain-language tip.

### `frontend/`

Built with **Expo Router** (file-based routing) on React Native 0.81 / React 19.

```
frontend/
├── app/                          Expo Router root — every file here becomes a route.
│   ├── _layout.jsx                Root layout: wraps the whole app in <UserProvider> and a
│   │                            <Stack> navigator (slide-from-bottom transitions).
│   ├── index.jsx                  Route "/" → re-exports screens/User/Login.jsx (the app's
│   │                            actual landing page).
│   ├── profile.jsx                 Route "/profile" → screens/User/Profile.jsx (the real
│   │                            onboarding/profile form).
│   ├── MainLayout.jsx               Shared chrome (NOT a route): background "blob" decoration,
│   │                              SafeAreaView, TopNavBar, and the hamburger menu
│   │                              (Profile / Log Out / Leave Group). Most screens wrap
│   │                              their content in <MainLayout>.
│   ├── context/
│   │   └── UserContext.js           Global app state via React Context: profile fields
│   │                              (name, age, weight, gender, height, tolerance, contacts),
│   │                              drinkLog + calculateBAC() (client-side Widmark calc used
│   │                              for optimistic UI before the backend responds), and all
│   │                              group networking (createGroup, joinGroup, joinGroupById,
│   │                              refreshGroupMembers, leaveGroup — this last one POSTs to
│   │                              /groups/leave, which the backend does not currently expose).
│   ├── components/
│   │   ├── TopNavBar.jsx             Fixed header: avatar, first name, group name, menu button.
│   │   ├── BottomNavBar.jsx           Fixed footer: Home / Map / Contacts / Reaction-test icons,
│   │   │                            using usePathname() to avoid redundant navigation.
│   │   └── ToastStyles.js              Shared toast StyleSheet (duplicated inline in a couple
│   │                                of screens instead of being imported everywhere).
│   ├── dashboard/                       Thin Expo Router route wrappers (route → screen):
│   │   ├── index.jsx                     "/dashboard" → screens/Dashboard/Dashboard.jsx
│   │   └── group-map.jsx                  "/dashboard/group-map" → screens/Dashboard/GroupMapScreen.jsx
│   ├── group/                             More thin route wrappers, now consistent with the
│   │   │                                rest of the route files (route → real screen):
│   │   ├── index.jsx                       "/group" → screens/Group/GroupScreen.jsx
│   │   ├── create.jsx                       "/group/create" → screens/Group/CreateGroup.jsx.
│   │   │                                  (Previously a disconnected placeholder with a dead
│   │   │                                  link — see Cleanup log. GroupScreen.jsx's "Create
│   │   │                                  Group" button pushes here, so this now correctly
│   │   │                                  reaches the real create-group form.)
│   │   └── join.jsx                          "/group/join" → screens/Group/JoinGroup.jsx.
│   │                                       (Same fix as create.jsx — GroupScreen.jsx's "Join
│   │                                       Group" button now reaches the real join-group UI
│   │                                       instead of a dead-end placeholder.)
│   ├── screens/                            The actual screen implementations, organized by
│   │   │                                 feature. Reached either via the short "/group/..."
│   │   │                                 aliases above (from GroupScreen.jsx) or via direct
│   │   │                                 "/screens/..." paths (from HomeScreen.jsx) — both
│   │   │                                 now resolve to the same real components.
│   │   ├── Home/
│   │   │   └── HomeScreen.jsx                Post-login landing screen: "Hi, {firstName}" +
│   │   │                                    Create Group / Join Group buttons (links directly
│   │   │                                    to screens/Group/CreateGroup.jsx and JoinGroup.jsx).
│   │   ├── User/
│   │   │   ├── Login.jsx                      Email/password form (client-side only — no real
│   │   │   │                                auth check); "REGISTER" button posts a hardcoded
│   │   │   │                                demo user to POST /users and jumps to GroupMapScreen.
│   │   │   ├── SignIn.jsx                      Similar sign-up form; checks GET /users/{email}
│   │   │   │                                to decide whether to route to Login or SignIn.
│   │   │   └── Profile.jsx                      Full onboarding/profile form: name, age, height,
│   │   │                                      weight, gender, phone, tolerance slider, photo
│   │   │                                      picker. Submits POST /users then PATCH
│   │   │                                      /users/{id} (tolerance), populates UserContext,
│   │   │                                      and routes to HomeScreen.
│   │   ├── Dashboard/
│   │   │   ├── Dashboard.jsx                     Core screen: BAC ring, standard-drink stepper,
│   │   │   │                                   NFC tag scanning (react-native-nfc-manager,
│   │   │   │                                   gracefully no-ops if unavailable e.g. in Expo
│   │   │   │                                   Go), calls /bac/estimate + /sobriety/recommendation,
│   │   │   │                                   "Notify Group" (/groups/notify) and "Go Home"
│   │   │   │                                   (deep-links to Uber, falls back to m.uber.com).
│   │   │   ├── BACRing.jsx                        SVG circular gauge; color-interpolates
│   │   │   │                                   blue→green→yellow→orange→red across the BAC range.
│   │   │   ├── ContextList.jsx                     Group member contact list (maps groupMembers
│   │   │   │                                   from context into rows with chat/call icons —
│   │   │   │                                   icons are not wired to real actions yet).
│   │   │   └── GroupMapScreen.jsx                    Static background image (Harry's Chocolate
│   │   │                                          Shop) with member avatars pinned at fixed
│   │   │                                          percentage coordinates (not a real map/GPS),
│   │   │                                          plus a draggable bottom sheet listing members.
│   │   ├── Group/
│   │   │   ├── GroupScreen.jsx                       "Your Group" hub: create/join buttons when
│   │   │   │                                       ungrouped, else group code + member list +
│   │   │   │                                       link to the map. Polls refreshGroupMembers
│   │   │   │                                       every 15s.
│   │   │   ├── CreateGroup.jsx                        Real create-group form (name → POST
│   │   │   │                                       /groups → auto-join → Dashboard).
│   │   │   ├── JoinGroup.jsx                            Real join-group UI: lists groups from
│   │   │   │                                       GET /groups/list, search/filter, join by
│   │   │   │                                       code or id.
│   │   │   └── index.jsx                                  Dummy default export, present only to
│   │   │                                                stop an Expo Router warning about a
│   │   │                                                missing default export in this folder.
│   │   └── SobrietyTests/
│   │       └── ReactionTest.jsx                             5-round reaction-time mini-game;
│   │                                                       stores latencies in UserContext and
│   │                                                       POSTs telemetry + current BAC to
│   │                                                       /sobriety/assess.
│   └── assets/                                                Logos, splash art, silhouette
│                                                              placeholder avatar, Harry's map
│                                                              background image.
├── assets/                                                      App-level icons (adaptive-icon,
│                                                                favicon, splash) referenced from
│                                                                app.json.
├── app.json                                                       Expo config: app name/slug
│                                                                "barbabes", NFC permission
│                                                                (declared twice — see Known
│                                                                Issues), Android package
│                                                                com.azeemme.barbabes, plugins
│                                                                (expo-router, expo-web-browser,
│                                                                react-native-nfc-manager).
├── package.json                                                     Expo/React Native
│                                                                   dependencies + scripts
│                                                                   (start/android/ios/web/prebuild).
└── .env.example                                                       Template for frontend/.env
                                                                      (EXPO_PUBLIC_API_URL).
```

### `scripts/`

```
scripts/
└── archive_session.py    Standalone cron-style job: pulls the last 7 days of "completed"
                          drink records out of MongoDB and archives them into a Snowflake
                          table (archive_drinks) for longer-term research/analytics. Reads
                          credentials from backend/.env via python-dotenv. Not invoked by
                          the backend itself — intended to run on an external scheduler.
```

## 3. Data model (MongoDB, database `saferound`)

| Collection | Key fields | Notes |
|---|---|---|
| `users` | `user_id` (unique), `first_name`, `last_name`, `age`, `weight_kg` (or `weight_lb` in `UserProfile`), `sex`, `primary_contact`, `is_cut_off`, `height_cm`, `tolerance` | `user_id` is treated as an arbitrary string (in practice the user's email). Field naming is inconsistent between models (`weight_kg` vs `weight_lb`) — see Known Issues. |
| `drinks` | `user_id`, `drink_id`, `alcohol_grams`, `timestamp` | Indexed on `(user_id, timestamp desc)` for cooldown/history lookups. |
| `groups` | `code` (unique 6-digit), `name`, `member_ids: [user_id]` | No `leave` support at the data-access layer yet. |

## 4. Configuration & environment

| File | Purpose |
|---|---|
| `backend/.env` (from `.env.example`) | `MONGODB_URI`, `GOOGLE_GEMINI_API_KEY`, `SNOWFLAKE_*` |
| `frontend/.env` (from `.env.example`) | `EXPO_PUBLIC_API_URL` — must be reachable from the physical device/simulator (e.g. an ngrok tunnel or LAN IP, not `localhost`, when testing on a phone) |

CORS in `backend/app/main.py` currently allow-lists a fixed set of origins (`localhost:3000`, `localhost:19006`, `127.0.0.1:19006`, and one hardcoded LAN IP `10.186.38.91:19006`) — this list needs to be updated per-developer machine or replaced with an env-driven config (see Known Issues / INITIAL_REPORT).

## 5. Running the app locally (as currently wired)

1. **Backend**: `cd backend`, create a venv, `pip install -r requirements.txt`, copy `.env.example` → `.env` and fill in `MONGODB_URI` (+ optionally `GOOGLE_GEMINI_API_KEY`), then `uvicorn app.main:app --reload`.
2. **Frontend**: `cd frontend`, `npm install`, copy `.env.example` → `.env` and set `EXPO_PUBLIC_API_URL` to the backend's reachable URL, then `npm start` (or `npm run android` / `npm run ios` for a native/NFC-capable build — NFC does not work in Expo Go).
3. NFC drink-scanning and the reaction-time/sobriety telemetry both require a physical device with a development build (`expo prebuild` + `expo run:android`/`run:ios`), not the Expo Go sandbox.

## 6. Known structural issues still worth knowing (not addressed by this cleanup)

These are functional/product gaps rather than tidiness issues, so they were left alone rather than changed silently — see `INITIAL_REPORT.md` for the fuller risk/impact discussion.

- **Dead/unreachable endpoint**: `routers/drinks.py` (`POST /validate-drink`) is fully implemented but never registered in `main.py`, so drink-cooldown/cut-off enforcement via that path is inactive. Registering it would change live app behavior (a new working endpoint), so it was left as-is pending a product decision.
- **Missing endpoint the frontend expects**: `UserContext.js`'s `leaveGroup()` calls `POST /groups/leave`, which does not exist in `routers/groups.py`.
- **Duplicate NFC permission entry** in `frontend/app.json`'s Android `permissions` array (`android.permission.NFC` listed twice) — harmless but worth a one-line fix.
- **Root project naming is inconsistent across three working titles**: "BarBabes" (README, app.json), "SafeRound" (`backend/pyproject.toml`, the Mongo database name `saferound`), and "SafetyNet" (was the removed root `package-lock.json`'s package name). Worth picking one and updating consistently.
- **Two unrelated onboarding forms** (`screens/User/Login.jsx` and `screens/User/SignIn.jsx`) with overlapping-but-diverging logic for the same "create an account" step.

## 7. Cleanup log (this pass)

**Removed — confirmed dead/duplicate code (verified via repo-wide grep for imports/usages before deletion):**

- `backend/app/logic.py` — third copy of the Widmark formula plus a second `validate_drink()`, unused; superseded by `routers/bac.py` and `services/drink_validation.py`.
- `backend/calc.py` — a fourth-ish copy of the Widmark formula, unused.
- `backend/app/widmark.py` — yet another standalone Widmark implementation (with its own derivation docstring), unused.
- `backend/api_int.py` — an earlier draft Gemini integration, unused; superseded by `services/gemini_sobriety.py`.
- `backend/app/crud.py` — direct MongoDB helpers unused by any router (routers query `db.users`/`db.drinks` inline instead).
- `frontend/app/data/contacts.js` — static mock contact/BAC fixture, not imported anywhere.
- `frontend/app/screens/Profile/index.jsx` — an orphaned bare "Profile Page" placeholder, unrelated to and unreferenced by the real `/profile` route (`screens/User/Profile.jsx`).
- Root `package.json` / `package-lock.json` — an orphaned npm manifest (lockfile's internal name was `"SafetyNet"`, yet another working title) with no scripts, no workspaces config, and no relation to `frontend/package.json`; not used by any build step.
- Root `commands` file — a stray one-line terminal-history artifact, unrelated to the app (removed prior to this pass).

**Fixed — real navigation bug, not just clutter:**

- `frontend/app/group/create.jsx` and `frontend/app/group/join.jsx` were previously disconnected placeholder screens (their own "Create/Join Group Screen" text + a button to nowhere useful), even though `screens/Group/GroupScreen.jsx` actually navigates to `/group/create` and `/group/join` as its primary "Create Group"/"Join Group" actions. They are now thin route wrappers — matching the pattern already used by `dashboard/index.jsx` and `group/index.jsx` — that render the real `screens/Group/CreateGroup.jsx` and `screens/Group/JoinGroup.jsx` components. `GroupScreen.jsx`'s buttons now reach working forms instead of a dead end.

Everything above was verified unreferenced via `grep` across the full repo before removal; nothing in `backend/app/main.py`'s router registration, `services/`, or any frontend screen imported the removed files.
