# pastir.redis

Redis server and (optionally) Redis Sentinel from the Ubuntu archive, with a pinned and held version.
Configuration changes are applied **without restarting Redis**.

## How configuration is managed

Redis and Sentinel rewrite their own config files: `CONFIG REWRITE` writes into `redis.conf`, and Sentinel keeps
epochs and known replicas/sentinels in `sentinel.conf`. The role does not fight over these files.

**Redis server**
- Settings go to `/etc/redis/redis.ansible.conf`, which is included at the **end** of `redis.conf`, so it wins.
- When the file changes, every setting is applied with `CONFIG SET` and saved with `CONFIG REWRITE`.
  `redis.conf` then holds the same values, so lines Redis appends there never shadow the managed file.
- If a setting cannot be changed at runtime, the role reports it. It restarts Redis only with
  `redis_restart_allowed: true`.
- Replication topology (`replicaof`) is never forced on a running set. Only an **empty, stand-alone** node is
  pointed at `redis_initial_master` at bootstrap. After a failover the role leaves the topology to Sentinel.

**Sentinel**
- `sentinel.conf` is written only while Sentinel monitors nothing yet. Afterwards the role uses
  `SENTINEL CONFIG SET` and `SENTINEL SET` (quorum, timeouts, auth-pass, announce-ip), and Sentinel persists them.
- `announce-ip` is always set. If it changes (or was never set), the role runs `SENTINEL RESET` one sentinel at a
  time. This removes stale entries such as a sentinel registered as `127.0.0.1`, which otherwise inflates the
  number of known sentinels and can block failover.

## Versions

| Release | Default `redis_version` |
|---|---|
| noble (24.04) | 7.0.15 |
| resolute (26.04) | 8.0.5 (7.0.15 with `redis_apt_suite: noble`) |

Do not mix major versions within one replication set: a Redis 7 replica cannot follow a Redis 8 master
(newer RDB format). A version change requires `-e redis_allow_upgrade=true`. Upgrade replicas first.

## Variables

| Variable | Default | Description |
|---|---|---|
| `redis_version` | per release | Exact upstream version |
| `redis_hold` / `redis_allow_upgrade` | `true` / `false` | |
| `redis_apt_suite` | `""` | Install from another Ubuntu release's archive, e.g. `noble` on 26.04 to keep 7.0.15. The default version follows this suite |
| `redis_apt_archive_url` | `http://archive.ubuntu.com/ubuntu` | Archive for `redis_apt_suite` |
| `redis_port` | `6379` | |
| `redis_bind` | default IPv4 + 127.0.0.1 | Must contain 127.0.0.1: the role talks to Redis locally |
| `redis_password` | `""` | `requirepass` + `masterauth` (+ Sentinel `auth-pass`) |
| `redis_config` | `protected-mode yes`, `maxmemory-policy noeviction` | Any directives. A list value gives one line per item (`save`) |
| `redis_restart_allowed` | `false` | Restart when a setting cannot be applied at runtime |
| `redis_managed_conf` | `/etc/redis/redis.ansible.conf` | |
| `redis_cluster_group` | `redis` | Hosts of one replication set |
| `redis_initial_master` | first host of the group | Master at bootstrap only |
| `redis_announce_ip` | default IPv4 | |
| `redis_sentinel_enabled` | `false` | |
| `redis_sentinel_port` / `redis_sentinel_bind` | `26379` / IPv4 + 127.0.0.1 | |
| `redis_sentinel_master_name` | `mymaster` | |
| `redis_sentinel_quorum` | `2` | |
| `redis_sentinel_down_after_ms` / `_failover_timeout` / `_parallel_syncs` | `30000` / `180000` / `1` | |
| `redis_sentinel_reset_on_announce_change` / `_reset_pause` | `true` / `15` | |

## Example

```yaml
- hosts: redis
  become: true
  roles:
    - role: pastir.redis
      vars:
        redis_version: "7.0.15"
        redis_sentinel_enabled: true
        redis_password: "{{ vault_redis_password }}"
        redis_config:
          protected-mode: "yes"
          maxmemory: 2gb
          maxmemory-policy: allkeys-lru
          save: ["3600 1", "300 100", "60 10000"]
```

### Taking over a running set

The first run adds the `include` line and applies the managed settings at runtime. Nothing is restarted, and
`replicaof` is left as it is. For Sentinel, the first run sets `announce-ip`, then resets the sentinels one by one.

## Testing

`molecule test`:
- a Sentinel set of two 24.04 nodes and one 26.04 node, all on 7.0.15 (`redis_apt_suite: noble`), and a standalone 26.04 node on 8.0.5;
- runtime change without restart, and persistence after a restart;
- real failover, then a role re-run that must keep the new topology;
- no phantom sentinels.

## Installation

```yaml
# requirements.yml
roles:
  - name: pastir.redis
    src: git+git@github.com:Pastir/ansible-role-redis.git
    version: main  # or a tag
```

```bash
ansible-galaxy install -r requirements.yml
```

## License

Apache-2.0
