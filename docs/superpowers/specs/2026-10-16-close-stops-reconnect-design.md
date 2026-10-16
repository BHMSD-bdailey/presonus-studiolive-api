# `Client.close()` must stop the client

Date: 2026-10-16. Author: Ben Dailey, with Claude.

Repository: this fork, `~/projects/presonus-studiolive-api`
(`git@github.com:BHMSD-bdailey/presonus-studiolive-api.git`, remote `fork`) of
`@featherbear/presonus-studiolive-api`.

This is piece 1 of 2. Piece 2 lives in av-control
(`~/projects/av-control/docs/superpowers/specs/2026-10-16-av-control-demand-driven-connections-design.md`)
and depends on the tag this piece produces. Do this piece first.

## Why

av-control's mixer bridge (a long-lived Node process on `apps01`) uses this
library to hold one connection per console. Measured 2026-10-16 on production
after 5½ days of uptime: **434 TCP sockets in `SYN-SENT` to one console**
(`10.110.9.102:53000`, a desk that was powered off and does not answer ping),
growing by about one a minute. Restarting the process cleared them; they began
piling up again within a minute.

Cause, in `src/lib/Client.ts`, `connect()`:

- `doConnect()` destroys the socket, arms a 2-second `fastReconnectTimer` that
  calls `doConnect()` again, and dials.
- The timer is a local inside the promise executor. The only thing that clears
  it is a successful `'connect'`.
- `close()` sends an unsubscribe packet, then destroys the socket. It does
  nothing to the timer, so the timer re-dials 2 s later, forever.
- The `connect()` promise never rejects (`reject` is never called; socket
  errors are swallowed by `this.conn.once("error", () => {})`).
- On a socket that never connected, `close()` itself awaits `_sendPacket(...)`
  before destroying, which may never resolve.

So every `Client` that a caller gives up on against an unreachable console keeps
one socket in `SYN-SENT`, re-dialled every 2 s, with no way to stop it. Each of
the 434 sockets is one abandoned client.

## Goal

A closed `Client` is inert: no timer, no socket, no further dial, and a pending
`connect()` settles.

## Non-goals

- A connect timeout option. Callers already race `connect()` against their own
  timer; that can stay. (Builder may add one if it falls out naturally, but it
  is not required.)
- Changing reconnect behaviour of a client that has **not** been closed. The
  2-second retry against an unreachable host, and `autoreconnect` after a
  dropped session, stay as they are.
- Submitting upstream to featherbear. Optional, later, Ben's decision.

## Anti-goals

- Changing anything about a connected client's normal operation (handshake,
  subscription, keep-alive, metering, property writes). The bridge drives live
  audio with this; a regression there is worse than the leak.
- A `close()` that can hang.

## Design

Branch from `master`, e.g. `fix/close-stops-reconnect`. Not from
`fix/log-volume-taper`, which is unmerged and unrelated (the bridge uses the
linear setter).

In `src/lib/Client.ts`:

- Move `fastReconnectTimer` from the `connect()` closure to a private instance
  field.
- Add a private `closed` flag.
- `close()`:
  - sets `closed`, clears the reconnect timer, stops the `KeepAliveHelper` if
    there is one, and destroys the socket;
  - **does not wait on the unsubscribe packet** before destroying. Send the
    unsubscribe only if the socket is actually connected, best-effort, and
    destroy regardless;
  - still calls `meterUnsubscribe()` as it does today;
  - makes a still-pending `connect()` reject with a clear error (e.g.
    `'Client closed before it connected.'`) and clears `connectPromise`;
  - is idempotent and resolves promptly whether or not the client ever
    connected.
- `doConnect()`, `reconnect()` and the keep-alive's auto-reconnect callback
  return immediately when `closed` is set.
- The `'connect'` listener does nothing when `closed` is set, so a handshake
  that completes after `close()` cannot start a session.

Whether a closed client can be reconnected is the builder's call. The bridge
never reuses one (it makes a fresh client per session and per verification
read), so "closed is final" is acceptable and simplest.

## Tests

Vitest, in `tests/`. Use fake timers and either a local `net` server that
never completes the protocol handshake, or a spy on the socket's `connect`, to
prove:

- after `close()`, no further dial happens across several retry intervals;
- a pending `connect()` rejects after `close()`;
- `close()` resolves promptly on a client that never connected;
- `close()` twice is harmless.

Existing tests stay green: `npm test` (it builds first).

## Packaging

The consumer will install this as a git dependency, and `package.json` has no
`prepare` script, so a git install would have no `dist/`. Add
`"prepare": "npm run build"`, and make sure the build's tools are available to
a git install (they are `devDependencies`; npm installs those for a git
dependency's `prepare`).

Bump the version (e.g. `1.9.1-bhmsd.1`; semver ordering against upstream's
`v1.9.1` doesn't matter, because the consumer pins the exact tag), commit, tag
(e.g. `v1.9.1-bhmsd.1`), and push branch and tag to `fork`. Don't merge into the
fork's `master`; it keeps tracking upstream. Report the tag name; piece 2 pins
to it as `git+https://github.com/BHMSD-bdailey/presonus-studiolive-api.git#<tag>`
(the repo is public, and production installs over HTTPS with no GitHub key).

`KeepAliveHelper` has no stop method today; add one that clears its interval.
Branched from `master`, the existing tests are `tests/cjs.test.ts` and
`tests/esm.test.ts` (`volume.test.ts` exists only on the taper branch). For the
"close() resolves promptly" test, use real timers with a bound well under 2 s:
without the fix, a never-connected `close()` only finishes once the next retry
destroys the socket, so fake timers would hang rather than fail.

Leave the untracked `PR_BODY.md`, `TASK-logVolumeToLinear.md` and
`package-lock.json` alone — leftovers from the taper work, not part of this
change.

## Acceptance

- New tests pass and fail without the fix.
- `npm test` green, `npm run lint` clean.
- Tag pushed to `fork`.
