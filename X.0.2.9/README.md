# TCSKA.docker X.0.2.9

Cloned from `X.0.2.8` with three Windows / Debian-EOL fixes baked in, plus a `.gitattributes` to keep future Windows checkouts from re-introducing the CRLF problem.

The TCSKA fork sources used by this image (via `[sources]` in `buildout.cfg`):

| Package | Repo |
|---|---|
| `senaite.core`    | `github.com/TCS-Laboratory/TCSKA.core.git` (branch `2.x`) |
| `senaite.impress` | `github.com/TCS-Laboratory/TCSKA.impress.git` (branch `2.x`) |

All other packages still come from upstream `senaite/*`.

---

## Fixes vs. X.0.2.8

### 1. Debian buster apt sources (Dockerfile)

**Why:** Debian buster reached EOL. The main mirror (`deb.debian.org`) no longer serves it, so the original `apt-get update` 404s. The X.0.2.8 Dockerfile already rewrites to `archive.debian.org`, but two further problems remain:

- `buster-updates` was never moved to `archive.debian.org` — apt 404s on that line and the whole `update` fails.
- The archived `Release` files are past their `Valid-Until` date — apt refuses them by default.

**Fix:** In the big `RUN` block (around line 34), the sed/apt prelude now reads:

```dockerfile
RUN sed -i 's|http://deb.debian.org/debian|http://archive.debian.org/debian|g' /etc/apt/sources.list && \
    sed -i 's|http://security.debian.org/debian-security|http://archive.debian.org/debian-security|g' /etc/apt/sources.list && \
    sed -i '/buster-updates/d' /etc/apt/sources.list && \
    echo 'Acquire::Check-Valid-Until "false";' > /etc/apt/apt.conf.d/99no-check-valid-until && \
    apt-get update \
```

Two added lines: drop the dead `buster-updates` entry, accept expired Release files.

### 2. CRLF in `build_deps.txt` / `run_deps.txt` (Dockerfile)

**Why:** When the repo is checked out on Windows with default `core.autocrlf=true`, the deps files get CRLF line endings. The Dockerfile does `$(grep ... | tr "\n" " ")`, which leaves the `\r` attached to every package name. `apt-get install` then can't find `dpkg-dev\r`, `gcc\r`, etc., and reports "Unable to locate package" for every single one — even though `apt-get update` succeeded.

**Fix:** Three occurrences in the Dockerfile (two for `apt install`, one for the final `apt purge`):

```diff
- $(grep -vE "^\s*#" /build_deps.txt | tr "\n" " ")
+ $(grep -vE "^\s*#" /build_deps.txt | tr -d "\r" | tr "\n" " ")
```

### 3. CRLF in `docker-entrypoint.sh` and `docker-initialize.py`

**Why:** Same Windows-checkout cause. With CRLF endings, Linux tries to exec `/bin/bash\r` from the shebang and fails with `no such file or directory`. The container starts then dies immediately.

**Fix:** Both files are committed with LF line endings in this folder, **and** a `.gitattributes` is added to lock them to LF on future checkouts regardless of the user's `core.autocrlf` setting:

```
*           text=auto eol=lf
*.sh        text eol=lf
*.py        text eol=lf
*.cfg       text eol=lf
*.txt       text eol=lf
Dockerfile  text eol=lf
```

---

## How to start the project

All commands assume you are in this folder (`X.0.2.9`) and Docker Desktop is running. On Windows git-bash, prefix with `MSYS_NO_PATHCONV=1` when passing absolute container paths via `--entrypoint` / `-v`.

### 1. Build the image

```bash
docker build -t tcska:X.0.2.9 .
```

First build: ~20–30 minutes (Plone 5.2.15 download, apt, buildout, all git clones, pip installs). Subsequent builds reuse layers and are much faster.

### 2. Run a standalone instance (development)

```bash
docker run -d --name tcska -p 8080:8080 tcska:X.0.2.9
```

Wait ~20–30 s for Zope to come up, then open **http://localhost:8080**. On the welcome page click **"Create a new Plone site"** and log in with:

- **Username:** `admin`
- **Password:** `admin`

### 3. Useful runtime commands

```bash
docker logs -f tcska                      # follow startup / runtime logs
docker exec -it tcska bash                # shell into the running container
docker stop tcska                         # graceful stop
docker start tcska                        # restart (preserves /data state)
docker rm -f tcska                        # remove container (data persists if mounted)
```

### 4. Dev loop (edit-in-container)

Sources are checked out at `/home/senaite/senaitelims/src/` as Plone develop eggs:

```bash
docker exec -it tcska bash
cd src/senaite.core         # or src/senaite.impress
# edit files in place
# restart Zope to pick up Python changes
gosu senaite bin/instance restart
# when happy:
git status
git commit -am "your message"
git push origin 2.x
```

