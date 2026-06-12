# 28 — Auth: premature logout / couple session lifetime to activity

> Read `conventions.md` first. Likely touches `maichess-auth-service` (token
> lifetimes / refresh) and `maichess-client` (`useTokenRefresh`, logout flow).

## Symptom (item 15)

Users are logged out "too fast" and it "doesn't seem to be coupled to user
interactions" — i.e. logout fires on a timer regardless of whether the user is actively
using the app.

## What's wired today

- `lib/hooks/useTokenRefresh.ts`: a `setInterval` every **10 min** calls
  `POST /api/auth/refresh`; access token is documented as **15-min** expiry. Also
  refreshes on tab `visibilitychange` if hidden > 10 min. Mounted globally in
  `app/layout.tsx` via `TokenRefresher`.
- `proxy.ts` redirects unauthenticated users off protected routes; `app/dashboard/page.tsx`
  redirects a stale token through `GET /api/auth/logout` (which clears the cookie).

## Things to investigate (in order)

1. **Refresh-token lifetime in `maichess-auth-service`.** If the *refresh* token is
   short-lived or single-use without rotation, the 10-min access refresh eventually fails
   and the user is bounced. Check the auth-service token config and whether
   `/auth/refresh` rotates/extends the refresh token. This is the most likely culprit for
   "too fast."
2. **Background-tab throttling.** `setInterval` is throttled/paused in background tabs, so
   a 10-min timer can drift past the 15-min access expiry while the tab is backgrounded.
   The visibility handler helps but only fires on focus. Consider a shorter refresh
   interval relative to expiry (e.g. refresh at half the access-token TTL) and/or refresh
   on `focus` unconditionally.
3. **Refresh actually succeeding?** Confirm `POST /api/auth/refresh` returns 2xx and sets
   a new cookie in the running environment (network tab / server log). A silently failing
   refresh (caught and swallowed in `useTokenRefresh`) presents exactly as "logged out for
   no reason."

## Desired behavior — couple to activity

Make an **actively-used** session never expire from idle-timer mechanics:
- Treat real user interaction (route changes, board moves, key/pointer activity) as
  "active" and ensure a refresh happens within each access-token window while active.
- Only let the session lapse after genuine inactivity (define the idle window, e.g. the
  refresh-token TTL). Decide whether idle logout is even desired; if so, make it explicit
  (an inactivity timer reset on interaction) rather than an artifact of refresh timing.
- Keep the existing stale-token → `/api/auth/logout` fallback for the hard-expiry case.

## Testing

- Auth-service: unit tests for token lifetimes + refresh rotation (100% on new code).
- Client: `npm run build` + `npm run lint`; manual — stay active across the access-token
  window and confirm no logout; background the tab and confirm refresh-on-return.
