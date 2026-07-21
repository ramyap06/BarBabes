# BarBabes — Initial Deployment Readiness Report

**Scope**: assessment of the codebase as it exists today (hackathon-stage prototype: Expo/React Native frontend + FastAPI/MongoDB backend + Gemini sobriety scoring), evaluated against "what happens if we actually put this in front of real users at a bar tonight." See `docs/FILE_STRUCTURE.md` for the full file-by-file map this report is based on.

**Bottom line**: this is a strong, coherent hackathon prototype that proves the concept end-to-end (signup → BAC tracking → group accountability → sobriety check → "go home" flow). It is **not** deployment-ready for real users, primarily because several of the actual *safety* mechanics the app is pitched on (emergency notification, drink cut-off enforcement, session security, location accuracy) are currently mocked, unreachable, or client-side-only. For an app whose core value proposition is safety for a vulnerable population, that gap matters more than it would for a typical consumer app.

---

## The Good: what's working well

1. **The full user journey actually exists and holds together.** Sign up → profile setup → home → create/join group → dashboard (BAC ring, drink logging) → group map → reaction-time sobriety test → notify group / go home. For a hackathon build, having every screen in the pitch wired to *something* real (not just mockups) is a genuine accomplishment.

2. **Clean backend separation of concerns.** `routers/` (HTTP layer), `services/` (business logic), `models.py` (schemas) is a sensible structure to build on. The FastAPI `lifespan` pattern correctly manages a shared Motor client and a shared `httpx.AsyncClient`, and indexes are created idempotently on startup (`services/indexes.py`).

3. **Graceful AI degradation.** `services/gemini_sobriety.py` tries three Gemini model variants in sequence and falls back to a deterministic, BAC-threshold-based recommendation if the API key is missing or every call fails. That's the right instinct for a safety feature — never let an AI outage mean "no guidance at all."

4. **Reasonable choice of BAC model.** The Widmark formula is the standard, well-understood approach for this kind of estimation, and the tolerance slider gives an obvious hook for personalizing it further.

5. **Secrets are externalized correctly.** `.env.example` files exist for both frontend and backend, real `.env` files are gitignored, and nothing looks hardcoded into source control as an API key or credential.

6. **Visual polish is well above typical hackathon bar.** The BAC ring's color interpolation, the draggable bottom sheet on the group map, and consistent theming show real UX investment, which matters for an app meant to be usable by an impaired user in a noisy bar.

---

## The Bad: biggest risks before this touches real users

Ranked roughly by severity for a live deployment.

### 1. No real authentication — this is the single biggest blocker

`screens/User/Login.jsx`'s "REGISTER" button doesn't send the email/password the user typed at all — it posts a **hardcoded** demo user (`First Last`, age 21, weight 70kg, phone `555-555-5555`) to `POST /users` and proceeds. There is no password hashing anywhere in the codebase, no login/session/JWT endpoint on the backend, and no auth middleware on any route. Combined with the next point, this means:

- Anyone who knows or guesses a `user_id` (which is just an email address) can read or overwrite that person's profile via `GET/PUT/PATCH /users/{user_id}` — no credential required.
- For an app whose core pitch is protecting women from real physical risk, an attacker being able to look up a stranger's name, age, weight, and **emergency contact phone number** by guessing their email is a serious safety and privacy failure, not just a bug.

### 2. The actual safety/emergency mechanics are mocked or unreachable

- `POST /groups/notify` ("Notify Group") doesn't send anything — it computes a message string and returns a count, with a comment saying real push notifications would be "plugged in later."
- `POST /validate-drink`, which enforces the cooldown and the "cut off" hard-stop on ordering more drinks, is fully implemented in `routers/drinks.py` but is **never registered** in `main.py` — it's dead code. There is also no UI anywhere that calls `PATCH /users/{id}/cut-off` to engage it in the first place.
- The "map" in `GroupMapScreen.jsx` is a static image of one specific bar with member avatars pinned at hardcoded percentage coordinates — it is not real GPS location. If a user in a genuine emergency situation believes their friends can see where they physically are, that's a dangerous false affordance.

