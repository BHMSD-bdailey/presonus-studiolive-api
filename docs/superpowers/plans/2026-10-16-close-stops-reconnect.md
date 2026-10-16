# Client.close() Stops the Client — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A closed `Client` is inert — no timer, no socket, no further dial — and a pending `connect()` rejects; then ship it as tag `v1.9.1-bhmsd.1` on the `fork` remote.

**Architecture:** Lift the reconnect timer and the `connect()` promise's `reject` out of the executor closure into private instance fields, add a `closed` flag that every reconnect path checks, and make `close()` synchronous at its core (clear timer, stop keep-alive, best-effort unsubscribe, destroy, reject pending connect). `KeepAliveHelper` gains `stop()`.

**Tech Stack:** TypeScript, Node `net`, Vitest 4 (`tests/**/*.test.ts`), Biome lint, tsdown build.

**Spec:** `docs/superpowers/specs/2026-10-16-close-stops-reconnect-design.md`

## Global Constraints

- Work on branch `fix/close-stops-reconnect` (already checked out, branched from `master`). Do not merge into `master`.
- Do not touch the untracked `PR_BODY.md`, `TASK-logVolumeToLinear.md`, `package-lock.json`. Never `git add -A` / `git add .`.
- No behaviour change for a client that has **not** been closed: the 2 s retry against an unreachable host and `autoreconnect` stay as they are. Handshake, subscription, keep-alive, metering, property writes unchanged.
- `close()` must never hang and must be idempotent. Closed is final: `connect()` on a closed client rejects.
- Error messages (exact): pending `connect()` rejected by `close()` → `"Client closed before it connected."`; `connect()` called after `close()` → `"Client is closed."`.
- Package version `1.9.1-bhmsd.1`, git tag `v1.9.1-bhmsd.1`, pushed to remote `fork` along with the branch.
- Acceptance: new tests fail without the fix and pass with it; `npm test` green; `npm run lint` clean.
- "Lint clean" means `npm run lint` exits 0 with no *new* diagnostics. `master` already has warnings (e.g. `noUnusedFunctionParameters` in Client.ts, `useNodejsImportProtocol` in DataClient.ts) — do not fix unrelated code.
- Unchanged: `close()` does not emit `"closed"` (only the keep-alive failure path does). Do not add it.

## Review Focus

1. **Connected client closed** — `close()` on a socket that is actually open must still deliver the unsubscribe packet to the console before the socket goes away. (Task 2, test "sends unsubscribe when connected".)
2. **Handshake completes after close** — a late `'connect'` event, or a handshake `Promise.all` resolving after `close()`, must not send the subscribe packet, start keep-alive or emit `connected`. (Task 2, tests "late connect event after close does nothing" and "in-flight handshake after close does not start a session".)
3. **Not-closed client still retries** — removing the leak must not remove the 2 s retry for a client nobody closed. (Task 2, test "keeps retrying while not closed".)
4. **`close()` before `connect()` / `connect()` after `close()`** — both are plausible caller orders; neither may throw synchronously, hang, or dial. (Task 2, tests "close before connect" and "connect after close rejects".)
5. **Keep-alive after stop** — once stopped, the keep-alive interval must neither write packets nor fire its fail callback (which would trigger `autoreconnect`). (Task 1 test.)

## Testing notes (verified against the unfixed code on 2026-10-16)

- Import the source directly: `import { Client } from "../src/lib/Client";` (Vitest transpiles TS; existing tests import `dist/`, which is fine to leave).
- Stub dialling with `vi.spyOn(net.Socket.prototype, "connect").mockImplementation(function (this: any) { return this; } as any)`. With fake timers (`vi.useFakeTimers({ toFake: ["setTimeout", "clearTimeout", "setInterval", "clearInterval"] })` — do not fake `nextTick`/`setImmediate`, Node streams need them), the dial spy counts 6 dials in total within 10 s of `close()` on the unfixed client (1 initial + 5 retries).
- The sandbox refuses unroutable addresses instantly, so a real `SYN-SENT` cannot be reproduced. Simulate "write never completes" with `vi.spyOn(net.Socket.prototype, "write").mockImplementation((() => true) as any)`; the unfixed `close()` then hangs (verified: a 500 ms real-timer race times out).
- `afterEach`: `vi.useRealTimers(); vi.restoreAllMocks();`.

