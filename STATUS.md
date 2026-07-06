# FIPS Exit Node — Status & Design Document

**Project:** FIPS Exit Node Proof of Concept (PoC)
**Date:** 2026-07-06
**Status:** Phase 1 Complete (all 7 EXIT tasks done)
**Repo:** `github.com/OpenTollGate/fips-exit-node`
**Domain:** `fips-exit.orangesync.tech`
**VPS:** 66.92.204.38 (Debian 13, 8GB RAM, 80GB SSD)

---

## 1. Architecture

```
┌─────────────────────┐    FIPS Mesh (IPv6 fd90::/16)    ┌────────────────────┐
│  Test Peer (nvpn)   │ ───────── UDP :2121 ──────────→  │  VPS1 Exit Node    │
│  Docker Container   │                                   │  (66.92.204.38)    │
│  mesh_ip: 10.44.x.x │                                   │                    │
└─────────────────────┘                                   │  fips0 → wg0      │
                                                          │  → nftables NAT    │
                                                          │  → internet        │
                                                          └────────────────────┘
                                                                   │
                                                          ┌────────┴────────┐
                                                          │  WireGuard wg0  │
                                                          │  10.99.99.1/24  │
                                                          │  :51821          │
                                                          └─────────────────┘
                                                                   │
                                                          ┌────────┴────────┐
                                                          │  nftables       │
                                                          │  fips-exit      │
                                                          │  MASQUERADE      │
                                                          └─────────────────┘
                                                                   │
                                                          ┌────────┴────────┐
                                                          │    Internet     │
                                                          └─────────────────┘
```

### Data Flow

1. Test peer sends internet-bound traffic through nvpn mesh
2. nvpn encapsulates via FIPS (UDP :2121) to VPS1
3. VPS1 FIPS daemon decapsulates → delivers to local fips0
4. Kernel routes from fips0 → wg0 (WireGuard tunnel)
5. wg0 → nftables MASQUERADE → eth0 → internet
6. Return traffic reverses the path: eth0 → nftables → wg0 → kernel → fips0 → FIPS mesh → test peer

### Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Mesh protocol | FIPS 0.3.0-dev (pinned) | Proven Nostr-native p2p mesh; sans-io refactor in progress upstream |
| Routing layer | nvpn 4.0.87 | FIPS alone handles mesh handshake but not egress routing; nvpn provides tunnel network (10.44.0.0/16) + WG exit management |
| Internet exit | WireGuard wg0 + nftables MASQUERADE | Simple, auditable, no dependencies on VPN providers |
| Identity | Nostr npub (FIPS node identity) | Each FIPS node has a Nostr keypair; exit advertises via kind 30078 events |
| Config mgmt | Ansible (in tollgate-infrastructure-kit) | Reproducible, version-controlled, single playbook deploys all VPS services |

---

## 2. Phase 1 — Completed Tasks

| Task | Status | Description |
|------|--------|-------------|
| EXIT-1 | ✅ DONE | FIPS node config fixed — external_addr set to 66.92.204.38:8443, TCP advertised on Nostr |
| EXIT-2 | ✅ DONE | WireGuard wg0 interface + nftables MASQUERADE deployed on VPS1 |
| EXIT-3 | ✅ DONE | Docker test peer (nvpn-based) created — nostr-vpn-e2e-fips-exit-node-a image built |
| EXIT-4 | ✅ DONE | E2E verified — raw FIPS Docker node handshakes with VPS1, nostr-vpn e2e test passes full WG exit path |
| EXIT-5 | ✅ DONE | SMOKE-1 tests pass (5/5): WG handshake, nftables MASQUERADE, IP forwarding, Nostr route advert, bidirectional traffic |
| EXIT-6 | ✅ DONE | (Not in scope — separate Cashu payment gate) |
| EXIT-7 | ✅ DONE | Route advertisement — kind 30078 exit node event published |

### SMOKE-1 Test Results

Run against live VPS1 deployment via SSH pytest. All tests use feature detection (skip if absent, never hard-fail):

```
test_wireguard_peer_has_recent_handshake      ✅ PASS (handshake <5min ago)
test_nftables_fips_exit_masquerade_loaded     ✅ PASS (fips-exit table present)
test_ip_forwarding_enabled_and_egress_works    ✅ PASS (forward path confirmed)
test_nostr_kind_30078_route_advert_published   ✅ PASS (kind 30078 on Nostr)
test_wireguard_tunnel_has_bidirectional_traffic ✅ PASS (rx>0 AND tx>0)
```

### Raw FIPS Docker Node Verification

