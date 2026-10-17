# HMI fleet service code

A small offline webpage for computing the current access code for a specific HMI panel, without needing a
separate authenticator-app entry per ship. See `index.html` -- open it (locally or via GitHub Pages), enter
the master secret once (saved only in that browser's local storage, never transmitted anywhere), scan or type
a device's ID, and it shows that device's current 6-digit code, refreshing every 30 seconds.

## Why this exists

Each HMI panel generates its own random settings-PIN secret on first boot, so a technician's day-to-day
access (regular authenticator app, one entry per ship they work on) is already per-device and individually
revocable. This tool is for **a different, smaller group**: staff who legitimately need access across the
whole fleet, without carrying one authenticator-app entry per ship. Instead of a separate secret per device,
they hold **one master secret**, and this tool derives each device's actual code on demand:

```
device_secret = HMAC-SHA256(master_secret, device_id)
code          = TOTP(device_secret, algorithm=SHA-256, digits=6, period=30s)
```

`device_id` is shown on the HMI's own lock/PIN screen (as both text and a QR code). The derivation is
one-way: knowing one device's derived secret (or its code) never reveals the master secret or any other
device's secret.

## Granting access

Share the master secret with the person **out-of-band** -- in person, not over chat/email, and never commit
it anywhere. They open this page once, paste it into "Master secret" and hit Save; after that it's usable
offline, same as any authenticator app.

## Revoking access

Because it's one shared master secret for this whole access tier, revoking one person's access means
**rotating the master secret**:

1. Generate a new master secret.
2. Update every HMI in the fleet to accept the new one (and stop accepting the old one) -- this is a real
   fleet-wide maintenance operation, the tradeoff of a master-key model; it's not instant like removing one
   person's own secret would be.
3. Give the new master secret to everyone who should still have access; the person who left keeps the old
   (now useless) one.

Day-to-day single-ship technicians are unaffected either way -- their own per-device secrets aren't touched
by a master-secret rotation.

## Status

- This tool: done, and self-contained (works fully offline once loaded; no server, no network calls at
  runtime).
- **Panel-side support (showing the device ID / QR on the lock screen, and accepting codes derived from a
  master secret) is not implemented yet** -- this page alone doesn't do anything useful against a real panel
  until that's wired up. See the `exor-hmi-os` repo for that work.

## Security notes

- This page itself is public (static GitHub Pages site) -- that's fine, it's just a generic calculator with
  no embedded secret. All the actual protection comes from who holds the master secret, not from hiding this
  tool's existence.
- The master secret lives only in the browser's `localStorage` on whatever device it was entered on. Clearing
  it (the "Clear & forget" button, or clearing site data) removes it from that device; it does **not**
  revoke it fleet-wide -- see "Revoking access" above for that.