> **Note:** git push from inside the container needs your GitHub credentials inside the container. Easiest: mount your host `~/.gitconfig` and SSH key, or use a Personal Access Token cached via `git config credential.helper store`.

### 5. ZEO cluster mode (closer to production)

```bash
docker run -d --name=zeo tcska:X.0.2.9 zeo
docker run -d --name=inst1 --link=zeo -e ZEO_ADDRESS=zeo:8080 -p 8081:8080 tcska:X.0.2.9
docker run -d --name=inst2 --link=zeo -e ZEO_ADDRESS=zeo:8080 -p 8082:8080 tcska:X.0.2.9
```

### 6. Verify the fork sources are actually loaded

```bash
docker exec tcska git -C /home/senaite/senaitelims/src/senaite.core    remote -v
docker exec tcska git -C /home/senaite/senaitelims/src/senaite.impress remote -v
```

Both should print `github.com/TCS-Laboratory/TCSKA.*.git`.

---

## English-label overrides (External_Deps)

The sibling repository [`TCS-Laboratory/External_Deps`](https://github.com/TCS-Laboratory/External_Deps) holds files used to **rebrand the SENAITE UI text** — i.e. replace upstream English strings (e.g. "Sample", "Analysis Request") with TCSKA-preferred wording. Everything in it is still English; it is *not* a foreign-language translation.

### Contents

| File | Purpose |
|---|---|
| `senaite.core.po` | gettext source — human-editable. Each `msgid` is the original English label; the matching `msgstr` is the TCSKA replacement. |
| `senaite.core.mo` | compiled binary form of the `.po` file. Plone's i18n machinery loads `.mo` at runtime, not `.po`. |
| `en.xml` | XML resource shipped alongside the translations (likely a Plone `portal_setup` / GenericSetup profile export). Review before use. |

### How to turn it on inside the running container

SENAITE/Plone reads translation files from `locales/en/LC_MESSAGES/` inside the `senaite.core` source tree. The `.mo` file is what gets loaded; `.po` is only consulted if you rebuild.

```bash
# 1. Clone the override repo on the host (one-time)
git clone https://github.com/TCS-Laboratory/External_Deps.git \
  ~/Desktop/TCS-Laboratory/External_Deps

# 2. Copy the compiled .mo into the running container's senaite.core locales
docker cp ~/Desktop/TCS-Laboratory/External_Deps/senaite.core.mo \
  tcska:/home/senaite/senaitelims/src/senaite.core/src/senaite/core/locales/en/LC_MESSAGES/senaite.core.mo

# 3. (Optional) Also drop the .po next to it for future rebuilds
docker cp ~/Desktop/TCS-Laboratory/External_Deps/senaite.core.po \
  tcska:/home/senaite/senaitelims/src/senaite.core/src/senaite/core/locales/en/LC_MESSAGES/senaite.core.po

# 4. Restart Zope so the new translation catalog is picked up
docker exec tcska gosu senaite bin/instance restart
```

> The exact destination path (`.../src/senaite/core/locales/en/LC_MESSAGES/`) follows the standard Plone/SENAITE layout. If a future version of `senaite.core` reorganises its locales directory, adjust accordingly. Verify with:
> ```bash
> docker exec tcska find /home/senaite/senaitelims/src/senaite.core -name "*.mo" -path "*/locales/*"
> ```

### Persistent, build-time override (preferred for production)

Instead of `docker cp` after every fresh container, bake the override into the image by mounting `External_Deps` as a volume or copying it during build. Two options:

**a) Volume mount at run time** — keeps the override editable on the host:

```bash
docker run -d --name tcska -p 8080:8080 \
  -v ~/Desktop/TCS-Laboratory/External_Deps/senaite.core.mo:/home/senaite/senaitelims/src/senaite.core/src/senaite/core/locales/en/LC_MESSAGES/senaite.core.mo:ro \
  tcska:X.0.2.9
```

**b) Bake into the image** — add to `Dockerfile` after the buildout step:

```dockerfile
COPY senaite.core.mo /home/senaite/senaitelims/src/senaite.core/src/senaite/core/locales/en/LC_MESSAGES/senaite.core.mo
```

…and place a copy of `senaite.core.mo` alongside the Dockerfile in this folder before building.

### Editing the strings

`.po` is the source of truth — edit `msgstr` entries (never `msgid`). Then recompile:

```bash
docker exec tcska msgfmt \
  /home/senaite/senaitelims/src/senaite.core/src/senaite/core/locales/en/LC_MESSAGES/senaite.core.po \
  -o /home/senaite/senaitelims/src/senaite.core/src/senaite/core/locales/en/LC_MESSAGES/senaite.core.mo
docker exec tcska gosu senaite bin/instance restart
```