---

### Task 1: `KeepAliveHelper.stop()`

**Files:**
- Modify: `src/lib/util/KeepAliveHelper.ts`
- Test: `tests/keepAlive.test.ts` (create)

**Interfaces:**
- Produces: `KeepAliveHelper#stop(): void` — clears the interval; safe to call before `start()` and more than once.

- [ ] **Step 1: Write the failing test** `tests/keepAlive.test.ts`

```ts
import { it, expect, vi, afterEach } from "vitest";
import "../src/lib/util/logging";
import KeepAliveHelper from "../src/lib/util/KeepAliveHelper";

afterEach(() => vi.useRealTimers());

it("stop() halts packets and the fail callback", () => {
	vi.useFakeTimers({ toFake: ["setTimeout", "clearTimeout", "setInterval", "clearInterval", "Date"] });
	const helper = new KeepAliveHelper(3000);
	const check = vi.fn();
	const fail = vi.fn();
	helper.start(check, fail);
	vi.advanceTimersByTime(1000);
	expect(check).toHaveBeenCalledTimes(1);
	helper.stop();
	vi.advanceTimersByTime(10000);
	expect(check).toHaveBeenCalledTimes(1);
	expect(fail).not.toHaveBeenCalled();
});

it("stop() before start() and twice is harmless", () => {
	const helper = new KeepAliveHelper();
	expect(() => { helper.stop(); helper.stop(); }).not.toThrow();
});
```

(`logging` sets the global `logger` the helper uses; check the import path exists, it is `src/lib/util/logging.ts`.)

- [ ] **Step 2: Run** `npx vitest run tests/keepAlive.test.ts` — Expected: FAIL, `helper.stop is not a function`.
- [ ] **Step 3: Implement** `stop(): void` in `KeepAliveHelper` as `clearInterval(this.#loop)`.
- [ ] **Step 4: Run** `npx vitest run tests/keepAlive.test.ts` — Expected: 2 passed. Then `npm run lint` — clean.
- [ ] **Step 5: Commit**

```bash
git add src/lib/util/KeepAliveHelper.ts tests/keepAlive.test.ts
git commit -m "Add KeepAliveHelper.stop()"
```

---

### Task 2: `Client.close()` makes the client inert

**Files:**
- Modify: `src/lib/Client.ts` — fields near line 57–63, `connect()` lines 215–326, `close()` lines 329–334
- Test: `tests/close.test.ts` (create)

**Interfaces:**
- Consumes: `KeepAliveHelper#stop()` (Task 1).
- Produces (all private, no public API change except behaviour):
  - `private closed = false`
  - `private fastReconnectTimer: ReturnType<typeof setTimeout>` (replaces the local in `connect()`)
  - `private rejectConnect: (err: Error) => void` — the pending `connect()` promise's `reject`, set in the executor, cleared (`null`) when the promise settles.
  - `async close(): Promise<void>` — same signature as today.

**Required behaviour (from spec):**
- `connect()`: if `this.closed`, return `Promise.reject(new Error("Client is closed."))` (check before the `connectPromise` reuse). On successful handshake resolve, clear `rejectConnect`.
- `doConnect()`, `reconnect()`, the keep-alive fail callback, the `this.conn` `'connect'` listener, and the handshake `Promise.all(...).then(...)` each return immediately when `this.closed`.
- `close()`: return at once if already closed; otherwise set `closed`, `clearTimeout(this.fastReconnectTimer)`, `this.keepAliveHelper?.stop()`, `this.meterUnsubscribe()`; if `!this.conn.pending && !this.conn.destroyed` (do **not** use `readyState === "open"`: on Node a never-connected `net.Socket` reports `"open"`), write the unsubscribe packet (`createPacket(MessageCode.JSON, unsubscribePacket)`) without awaiting its callback; `this.conn.destroy()`; then `this.rejectConnect?.(new Error("Client closed before it connected."))`, set `rejectConnect = null` and `connectPromise = null`. Nothing in `close()` may await I/O.
- If Review Focus test 1 shows that write-then-immediate-`destroy()` drops the packet on loopback, switch to `this.conn.end(packet)` followed by `destroy()` once `'finish'` fires **or** after 250 ms, whichever comes first — still bounded. Prefer the simple form if the test passes.

