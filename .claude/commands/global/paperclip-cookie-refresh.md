---
name: paperclip-cookie-refresh
description: "Test and (if needed) refresh the Paperclip web-API session cookie (~/.config/paperclip/session.txt). Headless — no Playwright. Shares one implementation with the paperclip-cookie-watchdog DAG."
---

# /paperclip-cookie-refresh

Refreshes the shared Paperclip session cookie (`paperclip-default.session_token`,
~7-day lifetime) that the hermes-adapter uses to fetch every process-agent's work.
All 14 process agents depend on it, so a lapsed cookie 401s the whole fleet.

Refresh is **headless** via better-auth email+password sign-in (svc acct
`dagu-automation@paperclip.local`, password in Keychain `paperclip-getaccess-cloud`).
There is ONE implementation — `00_Governance/scripts/paperclip-cookie-refresh.sh` — used
by both this command and the autonomous `paperclip-cookie-watchdog.yaml` Dagu DAG.

## Usage

```bash
# Refresh only if invalid/missing/older-than-6d (safe to run anytime):
~/projects/00_Governance/scripts/paperclip-cookie-refresh.sh --ensure

# Force a fresh mint now:
~/projects/00_Governance/scripts/paperclip-cookie-refresh.sh --force

# Probe only (exit 0 = valid, 1 = bad):
~/projects/00_Governance/scripts/paperclip-cookie-refresh.sh --check
```

The script atomic-writes the cookie to `~/.config/paperclip/session.txt`, mirrors a
backup to `~/.hermes/secrets/paperclip_session.txt`, and validates the minted cookie
**before** overwriting the existing one (never clobbers a working cookie with a bad mint).
The adapter reads the file per `/run`, so no restart is needed.

## If it fails

Exit nonzero means the headless sign-in failed — almost always the service-account
password changed or the account is locked. Fix Keychain entry `paperclip-getaccess-cloud`
(`security add-generic-password -U -s paperclip-getaccess-cloud -a dagu-automation -w <pw>`),
then re-run with `--force`. If the account itself needs re-bootstrapping, see
[reference_paperclip.md](/Users/jcords-macmini/.claude/projects/-Users-jcords-macmini-projects/memory/reference_paperclip.md).

## What this is NOT

- Not the container-side Claude OAuth refresh — that's `paperclip-token-watchdog.yaml`.
- Not for reading/writing issues — use `/paperclip`.
- No longer uses Playwright (the old browser-login flow is superseded by headless sign-in).