In short: the demo *looks* like it has group alerting, drink cut-offs, and live location — a user in a real crisis would find that none of the three actually functions.

### 3. ~~Duplicate, unreachable, or dead logic~~ — resolved in the 2026-07-21 cleanup

Previously, four separate Widmark BAC implementations (`routers/bac.py`, `app/logic.py`, `calc.py`, `app/widmark.py`) and two separate Gemini integrations (`services/gemini_sobriety.py`, `api_int.py`) coexisted, with only one of each actually wired up — a correctness risk, since a future BAC bugfix could easily miss the other silently-diverging copies. The unused copies (plus an unused `crud.py` and a couple of orphaned frontend screens/files) have since been deleted; see `docs/FILE_STRUCTURE.md`'s "Cleanup log" for the full list. `routers/bac.py` and `services/gemini_sobriety.py` are now each the single source of truth. The underlying risk pattern (untested numeric logic, no regression coverage) is still real — see point 4 — but the duplication itself is gone.

### 4. No automated tests anywhere in the repo

There is zero test coverage for the BAC math, the drink cooldown/cut-off logic, or the group join/leave flows. This is the calculation an intoxicated user is meant to trust to decide "should I call a ride." It should not ship, or change, without regression tests.

### 5. Group codes are a weak, guessable access boundary

Groups are joined via a 6-digit numeric code with no rate limiting on `POST /groups/join` and no expiry. That's one million combinations, brute-forceable in minutes from a single client, letting an outsider join any group and see its members' names, phone numbers, and BAC levels.

### 6. Hardcoded developer environment leakage

`main.py`'s CORS allow-list includes a literal hardcoded LAN IP (`10.186.38.91:19006`) — presumably a team member's personal dev machine, checked into source control. Beyond being dead weight, this is a small but real pattern smell: it suggests config that should be environment-driven is currently being hand-edited and committed instead, which won't scale past one developer's laptop, let alone a real deployment.

### 7. Core hardware dependency (NFC coasters) doesn't exist yet

The pitch's headline feature — NFC-tagged coasters at partner bars — has no bar partnerships or physical hardware behind it. The app degrades reasonably (a manual "tap to log a drink" button works fine, and NFC code is defensively feature-detected so it no-ops rather than crashing when unavailable), but this means the differentiated, hardware-based safety mechanism is currently just a manual counter — closer to a generic drink-tracking app than the pitched product until that partnership pipeline exists.

### 8. No visible data privacy/compliance posture

The app collects age, weight, phone number, and behavior/reaction-time data that could be considered sensitive for a vulnerable population (the pitch explicitly mentions assault survivors), with no privacy policy, consent screen, encryption-at-rest strategy, or data retention/deletion story visible anywhere in the repo. Age is validated as 18+ only at the schema level (`Field(..., ge=18)`), with no server-side identity verification — a strictly cosmetic guardrail.

### 9. Failure handling tends toward silence, not safety

Across the frontend, network/NFC errors are frequently swallowed in empty `catch (_) {}` blocks rather than surfaced or retried. Given the team's own retro calls out unreliable bar connectivity as a top challenge, an app that fails silently exactly when the network is flaky is failing in its most likely real-world failure mode — a user might believe a drink was logged, a group was notified, or an assessment ran, when none of that happened.

---

## Suggested priority order if moving toward a real pilot

1. Real authentication (hashed passwords or an identity provider) + authorization on every user/group endpoint.
2. Wire `POST /groups/notify` to an actual push/SMS provider; wire the cut-off flow end-to-end (UI toggle → `PATCH /cut-off` → enforced in a *registered* `/validate-drink` route).
3. Add rate limiting / expiry to group codes, or move to non-guessable invite tokens.
4. Consolidate the triplicated BAC logic and duplicated Gemini integration into one source of truth each, and add unit tests around the BAC math specifically.
5. Replace the static map image with real (opt-in, consented) location sharing, or clearly relabel it as a stylized/demo view so users don't overtrust it.
6. Add a privacy policy, consent flow, and a data retention/deletion story before collecting any real users' health-adjacent data.
