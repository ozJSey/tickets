# PUB-5 — the deploy script cannot publish to a 2FA account

**Owner, 2026-09-17: "Did you update the deploy script?"** No — the planner worked around it
instead, which is the wrong trade: the script IS the deploy mechanism, and it cannot complete a
release on the account it exists to release from.

## Measured tonight

`npm profile get` → `two-factor auth: auth-and-writes`. That mode challenges **every** CLI publish;
the npm website's "don't ask for 5 minutes" applies to the browser session, not to CLI writes.

`scripts/lib/release.mjs:94` runs a bare `npm publish`, inherits no TTY, and npm answers:

```
npm error code EOTP
npm error This operation requires a one-time password from your authenticator.
npm error You can provide a one-time password by passing --otp=<code> to the command you ran.
```

The run dies there. Six packages published tonight only because the owner ran them by hand;
`@ozjsey/vue-write-behind@0.2.0` is still unpublished because of this.

**A second defect, same root, that cost ~8 minutes tonight:** a publish attempt that fails
*after* uploading leaves the version STAGED, and every retry then gets
`E409 Cannot publish over previously staged version "0.2.0"` until npm releases it. The script
reports that as a generic failure, so the operator cannot tell "your code is wrong" from "wait,
npm is holding your last attempt."

## Work

1. **`--otp=<code>` passthrough.** Accept it on `scripts/publish.mjs` and thread it to the publish
   command. It must NOT be logged, and must not appear in any error output the script prints —
   an OTP in a terminal scrollback or CI log is a credential leak, even though it is short-lived.
2. **A code per package, not per run.** OTPs expire in ~30s and this queue publishes up to twelve
   packages, so one code cannot cover a whole run. Decide and implement one of:
   - **prompt per package** when stdin is a TTY (preferred — the operator pastes a fresh code as
     each package comes up), falling back to the flag for one-shot use; or
   - fail fast with a clear message when more than one package is queued and only `--otp` was given.
   Either is acceptable; silently letting later packages fail on an expired code is not.
3. **Classify the failures npm actually returns**, instead of one generic "publish failed":
   - `EOTP` → "this account is auth-and-writes; pass `--otp=<code>` or use an automation token"
   - `E409 … previously staged` → "your previous attempt is still staged; npm releases it after a
     few minutes — wait and re-run, do NOT bump the version"
   - `E401` / `ENEEDAUTH` → "run `npm login` first" (tonight this surfaced as a **404 on PUT**,
     which reads as "package not found" and is thoroughly misleading — say what it means)
   - `E403` → name it as a permissions/ownership problem on the scope
4. **Document the automation-token path** in the script header and `PUBLISHING.md`: a granular
   access token in `~/.npmrc` bypasses 2FA for writes and is what makes an unattended run possible.
   Do not create a token, do not touch `~/.npmrc` — that is the owner's.

## Explicitly NOT in scope

Changing the account's 2FA mode, creating tokens, or publishing anything. The script must keep
refusing to publish without an explicit `--publish`, and `ORDER` stays hard-coded.

## Acceptance — standing criteria (BOARD.md), agent at xhigh effort

1. Tests alongside the existing `scripts/publish.test.mjs` (PUB-4's structure — keep the pure-function
   split, do not regress it). With a mocked runner, assert: `--otp` reaches the publish command;
   it is absent from every logged line and every error path; each error code above produces its own
   message.
2. **Negative control:** an OTP planted in a failing publish's output must not appear in what the
   script prints. Demonstrate the test failing if the redaction is removed.
3. Verify the real `EOTP` and `E409` strings against tonight's logs in `~/.npm/_logs/` rather than
   inventing them — match on npm's error **code**, not on prose that npm may reword.
4. `node --test 'scripts/*.test.mjs'` green; report the real number.
5. Do NOT run `npm publish`, not even to test. Mock it.
