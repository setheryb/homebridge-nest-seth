# CLAUDE.md

Guidance for Claude Code when working in this repo.

## What this is

`homebridge-nest-seth` — a personal fork of the abandoned [`homebridge-nest`](https://github.com/chrisjshull/homebridge-nest) plugin (`upstream` remote), being revived to run under **Homebridge 2.x / HAP v2**. The `package.json` `name` is still `homebridge-nest` and the version uses a `4.6.9-seth.N` suffix scheme for fork builds.

- `origin` → `setheryb/homebridge-nest-seth` (this fork; push here)
- `upstream` → `chrisjshull/homebridge-nest` (original; do not push)
- Active work branch for the HAP v2 revival: `hap2-fix`

Plain JavaScript, **no build step**. Edit `.js` files directly.

## Layout

- `index.js` — plugin entry. Registers the `Nest` platform; builds `exportedTypes` (Service, Characteristic, **hap**, uuid) from `homebridge.hap` and injects it into the accessory modules.
- `lib/nest-device-accessory.js` — base accessory + dependency-injection factory. Holds `bindCharacteristic`/`updateData`; receives and re-exports the hap types (incl. `Formats`/`Perms`) to the child accessory modules.
- `lib/nest-*-accessory.js` — per-device types: `thermostat`, `protect`, `lock`, `homeaway`, `tempsensor`. Each calls `require('./nest-device-accessory')()` (no args) to get the already-injected types.
- `lib/nest-connection.js`, `lib/nest-endpoints.js`, `login.js`, `lib/protobuf/` — Nest API auth/transport.
- `config.schema.json` — Homebridge UI config schema.

## Conventions

- Lint with the repo config before committing: `npm run lint` (ESLint `eslint:recommended`, 4-space indent, single quotes, semicolons required). `eslint` is `devDependencies` only — if `node_modules` is absent, `npm install --no-save eslint@^5.16.0` then `npx eslint <files>`.
- Bump `package.json` version to the next `4.6.9-seth.N` on every change deployed to the NAS (the NAS reinstalls by branch and needs a new version to pick up the build).
- Commit messages: end with the `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>` trailer. Only commit/push when asked.

## HAP v2 gotchas (the revival is about these)

- **Never `require('@homebridge/hap-nodejs')` directly.** On the deploy target it lives nested at `homebridge/node_modules/@homebridge/hap-nodejs` and is **not resolvable** from a sibling plugin dir — it throws `Cannot find module` at runtime. Get hap types from the injected `homebridge.hap` / `exportedTypes.hap` flow instead.
- `Formats` and `Perms` moved off `Characteristic` to the hap top level (`hap.Formats`, `hap.Perms`). They're captured in `nest-device-accessory.js` and re-exported via the factory; child modules destructure them from `require('./nest-device-accessory')()`.
- `Characteristic.prototype.getValue()` was **removed**. Use `updateValue()` (see the `updateData` refresh loop, which stores the bound `getFunc` per characteristic).
- Other v2 changes already handled: dropped `_associatedHAPAccessory`, guard `setPrimaryService` (absent on the HAP v2 `Accessory`).

## Deploying to the Synology NAS

Homebridge lives at `/volume1/homebridge` (plugins under `node_modules/`). Reinstall a branch build:

```bash
sudo -u homebridge npm install setheryb/homebridge-nest-seth#hap2-fix
# then restart Homebridge
```

**Install as the `homebridge` user, never root** — a root install caused a `root:root` ownership lockout. If forced to use root, follow with `chown -R homebridge:homebridge` on the affected path.

Verify in the child-bridge log: `Child bridge started successfully (plugin v…)` plus clean `initing …` lines per accessory, with no `ERROR INITIALIZING PLUGIN` block or `TypeError`. Init completing ≠ Nest API healthy — confirm devices actually respond in the Home app (a "No Response" / "couldn't connect to the Nest service" issue is the auth/connectivity path, separate from the HAP v2 fixes).
