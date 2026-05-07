---
layout:     post
title:      "Paseo: Usage Notes and Fixes"
subtitle:   "Things to know when running Paseo on a dev server — diagnosed from first principles"
date:       2026-05-01
author:     "luyi"
header-img: "img/post-bg-debug.jpg"
tags:
    - paseo
    - debugging
---

Notes from setting up and using Paseo on a dev server — root cause, diagnosis, and fix for each.

---

## 1. Directory Suggestions Not Working

### Symptom

When typing a directory path (e.g., `/data/work`) in the Paseo app:

- No directory suggestions appear at all.
- Cannot select a workspace.
- The agent never starts — the train stays dead.

Relay logs showed healthy E2EE handshakes and message forwarding. Daemon logs confirmed the `directory_suggestions_request` was arriving and returning in < 3ms — too fast to be a timeout, too empty to be useful.

---

### Root Cause

Paseo daemon's `searchHomeDirectories()` (in `packages/server/src/utils/directory-suggestions.ts`) applies two hard filters:

1. **Absolute paths must live under `$HOME`.** If the daemon runs with `HOME=/home/user`, then `/data/work` is rejected by `isPathInsideRoot()` outright — it never reaches the suggestion list.

2. **Hidden directories are stripped.** Anything under HOME starting with `.` (`.claude`, `.paseo`, `.ssh`) is filtered out. If HOME contains only hidden directories, you see zero suggestions even when typing the HOME path itself.

3. **Symlinks don't help.** Creating `~/work → /data/work` still fails because `realpath()` resolves the symlink target, which remains outside HOME.

The impact chain is straightforward:

```
App input → relay → directory_suggestions_request → daemon
→ searchHomeDirectories() filters out non-HOME paths
→ returns empty result → App shows no suggestions
→ user stuck at workspace selection → agent never created
```

In other words: the directory you want to work in must be a subdirectory of the daemon's `$HOME`. If it isn't, the feature is invisible.

---

### Fix

The principle is simple: make the daemon's `$HOME` point to the parent of your target workspace, so the workspace falls inside the search scope. Symlink the config files the daemon needs from the original HOME.

```bash
# 1. Symlink config files into the new HOME
ln -sf /home/user/.claude   /data/.claude
ln -sf /home/user/.claude.json /data/.claude.json
ln -sf /home/user/.paseo    /data/.paseo

# 2. Stop old daemon and relay
kill $(cat /data/work/relay.pid) 2>/dev/null
kill $(cat /data/work/daemon.pid) 2>/dev/null

# 3. Restart relay
cd /data/work/tiny-paseo-relay
nohup node dist/index.js --port 39217 --host <SERVER_IP> \
  --log /data/work/relay.log \
  > /data/work/relay-stdout.log 2>&1 &
echo $! > /data/work/relay.pid

# 4. Restart daemon with HOME=/data
HOME=/data LD_LIBRARY_PATH=/data/work/lib \
  PASEO_RELAY_ENDPOINT=<SERVER_IP>:39217 \
  nohup paseo daemon start --foreground \
  > /data/work/daemon.log 2>&1 &
echo $! > /data/work/daemon.pid
```

Verify the fix:

```bash
# Confirm the daemon sees the new HOME
cat /proc/$(cat /data/work/daemon.pid)/environ | tr '\0' '\n' | grep HOME
# Should print: HOME=/data
```

Now type `/data/work` in the app — suggestions appear immediately, and the agent starts as expected.

---

## Key Takeaway

This isn't a relay bug, a permissions issue, or a code defect. It's a **design constraint**: the directory suggestion feature is deliberately scoped to `$HOME`. When your workspace lives outside the daemon's HOME, the fix is to widen HOME — not to patch the code.

---
