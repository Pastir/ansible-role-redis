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

**Check mode** (`--check --diff`)
- Changes made at runtime do not run, but are reported as `changed` with the live value, read without changing
  anything: `would CONFIG SET <key>: <live> -> <wanted>` (sizes such as `64mb` are compared in bytes, passwords
  shown as `***`), `would SENTINEL SET <key>: <live> -> <wanted>`, `would set Sentinel announce-ip: ... then
  SENTINEL RESET`. Package holds are shown for packages that are installed.
- Not predicted: settings Redis refuses at runtime (they need a restart), and the replica bootstrap of a fresh node.

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
| `redis_bind` | default IPv4 + 127.0.0.1 | Must contain `redis_cli_host` (or bind all: `*`, `0.0.0.0`, `::`, or be empty) |
| `redis_cli_host` | `127.0.0.1` | Address the role's `redis-cli` uses to reach this node's Redis (CONFIG SET/REWRITE, replica bootstrap). See below |
| `redis_password` | `""` | `requirepass` + `masterauth` (+ Sentinel `auth-pass`) |
| `redis_config` | `protected-mode yes`, `maxmemory-policy noeviction` | Any directives. A list value gives one line per item (`save`) |
| `redis_restart_allowed` | `false` | Restart when a setting cannot be applied at runtime |
| `redis_managed_conf` | `/etc/redis/redis.ansible.conf` | |
| `redis_cluster_group` | `redis` | Hosts of one replication set |
| `redis_initial_master` | first host of the group | Master at bootstrap only |
| `redis_announce_ip` | default IPv4 | |
| `redis_sentinel_enabled` | `false` | |
| `redis_sentinel_port` / `redis_sentinel_bind` | `26379` / IPv4 + 127.0.0.1 | `redis_sentinel_bind` must contain `redis_sentinel_cli_host` |
| `redis_sentinel_cli_host` | `{{ redis_cli_host }}` | Address the role's `redis-cli` uses to reach this node's Sentinel |
| `redis_sentinel_master_name` | `mymaster` | |
| `redis_sentinel_quorum` | `2` | |
| `redis_sentinel_down_after_ms` / `_failover_timeout` / `_parallel_syncs` | `30000` / `180000` / `1` | |
| `redis_sentinel_reset_on_announce_change` / `_reset_pause` | `true` / `15` | |

### Redis not on 127.0.0.1

If `127.0.0.1:<redis_port>` is taken by something else, for example a local HAProxy frontend that forwards to the
current master, the role must not talk to it: `CONFIG SET`/`CONFIG REWRITE` would land on another node. Bind Redis
to the node's own address and point the role there:

```yaml
redis_bind: ["{{ ansible_facts['default_ipv4']['address'] }}"]
redis_cli_host: "{{ ansible_facts['default_ipv4']['address'] }}"
```

The role checks that `redis_cli_host` is in `redis_bind` (and `redis_sentinel_cli_host` in `redis_sentinel_bind`).
On a fresh install (or an upgrade) Redis is restarted once so the managed `bind` is active before the role connects.
The package's first start still uses its own `bind 127.0.0.1`, so on a new host the port on 127.0.0.1 must be free
during installation.

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
- `redis_cli_host`: the standalone node binds only its own IP while a dummy listener holds 127.0.0.1:6379 and must
  receive nothing; one Sentinel node reaches Redis and Sentinel through its own IP;
- real failover, then a role re-run that must keep the new topology;
- a real `ansible-playbook --check` with changed settings: the runtime changes are reported and none is applied;
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
