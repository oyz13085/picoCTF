# WebDecode — Web Exploitation Writeup

**Category:** Web Exploitation (Easy)
**Challenge:** WebDecode (picoCTF 2024, by Nana Ama Atombo-Sackey)
**Author:** Ooi

## Scenario

A small three-page static site (HOME / ABOUT / CONTACT) that openly challenges the visitor to find a flag. The challenge tests whether you can use the browser's web inspector to find data hidden in the page's DOM that isn't visible in the rendered page.

**Tools used:** Firefox DevTools (Inspector), CyberChef (Base64 decode)

---

## Flag hidden in a DOM attribute (Easy)

**Objective:** Use the web inspector to locate hidden data in the page and decode it into the flag.

**Analysis:**

The HOME page is pure misdirection — a big "Ha!!!!! You looking for a flag?" banner with "Keep Navigating" underneath. No flag in the visible text; the message is just telling you to look at the other pages.

<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/14a22202-54be-4e5e-8c37-db21a49e0d5c" />

The ABOUT page gives the actual technique hint in its heading: **"Try inspecting the page!! You might find it there."** That's a direct instruction to stop reading the rendered page and open the browser's DOM inspector.

<img width="1919" height="1007" alt="image" src="https://github.com/user-attachments/assets/a973f621-604c-4421-b9f0-1502521f18a0" />

Opening DevTools → Inspector on the ABOUT page and expanding the `<body>` reveals a `<section class="about">` element carrying a **custom, non-standard HTML attribute**:

```html
<section class="about" notify_true="YWNhZGVteXt3ZWJfc3VjYzNzc2Z1bGx5X2QzYzBkZWRfM2NkMTQ0ZTV9">
```

<img width="1211" height="339" alt="image" src="https://github.com/user-attachments/assets/0108b32f-2b7d-44a4-b873-fcf23090d9c0" />

`notify_true` isn't a real HTML attribute — it was invented to smuggle data into the markup. Its value is a long alphanumeric string with the structure of **Base64** (A–Z, a–z, 0–9, no spaces). This is why the data is invisible in the rendered page but trivially findable in the DOM: browsers simply ignore attributes they don't recognize, so nothing displays, but the inspector shows the raw element.

Decoding the attribute value from Base64 (CyberChef → *From Base64*, or `base64 -d` in a terminal) produces the flag.

**Finding:** The flag was Base64-encoded and stashed in a custom `notify_true` attribute on the ABOUT page's `<section>` element — hidden from the rendered page but visible in the DOM inspector.

**Answer:** `academy{web_succ3ssfully_d3c0ded_3cd144e5}`

*(The official picoCTF instance yields the equivalent `picoCTF{...}` flag; this is the CyLab Academy mirror, which rebrands the prefix.)*

---

## Takeaways

- **View Source ≠ Inspector.** View Source shows the original HTML sent by the server; the Inspector shows the live DOM, including attributes and script-injected content. When a challenge says "inspect," open DevTools, don't just `Ctrl+U`.
- **Custom/non-standard attributes are a hiding spot.** Browsers silently ignore attributes they don't understand (`notify_true`, `data-secret`, etc.), so they never render but sit in plain sight in the DOM.
- **Recognize Base64 on sight.** Mixed-case letters + digits, often ending in `=`/`==`, length a multiple of 4. Decode with CyberChef *From Base64* or `echo <str> | base64 -d`.
- **Read the page as instructions.** The site literally told you the method ("Try inspecting the page") — challenge copy, headings, and taunts are recon hints, not decoration.

---
