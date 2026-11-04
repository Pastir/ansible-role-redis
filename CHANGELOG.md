# Changelog

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
