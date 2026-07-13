# 🔐 Dev Patrika — Authentication System Roadmap
### Google OAuth + GitHub OAuth + Email OTP (via Brevo)

> **Goal**: Unified auth system supporting three login methods, converging into a single `users` table, issuing JWT-based sessions.

---

## 🎯 Design Decisions (locked in)

| Decision | Choice | Why |
|---|---|---|
| Login methods | Google OAuth, GitHub OAuth, Email OTP | Covers dev + non-dev users, no forced GitHub/Google dependency |
| Password storage | **None** — OTP-only for email method | No hashing/reset flow complexity, better security |
| Email provider | Brevo | 300/day free, good deliverability |
| Session | JWT (short-lived access) + refresh token (httpOnly cookie) | Stateless, scalable, standard pattern |
| OAuth library | Authlib | Same pattern for Google + GitHub, clean FastAPI integration |
| User identity key | `email` (unique) | Prevents duplicate accounts when same person uses multiple login methods |

---

## Phase 0 — Schema & Foundation (0.5 day)

| Task | Detail |
|---|---|
| Create `users` table | `id, email (unique), name, avatar_url, auth_provider, provider_id, is_verified, created_at` |
| Create `otp_codes` table | `email, otp_hash, expires_at, attempts, created_at` |
| Add JWT config | `JWT_SECRET`, `JWT_ALGORITHM`, `ACCESS_TOKEN_EXPIRE_MINUTES`, `REFRESH_TOKEN_EXPIRE_DAYS` in `config.py` |
| Add env vars | `GOOGLE_CLIENT_ID/SECRET`, `GITHUB_CLIENT_ID/SECRET`, `BREVO_API_KEY`, `BREVO_SENDER_EMAIL` |
| Install deps | `authlib`, `python-jose[cryptography]` (or `PyJWT`), `httpx` |

**Risk**: Low. Pure setup, no user-facing behavior yet.

---

## Phase 1 — Email OTP Flow (1–1.5 days)

Do this first — it's fully self-contained, no external OAuth consent screens to configure, and validates your Brevo integration early.

| Task | Detail |
|---|---|
| `email_service.py` | Brevo REST client wrapper — `send_otp_email(to, otp)` |
| `otp_service.py` | `generate_otp()`, `verify_otp()` — hashing, expiry, attempt-limiting logic |
| `POST /api/auth/otp/request` | Accepts `{email}`, rate-limited (1 request/60s per email), triggers Brevo send |
| `POST /api/auth/otp/verify` | Accepts `{email, otp}`, validates, creates-or-fetches user, issues JWT + refresh token |
| Rate limiting | Simple in-memory or Redis-based throttle on `/otp/request` — protects Brevo quota |
| Local dev fallback | `DEBUG=True` → print OTP to console instead of sending real email (saves Brevo quota during dev) |
| Test end-to-end | Request OTP → receive email → verify → confirm JWT issued → confirm user row created |

**Risk**: Low-medium. Mainly around rate-limit abuse and OTP brute-force — both are addressed by `attempts` cap + expiry.

---

## Phase 2 — Google OAuth (1 day)

| Task | Detail |
|---|---|
| Google Cloud Console setup | Create project → OAuth consent screen → credentials → get `client_id`/`client_secret` |
| Register redirect URI | e.g. `https://yourdomain.com/api/auth/google/callback` (+ localhost for dev) |
| `Authlib` OAuth client setup | Register Google as an OAuth provider in `auth_config.py` |
| `GET /api/auth/google/login` | Redirects user to Google consent screen |
| `GET /api/auth/google/callback` | Exchanges auth code → fetches profile (email, name, avatar) → create-or-link user → issue JWT |
| Account linking logic | If email already exists (e.g. from OTP signup) → link this OAuth identity to existing user, don't duplicate |
| Test | Full login round-trip from frontend button click → redirect → callback → logged in state |

**Risk**: Medium — redirect URI mismatches are the most common gotcha (must match exactly, including trailing slashes).

---

## Phase 3 — GitHub OAuth (0.5–1 day)

Faster than Phase 2 since the pattern is now established.

