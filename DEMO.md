# Demo Runbook — for your group partner

Two ways to run this. **Path A (Docker)** is what's actually recommended
for a faculty demo: it works identically on Windows/Mac/Linux, takes about
10 minutes to set up, and needs no cloud account. **Path B (real Ubuntu
VM)** is more "authentic" to a cloud computing project if you already have
a VM — use it if you want to demo Redis reachable from a second machine
on the network.

Either way, start here:

## 0. Prerequisites (both paths)

```bash
git clone https://github.com/vasuag09/ansible-redis-cache.git
cd ansible-redis-cache
```

You also need the **vault password** — ask your teammate for it directly
(Signal/WhatsApp/in person, not over email/Slack in plaintext, and never
via git). It decrypts `group_vars/all/vault.yml`, which holds the Redis
password. Save it to a local file so you don't have to type it every run:

```bash
echo -n 'the-vault-password-your-teammate-gave-you' > .vault_pass
chmod 600 .vault_pass
```

(`.vault_pass` is gitignored — it will never show up in `git status`.)

---

## Path A — Docker demo (recommended)

### A.1 Install prerequisites

- **Docker Desktop**: https://www.docker.com/products/docker-desktop/ (Windows/Mac/Linux)
- **Ansible**:
  - macOS: `brew install ansible`
  - Windows: install WSL2 first (`wsl --install` in an admin PowerShell,
    reboot), then inside the WSL2 Ubuntu shell: `sudo apt update && sudo apt install -y ansible`
  - Linux: `sudo apt update && sudo apt install -y ansible`

Start Docker Desktop and wait until it says "Docker Desktop is running."

### A.2 Build and start the test VM (a real systemd-enabled Ubuntu container)

```bash
docker build -f tests/Dockerfile.systemd-ubuntu -t redis-verify-image tests/
docker run -d --name redis-verify-vm --privileged --cgroupns=host \
  --tmpfs /tmp --tmpfs /run --tmpfs /run/lock \
  -v /sys/fs/cgroup:/sys/fs/cgroup:rw \
  redis-verify-image

# confirm it's actually up (should print "running")
docker exec redis-verify-vm systemctl is-system-running --wait
```

### A.3 Run the playbook against it

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook -i tests/inventory.docker.ini playbook.yml \
  --vault-password-file .vault_pass \
  -e ansible_become=false
```

You should see `PLAY RECAP ... failed=0` at the end, ending with:
`"Redis is up and authenticating correctly."`

### A.4 Live demo commands (run these in front of faculty)

```bash
# Redis is running
docker exec redis-verify-vm systemctl is-active redis-server

# auth is enforced — this MUST be refused
docker exec redis-verify-vm redis-cli -h 127.0.0.1 -p 6379 ping

# with the correct password it works
REDIS_PW=$(ansible-vault view group_vars/all/vault.yml --vault-password-file .vault_pass | cut -d'"' -f2)
docker exec redis-verify-vm redis-cli -h 127.0.0.1 -p 6379 -a "$REDIS_PW" --no-auth-warning ping

# firewall rules
docker exec redis-verify-vm ufw status verbose

# kernel tuning actually applied
docker exec redis-verify-vm cat /sys/kernel/mm/transparent_hugepage/enabled
docker exec redis-verify-vm cat /proc/sys/vm/overcommit_memory

# idempotency — re-run the whole playbook, point out "changed=0"
ansible-playbook -i tests/inventory.docker.ini playbook.yml \
  --vault-password-file .vault_pass -e ansible_become=false
```

Talking points for each command are in `PROJECT_REPORT.md` §9.

### A.5 Clean up after the demo

```bash
docker rm -f redis-verify-vm
```

---

## Path B — Real Ubuntu VM

Use this if you have an actual Ubuntu 20.04+ VM (cloud instance, VirtualBox,
Multipass) and want to show Redis reachable from a second machine.

### B.1 On the Ubuntu VM

```bash
sudo apt update && sudo apt install -y ansible-core git
git clone https://github.com/vasuag09/ansible-redis-cache.git
cd ansible-redis-cache
ansible-galaxy collection install -r requirements.yml
```

Get `.vault_pass` onto the VM the same way as step 0 above.

### B.2 Set the allowed source for remote access (optional)

If you want a second machine to reach Redis, edit `group_vars/all/vars.yml`
and set `redis_allowed_cidr` to that machine's IP (e.g. `"10.0.1.15/32"`).
Leave it as `127.0.0.1/32` if you're only demoing locally on the VM.

### B.3 Run it

```bash
ansible-playbook playbook.yml --vault-password-file .vault_pass
```

### B.4 Live demo commands

```bash
sudo systemctl status redis-server
sudo ufw status verbose
REDIS_PW=$(ansible-vault view group_vars/all/vault.yml --vault-password-file .vault_pass | cut -d'"' -f2)
redis-cli -h 127.0.0.1 -p 6379 -a "$REDIS_PW" --no-auth-warning ping
redis-cli -h 127.0.0.1 -p 6379 ping   # no password -> refused, shows auth is real
```

From a second, allowed machine on the network:
```bash
redis-cli -h <VM_PRIVATE_IP> -p 6379 -a '<password>' ping
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `ERROR! Attempting to decrypt but no vault secrets found` | You didn't create `.vault_pass`, or forgot `--vault-password-file .vault_pass` on the command. |
| `Decryption failed` | Wrong vault password in `.vault_pass` — re-ask your teammate for it, watch for trailing newline/whitespace when pasting. |
| `docker: command not found` | Docker Desktop isn't installed or isn't on PATH — reinstall / restart terminal. |
| `Cannot connect to the Docker daemon` | Docker Desktop isn't running — open the app and wait for it to say "running." |
| `ansible-playbook: command not found` | Ansible isn't installed — see step A.1/B.1 for your OS. |
| Container `systemctl is-system-running` hangs / never says "running" | Docker doesn't have enough resources, or `--privileged`/cgroup mount was dropped — re-check the exact `docker run` command in A.2. |
| `couldn't resolve module/action 'community.general.ufw'` | Run `ansible-galaxy collection install -r requirements.yml` again. |
