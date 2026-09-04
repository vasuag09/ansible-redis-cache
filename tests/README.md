# Verification Harness

The playbook targets a real Ubuntu VM, but you can validate it locally
first — end-to-end, not just a syntax check — using a systemd-enabled
Ubuntu 24.04 Docker container as a stand-in VM. This is how the playbook
in this repo was actually run and verified during development.

## Why Docker instead of only `--syntax-check`

`--syntax-check` only catches YAML/Jinja errors. It won't tell you if
`redis-server` actually starts, if `requirepass` actually blocks
unauthenticated clients, or if a config change actually restarts the
service. Running against a real (if containerized) systemd + apt Ubuntu
environment catches all of that.

## Build and start the test VM

```bash
docker build -f tests/Dockerfile.systemd-ubuntu -t redis-verify-image tests/
docker run -d --name redis-verify-vm --privileged --cgroupns=host \
  --tmpfs /tmp --tmpfs /run --tmpfs /run/lock \
  -v /sys/fs/cgroup:/sys/fs/cgroup:rw \
  redis-verify-image

# confirm systemd actually came up as PID 1
docker exec redis-verify-vm systemctl is-system-running --wait
```

`--privileged` + the cgroup mount are required for systemd to manage
services inside the container. This is a test convenience, not something
you'd ever do for a real workload.

## Run the playbook against it

```bash
ansible-playbook -i tests/inventory.docker.ini playbook.yml \
  --vault-password-file .vault_pass \
  -e ansible_become=false
```

- `-e ansible_become=false` — `docker exec` already runs as root inside the
  container, so there's no privilege escalation step to perform (unlike a
  real SSH-based deployment where you `become: true` from a regular user).
- `tests/inventory.docker.ini` uses `ansible_connection=community.docker.docker`
  (from the `community.docker` collection) so Ansible talks to the
  container via `docker exec` — no SSH needed.

## What was actually verified this way

Every one of these was independently confirmed with `docker exec`, not
just trusted from Ansible's own task output:

| Check | Command | Result |
|---|---|---|
| Service running | `systemctl is-active redis-server` | `active` |
| Auth enforced | `redis-cli ping` with no password | `NOAUTH Authentication required.` |
| Auth works | `redis-cli -a <password> ping` | `PONG` |
| Bind address | `grep ^bind /etc/redis/redis.conf` | `127.0.0.1 <container_ip>` |
| Persistence config | `grep ^appendonly` | `no` (RDB only, as configured) |
| Memory policy | `grep maxmemory-policy` | `allkeys-lru` |
| THP disabled | `cat /sys/kernel/mm/transparent_hugepage/enabled` | `always madvise [never]` |
| Kernel tuning | `cat /proc/sys/vm/overcommit_memory` | `1` |
| Firewall | `ufw status verbose` | default deny, allow only 22/tcp and 6379/tcp from 127.0.0.1 |
| Idempotency | re-run the playbook | `changed=0` on the second run |
| Config-change handling | re-run with `-e redis_maxmemory=512mb` | handler fired, `CONFIG GET maxmemory` returned `536870912` after restart |
| Fail-fast validation | re-run with `-e redis_password=CHANGE_ME` | play aborts immediately with a clear error, before touching the system |

## Known Docker-only limitations

- **ufw inside Docker is not representative of real network isolation.**
  It did run successfully here (the container is `--privileged`), which is
  enough to confirm the *playbook's* ufw tasks are correct, but a
  container's network namespace isn't a full stand-in for a cloud VM's
  network/security-group stack. Treat the `ufw status` output above as
  "the rules Ansible intended to set were applied correctly," not as a
  penetration test.
- **openssh-server is intentionally not installed** in the test image
  (Ansible reaches the container via `docker exec`, not SSH), which is why
  the playbook allows SSH by port number (`22/tcp`) rather than ufw's
  `OpenSSH` application profile — that profile only registers itself when
  the `openssh-server` package's own files are present. Real Ubuntu
  cloud images ship `openssh-server` by default, so this only matters for
  minimal/custom images, but allowing by port sidesteps the dependency
  entirely either way.
- `redis_tune_kernel` writes to `/sys/kernel/mm/transparent_hugepage/enabled`
  and `/proc/sys/vm/overcommit_memory`, both of which happened to be
  writable in this `--privileged` container. On a real VM these are always
  writable (they're standard kernel tunables); on other container-based
  test setups without `--privileged` they may not be.

## Tearing down the test VM

```bash
docker rm -f redis-verify-vm
```

This never touches the real target inventory in `inventory.ini` — it's
an entirely separate inventory file (`tests/inventory.docker.ini`) used
only for local verification.
