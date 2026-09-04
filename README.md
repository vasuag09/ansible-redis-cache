# Redis Cache — Ansible Playbook (Ubuntu)

Installs, configures, and hardens Redis on an Ubuntu VM: password-protected,
bound to loopback + the VM's private IP, RDB persistence, memory eviction
policy, kernel tuning (`vm.overcommit_memory`, THP disabled), and a `ufw`
firewall rule scoping access to a chosen source.

This playbook runs **locally on the target VM** (`ansible_connection=local`
in `inventory.ini`) — no SSH setup required. Run it *from inside* the Ubuntu
VM you want to configure.

## Layout

```
ansible.cfg
inventory.ini                  # localhost, ansible_connection=local
playbook.yml
requirements.yml               # community.general collection (ufw module)
group_vars/all/vars.yml        # non-secret config (port, memory, CIDR, ...)
group_vars/all/vault.yml       # YOU CREATE THIS — vault-encrypted redis_password
templates/redis.conf.j2
templates/99-redis.conf.j2     # sysctl drop-in
templates/disable-thp.service.j2
```

## Prerequisites (on the Ubuntu VM)

```bash
sudo apt update
sudo apt install -y ansible-core python3-pip
ansible-galaxy collection install -r requirements.yml
```

## 1. Set the Redis password (once)

```bash
ansible-vault create group_vars/all/vault.yml
```

You'll be prompted for a **vault password** — remember it, you'll need it
every run (or store it in a local file, see below). Inside the editor that
opens, add exactly one line:

```yaml
redis_password: "paste a strong password here"
```

Generate one if you need it: `openssl rand -base64 24`

Save and exit. `group_vars/all/vault.yml` is now encrypted on disk and safe
to commit to git — Ansible decrypts it in memory at run time only.

To edit it later: `ansible-vault edit group_vars/all/vault.yml`

## 2. Review non-secret settings

Open `group_vars/all/vars.yml` and set, at minimum:

- `redis_allowed_cidr` — who besides `127.0.0.1` may reach Redis (defaults
  to `127.0.0.1/32`, i.e. nobody remote, until you change it). Point this
  at your application server's IP/subnet, e.g. `10.0.1.15/32`.
- `redis_maxmemory` / `redis_maxmemory_policy` — sized for the VM's RAM.

## 3. Run it

```bash
ansible-playbook playbook.yml --ask-vault-pass
```

(Or avoid the prompt each time: `echo 'yourvaultpw' > .vault_pass && chmod 600 .vault_pass`
then run with `--vault-password-file .vault_pass`. `.vault_pass` is already
in `.gitignore` — never commit it.)

The playbook is idempotent — re-running it after the first successful run
should report no changes (aside from restarting Redis if you edited the
config template or vars).

## 4. Verify manually

```bash
redis-cli -h 127.0.0.1 -p 6379 -a 'yourpassword' ping   # -> PONG
sudo systemctl status redis-server
sudo ufw status verbose
```

From another host on the allowed subnet:

```bash
redis-cli -h <VM_PRIVATE_IP> -p 6379 -a 'yourpassword' ping
```

## What it does, step by step

1. Asserts the target is Ubuntu 20.04+ and that `redis_password` is actually set.
2. Installs `redis-server` and `ufw` via apt.
3. Templates `/etc/redis/redis.conf`: auth (`requirepass`), bind addresses,
   RDB save points, `maxmemory` + eviction policy, systemd supervision.
4. Enables and starts the `redis-server` service; config changes trigger a
   restart via an Ansible handler.
5. Applies Redis's own recommended kernel tuning: `vm.overcommit_memory=1`
   (persisted in `/etc/sysctl.d/99-redis.conf`) and disables Transparent
   Huge Pages via a small systemd oneshot unit that runs before Redis starts
   on every boot.
6. Configures `ufw`: keeps SSH open (so you don't lock yourself out), allows
   the Redis port only from `redis_allowed_cidr`, denies everything else by
   default.
7. Waits for the port to come up and runs an authenticated `PING` to confirm
   the whole stack actually works, failing the play if it doesn't.

## Local verification (no cloud VM needed)

This playbook has been fully run and verified end-to-end against a real
systemd-enabled Ubuntu 24.04 container (not just syntax-checked). See
`tests/README.md` for how to reproduce that: build the container, point
Ansible at it instead of the real VM, and confirm auth, persistence,
firewall, and kernel tuning all actually took effect.

See `PROJECT_REPORT.md` for the full write-up (architecture, design
rationale, and verification evidence) — that's the document to walk your
faculty through.

## Notes for grading / write-up

- Secrets management: password lives only in an `ansible-vault`-encrypted
  file, never in plaintext vars or the config template source.
- Idempotency: every task uses Ansible modules (not raw shell) except the
  two `command`/`sh` invocations, which are guarded with `changed_when`/
  `register` so re-runs don't report spurious changes.
- Defense in depth: application-layer auth (`requirepass`) *and*
  network-layer restriction (`ufw` + bind address), not just one or the other.
