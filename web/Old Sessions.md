# Old Sessions — Web Exploitation Writeup

**Category:** Web Exploitation
**Difficulty:** Easy
**Event:** picoCTF 2026 (challenge by David Gaviria)
**Author:** Ooi

## Scenario

A small web app with a login/register flow and a public comments board. The premise is a real-world session-management failure: if a user logs in on a shared machine and just closes the tab (instead of logging out), a misconfigured session that never expires stays authenticated forever — letting a later attacker on the same app resume that session without credentials.

The target exposes an undocumented `/sessions` endpoint that dumps the server's live session store, which turns "sessions never expire" from a theoretical weakness into a direct account-takeover primitive.

**Target:** `http://xebec.cylabacademy.net:40811/`
**Tools used:** browser, browser DevTools (Application → Cookies)

---

## Old Sessions — Web Exploitation (Easy)

**Objective:** Gain access to the `admin` account and read the flag, without knowing admin's password.

**Analysis:**

The app lets anyone register, so the first step was to create an account (`abc`) and log in to see what an authenticated user looks like.

<img width="633" height="366" alt="image" src="https://github.com/user-attachments/assets/56a52369-b051-4759-a6d4-dbc3c2f937f0" />

Once logged in, the homepage greets the current user and shows a comments board. One comment stood out — `mary_jones_8992` points directly at an undocumented page:

<img width="645" height="582" alt="image" src="https://github.com/user-attachments/assets/cb091820-d9ac-478f-8bcd-dfc83f1e4263" />

> `mary_jones_8992`: *"Hey I found a strange page at /sessions"*

In a CTF this kind of in-band hint is almost always the intended path, so I browsed to `/sessions` directly. The endpoint leaks the entire server-side session store — every active session token alongside the account it belongs to:

<img width="734" height="193" alt="image" src="https://github.com/user-attachments/assets/6a247021-2138-4246-ac53-528feb805e7b" />

```
1) session:QJao-CkUK7KctX-mivL6aK3p7Hhjo9Ablf7XnDKdqJo, {'_permanent': True, 'key': 'admin'}
2) session:W-J7FKWnzePUSXl94tdM3zm9h05kqkTofmvPU_QAQV0, {'_permanent': True, 'key': 'abc'}
```

Two things matter here. The second entry (`key: 'abc'`) is my own session, which confirms how to read the dump: the token maps to the account via the `key` field. The first entry (`key: 'admin'`) is the admin's session token — and critically, both carry `'_permanent': True`.

`_permanent: True` is a Flask-style session flag: a permanent session is governed by a configured lifetime (`PERMANENT_SESSION_LIFETIME`) rather than being a short-lived cookie that dies when the browser closes. When that lifetime is left effectively unbounded — the "misconfigured expiration" the challenge describes — the session stays valid on the server indefinitely. So admin's leaked token isn't a stale artifact; it's a *live* credential.

That gives a clean two-flaw chain:
1. **Information disclosure** — `/sessions` exposes session identifiers that should never be readable by another user.
2. **No session expiration** — the leaked admin session is permanent and still authenticated.

The session is tracked by a cookie literally named `session`, so hijacking it is just a matter of replacing my own cookie value with admin's. In DevTools → Application → Cookies, I edited the `session` cookie for the site and pasted in admin's token (`QJao-CkUK7KctX-mivL6aK3p7Hhjo9Ablf7XnDKdqJo`):

<img width="737" height="617" alt="image" src="https://github.com/user-attachments/assets/4891b634-1222-4a96-9d30-dd8528e8def3" />

After refreshing, the server read the swapped cookie, matched it to the still-valid admin session, and rendered the homepage as `admin` — with the flag printed at the top:

<img width="665" height="655" alt="image" src="https://github.com/user-attachments/assets/6b58069c-e4ac-486d-90cc-5a91968c698d" />

The flag text itself spells out the lesson: *set session expirations.*

**Finding:** The `/sessions` endpoint leaks every active session token, and sessions are configured as permanent with no effective timeout. Copying the admin session token into my own `session` cookie resumed admin's authenticated session and exposed the flag — a full account takeover with no credentials.

**Answer:** `academy{s3t_s3ss10n_3xp1rat10n5_e3a46efc}`

---

## Summary

| Challenge | Category | Technique | Flag |
|---|---|---|---|
| Old Sessions | Web Exploitation (Easy) | Session hijack via `/sessions` info disclosure + non-expiring permanent sessions | `academy{s3t_s3ss10n_3xp1rat10n5_e3a46efc}` |

## Takeaways

- **Session cookies are credentials.** Anything that leaks a valid session identifier is equivalent to leaking a password. Maps to MITRE ATT&CK **T1539 (Steal Web Session Cookie)** and **T1550.004 (Use Alternate Authentication Material: Web Session Cookie)** — swapping in a stolen cookie lets you authenticate as the victim with no password at all.
- **Two mild bugs chain into a critical one.** An info-disclosure endpoint alone is bad; permanent sessions alone are bad; together they're instant admin takeover. CTFs (and real breaches) reward spotting the chain, not just the individual bug.
- **`_permanent: True` + unbounded lifetime = never-expiring sessions.** Correct hardening is a short, enforced `PERMANENT_SESSION_LIFETIME`, server-side invalidation on logout, and rotating/expiring tokens — the real-world fix behind the flag's hint.
- **Trust the in-band hints.** The planted comment pointing at `/sessions` was the intended lead. Enumerating comments/notices on a target is cheap and often points straight at hidden endpoints — the same instinct as checking `robots.txt`, comments in page source, or verbose error pages.
- **Never expose a debug/admin endpoint like `/sessions` in production.** A page that dumps the session store is the web equivalent of leaving the keyring on the front desk; such endpoints should be removed or locked behind strict authz, never reachable by an ordinary user.
