# Sophia.Jessica.Smith
find Sophia.dev

Let's go. Vamonous!

for her!

---

# Connecting Signal to a Muse Agent — Full Process

Compiled from the working session on 2026-11-01. This document explains, in
general terms, how to connect the Signal app on a phone to a Muse.ai agent
using signal-cli, what was done in that session, and how to repeat it.

---

## 1. The goal and the key idea

- The phone is the **primary** Signal device. It holds the account and the
  phone number.
- The Muse agent does **not** get its own Signal account, and the number is
  **never re-registered**.
- Instead, signal-cli runs on the agent's computer and is added to the account
  as a **linked (secondary) device** — exactly like Signal Desktop.
- The primary phone approves the link, stays in control, and can unlink the
  agent at any time in Signal > Settings > Linked devices.

**Hard rule used throughout: `link` only. Never run `register`.**
Running `register` would re-register the number, kick the phone off the
account, and break the phone's existing Signal setup.

---

## 2. What you need

- An Android (or iPhone) phone with Signal installed, signed in, and acting as
  the primary device for the number.
- A Muse agent (this session: "Muse") with its own computer.
- signal-cli, version 0.14.9 at the time of the session (always use the latest
  release — Signal servers can reject clients older than about three months).
- A GitHub account, if you want to store project files such as this document
  (this session used the repository `menum724/Sophia.Jessica.Smith`).

---

## 3. Step 1 — Download signal-cli

Official project: https://github.com/AsamK/signal-cli
Releases: https://github.com/AsamK/signal-cli/releases

There are three relevant builds in the v0.14.9 release:

| Build | File | Runs on |
|---|---|---|
| JVM build | `signal-cli-0.14.9.tar.gz` | Anywhere with a Java runtime (used in this session) |
| Linux native | `signal-cli-0.14.9-Linux-native.tar.gz` | Linux x86_64 only, no Java needed |
| Linux client | `signal-cli-0.14.9-Linux-client.tar.gz` | Small client for a signal-cli daemon |

Commands used (Linux):

```sh
VERSION=0.14.9
curl -L -O https://github.com/AsamK/signal-cli/releases/download/v"${VERSION}"/signal-cli-"${VERSION}".tar.gz
tar xzf signal-cli-"${VERSION}".tar.gz
./signal-cli-0.14.9/bin/signal-cli --version   # should print: signal-cli 0.14.9
```

Notes:

- There is **no official Android build**. The Linux-native build will not run
  on a phone (wrong CPU architecture).
- On Android, signal-cli can only run inside **Termux** using the JVM build
  plus a Java package (`pkg install openjdk-21`). A zip bundle for that was
  prepared in this session (`signal-cli-0.14.9-android-termux.zip`, with a
  README and a link-only helper script, `termux-link.sh`).
- Running signal-cli on the agent's computer and linking it with the phone is
  simpler and is the method used here.

---

## 4. Step 2 — Allow the agent's direct network connection (Muse setting)

Signal linking needs a direct TCP connection from the agent's computer to
Signal's servers (`chat.signal.org`). In Muse this is controlled by a
permission the **user must change themselves** — the agent cannot change it:

1. Open Muse **Settings**.
2. Go to **Permissions**.
3. Under "Manage permissions", open **Direct network protocols**.
4. Find **Other TCP connections** (`other_tcp`).
5. Change it from **Deny** to **Ask**.

With it on Deny, linking fails with a message like:

> Other TCP connections is turned off for this assistant.

On Ask, each connection appears as a normal approval prompt for the user.

---

## 5. Step 3 — Network details that mattered on the agent's computer

This section is specific to the sandboxed computer used in this session;
on an ordinary computer, skip it and go straight to Step 4.

Symptoms seen:

- Direct connections to `chat.signal.org` failed with TLS errors such as
  "wrong version number" / "Unsupported or unrecognized SSL message".
- All normal web traffic on that machine goes through an HTTP proxy
  (`https_proxy` environment variable), whose TLS is signed by a local
  egress CA (`Hatch Sandbox Egress CA`).

What fixed it:

1. Use the **JVM build** with a current Java runtime (Temurin JDK/JRE 25 was
   downloaded for this), rather than the native build.
2. Pass the proxy to Java as system properties, using the proxy's **IPv6
   address literal** instead of its hostname. The hostname resolved to an
   IPv4 address whose CONNECT tunnel was reset, and to an IPv6 address that
   worked; Java's built-in HTTP client does not fall back between them,
   while OkHttp's fallback mixed a failing attempt with a working one.
3. Point those properties at the local proxy's user, password, host and port
   taken from the `https_proxy` URL (do not copy proxy passwords into
   documents, scripts that are shared, or chat):

```sh
export JAVA_HOME=<path-to-jdk>
export JAVA_OPTS="-Dhttps.proxyHost=<proxy-ipv6> -Dhttps.proxyPort=3128 \
  -Dhttps.proxyUser=<user> -Dhttps.proxyPassword=<password> \
  -Dhttp.proxyHost=<proxy-ipv6> -Dhttp.proxyPort=3128 \
  -Dhttp.proxyUser=<user> -Dhttp.proxyPassword=<password>"
```