| Task | Detail |
|---|---|
| GitHub Developer Settings | Register OAuth App → get `client_id`/`client_secret` |
| Register callback URL | e.g. `https://yourdomain.com/api/auth/github/callback` |
| Register GitHub as Authlib provider | Reuse same pattern as Google |
| `GET /api/auth/github/login` | Redirects to GitHub consent |
| `GET /api/auth/github/callback` | Exchange code → fetch profile → **note: GitHub email can be private/null**, handle fallback (prompt user to confirm email, or use GitHub's `/user/emails` endpoint with proper scope) |
| Account linking logic | Same dedup-by-email logic as Google |
| Test | Full round-trip |

**Risk**: Medium — GitHub's email visibility settings are the main gotcha (some users hide their email; request `user:email` scope explicitly).

---

## Phase 4 — Unified Session Layer (1 day)

This ties all three methods together.

| Task | Detail |
|---|---|
| `jwt_service.py` | `create_access_token()`, `create_refresh_token()`, `decode_token()` |
| Cookie strategy | Access token → short-lived, sent in response body or memory (frontend holds it); Refresh token → httpOnly, secure, sameSite cookie |
| `GET /api/auth/me` | Reads JWT from header/cookie → returns current user profile |
| `POST /api/auth/refresh` | Accepts refresh token → issues new access token |
| `POST /api/auth/logout` | Clears refresh cookie, optionally blacklist token |
| FastAPI dependency | `get_current_user()` — reusable dependency for protecting routes (chat, personal notes, etc.) |
| Apply to existing routes | Decide which of your 17 existing endpoints need auth (e.g. `/api/ai/chat` should probably be tied to `user_id` instead of just `session_id`) |

**Risk**: Medium — this is where bugs silently leak (e.g. expired token not handled gracefully on frontend). Test token expiry paths explicitly, not just happy path.

---

## Phase 5 — Security Hardening (0.5–1 day)

| Task | Detail |
|---|---|
| CORS tightening | Remember your report noted "allow all origins" — lock this down to your actual frontend domain now that auth exists |
| CSRF consideration | If using cookies for refresh token, add CSRF protection (double-submit cookie or SameSite=Strict where possible) |
| OTP brute-force protection | Confirm `attempts` cap + expiry actually blocks repeated guesses |
| HTTPS-only cookies | `secure=True` on all auth cookies in production |
| Input validation | Email format validation, OTP format (6-digit numeric only) via Pydantic |
| Audit `chat_messages` linkage | Migrate from anonymous `session_id` to `user_id` where appropriate, keep guest mode optional if you want unauthenticated chat to still work |

**Risk**: Low individually, but skipping this phase is the highest long-term risk of the whole roadmap.

---

## Phase 6 — Frontend Integration Touchpoints (parallel, ongoing)

*(Backend-facing checklist — actual frontend work depends on your framework choice)*

| Task | Detail |
|---|---|
| Login buttons | Google / GitHub → simple redirect to `/api/auth/{provider}/login` |
| OTP form | Two-step: email input → OTP input, calling `/otp/request` then `/otp/verify` |
| Token storage | Access token in memory/state (not localStorage — XSS risk), refresh token auto-handled via httpOnly cookie |
| Auth state management | Global context/store checking `/api/auth/me` on app load |
| Protected route handling | Redirect to login if `/me` returns 401 |

---

## 📅 Suggested Timeline

| Phase | Duration | Can start after |
|---|---|---|
| 0. Schema & foundation | 0.5 day | — |
| 1. Email OTP (Brevo) | 1–1.5 days | Phase 0 |
| 2. Google OAuth | 1 day | Phase 0 (parallel to Phase 1 possible) |
| 3. GitHub OAuth | 0.5–1 day | Phase 2 (reuses pattern) |
| 4. Unified session layer | 1 day | Phases 1–3 |
| 5. Security hardening | 0.5–1 day | Phase 4 |
| 6. Frontend integration | ongoing/parallel | Phase 1 onward |

**Total: ~4.5–6.5 days** backend-side for a solo dev.

---

## ⚠️ Key Risks to Watch

1. **Duplicate users on multi-method login** — always dedupe by `email`, not by `provider_id`. Someone signing up via OTP today and logging in via Google tomorrow (same email) should land on the *same* user row.
2. **GitHub private email** — don't assume `email` is always present in the GitHub profile response; request the right scope and handle nulls.
3. **Redirect URI mismatches** — Google/GitHub OAuth apps are strict about exact URI match, including dev vs prod URLs. Keep separate OAuth app registrations for local dev and production.
4. **Brevo quota exhaustion** — 300/day is fine for real usage but easy to burn through in testing. Use the console-print fallback in dev.
5. **Token storage on frontend** — don't put access tokens in `localStorage` (XSS-exploitable). Memory + httpOnly refresh cookie is the safer pattern.
6. **Guest vs authenticated chat** — decide early whether `/api/ai/chat` requires login or stays open with optional auth — this affects your `chat_messages` schema (`session_id` vs `user_id`).

---

> **Recommended starting point**: Phase 0 + Phase 1 (Email OTP via Brevo) first — it's fully in your control, validates the `users` table design, and doesn't depend on external OAuth app approval/setup delays. Then Google and GitHub OAuth in parallel once the pattern is proven.