When happy, commit the updated `.po` and `.mo` back to `TCS-Laboratory/External_Deps`.

---

## ATLAS — built-in Gemini chatbot

The fork pulls **`TCSKA.core`** which contains the `senaite.core.browser.aichat` subpackage (the ATLAS floating chat widget). It is loaded automatically — no toggle, no install step. It hits the live Zope catalog so it can answer questions like:

- *how many clients do I have?*
- *list my samples*
- *tell me about client sagner with all details*

ATLAS works in two modes depending on whether you have a Gemini key:

| Mode | What you get |
|---|---|
| **No `GEMINI_API_KEY`** set | Widget still renders. Replies are stub messages that include the live catalog counts. Useful for confirming the wiring is alive. |
| **`GEMINI_API_KEY` set** | Real Gemini 2.5 Flash replies, grounded in the catalog summary + top hits + object-field details. |

### Getting a Gemini key

Free key (free-tier quota): https://aistudio.google.com/app/apikey

> A key in the `AIzaSy…` (39 chars) format is what `gemini-2.5-flash` accepts. Keys in `AQ.Ab…` format are OAuth tokens, not API keys.

### Run with the key (Windows / Linux / macOS)

```bash
docker build -t tcska:X.0.2.9 .
docker run -d --name tcska -p 8080:8080 \
  -e GEMINI_API_KEY=AIzaSy....your-key.... \
  tcska:X.0.2.9
```

The key is **only** passed as a process env var. It is not written to any file in the repo and **must not** be committed. If you accidentally do, revoke at https://aistudio.google.com/app/apikey and re-issue.

### Hot-swap the key on an existing container

You can't change env vars on a running container, but you can preserve the ZODB volume:

```bash
docker rename tcska tcska-old
docker run -d --name tcska -p 8080:8080 \
  -e GEMINI_API_KEY=AIzaSy...new... \
  --volumes-from tcska-old \
  tcska:X.0.2.9
docker rm tcska-old        # only after you confirm tcska works
```

### Verifying ATLAS

After the container boots:

```bash
# Should print {"ok": true, "reply": "...", "summary": {...}, "context": [...]}
curl -u admin:admin "http://localhost:8080/senaite/@@aichat-query?message=how+many+clients"
```

In the browser, log into `/senaite/` and look for the **blue ☀ button bottom-right** — that's ATLAS. Click → ask anything. If the button doesn't appear, hard-refresh (Ctrl+F5) to bust the old viewlet cache.

### Switching the Gemini model

Edit `GEMINI_ENDPOINT` in `src/senaite.core/src/senaite/core/browser/aichat/gemini.py` and `docker restart tcska`. Free-tier accessible models with a fresh AI Studio key currently include `gemini-2.5-flash`, `gemini-2.5-flash-lite`, `gemini-flash-lite-latest`. `gemini-2.0-flash` has zero free quota.

### What ATLAS sees

Every chat turn the backend sends Gemini:

1. **`COUNTS`** — across all SENAITE catalogs (`portal_catalog`, `senaite_catalog_client`, `senaite_catalog_sample`, `senaite_catalog_analysis`, `senaite_catalog_worksheet`, `senaite_catalog_setup`, `bika_setup_catalog`, `bika_catalog`).
2. **`SEARCH RESULTS`** — top 10 matches. Keyword-first (`client`, `sample`, `batch`, `worksheet`, `analysis`, `instrument`, …) then `SearchableText` fallback.
3. **`DETAILS`** — top 5 results have their content object fetched and ~30 common SENAITE accessors serialized (`getName`, `getEmailAddress`, `getPhone`, `getTaxNumber`, `getClientID`, `getRequestID`, `getDateSampled`, `getSampleTypeTitle`, `getContactFullName`, etc.).

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `apt-get update` returns `404` on `buster-updates` | Debian retired the suite | Already patched in this Dockerfile. |
| `apt-get install` says `Unable to locate package <name>` | CRLF in deps files | Already patched (`tr -d "\r"`). |
| Container exits immediately with `exec /docker-entrypoint.sh: no such file or directory` | CRLF in entrypoint script | Already patched via LF on disk + `.gitattributes`. |
| `docker build` fails with `docker-credential-desktop: executable file not found` | git-bash PATH missing Docker Desktop's bin | `export PATH="/c/Program Files/Docker/Docker/resources/bin:$PATH"` (or restart the terminal after installing Docker Desktop). |
| Build errors about path-conversion from git-bash | MSYS mangles `/bin/...` paths in `--entrypoint` | `export MSYS_NO_PATHCONV=1` before docker commands. |
