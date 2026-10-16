# wraith-names Storage Architecture

## Storage Decisions

### Per-Name Data (Persistent)
- **Name → NameEntry**: Maps name hash to name, stealth meta-address, and owner
  - **Reason**: Scales with number of registered names; must survive contract upgrades
  - **TTL**: Extended on each access (resolve, update, release)
  
- **Subdomains**: `NameEntry.parent` holds the parent's name hash and
  `NameEntry.era` ties a subdomain to one registration of that parent. Every
  top-level registration draws a random 32-bit era; a subdomain stores its
  parent's and is live only while the two match. Releasing a parent therefore
  retires all of its subdomains without iterating over them: their entries are
  no longer renewed and expire, and registering the same subdomain name again
  replaces the dead entry (and its stale reverse lookup).
  - **Why not a counter**: a counter would live in instance storage, which
    every call loads and which `register` would then rewrite; the era costs
    20 bytes per entry and no extra ledger entries.

- **Reverse Lookup (Meta-Address Hash → Name Hash)**: Maps stealth meta-address hash to name hash
  - **Reason**: Lookup data that scales with registry size; durable
  - **TTL**: Extended alongside name entry

- **Replay Protection (Message Hash → Bool)**: Prevents signature replay in `*_on_behalf` calls
  - **Reason**: Security-critical; must be permanent to prevent old signatures from being replayed
  - **TTL**: Extended indefinitely (single write, rarely accessed)

### Contract Metadata (Instance) - REMOVED
- **Previous approach**: Some code extended instance storage
  - **Issue**: Instance storage is small, bounded, and doesn't scale well; wastes ledger resources
  - **New approach**: Only persistent entries are managed; instance TTL is skipped

## Migration Path
1. All per-name data was already persistent in the original codebase
2. Replay keys must use persistent storage (security-critical)
3. Remove all instance storage extends from `extend_ttls()` and TTL maintenance
4. The keeper service (#50) updates to extend only persistent keys

## TTL Constants
- `TTL_THRESHOLD`: 17,280 ledger intervals (~1 day)
- `TTL_EXTEND_TO`: 518,400 ledger intervals (~30 days)
