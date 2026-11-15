# Edge Authentication Architecture for Self-Hosted Huly

Status: design notes. Nothing here has been built or tested. Written 2026-11-15.
Supersedes the verdict in the earlier findings file; adds the reviewed "zero trust blueprint" outcome.

## 1. Goal

Remote users reach a self-hosted Huly stack without a VPN. A client credential (certificate, or a pasted token) is enforced **before** any packet reaches Huly. Target hosts: Unraid and Proxmox. Real-time media (LiveKit) is carried separately because it is UDP.

## 2. Review of the earlier "zero-trust blueprint"

The blueprint was written for a different context (a custom client with an app-hosted CA). Reviewed against this stack:

| Item | Verdict |
|---|---|
| Two planes: TCP control behind edge auth, UDP media protected by short-lived signed LiveKit tokens | Keep. Matches what the repo and LiveKit do. |
| Electron `select-client-certificate` callback returning `{ data: pfx, password }` | Does not work. The callback takes a `Certificate` chosen from `certificateList` ([Electron app docs](https://electronjs.org/docs/latest/api/app), [Certificate object](https://electronjs.org/docs/latest/api/structures/certificate)). Desktop certificate use means the OS store, or header-based auth. |
| CA built into the Huly backend with an admin GUI | Reject. Online CA key on the app server, new feature across account service, UI and DB, and the sample code has weak serials, 3DES PKCS#12, no client-auth usage extension, and a stubbed CRL. |
| Caddy `trusted_ca_cert_file` / `trusted_ca_crl_file` | Syntax is outdated or unverified. Use `trust_pool` and check the current [Caddy docs](https://caddyserver.com/docs/caddyfile/directives/tls); revocation needs the revocation module or a plugin. |
| Traefik `crlFiles` | Not in Traefik's [TLS options](https://doc.traefik.io/traefik/reference/routing-configuration/http/tls/tls-options/). Traefik does not check CRLs natively; a plugin exists. |
| LiveKit "UDP 7881", `port_range: min/max`, port 49152 | Wrong. 7881 is TCP, 7882 is UDP, keys are `rtc.port_range_start/end`, and the container port must equal the port playit assigns. |
| playit container with `network_mode: host` and `49152:49152/udp` published | Rejects the "nothing on the LAN" goal. Keep the agent on the internal Docker network. |
| Hand out `.pfx` files on USB keys | Contradicts the low-friction goal. |

## 3. Design principles (apply to every option)

1. The app never holds CA keys and never issues certificates.
2. A root CA is generated once, kept offline, and signs one intermediate.
3. The online CA is a separate, small service (or Cloudflare itself), not part of Huly.
4. Leaf certificates carry client-authentication usage, random unique serials, and a bounded lifetime.
5. Client private keys are generated on the client when possible; otherwise the issuer hands over a password-protected PKCS#12 once.
6. Revocation is either (a) short lifetimes plus blocked renewal, or (b) an explicit CRL/policy change. Pick one per option and document it.
7. Nothing is published on the LAN. Huly's backend network is `internal: true`; only the tunnel agents have outbound access.

## 4. Option A: Cloudflare handles it (recommended first)

Cloudflare terminates TLS, so it must be the one validating client certificates. The origin (nginx/Caddy) never sees them.

```
Browser/PWA --(HTTPS + client cert)--> Cloudflare edge (mTLS policy)
      --> Cloudflare Tunnel --> cloudflared container --> nginx --> Huly
Browser/App --(UDP/TCP media)--> playit.gg --> playit agent --> LiveKit
```

Steps (dashboard):

1. Create the tunnel; route `huly.example.com` to `http://nginx:80`, and `livekit.example.com` to `http://livekit:7880`.
2. Choose where certificates come from:
   - **Cloudflare-issued:** generate client certificates in Cloudflare's PKI ([client certificates](https://developers.cloudflare.com/ssl/client-certificates/)), enable mTLS for the hostnames. No CA to run.
   - **Bring your own CA:** upload the CA to Cloudflare and associate the hostnames ([BYO CA](https://developers.cloudflare.com/ssl/client-certificates/byo-ca/)). If clients are signed by an intermediate, upload the whole chain ([Access mTLS](https://developers.cloudflare.com/cloudflare-one/access-controls/service-credentials/mutual-tls-authentication/)).
3. Enforce: either a WAF custom rule blocking when `cf.tls_client_auth.cert_verified` is false, or an Access policy with the **Valid Certificate** or **Common Name** selector. Use action **Service Auth** for clients that cannot do an IdP login.
4. Browser/PWA: import the `.p12` once per device.
5. Desktop app options: (a) install the same certificate in the OS store (Electron uses the first store certificate by default), or (b) the header patch (Cloudflare service token in `CF-Access-Client-Id` / `CF-Access-Client-Secret`, stored with `safeStorage`).

Revocation: Cloudflare-issued certificates are revoked in the dashboard. For BYO CA, change the policy (for example, restrict by Common Name) or use short lifetimes; verify Cloudflare's CRL behavior before relying on it.

Pros: no inbound ports, no CA to run, works on Unraid and Proxmox identically. Cons: Cloudflare sees decrypted traffic; vendor dependency; apps that cannot present a certificate need the header workaround.

## 5. Option B: Self-managed CA and Caddy or Traefik

Requires that the client certificate reaches *your* proxy. Cloudflare Tunnel cannot do that (it terminates TLS). Candidate transports:

- Port-forward 443 to the proxy (breaks "no inbound ports").
- A raw **TCP** tunnel (for example playit TCP, not its HTTPS/Caddy-managed tunnel) pointed at the proxy's port 443, so TLS passes through untouched. Untested idea; the public certificate would need DNS-01 issuance because port 80 is not reachable.

```
Client --(TLS + client cert)--> raw TCP tunnel --> Caddy :443 (verifies client cert)
      --> nginx/front --> Huly (internal network)
Offline root CA --signs--> Intermediate (step-ca container) --issues--> client certs
```

CA layout:

1. Create root and intermediate with [step-ca](https://smallstep.com/docs/step-ca/). Keep the root key offline (encrypted USB, not on the server). The intermediate key lives only in the CA container.
2. step-ca is designed around short-lived certificates and *passive revocation*: revoking blocks renewal and the certificate expires on its own ([revocation docs](https://smallstep.com/docs/step-ca/revocation/)). The open-source version can also generate lightweight CRLs ([overview](https://smallstep.com/open-source/)).
3. Issue with `step ca certificate`, export with `step certificate p12`, import on the device.

Proxy sketch (verify exact syntax against current docs):

```caddy
huly.example.com {
    tls {
        client_auth {
            mode require_and_verify
            trust_pool file { pem_file /certs/ca-chain.pem }   # check current syntax
        }
    }
    reverse_proxy nginx:80
}
```

Traefik: `clientAuth.clientAuthType: RequireAndVerifyClientCert` with `caFiles`; no native CRL, so use short lifetimes or the CRL plugin.

Pros: no third party decrypts traffic; you control revocation. Cons: you operate a CA and renewals; a raw TCP tunnel path is unproven; mobile and desktop apps still depend on OS certificate stores.

## 6. Platform layouts

**Unraid (Docker Compose Manager)**
- Dedicated `huly` share on cache. Stack in one compose file plus env file (see `Burnt-Pancakes/huly-selfhost`, `unraid/`).
- Option A: `cloudflared` + `playit` containers on an `egress` network; Huly on an `internal: true` network with a fixed LiveKit IP.
- Option B adds `caddy` and `step-ca` containers; CA data in its own share; root key never stored on the array.

**Proxmox**
- Run Docker in one LXC or VM for Huly (databases on a fast pool). Put step-ca in a *separate* small LXC with its own snapshot policy; keep the root offline.
- Option A: same containers as Unraid. Option B: Caddy/Traefik in its own LXC bridging the tunnel and the Huly network; firewall the Huly VLAN to accept only the proxy.
- Back up the CA LXC separately from Huly; losing the intermediate means re-issuing every client certificate.

## 7. Media plane (both options)

- LiveKit: `rtc.udp_port` and `rtc.tcp_port` set to the playit-assigned public ports (LiveKit advertises its local port; it has no separate external-port setting, see [livekit#4814](https://github.com/livekit/livekit/issues/4814)). `rtc.node_ip` is the tunnel's public IP.
- Order: start the playit agent, create UDP and TCP tunnels, read the assigned ports, then set the LiveKit ports and edit the tunnel origins to match.
- Media is not mTLS-protected; it relies on DTLS-SRTP and short-lived signed LiveKit tokens.

## 8. Decision

| Criterion | A: Cloudflare | B: Own CA + Caddy/Traefik |
|---|---|---|
| Setup effort | Low | High |
| Inbound ports | None | None only if the raw-TCP tunnel idea works |
| Who sees plaintext | Cloudflare | Only you |
| Revocation | Dashboard / policy | Short-lived certs or CRL plugin |
| Desktop app | OS store or header patch | OS store |
| Failure recovery | Dashboard | Re-issue certs, restore CA |

Recommendation: start with **Option A** (Cloudflare-issued certificates) to get the stack running and tested through the web client. Revisit Option B only if you need to keep plaintext away from Cloudflare.

## 9. Still unverified

- Whether the Huly web client and PWA work with the Access manifest fetch and a client certificate.
- Whether Electron accepts added headers on WebSocket handshakes.
- Whether a raw TCP tunnel preserves TLS client authentication end to end with Caddy.
- Exact Caddy `trust_pool` and revocation syntax in the version you deploy.
- Cloudflare's revocation behavior for BYO CA certificates.
- Whether any Huly mobile client can present a certificate or headers.
