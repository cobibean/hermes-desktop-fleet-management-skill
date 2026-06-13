---
name: hermes-desktop-fleet-management
description: Diagnose and safely repair Hermes Desktop, Hermes Dashboard, and remote Hermes agent fleet issues. Use when Hermes Desktop is stuck connecting, a remote gateway URL or session token may be wrong, profiles route to the wrong backend or workspace, dashboard services need verification, `/api/status` works but profile/session routes fail, Electron cache/state may be stale, or a remote Hermes profile works in one surface such as Telegram but fails in Desktop or Dashboard.
---

# Hermes Desktop Fleet Management

Use this skill to troubleshoot Hermes Desktop and remote Hermes fleets without
breaking working profiles. Treat it as an evidence-first runbook for routing,
auth, service health, version skew, Desktop state, and workspace/cwd bleed.

Never print tokens, API keys, private SSH keys, or bearer values.

## First Principle

Do not start by reinstalling, rotating every token, or restarting every service.
First identify which layer is failing:

- Desktop app bundle
- Desktop user data, active profile, or local storage
- URL/token pairing
- remote dashboard service
- remote agent runtime
- model/provider auth
- dashboard/backend API version
- recorded session cwd
- current renderer cwd state

When one profile fails, preserve working profiles unless evidence shows a shared
dependency is down.

## Fast Triage

Answer these before making changes:

1. Which surface fails: Desktop, Dashboard, messaging platform, or all of them?
2. Does the relevant backend answer `/api/status`?
3. Does auth fail with 401/403, or does a specific route 404?
4. Is Desktop using the intended global/default backend or a named-profile
   override?
5. Is the profile inheriting a global all-profiles backend when it should have
   a dedicated backend?
6. Is Desktop booting with the intended active profile?
7. Are fresh sessions wrong, or only old recorded sessions?
8. Are all-profile endpoints slow enough to trip Desktop timeouts?

Keep a short table of backend URL, profile, purpose, service name, auth source,
and default cwd. Redact tokens.

## Symptom Map

- `/api/status` works but a specific route returns 404: suspect Desktop/frontend
  and remote dashboard backend version skew. Align versions and restart only the
  affected dashboard service.
- 401/403 on dashboard REST or WebSocket: suspect wrong URL/token pair or expired
  dashboard token.
- Agent response says auth/provider expired while dashboard endpoints work:
  refresh model/provider auth for that profile, not dashboard tokens.
- Telegram or another messaging surface works but Desktop fails: suspect Desktop
  routing, Desktop local state, dashboard backend, or cwd/session state before
  blaming the agent runtime.
- Desktop stuck on "connecting" with timeout logs: time `/api/status`,
  `/api/profiles`, and `/api/profiles/sessions`. Heavy all-profile aggregation
  can make a healthy backend look dead to Desktop.
- A named profile creates sessions in another profile's workspace: suspect
  Desktop active profile, renderer cwd state, stale local storage, or recorded
  session cwd.

## Preferred Fleet Shape

Use a global backend for fleet overview and dedicated backends for active chat
profiles:

```text
All Profiles / default -> global machine backend
named profile A        -> dedicated profile backend
named profile B        -> dedicated profile backend
```

Rules:

- The global backend is for fleet overview and all-profile aggregation.
- Active, frequently used profiles should have dedicated isolated dashboard
  services.
- Desktop should have explicit per-profile overrides for dedicated profiles.
- Do not treat "default" and a named profile as interchangeable.
- Keep URL/token pairs matched; never combine a URL from one profile or host with
  a token from another.

## Mac Desktop State

Deleting the app bundle does not clear Desktop state. Check the app support
folder for:

```text
~/Library/Application Support/Hermes/connection.json
~/Library/Application Support/Hermes/active-profile.json
~/Library/Application Support/Hermes/Local Storage/leveldb/
```

Check logs:

```text
~/.hermes/logs/desktop.log
~/.hermes/logs/bootstrap-installer.log
```

