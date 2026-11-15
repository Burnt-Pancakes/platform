# Edge authentication and desktop build: findings

Status: research notes, nothing here has been built or tested yet. Written 2026-11-14.

## Goal

Run the self-hosted stack behind Cloudflare Tunnel with client-side authentication at the edge (mTLS or header-based), so remote users need no VPN. First milestone: confirm the Docker stack comes up and works through the **web** client. The desktop client patch comes later.

## Context

- The hosted service was shut down. The original `hcengineering/platform` is frozen according to a maintainer in hcengineering/platform#11033. This fork tracks `Platform-Collective/platform`.
- Deployment files live in `Burnt-Pancakes/huly-selfhost` under `unraid/`.

## Findings: server side

- Code search found no mTLS or client-certificate code in this repo or in upstream.
- Services use plain `http` and `ws` internally (for example, `pods/server/src/server_http.ts` uses `http.createServer`). Service-to-service auth relies on a shared secret and JWTs.
- Absence of a hit is not proof; GitHub code search can miss content in new forks. Closed-source hosted code, if any, is not visible.
- Native mTLS between services would touch 30+ services and every internal HTTP client. Not planned.

## Findings: desktop client (`desktop/src/main/start.ts`)

- The window loads a bundled `index.html` from disk; network calls come from Chromium in the renderer.
- Server URL sources: `--server` flag, saved settings, or `config.json` in the user data folder (`PackedConfig` has `server` and `updatesChannelKey`).
- One persistent session partition (`persist:huly`) is used for all windows.
- No handler for `select-client-certificate`, `certificate-error` or `onBeforeSendHeaders` exists in `desktop/src`. Electron's default is the first certificate in the OS store.
- Updates: defaults to `https://dist.huly.io`, overridable with `DESKTOP_UPDATES_URL`. The update dialog says an update is required to continue.

## Options for client auth at the edge

| Option | Client effort | Code change | Catch |
|---|---|---|---|
| Browser + Cloudflare client certificate | Import certificate once per device | None | Mobile import is clumsy; LiveKit hostname prompts again |
| Desktop + OS-store certificate | Import into OS | None | Breaks with multiple certificates |
| Desktop + service-token headers (preferred) | Paste two values into `config.json` | About 30 lines in `start.ts` and `config.ts` | Shared secret, not a certificate |

Header approach: `session.webRequest.onBeforeSendHeaders` with a URL filter for the front and LiveKit hosts, adding `CF-Access-Client-Id` and `CF-Access-Client-Secret`. Untested: whether Electron lets us add new header names on WebSocket handshakes. Test before relying on it.

Sketch (not built):

```ts
const edge = packedConfig?.edgeHeaders
if (edge !== undefined) {
  const filter = { urls: [`${new URL(FRONT_URL).origin}/*`] }
  for (const s of [session.defaultSession, session.fromPartition(sessionPartition)]) {
    s.webRequest.onBeforeSendHeaders(filter, (d, cb) => cb({ requestHeaders: { ...d.requestHeaders, ...edge } }))
  }
}
```

`PackedConfig` also needs `edgeHeaders?: Record<string, string>`.

## Findings: desktop build

- Rush + pnpm monorepo. `desktop-package` uses electron-builder 26 and Electron 43.
- Build steps: `rush package --to desktop`, then electron-builder in `desktop-package`. Version comes from the git tag and model version.
- CI: `dist-build` in `.github/workflows/main.yml` runs on `v*` or `s*` tags on a macOS runner and needs only the `build` job. It produces Linux x64, Windows x64/arm64, macOS x64/arm64.
- macOS signing uses Apple secrets; Windows has `verifyUpdateCodeSignature: false` and no visible signing config.
- Publishing: upload to a Cloudflare R2 bucket (`desktop-distro`) using the upstream project's secrets.
- `build.mac/win/linux.publish.url` in `desktop-package/package.json` are all `https://dist.huly.io`.
- Not yet read: `desktop-package/scripts`, `desktop-package/config`, `.nvmrc`.

## Risks

- Unsigned macOS builds hit Gatekeeper; macOS auto-update likely needs a signed app (unverified). Windows shows SmartScreen warnings.
- The update channel is derived from server config (`DESKTOP_CHANNEL`, for example `0.7.426`). Each server version bump may require a matching desktop release. Channel file naming not confirmed.
- If the update host sits behind the same edge auth, the updater needs the headers too; unverified whether `electron-updater` uses the hooked session. Safer: a separate public update host.

## Plan

1. **Web first:** bring up the Docker stack (`Burnt-Pancakes/huly-selfhost`, `unraid/`) behind Cloudflare Tunnel, verify login, workspaces, file upload, collaboration, and calls in a browser.
2. Add Cloudflare Access (client certificate or service token) in front of the hostnames and re-test in the browser.
3. Branch `edge-auth-desktop`: apply the header patch and `edgeHeaders` config field.
4. Change the three `publish.url` values; add a slim `workflow_dispatch` desktop workflow without signing or R2 steps. Linux and Windows first, macOS later.
5. Test one build against the tunnel before wider rollout.

## Open questions

- Does Electron allow adding new headers (not just editing) on WebSocket upgrades?
- Does the front serve `DESKTOP_UPDATES_URL` from its config, and how is the channel file named?
- Does the updater honor the session header hook?
- Is there a mobile client, and can it send a certificate or headers? Not inspected.
