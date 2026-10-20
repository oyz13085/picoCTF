# Cookies — Web Exploitation Writeup

**Category:** Web Exploitation (Easy)
**Challenge:** Cookies (picoCTF 2021, by madStacks)
**Author:** Ooi

## Scenario

A "cookie search page" lets you search for your favourite kind of cookie. Each search sets an HTTP cookie tracking which one you picked. The server trusts that cookie value without validation, so tampering with it lets you reach a cookie choice that was never offered in the search box — the one that discloses the flag.

**Tools used:** Firefox DevTools (Storage → Cookies)

---

## Cookie tampering / insecure direct object reference (Easy)

**Objective:** Manipulate the session cookie to reach the "best" cookie and reveal the flag.

**Analysis:**

Searching a cookie name (e.g. the default "snickerdoodle") loads `/check` and the server responds "I love snickerdoodle cookies!" Opening DevTools → **Storage → Cookies** shows the server set a cookie:

```
name = 0
```

<img width="1919" height="1078" alt="image" src="https://github.com/user-attachments/assets/79e3a2f5-210c-4b9c-b841-7571f7278aee" />

So the application doesn't track the cookie *name* you typed — it stores an **integer index** in the `name` cookie. Searching a different cookie changes that index. Editing the cookie value directly to `1` and refreshing returns a different cookie:

```
name = 1   →   "I love chocolate chip cookies!"
```

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/12ffa80c-7fa6-42bd-b340-1655ff16e7a8" />

This confirms the key weakness: the server reads the `name` cookie, maps the integer to a cookie from an internal list, and **trusts whatever value the client sends** — no validation, no signature, no server-side check that the index is one you were legitimately given. Because the cookie lives in the browser, the value is fully attacker-controlled.

The flag belongs to a "best" cookie at an index never shown in the search UI. Incrementing the `name` cookie value and refreshing walks through the list; at index **18** the server returns the flag instead of a cookie message:

```
name = 18   →   Flag revealed
```

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/34918c39-8b00-4974-a8ba-b6e0daa2db03" />

**Finding:** The application used a client-side cookie (`name`) as a trusted integer index into a server-side list of cookies, with no validation. Tampering with the cookie value in DevTools let an unprivileged user reach an index (18) that disclosed the flag — a classic cookie-tampering / insecure-direct-object-reference (IDOR) issue.

**Answer:** `academy{3v3ry1_l0v3s_c00k135_90a3a7cb}`

*(The official picoCTF instance yields the equivalent `picoCTF{...}` flag; this is the CyLab Academy mirror, which rebrands the prefix.)*

---

## Takeaways

- **Cookies are client-side and attacker-controlled.** The browser stores them and sends them back; you can read and edit any non-`HttpOnly` cookie in DevTools → Storage → Cookies. Never trust a cookie value server-side without validating or signing it.
- **Watch what a form actually stores.** The search box looked like the input, but the real state lived in the `name` cookie as an integer — the UI was a layer over an indexed lookup.
- **This is IDOR.** Supplying an object reference (an index/ID) the UI didn't offer, and the server honouring it, is the core of Insecure Direct Object Reference. The same pattern appears with `?id=`, `?user=`, and account numbers — always test adjacent/out-of-range values.
- **Enumeration beats guessing.** The flag index (18) wasn't shown anywhere; stepping through values until the response changed is the intended approach. For larger ranges, script it with `curl --cookie "name=<n>"` in a loop instead of clicking.
- **Defender's view:** validate the index server-side against what the user is authorized to see, sign cookies (e.g. a keyed MAC) so tampering is detectable, and set `HttpOnly`/`Secure`/`SameSite` appropriately.

---
