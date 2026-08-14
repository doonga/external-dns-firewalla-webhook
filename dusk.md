---
dusk: v1alpha1
namespace: stout
kind: repository
name: external-dns-firewalla-webhook
title: External-DNS Firewalla Webhook
attributes:
  language: javascript
  public: true
  image: ghcr.io/theoutdoorprogrammer/external-dns-firewalla-webhook
---

An [external-dns](https://github.com/kubernetes-sigs/external-dns) webhook provider that publishes DNS records onto a Firewalla router by writing dnsmasq config files.
It is what turns a hostname annotation on a Kubernetes Service or Ingress into a name the whole network can resolve, so when it stops working, new workloads come up with no DNS.

The repository holds two halves that ship completely differently.

`src/` is the **provider**, an Express app that runs on the router itself under the router's own bundled Node at `/home/pi/firewalla/bin/node`.
`scripts/install.sh` shallow-clones this repository to `/opt/external-dns-firewalla-webhook`, installs the unit from `systemd/`, and writes a sudoers drop-in letting the `pi` user run exactly one privileged command: `systemctl restart firerouter_dns`.

`webhook-proxy/proxy.js` is the **sidecar**, the container image built by `.github/workflows/docker-build.yml` and published to GHCR, and it is what external-dns is actually pointed at.
It mints a JWT signed with `SHARED_SECRET` on every forwarded request.
That shared secret, configured identically on both ends, is the only thing authenticating the pair.

The protocol is external-dns' webhook provider API: `GET /` negotiates the domain filter, `GET /records` reads, `POST /records` applies a create/updateOld/updateNew/delete change set, and `POST /adjustendpoints` drops record types the provider cannot serve, leaving only A, TXT and CNAME.
Records land as one file per DNS name under `~/.firewalla/config/dnsmasq_local/`, with a `.txt` suffix for TXT, and any non-empty change set ends with a `firerouter_dns` restart.

## Gotchas

**A router firmware update is the quiet failure.** Everything the installer puts down lives in paths the router does not preserve (`/opt`, `/etc/systemd/system`, `/etc/sudoers.d`), while the dnsmasq files the provider already wrote live under the persisted `.firewalla` config directory. So after a firmware update the service is simply gone, every name that already existed keeps resolving exactly as before, and only *new* records silently stop being published. Nothing appears to be down; DNS just quietly stops being current. The fix is to re-run `scripts/install.sh`.

**Adding a dependency means committing `node_modules`.** The install is a git clone onto the router with no npm step, so `express` and `jsonwebtoken` are vendored on purpose, 745 of the repository's 770 tracked files, and `.gitignore` carries a note saying so.

**The content type must be exactly `application/external.dns.webhook+json;version=1`.** That is why the handlers call `res.send(JSON.stringify(...))` rather than `res.json()`: Express would append `; charset=utf-8` and external-dns then rejects the response. The proxy echoes the request's `Accept` header back as `Content-Type` for the same reason.

**Negotiation is unauthenticated.** The auth middleware is mounted only on `/records` and `/adjustendpoints`, so `GET /` answers anyone who asks. Health is a separate Express app on its own port serving only `/healthz`.
