# Changelog

## 2.2.0 — 2026-11

### Added
- `redis_cli_host` (default `127.0.0.1`): address used for every `redis-cli` call to the local Redis
  (`CONFIG SET`, `CONFIG REWRITE`, replica bootstrap). Lets Redis listen only on the node's own IP when
  127.0.0.1:6379 is taken, e.g. by a local HAProxy frontend for the master.
- `redis_sentinel_cli_host` (default `redis_cli_host`): the same for Sentinel (`SENTINEL SET`, `RESET`, wait).
- Validation: `redis_cli_host` must be in `redis_bind` and `redis_sentinel_cli_host` in `redis_sentinel_bind`
  (or bind all / empty).
- Wait for Redis to accept connections on `redis_cli_host` before talking to it.

### Changed
- `redis_bind` no longer has to contain 127.0.0.1.
- After a fresh install or an upgrade Redis is restarted once, so the managed `bind` is active before the role
  connects (an empty, just-installed Redis; the package upgrade restarts it anyway).

## 2.1.0 — 2026-11

### Added
- `redis_apt_suite`: install from another Ubuntu release's archive, so 26.04 hosts can keep noble's 7.0.15. The source
  `ubuntu-<suite>.sources` gets apt priority 100, so only explicitly requested versions come from it.

## 2.0.0 — 2026-11

Rewrite.

### BREAKING
- Redis is installed from the **Ubuntu archive** with a pinned version, not `latest` from packages.redis.io.
  Removed `redis_repo_keyserver`, `redis_repo_key`, `redis_bind_interface` (now `redis_bind`, a list),
  `redis_loglevel`, `redis_logfile` (use `redis_config`).

### Added
- Configuration management through an include file, applied at runtime (`CONFIG SET` + `CONFIG REWRITE`) without
  restarts.
- Redis Sentinel: initial config, runtime settings, `announce-ip`, rolling `SENTINEL RESET` to remove stale
  sentinels.
- Bootstrap of replicas without touching an existing topology.
- Version pin/hold with an upgrade guard. Molecule test with a real failover.