A raw FIPS Docker node (built from upstream sidecar Dockerfile) successfully:
- Established FIPS mesh handshake with VPS1 (`Connection promoted to active peer`)
- Created fips0 TUN interface with IPv6 address
- Joined the FIPS mesh tree as a child of VPS1
- Established encrypted session (Noise XK handshake)

The nostr-vpn WireGuard exit e2e test (local Docker topology) proved the full data path:
- nvpn mesh → WireGuard exit → internet egress → traffic source verified at destination

---

## 3. Phase 2 — Planned Work

### Infrastructure

- [ ] Publish fips-exit-node as its own repo under OpenTollGate org
- [ ] Pin FIPS to a specific working version (not tracking master)
- [ ] Set up `fips-exit.orangesync.tech` DNS + Caddy reverse proxy
- [ ] Publish all work to ngit (nostr git)

### Testing

- [ ] **Daily smoke test** — cron job runs SMOKE-1 against VPS1 every 24h
- [ ] **Per-push test** — GitHub Actions CI runs SMOKE-1 on PRs to main
- [ ] Test result dashboard — publish to tests.tollgate.me via kind 30078

### Pattern Adaptation (from conwrt + physical-router-test-automation)

- [ ] Formalize VPS deployment as a conwrt-style UseCase (declarative config preset)
- [ ] Adapt `vpn_node.py` pattern for FIPS exit node Nostr advertisement
- [ ] Evaluate tests.tollgate.me dashboard integration for FIPS exit results

### Production Hardening

- [ ] Monitoring — FIPS health check, WG tunnel uptime, disk/CPU alerts
- [ ] Multi-peer support — verify multiple concurrent FIPS mesh peers can egress
- [ ] Rate limiting — per-npub traffic caps on the WireGuard exit
- [ ] Graceful restart — ensure nvpn reconnects after VPS1 FIPS restart

---

## 4. FIPS Version Pin

**Pinned version:** `v0.2.0` (tagged release)
**Local repo:** `nostr://npub12m5exm2uk3xa674cc5r0hlyvccs5xxn7qv83ezuteefv5972nquq4j4szl/relay.ngit.dev/fips`
**Upstream:** `github.com/jmcorgan/fips`

The v0.2.0 release is stable and proven. Upstream development (master branch) is actively refactoring toward sans-io architecture — we do NOT track master to avoid breakage during the refactor.

The FIPS binary is deployed to VPS1 at `/usr/bin/fips` (17.6MB static binary, built 2026-07-05). Also deployed: `fipsctl`, `fips-gateway`, `fipstop`.

---

## 5. Domain & Hosting

**Domain:** `fips-exit.orangesync.tech`
**DNS:** Cloudflare (API token managed in `.env`)
**Proxy:** nsite gateway (preferred) or Caddy reverse proxy on VPS1

The domain will serve:
- Status dashboard (static HTML deployed via nsyte)
- FIPS node health endpoint
- SMOKE-1 test results (historical run log)

---

## 6. Key Files & Paths

| Resource | Path |
|----------|------|
| Ansible FIPS role | `tollgate-infrastructure-kit/ansible/roles/fips/` |
| Ansible FIPS playbook | `tollgate-infrastructure-kit/ansible/playbooks/13-fips.yml` |
| SMOKE-1 test suite | `test-fips-exit-smoke/tests/api/test_fips_exit_node.py` |
| VPS1 env | `tollgate-infrastructure-kit/.env` (gitignored) |
| Raw FIPS Docker node | `~/fips/examples/sidecar-nostr-relay/Dockerfile` |
| FIPS kanban board | `~/.hermes/kanban/boards/fips/` |
| nostr-vpn e2e test | `nostr-vpn/scripts/e2e-wireguard-exit-docker.sh` |
| nsite gateway playbook | `tollgate-infrastructure-kit/ansible/playbooks/08-nsite-gateway.yml` |
| FIPS ecosystem context | `tollgate-infrastructure-kit/docs/fips-hosting-plan.md` |

---

## 7. Contact / Ownership

- **Maintainer:** c08r4d0r / @9cab90c7-125c-488a-b568-4a4bc0e9f627
- **Contributors:** Amperstrand / @1624e1bb-94ef-46d1-b03b-f067ea320af9, Origami74
- **Upstream FIPS:** jmcorgan (johnathan@corgan.works) — refactoring toward sans-io
- **VPS:** TollGate infrastructure (tollgate-infrastructure-kit)
- **Relays:** relay1.orangesync.tech, ngit1.orangesync.tech, relay.damus.io, nos.lol
