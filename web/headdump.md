# head-dump — Web Exploitation Writeup

**Category:** Web Exploitation (Easy)
**Challenge:** head-dump (picoCTF 2025, by Prince Niyonshuti N.)
**Author:** Ooi

## Scenario

A simple blog-style news website ("picoCTF News") serving articles about various tech topics. One article discusses backend development and links to API documentation. The goal is to explore the application, find the endpoint that generates a snapshot of the server's memory (a heap dump), download it, and recover a flag that was left sitting in memory.

**Tools used:** Firefox, Swagger UI (the app's own `/api-docs`), `curl`, `file`, `grep`

---

## Finding and exploiting the heap dump (Easy)

**Objective:** Explore the web app, locate the endpoint that dumps server memory, and extract the hidden flag from it.

**Analysis:**

The site is a blog with several posts. The post that matters is the one about **backend development**, tagged `#nodejs`, `#swagger UI`, and `#API Documentation` — a clear pointer that the backend exposes Swagger-generated API docs.

![Homepage post pointing to API documentation](headdump_media/01_homepage_api_post.png)

Following that lead to `/api-docs` opens the Swagger UI, which enumerates every endpoint the backend exposes. Most are ordinary blog routes (`GET /`, `GET /about`, `GET /api/posts`, etc.), but under a **"Diagnosing"** group there's a standout:

```
GET /heapdump   —   Diagnosing the memory allocation.
```

![Swagger UI listing the /heapdump endpoint](headdump_media/02_swagger_endpoints.png)

A **heap dump** is a snapshot of everything the process has allocated in memory at that moment. In a Node.js app, anything the server has touched — config values, environment variables, strings, secrets — can end up captured in it. If a flag is held in memory, a heap dump leaks it. This is the real-world risk class behind exposed diagnostic/debug endpoints (`/heapdump`, `/debug`, actuator endpoints, etc.): memory-inspection tooling left reachable in production.

Executing the endpoint from Swagger returns a downloadable `.heapsnapshot` file with HTTP 200 and `content-type: application/octet-stream`.

![Executing /heapdump in Swagger UI returns a downloadable file](headdump_media/03_heapdump_execute.png)

Rather than click through Swagger, the clean way to grab it is directly with `curl`. `-O` saves the response to a local file named after the URL path:

```bash
curl -O http://chatelaine.cylabacademy.net:16681/heapdump
file heapdump
# heapdump: ASCII text, with very long lines (64671)
```

![curl download and file identification of the heapdump](headdump_media/04_curl_file.png)

`file` reports the dump is just ASCII text with extremely long lines — so there's no need for a dedicated heap-snapshot viewer. A plain text search finds the flag. The flag format is `prefix{...}`, so an extended-regex grep for that shape pulls it straight out without having to scroll through a 64k-character line:

```bash
grep -oE "[A-Za-z0-9_-]+\{[^}]+\}" heapdump
```

- `-o` prints **only** the matched substring, not the whole giant line it lives on.
- `-E` enables extended regex so `{`, `}`, and `+` work without backslash-escaping.
- `[A-Za-z0-9_-]+\{[^}]+\}` matches `anyprefix{anything-up-to-a-closing-brace}` — so it catches the flag regardless of its prefix, which is handy across mirrors that rebrand the `picoCTF{...}` prefix.

![grep extracting the flag from the heapdump](headdump_media/05_grep_flag.png)

The grep returns the flag on the first match (alongside a few unrelated `{...}` noise matches like `d{1,4}` from the dump's internal data, which are easy to tell apart).

**Finding:** The backend exposed an unauthenticated `/heapdump` diagnostic endpoint (discoverable via its own Swagger docs at `/api-docs`) that serialized process memory to a downloadable file. The flag was stored in memory and therefore present as plain text inside the dump.

**Answer:** `academy{Pat!3nt_15_Th3_K3y_b3a35ec3}`

*(On the official picoCTF instance the same technique yields the equivalent `picoCTF{...}` flag — the screenshots here are from the CyLab Academy mirror, which rebrands the prefix.)*

---

## Takeaways

- **Read the content for recon.** The vulnerable endpoint was signposted by a blog post's tags (`#swagger UI`, `#API Documentation`). On web challenges, the site's own text, links, and hashtags are enumeration hints.
- **Swagger / `/api-docs` is free enumeration.** When an app exposes API docs, it hands you the full endpoint list — including diagnostic routes that were never meant to be public.
- **Diagnostic/debug endpoints are a real vuln class.** `/heapdump`, memory snapshots, actuator/debug routes, and profiling endpoints leak secrets straight out of process memory. Finding one reachable without auth is the whole exploit — no payload needed.
- **A heap dump is just text.** `file` first; if it's ASCII, skip the specialised snapshot viewers and `grep` for the flag pattern.
- **Prefix-agnostic flag grep:** `grep -oE "[A-Za-z0-9_-]+\{[^}]+\}" <file>` finds `prefix{...}` flags without knowing the prefix, and `-o` keeps the output sane on files with enormous single lines.

---
