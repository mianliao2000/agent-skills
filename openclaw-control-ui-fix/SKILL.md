---
name: openclaw-control-ui-fix
description: "Fix missing Control UI assets after upgrading OpenClaw to 2026.3.22+. Use when: `openclaw gateway` reports 'Missing Control UI assets at .../dist/control-ui/index.html' and the browser UI is unavailable. Builds the Vite/Lit SPA from source and copies it into the installed npm package."
metadata:
  {
    "openclaw":
      {
        "emoji": "🔧",
        "requires": { "bins": ["git", "pnpm", "node"] },
      },
  }
---

# OpenClaw Control UI Fix

## Problem

After upgrading OpenClaw to **2026.3.22** (or later), running `openclaw gateway` shows:

```
Missing Control UI assets at .../node_modules/openclaw/dist/control-ui/index.html.
Build them with `pnpm ui:build` (auto-installs UI deps).
```

The `dist/control-ui/` directory is **not included** in the published npm package — it must be built from the OpenClaw source repository. The `scripts/ui.js` build script is also absent from the npm package, so `pnpm ui:build` cannot be run directly against the installed package.

## Prerequisites

- `git` installed
- `pnpm` installed (`npm install -g pnpm` if missing)
- `node` 18+

## Fix Steps

### 1. Find the installed openclaw package path

```bash
# On Linux/macOS
OPENCLAW_PKG=$(npm root -g)/openclaw

# On Windows (Git Bash / PowerShell)
OPENCLAW_PKG=$(npm root -g | sed 's|\\|/|g')/openclaw
```

Verify:

```bash
ls "$OPENCLAW_PKG/dist"   # should exist, no control-ui folder yet
```

### 2. Clone the openclaw source repo (shallow)

```bash
git clone --depth=1 https://github.com/openclaw/openclaw.git /tmp/openclaw-src
```

### 3. Install UI build dependencies

Run from inside the cloned repo root (not the `ui/` subdirectory) so the pnpm workspace resolves correctly:

```bash
cd /tmp/openclaw-src
CI=true pnpm install
```

`CI=true` is required to skip the interactive TTY prompt when node_modules already exist.

### 4. Build the Control UI

```bash
cd /tmp/openclaw-src/ui
pnpm run build
```

The output lands at `/tmp/openclaw-src/dist/control-ui/`.

### 5. Copy built assets into the installed package

```bash
cp -r /tmp/openclaw-src/dist/control-ui "$OPENCLAW_PKG/dist/control-ui"
```

On **Windows** (Git Bash):

```bash
cp -r /tmp/openclaw-src/dist/control-ui "$(npm root -g | sed 's|\\\\|/|g')/openclaw/dist/control-ui"
```

### 6. Verify

```bash
ls "$OPENCLAW_PKG/dist/control-ui"
# Expected: apple-touch-icon.png  assets/  favicon-32.png  favicon.ico  favicon.svg  index.html
```

### 7. Start the gateway

```bash
openclaw gateway
```

Open [http://127.0.0.1:18789/](http://127.0.0.1:18789/) — the Control UI should load.

## Notes

- This fix only needs to be applied once per openclaw version upgrade if the package omits the built UI.
- The cloned source (`/tmp/openclaw-src`) can be deleted after the copy.
- On Windows, `/tmp` maps to `C:\Users\<you>\AppData\Local\Temp` in Git Bash.
- If `pnpm install` fails with `ERR_PNPM_ABORTED_REMOVE_MODULES_DIR_NO_TTY`, set `CI=true` as shown above.
