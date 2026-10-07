# Crack the Gate 1 — Web Exploitation Writeup

**Category:** Web Exploitation
**Difficulty:** Easy
**Event:** picoMini by CMU-Africa (challenge by Yahaya Meddy)
**Author:** Ooi

## Scenario

An investigation into a person of interest, *ctf-player*, who is hiding data behind a restricted login portal. We're given his login email — `ctf-player@cylabacademy.org` — but not his password, and password guessing is a dead end. The hint is that "the developer left a secret way in," which points away from credential attacks and toward something left behind in the app itself.

**Target:** `http://chatelaine.cylabacademy.org/` (Express backend, per `X-Powered-By: Express`)
**Known email:** `ctf-player@cylabacademy.org`
**Tools used:** browser, browser View Source, ROT13 decoder (CyberChef), HTTP request editor

---

## Crack the Gate 1 — Web Exploitation (Easy)

**Objective:** Authenticate to the portal as `ctf-player` and retrieve the flag without knowing the password.

**Analysis:**

The obvious first instinct with a known email and unknown password is to probe for SQL injection in the login form. Dropping a classic payload into the email field (`ctf-player@cylabacademy.org'--`) never even reached the server — the browser's built-in HTML5 `type="email"` validation rejected it client-side:

<img width="575" height="441" alt="Screenshot 2026-10-08 004502" src="https://github.com/user-attachments/assets/0a644132-a3fe-412f-b075-fac4b776d6d5" />

That's worth noting rather than hiding: HTML5 email validation is **not** a security control (it's trivially bypassed by crafting the request directly), but it was enough to signal that the form wasn't the intended path — and the challenge hint was explicitly about a *developer* leaving a way in, not an injection. That reframes the hunt toward the application's own artifacts, so the next move was to read the page source.

The source contained a comment the developer clearly meant to strip before shipping — an obfuscated note plus a telltale reminder:

<img width="621" height="568" alt="Screenshot 2026-10-08 004613" src="https://github.com/user-attachments/assets/baf4dffa-4ef3-4732-af39-e146b7e67561" />

```html
<!-- ABGR: Wnpx - grzcbenel olcnff: hfr urnqre "K-Qri-Npprff: lrf" -->
<!-- Remove before pushing to production! -->
```

The string `ABGR` is a dead giveaway for **ROT13** — a Caesar cipher with a fixed shift of 13, which is its own inverse (applying it twice returns the original). `ABGR` → `NOTE` is the classic tell. Running the whole comment through a ROT13 decoder confirms it:

<img width="570" height="452" alt="Screenshot 2026-10-08 004620" src="https://github.com/user-attachments/assets/bb9495a8-4ea3-481c-89b6-1ca5c6880e54" />

```
NOTE: Jack - temporary bypass: use header "X-Dev-Access: yes"
```

So the developer shipped a **backdoor authentication bypass**: any request carrying the header `X-Dev-Access: yes` is treated as privileged, no password required. ROT13 added only a thin layer of obscurity — security through obscurity, not encryption — so once the comment is read, the bypass is fully exposed.

To use it, I replayed the portal's login request (a JSON `POST`) and added the custom header `X-Dev-Access: yes`:

<img width="328" height="565" alt="Screenshot 2026-10-08 005010" src="https://github.com/user-attachments/assets/0dc8416d-6aa8-4bce-836f-1febce53c7ed" />

The server accepted the bypass and returned `200 OK` with a JSON body containing `success: true` and the flag:

<img width="361" height="378" alt="Screenshot 2026-10-08 005020" src="https://github.com/user-attachments/assets/22f111c6-df8a-47b2-9909-e98c20ff5314" />

```json
{
  "success": true,
  "email": "ctf-player@cylabacademy.org",
  "firstName": "pico",
  "lastName": "player",
  "flag": "academy{brut4_f0rc4_79ebc48a}"
}
```

**Finding:** The developer left a debug authentication-bypass header (`X-Dev-Access: yes`) in production, documented in a ROT13-obfuscated HTML comment that was never removed. Reading the comment, decoding it, and sending the header authenticates as the target with no password and returns the flag.

**Answer:** `academy{brut4_f0rc4_79ebc48a}`

---

## Summary

| Challenge | Category | Technique | Flag |
|---|---|---|---|
| Crack the Gate 1 | Web Exploitation (Easy) | Leaked HTML comment (ROT13) revealing an `X-Dev-Access: yes` auth-bypass header | `academy{brut4_f0rc4_79ebc48a}` |

## Takeaways

- **Read the source — comments leak secrets.** HTML/JS comments are shipped to every visitor. Leaving notes, endpoints, or bypasses in them is **CWE-615 (Information Exposure Through Comments)**; "remove before production" reminders are a reliable flag that something sensitive is nearby.
- **ROT13 is obfuscation, not encryption.** A fixed-13 Caesar shift, self-inverse, instantly reversible. Recognize it on sight (`ABGR` → `NOTE`, scrambled-but-English-shaped text) and decode — it provides zero real protection.
- **Hidden "dev bypass" headers are backdoors.** A magic header that skips authentication is a textbook backdoor pattern (maps to MITRE ATT&CK **T1190 — Exploit Public-Facing Application**, and mirrors real hardcoded-bypass incidents in routers, appliances, and IoT firmware). Debug shortcuts must never reach production, and anything secret must live server-side, not in client-delivered code.
- **Client-side validation is not a security boundary.** The HTML5 email check blocked a payload in the browser, but that logic runs on the attacker's machine — it can always be bypassed by crafting the request directly. Never rely on it to stop injection or malformed input.
- **Let the hint reframe the attack surface.** "The developer left a way in" steered away from brute force / SQLi and toward leaked artifacts. Matching technique to the challenge's framing saves time chasing dead ends.