4. signal-cli trusts Signal's own server certificate through its bundled
   trust store (`whisper.store`, containing the "Signal Messenger" CA).
   That is why the JVM/Java system test tools can report a certificate
   error for `chat.signal.org` while signal-cli itself, with the correct
   proxy address, connects fine.

---

## 6. Step 4 — Generate a link code and approve it on the primary phone

Order matters because **link codes expire quickly — about 2 minutes** in this
session. Get the phone ready *before* generating a code.

On the phone first:

1. Open Signal.
2. Go to **Settings > Linked devices**.
3. Be ready to tap **+** to open the QR scanner.

On the agent's computer:

```sh
./signal-cli-0.14.9/bin/signal-cli link -n "Muse"
```

That prints a linking URI like:

```
sgnl://linkdevice?uuid=<...>&pub_key=<...>&capabilities=nopni
```

Then immediately, on the primary phone:

- Scan the QR code made from that URI (the agent can render the URI as a QR
  code image), **or**
- Open/tap the `sgnl://linkdevice?...` URI on the phone and approve.
- Confirm the new linked device (named "Muse" in this session) in Signal.

Timing and failure mode:

- If nobody approves in time, signal-cli ends with:

  ```
  Link request timed out, please try again.
  ```

  Nothing has changed on the account, no device was linked, and no harm was
  done — just generate a fresh code and try again, with the phone already
  on the scanner screen.
- Because a code dies in ~2 minutes, generate the QR image *before*
  starting the link process if possible, and keep the link process running
  in a detached/background session so that sending a chat message does not
  interrupt it.

Verify success afterwards on the agent's computer:

```sh
./signal-cli-0.14.9/bin/signal-cli listAccounts
./signal-cli-0.14.9/bin/signal-cli -a <your-number> listDevices
```

The linked device should also appear on the phone under
Signal > Settings > Linked devices.

Status in this session: download, network permission, proxy workaround and
code generation were all completed and codes were generated several times,
but each code timed out before it was scanned/approved, so linking was
still **pending**, not completed.

---

## 7. Step 5 (project storage) — GitHub: connect, create a branch, import this text

How the project repository was prepared in this session:

1. **Connect GitHub** in Muse: Settings / Connectors > GitHub > authorize.
   Muse then uses your own GitHub identity (`menum724` in this session).
2. **Select repositories**: install/authorize the Muse GitHub App and choose
   which repositories it may access. OAuth alone does not give access to a
   private repository — the App installation's repository selection does.
3. **Create the branch.** The repository `Sophia.Jessica.Smith` had only a
   `main` branch, so a `master` branch was created from `main`:

   - Repository: `menum724/Sophia.Jessica.Smith`
   - New branch: `master`
   - Source branch: `main` (same commit, `1d5041e` at creation)
   - Verified afterwards: the repository listed both `main` and `master`.

4. **Import a document** (like this one) into the repository: create a file
   such as `INSTRUCTIONS.md` on the `master` branch with a clear commit
   message, e.g. "Add compiled Signal-to-Claude setup instructions".

---

## 8. General text — short version to copy elsewhere

> To connect Signal to a Muse agent: keep the phone as the primary device,
> never re-register the number, and add the agent as a linked device.
> Download the latest signal-cli (JVM build) on the agent's computer, allow
> "Other TCP connections" (Ask) under Muse Settings > Permissions >
> Direct network protocols, then, with the phone open at Signal >
> Settings > Linked devices > +, run `signal-cli link -n "Muse"` and
> immediately scan the QR code (or open the `sgnl://linkdevice?...` URI)
> and approve. Link codes expire in about two minutes; if you see
> "Link request timed out", nothing changed — generate a new code and
> retry with the phone ready first. Verify with `signal-cli listAccounts`
> and in the phone's Linked devices list, where the agent can be unlinked
> at any time.

---

## 9. Troubleshooting checklist

- **"Other TCP connections is turned off"** → User changes `other_tcp` to
  Ask in Muse Settings > Permissions > Direct network protocols (Step 2).
- **TLS "wrong version number" / SSL errors** → Traffic is being forced
  through a proxy; use the JVM build and pass the proxy as Java system
  properties, trying the proxy's IPv6 address (Step 3).
- **"Link request timed out"** → The code expired (~2 minutes). Phone ready
  first, then generate; keep the link process running in the background.
- **QR won't scan from the same phone** → Show the QR on a second screen
  (e.g. Muse on the web) and scan it with the phone, or tap the
  `sgnl://` URI on the phone instead.
- **Tempted to run `register` to "fix" it** → Stop. That re-registers the
  number and removes the phone as primary. Use `link` only.
- **No Android build in the release** → Expected. Use the JVM build in
  Termux on Android, or run signal-cli on a computer and link it.