- [ ] **Step 1: Write the failing tests** in `tests/close.test.ts`. Shared setup:

```ts
import { describe, it, expect, vi, afterEach } from "vitest";
import net from "node:net";
import { Client } from "../src/lib/Client";

const fakeTimers = () =>
	vi.useFakeTimers({ toFake: ["setTimeout", "clearTimeout", "setInterval", "clearInterval"] });
const stubDial = () =>
	vi.spyOn(net.Socket.prototype, "connect").mockImplementation(function (this: any) { return this; } as any);
const within = <T>(p: Promise<T>, ms: number) =>
	Promise.race([p.then(() => "settled" as const, () => "settled" as const),
		new Promise<"timeout">((r) => setTimeout(() => r("timeout"), ms))]);

afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); });
```

Tests (names and assertions; each creates `new Client({ host: "127.0.0.1", port: 1 })` unless stated):

1. `"no further dial after close()"` — `fakeTimers(); const dial = stubDial();` `connect().catch(() => {})`; `expect(dial).toHaveBeenCalledTimes(1)`; `await client.close()`; `vi.advanceTimersByTime(10_000)`; `expect(dial).toHaveBeenCalledTimes(1)`.
2. `"keeps retrying while not closed"` (Review Focus 3) — same setup, no close; `vi.advanceTimersByTime(6_000)`; `expect(dial).toHaveBeenCalledTimes(4)`. Then `await client.close()` for cleanup. (Passes before the fix too — it is a regression guard.)
3. `"pending connect() rejects after close()"` — `stubDial()`; `const p = client.connect()`; `await client.close()`; `await expect(p).rejects.toThrow("Client closed before it connected.")`.
4. `"close() resolves promptly on a never-connected client"` — real timers; `stubDial()`; `vi.spyOn(net.Socket.prototype, "write").mockImplementation((() => true) as any)`; `client.connect().catch(() => {})`; `expect(await within(client.close(), 500)).toBe("settled")`.
5. `"close() twice is harmless"` — `stubDial()`; `connect().catch(() => {})`; `await client.close(); await client.close();` resolves; `await Promise.all([c2.close(), c2.close()])` on a second client also resolves.
6. `"close before connect"` (Review Focus 4) — `const dial = stubDial()`; `await client.close()` resolves without calling `connect()`; `expect(dial).not.toHaveBeenCalled()`.
7. `"connect after close rejects"` (Review Focus 4) — `const dial = stubDial()`; `await client.close()`; `await expect(client.connect()).rejects.toThrow("Client is closed.")`; `expect(dial).not.toHaveBeenCalled()`.
8. `"late connect event after close does nothing"` (Review Focus 2) — `stubDial()`; `const write = vi.spyOn(net.Socket.prototype, "write").mockImplementation((() => true) as any)`; `connect().catch(() => {})`; `await client.close()`; `write.mockClear()`; `(client as any).conn.emit("connect")`; `expect(write).not.toHaveBeenCalled()`; `expect((client as any).keepAliveHelper).toBeUndefined()`.
9. `"sends unsubscribe when connected"` (Review Focus 1) — real timers, no stubs. Start `net.createServer` on `127.0.0.1` port `0` that collects every received chunk into a `Buffer` and records `'end'`/`'close'` of the server-side socket; never replies (handshake never completes). `const client = new Client({ host: "127.0.0.1", port })`; `const p = client.connect(); p.catch(() => {})`; wait until the server has received the first chunk (the subscribe packet). `await client.close()`. Wait (≤ 1 s) for the server socket's `'close'`. Assert the received bytes contain `createPacket(MessageCode.JSON, unsubscribePacket)` — import `createPacket` from `../src/lib/util/messageProtocol`, `unsubscribePacket` from `../src/lib/util/subscriptionUtil`, `MessageCode` from `../src/lib/constants` — via `received.includes(expected)`. Also `await expect(p).rejects.toThrow("Client closed before it connected.")`. Close the server in `finally`.
10. `"in-flight handshake after close does not start a session"` (Review Focus 2; DataClient's async queue can deliver packets after `destroy()`) — `stubDial()`; stub `write` as in test 8; `const start = vi.spyOn(KeepAliveHelper.prototype, "start")` (import `KeepAliveHelper` from `../src/lib/util/KeepAliveHelper`); `const connected = vi.fn(); client.on("connected", connected)`; `connect().catch(() => {})`; `(client as any).conn.emit("connect")`; `await client.close()`; `(client as any).emit(MessageCode.ZLIB, {})`; `(client as any).emit(MessageCode.JSON, { id: "SubscriptionReply" })`; `await new Promise((r) => setImmediate(r))`; `expect(connected).not.toHaveBeenCalled()`; `expect(start).not.toHaveBeenCalled()`.

- [ ] **Step 2: Run** `npx vitest run tests/close.test.ts` — Expected on the unfixed code: tests 1, 3, 4, 7, 8, 9, 10 FAIL (4 returns `"timeout"`; 3, 7, 8, 9 fail by hitting the 10 s test timeout — expected, not a broken setup); 2, 5, 6 pass. Vitest may also report ~2 "Unhandled Errors: Socket is closed" from the unfixed `close()` — expected. Record the result in the task report.
- [ ] **Step 3: Implement** the "Required behaviour" list above in `src/lib/Client.ts`. Leave the `console.log("conn was closed so will reconnect")` and every other line of the handshake as-is apart from the `closed` guards.
- [ ] **Step 4: Run** `npx vitest run tests/close.test.ts tests/keepAlive.test.ts` — Expected: all pass. Then `npm test` (builds, runs every test) — green; `npm run lint` — clean.
- [ ] **Step 5: Commit**

```bash
git add src/lib/Client.ts tests/close.test.ts
git commit -m "Make Client.close() stop the reconnect loop and settle connect()"
```

---

### Task 3: Package as a git dependency and tag

**Files:**
- Modify: `package.json` — `"version": "1.9.1-bhmsd.1"`, add `"prepare": "npm run build"` to `scripts`.

Edit `package.json` by hand (not `npm version`, which would rewrite the untracked `package-lock.json`).

- [ ] **Step 1: Edit** `package.json` as above.
- [ ] **Step 2: Commit**

```bash
git add package.json
git commit -m "Add prepare script for git installs; version 1.9.1-bhmsd.1"
```

- [ ] **Step 3: Verify a git install builds `dist/`.** In a fresh scratch directory: `npm init -y && npm install "git+file:///home/bdailey/projects/presonus-studiolive-api#fix/close-stops-reconnect"`. Expected: install succeeds, `node_modules/@featherbear/presonus-studiolive-api/dist/api.cjs` and `dist/types/api.d.ts` exist, and `node -e 'console.log(typeof require("@featherbear/presonus-studiolive-api").Client)'` prints `function`. Delete the scratch directory afterwards. If the network blocks the install (prepare needs devDependencies from the registry), fall back to `npm run build && npm pack --dry-run` and confirm `dist/api.cjs` is listed; say which check ran in the report. Note: a git install resolves devDependencies fresh (ignores `pnpm-lock.yaml`); mention it in the report, no change needed.

- [ ] **Step 4: Final checks** — `npm test` green, `npm run lint` clean, `git status --short` shows only the three untracked leftovers.
- [ ] **Step 5: Tag and push**

```bash
git tag -a v1.9.1-bhmsd.1 -m "Client.close() stops the reconnect loop"
git push fork fix/close-stops-reconnect
git push fork v1.9.1-bhmsd.1
```

Expected: both pushes succeed. Report the tag and the consumer spec `git+https://github.com/BHMSD-bdailey/presonus-studiolive-api.git#v1.9.1-bhmsd.1`.
