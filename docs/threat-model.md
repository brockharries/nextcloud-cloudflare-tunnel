# Threat Model: What This Design Defends Against (and What It Doesn't)

A short, honest threat model. The value of a security architecture is knowing its
boundaries, not pretending it has none.

## Assets
- **User data** in Nextcloud (files, calendars, contacts): confidentiality, integrity, availability.
- **The home network** behind the service, which must not become reachable just because the service is.
- **The origin's IP address**, which should not be publicly associated with a named service.

## What this design defends against well

| Threat | How it's mitigated |
|---|---|
| Internet background scanning / mass exploitation of the origin | No inbound ports are open for this service; a scan finds no web-facing listener. (The network's one WAN exposure, a two-port realtime-media forward to an isolated DMZ host, is documented in the vlan repo.) |
| Origin IP exposure / targeted attack on the residence's connection | Public DNS resolves to Cloudflare anycast IPs; the origin IP is never published. |
| Volumetric DDoS against the service | Absorbed at Cloudflare's edge before it can reach the home uplink. |
| Common web attacks (injection, known CVE probes) | Partially. Cloudflare's edge offers rate limiting and WAF rules (coverage depends on plan and configuration), which cuts down the noise. Requests that pass still reach Nextcloud, so this reduces exposure rather than replacing app patching; see residual risk below. |
| Expired-certificate outages | TLS terminates at Cloudflare with automatic renewal; no manual cert lifecycle. |
| A firewall rule silently failing "open" | There are no inbound rules to misconfigure; the safe state is the default state. |

## What it does NOT defend against (residual risk)

- **Cloudflare is in the trust path.** At the TLS-terminating edge, Cloudflare can see plaintext traffic to the proxied hostname. This is an accepted trade for a personal workload; a "no third party in the path" requirement would rule this design out.
- **Application-layer compromise of Nextcloud itself.** If Nextcloud has an auth bypass or RCE, the tunnel faithfully delivers the attacker's request to it. The edge reduces *exposure*, it does not patch the app. Mitigations: keep Nextcloud updated, enforce strong/MFA auth, and (at scale) gate the hostname behind Cloudflare Access so unauthenticated traffic never reaches the login page.
- **Connector compromise / lateral movement.** A compromised `cloudflared` host could pivot on its network segment. Mitigated by VLAN segmentation + firewall rules between segments, and (at scale) network policy limiting the connector to only the one service it fronts.
- **A single connector is an availability SPOF.** One `cloudflared` instance means a restart drops the public path. Mitigated at scale with multiple connector replicas.
- **Cloudflare account compromise.** Whoever controls the Cloudflare account controls the tunnel and DNS. Protect it with a strong password + hardware MFA; it is now part of the attack surface.

## Separate, non-public admin path
Full administrative access (SSH, hypervisor UI) is not exposed through the public tunnel at all. It runs over a **Tailscale** mesh VPN with identity-based access. The public tunnel only ever knows about the single Nextcloud hostname. This keeps the blast radius of the public entry point to exactly one application.
