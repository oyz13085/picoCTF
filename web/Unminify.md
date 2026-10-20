# Unminify — Web Exploitation Writeup

**Category:** Web Exploitation (Easy)
**Challenge:** Unminify (picoCTF 2024, by Jeffery John)
**Author:** Ooi

## Scenario

The site's author "squished" their page's code to make it load faster — i.e. the HTML has been **minified** (whitespace and line breaks stripped, everything crammed together). The flag was swept up in that minified markup. The task is to inspect the page and dig it out.

**Tools used:** Firefox DevTools (Inspector)

---

## Flag buried in minified markup (Easy)

**Objective:** Browse to the page and find the flag hidden in its squished/minified code.

**Analysis:**

The rendered page is a friendly "Welcome to my flag distribution website!" message that explicitly jokes: *"I just deliver flags, I don't know how to read them…"* — i.e. the flag is present in the page's code, just not displayed as text.

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/cac97797-2381-472c-8c2d-ea983dc51d73" />

Minification removes line breaks and indentation so the whole document sits on a few long lines — great for load speed, miserable to read by eye. The browser's Inspector solves that for free: it re-renders the live DOM as a clean, indented tree regardless of how the source was formatted. Expanding the `<body>` shows the flag hiding in plain sight as a bogus **`class` attribute** on an otherwise empty paragraph:

```html
<p class="academy{pr3tty_c0d3_f2dc9e9e}"></p>
```

<img width="1215" height="644" alt="image" src="https://github.com/user-attachments/assets/55ccee20-1e0f-43a1-925c-e68cef8bacb2" />

The `<p>` has no text content — its only job is to carry the flag as a class name, where it blends into the surrounding `class="academy{}"` attributes used throughout the page as styling noise. No decoding needed; the flag is plaintext, just buried in minified markup.

**Finding:** The flag was stored as the `class` attribute of an empty `<p>` element and hidden inside minified HTML. The DOM inspector's auto-formatted tree exposes it immediately.

**Answer:** `academy{pr3tty_c0d3_f2dc9e9e}`

*(The official picoCTF instance yields the equivalent `picoCTF{...}` flag; this is the CyLab Academy mirror, which rebrands the prefix.)*

---

## Takeaways

- **Minified code isn't hidden code.** The Inspector (and "Pretty Print `{ }`" in the Debugger/Style Editor) re-formats squished HTML/JS/CSS into a readable tree, so minification stops nothing.
- **Attributes are a common flag hiding spot.** `class`, `id`, `data-*`, and custom attributes get overlooked because they don't render — always scan element attributes, not just visible text.
- **Decode only when needed.** This flag was plaintext; not everything in an attribute is Base64. Read it first, decode only if it actually looks encoded.
- **Recurring recon pattern:** the flag lives where the rendered page doesn't show it but the DOM/response does — endpoints, custom attributes, class names, comments. Inspect first, always.

---
