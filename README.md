# FIPS Exit Node

WireGuard + nftables exit node on VPS1 with FIPS mesh peering.

**Live:** https://fips-exit.orangesync.tech

## Architecture

FIPS mesh peers connect via UDP :2121 to VPS1. Traffic egresses through
WireGuard wg0 → nftables MASQUERADE → internet.

## Status

Phase 1 complete. See [STATUS.md](./STATUS.md) for full details.

## Quick Start

```bash
# SSH to VPS1
ssh debian@66.92.204.38

# Check FIPS status
systemctl status fips

# Check WireGuard tunnel
sudo wg show

# Check nftables MASQUERADE
sudo nft list table fips-exit

# Run smoke tests
source fips-exit.env
pytest tests/test_fips_exit_node.py -v
```

## Repo Structure

```
fips-exit-node/
├── STATUS.md           # Design doc + status
├── tests/              # SMOKE-1 pytest suite
├── ansible/            # FIPS deployment roles
├── fips-pin.txt        # Pinned FIPS version
└── dashboard/          # nsite dashboard (WIP)
```

## FIPS Version

Pinned to **v0.2.0** — see [fips-pin.txt](./fips-pin.txt).

## Related

- [tollgate-infrastructure-kit](https://github.com/OpenTollGate/tollgate-infrastructure-kit)
- [jmcorgan/fips](https://github.com/jmcorgan/fips)
- [nostr-vpn](https://github.com/c03rad0r/nostr-vpn)