The active profile file should normally boot Desktop into the global/default
backend, not a single busy named profile:

```json
{
  "profile": "default"
}
```

If Desktop must be reset, prefer a controlled user-data reset over a blind app
reinstall:

1. Quit Hermes Desktop.
2. Back up `~/Library/Application Support/Hermes`.
3. Recreate the live folder with only known-good `connection.json`,
   `active-profile.json`, and optional UI preference files.
4. Set `active-profile.json` to the desired global/default profile.
5. Relaunch and verify logs show Desktop connecting to the intended global
   backend.

Only delete app bundles or `~/.hermes/hermes-agent` when evidence points at a
broken install or source checkout.

## Remote Service Checks

Hermes dashboard services may run under a user systemd manager rather than the
system manager. Verify the actual service manager before concluding a service is
missing.

For root user systemd setups:

```bash
export XDG_RUNTIME_DIR=/run/user/0
systemctl --user list-units --type=service --all | grep hermes
systemctl --user status hermes-dashboard-fleet.service
systemctl --user status hermes-dashboard-<profile>.service
```

Useful process and port checks:

```bash
ss -ltnp | grep ':<port>'
ps -eo pid,ppid,etimes,user,args | grep 'hermes dashboard' | grep -v grep
```

Restart only the failing service unless a shared dependency is proven down:

```bash
export XDG_RUNTIME_DIR=/run/user/0
systemctl --user restart hermes-dashboard-<profile>.service
```

## API Checks

Use bearer auth for protected REST calls, but do not print tokens:

```bash
curl -sS "$BASE_URL/api/status"
curl -sS -H "Authorization: Bearer $TOKEN" "$BASE_URL/api/profiles"
curl -sS -H "Authorization: Bearer $TOKEN" "$BASE_URL/api/profiles/sessions?profile=<profile>&limit=5"
curl -sS -H "Authorization: Bearer $TOKEN" "$BASE_URL/api/fs/default-cwd"
```

Time slow endpoints:

```bash
time curl -sS -H "Authorization: Bearer $TOKEN" "$BASE_URL/api/profiles" >/dev/null
```

Interpretation:

- fast `/api/status` proves reachability, not full compatibility.
- slow `/api/profiles` or all-profile session aggregation can break Desktop
  even when profile-specific chat works.
- a dedicated profile backend's `/api/fs/default-cwd` should resolve to that
  profile's intended workspace.

## CWD And Session Bleed

Separate three cwd layers:

- remote service default cwd: what the backend reports as its default workspace.
- Desktop renderer cwd state: live Electron/local-storage workspace state.
- recorded session cwd: cwd stored at session creation time.

Old bad sessions keep their stored cwd after configuration is fixed. Verify with
fresh sessions when checking a repair.

A neutral global cwd, such as a Hermes config directory, may be acceptable for
fleet/default views. The dangerous case is a named profile creating a fresh
session inside another named profile's project workspace.

If fresh sessions still bleed into another profile's workspace after service and
connection config look right, perform the controlled Desktop user-data reset.

## Verification

After any fix:

1. Confirm only intended services changed.
2. Confirm `/api/status` on global and affected profile backends.
3. Confirm Desktop logs show the intended global/default backend on startup.
4. Start genuinely new Desktop chats for the affected profiles.
5. Ask each profile to report active profile, cwd, host, and home/workspace.
6. Test the same profile on another surface, such as Dashboard or messaging, if
   available.
7. Record what changed, what was verified, and what remains risky.

## Do Not Do

- Do not paste tokens into chat, docs, commits, screenshots, or public repos.
- Do not rotate all dashboard tokens for a profile-specific failure unless auth
  evidence requires it.
- Do not restart the whole fleet because one profile is broken.
- Do not put a remote URL in a local gateway field.
- Do not assume reinstalling the app clears Electron user data.
- Do not rely on local app source patches as the durable fix when app updates can
  overwrite them.
